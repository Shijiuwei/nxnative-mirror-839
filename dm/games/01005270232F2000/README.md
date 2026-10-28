# Octopath Traveler 0 (`01005270232F2000`)

Update 1.0.5 (v327680) over the base game, an XCI; SDK 19.3.13. Square Enix, Unreal Engine 5 (the paged global distance field shaders and the LowLevelTasks task system) on its mobile renderer (`SF_GLSL_SWITCH_MOBILE` shaders), NVN, CRIWARE sound and movies. Modules `main`, `subsdk0` (the movie library, which `main` imports from) and `sdk`. Plays on the iPhone at 60 fps (30 on the Switch) by the host's writes of Unreal's frame limits.

Status: brought up on the Mac through character creation and the story opening into free play (26 minutes from a fresh save with no red line, 2.74 GB resident). Playable on an iPhone: 30 fps through character creation, name entry and the story opening, no audio underruns, three saves, a footprint of about 2.35 GB after 3.4 minutes; it plays at 60 fps on an iPhone too (`fps_target 60`). Battles are not tested yet.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `fps_target 60` | [`game.tsv`](game.tsv), `hle_fpslock.c` | the host writes 60 into `t.MaxFPS` and 1 into `rhi.SyncInterval`; see [Frame rate](#frame-rate) |
| FMallocBinned3 constructor cut: `main+0x12e40` `bl 0x9f480` → `nop` | [`patches.py`](patches.py), `main` `9fe60a21…` | the constructor reserves 37 GB of address space; the host's allocator runs Unreal's heap (`hle_uemalloc.c`); see [Memory](#memory) |
| `game_args -SwitchVirtualAddressSpaceSize=2048` | `game.tsv` | caps the engine region Unreal's `nninitStartup` reserves (62 GB by default) at 2 GB |
| Node area cut 1 GB → 62 MB (`sdk+0x1423fc` / `+0x142468`) | `dm/tools/sdk_patches.py`, `sdk` build `7bba185a…` | Unreal's region then starts 62 MB into the alias region, and the guest block stays small enough for an iPhone |
| 3.75 GB guest block: `mem_reserve 0xF0000000`, `mem_alias 0xC8000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | alias region = node area + up to 1 GB of alignment (regions of 1 GB and more are 1 GB aligned) + the 2 GB region; 256 MB stacks, 128 MB heap |
| `tex_dyn_slots on` | `game.tsv` | 61 particle vertex shaders pick one of several c2 table slots at run time (separate texture slots 0-10, samplers 0-4); see [Shaders](#shaders) |

No `pad_layout` key: the iPhone gamepad starts in the 2D layout ([README](../../../README.md)).

## Frame rate

- **Current:** 60 fps (30 on the Switch). The game's config caps Unreal at 30: `t.MaxFPS=30` in `[ConsoleVariables]` of `DefaultEngine.ini` (and `FrameRateLimit=30` in `SwitchGameUserSettings.ini`, which sets it again), and `rhi.syncinterval=2` in `SwitchDeviceProfiles.ini`, the present interval 2 the game asks for. The shipping build reads no console variables from the command line (no `-ExecCmds`, `-DPCVars` or `-ini:`), so the host writes 60 and 1 into the two console variables (`hle_fpslock.c`, profile key `fps_target 60`), whenever the game sets them back; the game then asks for present interval 1 itself.
- Unreal steps the game by the time that really passed: on the Mac, `DM_NVN_REGLUM` traces of the same input script from a fresh save at 30 and 60 fps (frame time p50 33.4 → 16.7 ms) put the 1.5 s fade to white 70 s in at the same real time with the same shape, and both runs reach the same screens at the same times.
- `DM_FPS_TARGET=30` (`Documents/env.txt` on the iPhone) keeps the game's own 30 and saves power.
- Unreal paces frames by the layer's frame number (`nn::vi::GetLatestFrameNumber`, `main+0x10140e8`); see [Services](#services).

## Memory

- **Resolved: `exit(1)` in `nninitStartup`** (`main+0x12e74`; the function is at `main+0x12d74`). Unreal's `nninitStartup` needs virtual address memory and reserves one address region for the engine with `AllocateAddressRegion`, 62 GB by default (`-SwitchVirtualAddressSpaceSize=` in MB, or in bytes above 4095; the other options it reads are `-SwitchMaxMemorySize=`, `-NVNCommandBufferGranularity=`, `-NVNControlBufferSize=`, `-SwitchMainThreadStackSize=`, `-NVNTempMemoryPageSize=`, `-ResX=` / `-ResY=` and `-SwitchDisableVirtualAllocator`, which also exits). It reads them from the game arguments, which the host used to set only after `nninitStartup`; they are set before it now, as on hardware, and the profile key `game_args` supplies them on iOS.
- **Current: the allocator is the host's.** Unreal's allocator, an FMallocBinned3, is constructed in place at `main+0x68022c8` (`main+0x12e40` → constructor `main+0x9f480`) and reserves 37 GB of the region (`main+0x9f748`: 74 pools of 512 MB, pool index = offset >> 29; the null read in the out-of-memory report at `main+0x1005824` is the symptom when the region is smaller). The pools are uneven: at character creation the largest holds 68 MB, the 74 together 732 MB (measured with `DM_ALIAS_WINDOWS=512`, [05, Memory growth](../../../docs/agents/05-debugging.md#memory-growth)), so no smaller window fits a phone's ~5 GB (an iPhone has about 5 GB of address space without the extended virtual addressing entitlement). The constructor call is cut (`patches.py`) and the host puts its own allocator's vtable in the object (`hle_uemalloc.c`: Malloc, Realloc, Free and the size queries on a host malloc zone, the trivial slots kept). Every allocation goes through it: FMemory reaches the object through `FPlatformMemory::BaseAllocator` (`main+0x132b4`), and the thread-local storage allocator `nninitStartup` registers (`main+0x12e80` / `main+0x12ec0`) through its vtable. The Unreal heap logs its peak as `[mem] Unreal heap … in use`.
- **Current: a 2 GB region in a 3.75 GB block.** With the host allocator the region holds only what the engine maps itself (709 MB at character creation, spread over 1 GB), so it is capped at 2 GB (`game_args`); SDK 19's AddressRegionManager node area is cut to 62 MB as Engage's (`sdk+0x1423fc` / `+0x142468`), so the guest block is 3.75 GB. The Unreal heap lives outside the guest block.
- 15 minutes into the story on the Mac: region mapped up to 1.26 GB of the alias region, Unreal heap 0.75 GB, 2.8 GB resident, 1.84 GB of GPU allocations.

## Shaders

- 10,324 unique shaders (1,457 vertex, 6,346 pixel, 2,521 compute) in the pak's two shader code archives, `ShaderArchive-Global-SF_GLSL_SWITCH_MOBILE-SP_GLSL_SWITCH_MOBILE.ushaderbytecode` (1,219 shaders) and `ShaderArchive-Octopath_Traveler0_Chunk0-…` (9,377), both stored in the pak. The profile groups them as `global` and `game` (`shader_groups`).
- Version 2 archives: shader map hashes, shader hashes, shader map entries, shader entries `{u64 offset, u32 size, u32 raw size, u8 frequency}` (17 bytes packed; frequency 0 vertex, 3 pixel, 5 compute), preload entries and shader indices, then the code. Every shader is Oodle-compressed (Mermaid, block header `8C 0A`). The repository has no Oodle decoder, so the game's own `OodleLZ_Decompress` (statically linked into `main`, `main+0x4f77e10`, found as the one function that takes the address of the library's own name string, its profiler scope) decodes them in the Mac host: `DM_OODLE_JOB` (`dm/host/hle_oodle.c`), the NSP script's `decode_ue_shaders`, as for any game with Unreal paks; about 10 s for all of them (see [Open](#open)).
- A decoded shader is an `LSLG` record: `LSLG`, a stage tag (`SV`, `SP`, `SC`), a header with the source name (`/Engine/Private/…usf:MainPS`), then `u32` code length, the code area, `u32` control length (0x880), the control section and a 24- or 29-byte tail, all at unaligned offsets (`uepak.lslg`). The host keys came out exactly: 37 of 37 shaders a first run reported missing were in the two archives.
- Vertex and pixel shaders combine per draw from Unreal's pipeline state, so every vertex shader is paired with the single-stage cover and made to link with any pixel shader (`shader_split.py` `separable_pairs`, `dmsh_build.py` `vs_pad`).
- Bindings: separate textures and samplers, as Engage's (`0x168` / `0x568`).
- **Current: 61 vertex shaders pick a texture slot at run time** (particles reading their data from buffer textures). The shader reads one of several c2 texture-table words, chosen by branches, and ORs it with a sampler-table word; the translator cannot trace that to one slot and falls back to Ryujinx's bindless arrays (`array<texture_buffer<float>, 32> vp_tb_cb0_40000000`, `array<sampler, 32> vp_s_cb0_40000000`), which used not to load. Every integer read of c2 in those shaders is a table slot (texture slots 0-10, samplers 0-4), so `tex_dyn_slots` is right here. Host only: the chain version does not change.
- **Current: 2,084 shaders do atomics on buffer images** (Unreal's GPU particles), native texture atomics on the buffer texture since chain `0ce4b5e8aefb` ([Image atomics](../../../docs/reference/shaders.md#image-atomics)). On the fresh-save route (2026-10-27, Mac) three compute shaders run them (`766bc02c3b4c76d3`, `65327534de18f480`, `df56bcd022bf1e73`, image slot 2, texture ID 2049: an 8 KB R32Uint buffer texture), with no red line to frame 14,400 under Metal API validation (no validation error).
- **Current: a buffer texture at pool offset `0x168`**, read by the particle shaders, is created there (it used to be left a dummy, below Metal's reported 16-byte alignment).
- **Resolved: 16 compute shaders with atomics on a 3D `r32ui` image did not build** (SPIRV-Cross's MSL 2.3 emulation indexed its pointer with the `int3` coordinate: `array subscript is not an integer`; GitHub issue #290 listed them beside 2,077 returning atomics, which build since `2e655b7`: `40b643b` builds 10,308 of 10,324). With native image atomics all 10,324 build (chain `0ce4b5e8aefb`, 2026-10-27). They are Unreal's paged global distance field (`GlobalDistanceFieldCompositeObjects.usf:CompositeObjectsIntoPagesCS`), which the mobile renderer does not dispatch (none in any run).

## Files

- Everything is in one pak, `Octopath_Traveler0/Content/Paks/pakchunk0-Switch.pak` (1.5 GB, 159,757 files; the profile's `romfs_marker`), next to loose CRIWARE `.awb` sound banks and one `.usm` movie.
- A version 11 pak: its footer is the last 0xdd bytes (encryption key GUID, encrypted-index flag, magic `0x5A6F12E1`, version, index offset and size, SHA-1, five compression method names), its magic 0xc8 bytes from the end. The index is AES-256 (ECB) encrypted: mount point, entry count, path hash seed, path hash index, full directory index (a second encrypted block) and the encoded entries (one bit-packed u32 per file).
- The key is in `main`: the one 32-byte window of its read-only or data segment that decrypts the index start to the mount point `../../../` (`main+0x57c7f70`, found among 116,287 candidates in under a second). Never print it.
- Most files are Oodle-compressed in 64 KB blocks; the two shader code archives are stored. `dm/gfx/tools/uepak.py` reads all of this (AES through macOS CommonCrypto) and lists a pak (`uepak.py list <pak> <main NSO> <out tsv>`). Details: [01, Compressed RomFS content](../../../docs/agents/01-extract.md#compressed-romfs-content).

## Rewriter

- **Resolved: a `jump table <addr>` assertion on a byte table whose `add` is far from the `br`** (`main+0x37f9bc8`: `adr x18,#L; ldrb w0,[x17,x15]; add x18,x18,x0,lsl #2`, then 17 register moves before `br x18`). The byte / halfword search looked 16 instructions back; `jump_bytes_far` looks 32 back. One of the 12 `br x18` sites in this `main`.
- `main` has a 71 MB stretch without unwind entries; it got through the rewrite on a quieter Mac before functions were analysed in pieces, and comes out the same with 1 MB pieces and `b` within 256 KB.
- The spin-wait search takes 75 → 78 loops in `main` since the store rule (a parser state that leaves on the next pass, `main+0x4ebfa54`, among them), and the movie library's thread loop is wider in `subsdk0`.

## Services

- **Resolved: an abort after `UNIMPL` on the weakly imported `thread_local` init function `_ZTHN13LowLevelTasks5FTask10ActiveTaskE`.** The loader bound every undefined import to a named stub, so the address test the compiler emits for a weak symbol saw a function. Such a symbol is now 0 (`loader.c`, `undefined weak … bound to 0`).
- **Resolved: SIGSEGV in `android::Surface::getQueuedFrameNumber` under `nn::vi::GetLatestFrameNumber`** (Unreal paces frames by the layer's frame number, `main+0x10140e8`): the SDK read the vi service's surface, which the host never made. `hle_vi.c` answers with the frames presented so far; `ListFrameHistory` returns no entries.
- **Resolved: SIGSEGV at address 0 in `nn::fs::GetSaveDataSizeMax`, about 13 minutes into a new game** (Unreal's save system asks before its first save): it now reports 256 MB of data and journal, as `GetSaveDataSize` does (`hle_fs.c`). Listing `main`'s undefined imports that no host hook names, before they crashed, gave `fs::GetCacheStorageSize` and `fs::GetFreeSpaceSize` (the same answers), `account::PushOpenUsers` (nothing to hand over), `web::ShowOfflineHtmlPage` (cut, as `ShowWebPage`) and `oe::ExecuteProgram` (logged red: the host runs one program). The rest were pure computations (the swkbd presets, the web argument constructors, `time::GetDayOfWeek`, the audio resampler) or socket calls behind a `Socket()` that already fails offline.
- **Saves:** four saves in a 26-minute run, and a later boot continues from them.
- **NVN calls new with Unreal:** `nvnTexturePoolGetSize` / `nvnSamplerPoolGetSize` and their memory pool / offset getters (an abort in device setup when they returned 0), and `nvnCommandBufferFenceSync`, which Unreal's RHI records in its command buffers (`OP_FENCE_SYNC`: the replay fences the sync with the submission that reaches it).

## Input

- **Keyboard:** the character's name and favourite dish come from the system keyboard (`DM_KBD` on the Mac, the input box on the iPhone).
- **Resolved: the sound stuttered and the input box lagged while the player typed a name** (on an iPhone: one 4.7 s frame, 5,035 wakeups a second instead of about 1,800, no audio underruns). The host froze the guest clock while `swkbd::ShowKeyboard` was up; Unreal calls it off its game thread, so the game kept running, and every thread that waits on the clock spun while its audio mixing starved. On the Switch the applet blocks only the caller while the clock runs; the host now does the same (`hle_swkbd.c`). The iOS input box froze the clock a second time around its own wait (`front_ios.m`), so the first fix changed nothing on the phone (4,977 ms and 2,577 ms frames, 5,181 wakeups a second). On the Mac, `DM_KBD=<text> DM_TEST_SWKBD_HOLD_MS=8000` holds the keyboard up: frozen, 8.8 fps and 7,936 wakeups a second in that window; running, 30 fps and 1,788.
- The game reads a mouse and no touch screen ([Romance of the Three Kingdoms 8 Remake's note](../0100C2A01A730000/README.md)).

## Mac routes

- A fresh save reaches the story with `DM_KBD=Lyra DM_KEYS="20:Plus,24:A*27/4,150:Plus,154:A*62/3,340:Right*12/3,341.5:A*12/3,380:Plus*5/3,396:A*20/3,470:A,474:Up,476:A,480:A*140/3"` (initial settings, the prologue, character creation with three belongings, "Begin Lyra's journey?").

## Found on this game

General host changes first needed here; each applies to every game (or every Unreal game):

- **Unreal's allocator replaced by the host's** (`hle_uemalloc.c`, a rewriter cut of the constructor), explained in [Memory](#memory) above; the later Unreal builds and their vtable layouts: [Unreal's FMallocBinned3 replaced by the host allocator](../../../docs/reference/memory.md#unreals-fmallocbinned3-replaced-by-the-host-allocator).
- **Unreal's frame limits written in game memory** (`hle_fpslock.c`, `fps_target`), explained in [Frame rate](#frame-rate) above.
- **Game arguments set before `nninitStartup`** (`main.c`, profile key `game_args`): [Unreal's engine address region and its options](../../../docs/reference/memory.md#unreals-engine-address-region-and-its-options).
- **Unreal pak reading and Oodle-compressed shaders decoded by the game's own Oodle** (`uepak.py`, `DM_OODLE_JOB`, `decode_ue_shaders`): [01, Compressed RomFS content](../../../docs/agents/01-extract.md#compressed-romfs-content), [Unreal Engine 5 shader code archives (Oodle)](../../../docs/reference/shaders.md#unreal-engine-5-shader-code-archives-oodle).
- **Texture slots picked at run time** (`tex_dyn_slots`, `NVN_RB_TEXD` / `NVN_RB_SAMPD`), **image atomics, now native texture atomics** (`dmsh_atomic.py`, `img_usage`, `dummy_img_atomic`), **buffer textures below 16 bytes**: [Texture slots picked at run time](../../../docs/reference/shaders.md#texture-slots-picked-at-run-time), [Image atomics](../../../docs/reference/shaders.md#image-atomics), [Buffer textures at offsets below 16 bytes](../../../docs/reference/rendering.md#buffer-textures-at-offsets-below-16-bytes).
- **Rewriter:** byte tables far from their `br` (`jump_bytes_far`): [Byte tables whose add is far from the br](../../../docs/reference/rewriter.md#byte-tables-whose-add-is-far-from-the-br). Not first found here but met here: long functions analysed in pieces (`pieces`, [Long functions analysed in pieces](../../../docs/reference/rewriter.md#long-functions-analysed-in-pieces)) and spin waits that store (`patch_nso.py` "Spin waits", [development.md, Power](../../../docs/development.md#power)).
- **Undefined weak imports bound to 0** (`loader.c`), **`vi::GetLatestFrameNumber`** (`hle_vi.c`), **`fs::GetSaveDataSizeMax`** (`hle_fs.c`), **NVN pool getters and `nvnCommandBufferFenceSync`** (`OP_FENCE_SYNC`), **the guest clock left running behind the software keyboard** (`hle_swkbd.c`, `front_ios.m`, `guest_clock_freeze_for`): [Undefined weak imports bind to 0](../../../docs/reference/services.md#undefined-weak-imports-bind-to-0), [Layer frame numbers](../../../docs/reference/services.md#layer-frame-numbers), [Save data size queries](../../../docs/reference/services.md#save-data-size-queries), [Pool size getters, texture address and command memory size](../../../docs/reference/nvn-objects.md#pool-size-getters-texture-address-and-command-memory-size), [Fence syncs recorded in command buffers](../../../docs/reference/nvn-objects.md#fence-syncs-recorded-in-command-buffers), [The software keyboard blocks only its caller](../../../docs/reference/services.md#the-software-keyboard-blocks-only-its-caller).
- **`DM_ALIAS_WINDOWS`**, the alias-region window census: [05, Memory growth](../../../docs/agents/05-debugging.md#memory-growth).

## Open

- Battles are not tested yet.
- Shader decode counts and times differ between sources: `docs/development.md` says 10,324 shaders decoded in about 10 s; `dm/games/README.md` says 10,596 shaders in 2 s (10,596 is the two archives' total, 1,219 + 9,377; 10,324 the unique count).
- Touches as a mouse would suit Romance of the Three Kingdoms 8 Remake, but this game, Tears of the Kingdom and Unicorn Overlord also read a mouse and no touch screen, so such a rule needs testing here first.


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

