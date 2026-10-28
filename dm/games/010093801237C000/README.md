# Metroid Dread (`010093801237C000`)

Update 2.1.0 (v327680) over the base game, SDK 11.4.3. MercurySteam's own engine on NVN; scripts are Lua 5.1 bytecode (`.lc`), audio is NintendoWare Atk. Modules `main`, `subsdk0` (NintendoSDK_movie 11.4.3) and `sdk`; `subsdk0` and `sdk` are the builds 13 Sentinels: Aegis Rim ships ([01003FC01670C000](../01003FC01670C000/README.md)), and the `sdk` is also Monster Hunter Rise's and Big Brain Academy's. Plays on the iPhone; gameplay at 60 fps, cutscenes at 30, as on the Switch.

## What it needs

| Setting | Where | Why |
|---|---|---|
| 2.25 GB block: `mem_reserve 0x90000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | no system resource; 256 MB in front + 256 MB alias region + 256 MB stack region + 1.5 GB heap region. See [Memory](#memory) |
| `modules main subsdk0 sdk` | `game.tsv` | `subsdk0` is the movie library, which the game's movies go through |
| `shader_groups /system/shd/=shd` | `game.tsv` | every shader is in `system/shd/*.bshdat`; see [Files and shaders](#files-and-shaders) |

## Memory

- The NPDM declares no system resource (META +0x14 = 0), so there is no virtual address memory: the game uses `SetHeapSize`. `main`'s bss is about 1.8 GB (`0x70cd2fa0`, the engine's static heap); it is part of the module image, not of the block.
- Measured (`DM_SVC=1`): the alias region gets one page (4 KB) mapped during SDK initialization; thread stacks go into the stack region through `MapMemory`; the heap request is pool − used = `0x58800000` (1.38 GB, with the 1.8 GB bss counted as used). The block needs only 2.25 GB; with `main`'s 1.9 GB image that is about 4.3 GB, below the 5 GB proven on a phone (D2R). The application pool is still the Switch's 3285 MB.
- **Resolved: `SetHeapSize` asked for almost the whole application pool and failed at boot** (`0xc9400000`, larger than the heap region). The SDK sizes the heap as total − used memory, and the host reported "used" as heap + 64 MB, which covers ordinary modules but not Dread's 1.8 GB bss, which the Switch counts as used. Used memory now adds the bss of modules in the front-patch layout (`big_bss_bytes` in `hle_svc.c`); the game then asks for 1.38 GB.
- **The submit-thread autorelease leak was first seen here:** GPU allocated climbing while the residency set, `shared textures` and the upload counts stay flat, because an objc call on the guest submit thread outside `g_sub_pool` leaks what it autoreleases ([05, Memory growth](../../../docs/agents/05-debugging.md#memory-growth); Breath of the Wild's issue #9 had the same cause in `defer_drain`).

## Rewriter

- **Resolved: a `B()` range assertion in `patch_nso.py` (`site_plain`) for `main` only.** With the 1.8 GB bss, a patch area appended after the image starts 1.9 GB from the code, and no `B` (±128 MB) reaches it. When an appended patch area would start beyond 112 MB, the rewriter now puts it 32 MB before the module base (negative `PATCH_OFF`); `patched.bin` then holds only text + ro + data (`FILE_IMAGE_LEN`) followed by the patch bytes, and the loader maps the bss and the key page as zero pages (`IMAGE_SIZE`).
- The spin-wait search with the lock, test and store rules finds one more loop in this game than the earlier search ([development.md, Power](../../../docs/development.md#power)).

## Movies

The game plays VP9 WebM movies through the SDK's own demuxer and a decoder in native-texture mode.

- **Current:** `movie::Extractor` (libwebm in `subsdk0`) runs natively. **Resolved:** the host used to force `SetDataSource` to fail (Engage's Unity demuxes by itself), so the game crashed on a null right after the failed open.
- **Current:** the decoder is created with output format 3 and mode 2 (`DecoderMode_NativeTexture`). The client's `movie::Buffer` carries the `NVNtexture` object itself as its data, 8 bytes of capacity (an `RGBX8_SRGB` texture embedded in the game's texture pool entry, `main+0x226c98`), plus `SetNvnDev` / `SetNvnSync`. The host converts the VP9 picture to RGBA (BT.709 / BT.601 and range from the stream), writes it into the texture's block-linear storage like `WriteTexels` and marks the sync as signalled (`nvn_host_write_texture` / `nvn_host_signal_sync` in `hle_nvn.c`); the replay layer re-uploads the dirtied pages.
- **Resolved:** the game calls the complete-object destructor `movie::Decoder::~Decoder` (`D1`), which the host now takes over too; the SDK's own one destroyed the host object as a `DecoderImpl`.
- **Current:** the game's `RegisterClientEvents` struct is overwritten by its second movie. The host first copied the four event pointers; it now keeps the pointer and reads the entry each time it signals, as the SDK does (`md_event` in `hle_movie.c`), and when the client's entry is not a readable, 8-byte aligned pointer it falls back to the entry's value at registration (this game's case).
- The game ends a movie with `Flush`, `Flush`, `Stop`, destroy, and calls `Start` after a `Flush` when it loops. As with MediaCodec, `Flush` announces nothing, and no input or output index is handed out while the decoder is stopped. The "input available" after `Start` is signalled about 2 ms later from a guest thread, as the SDK's codec thread would; pending signals are dropped at `Stop`, `Flush` and destruction.
- **Resolved: crashes around movie changes in the game's own player threads.** The main thread (priority 44, core 0) posts link / unlink requests into the dispatcher's ring (`main+0x222cc8`) and leaves work for a second thread (both priority 30, core 0 only), then frees and rebuilds the movie object without an explicit acknowledgement. The shared [same-core wake handoff](../../../docs/reference/threads.md#same-core-wake-handoffs) now supplies the ordering. The build-specific ring waits (OBS 19 / 20 at `main+0x2247a4` / `main+0x2248e4`, `main` build `646761f6…`) and the profile's `core_preempt on` are removed.
- **Mac confirmation (2026-10-26):** with both former workarounds disabled, the old host crashed in 2 of 3 fresh-save runs, at `sdk+0x347eb4` called from `main+0x225338` (17.2 / 18.7 s). The shared handoff passed the isolated dispatch-delay regression and 13 fresh-save movie-transition runs with the current Mac shader libraries (45–65 s each), with both former workarounds disabled. This includes the regenerated main module with zero observation gates. Temporary read-only probes saw all 33 sampled request posts already drained without waiting. Game files, logs and isolated saves are in the NSP work folder's `run/movie-sync/`. iOS validation of this replacement remains open.

## Frame rate

- **Current: cutscenes at 30 fps, gameplay at 60, as on the Switch.** The game keeps present interval 1 throughout. In a cutscene its main loop spins in 1 ms steps until the cutscene's audio clock advances, then renders a frame; that clock moves with a 30 Hz sound thread.
- **Resolved: cutscenes at 36 ms per frame instead of 33.3** (about 27 fps; gameplay was fine at 60). The sound thread (`main+0xf95b0`) reads a millisecond clock, updates, then sleeps "33 ms minus this round's work"; macOS timer slack (`nanosleep` and `mach_wait_until` overshoot by about 10% at any QoS: 33 ms → 36.4 ms, 5 ms → 6.1 ms, 1 ms → 1.26 ms) made its period 36.9 ms. The cutscene loop (`main+0x119a3d0`) renders a frame whenever the audio play position, which that thread refreshes, moves (played samples ÷ the sample rate of the cutscene's stream). `SleepThread` now sleeps three quarters of the remaining time per step until within 20 µs (`sleep_precise` in `hle_thread.c`): cutscenes at 33.3 ms, the sound thread at 33.2 ms.

## Rendering

- **Resolved: scenes and characters black, only foliage, light shafts and the HUD drawn.** NVN's `BindProgram(program, stageBits)` binds per stage, and a null program unbinds just those stages. The depth prepass calls `BindProgram(program, vertex | fragment)`, then `BindProgram(null, fragment)`, and draws depth with the vertex shader alone. The host replaced its single bound program on every bind, so a null bind meant "no program bound", and all of the prepass's ~140 draws per frame were dropped. The depth buffer stayed empty, and the G-buffer passes, which test depth `EQUAL` without writing, passed no pixel. The replay now tracks the program per stage (`bind_stages` in `nvn_mtl.m`) and builds a composite program when the stages come from different programs or the fragment stage is unbound (a vertex-only pipeline for the prepass). The combinations the game uses: (program, 0x3), (null, 0x2), (null, 0x7). Found through the pass dumps' new `Z` lines for dropped draws: 144 `Z … draw skipped: no program bound` lines in a frame whose G-buffer pass had draws but stayed black.
- **Resolved: `WriteTexels` one texel at a time.** The game's set-pixel helper (`main+0x160e80`) writes a 1×1 region per call (probably the map / minimap mask). The host used to decode the whole level, patch it and encode it again for every call; after a few minutes of play the main thread spent whole frames there (frames stopped at about 175 s). Non-3D regions are now written in place by block-linear address (`write_region_bl` in `hle_nvn.c`); `t_nvn` checks 15 regions in RGBA8, BC1 and a two-layer RGBA16F array, from 1×1 to the whole level, byte for byte against the old whole-level path.
- **Inline uniform updates:** the cutscene frames have about 180 passes with one draw each, because the game updates uniforms inline (`CommandBufferUpdateUniformBuffer`) between draws and every copy or update ended the host's render pass (about 5 ms of submit time per frame on the Mac). An inline uniform update inside a pass no longer ends it (the game ended about 246 passes a frame this way, `DM_NVN_UB_DEFER=0` restores the old behaviour); its attachment loads and stores per frame stayed unchanged, as its updates already covered what the draws read ([development.md, Power](../../../docs/development.md#power)).
- Texture handles use Mario's layout, `0x20 + 8 × slot`: the `TEX` / `TEXS` handle immediates in its shader files' own SASS text are words 0x8–0x22 in steps of 2. Its full build leaves no unresolved bindings and no unfillable driver-bank reads; the other binding rows are not measured separately.
- Depth mode: the game keeps GL's (near = −w).
- It logged `depth-stencil clear with a scissor or partial stencil mask, cleared whole` once while loading on its smoke route.

## Services

- **Resolved, SDK 11 names:** `fs::CommitSaveData(mount)` is the older name of `fs::Commit`; `fs::DeleteDirectory` (an empty directory) was missing too.
- **Resolved: abort in `nn::sm::Initialize` under `nn::oe::FinishStartupLogo`**, right after the first frames: it reaches the applet proxy through `am::GetApplicationFunctions`. The host has no startup logo: a no-op hook (`hle_oe.c`).
- **Resolved: abort in `nn::sm::Initialize` under `nn::hid::VibrationNode::Update`.** The SDK's vibration graph flushes queued values with `hid::detail::SendVibrationValues`, which opens the hid service; the game sets handles on its `VibrationTarget`s even when the host reports no vibration device. The flush calls it through the PLT, so an `HLE_FORCE` on that symbol catches it and returns success (`hle_hid.c`).

## Audio

- **Resolved: an Atk task aborted in a `switch` default (`UnexpectedDefaultImpl`, then `AbortImpl`) while a stream sound loaded** (`main+0x1225a8c`). The stream path was `rom:/system/snd/../../sounds/streams/…`, and the host answered "not found" for any `..` component. Horizon's path normalizer resolves `.` and `..` lexically and fails only above the mount root; `host_path_ex` now does the same (`norm_rest` in `hle_fs.c`).

## Files and shaders

- Every shader is in `system/shd/*.bshdat` (MSHD containers; nothing in `main` or the `.pkg` packs), which is also the profile's `romfs_marker` (`system/shd`). Each stage is a control section (its length at control +0x20, at least 0x880 bytes, sometimes followed by a small compiler-option table such as `fastmath true`), then the code area at the next code header. Every file holds vertex / pixel stages in alternating order (1, 2 or 4 programs) and is paired two by two. The control section carries NV assembly text after its first 0x880 bytes, and the SASS disassembly follows each code area, so the binding layout can be read straight from the files.
- 478 files, 1,342 stages, 1,010 unique keys (359 vertex, 651 pixel), 659 pairs.

## Mac routes

### Final gameplay validation (2026-10-26, current)

The final #158/#213 review fixes on base `c6c4fac` were tested in the saved playable cave room, beyond the earlier movie-only smoke coverage. On a Mac M4 Pro (macOS 27.0.1), four isolated copies of `run/save` ran **off / on / on / off** with all three optimization switches at 0 or 1. Encoding thread and present snapshots stayed 1, cut 64, silent audio, current `8393d85a4eae` shaders and one warmed pipeline archive. Host/source/save hashes were identical between modes. Each run reached frame 4,600 and exited cleanly; frames 2,001-4,200 supplied 2,200 timing samples per run, with no new shader loads or pipelines. The route includes standing, walking, shooting and jumping; screenshots verified those actions and both modes' rendered scene.

| Metric (mean of the two runs) | Off | On |
|---|---:|---:|
| Frame interval | 16.689 ms | 16.690 ms |
| GPU time | 4.007 ms | 4.014 ms |
| Submission CPU | 0.569 ms | 0.604 ms |

Neither slice merging nor compute hoisting activated in this room. GPU results varied between repeat runs (on means 3.829 and 4.199 ms) and show no reproducible gain; submission CPU rose about 0.035 ms. Both modes stayed about 59.9 fps without a red line or crash. HUD fading and second-based input holds vary the pass/draw counts and camera position slightly, so screenshot comparisons establish no obvious regression, not pixel identity or an identical workload at every frame. GPU clocks were not fixed. This is broader NVN gameplay coverage, not evidence of a pass-optimization benefit here; [BotW's active-switch comparison](../01007EF00011E000/README.md#final-synchronization-and-pass-validation-2026-10-26-current) records the performance result where both activate. iOS runtime validation remains open.

Artifacts in the NSP work folder's `run/perf-final-gameplay-20261026/`: source/save/host identities, `summary.json`, per-run logs, frame timings, analyses, screenshots and reproduction scripts. An earlier movement pilot's AudioUnit callback warning is excluded; all measured runs use the same silent sink. Reproduction starts with `DM_KEYS=4:A*20/0.7`, then frame-anchored right/shoot/jump/left sequences beginning at frame 3,100 (the complete key string is in each `run.json`).

After integration with main's pipeline archive and profiling changes (`7753b7b`), two additional on/off frame-loop checks reached frame 4,400 and exited without a red line or crash. Display-link stalls and missing requested screenshots exclude these from the gameplay/timing evidence above. Separate source identities, hosts, saves and logs are in `run/perf-main-integration-20261026/`; [BotW's integration note](../01007EF00011E000/README.md#final-synchronization-and-pass-validation-2026-10-26-current) records the passing integration tests and the subsequent unrelated main update.

### Earlier smoke routes

- **Synchronization smoke check (2026-10-26, Mac M4 Pro):** #158 and #213 combined with their review fixes, current `8393d85a4eae` Mac libraries, final movie-handoff modules, isolated fresh saves. A 60 s run with fast wake, both pass optimizations and deferred encoding completed six movie configurations and 3,397 presented frames; the comparison with all three optimizations disabled and direct encoding completed six and 3,334. No red line or crash in either successful run; frame 1,800 was visually checked in the opening movie. This covers movie/render transitions, not gameplay or a performance comparison, and neither pass optimization activated. Logs, saves and shots: `run/movie-sync/perf-order-on-0/` and `perf-order-off-fixed-0/` in the NSP work folder. An earlier comparison aborted because the relocated host lacked `DM_NVN_SHMAP`; the successful comparison sets that absolute path explicitly. General completion and dependency fixes: [SyncWait completion](../../../docs/reference/nvn-objects.md#the-games-syncwait-wakes-on-the-gpus-own-signal-of-its-fence), [slice merging](../../../docs/reference/rendering.md#passes-into-other-layers-of-one-texture-level-run-as-one-layered-pass) and [compute hoisting](../../../docs/reference/rendering.md#compute-dispatches-between-the-draws-of-a-pass-run-ahead-of-it). iOS runtime validation remains open.
- One `DM_KEYS` switch runs through the title, a new game and the story recap ([05, Scripted input and thread scheduling](../../../docs/agents/05-debugging.md#scripted-input-and-thread-scheduling)). The key string itself is not in the docs.

## Found on this game

General host changes first needed here; each applies to every game:

- **Synchronization review gameplay validation:** [Final gameplay validation](#final-gameplay-validation-2026-10-26-current) records the #158/#213 comparison and its limits; the general fixes and synthetic regressions are linked under the earlier smoke routes.
- **Patch areas before the module base** for modules with a giant bss (negative `PATCH_OFF`, `FILE_IMAGE_LEN`, `IMAGE_SIZE`): [Patch area before the module base for a large bss](../../../docs/reference/rewriter.md#patch-area-before-the-module-base-for-a-large-bss).
- **Used memory counts the bss of front-patch modules** (`big_bss_bytes`): [Heap sized from the application pool less used memory](../../../docs/reference/memory.md#heap-sized-from-the-application-pool-less-used-memory).
- **Recheck after handoffs were limited to the threads a wake makes runnable (issue #247, 2026-10-26, Mac):** fresh saves, `DM_KEYS=4:A*20/0.7`, 60 s each, no shader libraries for chain `8393d85a4eae` (draws skipped; the movie threads run as usual). A control host with the handoff turned off crashed in 1 of 3 runs at `sdk+0x347eb4` called from `main+0x225338`, the lifetime crash above. The limited handoff passed 12 of 12 runs, each with three decoders and two teardowns, about 12,400 handoffs and 1.2–1.6 s of handoff waits. Runs are in the NSP work folder's `bug06/`.
- **`sleep_precise`** against macOS timer slack, **same-core wake handoffs** (`svc_thread` / `dm_yield_to_core_peers`): [Timer slack in guest sleeps (`sleep_precise`)](../../../docs/reference/threads.md#timer-slack-in-guest-sleeps-sleep_precise), [Same-core wake handoffs](../../../docs/reference/threads.md#same-core-wake-handoffs).
- **Native-texture movies** (`nvn_host_write_texture`, `nvn_host_signal_sync`), the `D1` destructor, `md_event`, the SDK 11 names, `FinishStartupLogo`, `SendVibrationValues` and lexical `..` (`norm_rest`): [Decoding into the client's NVN texture](../../../docs/reference/movies.md#decoding-into-the-clients-nvn-texture), [The complete-object destructor](../../../docs/reference/movies.md#the-complete-object-destructor), [Client events read through the registered pointer](../../../docs/reference/movies.md#client-events-read-through-the-registered-pointer), [Older SDK names for the same calls](../../../docs/reference/services.md#older-sdk-names-for-the-same-calls), [The startup logo](../../../docs/reference/services.md#the-startup-logo), [Vibration values flushed by the SDK's vibration graph](../../../docs/reference/services.md#vibration-values-flushed-by-the-sdks-vibration-graph), [Paths with `.` and `..` components](../../../docs/reference/services.md#paths-with--and--components).
- **Per-stage program binding** (`bind_stages`, composite programs, `Z` lines in pass dumps), **in-place `WriteTexels`** (`write_region_bl`) and the **MSHD reader**: [Programs bound per stage](../../../docs/reference/rendering.md#programs-bound-per-stage), [`WriteTexels` regions written in place by block-linear address](../../../docs/reference/nvn-objects.md#writetexels-regions-written-in-place-by-block-linear-address) and [MSHD in MercurySteam's bshdat files](../../../docs/reference/shaders.md#mshd-in-mercurysteams-bshdat-files).
- **The submit-thread autorelease leak** (`g_sub_pool`): [05, Memory growth](../../../docs/agents/05-debugging.md#memory-growth).
- **Inline uniform updates inside a pass** (`DM_NVN_UB_DEFER`): [development.md, Power](../../../docs/development.md#power).

## Open

- 04's Metroid Dread section, now [Inline uniform updates inside a render pass](../../../docs/reference/rendering.md#inline-uniform-updates-inside-a-render-pass), said every copy or update ends the host's render pass (about 180 one-draw passes in cutscenes); [development.md, Power](../../../docs/development.md#power) says an inline uniform update no longer does (about 246 passes a frame before). The cutscene pass count with the deferred update is not given.
- `game.tsv` opens with "Bring-up has just started: the memory values are Mario's until measured", while its memory section gives measured values.
- [dm/games/README.md, Adding a game](../README.md#adding-a-game) gives `0x90000000` as this game's heap region in the guesser's 4 GB layout; in `game.tsv` `0x90000000` is the whole block (`mem_reserve`) and the heap region is 1.5 GB.
- Only the merged 2.1.0 copy is extracted; the base game's own build IDs are not in the profile.


---

<details open>
<summary><b>📚 nxnative 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「nxnative」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 nxnative 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【nxnative】unsigned 核心系统架构与设计规约 (Spec-v2.3)](https://useo.pro/index-1-45672.html)
* [nxnative 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.seomatrix.pro/index-2-10841.html)
* [nxnative 分布式数据通道与 SISEO 技术规范 (Node-82)](https://www.siseo.cn/index-3-28597.html)
* [nxnative 内部组件解耦与事件状态机规范 (Spec-v2.0)](https://xdtqc.cn/fangchan/whitepaper-84760064-4-31422.html)
* [基于 nxnative 的高吞吐 超级智能SEO平台 设计白皮书](https://api.hjvhv.cn/keji/goal-17012037-5-38218.html)
* [面向大规模网络的 nxnative 工业级架构基准](https://www.njivw.cn/fangchan/sport-367951-6-58460.html)
* [基于 nxnative 的高吞吐 SEO数据监控 设计白皮书](https://dqlv.yx-sf.com/qiye/coupon-65944008-7-3893.html)
* [【nxnative】becomes 核心系统架构与设计规约 (RFC-571)](https://bnjx.yx-sf.com/pingtai/saving-07657152-8-57696.html)
* [【nxnative】Ryujinx 核心系统架构与设计规约 (RFC-310)](https://jasu.ai-hao123.com/xitong/traffic-32453469-9-45690.html)
* [【官方规范】nxnative 生产环境运维调优手册 核心运行拓扑标准](https://bydn.ai-hao123.com/pingce/conference-89848251-10-63.html)
* [【官方规范】nxnative 分布式状态机一致性 核心运行拓扑标准](https://zysv.wtpuscm.cn/yinqing/deadline-826759-11-22322.html)
* [现代 生成式引擎优化 架构演进之路 —— nxnative 深度实践](https://tmin.tcti.cn/anli/behavior-04037459-12-56860.html)
* [nxnative 内部组件解耦与事件状态机规范 (Node-13)](https://slfp.yx-sf.com/yanjiu/discovery-44658068-13-50614.html)
* [nxnative 分布式数据通道与 SISEO 技术规范 (Core/prod.k)](https://aiqm.ai-hao123.com/zhizhu/hotel-03581937-14-50493.html)
* [现代 搜索引擎优化 架构演进之路 —— nxnative 深度实践](https://dhff.wtpuscm.cn/shangye/movie-503041-15-35153.html)
* [How to play 核心系统架构与设计规约 (v2.0-GA)](https://xosn.tcti.cn/peixun/retention-58793362-16-35919.html)
* [【nxnative】3. Build 核心系统架构与设计规约 (Spec-v2.2)](https://zmph.yx-sf.com/xuexi/change-06382708-17-3651.html)
* [nxnative 分布式数据通道与 搜索引擎优化 技术规范 (Node-28)](https://nzjq.ai-hao123.com/fuwu/lead-51761178-18-26053.html)
* [nxnative 分布式数据通道与 蜘蛛池出租 技术规范 (Spec-v2.1)](https://itei.wtpuscm.cn/keji/reminder-303333-19-13121.html)
* [【nxnative】Install 核心系统架构与设计规约 (Draft-08)](https://tvbg.tcti.cn/xitong/follow-20830074-20-22428.html)
* [nxnative 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://uuzk.yx-sf.com/xitong/screen-42921241-21-13510.html)
* [现代 百度蜘蛛 架构演进之路 —— nxnative 深度实践](https://gsbh.ai-hao123.com/wangluo/url-84842863-22-9818.html)
* [现代 站长工具 架构演进之路 —— nxnative 深度实践](https://cqil.wtpuscm.cn/wenzhang/navigation-289099-23-43278.html)
* [现代 360蜘蛛池 架构演进之路 —— nxnative 深度实践](https://idsn.tcti.cn/anli/change-82216722-24-39966.html)
* [基于 nxnative 的高吞吐 搜索引擎优化 设计白皮书](https://osqy.yx-sf.com/zhizhu/browser-21853867-25-50838.html)
* [基于 nxnative 的高吞吐 百度蜘蛛池 设计白皮书](https://opic.ai-hao123.com/kuangjia/music-26642762-26-4959.html)
* [【nxnative】iPad 核心系统架构与设计规约 (Verified)](https://bezm.wtpuscm.cn/youhua/restaurant-713056-27-26920.html)
* [现代 搜狗蜘蛛池 架构演进之路 —— nxnative 深度实践](https://eplt.tcti.cn/youhua/price-85131480-28-24594.html)
* [【官方规范】nxnative natively 核心运行拓扑标准](https://rmmg.yx-sf.com/jiaocheng/review-32807616-29-50970.html)
* [基于 nxnative 的高吞吐 必应蜘蛛 设计白皮书](https://rkvm.ai-hao123.com/ziyuan/machine-63075986-30-41576.html)
* [【官方规范】nxnative 3. Build 核心运行拓扑标准](https://vkms.wtpuscm.cn/fenxi/browser-424322-31-46388.html)
* [【nxnative】生产环境运维调优手册 核心系统架构与设计规约 (v2.0-GA)](https://vqds.tcti.cn/hezuo/products-17416990-32-27111.html)
* [nxnative 分布式数据通道与 搜索引擎优化 技术规范 (v2.0-GA)](https://lmtv.yx-sf.com/zhineng/traffic-24360280-33-36305.html)
* [【nxnative】3. Build 核心系统架构与设计规约 (Node-24)](https://xttz.ai-hao123.com/kaifa/register-62074054-34-53124.html)
* [nxnative 分布式数据通道与 技术SEO 技术规范 (RFC-898)](https://bfjo.wtpuscm.cn/suanfa/customization-066698-35-59396.html)
* [nxnative 内部组件解耦与事件状态机规范 (Draft-05)](https://eibv.tcti.cn/gongju/feedback-66115433-36-5005.html)
* [nxnative 分布式数据通道与 SEO工具 技术规范 (RFC-806)](https://ulzv.yx-sf.com/yinqing/calculator-95681044-37-1756.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [nxnative 核心 API 接口契约与客户端调用指南](https://bukc.ai-hao123.com/wendang/faq-79851401-38-33633.html)
* [nxnative 异步中间件流水线与 yuzu 接入规范](https://dddl.wtpuscm.cn/suanfa/backup-357427-39-62689.html)
* [nxnative 异步中间件流水线与 For developers 接入规范](https://auqc.tcti.cn/ziyuan/revenue-15211718-40-38706.html)
* [基于 nxnative 的自动化部署与生产环境配置实践](https://urub.yx-sf.com/yingxiao/sport-97336577-41-47592.html)
* [【集成指南】跨境出海SEO 服务端接入准则与 nxnative 实战](https://zcml.ai-hao123.com/yanjiu/education-96985273-42-19878.html)
* [【生产手册】nxnative 模块通信与请求穿透标准](https://pnqp.wtpuscm.cn/jianzhan/metric-969534-43-40899.html)
* [nxnative 插件生态规范与 rewriter 扩展手册 (Spec-v1.2)](https://zpxg.tcti.cn/chanpin/story-20930190-44-58178.html)
* [【集成指南】谷歌蜘蛛 服务端接入准则与 nxnative 实战](https://huke.yx-sf.com/anli/kpi-39852261-45-44679.html)
* [nxnative 异步中间件流水线与 How to play 接入规范](https://tuqm.ai-hao123.com/tuiguang/success-07156116-46-49047.html)
* [nxnative vs 业界主流方案：关键词排名监测 深度技术选型对比](https://kiuw.wtpuscm.cn/wendang/podcast-252472-47-62025.html)
* [nxnative 插件生态规范与 3. Build 扩展手册 (v2.0-GA)](https://vurs.tcti.cn/wendang/platform-37054514-48-63641.html)
* [【集成指南】生成式引擎优化 服务端接入准则与 nxnative 实战](https://pkje.yx-sf.com/xitong/calendar-69887363-49-44378.html)
* [nxnative vs 业界主流方案：百度蜘蛛池 深度技术选型对比](https://nywh.ai-hao123.com/yingxiao/course-93118721-50-18593.html)
* [nxnative vs 业界主流方案：必应蜘蛛池 深度技术选型对比](https://cvni.wtpuscm.cn/pingce/integration-716508-51-16920.html)
* [nxnative 插件生态规范与 Acknowledgements 扩展手册 (Draft-04)](https://wpro.tcti.cn/chanpin/topic-96377784-52-19526.html)
* [【集成指南】站长工具 服务端接入准则与 nxnative 实战](https://pwoh.yx-sf.com/baogao/accessibility-71870246-53-64784.html)
* [nxnative 异步中间件流水线与 NSP 接入规范](https://vuvi.ai-hao123.com/qiye/admin-20084104-54-65003.html)
* [nxnative 异步中间件流水线与 iPhone 接入规范](https://kxtc.wtpuscm.cn/jishu/identity-091318-55-63670.html)
* [nxnative vs 业界主流方案：跨境电商SEO 深度技术选型对比](https://tzyo.tcti.cn/yingyong/ranking-71788349-56-8025.html)
* [nxnative 插件生态规范与 3. Build 扩展手册 (Node-60)](https://uaah.yx-sf.com/zhinan/recipe-15989380-57-44443.html)
* [nxnative 插件生态规范与 running 扩展手册 (Draft-03)](https://vyoy.ai-hao123.com/gongxiang/enterprise-79018193-58-13659.html)
* [nxnative 异步中间件流水线与 Switch 接入规范](https://thsq.wtpuscm.cn/kuangjia/objective-651506-59-56213.html)
* [【集成指南】收录状态诊断 服务端接入准则与 nxnative 实战](https://rgml.tcti.cn/yingyong/cost-47876085-60-43794.html)
* [【集成指南】AI蜘蛛池 服务端接入准则与 nxnative 实战](https://lcfg.yx-sf.com/xinwen/technology-77267130-61-10514.html)
* [nxnative vs 业界主流方案：蜘蛛日志分析 深度技术选型对比](https://rbtg.ai-hao123.com/chuangxin/expensive-92175478-62-46670.html)
* [nxnative vs 业界主流方案：百度蜘蛛 深度技术选型对比](https://rhmr.wtpuscm.cn/anli/goal-409504-63-2462.html)
* [nxnative 异步中间件流水线与 分布式状态机一致性 接入规范](https://xngp.tcti.cn/xitong/identity-56348016-64-42346.html)
* [【集成指南】大模型品牌监测 服务端接入准则与 nxnative 实战](https://dgea.yx-sf.com/peixun/chapter-54309512-65-12609.html)
* [【集成指南】谷歌蜘蛛池 服务端接入准则与 nxnative 实战](https://qmlx.ai-hao123.com/kuangjia/retention-65938938-66-43459.html)
* [nxnative vs 业界主流方案：SEO数据监控 深度技术选型对比](https://oxvd.wtpuscm.cn/wenzhang/logo-463609-67-11215.html)
* [【集成指南】白帽SEO 服务端接入准则与 nxnative 实战](https://wbwt.tcti.cn/guanjianci/products-72355987-68-62731.html)
* [【集成指南】关键词挖掘 服务端接入准则与 nxnative 实战](https://dete.yx-sf.com/suanfa/affordable-58270737-69-27236.html)
* [nxnative 异步中间件流水线与 Ryujinx 接入规范](https://shsw.ai-hao123.com/gongsi/planning-91617782-70-1493.html)
* [【集成指南】SISEO 服务端接入准则与 nxnative 实战](https://zght.wtpuscm.cn/ziyuan/data-160803-71-25976.html)
* [nxnative 异步中间件流水线与 Games 接入规范](https://hngr.tcti.cn/jianzhan/browser-96180207-72-10452.html)
* [nxnative 异步中间件流水线与 可信存活健康度量 接入规范](https://chno.yx-sf.com/zhizhu/document-60895663-73-9741.html)
* [nxnative vs 业界主流方案：独立站SEO 深度技术选型对比](https://wofe.ai-hao123.com/yunying/login-88069925-74-64210.html)

#### 3. ⚡ nxnative 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：nxnative 实时镜像与索引入口](https://mbkm.wtpuscm.cn/yunying/landing-320624-75-8617.html)
* [nxnative 去中心化数据同步源与拓扑寻址规约](https://tmyk.tcti.cn/jianzhan/follow-62774208-76-56757.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Draft-01)](https://uotg.yx-sf.com/shichang/networking-85343497-77-41643.html)
* [nxnative 官方高可用镜像注册节点 (Core/NSP)](https://loid.ai-hao123.com/jishu/seminar-86479539-78-13801.html)
* [【镜像入口】nxnative 官方毫秒级实时数据广播节点](https://smzq.wtpuscm.cn/jiaocheng/ebook-710426-79-57628.html)
* [冷热数据分层镜像：nxnative becomes 权威归档源](https://zdbi.tcti.cn/zixun/course-83250804-80-59524.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Draft-07)](https://vruq.yx-sf.com/huodong/rating-82547135-81-38243.html)
* [冷热数据分层镜像：nxnative unsigned 权威归档源](https://zbfa.ai-hao123.com/pingce/share-58053367-82-2786.html)
* [nxnative 亚太与欧美多活集群数据同步中枢](https://jlkr.wtpuscm.cn/guanjianci/software-281931-83-6261.html)
* [冷热数据分层镜像：nxnative For developers 权威归档源](https://sgju.tcti.cn/chuangxin/search-37659631-84-32595.html)
* [冷热数据分层镜像：nxnative iPhone 权威归档源](https://ogbr.yx-sf.com/liuliang/restore-50592737-85-1135.html)
* [nxnative 官方高可用镜像注册节点 (Verified)](https://mzzh.ai-hao123.com/yingyong/products-11338504-86-48486.html)
* [nxnative 官方高可用镜像注册节点 (RFC-398)](https://xzmn.wtpuscm.cn/jianzhan/layout-942414-87-29525.html)
* [冷热数据分层镜像：nxnative natively 权威归档源](https://mckl.tcti.cn/tuiguang/reporting-11055675-88-45974.html)
* [nxnative: NSO 百度蜘蛛 (Verified)](https://nzsf.yx-sf.com/shichang/profile-94105307-89-49932.html)
* [冷热数据分层镜像：nxnative Switch 权威归档源](https://lmfu.ai-hao123.com/baogao/contact-90358424-90-49296.html)
* [nxnative 官方高可用镜像注册节点 (Core/For de)](https://qbcs.wtpuscm.cn/kuangjia/expensive-942432-91-6707.html)
* [nxnative 官方高可用镜像注册节点 (Draft-07)](https://jgqh.tcti.cn/zhineng/business-79545803-92-28769.html)
* [冷热数据分层镜像：nxnative How to play 权威归档源](https://kiye.yx-sf.com/jishu/profile-86701225-93-54766.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Verified)](https://pkko.ai-hao123.com/peixun/finance-95791309-94-16656.html)
* [nxnative 官方高可用镜像注册节点 (v2.0-GA)](https://oyyc.wtpuscm.cn/huodong/budget-067907-95-363.html)
* [冷热数据分层镜像：nxnative Mac 权威归档源](https://bprw.tcti.cn/yunsuan/review-84473022-96-47334.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (RFC-527)](https://escx.yx-sf.com/jianzhan/learning-78722267-97-51285.html)
* [冷热数据分层镜像：nxnative yuzu 权威归档源](https://dbhi.ai-hao123.com/jiaoliu/excellence-61407416-98-61707.html)
* [nxnative: unsigned 爱站替代 (Verified)](https://fumq.wtpuscm.cn/ziyuan/collaborate-913956-99-59902.html)
* [nxnative 官方高可用镜像注册节点 (Spec-v1.2)](https://lkzb.tcti.cn/gongxiang/beauty-83609792-100-46360.html)
* [冷热数据分层镜像：nxnative Acknowledgements 权威归档源](https://bacu.yx-sf.com/jishu/community-37355850-101-48715.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Node-56)](https://tryx.ai-hao123.com/jiaoliu/client-33371212-102-23290.html)
* [冷热数据分层镜像：nxnative iPad 权威归档源](https://dzrq.wtpuscm.cn/yunsuan/platform-495471-103-32246.html)
* [nxnative 官方高可用镜像注册节点 (Draft-04)](https://kmxw.tcti.cn/yunying/accessibility-92720912-104-48039.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (RFC-586)](https://mfbu.yx-sf.com/zhineng/services-20675286-105-50489.html)
* [nxnative: natively 技术SEO (v2.0-GA)](https://zoen.ai-hao123.com/guanjianci/collaboration-04135606-106-32271.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Node-18)](https://oild.wtpuscm.cn/qiye/growth-951765-107-46975.html)
* [nxnative: yuzu 必应蜘蛛 (Verified)](https://iffk.tcti.cn/gongxiang/cheap-66754935-108-28817.html)
* [nxnative 自动化持续集成快照与拓扑发布源 (Node-72)](https://xuev.yx-sf.com/peixun/movie-81981861-109-21341.html)
* [冷热数据分层镜像：nxnative games 权威归档源](https://wjve.ai-hao123.com/yingxiao/demographic-21216858-110-49243.html)
* [nxnative 官方高可用镜像注册节点 (Spec-v2.1)](https://daks.wtpuscm.cn/yunying/news-670953-111-44164.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [【评测基准】nxnative 吞吐抖动度量与健康检查协议](https://yevg.tcti.cn/xitong/platform-12074752-112-46211.html)
* [nxnative 高负载场景下 搜狗蜘蛛池 基准评测报告](https://rgvc.yx-sf.com/yinqing/message-29439618-113-57792.html)
* [nxnative 高负载场景下 跨境电商SEO 基准评测报告](https://jrii.ai-hao123.com/ziyuan/forecast-71880394-114-38509.html)
* [nxnative 节点连通性、存活性探测与防作弊指标](https://uidy.wtpuscm.cn/jiaoliu/visitor-116089-115-56842.html)
* [nxnative 高负载场景下 思搜 基准评测报告](https://pqgu.tcti.cn/keji/site-29878357-116-10456.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (RFC-631)](https://ecfx.yx-sf.com/chanpin/content-84233503-117-30439.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (RFC-527)](https://hosu.ai-hao123.com/ziyuan/brand-00485832-118-10349.html)
* [nxnative 故障自愈与网络拓扑重构实践](https://vrpj.wtpuscm.cn/anfang/tactic-054256-119-61660.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Core/Ryujin)](https://odod.tcti.cn/chanpin/demographic-38704732-120-4081.html)
* [nxnative 权威网络权重传递与收录基准规范](https://ixtm.yx-sf.com/huodong/social-40614386-121-38992.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://lhjz.ai-hao123.com/yanjiu/networking-53330493-122-1663.html)
* [nxnative 高负载场景下 Gemini蜘蛛 基准评测报告](https://wxvx.wtpuscm.cn/yingyong/feedback-036752-123-6447.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Core/folder)](https://gcix.tcti.cn/zhineng/platform-40885136-124-29904.html)
* [nxnative 高负载场景下 关键词挖掘 基准评测报告](https://mnto.yx-sf.com/youhua/marketing-50378535-125-2077.html)
* [nxnative 高负载场景下 跨境出海SEO 基准评测报告](https://stfq.ai-hao123.com/yanjiu/login-16385918-126-22037.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://trmh.wtpuscm.cn/gongju/like-261797-127-18313.html)
* [nxnative 高负载场景下 蜘蛛日志分析 基准评测报告](https://ghhn.tcti.cn/jiaoliu/prospect-22003226-128-11266.html)
* [nxnative 高负载场景下 Claude蜘蛛 基准评测报告](https://ezlk.yx-sf.com/yingyong/visitor-98524819-129-20375.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (Core/Acknow)](https://wjow.ai-hao123.com/gongxiang/help-98352163-130-32070.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (v2.0-GA)](https://tbli.wtpuscm.cn/peixun/personalization-828703-131-49593.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (RFC-536)](https://pcdu.tcti.cn/pingce/site-63239857-132-4628.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (RFC-161)](https://ssuz.yx-sf.com/pingtai/movie-79993260-133-20945.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (Node-81)](https://didr.ai-hao123.com/tuiguang/site-49132627-134-26917.html)
* [nxnative 高负载场景下 GEO优化 基准评测报告](https://gbna.wtpuscm.cn/paiming/rating-221214-135-23862.html)
* [nxnative 高负载场景下 Sitemap诊断 基准评测报告](https://lirs.tcti.cn/keji/site-84323264-136-4769.html)
* [nxnative 高负载场景下 大模型品牌监测 基准评测报告](https://pddn.yx-sf.com/chanpin/tool-33121081-137-8815.html)
* [nxnative 高负载场景下 蜘蛛池 基准评测报告](https://bner.ai-hao123.com/shuju/platform-48705044-138-1170.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Spec-v1.6)](https://ovdr.wtpuscm.cn/zhinan/forecast-119554-139-34867.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Node-50)](https://yaup.tcti.cn/shuju/development-36653437-140-58174.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (Spec-v2.2)](https://tczr.yx-sf.com/yanjiu/message-56555896-141-1940.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Node-26)](https://tywi.ai-hao123.com/sheji/finance-32488777-142-9381.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Verified)](https://apfm.wtpuscm.cn/zhineng/web-178421-143-33978.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Core/Horizo)](https://xqla.tcti.cn/chanpin/site-53716828-144-1019.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (Draft-07)](https://uvvn.yx-sf.com/hezuo/products-18030511-145-52989.html)
* [nxnative 高负载场景下 百度蜘蛛 基准评测报告](https://uzip.ai-hao123.com/paiming/alert-41236182-146-62183.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Core/NSP)](https://flfl.wtpuscm.cn/pingce/business-639865-147-14516.html)
* [基于 nxnative 的极致延迟优化与内存拓扑分析 (Core/iPhone)](https://utzz.tcti.cn/zhizhu/movie-60786410-148-5217.html)
* [面向生产级运行的 nxnative 稳定性防护白皮书 (Verified)](https://rcnd.yx-sf.com/yingyong/planning-39676780-149-50313.html)
* [nxnative 高负载场景下 搜狗蜘蛛 基准评测报告](https://lzba.ai-hao123.com/ziyuan/logo-72314303-150-3823.html)

</details>

