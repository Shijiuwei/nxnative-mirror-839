# DRAGON QUEST VII Reimagined (`0100A9D01C446000`)

Update 1.2.0 (v131072) over the base game, SDK 19.3.1. Square Enix / HEXA DRIVE, Unreal Engine 4.27 (PhysX 3.4, so UE4; the source paths say `UEngine/4.27-source`): IoStore containers beside one pak, the desktop renderer with forward shading, NVN, CRIWARE sound (a 6-channel AudioOut) and Sofdec movies (VP9 through the SDK's movie decoder, mode 1, NV12). Modules `main`, `subsdk0` (the movie module, which Sofdec feeds its VP9 video to) and `sdk`. Plays on the iPhone at 30 fps; 60 fps (30 on the Switch) on the Mac by the host's writes of Unreal's frame limits.

Status: brought up on the Mac: the logos and opening movie with sound, the title, a new adventure log (name entry, settings), the opening cutscene (the well, the night scene with Maribel by the boat) into play at Pilchard Bay with the controls tutorial, and a saved log continued, at 30 fps with saves; the opening cutscene into the first free play (the hero's house) again at 60 fps (`fps_target 60`). No red line at run time apart from the geometry shaders below; about 2.6 GB resident (1.1 GB of Metal allocations) in the cutscene. Playable on an iPhone at 30 fps; 60 fps on an iPhone is not tested yet.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `fps_target 60` | [`game.tsv`](game.tsv), `hle_fpslock.c` | the host writes 60 into `t.MaxFPS` and 1 into `rhi.SyncInterval`; see [Frame rate](#frame-rate) |
| FMallocBinned3 constructor cut: `main+0x32758dc` `bl 0x291714` → `nop` | [`patches.py`](patches.py), `main` `f7ec03e4…` | its pools take 37 GB of the engine region; see [Memory](#memory) |
| Observation gate `OBS_UE_MALLOC` at `main+0x32758e4` (orig `mov x0, x19`) | `patches.py` | puts the host allocator's vtable into the new object in x19 (`hle_uemalloc.c` layout `UE4_27`) |
| `game_args -SwitchVirtualAddressSpaceSize=2048` | `game.tsv` | a 2 GB engine region instead of 62 GB |
| Node area cut 1 GB → 62 MB (`sdk+0x1423fc` / `+0x142468`) | `dm/tools/sdk_patches.py`, `sdk` build `145dde94…` | the same AddressRegionManager constructor as Octopath Traveler 0's SDK 19.3.13 build, at the same offsets |
| 3.75 GB guest block: `mem_reserve 0xF0000000`, `mem_alias 0xC8000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | alias region = node area + up to 1 GB of alignment (regions of 1 GB and more are 1 GB aligned) + the 2 GB region; 256 MB stacks, 384 MB heap region |
| `sys_resource 0x5000000` | `game.tsv` | the NPDM declares 80 MB of system resource |
| Integer output row `f555b53f62ec5196` (`fout` 0 `uint`) | [`shader_facts_manual.tsv`](shader_facts_manual.tsv) | `ShaderArchive-Global-SF_GLSL_SWITCH` shader 688 (pixel) writes integers it computes per pixel, bit-cast into its output, to an `R8Uint` target (seen at the camera settings' preview in the new-game menus); Unreal's `GLSL_SWITCH` shaders carry no output types, and Metal refuses a float output on an integer attachment |
| `pad_layout full` | `game.tsv` | a 3D game with a free camera on the right stick. It reads the controller only, so it stays docked (1920x1080) |

## Frame rate

- **Current:** 60 fps (30 on the Switch). The game's config caps Unreal at 30 twice: `FrameRateLimit=30` in `DefaultGameUserSettings.ini` and `SwitchGameUserSettings.ini`, which `UEngine::SetMaxFPS` (`main+0x134d2c0`) puts in `t.MaxFPS`, and `rhi.syncinterval=2` in `SwitchDeviceProfiles.ini`, the present interval 2 it asks for while `r.VSync=1` (`BaseSwitchEngine.ini`) locks presents to vsync. As for Octopath Traveler 0 ([01005270232F2000](../01005270232F2000/README.md)), the host writes 60 and 1 into the two console variables (`hle_fpslock.c`, profile key `fps_target 60`) whenever the game sets them back, and the game then asks for present interval 1 itself.
- StevensND's 60 fps mod for the base game's `main` (build `ab0602…`; GameBanana) lifts the same two limits with patches in code: it returns at once from `UEngine::SetMaxFPS`, so `t.MaxFPS` stays 0 (no limit), and it overwrites the first two characters of the UTF-16 name `r.VSync`, so the config's `r.VSync=1` finds no variable and presents are no longer locked to `rhi.SyncInterval`. Its "Uncap" variant also patches the `sdk` for present interval 0. Neither patch touches how the game steps time.
- Unreal steps the game by the time that really passed. On the Mac, `DM_NVN_REGLUM` traces of `run/to_maribel.sh` from the same save at 30 and 60 fps (frame time p50 33.4 → 16.7 ms) match at time scale 1.00 with a constant offset (−1.75 ± 0.2 s over the last 90 s, correlation 0.998–1.000), the night scene comes out at the same resolution, and no new red line appears.
- `DM_FPS_TARGET=30` (`Documents/env.txt` on the iPhone) keeps the game's own 30 and saves power.

## Memory

- `nninitStartup` (`main+0xe89fa4`) needs virtual address memory, sets a 6 MB heap for the C library's malloc and reserves the same 62 GB region as Octopath Traveler 0's (`main+0xe8a2c0`) unless `-SwitchVirtualAddressSpaceSize=` caps it. Its FMallocBinned3 cuts 37 GB of pools out of that region (the mapped high-water mark reached +38 GB of a 64 GB alias region at boot), so a capped region would starve it.
- **Current: the allocator is the host's.** Here the allocator is not in the bss: `GCreateMalloc` (`main+0x32758b0`) mallocs 0x1c38 bytes on the first allocation, during the static constructors, constructs them (`main+0x32758dc` → `main+0x291714`) and stores `GMalloc` (`main+0x77767e0`). The constructor call is cut, and the observation gate right after it (`OBS_UE_MALLOC`, x19 = the new object) puts the host allocator's vtable in (`hle_uemalloc.c`). No code outside the allocator reads its pool base (`main+0x7744770`, read only there) or calls its methods directly: every FMemory call goes through the vtable. UE 4.27's FMalloc has one FExec slot, not four, so the slot numbers differ from UE5's (Malloc 3 … GetAllocationSize 9; `Free` is `main+0x10`, right after the 16-byte module header).
- How the allocator was found: from the commit calls (probe `nn::os::AllocateMemoryPages` past its `adrp`, `sdk+0x140f84`: every backtrace runs through the allocator's methods around `main+0x3270000`), the relative relocations that point at them (the vtable `main+0x5e4c0e8`; this build has no RTTI) and the one store of that vtable outside the destructors.
- **Current: a 2 GB region** (`game_args -SwitchVirtualAddressSpaceSize=2048`) with Octopath Traveler 0's node-area cut on this SDK build, in a 3.75 GB guest block. The Unreal heap lives on a malloc zone outside the guest block.
- On the Mac: up to +0.75 GB of the alias region with 0.5 GB mapped through the opening and the new-game menus, beside an Unreal heap of 1 GB and 0.9 GB of Metal allocations (about 2.3 GB resident; `game.tsv`); about 2.6 GB resident in play (the Unreal heap about 1.25 GB, 0.9 GB of Metal allocations).

## Rendering

The desktop renderer with forward shading: one RG11B10 colour pass draws every object, lit per pixel from the light grid, the volumetric lightmap and the screen-space shadow mask, with HEXA DRIVE's toon lighting on top.

- **Resolved: the hero drawn as a dithered silhouette inside a dithered square.** Unreal's stencil shader resources are second textures on the depth buffer's storage built with `nvnTextureBuilderSetDepthStencilMode(STENCIL)`, which the host only logged, so the character occlusion mask (`edbfc7a088be545e`) read depth where it tests stencil bits. The builder's mode now goes to the texture (`tex_t.stencil`), whose sampling view is the `X32_Stencil8` one a stencil-mode texture view already got.
- **Resolved: characters and props glowing white in the opening cutscene only, about 10⁴ too bright, while walls and gameplay looked right.** The per-pixel volumetric lightmap read brick (0, 0, 0) of the brick atlas for almost every cell. Unreal merges each streamed level's lightmap into one atlas on the GPU: brick copies (`0a7ff4171bd1a82a`), then a patch of the shared 16³ `RGBA8_UINT` indirection texture (`7b958a7b21e72801`) whose bounds come from `GetDimensions` on the very `RWTexture3D` it writes. That size query is a TXQ on the image's handle, c2 `0x120` (image slot 0): the build numbered it combined texture slot 32, the slot that shares that word, and with no type in the facts table the translator read it as 2D. The host bound the empty texture slot 32, the 1×1 dummy's size passed only thread (0, 0, 0), so only that cell was written, every other cell kept size 0, and the shader fell back to the atlas origin: the first daytime level's bricks, baked about 2¹⁵ brighter (the same bricks in gameplay hold 0.5–1.4). Two general fixes: a texture binding of slot 32–39 reads the image bound in slot 0–7 when there is one, as c2 holds it (`nvn_mtl.m` `tex_slot_h`), and a query on an image handle the facts do not type is translated again with the type of that image from the first translation (`dmsh_xlate` `ImageTypesOf`).
- How the glow was found: the raw scene colour (`texraw`) put the characters at 1,000–2,000 and the walls at 0.05; a debug build of one character shader that wrote its terms in alternating 8-pixel columns isolated the volumetric lightmap ambient term; dumps of the atlas (`texlist` / `texraw`), of the uploaded level bricks (the 3D upload dump) and of the indirection texture, with the cell computed from the primitive's origin and View `VolumetricLightmapWorldToUVScale / Add` (View 220 / 221), gave the rest.
- Integer textures and images read and written through float declarations (the indirection `RGBA8_UINT` texture read by `texture3d<float>` and bit-cast, `R16_UINT` / `R32_UINT` light grid buffers through `texture_buffer<float>`, integer stores into a float-declared `texture3d` image): checked on an Apple GPU, all keep the integer bits, so they need no facts.
- **Unbuilt:** 62 of the 63 geometry shaders (one-pass cube shadow maps, six layers from one draw) do not build; their draws are skipped (point light shadows).

## Files and shaders

- Cooked packages in IoStore containers (`DOLL/Content/Paks/pakchunk0-Switch.utoc` / `.ucas`, 2.8 GB, and smaller chunks beside it; `.ucas` is the profile's `romfs_marker`); everything else in `pakchunk0-Switch.pak` (1.5 GB, 3,972 files: configs, the asset registry, fonts, CRIWARE `.awb` banks, the two shader code archives). The other `pakchunk*-Switch.pak` files are 339-byte empty paks.
- The pak is version 11 with the same 0xdd-byte footer as Octopath Traveler 0's, but the magic sits 0xcc from the end, not 0xc8 (five whole 32-byte compression method names, `Zlib` and `Oodle`); `uepak.py` takes either. The index is not encrypted. The update (v131072) is a delta over the base RomFS, so the merged extraction holds the whole game.
- Shaders: `ShaderArchive-Global-SF_GLSL_SWITCH.ushaderbytecode` (1,764 shaders) and `ShaderArchive-DOLL_Chunk0-SF_GLSL_SWITCH.ushaderbytecode` (63,185), stored in the pak: the same version 2 archives and `LSLG` records as Octopath Traveler 0's, but every shader is an LZ4 block (the first bytes are LZ4 tokens followed by `LSLG`; LZ4 is Unreal's default shader compression since 4.26), which `uepak.py` decodes itself (`lz4_shaders`: every compressed shader must decode to its raw size as an `LSLG` record, else the archive goes to the Oodle job as before). The NSP script runs no Oodle host for them.
- 62,901 unique shaders (10,114 vertex, 50,756 pixel, 1,968 compute, 63 geometry) of 64,949 in the archives. The game has no PC version, so the facts table holds only compute group sizes; every vertex shader is made to link with any pixel shader (`shader_split.py` `separable_pairs`).

## Audio and movies

- **Resolved: SIGSEGV at address 0x18 in `UpsamplerManager::Allocate` on the render thread.** The audio renderer runs at 32 kHz (rate 32000, 160 samples per frame, `GenerateSinkCommands` `sdk+0x225a80`). A DeviceSink below 48 kHz is upsampled by the SDK's upsampler, and `AudioRenderSystem::SetUpRenderResourceSet` allocates the `UpsamplerManager` only for the DSP (`sdk+0x21eb6c`: rate 0x7d00 and rendering device +0x48 == 0); with the host's CPU rendering the manager pointer (`AudioRenderSystem+0x150`, the render context's +0x20) is null. The host answers `Allocate` on a null manager with no upsampler, so the generator skips the upsample command (`sdk+0x225b40`), and resamples the DeviceSink's stereo downmix 3:2 itself (a 16-tap windowed sinc in three phases, `up_3_2` in `hle_audren.c`), 240 samples per 5 ms frame for the 48 kHz output.
- **Resolved: a movie in grey with vertical bands of colour, its luma sharp.** CRIWARE's Sofdec hands its VP9 video to `movie::Decoder` in mode 1, NV12 into its own buffers, and draws the planes with its shaders; the bands were the top row's chroma, every chroma row read from the first one. Sofdec reads `nv12-uv-stride` from the output format, which the host never set. The host now sets everything the SDK's `DecoderImpl::GetOutputBufferFormatFromDecoder` (SDK 19 `subsdk0+0x121f50`) does: also `nv12-uv-stride` (the Y stride), `nv12-width` and `nv12-height` (`md_format` in `hle_movie.c`).

## Services

- **Resolved: an abort in `sm::Initialize` under `audio::GetAudioDeviceOutputVolume`** (the game thread, `main+0xee8edc`): the volume `SetAudioDeviceOutputVolume` sets is now kept and read back (`hle_audren.c`).
- The import list against the hooks then gave the rest before they were met: `ec::FinalizeForShopServiceAccessors`, the shop response's `Cancel` and `GetErrorCode`, `ec::ShowShopSubscriptionDetails`, the Nintendo Account request's `GetAuthorizationCode` and `GetState` (both fail, as `GetIdToken`), and `kpr::GetKeyCodeMap`, which asks the settings service (`settings::detail::GetKeyCodeMap`) for the keyboard layout's map; the game zeroes a 0x1000-byte map and fetches it only once a keyboard reports keys (`main+0x1f0eb4`), and the host leaves it zero. The rest fill the game's own structures or sit behind a socket that fails offline.

## Rewriter

- **Resolved: `observation gate neither at a function entry nor after a bl`** for the gate on `mov x0, x19` at `main+0x32758e4`, after a cut `bl` and an `adrp`. A site is now also accepted when the straight-line code from it to the next `bl` reads none of x9–x17 before writing it (`scratch_dead`). The gate table takes the original instruction as a word value (`aa1303e0`), the cut table as bytes (`e00313aa`).
- `main` has two 23 MB stretches without unwind entries; they got through on a quieter Mac before functions were analysed in pieces, and the rewrite by the pieces rule is byte-identical to the previous `patch_nso.py`'s. It is the only `main` of 120 modules of 33 games with other `x18` sites before its first FDE (110 before `+0xb310`, none in a window or a `br x18`), and its rewrite stayed byte-identical when that text became one function.

## Mac routes

- The work folder's `run/ctl.sh` drives a run started with `DM_CTL` (`wait S` / `snap NAME`), and `run/to_maribel.sh <run>` goes from the title through a new adventure log into the opening cutscene's night scene by the boat (a save with an adventure log in the first slot): [05, Scripted input](../../../docs/agents/05-debugging.md#scripted-input-and-thread-scheduling).

## Found on this game

General host changes first needed here; each applies to every game:

- **Unreal Engine 4.27's FMalloc layout and the allocator made on the first allocation** (`hle_uemalloc.c` layout `UE4_27`, cut + `OBS_UE_MALLOC` gate): [Unreal's FMallocBinned3 replaced by the host allocator](../../../docs/reference/memory.md#unreals-fmallocbinned3-replaced-by-the-host-allocator).
- **LZ4 Unreal shaders decoded by `uepak.py` itself** (`lz4_shaders`): [Unreal Engine 4.27 shader code archives (LZ4)](../../../docs/reference/shaders.md#unreal-engine-427-shader-code-archives-lz4), [01, Compressed RomFS content](../../../docs/agents/01-extract.md#compressed-romfs-content).
- **Stencil-mode texture builders** (`tex_t.stencil`), **texture slots 32–39 reading the bound image** (`tex_slot_h`) and **image size queries typed from the image** (`dmsh_xlate` `ImageTypesOf`): [Stencil-mode textures from the texture builder](../../../docs/reference/rendering.md#stencil-mode-textures-from-the-texture-builder), [A texture slot that shares its c2 word with an image slot](../../../docs/reference/shaders.md#a-texture-slot-that-shares-its-c2-word-with-an-image-slot).
- **A 32 kHz audio renderer resampled 3:2 on the CPU** (`up_3_2`): [A renderer below 48 kHz with no upsampler on the CPU](../../../docs/reference/audio.md#a-renderer-below-48-khz-with-no-upsampler-on-the-cpu).
- **`nv12-uv-stride`, `nv12-width`, `nv12-height` in the movie output format** (`md_format`), **`GetAudioDeviceOutputVolume`**, **`kpr::GetKeyCodeMap`** and the shop / Nintendo Account calls: [Output format keys (`md_format`)](../../../docs/reference/movies.md#output-format-keys-md_format), [Audio device output volume and names](../../../docs/reference/audio.md#audio-device-output-volume-and-names), [Mouse, keyboard and key code maps](../../../docs/reference/services.md#mouse-keyboard-and-key-code-maps), [Shop service accessors](../../../docs/reference/services.md#shop-service-accessors), [Account, friends, play data and Nintendo Account calls](../../../docs/reference/services.md#account-friends-play-data-and-nintendo-account-calls).
- **Rewriter:** gates before code that writes the scratch registers first (`scratch_dead`): [Observation gates where x9–x17 are dead](../../../docs/reference/rewriter.md#observation-gates-where-x9x17-are-dead).
- **`DM_CTL` route scripts** (`run/ctl.sh`): [05, Scripted input and thread scheduling](../../../docs/agents/05-debugging.md#scripted-input-and-thread-scheduling).

## Open

- 60 fps on the iPhone: held for 3 minutes in an iPhone log of 2026-10-23 (GPU 13–15 ms mid-session, compile hitches up to 0.66 s). The same log had red lines for 7 point-light geometry shaders (missing shadows) and 1 texture type mismatch.
- 62 of the 63 geometry shaders do not build, so point light shadows are not drawn.


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

