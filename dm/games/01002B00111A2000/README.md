# Hyrule Warriors: Age of Calamity (`01002B00111A2000`)

Update 1.3.0 (v327680) over the base game, with four DLC packages (one is only a licence). SDK 9.6.0. Koei Tecmo's KTGL engine on NVN. Modules `main`, `subsdk0` (Nintendo's movie library, NintendoSDK_movie, as in Metroid Dread) and `sdk`. Plays on the iPhone; 60 fps (30 on the Switch) through the host's frame step. Started from the NSP script's provisional profile.

## What it needs

| Setting | Where | Why |
|---|---|---|
| Frame step, `fps_target 60` | [`patches.py`](patches.py) `FRAME_STEP`, [`game.tsv`](game.tsv) | three stubs run the game's fixed-step updates at the cap; see [Frame rate](#frame-rate) |
| `fps_hold off` | `game.tsv` | the updates step by the cap whatever the hold, so holding late frames only paces the screen: each frame is shown as soon as it is ready (the 60fps IPA ran battles at 45–57) |
| `mem_used 0x4800000` (72 MB) | `game.tsv` | the three modules take 66.6 MB, more than the default 64 MB, so `SetHeapSize` failed; see [Memory](#memory) |
| 4 GB layout: `mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | the NPDM declares no system resource (META +0x14 = 0), so the game uses `SetHeapSize`. Link's Awakening's partition, not measured for this game |
| `tex_flush_upload on` | `game.tsv` | movie frames are CPU-written and published with `nvnMemoryPoolFlushMappedRange`; without the upload the intro movie stayed black under its "+ Skip" prompt |
| `vconv_range on` | `game.tsv` | KTGL binds whole mesh pools; the CPU vertex conversion runs only over the vertices each draw reads; see [Battles](#battles) |
| `pad_layout full` | `game.tsv` | the 3D gamepad layout: the camera is on the right stick |

## Frame rate

- The game steps a fixed amount per update, from three 30 fps values:
  - a KIDS task on `KIDSTaskScheduler` threads (`main+0x166d980`, once per update) waits until 1 / fps has passed since the last update (fps at time +0x30: 30 from a frame-rate object, `main+0x166daa4`; `csinc` at `main+0x166db80`) and then steps the game clock (system +0x3348, the step at +0x3350) by that object's step (x23 +0x34: 1/30 s times the game's timescale, a second time at +0x44, fps 30 at +0x48), not by the time that passed (the measured path, `main+0x166dca8`, runs only without the object);
  - the battle steps 1 / the game's frame rate per update: a block at `main+0x269e150` {fps, interval, fps, float fps} = {30, 2, 30, 30.0}, read by the scene update's caller (`main+0x8c3610`: step = 1.0 / fps) and its jobs (32 reads; the game's own writers at `main+0x8a35e0` / `main+0xfc1618`, a run-time interval request at +0x1d58, never ran in a probe);
  - the event scripts step by the KTGL device's frame rate (device +0x32d0), 59.94 / the present interval in its config (2). nnMain builds the device config on its stack and inits the device once (`main+0x8d94c4` → `main+0x17c34f0`): {1920, 1080, 3, 9, 5, interval 2, 9, …}. The device keeps the interval (`main+0x17c4ba8` → device +0x2aa4, passed to every present at `main+0x17ced78`, which sets it on the window when it changes) and its frame rate, 59.94 / interval (`main+0x17c37c8` → device +0x32d0).
- **Resolved:** the first 60 fps cuts (interval 1, a 1/60 s wait) were checked only on cutscenes, which follow the device: `DM_PROBE` showed the game clock at 2.00 × real time, and battles ran twice as fast on an iPhone. Fixing the clock alone was not enough: in a 3 s stick hold Link still covered twice the ground of the 30 fps original, and the enemy waves came early, until a `sample` profile of a battle led to the frame-rate block.
- **Current:** the game's updates are not held by the presents (the task kept 60 updates a second while the host held frames 3 vsyncs), so three frame-step stubs (`FRAME_STEP`) run the updates at the cap and the host's frame step paces only the presents:
  - at `main+0x166dab4` (where both fps paths join, before the update's tasks and the wait) the task's fps and the frame-rate block become the cap (60, or 30 at a 30 cap), the update's vsyncs (60 / cap) are marked (state +0x28) and the device frame rate becomes 59.94 / those vsyncs;
  - at `main+0x166dc60` the object's steps are scaled by its fps × vsyncs per update / 60, and the vsyncs are counted;
  - at `main+0x8d94b0` nnMain's device config gets interval 1 above a 30 fps cap (the engine's own 60 fps mode), else the game's 2.
- Every step stays per update, as on the Switch, so an update rate below the cap slows them all together. The host shows each frame as soon as it is ready (`fps_hold off`); the updates keep their speed whatever is shown.
- `DM_FPS_TARGET=30` runs updates and presents at 30; `DM_FPS_TARGET=0` is the Switch original (interval 2, the stubs' original paths; the device frame rate follows the game's fps, 29.97 at 30, and the steps and the block stay the game's).
- Checked on the Mac: the forced 30 and the original put Link on the same spot after the same stick hold, and the opening scenes at frame 2N at 60 match the original's frame N (same subtitle, same poses).
- x16, x17, s16–s18 and the flags are dead at the stub sites; the function uses no x16 / x17.

## Memory

- **Resolved: `nnMain` returned right after `nv::InitializeGraphics`, with no error.** The game's `nninitStartup` (`main+0x8ad318`) sizes its heap as total − used memory (`QueryMemoryInfo`), calls `SetMemoryHeapSize` and hands the block to its own malloc. The host reported 64 MB used beside the heap, but `SetHeapSize` checks a request against the real size of the loaded modules (66.6 MB, 68 MB rounded), so `SetHeapSize(pool − 64 MB)` failed with ResultLimitReached, `AllocateMemoryBlock` returned 0x1003, malloc was never set up, the 866 MB graphics heap allocation came back null (`main+0x8aeb4c`) and `nnMain` returned. Found by probing `DM_SVC=1`'s `svc 0x1` request and the `AllocateMemoryBlock` result. `mem_used` 72 MB (the image plus stacks) gives a heap of 0xc8c00000 (3212 MB), in which the game's fifteen sub-heaps (3098 MB) fit.

## Audio

- **Resolved: the main thread spun at start-up in a resource wait** (a `YieldThread` loop, no crash, right after `OpenAudioRenderer`; nnMain's resource wait at `main+0x8d9bc8`). Koei Tecmo's sound engine (KTSL) failed its setup quietly. The init (`main+0x8d9610`) hands it three heaps cut from one block: 15 MB engine memory, 2 MB and 142 MB sound data (sizes built by `mov x8,#0xf00000; movk x8,#0x20,lsl #48` and `mov w9,#0x8e00000`; constructor `main+0x12647a0`, heaps created at `main+0x1a2ba58`). In the 15 MB heap the setup allocated the renderer config and local server work buffer (0x760000, including a 0x40a9c8 command buffer) and one block per player. The fourth player block (`main+0x1a32f70`, size 0x1b5860) came back null, setup released its memory pools (`RequestAttachMemoryPool` then `Detach` with no update between) and returned −51, and the first sound archive (ASRS) never loaded.
- How it was found: probe the pump the wait calls (`main+0x8e1c0`: first missing stage bit → the type's stage handler), the handler's dependency list (an incomplete entry of type `0xbbd39f2d`, ASRS), then the memory pool states (`MemoryPoolInfo` +0x10: 1 new, 3 detached, 4 request attach, 5 attached, 6 unused) and the engine allocator's arguments and results.
- **Measured, 2026-10-26:** with both heap cuts restored to the game's original instructions, `DM_PROBE=main:1a32f74` reproduced the null fourth allocation. The unchanged renderer parameter is 48 kHz, 240 samples, 132 mix buffers, 19 submixes, 138 voices, 1 sink, 16 effects, 0 performance frames, 85 splitters, 1800 splitter destinations and revision `REV8` (`0x38564552`). The SDK 9.6 local `AudioRenderSystem::GetWorkBufferSize` (`sdk+0x160320`) returns **0x760000 for DSP/auto, CPU/auto and CPU/manual**; the public CPU/manual query returns the same. The internal parameter was packed as the public wrapper (`sdk+0x1533a0`) does: device / mode bytes +0x22 / +0x23, revision +0x30. Comparing these local modes alone did not explain the shortage.
- **Resolved root cause: the host exposed the local SDK server's larger command representation as the DSP service's requirement.** The installed Horizon audio sysmodule (`0100000000000014`, NSO build `3a75b4a62dad189b2c8b30e4dedb4e15`, process renderer revision 15) was extracted into this game's work folder. Its service size method (`audio+0x47be4`) directly returns `AudioRenderSystem`'s calculation (`audio+0x47c14`). Isolated execution of the relocated machine code gives **0x61d000** for the same parameter, in all three modes. Its command-buffer calculation (`audio+0x58344`) returns **0x2c9a60**, while the SDK 9.6 local one (`sdk+0x166f10`) returns **0x40a9c8**. The command contribution accounts for 0x140f68 of the total **0x143000 excess**; other state sizes and page rounding account for the rest. The local routine sizes its compiled CPU structures even when passed DSP/auto, so that earlier comparison could not measure the service budget. Reporting the measured service size alone, with backend storage supplied separately, made all players fit: this explains why the original heap can work on Horizon.
- **Current fix:** the host computes the service's revision-dependent requirement and sizes the local backend separately ([service and backend work buffers](../../../docs/reference/audio.md#a-sound-engines-heap-too-small-for-its-renderer-and-players)). This game receives **0x61d000**, with the local SDK still given its required **0x760000**. The temporary one-page policy and both per-build heap cuts are removed. With the original 15 / 2 / 142 MB heaps, all five observed player allocations succeed (0x1220e8, three of 0x1b5860, then 0x84c80), the resource wait returns at `main+0x8d9be0`, and the renderer produces non-silent PCM. This is a windowless audio check without rebuilt shader libraries; iOS and battles have not been retested for this change. Independent fixture cases and 360 seeded parameter sets across REV1–REV15 match executed Horizon code; the tests also cover the public size/open/close contract and buffer lifetime. Firmware is used only for the investigation, never by the host.
- The multi-stream hardware Opus decoder is the SDK's software one (`HardwareOpusMultiStreamDecoder`, see [Services](#services)).

## Rendering

- **Resolved: draws after a dispatch were dropped, so the intro movie stayed black.** The game interleaves dispatches and draws; `BindProgram` with the compute bit replaced the one bound program, and the intro movie's NV12 → RGB draw was dropped (black under its "+ Skip" prompt). Compute programs now have a binding of their own in the replay.
- Images are registered on their own (`nvnTexturePoolRegisterImage`).
- Its GPU-driven effects are indirect quads, converted to indexed indirect arguments by a compute pass.
- **Movie frames:** the decoder's NV12 output (VP9) is copied by the game into a 1280×720 R8 and a 640×360 RG8 texture in a CPU-visible pool and published with `nvnMemoryPoolFlushMappedRange`, so the profile turns on `tex_flush_upload`. Buffer textures (a 1,048,576-texel R32F one) stay out of that path: they are views onto the pool buffer, and an upload would have sent them past Metal's 2D size limit.
- **Light and decal lists:** 16,438 shaders (98,970 bindings) read KTGL's light and decal lists (slots 16–21) from buffer textures with the instruction Maxwell also uses for 1D textures. The translation read them as 2D and the host bound a dummy, so every list read 0. They are now read as rows of 16,384 texels from a linear 2D texture over the buffer (row wrap). No shader both size-queries and fetches such a slot. A 15-minute Mac battle bound 15.7 million of them through 320 row textures, none failed; the first battle is darker and more saturated than with the zeros (mean luma 163 against 178 at the start of the battle), GPU time unchanged.
- **Open red line:** a 2D texture bound where three vertex shaders sample 3D (slots 8 and 9, `texture type mismatch … declares slot 8 as Metal type 7`) is still bound as is; it was hidden behind the earlier red line.

## Battles

- **Vertex conversion over the draw's vertices (`vconv_range`).** The soldiers' Half2 and Short2 attributes feed integer shader inputs, which the replay converted on the CPU (`vconv_bind` in `nvn_mtl.m`); without that conversion, reading the raw bytes as integers gave denormals and the soldiers' UVs collapsed onto one texel. KTGL binds whole mesh pools of up to ~590,000 vertices, so converting the bound ranges came to 4.4 million vertices per submission in the first battle: 24 of the 32 ms the replay took per frame on the Mac, 26 fps at a 30 or a 60 fps target. `vconv_range` converts only what each draw reads (219,000 vertices per submission, about twelve submissions a frame), and the half and 8 / 16-bit integer formats have conversion loops of their own. Keeping the conversions across frames did not pay: the soldiers' attributes change every frame. `DM_NVN_VCONV_CHECK=1` found no stale index range in 983,000 reuses.
- The visibility buffer ring was 4 deep, so the submit thread waited for the previous frame's GPU work; it now has as many buffers as command allocators (8).
- Then: the battle ran at 58.6 fps on the Mac (M4 Pro; frame p50 16.9 ms, replay CPU 10 ms, GPU 11 ms). On an iPhone the GPU decides (14–16 ms per frame at p50 in the first battle, 40–60 ms once the phone reaches thermal state 2): the frame step holds 30 there.
- **Shadow vertex inputs** have since replaced that CPU conversion, a quarter of the replay thread: 2,259 side libraries, 77 MB each for Mac and iOS; with more than 7 integer inputs the last ones get only the float shadow (169 of 8,201 inputs). Mac battle at 60 fps against the CPU conversion: 0 vertices converted instead of 545,000 per submission, submit + present 10.0 → 5.7 ms a frame, CPU 3.97 → 2.44 W, GPU unchanged, the same picture. `vconv_range` still serves the draws that keep the CPU conversion (transform feedback, `DM_NVN_VCONV_CPU=1`).
- **Resolved: battles after the first cutscene ran at 17 fps on the iPhone.** An iPhone log (2026-10-21, `fps_hold off`, thermal state 0) went from about 57 to 17 fps after the first battle's first cutscene: the same 1,000–1,400 draws a frame, but NVN calls from 82,000 to 287,000 a frame, GPU 15 → 51 ms and the replay 12 → 24 ms. The battle then runs more small compute dispatches (on the Mac about 280 → 900–1,500 a frame, each with a game barrier and about 17 texture binds), and two host costs grew with them:
  - every dispatch had a compute encoder of its own with full queue barriers on both sides (about 10 µs of GPU each on the Mac, GPU 9 → 18 ms a frame). Dispatches now share the copy encoder (about 1.3 µs each; GPU 7.3 → 8.1 ms across the cutscene);
  - KTGL registers a texture into the next ID of a ring before every bind (`RegisterTexture`, `GetTextureHandle` and `BindTexture` counts are equal: 8,000 a frame in battle, up to 28,000 after the cutscene). Only 1 in 6 registrations repeated what the ID held, so the per-ID view was rebuilt for half of all binds (`newTextureViewWithPixelFormat` took 45% of the replay thread). Views are now cached by texture serial.
  - Mac runs side by side, after the cutscene: GPU 18.0 → 8.1 ms and frame 35 → 22 ms with the dispatch change; the view cache took the replay before the cutscene from 14.3 to 10.8 ms (2,036 views made in 15 minutes, 211 million reused).
- The game builds 8,705 programs during its first frame (about 10 s on the Mac).

## Services

- **Resolved: null read in `oe::InitializeCopyrightFrameBuffer`** (right after the window setup): `InitializeCopyrightFrameBuffer`, `SetCopyrightImage` and `SetCopyrightVisibility` are host no-ops (`hle_oe.c`).
- **`HardwareOpusMultiStreamDecoder`:** the game builds it with the imported constructor and then calls it through its vtable; the constructor hook builds the SDK's software `OpusMultiStreamDecoder` in place (`hle_codec.c`). This game needed only the class and the four-argument work buffer size.
- **Other games' saves:** it asks whether Breath of the Wild's (and `0100A2C01320E000`'s) save exists for bonus items; the host says no.

## Files and shaders

- Everything sits in RDB archives: `asset/*.rdb` is the index (`_DRK0000` header, one `IDRK0000` record per file); `asset/*.rdb.bin*` and `asset/data/*.file` hold `IDRK` entries, zlib in 16 KB chunks (flag 0x100000). The DLC folders add `.rdb.bin3_0` files of the same layout ([01](../../../docs/agents/01-extract.md#compressed-romfs-content)). `romfs_marker` is `asset/MaterialEditor.rdb.bin`.
- Shaders: 40,726 GLSLC outputs (37,043 pixel, 3,440 vertex, 241 compute, 2 geometry): KTGL's G2S containers (`_S2G3000`, type KTID `0x7bcd279f`; 23,108 entries in `asset/system.rdb.bin*`, 2.3 GB inflated) and one `_DOK0000` container (type `0x20a6a0bb`, the engine's system resources: 151 bare GLSLC outputs, 21 of them drawn in the first battle), plus 117 in `main` (`shader_split.py` `idrk_shaders`). Many have 0x850-byte controls and the compute ones no code header. Mario's texture handle layout (`0x20 + 8 × slot`); pairs come from the single-stage cover (`separable_pairs`).
- `glslc_section` took 3 more outputs (GLSLC version 0x1d.0, whose control +0x20 holds odd values) that used to be dropped ([GLSLC outputs read through their section headers](../../../docs/reference/shaders.md#glslc-outputs-read-through-their-section-headers)).

## Mac routes

- With a save whose first-boot settings were made, `DM_KEYS="f300:A,f420:A,f540:A,f660:A,f780:A,f900:A,f1300:Plus,f1350:Plus"` confirms the settings, skips the intro movie and plays the opening into the first battle, The Battle of Hyrule Field (about frame 4000).

## Found on this game

General host changes first needed here; each applies to every game:

- **Rewriter:** byte jump tables indexed by x30 (`main+0x138216c`; capstone names x30 `lr` and x29 `fp`, mapped back): [Byte tables indexed by x30 (capstone register names)](../../../docs/reference/rewriter.md#byte-tables-indexed-by-x30-capstone-register-names).
- **`mem_used`** for games whose modules exceed 64 MB: [Heap sized from the application pool less used memory](../../../docs/reference/memory.md#heap-sized-from-the-application-pool-less-used-memory).
- **Copyright frame buffer no-ops** and the **hardware Opus multi-stream decoder** as the SDK's software one (`hle_oe.c`, `hle_codec.c`): [Copyright frame buffer overlay](../../../docs/reference/services.md#copyright-frame-buffer-overlay), [Hardware Opus decoders built on the SDK's software decoders](../../../docs/reference/audio.md#hardware-opus-decoders-built-on-the-sdks-software-decoders).
- **Host ownership of the local audio renderer's work buffer:** [A sound engine's heap too small for its renderer and players](../../../docs/reference/audio.md#a-sound-engines-heap-too-small-for-its-renderer-and-players).
- **Compute's own binding** (`S.cprog`, `g_pp_cprog`), **`nvnTexturePoolRegisterImage`**, **indirect draws and dispatches** (`ixc_run`, `DM_NVN_IXCLOG`), **dispatches in the copy encoder** (`dispatch_enc`, `DM_NVN_DISPATCH_OWN=1`) and **texture views cached by texture** (`vc_get`): [Compute programs bound apart from graphics programs](../../../docs/reference/rendering.md#compute-programs-bound-apart-from-graphics-programs), [Images registered on their own](../../../docs/reference/nvn-objects.md#images-registered-on-their-own), [Indirect draws and dispatches read by the GPU](../../../docs/reference/rendering.md#indirect-draws-and-dispatches-read-by-the-gpu), [Compute dispatches share the copy encoder](../../../docs/reference/rendering.md#compute-dispatches-share-the-copy-encoder), [Texture views cached by texture and view parameters](../../../docs/reference/rendering.md#texture-views-cached-by-texture-and-view-parameters).
- **CPU vertex conversion over the draw's vertices** (`vconv_range`, `vconv_bind`): [dm/games/README.md](../README.md).
- **Shadow vertex inputs** (`dmsh_vin.py`, `vin_ct` / `vin_on`) and **row wrap** for buffer textures read by 1D fetches (`fetch1d`, `tbuf_as_2d`, `nvn_tbuf_rows`): [Shadow vertex inputs](../../../docs/reference/rendering.md#shadow-vertex-inputs), [Buffer textures read by 1D integer fetches (row wrap)](../../../docs/reference/rendering.md#buffer-textures-read-by-1d-integer-fetches-row-wrap).
- **`fps_hold off`** for games whose updates do not wait for the presents: [dm/games/README.md](../README.md).

## Open

- The 3D texture mismatch on slots 8 and 9 above.
- Sources disagree on small counts: `game.tsv` says 118 shaders in `main`, the shader notes 117; `game.tsv` puts the dispatches after the first cutscene at 1,000 to 2,800 a frame and the iPhone battle's GPU at 10 → 41 ms, the development notes at 900–1,500 a frame (Mac) and 15 → 51 ms (iPhone log); `patches.py` speaks of "two 30 fps values" before describing the third.
- The memory layout is Link's Awakening's, not measured for this game (`game.tsv`).


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

