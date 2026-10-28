# Fire Emblem: Three Houses (`010055D009F78000`)

Update 1.2.0 (v327680) over the base game, SDK 6.5.0. Koei Tecmo's older KTGL engine on NVN (the game is co-developed with Intelligent Systems). Modules `main`, `subsdk0` (GLSLC, NVN's runtime shader compiler), `subsdk1` (the movie library, NintendoSDK_movie) and `sdk`. Plays on the iPhone; 60 fps (30 on the Switch) through rewriter cuts on 1.2.0 `main`.

## What it needs

| Setting | Where | Why |
|---|---|---|
| Present interval table → 1 for every render mode (int and float tables) | [`patches.py`](patches.py) `CUTS`, `main` build `89048449…` | 60 fps at the same game speed: KTGL already updates the game 60 times a second; see [Frame rate](#frame-rate) |
| Render thread's forced interval 2 ignored (`main+0x50fb7c`) | `patches.py` | the opening movie and scenes such as character creation run at 60 too |
| Frame limiter sleeps instead of yielding (`main+0x5be4c0`–`0x5be4d0`) | `patches.py` | its yield loop spun on the host: 59.5 updates a second and 1.5 W of CPU on the Mac |
| Render thread waits a frame instead of 16 µs (`main+0x50f8e8`–`0x50f8f4`) | `patches.py` | its wait polled about 50,000 times a second at 60 fps |
| GLSLC capture (`/glslc_capture/=runtime` in `shader_groups`) | [`game.tsv`](game.tsv), NSP script | 9 built-in shaders exist only as compiles of the game's own GLSLC at boot; see [Files and shaders](#files-and-shaders) |
| 4 GB layout: `mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | the NPDM declares no system resource (META +0x14 = 0): the game uses `SetHeapSize`. Link's Awakening's partition, not measured for this game |
| `pad_layout full` | `game.tsv` | the 3D gamepad layout: the camera is on the right stick in the monastery and in battle |

## Frame rate

- KTGL runs the game on the main thread at 60 updates a second whatever it shows: each update steps by 1/fps with fps = 60.0 (`main+0x1a38f08`, set at `main+0x50e114`; dt at `main+0x510010`), and the frame end (`main+0x50f680`) waits out 60/fps ticks of 1/59.94 s in a yield loop (`main+0x5be430`).
- The render thread (`main+0x50f8c0`) draws the latest update and presents it with the interval of the render mode (frame +0xbb84, set at `main+0x50f3d0`): 0 = 1080p docked, 2 = 720p handheld, 3 / 5 their two-view variants (interval 2); 1 and 4 are 720p / 1080p modes at interval 1 that nothing selects. Measured on the Mac before the cuts: 59.2 updates and 29.8 presents a second.
- **Current:** the per-mode interval table (int at `main+0xd13de0`, `{2,1,2,2,1,2}`, read by `main+0x4b0420` for the present and by the 6-tick fade counter in `main+0x4b7530`; float at `main+0xd13e40`, the per-frame scroll step in `main+0x4c2b30`) becomes 1 for every mode, so every update is shown, the game runs at the same speed and the render thread's own steps stay at the same speed. Resolutions stay per mode.
- **Current:** the render thread also forced interval 2 while either of two flags was set: `main+0x21c9a03`, a full-screen image drawn before the frame (`main+0x4b4908`: the opening movie), and frame +0xc55f / +0xc560 (from the object at app +0x40: the character creation screen after the movie, and so on). In the engine's own 60 fps modes this traded 60 fps for resolution in those scenes (`main+0x50f7d0` drops 720p mode 1 to 1080p mode 0). A cut at `main+0x50fb7c` (`orr w8,w9,w8` → `mov w8,wzr`) ignores both, so those scenes run at 60 too; the movie still lasts 210.6 s (6,318 frames at 30 fps), as in the original.
- **Current:** the frame limiter (`main+0x5be430`: yield, read the tick, until the target has passed; x8 = time passed and x23 = target, in 100 ns) yielded and polled the clock, which on the host spun its first 2 ms on a performance core and then overslept by up to 250 µs: frames took 16.8 ms (59.5 fps, a repeated display frame every 2 s) and CPU energy went from 0.16 W at 30 fps to 1.5 W on the Mac (228,000 yields a frame on an iPhone). The yield becomes `nn::os::SleepThread` of 96% of the time left (PLT `main+0xafc300`), and the loop goes back to the limiter's own tick read at `main+0x5be44c`; the frames take 16.68 ms.
- **Current:** the render thread waits for the next update with KTGL's event wait (`main+0xaa90e0`, timeout in µs: × 1000 → `nn::os::TimedWaitEvent`) and passed 1/fps × 1000 = 16, one frame in ms, so it polled every 16 µs (42,000–53,000 wakeups a second on the Mac at 60 fps, against 1,200 at 30 fps, where it waited in the present instead). The timeout becomes 1000/fps × 1000 = 16,666 µs. The other four callers wait without a timeout.
- **Current: both pacing cuts stay per-build, and neither stands in for a host fault** (checked 2026-10-26, host at `7753b7b`, headless boot to the title on the Mac, 100 s per run, each cut undone in the rewritten `main`). Without the limiter cut the general yield back-off (`thr_yield`) holds 59.7–59.8 fps (frame p50 16.73 ms against 16.68) at 0.9–1.2 W of CPU energy against 0.14–0.42 W: about 9,700 plain yields a frame before the back-off starts at 2 ms. Without the timeout cut: 53,000–55,000 wakeups a second against 1,450, and 0.7–1.0 W. The 16 µs is the game's own: the render thread passes (1/fps) × 1000 = 16 to `main+0xaa90e0`, which multiplies it by 1000 into the `TimeSpan` (nanoseconds) of `nn::os::TimedWaitEvent`, so a frame in milliseconds goes where microseconds are taken. Only a wait that knows its deadline can sleep until it, and only the game's code holds it: of the 206 modules on hand, the limiter (`main+0x5be4b4`) is the only loop whose only calls are `YieldThread` and `GetSystemTick`, so a rewriter pass for such loops would match this one site.
- Prologue battle on the Mac with the cuts: 59.9 fps, 55% of a core and 1.8 W of CPU energy (the original at 30 fps: 44% and 1.7 W).
- Before the limiter cut, the host's yield back-off alone took its CPU time on the Mac from 78% to 19% of a core, same frame times ([development.md, Power](../../../docs/development.md#power)).

## Memory

- About 3.5 GB resident and 2.8 GB of Metal allocations on the Mac; the game itself makes a 1.5 GB GPU-only memory pool.

## Rendering

Game-specific findings. The general host changes they led to are listed under [Found on this game](#found-on-this-game).

- **Resolved: frames stopped after about 15 s; a game thread spun in its command-memory callback.** KTGL gives its main command buffer one 16 MB chunk of a ring and, in its callback (`main+0x9ca960`), takes chunk start + `GetCommandMemoryUsed` as the position the GPU has reached; the ring allocator (`main+0x9cc120`) waited for fences that had all signalled. NVN counts used bytes from the last `AddCommandMemory`; the host counted from `BeginRecording`, so the position stayed at one frame's size and the ring never freed.
- **Resolved: `FATAL submit re-entered` in the first frames.** The game initializes two NVN queues and submits on both from two threads; submissions are serialized now.
- **Resolved: menus showed plain bars or only the cursor; the screen was black.** The scene binds a 12,544-entry texture pool and a 3,856-entry sampler pool, the 2D UI's command buffer binds 4,096-entry ones, and both fill IDs from 256. With one ID table the scene's default 8×8 white texture overwrote the UI's textures: the title menu's labels drew as plain bars, other menus were empty except for the cursor. The pools are now kept apart. The scene registers each draw's textures into fresh IDs of a ring (8,729 and up a few seconds in), which used to read as empty slots and left the screen black.
- **Resolved: black silhouettes, then flat tan characters.** The character shaders read Half2 UVs through integer-declared inputs: Metal refused the pipeline (`Vertex attribute m_36(5) of type uint4 cannot be read using MTLAttributeFormatHalf2`), so the characters' shading draws were dropped, and reading the raw bytes as integers gave denormals (UVs collapsed onto one texel). They are now converted to float32 bits.
- **Current:** depth clamp is replayed (the game turns `SetDepthClamp` on; it used to be ignored).
- **Resolved: the ground under close-up battle cameras was flat steel blue.** The terrain's vertex shader (`56778feb1cfdee7e`) reads its grid position from an `RGBA16I` attribute: x and z the grid cell, y the height, which spans the whole int16 range; so does the side pass that draws the river surface into targets of its own (`db725c85ca060403`). The input was declared `uint4`, and the host swapped `Short4` for `UShort4`, which widens with zeros: every negative height read as 32,768–65,535, and those vertices went far up out of view. Where the ground was missing nothing wrote the depth (reversed-Z, cleared to 0), and the lighting takes depth 0 for sky and paints the fog colour. The G-buffer colour targets are never cleared, so they kept an old image there: cobblestones of a mesh under the terrain, which from the map camera showed through the gaps like a road.
  - How it was traced, since two steps misled: a `HILITE` colour over the gap was stale too (the never-cleared albedo), and the end-of-frame depth showed 0 there. Recomputing every draw's clip positions from `CBDUMP` data showed that no draw of the shader reached the gap. Then the side pass's layer, which showed only scattered tiles of the river, led to the vertex format. Ruled out on the way: depth clamp (Metal's clip and clamp modes gave the same depth), the pass without a depth attachment (142595 is a screen-space decal pass: it samples the depth and writes albedo and normals), polygon offset (none), dropped draws.
- **Light and decal lists:** 321 shaders read them through the instruction Maxwell also uses for 1D textures, and they used to read 0. The decal pass (`417a523053457352`, slots 16 and 18) and the lighting (`094a2d11253f126c`, `f54830a75e07e14a`, slots 18–20) read them from texture buffers declared `texture2d` (`texture type mismatch: shader declares 2, texture is 9`). Since row wrap they read the real lists: slot 18 holds 1,052,640 elements, 65 rows. On the Mac prologue route the torches and burning houses now light the battle map and the close-up ground (they lit nothing before), and the blood decals take the scene's light instead of showing as dark blue patches. Its Half2 / RGBA16I vertex attributes go through shadow vertex inputs instead of the CPU conversion.
- **Resolved: the first loading screen took 78 s on the iPhone (5 s on the Mac).** KTGL's loader uploads each resource with a copy on the queue followed by `nvnQueueFinish`, hundreds of times per loading screen (effect banks, face models), and that call waited for the display to take every queued frame: one to two display frames per call. The windowless Mac runs present at once, so they never showed it; `DM_WINDOW=1` does (one step took 20 s instead of 1.7 s). `nvnQueueFinish` now waits only for the GPU. Windowed Mac run after the fix: first load 6.1 s, frame pacing unchanged. Found on the device with `Documents/env.txt` holding `DM_WATCH=50` (thread table at 50 s: only the yield-polling main thread used CPU) and `DM_FSLOG=<absolute path>` (the reads came in bursts, with stalls of 4 / 24 / 13 s right after the effect banks).

## Audio

- **Resolved: no sound; red `audren: command 1 in the command list is malformed (magic 0 size 0)`.** SDK 6's DSP command header is +0x04 size (s16), +0x06 command ID (s16), +0x10 enabled, where SDK 8–20 have +0x04 enabled, +0x05 ID, +0x06 size (`CommandListProcessor::Process`, `sdk+0xaaa00`); its DeviceSink is ID 0x10.

## Services

- **Resolved: the first loading screen never ended; `DM_FS=1` showed `save:/system` "not found" and no `fs mount save`.** In SDK 6 `account::OpenPreselectedUser` returns a Result, and the game skips its save mount when it is not 0 (`main+0x53b7e0`). The host's hook returned void and left the handle pointer in w0; it returns 0 now.
- DLC: the Mac needs `DM_AOC_DIR=<NSP folder>/010055D009F78000_build/aoc`, or a "DLC deleted" dialog blocks loading a save.

## Files and shaders

- RomFS: the LINKDATA archive (`DATA0.bin` is an index of 32-byte entries `{u64 offset, u64 length, u64 stored length, u64 compressed}` into `DATA1.bin`, 6.9 GB), loose overrides in `patch1`–`patch4` (listed by `patchN/INFO0.bin`, 0x120-byte entries ending in a `rom:/patchN/…` path), WebM movies in `nx/movie` and KTSL sound banks (`*.ktsl2stbin`). Compressed entries and `*.gz` files are Koei Tecmo's chunked zlib (`kt_gz`; [01](../../../docs/agents/01-extract.md#compressed-romfs-content)).
- Shaders: 1,364. `shader_split.py` (`kt_shaders`) reads the SPKG packages (G2S containers, as in Age of Calamity) in `DATA1.bin` (132 entries) and `patch*/nx/shader`, a `_SPK4100` post-effect package without G2S (`DATA1.bin` entry 549), map and model archives carrying packages of their own, and the BNSH and GLSLC outputs in `main`. `DATA1.bin` is split into 64 scan jobs by entry range (`linkdata_jobs`).
- Its GLSLC (output version 0x1d.2) writes 0x848 / 0x850-byte control sections and compute code with no header, which the reader used to drop: 24 shaders before, 1,355 after. A few outputs of version 0x1d.0 keep something else at control +0x20 (odd values such as 0x136b), so a control length must be a multiple of 4; `glslc_section` later took 8 more of them ([GLSLC outputs read through their section headers](../../../docs/reference/shaders.md#glslc-outputs-read-through-their-section-headers)).
- **Shaders compiled at run time:** KTGL compiles 9 built-in GLSL shaders at boot with the game's own GLSLC (`glslcCompile` from `main+0x9c89f8`; the sources are strings in `main`): 2D primitives, text, the loading quill. No game file holds their output, so they are captured on the Mac: the game runs with `DM_GLSLC_CAPTURE=<version dir>/glslc_capture` (`hle_glslc.c`), and `shader_split.py` reads that folder. The outputs are identical on every run. The NSP script does this by itself in its `GLSLC capture` stage, so the NSPs and prod.keys are enough: it builds the Mac host into `<id>_build/tools/` and boots the game until the 9 files are in `<id>_build/extract/v327680/glslc_capture`. Without the folder the split tables would differ from the profile, and the script would stop at its table check. By hand, run the game for a few seconds with that variable (step 5 of the build by hand).

## Mac routes

- A fresh save needs `DM_KBD=<name>` for the name entry, and its confirm dialogs default to No.
- From a new game into the prologue battle (frame-based at 60 fps, about 7 minutes): `DM_KEYS="f720:A*185/90,f2400:Plus*35/438,f17400:Left~0.3*120/500,f17480:A*120/500,f17560:Left~0.3*120/500,f17640:A*120/500,f18020:Plus*100/562"`; the game offers a save right before the battle. The repeating Left, A, Left, A reaches Yes whether a confirm dialog is open or not (A, Left, A loops once it starts on an open dialog: A answers No, Left changes the selection, A reopens it). A presses during the opening movie pause it, so the scenes after it start a few seconds later from run to run.
- To a first fight, from a save right before the prologue battle, with `DM_AOC_DIR` set: `DM_KEYS` with A every 90 frames from `f720` to `f2610`, then `f2700:B,f2800:B,f2900:A,f3000:Up,f3060:Up,f3120:Up,f3180:Up,f3240:A,f3340:A,f3440:A,f3540:A,f3640:A,f3740:A` closes the movement tutorial, moves Byleth up to the thief and attacks: the combat close-up runs at frames 3,400–3,700.

## Found on this game

General host changes first needed here; each applies to every game:

- **Resolved in the sanitizer audit:** [Unaligned command headers during layout probing](../../../docs/reference/audio.md#unaligned-command-headers-during-layout-probing), reproduced with the SDK 6 command-list fixture.
- **Submissions serialized across queues** (`g_sub_mu`) and **`GetCommandMemoryUsed` counted from the last `AddCommandMemory`**: [Submissions from two queues on two threads](../../../docs/reference/nvn-objects.md#submissions-from-two-queues-on-two-threads), [What `GetCommandMemoryUsed` counts](../../../docs/reference/nvn-objects.md#what-getcommandmemoryused-counts).
- **SDK 6:** `account::OpenPreselectedUser` returning a Result, SDK 6's DSP command header (`capture` in `hle_audren.c`) and **run-time GLSLC capture** (`DM_GLSLC_CAPTURE`, `hle_glslc.c`): [SDK 6's `account::OpenPreselectedUser` returns a Result](../../../docs/reference/services.md#sdk-6s-accountopenpreselecteduser-returns-a-result), [SDK 6: the DSP command header layout](../../../docs/reference/audio.md#sdk-6-the-dsp-command-header-layout), [Built-in shaders compiled with the game's own GLSLC](../../../docs/reference/shaders.md#built-in-shaders-compiled-with-the-games-own-glslc); the NSP script's capture stage: [development.md](../../../docs/development.md#the-packing-script-in-detail).
- **Overlapping texture / sampler pool IDs** kept apart (`bind_handles`, pool base in `cmd_t`), **integer vertex inputs fed a float-type format** (`vconv_bind`) and **other-signedness integer formats** (`vconv_int`, since replaced by shadow vertex inputs), **depth clamp** (`OP_DEPTH_CLAMP`), **`nvnQueueFinish` waits only for the GPU** (`mtl_finish`): [Texture and sampler pools with overlapping IDs](../../../docs/reference/shaders.md#texture-and-sampler-pools-with-overlapping-ids), [CPU conversion of vertex attributes the input type cannot take](../../../docs/reference/rendering.md#cpu-conversion-of-vertex-attributes-the-input-type-cannot-take), [Depth clamp](../../../docs/reference/rendering.md#depth-clamp), [`nvnQueueFinish` waits for the GPU, not the display](../../../docs/reference/nvn-objects.md#nvnqueuefinish-waits-for-the-gpu-not-the-display).
- **Row wrap** and **shadow vertex inputs**: [Buffer textures read by 1D integer fetches (row wrap)](../../../docs/reference/rendering.md#buffer-textures-read-by-1d-integer-fetches-row-wrap), [Shadow vertex inputs](../../../docs/reference/rendering.md#shadow-vertex-inputs).
- **GLSLC 0x848 / 0x850 controls and headerless compute code** (`glslc_shaders`) and the **SPKG / LINKDATA reader** (`kt_shaders`): [SPKG packages in KTGL LINKDATA](../../../docs/reference/shaders.md#spkg-packages-in-ktgl-linkdata).
- **Yield back-off** and the **yield-progress rule** (`thr_yield`; this game's limiter is a poll that keeps its registers): [development.md, Power](../../../docs/development.md#power).

## Open

- **To confirm:** 64-pixel rectangles in the albedo where decals land; skipping the decal pass removed them. They did not appear on the Mac prologue route before or after row wrap (no 64-pixel structure in the close-up frames), so the scene where they were seen still has to be checked. The player-facing README lists "small square patches on the ground where decals land" as a known issue.
- Deferred binds are not remapped to the pool base (red line).
- `game.tsv` still describes the game as "In bring-up" and the memory layout as Link's Awakening's, not measured; the development notes call SDK 6.5.0 "the oldest so far", though DARK SOULS: REMASTERED (SDK 4.5.0) and Mario Kart 8 Deluxe (SDK 4.4.0) are older.
- The capture boot's time: the development notes give "about 10 s in all" for the NSP script's stage and "about a second after launch" for the 9 files.


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

