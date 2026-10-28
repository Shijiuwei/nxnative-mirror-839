# 04 — Graphics: NVN on Metal for a new title

The graphics layer records NVN commands into its own command format (`dm/gfx/nvn/hle_nvn.c`) and replays them as Metal 4 at submit time (`nvn_mtl.m`). Shaders are never translated at run time: the original Maxwell shader binaries are translated offline (Ryujinx's translator → SPIR-V → spirv-cross → metallib) and looked up by key. See `dm/gfx/README.md` for the full pipeline; this page lists what changes per title.

## 1. Find every shader container

Do not stop at the obvious archive. In Mario the main `bfsha` held 4,888 of the 12,811 shaders the game actually uses.

`dm/gfx/tools/shader_split.py <RomFS> <ExeFS main> <pkg dir> <map out> <facts out> [<pairs out>]` scans **every** RomFS file (zstd-decompressed, SARC expanded recursively) and `main`, and recognizes the containers listed by engine in [reference/shaders.md](../reference/shaders.md#containers-by-engine). The pairs table lists the (vertex key, pixel key) of every BNSH variation or Unity program. Unity shaders need a second pass: files whose type trees are missing are scanned again once the bundles have supplied the `Shader` tree (or with the one `unityfs.py` has written out for that hash, when no bundle has it). For a base game plus an update, give RomFS and `main` as comma-separated lists, base first: keys are content hashes, so the output is the union of both versions and one shader build serves both (Mario 1.2.1 recompiled 3,451 of its pixel shaders; v0 rows keep their package indices).

Validate a new title's coverage at run time: `DM_NVN_SHDUMP=<dir>` makes the host dump the control section and code of every program it can't find in the map; search those bytes in the game files to find the container that was missed.

## 2. Key shaders by control section + code

The host key is `sha256(control[0:0x880] + code)[:16]` (`prog_get` in `nvn_mtl.m`, `shader_split.py`). D2R keyed by the control section alone; in Mario, 6,367 shader pairs share a control section but differ in code. A different code length or layout in a new title must be reflected in both places.

## 3. Find the binding layout by census

The NVN driver puts its binding tables in driver constant banks. Every game so far, NVN 53 and 55 drivers from SDK 6 to 21, has the one layout below, which `dmsh_build.py` decodes and `nvn_mtl.m` fills at run time. Check a new game against it with a census of what its translated shaders reference (the translator alone takes seconds) rather than assuming it.

| Binding | Where | How it was measured |
|---|---|---|
| Combined texture handles (`nvnCommandBufferBindTexture`) | **`c2[0x20 + 8 × slot]`**, 32 slots; a higher slot's word lies in the image table | Mario: immediate handles of all TEX instructions, 0x20…0xF0 in steps of 8. Hades II binds slot 34 (0x130) and no image |
| Image handles | `c2[0x120 + 8 × slot]`, 8 slots | agl compute shaders: 0x120 / 0x128 / 0x130 |
| Separate textures (`BindSeparateTexture(s)`) | `c2[0x168 + 8 × slot]`, 128 slots: the host's texture slots 128 + n (`NVN_SEPTEX_SLOT0`) | Engage (Unity binds only separate textures and samplers): the translator's texture names (`fp_t_cb<slots>_<handle>`, Ryujinx `TextureHandle` packing) give word 0x5A = 0x168 for the texture and 0x15A = 0x568 for the sampler. D2R (NVN 53) had the same |
| Separate samplers | `c2[0x568 + 8 × slot]`, 128 slots | as above |
| Uniform buffers | `c[3 + slot]` | |
| Storage buffers | `c0[base + 16 × slot]`, base vs 0x110 / cs 0x310 / ps 0x510 | |
| Largest point size | `c0 + 0x610` | Super Mario Odyssey (SDK 7): GLSLC clamps a shader-written `gl_PointSize` to it (`alPointSpriteShader`: `min(size, c0[0x610])`). It is not the `SetPointSize` value, which the game leaves at 1. The host writes 2047 there for the graphics stages |

Ace Attorney Investigations Collection (Unity 2022.3, SDK 17.5.3) binds separate textures and samplers like Engage: all 4,950 translatable shaders decode every binding (`shaders with a binding whose slot was not decoded 0`), and 190 read `c2` rows as data.

Metroid Dread (SDK 11.4.3) uses Mario's texture handle layout, `0x20 + 8 × slot`: the `TEX` / `TEXS` handle immediates in its shader files' own SASS text are words 0x8–0x22 in steps of 2. Its full build leaves no unresolved bindings and no unfillable driver-bank reads; the other rows are not measured separately.

Hades II (SDK 19.3.9) binds combined textures at fixed, sparse slots up to 34 (`fp_t_tcb_4C`: c2 word 0x4C = 0x20 + 8 × 34, in `SceneFade.frag`). With a 32-slot table, `dmsh_build.py` reported that binding as not decoded ("shaders with a binding whose slot was not decoded 1", which prints a red line when the shader loads). The combined table now runs up to the separate textures at 0x168 (41 slots); past slot 31 it shares its words with the image table, and the replay writes a bound texture there when the image slot is unbound. The census of `fp_t_*` names in the translated libraries gives the slots in use.

Monster Hunter Rise (RE Engine, SDK 11.4.3) uses both tables at once: combined handles from `nvnCommandBufferBindTexture` at `0x20 + 8 × slot` and separate textures from `BindSeparateTexture(s)` at `0x168 + 8 × slot`, with separate samplers at 0x568, and binds slot 0 of both in the same frames. A census of 23,478 translated shaders' texture names: 33,813 combined (`fp_t_tcb_1E`), 29,342 separate textures with a constant sampler ID (handle type 3, `fp_t_cb65538_3041005C`: texture at c2 word 0x5C, sampler 0x104 ORed in), 8,265 separate texture + separate sampler (type 1, `fp_t_cb196610_1017815A`), 5,227 separate textures whose own handle carries the sampler (`fp_t_tcb_5A`), about 1,500 uniform-buffer handles with a constant sampler (`fp_t_cb65541_30410028`), and 121 fully dynamic handles (type 4, `fp_t2d_cb0_40000000`, not supported). Separate binds go to slots 128 + n of the host's table and their handles decode to 128 + n, as for every game; a constant sampler is the binding kind `samp_id` (`NVN_RB_SAMPI`, reg = the sampler ID).

`dmsh_build.py` decodes the bindings with this layout (`decode_tex`), `Program.cs` numbers slots the same way when it looks up per-slot texture / image types in the facts table, and the replay layer fills driver banks with it: the combined, image, separate texture and separate sampler tables of `c2` (`NVN_RB_DRV2` in `encode_args`). Some shaders read `c2` rows as data at run time (Engage: 243, the texture and sampler handles of slots 0–11); the build lists reads outside those tables as `cannot fill`. After a full build, the summary must say `shaders with a binding whose slot was not decoded 0`.

## 4. Bindless textures

Mario's shaders read 38 % of their textures through handles stored in the game's own uniform buffers (`fp_t_cb7_2A` = handle at word 0x2A of c[7] = uniform slot 4). These became binding kinds `tex_ubo` / `samp_ubo` (`NVN_RB_TEXU` / `NVN_RB_SAMPU`): at replay the host reads the 32-bit word from the bound uniform buffer (low 20 bits texture ID, high 12 bits sampler ID).

Consequence: any optimization that infers "which textures a draw reads" from bound slots is wrong for such shaders. The TBDR optimizations (store / load elision, pass merging) are therefore off by default in this repository (`tbdr_on()`, `DM_NVN_TBDR=1` to enable).

## 5. Depth convention

`nvnDeviceSetDepthMode` near = −w (GL, z in [−w, w], NVN's default) versus near = 0 (D3D / Metal, z in [0, w]). Metal clips anything with z < 0, so a GL-mode game's vertex shaders must end with z' = z/2 + w/2. The mode is runtime state, and games differ: Mario, Metroid Dread, Super Mario 3D World and Breath of the Wild keep GL; Link's Awakening, 13 Sentinels and Engage set near = 0.

So the translator leaves z alone (`QueryGraphicsState` depth mode false in `Program.cs`), and `dmsh_build.py` (`vs_depth`) writes two entry points into every vertex library: `main0` with the GL rewrite (the same arithmetic Ryujinx emitted) and `main0_z` without it (reflection `entry_z`). The host picks one when it loads a vertex shader, from the mode the game set (`mtl_set_depth_zero`, `shader_load` in `nvn_mtl.m`). A new game needs no setting; if a game changed its mode after vertex shaders were loaded, a red line says so.

- Rewriting a near = 0 game for GL still draws correctly (depth tests keep their order), but every depth lands in 0.5..1, so effects that turn depth back into distance are wrong: Link's Awakening's outdoor depth of field blurred the whole screen.
- Not rewriting a GL game: the full-screen copy into the window is clipped away, so the window stays black even with a solid-colour fragment shader.
- Metal function constants could pick the variant too, but any function with a function constant has to be specialized before a pipeline can use it (measured: about 2.4 ms per vertex shader on an M4 Pro); the second entry point only costs build time and about 30% more library size.

## 6. Where the per-title findings are

What earlier titles needed beyond sections 1–5 is filed by mechanism: textures, passes, clears, vertex input, uniforms, conditional rendering and the Vulkan and OpenGL layers in [reference/rendering.md](../reference/rendering.md); containers by engine, control sections, translation, bindings and run-time shaders in [reference/shaders.md](../reference/shaders.md); memory pools, command memory, queues, syncs and CPU-written textures in [reference/nvn-objects.md](../reference/nvn-objects.md); present intervals and display rates in [reference/frame-rate.md](../reference/frame-rate.md). Each entry names the games it was found on, and each game's own findings (its passes, shader keys, measurements, profile keys) are in its note, `dm/games/<id>/README.md` ([games table](../../dm/games/README.md#games)). Search [symptoms.md](symptoms.md) for what you see first.

## 7. Diagnostic switches (host)

| Switch | Output |
|---|---|
| `DM_NVN_SHOT=f1,f2:path_%d.png` | Window screenshots at those presented frames |
| `DM_NVN_PASSDUMP=f`, `f1-f2` or `every:n` (+ `DM_NVN_PASSDUMP_ALL=1`) | Every render pass of those frames (`every:n`: one frame in every n, halfway between two `DM_NVN_GPUPROF=every:n` frames): `P` attachments (with `mtl=` the Metal texture behind them), `C` clear colours, `K` shader keys, `B` each binding's actual source (also with `mtl=`), `d`/`D` draw state (`b0`–`b3` the blend states of targets 0–3; on `d`, `po=` the polygon offset enables / factor, units, clamp and `dc=` depth clamp), `Y` inline updates, `X` texture clears, `T` texture copies, `Z` the game's barriers (their bits, and whether one ended the pass), `N` passes skipped because no attachment could be attached, `C` buffer-to-texture copies (target, size, format, region, source address, row and image strides), `W` image bindings of compute dispatches; a buffer texture's `B` line gives its format, element count and storage address (for the Mac's `DM_CTL` `memdump`). Without `_ALL` the `K` / `B` / `R` lines cover only the first draw of each pass |
| `DM_NVN_TEXDUMP=f:path_%d.png` (+ `DM_NVN_TEXSNAP=i,…`) | Every colour render target of frame f, as PNG (`%d` = serial × 100 + layer × 10 + level; 2D arrays by the attached layer). `TEXSNAP` copies the attachments of in-frame pass i (2D, or slice 0 of a 2D array) in a separate command buffer, which does not run at the pass's position in the GPU timeline: a snapshot taken right after a clearing pass showed a complete image, so read it as "what this target holds around the frame", not "right after pass i" |
| `DM_NVN_TEXDUMP=size=WxH:path_%d.png` | The same dump for the first frame that opens a render pass whose first colour attachment level is W×H: for one-off work (a mask built when a prompt appears) whose frame differs between runs. `layers=<n>` instead of `size=` takes the first frame with at least n draws into single layers of small (at most 64×64) array or cube levels (Breath of the Wild's cube map filter frame), the same frame with or without `DM_NVN_SLICE_MERGE`. With `DM_NVN_SNAPRAW=1` each image also gets a `.raw` file with the alpha channel |
| `DM_NVN_CBDUMP=dir` (with PASSDUMP) | Raw uniform buffer contents per binding, and the source bytes of every buffer-to-texture copy into a 3D texture (`F<frame>_c3d_<serial>.bin`: what the game uploaded, before any layout question) |
| `DM_NVN_HILITE=fskey,…` | Replace those fragment shaders with solid colours: tells "geometry is wrong" from "shading is wrong" |
| `DM_NVN_DEPTHOVR=fskey,…` | Those fragment shaders' draws compare depth with `ALWAYS`: with `HILITE`, tells "the depth test rejects every fragment" from "nothing rasterizes". With `PASSDUMP`, the `D` line shows the depth range (`dr=`) and logic op (`lo=`), and `c` lines each recorded clear (slot, serial, mask, scissor; depth / stencil values) |
| `DM_NVN_SHDUMP=dir` | Control + code of programs missing from the shader map |
| `DM_NVN_VSDUMP=<file>` / `DM_NVN_VSDUMP_KEY=<vertex key>` | One native post-vertex capture: a u32 header (first vertex, count, stride in words, base instance), then the 144-word geometry records. Refuses shaders with writes. With key `*`, saves position-only files by draw/key; `DM_NVN_VSDUMP_FROM` selects the first frame, `DM_NVN_VSDUMP_MAX` limits captures (default 256). These diagnostics drain GPU work |
| `DM_NVN_ALLOCPROF=1` | Metal allocation growth per replayed op type and per function, command allocator sizes, new textures |
| `DM_NVN_CBDUMP_KEY=vskey` (with `CBDUMP` and `PASSDUMP_ALL`) | Only that vertex shader's draws: uniform buffers plus vertex streams and index buffer (`F<frame>_p<pass>_vb<stream>_<addr>.bin`, `_ib<type>_<addr>.bin`) |
| `DM_NVN_TEXINFO=WxH` | Storage and description (pool, offset, flags, packaged or not, storage size and how many of its bytes in guest memory are non-zero, i.e. written by the CPU) of every texture of that size on first use, each pool registration of one (`TEXINFO register`: builder channel mapping, view and view mapping), and each one as it is initialized (`TEXINFO init`, render targets included, the first 200: serial, pool and offset, target, format, flags, storage size) |
| `DM_NVN_ZBLOCK=1` or `=<folder>` | Each ASTC texture uploaded on the CPU whose bound levels hold all-zero blocks (ASTC's error colour, magenta): `[zb]` lines with each level's zero blocks, its total and whether its pages are mapped (sparse pools). With a folder, also the raw storage of the first one of each format and size (`zb_<serial>.bin`; 2 MB and a byte-per-4 KB-page map `zbm_<serial>.bin` for sparse textures) |
| `DM_NVN_TEXRAW=info` (with `TEXDUMP`) | Raw bytes of the non-render-target textures `TEXINFO` picked out, at the `TEXDUMP` frame |
| `DM_NVN_TEXRAW=ser,…` (with `TEXDUMP`) | Raw bytes of those textures (by serial, as `PASSDUMP` prints them) at the `TEXDUMP` frame; a Depth32Float_Stencil8 texture gives its 32-bit float depth plane (a shadow map) |
| `DM_NVN_T2T_TRACE=WxH:file` (+ `DM_NVN_T2T_FROM=f`) | After each texture copy into an RGBA16F texture of that size, a GPU-timeline copy of the result into a trace buffer; 240 entries written to `file` (header line, 240 frame numbers, then the texels) |
| `DM_TEST_PRESENT_LAG=n,ms` | Sleep `ms` before every `n`th present: a slow frame on demand (Mario then switches to present interval 2) |
| `DM_NVN_BLIT_ORDER=0` | No barriers between dependent operations in the copy encoder (reproduces the fountain bug) |
| `DM_NVN_KEYDUMP=<fs key>[:n[:every]]` | For up to n draws of that fragment shader (default 6; at most one every `every` frames), the `PASSDUMP` binding lines plus `[kd]` lines: vertex attribute formats, each vertex stream's data (all instances of per-instance streams) and every value of both stages' uniform buffers. Meant for devices: no frame number to guess; compare with the same switch on the Mac |
| `DM_NVN_MAGSCAN=<fs key>[:every[:phase]]` | Device self-check for solid colour blocks. Every `every` frames (default 30) a compute kernel reads each render pass's colour attachments as the pass ends and logs `[mag]` lines for 16×16 blocks that hold one magenta value throughout, or non-finite pixels, naming the pass relative to the one holding that shader. Every `phase` frames (default 600) it rotates: 0 draws as usual, 1 drops that shader's draws, 2 / 3 keep its geometry with a constant grey / bright magenta fragment output. Each phase ends with a summary line, including the magenta pixel count in that shader's own pass |
| `DM_NVN_PREDICT=0` | No parallel precompile prediction: a pipeline miss compiles only the pipeline it needs. For telling a fault of the look-ahead from one of the pipelines ([reference](../reference/shaders.md#the-precompile-look-ahead-restores-the-scissor)) |
| `DM_NVN_FB_SNAP=0` / `all` / `<keys>` | Which fragment shaders get the snapshot for same-pass feedback sampling (a draw sampling a colour attachment of its own render pass reads a copy taken just before it): none, every shader (slow, see BotW above), or a comma-separated key list instead of the profile's `feedback_snap` |
| `DM_NVN_CUT=<draws>` | Commit the replay's command buffer every that many draws, between passes (default 64; 0 = one command buffer per submission, except the cut at the first pass boundary after a buffer → texture copy, which always happens: Seen in Catherine: Full Body). Growing segments (64, 128, 256 … draws) were tried for fewer commits: Breath of the Wild fell from 60 to 30 fps in the same scene (GPU idle while the long last segments were encoded), and the commits cost little once the residency set stopped churning |
| `DM_NVN_UB_DEFER=0` | End the render pass at every inline uniform update (`nvnCommandBufferUpdateUniformBuffer`), as before. By default an update inside an open pass is deferred to the pass end; the draws after it read a shadow made of the deferred updates when they cover all the shader can read of the binding (`cb_extent.tsv`), or of memory and the updates queued in the submission when they cover part of it (`DM_NVN_UB_PARTIAL`), and the pass still ends first for a draw that needs more (the data read as vertices, indices, indirect arguments or a storage buffer, or a partly covered block memory cannot give). The `residency set` line counts deferred updates, shadows read from memory and early ends |
| `DM_NVN_UB_PARTIAL=0` | A uniform block the deferred updates cover only in part ends the pass, as before. By default its shadow starts from memory and takes every inline update queued in the submission, newest last, when nothing else can have written the range by the time the pass runs: no copy, fill or transform-feedback write queued in the submission, no storage buffer a shader may write (every one but the slots `cb_extent.tsv` proves unwritten) and no texture→buffer copy over it in the submission, and no GPU write or counter report of an earlier submission still running (by 4 KB page). A counter report of the same submission does not count: the host writes reports on the CPU after the whole submission completes. The `residency set` line splits the refusals by reason |
| `DM_NVN_UBP_CHECK=1` | Check every uniform shadow read from memory: the pass ends, the updates land, the replay waits for the GPU and compares (`uniform shadow check: N equal, M different`; a difference is a red line). Breath of the Wild outdoors: 388,000 equal, 0 different |
| `DM_NVN_LAZY_CLEAR=0` | Clears with no draw after them open a pass of their own when the game sets other targets, as before. By default they wait: the next pass that attaches the same texture level and layer through the same view clears it as its load action, and the rest land in passes of their own before anything else could see their textures (a draw sampling one, a pass not attaching it, any operation other than targets, clears, state, uniform updates and buffer copies or fills, the end of the command list). Off in TBDR mode. The `residency set` line counts clears taken by a later pass and those landed alone |
| `DM_NVN_FLUSH_FILL=0` | A new GPU texture in a pool the CPU can write starts as zeros, as before. By default one whose storage the game flushed (`nvnMemoryPoolFlushMappedRange`) before its first use is filled from that storage once (`initial content: … flushed by the CPU before its first use` lines; Breath of the Wild's sky tables) |
| `DM_NVN_BARRIER=game` / `full` / `ab` | How draw passes are ordered (`ab`: alternating with the `DM_AB` phases). `full` (default): every pass waits for all earlier GPU work. `game`: a pass waits only for the previous pass's fragment stage, so its vertex work overlaps the previous pass's fragments, unless the game called `nvnCommandBufferBarrier` in between or a compute / copy / upload ran (those keep full barriers). Only vertex-stage reads of the previous pass's output without a game barrier change order |
| `DM_NVN_GPU_TS=segment` / `submission` | What timestamp reports record, instead of the profile's `gpu_timestamps`: the GPU start / end of the segment holding each report, or one time for the whole submission |
| `DM_NVN_DISPATCH_OWN=1` | Every compute dispatch in a compute encoder of its own with full barriers, as before (by default dispatches share the copy encoder, `dispatch_enc`) |
| `DM_NVN_VCONV_CPU=1` | Integer vertex inputs fed formats of another class are converted on the CPU, as before (by default the shader's `<key>_bv` library takes them through shadow inputs, `vin_on`) |
| `DM_NVN_GPUPROF=f`, `f1-f2` or `every:n` | GPU start and duration of every encoder of those frames (`every:n`: the frames that are multiples of n, for a phone), from Metal 4 counter heap timestamps (`[gpuprof]` lines: kind, size, attachments, draws, command-buffer segment, attachment load / store KB, Metal pixel formats of colour 0 and depth; a `frame` line with render and compute / copy time, attachment traffic and the span from first start to last end). `dm/tools/gpuprof.py` sums them by pass shape ([05](05-debugging.md#gpu-time-on-the-device)) |
| `DM_AB=<frames>` | Switches set to `ab` alternate in phases of that many frames, in one process and scene; an `[ab]` line per phase with GPU and frame time and attachment traffic ([05](05-debugging.md#gpu-time-on-the-device)) |
| `DM_GPU_CAPTURE=f[:n]` or `gpu:<ms>[:<after>]` | A Metal GPU trace of those frames for Xcode's Metal debugger, in `gpu_captures/` of the working directory ([05](05-debugging.md#gpu-time-on-the-device)) |
| `DM_NVN_ENC_THREAD=1` | Metal encoding on a thread of its own: the submit thread records the encoder calls and a worker replays them (`nvn_rec.h`); on by default on iOS (`DM_NVN_ENC_THREAD=0` turns it off), off on macOS |
| `DM_NVN_UB_LAZY=n` | The most deferred inline uniform updates kept across a render pass end (default 160): they land before the next copy, a draw or dispatch that reads their memory other than through uniform bindings the shadows cover (`ubd_conflict`, `ubd_conflict_cs`), a transform feedback draw, or the end of the submission, instead of at every pass end; 0 lands them at every pass end, as before |
| `DM_NVN_UB_COPY_THREAD=0` | Uniform shadows of 512 bytes or more are copied on the submit thread, as before (by default a worker copies them) |
| `MTL_DEBUG_LAYER=1 MTL_DEBUG_LAYER_ERROR_MODE=nslog` | Metal validation without aborting |

A first-frame bring-up that worked for Mario: screenshot is black → `TEXDUMP` shows which targets have content → `PASSDUMP` the pass that writes the window → `HILITE` its fragment shader → `CBDUMP` its uniforms → read the translated `.metal` of its vertex shader.


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

