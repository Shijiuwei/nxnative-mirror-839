# Monster Hunter Rise (`0100B04011742000`)

Update 16.0.2 (v2228224, Sunbreak included) over the base game, SDK 11.4.3. Capcom's RE Engine on NVN. Modules `main` (314 MB, 125 MB of text), `subsdk0` (the movie library, NintendoSDK_movie 11.4.3) and `sdk`; `subsdk0` and `sdk` are the same builds as Metroid Dread's ([010093801237C000](../010093801237C000/README.md)) and 13 Sentinels' ([01003FC01670C000](../01003FC01670C000/README.md)). The base game is an XCI (cartridge dump), the update an NSP, and its 252 DLC NSPs are licences only; only the merged 16.0.2 copy is extracted. Plays on the iPhone; 60 fps (30 on the Switch) through the host's write of the engine's frame limit.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `fps_target 60` | [`game.tsv`](game.tsv), `hle_fpslock.c` (`main` build `c9a3dd77…`) | the host raises RE Engine's frame limit from 30 to 60; see [Frame rate](#frame-rate) |
| `present_interval display` | `game.tsv` | the game still asks for present interval 2; display mode ignores it. Both keys are needed for 60 |
| `tex_flush_upload on`, `tex_dirty_upload on` | `game.tsv` | RE Engine fills small textures from the CPU in place (the menu cursor's 8 × 8 BC1 texture: 32 bytes written into a CPU-visible pool, no packaged layout, no `WriteTexels`); without them the cursor highlight composited black on black and the menus looked frozen |
| `mem_used 0x14800000` (328 MB) | `game.tsv` | the loaded modules take 0x13fa4000 (`main` alone 314 MB of text + ro + data), so `SetHeapSize(pool − 64 MB − bss)` (0xc9000000) failed at boot (`[mem] SetHeapSize … + modules … > app pool`). With 328 MB the heap is 2.88 GB |
| 4 GB layout: `mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | the NPDM declares no system resource, so there is no virtual address memory: 256 MB alias, 256 MB stacks, 3.25 GB heap region |
| `pad_layout full` | `game.tsv` | a 3D action game: the camera on the right stick, the item bar on the D-pad |

## Frame rate

- **Current:** the frame loop (`main+0x6b106c8`) sleeps until `[obj+0x430] / [obj+0x434]` sixtieths of a second have passed (`[+0x430]` = 60) and steps the game by the time that really passed (+0x41c), so a higher limit runs the game faster in frames at the same speed. The game sets the limit, a float at `[main+0x1295b3f0]+0x434` (the pointer is a global in `main`'s bss), to 30 after boot and asks for present interval 2. The host writes the profile's `fps_target` there (`hle_fpslock.c`, the address from masagrator's FPSLocker config for this build, FPSLocker-Warehouse `patches/0100B04011742000/C9A3DD7702075ECD.yaml`), and `present_interval display` ignores the interval.
- `DM_FPS_TARGET=30` (`Documents/env.txt` on the iPhone) gives back 30 and saves power. The Mac `DM_KEYS` routes below count frames at 30, so run them with it.
- The spin-wait search takes 166 loops in `main` (90 before its store rule; 72 of the new ones are one inlined countdown, `ldr w9,[x21,#0x24]; subs w9,w9,#1; b.ge; str w22,[x21,#0x24]; ldr; cbz`, whose reset makes the next pass leave). The game runs alike on the Mac with both rewrites from a fresh save (180 s headless from the boot each, the same host: 60 fps, frame p95 about 17.5 ms, the same red lines, no hang). Rewriting `main` takes 342 s of CPU (507 s before the x18 scan decoded in blocks) ([development.md, Power](../../../docs/development.md#power)).

## Memory and the GPU heap

- **The GPU heap is one 64 GB sparse virtual pool** built from separately allocated 4 MB physical pools, each mapped at a virtual offset of its own (45 at boot, over the first 180 MB). **Resolved:** `[RED] MapVirtual: non-contiguous mapping (true sparse) not implemented`. The pool now gets an address range of its own at `MemoryPoolInitialize` (reserved, inaccessible: 64 GB on the Mac, at most 1 GB on iOS), and each mapping request `vm_remap`s the physical pages over its part of the range (`vpool_*` in `hle_nvn.c`).
- **Resolved: `CRASH sig=10` in `memmove` from `pkg_upload` → `nvn_deswizzle`, at an address 32 GB into the pool.** The streamed textures (flag 0x20, packaged layout) live 32 GB into the virtual pool; the game writes their tiles on the CPU into 4 MB physical pools and then maps them, with no GPU copy. NVN maps at 4 KB and the host aliases at its 16 KB page, so a request whose offsets differ modulo 16 KB cannot be aliased: the ASTC 6x6 textures' mip tail starts 0x81000 into the texture. The replay now reads a sparse pool's textures through a 4 KB page table (`vpt_*`, `nvn_sparse_copy`) and uploads one again only when a mapping in its range changed or the CPU wrote a physical page behind it (`nvn_sparse_since`). Before that the untracked alias counted as written on every look: 32 textures, 18 MB, were uploaded again every frame, and those with unaliased tails came out magenta. Requests beyond the reserved range (iOS reserves 1 GB: the buffers stay under 0x20100000 through the opening) are kept in the table only: textures can read them, the GPU cannot address them.

## Rendering

Game-specific findings. The general host changes they led to are listed under [Found on this game](#found-on-this-game).

- **Binding layout:** the game uses both texture tables at once: combined handles from `nvnCommandBufferBindTexture` at `0x20 + 8 × slot` and separate textures from `BindSeparateTexture(s)` at `0x168 + 8 × slot`, with separate samplers at 0x568, and binds slot 0 of both in the same frames (combined slot 0 beside separate slots 0–4). Separate binds are texture slots 128 + n. A census of 23,478 translated shaders' texture names: 33,813 combined (`fp_t_tcb_1E`), 29,342 separate textures with a constant sampler ID (handle type 3, `fp_t_cb65538_3041005C`: texture at c2 word 0x5C, sampler 0x104 ORed in), 8,265 separate texture + separate sampler (type 1, `fp_t_cb196610_1017815A`), 5,227 separate textures whose own handle carries the sampler (`fp_t_tcb_5A`), about 1,500 uniform-buffer handles with a constant sampler (`fp_t_cb65541_30410028`), and 121 fully dynamic handles (type 4, `fp_t2d_cb0_40000000`, not supported).
- **Subgroup operations in vertex shaders** (about 4% of them): RE Engine scalarizes uniform loads (ballot the active lanes, shuffle the first lane's value to all, take a fast path when all agree); `dmsh_build.py` rewrites them for a one-lane subgroup (`spv_single_lane`).
- **Resolved: the title screen showed only the UI.** The scene is GPU-driven: compute culling writes the draw records (stride 20) and a u32 count, then up to `max` draws (1–49 per call) are made from them with `nvnCommandBufferMultiDrawElementsIndirectCount` / `MultiDrawArraysIndirectCount`. Both were empty, and the G-buffer pass had no draws. The game draws the same records with two counts.
- **Resolved: Kamura Village frame rate on the iPhone.** The replay used to commit its work and wait for the GPU before the first such draw after each GPU write, then read the count on the CPU (`mdi_wait`): in Kamura Village that wait held the submitting thread 14 of its 20 ms a frame on an iPhone (36–54 fps). On the Mac the same scene's submit took 20 ms of wall time for 6.1 ms of work, and now 5.1 ms, all of it work (`mdx_run`).
- **Resolved: the hunter's innerwear (ASTC 6x6) came out magenta and the face black.** Textures in the sparse pool hold what the CPU wrote into physical pages before mapping them: the streamed ones and plain sampled ones with no packaged layout. The replay took the plain ones for GPU textures, so their Metal copies stayed 0. A sampled texture in a sparse pool is now uploaded like a packaged one, through the pool's page table.
- **Resolved: magenta trousers, sleeves and gloves of Hinoa and Minoto, worst in Kamura Village.** Sparse textures have a layout of their own (64 KB tiles); the game's files hold its streamed textures pre-swizzled that way, and it places each level at the offset the host reports. Laid out unpadded, their 1024×1024 ASTC 6×6 clothes (43 GOBs wide instead of 48, level 1 at 0x81000 instead of 0x90000) read 2,183 all-zero blocks in level 0 and 3,043 in level 1, which ASTC decodes as its error colour (`nvn_tex.c` `level_walk`). `DM_NVN_ZBLOCK` finds such textures.
- **Resolved: magenta on the title screen's clothes.** The translator drops TEX's LOD clamp and its residency predicate, so streamed textures were sampled at levels the game had not mapped yet, which read 0. A sparse texture is now bound as a view from its first fully mapped level (`sparse_view`).
- Bindless handles in uniform buffers written earlier in the same submission are read from the newest pending write (`pend_read`); the game queues more than the old limit of 256 writes per submission, after which every address counted as written.
- A whole-level clear of a texture made without render-target use (a 32x80 packaged one) upgrades it first (`tex_upgrade`).
- About 30 compute shaders read storage buffer addresses from the game's own uniform buffer (c3 + 0x38), marked `sbuf_ubo`.
- 218 pixel shaders keep a bindless handle in a loop variable (`c2[0x1A0] | 0x10500000`: separate texture slot 7, constant sampler 0x105); Ryujinx's bindless elimination gave up on the loop phi and the shader did not load (`no resolved slot number`) until the vendored translator followed the phis ([third_party/ryujinx/README.md](../../third_party/ryujinx/README.md)).
- Four size queries on image handles (`cp_t_tcb_48` / `_4A`) are of the kind `tex_slot_h` / `ImageTypesOf` handle ([A texture slot that shares its c2 word with an image slot](../../../docs/reference/shaders.md#a-texture-slot-that-shares-its-c2-word-with-an-image-slot)).
- Shadow vertex inputs: 16,781 side libraries, 344 MB on iOS (21% of its shader libraries, 1.5% of its IPA); its Mac and iPhone runs so far load none. Its 8- and 16-bit integer formats feeding float inputs (red line 99, about 60,000 vertices per submission on its Mac route into the opening) still go through the CPU, since shadows cover only integer inputs.
- **Geometry shaders** (53) do not translate: SPIRV-Cross writes `unknown main0_out main0()` for the geometry stage, and Metal has none.

## Movies

- H.264 (type 1) into the client's NVN texture (output 3 = RGBA, mode 2), with AAC 5.1 sound (type 4, PCM16). VideoToolbox decodes to BGRA for that mode (`mavc_set_rgba`), and AAC goes through AudioToolbox's converter with an MPEG-4 ES descriptor built from the AudioSpecificConfig in `csd-0`. The game ends a stream with time −1 and the whole 4 MB input buffer as its length: that packet carries no data (decoding it gave VideoToolbox −12909 and AudioToolbox −50). It also deletes its decoders (`D0`), so the host frees the object with the game's `operator delete` after its own state. Its movies never bring parameter sets to `Configure`.

## Services

- **SDK 11 calls** (one boot each; the import list against the hooks and the other SDK 11 games found most of them before the first boot): `oe::IsCpuOverclockEnabled` (a null read in `sdk+0x35258`, apm's session, the first call in `nnMain`), `album::Initialize` (abort in `sm::Initialize` under `capsrv::InitializeForApplication`; saving to the album now fails, `hle_offline.c`), `oe::GetDefaultDisplayResolution` and its change event, `oe::SendServerMaintenanceOverlayNotification`, `time::LoadTimeZoneRule` and the `ToCalendarTime` overload with a rule (the host's zone is used), `time::StandardNetworkSystemClock::IsAccuracySufficient` (offline: no), `hid::GetKeyboardStates`, `fs::GetSaveDataSize` / `ExtendSaveData` and the one-argument `fs::MountCacheStorage`. `nvnDeviceGetTexelFetchHandle` returns the texture ID with sampler 0.
- **Resolved: one guest thread at about 1000 ms/s on the iPhone only, the phone hot, CPU energy 5–7 W.** `MessageSuperviseThread` (priority 32) waits on `oe::GetNotificationMessageEvent()` with `os::WaitAny`, which does not clear an event, then pops notifications until the queue is empty. The host only ever signalled its notification event, so after the first focus change (iOS changes focus at launch; the Mac does not) it stayed set and the loop never blocked again. The event now follows the queue (`hle_oe.c` `mq_ev_sync`). Mac repro: `DM_OE_SCRIPT=focus=2@5,focus=1@6`: before 995 ms/s, CPU 109%, 6.97 W; after, the thread is idle and CPU energy is 0.05 W.
- Focus handling mode 1 (focus notifications are sent).

## Keyboard and names

- The hunter, Palamute and Palico are named through the inline keyboard (`DM_KBD` on the Mac, the input box on the iPhone). The inline keyboard keeps the guest clock frozen while it is up: the game does not block on hardware at all, and the host's input box blocks it.
- **Resolved: texture creation returned null right after the inline keyboard's `Initialize`, then a null read at `main+0x7713044`.** The game makes three textures of `swkbd::InlineKeyboard::GetWindowSize`, and the host answered 0 × 0 (its creation function returns null for a size of 0, `main+0x77112b8`). It is 1280 × 720 as on hardware now.
- **Resolved: the hunter's name box stayed empty, then stayed open with the name in it.** The text now arrives through the enter callback (read off the game's callbacks at `main+0x6fb3910` / `main+0x6fb3afc`), and the game finishes the entry only when `Calc` returns 1 (ready / hidden) after a non-zero value (`main+0x6fb3e6c`).
- **Resolved: `CRASH sig=10` at a heap address, called from `main+0x6bca160`, right after the name was entered.** The game masks the name through `nn::ngc::ProfanityFilter`'s vtable (slot 8, `MaskProfanityWordsInText(int *, char16_t *, uint32_t)`), and the constructor hook left the object with what the heap held. It now gets a host vtable (`ngc_vtable` in `hle_misc.c`). Nothing is masked or reported.

## Files and shaders

- The base game is a cartridge dump: the card header's magic `HEAD` sits at 0x100, and its root HFS0 holds the `update`, `logo`, `normal` and `secure` partitions; the game's NCAs are in `secure` (the 7 GB program NCA, control, legal information, HTML manual and the meta NCA), with their key in the key area, so there is no ticket ([01](../../../docs/agents/01-extract.md#nsp--nca--exefs--romfs)).
- Everything sits in KPKA PAK archives: `re_chunk_000.pak` plus `re_chunk_000.pak.patch_001.pak` … `patch_016` (19.6 GB in all; KPKA v4, zstd / deflate entries, file names only as murmur hashes), later patch PAKs overriding earlier ones by path hash ([01](../../../docs/agents/01-extract.md#compressed-romfs-content)). `romfs_marker` is `re_chunk_000.pak.patch_001.pak`.
- Shaders: 63,111 in 2,685 SDF files (30,233 vertex, 21,475 pixel, 11,271 compute, 53 geometry, 33 hull, 46 domain) and 102,345 vertex / pixel pairs, in 5 s with 12 jobs (`shader_split.py` `kpka_jobs` / `sdf_shaders`); an SDF entry's stages are one program; control sections 0x880 bytes, compute code without a header ([SDF definitions in RE Engine PAKs](../../../docs/reference/shaders.md#sdf-definitions-in-re-engine-paks)).

## Rewriter

- `main` has 125 MB of text: with the patch area 32 MB before the base, sites near the end of the text were more than 128 MB away (`B()` range assertion, then `patch area of … bytes does not fit in the … before the base`). The patch area now goes right before the base (`BIG_TEXT`); the first pass tries a 2.6 MB gap, and since the patch area is larger (3.6 MB, 34,322 sites) the rewriter runs itself again with exactly that size (`DM_PATCH_FRONT`).
- **Resolved: `KeyError` at `site_br` for a `br x18` byte jump table** (`main+0x6b346d0`): the function held a trap word (`0xe7ffdefe`, `main+0x6b34540`) before the table, where capstone stopped. Function disassembly now goes on after undecodable words (`disasm_fn`).
- Halfword jump tables padded only to their largest entry (`half_count`, found on Animal Crossing: New Horizons) changed 1 to 6 tables in this `main`.

## Mac routes

Run with `DM_FPS_TARGET=30`: the frame numbers count at 30.

- A fresh save (with the first-boot notices accepted) reaches character creation with `DM_KEYS="f4100:A~0.3,f6300:A~0.3,f7000:A~0.3,f7600:A~0.3"`.
- With `DM_KBD=Hunter`, `DM_KEYS="f4100:A~0.3,f6300:A~0.3,f7000:A~0.3,f7600:A~0.3,f8000:A~0.3,f8150:Down~0.25*6/40,f8450:A~0.3,f8650:A~0.3,f8900:A~0.3,f9100:A~0.3,f9500:A~0.3,f9800:Down~0.25*9/40,f10250:A~0.3,f10450:A~0.3,f10700:A~0.3,f10950:A~0.3,f11300:A~0.3,f11800:Down~0.25*10/40,f12250:A~0.3,f12450:A~0.3,f12700:A~0.3,f12950:A~0.3,f13300:A~0.3,f16000:A~0.3,f16300:A~0.3"` goes on through creation (hunter, Palamute, Palico, each named) and saves to slot 1. Holds under 0.25 s are not seen, and 0.5 s repeats in lists.
- From a save made that way, `f4100:A~0.3,f6300:A~0.3,f6800:A~0.3,f7100:A~0.3,f7500:A~0.3` continues into the opening.

## Found on this game

General host changes first needed here; each applies to every game:

- **Rewriter:** the patch area before the base for big text (`BIG_TEXT`, `DM_PATCH_FRONT`) and disassembly past undecodable words (`disasm_fn`): [Patch area right before the base for a long text](../../../docs/reference/rewriter.md#patch-area-right-before-the-base-for-a-long-text), [Disassembly goes on after undecodable words](../../../docs/reference/rewriter.md#disassembly-goes-on-after-undecodable-words).
- **`mem_used`** covering the modules: [Heap sized from the application pool less used memory](../../../docs/reference/memory.md#heap-sized-from-the-application-pool-less-used-memory).
- **Sparse virtual pools** (`vpool_*`, `vm_remap`) and their **4 KB page table** (`vpt_*`, `nvn_sparse_copy`, `nvn_sparse_since`): [Sparse virtual pools built from separate physical pools](../../../docs/reference/nvn-objects.md#sparse-virtual-pools-built-from-separate-physical-pools), [Textures read through a 4 KB page table of a sparse pool](../../../docs/reference/nvn-objects.md#textures-read-through-a-4-kb-page-table-of-a-sparse-pool).
- **Buffer alias visibility and ordering** (resolved in a synthetic regression using this game's sparse-pool mechanism; gameplay and iPhone not rerun): [Buffer copies through aliased Metal addresses](../../../docs/reference/rendering.md#buffer-copies-through-aliased-metal-addresses).
- **SDK 11 calls**, the **inline keyboard** (window size, `ikb_str_arg`, `Calc` results), **`ProfanityFilter` vtable** (`ngc_vtable`), **native-texture H.264 / AAC movies** (`mavc_set_rgba`), the **notification event following the queue** (`mq_ev_sync`, `DM_OE_SCRIPT`) and focus handling modes: [Listing a game's unhooked imports before they are met](../../../docs/reference/services.md#listing-a-games-unhooked-imports-before-they-are-met), [Inline keyboard window size, setters and callbacks](../../../docs/reference/services.md#inline-keyboard-window-size-setters-and-callbacks), [Inline keyboard text callbacks and `Calc` states](../../../docs/reference/services.md#inline-keyboard-text-callbacks-and-calc-states), [`nn::ngc::ProfanityFilter` vtable and by-name overloads](../../../docs/reference/services.md#nnngcprofanityfilter-vtable-and-by-name-overloads), [H.264 into an RGBA texture and AAC through AudioToolbox](../../../docs/reference/movies.md#h264-into-an-rgba-texture-and-aac-through-audiotoolbox), [The oe notification event and its queue](../../../docs/reference/threads.md#the-oe-notification-event-and-its-queue), [Focus handling mode decides whether focus changes are notified](../../../docs/reference/services.md#focus-handling-mode-decides-whether-focus-changes-are-notified).
- **Both texture tables at once** (separate binds at slots 128 + n, `samp_id`): [04, section 3](../../../docs/agents/04-graphics.md#3-find-the-binding-layout-by-census).
- **One-lane subgroups** (`spv_single_lane`), **indirect-count multi-draws** (`mdx_run`, `mdx_next`), **sampled textures in sparse pools uploaded through the page table**, **the sparse texture layout** (`level_walk`), **`sparse_view`**, **`pend_read`**, **`tex_upgrade`**, **`sbuf_ubo`**, **bindless handles through loop phis**: [Subgroup operations rewritten for a one-lane subgroup](../../../docs/reference/shaders.md#subgroup-operations-rewritten-for-a-one-lane-subgroup), [Indirect-count multi-draws](../../../docs/reference/rendering.md#indirect-count-multi-draws), [Sampled textures in a sparse pool uploaded through the page table](../../../docs/reference/nvn-objects.md#sampled-textures-in-a-sparse-pool-uploaded-through-the-page-table), [Sparse textures laid out in 64 KB tiles](../../../docs/reference/rendering.md#sparse-textures-laid-out-in-64-kb-tiles), [Sparse textures bound from their first mapped level](../../../docs/reference/rendering.md#sparse-textures-bound-from-their-first-mapped-level), [Handles in uniform buffers written earlier in the same submission](../../../docs/reference/shaders.md#handles-in-uniform-buffers-written-earlier-in-the-same-submission), [Render-target use added on demand](../../../docs/reference/rendering.md#render-target-use-added-on-demand), [Storage buffer addresses in the game's own uniform buffer](../../../docs/reference/shaders.md#storage-buffer-addresses-in-the-games-own-uniform-buffer), [Bindless handles through a loop phi](../../../docs/reference/shaders.md#bindless-handles-through-a-loop-phi).
- **Spin waits through stores** (`patch_nso.py` "Spin waits"; this game's `main` has the most new loops): [development.md, Power](../../../docs/development.md#power).

## Open

- Texture layout: NVN's automatic block height now truncates for every game (`pick_tile(ah * 3 / 16)`), which matters for this game's sparse textures; not re-checked here ([Automatic block height truncates](../../../docs/reference/rendering.md#automatic-block-height-truncates)).
- The 53 geometry shaders and the 121 fully dynamic handles (type 4) are not supported.
- 8- and 16-bit integer formats feeding float inputs still go through the CPU conversion (red line 99).


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

