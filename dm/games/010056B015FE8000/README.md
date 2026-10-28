# NieR:Automata The End of YoRHa Edition (`010056B015FE8000`)

Update 1.0.3 (v131072) over the base cartridge dump. PlatinumGames' engine, SDK 12.3.9, NVN, CRI CPK archives and Sofdec USM movies, Wwise audio through `AudioOut`. Modules `main`, `subsdk0` (NintendoSDK_movie) and `sdk`. Mac and iPhone gameplay have been tested; platform-specific performance and integration results are recorded below.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `mem_reserve 0x140000000`, `mem_alias 0x118000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | [`game.tsv`](game.tsv) | 5 GB guest block with a 4.375 GB alias region; the game maps its own memory there and never sets a heap during startup |
| `sys_resource 0x1000000` | `game.tsv` | the NPDM declares 16 MB of system resource, enabling virtual address memory |
| `pad_layout full` | `game.tsv` | movement and camera/aiming use both sticks |
| `tex_flush_upload on` | `game.tsv` | CPU-filled texture storage is published through mapped-range flushes; without uploads the title background is magenta |
| `tex_dirty_upload on` | `game.tsv` | tracks later CPU texture writes that the game does not publish with an NVN flush |
| `feedback_snap fd2c47b73f0e7e24` | `game.tsv` | `filter09.pso` samples the HDR attachment it writes through separate texture slot 128; a snapshot prevents NaNs spreading through bloom |
| `present_interval display` | `game.tsv` | replaces the game's interval-two presentation target with 60 Hz; the native clock already measures elapsed time |
| `fps_target 60` | `game.tsv`, `hle_fpslock.c` | initializes the verified build's engine interval at 60 fps; the launcher can select 30 fps independently of resolution policy |
| `gpu_timestamps segment` | `game.tsv` | the renderer subtracts two raw timestamp reports; one timestamp for the whole submission makes its measured GPU cost zero |
| Four dynamic-resolution threshold cuts | [`patches.py`](patches.py) | the verified 1.0.3 main build uses 22.5 / 16.005 / 15 ms thresholds for a 60 fps budget |

## Files and shaders

- 660 RomFS files, 10.84 GB: 30 CRI CPK archives (6.20 GB), 162 USM movies (3.52 GB), loose Wwise banks and streams, and Enlighten data.
- **Resolved: the shader split found zero shaders.** Five DAT archives in `data000.cpk` contain 1,371 size-delimited shader objects: 819 pixel, 517 vertex, 31 compute and four geometry objects, before deduplication. Every object has a 0x88-byte binding prefix, a length-prefixed NVN code area, then a length-prefixed 0x880-byte control section. All section bounds and control code lengths agree. They produce 1,225 distinct keys, including the initial runtime keys `f2fd6cfd0ff14a18`, `7855550732b71e48` and `62026f624061e259`. The general reader is linked below.
- The full RomFS scan finds 451 distinct vertex, 739 pixel, 31 compute and four geometry shaders, with 902 single-stage cover pairs.
- **Resolved: magenta title background.** The `render02` / `flush01` route captures frame 1,200 with identical inputs; enabling `tex_flush_upload` alone changes the magenta pixel share from 91.8% to zero and restores the cream title background. `TEXINFO` also finds populated guest texture storage without packaged layouts (for example a 1024×1024 format `0x7d` texture, 279,870 nonzero bytes of 704,512). That comparison had dirty-write tracking disabled; the current profile also tracks later unflushed writes.
- **Resolved: geometry shader compilation and binding failures.** All four geometry objects use viewport-mask output in their generated raster reader. The reader's scalar/array mismatch, missing geometry storage-buffer table and incorrect write-flag interpretation are general chain/host fixes linked below. All 1,225 shaders build for Mac and iOS after these changes.
- **Resolved: passthrough output bounds and raster emission.** All four geometry objects set SPH `GpPassthrough` with a raw output bound of one. The earlier metadata underallocated the raw scratch/index region and read the wrong stride, while native passthrough support left compute raster output absent. The shared fixes derive bounds/output topology from input, explicitly emit positions and generic attributes, and preserve viewport-zero culling. Native canaries, compaction and real-translator GPU tests cover every topology and the full capture map. Rebuilt flight scenes render the ship models and effects.
- **Resolved: green flight scene and dark rectangles.** HDR targets contained sparse red/blue NaNs, most on 16-pixel seams; bloom spread them across both channels. `filter09.pso` (`fd2c47b73f0e7e24`) samples the HDR attachment it also writes, through a separate texture slot. The host's snapshot path skipped separate slots before its general fix. `snap01` with the snapshot restores ship, cloud and exhaust colors; scans at frames 3,000, 3,060, 3,540 and 3,600 find no non-finite passes. Both default and active LUTs matched deswizzled guest storage byte-for-byte, ruling out their upload/layout. Enabling dirty-write tracking alone did not fix the green output; the separate-slot snapshot fix resolved it.

## Status on the Mac

- The title, menus, keyboard naming, a new Easy game with auto-chips, save creation and save loading pass. The opening formation cinematic and ocean combat render ships, clouds, lighting, exhaust, shots and HUD correctly. Left/right movement and firing respond to input (`snap01_control_*`, `fps60_opt_*`).
- Both logo movies decode through NintendoSDK_movie. The final 300-second run exits cleanly after the test's SIGTERM; Wwise output has zero underruns through 28,000 device callbacks. Resident memory peaks around 2,811 MB and Metal allocations around 2,995 MB, then settle below those peaks while the route streams more objects; its last report is 2,794 / 2,933 MB at frame 17,700.
- **Resolved: repeated geometry drains and sorting.** The original heavy flight route spent most sampled main-thread time waiting in `mdi_wait`, followed by sorting capture indices. The shared synchronization/bitset fix linked below removes those hotspots.
- **Resolved: serial geometry prefix scan.** A GPU profile of frame 4,600 in the 720p `fps60_seg` records 79 geometry sequences: 7.246 ms in their serial scans out of a 16.19 ms GPU span, with 521 encoders. The general parallel scan and combined compute encoder preserve ordered primitive compaction; native checks cover 129,032 groups and the translated geometry stages. The final 1080p route's frame 4,600 uses 284 encoders, 4.19 ms compute/copy and an 11.17 ms GPU span. The mode and camera timing differ, so these totals are supporting encoder evidence rather than an isolated same-frame speedup.
- Initial upstream build validation (before the integration below): both standard NSP build workflows pass. The unsigned `NieRAutomata.ipa` is 11,712,290,740 bytes (about 10.9 GiB), with all 1,225 Mac/iOS shader libraries on chain `20fe3d62b9f6` and 331 recorded prewarm entries. Its four cuts and all three embedded main sections match the tested rewrite; all 3,551 archive entries pass CRC verification. The normal Mac executable is `010056B015FE8000_build/tools/dmhost`.
- Work folder: `/Volumes/T7/nsp/NieR Automata The End of YoRHa Edition/010056B015FE8000-work`. `run/boot.py <name> <seconds> [KEY=VALUE ...]` runs the adjacent `010056B015FE8000_build/tools/dmhost` with bounded duration, control file `run/<name>.ctl`, logs and save data beside it.

## 60 fps

Four of main build `3c10ec35c5ef4c3686b4053afb0e58e7`'s eight verified cuts replace the threshold constants at `0x9bf434`, `0x9bf484`, `0x9bf4d4` and `0x9bf4f8`, with original-byte assertions, following the [FPSLocker 1.0.3 threshold sites](https://vrpj.wtpuscm.cn/anfang/tactic-054256.html). The other four initialize the 60 Hz interval and make the native resolution lower bounds follow the preset minimum. All addresses are build-gated; no function-matched cuts are enabled.

The native clock at `main+0x8fc7a8` converts elapsed milliseconds to 60 Hz ticks and clamps each step to [1, 2]. It needs no clock rewrite. Probe `main:8fc888@x19+b0` reads step, accumulator, scale and elapsed milliseconds. The paired title runs measure 33.34 ms / step 2 at the original target and 16.66 ms / step 1 at 60 Hz. Slow flight before the geometry fix clamps to step 2 and loses game time; raising the presentation target alone was insufficient.

The pre-integration M4 Pro run, `fps60_final`, uses the profile's default console mode (1920×1080), 60 Hz presentation, the four 1.0.3 cuts, the current shader chain, sound and native-clock/DR probes. No resolution reduction is required for this route:

| Window after launch | Average fps | Frame p50 / p95 | Average GPU time | Game seconds / wall second |
|---|---:|---:|---:|---:|
| 80–110 s, heavy opening formation | 59.79 | 16.67 / 17.99 ms | 10.15 ms | 1.011 |
| 120–150 s, flight transition | 60.03 | 16.67 / 17.23 ms | 10.99 ms | 1.010 |
| 170–200 s, free flight | 60.00 | 16.66 / 17.39 ms | 10.93 ms | 1.011 |
| 220–250 s, ocean combat | 59.92 | 16.66 / 17.26 ms | 10.06 ms | 1.009 |

These are measured windows, not a full-game or iPhone frame-rate guarantee. The frame rate stays near 60 with occasional longer frames; the native clock's minimum-one-tick clamp gives about 1% timing bias with host jitter in these flight windows.

Reproduce the save-load route with `DM_KEYS=40:A,45:A,50:A,52:Left,53:A,60:A`, `DM_PRESENT_LINK=0` and sound enabled. Headless timing runs unset `DM_WINDOW`. To compare the original presentation target, set `DM_PRESENT_INTERVAL=game`; `display` selects 60 Hz. `DM_OPMODE=0` selects the game's native 720p handheld mode; `1` selects 1080p console mode.

[`smoke.tsv`](smoke.tsv) uses `saves/opening-flight` inside the NSP folder: an Easy save frozen immediately before Start Game, after confirming the name, difficulty and auto-control chips. Each revision loads its own copy for the opening formation and ocean flight.

**Resolved: dynamic-resolution cost reads zero or 32 times too small.** The renderer's producer at `main+0x9ba6d4` waits for a ring fence, maps its counter buffer and subtracts timestamp fields at +8 and +0x18; its inline conversion publishes milliseconds through renderer +0x10c4. Submission timestamps gave zero. Segment timestamps need the shared raw-unit fix linked below: this consumer expects 614.4 MHz, while the host previously wrote 19.2 MHz. Both report paths and the nanosecond API now use the proper rate. The current preset policy is described under [iPhone resolution and frame-rate controls](#iphone-resolution-and-frame-rate-controls).

## Geometry optimization review

The objects in `data000.cpk/shader.dat` decode as follows:

| Geometry key | `IndexBuildIndependent_` suffix | ALD attribute addresses |
|---|---|---|
| `4a2b0df808307d63` | `bccw.gso` | `0xb0`, `0xb4`, `0xc0`, `0xd0` |
| `5dc6fcf19e502bf2` | `bcw.gso` | `0xb0`, `0xb4`, `0xc0`, `0xd0` |
| `137b54dc5462e5bb` | `xccw.gso` | `0xa0`, `0xa4`, `0xb0`, `0xc0` |
| `636a45f7fa065f43` | `xcw.gso` | `0xa0`, `0xa4`, `0xb0`, `0xc0` |

- **Verified from this dump:** all four objects match the committed keys and set `GpPassthrough`. Their ALD instructions load generic attributes only. None loads `PrimitiveID` (`0x60`) or a physical attribute address. The lean kernels retain four capture words each and the visible-index storage writes.
- **Compute-split follow-up:** the [shared primitive-ID correction](../../../docs/reference/shaders.md#split-geometry-primitive-ids) now retains the draw's primitive offset through compute chunks too. The synthetic Maxwell regression covers the ID behavior; these four shaders still do not load it.
- **General correctness fix found during review:** the mesh composer repeated guest primitive IDs across groups. These four shaders do not consume that ID; their invocation indices address the capture and generated output. The [general finding and GPU regression](../../../docs/reference/shaders.md#mesh-geometry-primitive-ids-restart-per-group) distinguish the synthetic failure from this game's verified behavior.
- **Integration:** the lean and mesh paths retain #336's dependency-aware batching, index cache, parallel compaction and upload fixes, reconciled with main through `d0b7c846` (the four-game gameplay comparison below used `3e77816c`; final shader equivalence and native checks cover the subsequent integration). The M1 measurements in PR #416 describe the original implementation; validation of the reconciled host is recorded below.

- **Rebuilt for this reconciliation:** all 1,225 shaders and 96 mesh pairs for Mac and iOS, for both global and Asian profiles in separate stamped directories, on chain `a78f966a7b38`. The native graphics suite (including 1,920 geometry checks), host suite, tool tests, translated geometry GPU tests and 178-source iPhone SDK compile check pass. A further 110 native checks cover restored geometry-feedback capture, pause/resume and raster discard. The original mesh ID mapping fails the strengthened GPU oracle at primitive 32 of a 33-triangle draw; the fix passes all 48 cases.
- **M6 Mac, full 220-second opening-flight comparison against `3e77816c`:** baseline/candidate frame rate 59.1/59.5 fps, frame p99 17.58/18.26 ms, peak footprint 2,746/2,022 MB. These are this route's measurements; the M1 performance figures from PR #416 remain author-reported. Both runs reach the route target without crashes, hangs or new red diagnostics; 902 pipeline bases prebuild without failures. At frame 12,000, the candidate's preceding 300 frames contain 11,334 mesh draws and no draws waiting for a mesh pipeline.
- **Screenshot review:** the raw harness flags the moving formation at 180 seconds (17 dHash bits, RGB difference 8.52). An earlier repeat of unchanged `df694e3` produced a larger RGB difference (16 bits, RGB 9.90). The reviewed images retain all three ships at different positions, with animated effects. This is classified as flight-motion variance; the raw warning is retained. The 100-second flight image does not trigger the comparison threshold.

## Found on this game

- [Size-delimited PlatinumGames shader objects](../../../docs/reference/shaders.md#size-delimited-platinumgames-shader-objects).
- [Mapped-range flushes taken as uploads](../../../docs/reference/nvn-objects.md#mapped-range-flushes-taken-as-uploads-tex_flush_upload).
- [Geometry reader viewport masks](../../../docs/reference/shaders.md#geometry-reader-viewport-masks).
- [Geometry-stage storage-buffer table](../../../docs/reference/shaders.md#geometry-stage-storage-buffer-table).
- [Storage-buffer write flag in translation metadata](../../../docs/reference/shaders.md#storage-buffer-write-flag-in-translation-metadata).
- [Passthrough geometry output bounds follow input topology](../../../docs/reference/shaders.md#passthrough-geometry-output-bounds-follow-input-topology).
- [Passthrough geometry kernels lose raster vertices](../../../docs/reference/shaders.md#passthrough-geometry-kernels-lose-raster-vertices).
- [Same-pass feedback read through a snapshot](../../../docs/reference/rendering.md#same-pass-feedback-read-through-a-snapshot).
- [Geometry capture waits for every indexed draw](../../../docs/reference/shaders.md#geometry-capture-waits-for-every-indexed-draw).
- [Geometry compaction serial prefix scan](../../../docs/reference/shaders.md#geometry-compaction-serial-prefix-scan).
- [Geometry jobs preserve transform-feedback replay](../../../docs/reference/shaders.md#geometry-jobs-preserve-transform-feedback-replay), found while checking the integration against main's existing feedback path; the native replay fixture covers the restored behavior.
- [Raw NVN timestamp units](../../../docs/reference/frame-rate.md#raw-nvn-timestamp-units).
- [GPU timestamps per submission segment](../../../docs/reference/frame-rate.md#gpu-timestamps-per-submission-segment).
- [Controller haptic engines without advanced players](../../../docs/reference/services.md#controller-haptic-engines-without-advanced-players): the game's rumble on a GameSir Super Nova (iPhone).
- [Regression rewrites register copied profiles twice](../../../docs/reference/rewriter.md#regression-rewrites-register-copied-profiles-twice).
- [Mesh geometry primitive IDs restart per group](../../../docs/reference/shaders.md#mesh-geometry-primitive-ids-restart-per-group): the general defect was exposed by a synthetic shader during this review.
- [Passthrough geometry shaders drawn by the vertex shader](../../../docs/reference/shaders.md#passthrough-geometry-shaders-drawn-by-the-vertex-shader): the lean kernels and dependency-aware batching; the pairs in `geo_mesh.tsv` (shadow-map vertex shaders `zmap_vs_*` with the four `IndexBuildIndependent_*` kernels) as mesh draws.

## Open

- Conditional rendering is still drawn unconditionally (`SetRenderEnableConditional` is the route's only `[RED]` warning); no visible fault was found from it in this route.
- iPhone performance remains scene-dependent; a constant 60 FPS across the full game has not been established. The current reconciliation's regression coverage is recorded in the geometry optimization review above.


## Rendering and performance follow-up

The following checkpoints and screenshots were reported with PR #336 before the current reconciliation. They document the original Mac and iPhone tests; validation of the reconciled host is recorded in the geometry optimization review above.

### Factory combat capture and 9S texture follow-up

9S's magenta face/hair was traced to ASTC images already present in CPU-uncached storage before texture registration. Upstream now has that first-use fill; this integration keeps it and adds the original mip/color/reuse regression test. Dirty uploads are also enabled for textures rewritten later without a mapped flush. See [initial uncached storage](../../../docs/reference/nvn-objects.md#storage-of-a-cpu-uncached-pool-at-first-use).

### CPU preparation after query batching

Independent geometry draws share capture and compute encoders across compatible state and query boundaries. Exact-byte index caching reuses primitive maps and capture IDs; CPU edits and GPU writes invalidate reuse. Unknown dependencies and sparse aliases stop reordering. Newer upstream hazard proofs, unaligned-stream preparation and large-draw splitting remain. See [geometry runs](../../../docs/reference/rendering.md#geometry-shader-draws-encoded-a-run-at-a-time), [dependencies](../../../docs/reference/shaders.md#geometry-runs-retain-external-buffer-dependencies) and [query boundaries](../../../docs/reference/shaders.md#geometry-runs-across-occlusion-query-boundaries).

### Upload stability follow-up

Upload slots retain bounded primary and overflow staging buffers. The integration keeps upstream's batch-memory budget and retirement on upload completion. The warmup seed adds 841 unique shader/pipeline records from later Mac checkpoints and iPhone factory combat. See [staging reuse](../../../docs/reference/frame-rate.md#reusing-upload-staging-after-a-slot-completes).

### Library checkpoint: geometry GPU cost

Small geometry outputs use fused compaction; large outputs use ordered block scans. Capture initializes only missing words. Masked depth draws omit color attachments only when attachment bounds and state prove that safe. Upstream's full 32-attribute map, scratch limits and overflow guards remain. See [capture initialization](../../../docs/reference/shaders.md#geometry-capture-initializes-only-missing-words).

### Small light-probe array uploads

The bounded RGBA16F array path uploads every face in one compute dispatch per mip through an integer view, preserving the stored bits. Unsupported formats and sparse pools keep staging. Temporary views survive upload completion; targets participate in dropped-upload recovery. See [small array uploads](../../../docs/reference/frame-rate.md#small-array-texture-uploads-without-per-face-staging).

### Library shadow sequence and geometry ordering

Geometry encoder boundaries may submit prepared work through the existing segment policy. Queries, events, dependent storage writes and depth coverage retain their order. The independent unwritten-attachment store proof is enabled; broader lookahead/pass merging remains at its upstream default. See [early geometry submission](../../../docs/reference/frame-rate.md#geometry-boundaries-submit-prepared-work-early) and [unwritten stores](../../../docs/reference/rendering.md#unwritten-attachment-stores-without-lookahead).

### iPhone resolution and frame-rate controls

Frame rate defaults to 60 FPS and Dynamic resolution defaults off. Both remain editable, along with console mode, the FPS counter and performance HUD. Verified 1.0.3 rewrites expose the resolution control through capability metadata: on selects a 0.7 preset minimum; off holds full resolution. The native scale/UV path remains, independently of the 30/60 FPS choice. Guest-main QoS follows configured policy. Opt-in player builds discard saved diagnostic overrides while retaining launcher choices.

The earlier direct-encoding experiment is not forced. The iPhone retest below is noticeably better after integration, but remains below a constant 60 FPS across the full game.

### Mac: M4 Pro

The author of [PR #336](https://daks.wtpuscm.cn/yunying/news-670953.html) reports a complete playthrough at stable 60 FPS on that revision. The 2B/Adam capture below shows 60.00 FPS, 8.48 ms GPU time and a 16.67 ms frame interval on macOS 27.0. That screenshot is one scene; the full-game result comes from that reported playthrough, not from the still image. The HUD's 2560×1440 value is the presentation size, not proof of the game's internal render resolution.

![NieR on M4 Pro: 2B facing Adam, HUD at 60.00 FPS](../../../docs/images/nier-mac-m4-pro-adam-gameplay.png)

### iPhone: 17 Pro Max

The author of PR #336 reports improved performance after retesting that revision on iPhone 17 Pro Max. It reaches 60 FPS in some areas, but it is not a constant 60 FPS through the whole game. The 2B/9S Bunker capture below shows 59.99 FPS, 11.23 ms GPU time and a 16.67 ms frame interval on iOS 27.0. This is a scene-specific example, not a full-game 60 FPS claim.

![NieR on iPhone 17 Pro Max: 2B and 9S in the Bunker, HUD at 59.99 FPS](../../../docs/images/nier-iphone17-pro-max-bunker-gameplay.png)


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

