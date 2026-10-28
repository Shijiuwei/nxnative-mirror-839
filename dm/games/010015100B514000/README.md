# Super Mario Bros. Wonder (`010015100B514000`)

Base game v0 (1.0.0, SDK 16.2.0), update 1.0.1 (v65536, SDK 16.2.0; short Mac crash regression only) and update 1.2.1 (v327680, a new `main` build on SDK 16.2.12); the profile records these builds and packs 1.2.1 by default. Nintendo EPD's engine (sead, agl), on NVN 55 (it requires 55.x with minor ≥ 15). Modules `main` and `sdk`. The first game of this repository, after Diablo II: Resurrected in d2r-mobile: much of the host was brought up on it. Plays on the iPhone. The README lists its IPA at about 4 GB.

## What it needs

| Setting | Where | Why |
|---|---|---|
| 5 GB guest layout: `mem_reserve 0x140000000`, `mem_alias 0xB8000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | [`game.tsv`](game.tsv) | 256 MB in front + 2.875 GB alias region + 256 MB stack region + 1.625 GB heap region: the game maps 1.75 GB of virtual address memory on demand into the alias region and asks for a 1.47 GB heap |
| `shader_versions v0 v327680` | `game.tsv` | the shader tables are the union of both versions, so packing either needs both NSPs |
| `feedback_snap a7e5c6fd501cf21f 22a61cb1ec732aef` | `game.tsv` | two full-screen passes sample their own attachment; without a snapshot the Mac showed flickering black and neon green 16×16 blocks; see [Rendering](#rendering) |

No game-specific executable cuts. No `dm/tools/sdk_patches.py` cut for its `sdk` builds (`3053b229…` v0, `e4bd58ca…` 1.2.1). No `render_cond`: a minute's boot counted 65,536 or more clears under render conditions, and the game needs none. No `pad_layout`: the virtual gamepad starts in the 2D layout.

## Versions

- v0 (1.0.0), 1.0.1 (v65536) and 1.2.1 (v327680). With 1.2.1 layered over v0, unchanged files become APFS clones, so the merged RomFS adds only about 114 MB to the 3.5 GB base.
- 1.2.1 imports 28 SDK functions v0 did not, which the host answers: `nn::nfp` (amiibo, initialized at boot) finds none, `pl::IsRunningOnOunce` (is this a Switch 2) is false, `ec::ShowShopApplicationInformationForUpgrade` is a no-op, `account::PopOpenUsers` returns none and `aoc::CountAddOnContent` none. Mario only reads the count from `PopOpenUsers` and falls back to `TryOpenPreselectedUser` on 0, and counts add-on content only on a Switch 2 (`hle_sdk16.c`, `hle_account.c`, `hle_misc.c`).
- 1.2.1 compresses the NACP's title area (a u16 size, then raw deflate inflating to 32 entries of 0x300 bytes), which LibHac 0.19 does not decode; `nx_extract` and the NSP script do. Its patch NCA (key generation 17) makes hactool stop with `Invalid BKTR layout!`, which is why `nx_extract` uses LibHac.

## Memory

- **Resolved: resource heaps exhausted by oversized host command records.** The game measures scratch recordings while building model resources (`main+0x57500`, shape/pass recording `+0x5b8c0`, mesh draw measurement `+0x593ec`, 1.2.1). Those measurements feed the GPU chunk size, although the resource heap itself is fixed by Nintendo's resource-size table. The measured templates contain uniform and vertex-buffer bindings whose host records were larger than the native commands. Compacting these shared host records fixes the allocation. See [Model heaps sized by command records](../../../docs/reference/memory.md#model-heaps-sized-by-the-hosts-command-record-sizes).

- **Measured failure and fix (1.2.1).** With the old heap allowance disabled, `Model/PlayerMarioSuper.bfres` receives table value `0x80b660` + `0x1008`, a `0x80c668`-byte ExpHeap. Its final MatBlock chunk was `0xc000` bytes, leaving `0x648` free for a `0x6b8` descriptor: 112 bytes short. Payloads, block headers and alignment padding account for the entire heap, so this was not free-list corruption. Compact uniform bindings alone reduce the requested GPU budget from `0xb8d4` to `0xb76c` and still fail; compact single vertex bindings reduce it to `0xa6ec`, so the chunk becomes `0xb000` and the descriptor fits. Compact render-state records had no effect because these scratch templates do not contain them; that experiment was reverted.

- **Validation and remaining scope.** On the Mac, fresh isolated saves and `DM_KEYS='5:A,7:A'` reproduce the unpatched 1.2.1 null-pool crash in about 8 seconds. The general fix passed that route, then 60-second headless runs of both 1.0.0 and 1.2.1 with additional A presses, each reaching reported frame 3300 without the heap allowance, null-pool diagnostic or crash. `t_nvn` passes 152 checks; `t_reorder_chain` passes 171 including compact/full uniform-binding pixel equivalence; `t_vbuf` passes 45 including compact vertex bindings and GPU readback. These game runs use no translated shader libraries and establish CPU allocation progress, not full rendering, gameplay or iPhone acceptance. A fresh 1.2.1 rewrite after removing the profile patch emits zero cuts and is byte-identical to the executable used in those successful runs. 1.0.1 was not rerun with this fix.

- **Removed workaround.** The former +16 KB per-resource cuts at v0 `main+0x1e62fc`, 1.0.1 `+0x1f6890` and 1.2.1 `+0x1cf9b0`, and their `MATCH_CUTS` rules, are removed. The original v0 failure was the same null-pool symptom for `Model/NpcCommonerA.bfres` after its `0x9000` MatBlock chunk. The shared [function matcher](../../../docs/reference/rewriter.md#function-matched-cuts-for-unregistered-builds), originally verified using this workaround's 1,420-byte heap-setup function across v0 and 1.2.1, remains available to other profiles; Wonder no longer enrolls a rule.

- The game reserves "virtual address memory" (1.75 GB in `System/HeapSize/*.HeapDef.byml`) and maps it into the alias region with `MapPhysicalMemory` on demand. With the alias region left after the node area too small, large resources were silently skipped and a null deref followed where they were used, also visible as `SetHeapSize` asking for much less than the pool. The 5 GB layout above fixed it.
- Its two modules are about 74 MB, and its heap request matches hardware with the host's 64 MB of "used" memory.
- The single 128 MB NVN virtual memory pool (flag 0x200) is mapped onto one physical pool in one request, so the host aliases it (`hle_nvn.c`). Before, a failed `MapVirtual` gave a buffer with a null pool right after a scene change.
- How the heap was traced: probes on the gsys GPU allocator (`main+0x4290c0` in v0, `+0x4286e0` in 1.2.1) and the heap's `tryAlloc` (vtable slot +0x30); the `!xN` heap walk uses Mario's sead layout by default (`DM_PROBE_HEAP` `20,28,30,b0,c8`; [05, Probes](../../../docs/agents/05-debugging.md#probes-without-a-debugger-dm_probe)).

## Frame rate

- **Current:** 60 fps with a dynamic present interval. The game times every frame (`main+0xcfd24`); after a frame that ran past one vertical sync it sets present interval 2 for the next frame and sets its own time step to the interval (a float at `main+0x36ceff0`), so that frame catches up 2/60 s (Mario 1.2.1 on the iPhone). A user's 2-minute iPhone session switched 123 times, mostly for one frame.
- **Resolved:** in `display` mode those frames were shown for 16.7 ms and the game visibly jumped ahead ("sometimes the speed increases when jumping"). In `game` mode (the default since 2026-10-15) the display link keeps a frame whose period is over 25 ms on screen for its whole period (`g_link_hold_until` in `nvn_mtl.m`), and the 30 fps target waits until interval 2 has held for a full second (`nvn_game_fps` in `hle_nvn.c`), so one-frame switches no longer flip the display-link mode. `DM_TEST_PRESENT_LAG=90,30` (30 ms before every 90th present) makes Mario do this on the Mac; run it with `DM_WINDOW=1` to get the display link.
- `core_preempt` stays off: with it Mario's loader threads waited for busy decompression threads and its first frame moved from 0.6 to 2.9 s. The shared [wake-triggered handoff](../../../docs/reference/threads.md#same-core-wake-handoffs) applies with it off: a 2026-10-26 boot smoke check reached the first frame in 1.36–1.56 s versus 1.31–1.50 s with the old host (three fresh saves each, no current shader libraries; this checks startup scheduling, not rendering performance). Logs are in the NSP work folder's `wake-handoff-check/`.
- The precise `SleepThread` (`sleep_precise`, found on Metroid Dread) took Mario's frame-time p95 from 19.7 to 18.4 ms.
- The spin-wait rewrite finds 175 loops in `main`; the store rule added one more in each build.

## Rendering

Game-specific findings; the general host changes they led to are listed under [Found on this game](#found-on-this-game).

- **Resolved: tall blue / textured panels flickering over the fountain.** Mario writes the same uniform slot twice in a row each frame (4 bytes of −1, then 16 bytes of wave coefficients). Without an intra-encoder barrier the writes occasionally landed in reverse order, the fountain's GPU wave simulation (a 512×1 RGBA16F texture, render → copy back → splash pass every frame) ran with −1 as its self coefficient, diverged to ±65504, and its vertex shader pushed the water surface to the top of the screen. How it was found ([05, Intermittent rendering glitches](../../../docs/agents/05-debugging.md#intermittent-rendering-glitches)): screenshots of frames 4430–4590 scored by a metric tuned to the glitch (long vertical straight edges in a sky region: ≈1.0 against ≈0.3), a control run, `DM_NVN_SKIP_FS` bisection, `DM_NVN_CBDUMP` showing byte-identical inputs, then `DM_NVN_T2T_TRACE` and a numpy re-implementation of the shader step: a coefficient of −1 instead of 1.81 explained every bad texel. `DM_NVN_BLIT_ORDER=0` reproduces it.
- **Resolved: the whole picture darker and more saturated (Mac and iPhone).** Mario's last pass writes the `RGBA8` swapchain texture through an `RGBA8_SRGB` view (`TextureViewSetFormat` 0x38 on a 0x25 texture); the host attached the base texture and stored the linear values unencoded (against a Switch screenshot, 0.55 → 0.26, 0.75 → 0.52). It is the only such view, 1 of 80 passes in a course frame.
- **Resolved: menus and dialogue boxes with no text.** Glyphs are rasterized on the CPU into a linear 2048×8192 R8 texture (flags IMAGE | LINEAR) in a CPU-visible pool; the font atlas is sampled with the swizzle (1, 1, 1, R), without which glyphs were white boxes.
- **Resolved: a black sky.** Four 64×1 RGBA16F sky colour ramps are CPU-written linear textures.
- **Resolved: a black band and flicker (iPhone; Mac runs of the scene looked clean).** Mario writes the byte 1 at `NVNcommandBuffer` +0x10 (`main+0x4da5c`, `main+0xb8dec`), which was the low byte of the host's `end`. Each write shrank the 512-byte material setup blocks, their recordings overflowed into host segments, `GetCommandMemoryUsed` returned 0, the game marked the setups empty and skipped calling them, and draws ran with the previous draw's vertex buffers (out-of-range vertices, NaN). `cmd_t` now reserves the byte (`nvn_flag10`).
- `NVNprogram` is smaller in Mario than the 0xC0 bytes in `nvn.h`: a 168-byte host record corrupted the next object (a crash in SHA-256 on the code address 0x16300), so `prog_t` stays 112 bytes.
- Mario calls `nvnProgramSetShaders` for some programs with a code address not yet mapped, so such a stage is hashed at its first draw.
- Its transient render targets are new texture objects every frame over the same storage, never finalized; without sharing host textures by `(pool, offset, description)` they leaked about 13 MB a frame.
- 8-bit index buffers in its full-screen passes and final copy; scissored zero clears of sub-rectangles of small targets (pyramid levels packed into 480×272, 256×256 and 64×36 targets); a depth clear scissored outside a pass, which the host still clears whole (`depth-stencil clear with a scissor or partial stencil mask, cleared whole`).
- Its 4×4 fallback textures (black, white, transparent, red) are filled with `nvnTextureWriteTexels`; since they upload from guest memory, materials that sample the default white texture got slightly brighter.
- **Resolved: flickering black and neon green 16×16 blocks in courses and dialogue (Mac: Apple M6, v0 and 1.2.1; Apple M1, 1.2.1, [seedds/nxnative#377](https://bezm.wtpuscm.cn/youhua/restaurant-713056.html)).** Black or (0, 252, 0), aligned to the 16-pixel grid, sometimes only part of a tile; most visible on the speech bubbles (edges, corners, between words), also in the lower left of the first course; title screen and menus clean. Two full-screen passes sample the 1920×1080 RG11B10F scene they render into (`a7e5c6fd501cf21f`, blended, from `GameAglShader`; `22a61cb1ec732aef`, agl's `agl_technique_pfx`), the same keys in v0 and 1.2.1; with both snapshotted the 1.2.1 log names only these two (`ser=4190`, format 92), 2 snapshots a frame. Screenshots counted by a detector of solid grid-aligned tiles that differ from their surroundings and from the same tile in the screenshots before and after: v0, the first course's opening dialogue, with `DM_NVN_FB_SNAP=0` 6 of 160 screenshots, with `DM_NVN_FB_SNAP=all` 0 of 80, with only these two keys 0 of 120, and 0 of 150 with them in the profile; 1.2.1 from #377's save, the Poplin's dialogue in the Badge House (route in [Mac routes](#mac-routes)), 23 of 158 with `DM_NVN_FB_SNAP=0` (22 black and 4 green tiles) and 0 of 158 with the profile. The two snapshots cost about 0.05 ms of submit and 0.3 ms of GPU time a frame there (GPU p50 3.95 → 4.27 ms); snapshotting every shader instead raised submit time from 1.4 to 4.0 ms. The iPhone had shown no magenta blocks from them. See [Same-pass feedback read through a snapshot](../../../docs/reference/rendering.md#same-pass-feedback-read-through-a-snapshot).
- Uniform updates inside a pass: attachment loads and stores unchanged by the deferred updates (its updates already covered what the draws read).
- It ships agl's BC4 block encoder `0544282c162305db` (the shader Tears of the Kingdom draws into a BC4 3D texture).
- Mario keeps NVN's GL depth mode (near = −w).

## Shaders

- 12,811 shaders in v0 and 16,268 across both versions; 16,266 compile. In v0 the main `bfsha` held only 4,888 of the 12,811.
- Containers: `Shader/*.bfsha` (BNSH), `Lib/agl/*.sarc` + `Shader/Secred.sarc` (agl BAHS `sharcb`), `Effect/*.esetb` (BNSH), `Layout/*.blarc` (BNSH) and two BNSH inside `main`. Groups in `game.tsv` (`SecredUber`, `agl`, `effect`, `layout`).
- Texture handles at `0x20 + 8 × slot` (the docs call it "Mario's binding layout"). 38% of texture reads go through handles stored in the game's uniform buffers (`fp_t_cb7_2A`: word 0x2A of c[7] = uniform slot 4), the binding kinds `tex_ubo` / `samp_ubo`.
- Many shaders share a control section and differ in code, so keys hash both.
- All 71 compute shaders have a header (magic `0x12345679`) and zero padding, instructions at +0x100. Fed from +0x30 at first, 12 compute shaders (particles, post effects) translated into an endless loop that hung the GPU at the first gameplay scene.
- 2 agl post-effect compute shaders still produce SPIR-V that SPIRV-Cross cannot parse (7 before the compute start-offset fix).
- Integer vertex inputs: 529 of 4,854 vertex shaders are re-translated with `uint` inputs (skinned models read bone indices as integer bits; without it every character was stretched into streaks).
- Pairs: 331 of 9,043 vertex / pixel pairs failed to link before `vs_pad` (the opening course used none of them; later levels did, about 11,500 failed builds in one 11-minute session). The offline link check: 8,712 linked / 331 failed before, 9,043 / 0 after, 14,603 / 0 for the v0 + 1.2.1 tables.
- Vertex format `RGB10A2SN` (0x69) → `MTLVertexFormatInt1010102Normalized`.

## Audio

- The `sdk` carries the audio renderer (SDK 16): the host switches it to CPU rendering with manual execution and drives it from a guest thread, no IPC ([Running the SDK's own renderer on the CPU](../../../docs/reference/audio.md#running-the-sdks-own-renderer-on-the-cpu)).
- **Resolved:** Mario updates the renderer twice in a row at start-up; with a real audio device the first frame came after both, and the light limiter later crashed on a null state pointer (`ApplyLightLimiterEffect`, `sdk+0x228494`). The `AudioRenderSystem::Update` hook renders a catch-up frame first when nothing was rendered since the last update.

## Services and survey

- `main` imported 1,107 symbols, 692 of which D2R never imported (`nn::audio` 197, `nn::hid` 48); 172 of the host's 186 hook names then existed in its `sdk`. Its SVC set was a subset of D2R's.
- NVN entry points: the loader (`main+0x1070910`) calls `GetProcAddress` per name and stores through the GOT (`nvn_slots_got.py`). It fetched 534 functions and called 236; 79 were new to the host (CommandBuffer 31, Texture 15, Buffer 10, Sampler 10).
- Its device setup checks `nvnDeviceGetInteger(API_MAJOR/MINOR)` and returns silently on a failed check.
- Files: `System/Resource/ResourceSizeTable.*.rsizetable` (RESTBL), `Font/*.bfarc`, `Mals/<Region><lang>.sarc`, `Voice/Resource/<Region><lang>`; `*.zs` is plain zstd with no dictionary; BYML version 7.
- Fonts: SDK 16 and `main` imports `ScalableFontEngineHelper::Decode`, so the shared font is served as BFTTF (marker + size + TTF). Controller-button icons in text come from the Nintendo-extension font (`GameMsg/Key.msbt`: U+E0E0–E0F0, U+E0AB–E0AE, U+E090–E097, U+E100/E101), served from `make_ext_font.py`'s `nintendo_ext.ttf`.
- **Resolved:** a crash indexing a per-core table with index −2 in Mario's font engine (`main+0x107dbc0`): threads created with ideal core −2 now resolve to the process default core.
- Focus handling mode 2 (focus notifications are sent). Mario never throws C++ exceptions.
- Ghidra: `main` (28 MB) took about 35 minutes to analyse, 138k functions.

## Mac routes

- `DM_KEYS="12:A,14:A,20:A,26:A,32:A"` reaches the opening course.
- v0 with a fresh save, step by step through `DM_CTL`: `L+R~0.3` at the title, then `A` (New Game), `A` (one player), `LUp~0.2` and `A` (a character), `A` (OK); about 15 s later the course's opening dialogue, where `shot` lines every 0.35 s gave the black-block counts in [Rendering](#rendering).
- 1.2.1 from [#377](https://zysv.wtpuscm.cn/yinqing/deadline-826759.html)'s save (World 1, at the Badge House on the map), through `DM_CTL` from about 35 s after start: `L+R~0.3`, `A` (the user), `A` (one player), `A` (Mario), `A` (OK); on the map `A` enters the Badge House, then `LLeft~0.8` and `LRight~0.9`, and the Poplin's dialogue opens; screenshots every 0.3 s there gave the 1.2.1 counts.

## Found on this game

General host changes first needed here; each applies to every game:

- **Resolved: Mac launcher metadata matches native defaults**: [A Mac front end's default settings change Metal execution paths](../../../docs/reference/packing.md#a-mac-front-ends-default-settings-change-metal-execution-paths), found by metadata generation and source inspection; no gameplay measurement.
- **zstd files inside a RomFS image** (`shader_split.py` `unzs`): [The zstd command given a path inside the image](../../../docs/reference/packing.md#the-zstd-command-given-a-path-inside-the-image).
- **Function matching for opt-in cuts across updates**: [Function-matched cuts](../../../docs/reference/rewriter.md#function-matched-cuts-for-unregistered-builds). Shared rewriter support first verified on Wonder; its heap workaround and enrollment have since been removed.
- **NVN on Metal basics** in [rendering.md](../../../docs/reference/rendering.md): [8-bit index widening](../../../docs/reference/rendering.md#8-bit-index-buffers), [texture swizzles](../../../docs/reference/rendering.md#texture-swizzle-through-swizzled-metal-views) (`nvn_fmt_to_mtl`, `img_fit`), [format-changing render-target views](../../../docs/reference/rendering.md#render-target-views-that-change-the-format) (`attach`, `present_raw_src`), [storage-shared transient targets](../../../docs/reference/rendering.md#transient-render-targets-shared-by-storage), [inline uniform updates as in-order copies](../../../docs/reference/rendering.md#inline-uniform-updates-as-ordered-gpu-copies) snapshotted at encode (`mtl_sub_update`), [CPU-written linear textures](../../../docs/reference/nvn-objects.md#linear-textures-in-cpu-visible-pools) on the pool buffer (`tex_linear`), [integer vertex inputs](../../../docs/reference/rendering.md#integer-vertex-inputs) (`int_inputs`, `vfmt_for_ct`), [intra-encoder barriers](../../../docs/reference/rendering.md#metal-4-does-not-order-operations-inside-one-encoder) (`blit_buf`, `DM_NVN_BLIT_ORDER`), [zero-filled new textures](../../../docs/reference/rendering.md#new-textures-start-as-zero) (`tex_zero_fill`), [scissored zero clears](../../../docs/reference/rendering.md#scissored-colour-clears) (`clear_rect_zero`), [vertex output padding from shader pairs](../../../docs/reference/shaders.md#pixel-shader-inputs-the-vertex-shader-does-not-write) (`vs_pad`, `shader_pairs.tsv`), [virtual pools aliased onto one physical pool](../../../docs/reference/nvn-objects.md#virtual-memory-pools-mapped-onto-one-physical-pool).
- **Binding layout and handles in uniform buffers** (`NVN_RB_TEXU` / `NVN_RB_SAMPU`): [04, section 3](../../../docs/agents/04-graphics.md#3-find-the-binding-layout-by-census); the compute code start: [Compute code with and without a header](../../../docs/reference/shaders.md#compute-code-with-and-without-a-header).
- **The SDK's audio renderer in-process** and the catch-up frame between two updates: [Running the SDK's own renderer on the CPU](../../../docs/reference/audio.md#running-the-sdks-own-renderer-on-the-cpu), [At least one rendered frame between two updates](../../../docs/reference/audio.md#at-least-one-rendered-frame-between-two-updates).
- **Compact buffer bindings keep model heaps within their budgets**, **memory partition** (`mem_init`), **virtual NVN pools**: [Model heaps sized by command records](../../../docs/reference/memory.md#model-heaps-sized-by-the-hosts-command-record-sizes), [Partitioning the guest block between alias region, stacks and heap](../../../docs/reference/memory.md#partitioning-the-guest-block-between-alias-region-stacks-and-heap), [Virtual memory pools mapped onto one physical pool](../../../docs/reference/nvn-objects.md#virtual-memory-pools-mapped-onto-one-physical-pool).
- **Update imports diffed before the first run**, **fixed-address code gated by `module_t.build_id`**, **BFTTF shared font** (`hle_sdk16.c`), **the extension font**, **`GetCommandMemoryUsed` counting host segments**, **`NVNcommandBuffer` +0x10** (`nvn_flag10`), **`prog_t` size**: [A game update imports SDK calls the base game did not](../../../docs/reference/services.md#a-game-update-imports-sdk-calls-the-base-game-did-not), [Code inherited from another title at fixed addresses](../../../docs/reference/services.md#code-inherited-from-another-title-at-fixed-addresses), [Shared font address: BFTTF header or bare TTF](../../../docs/reference/services.md#shared-font-address-bfttf-header-or-bare-ttf), [Nintendo-extension glyphs for button icons](../../../docs/reference/services.md#nintendo-extension-glyphs-for-button-icons), [What `GetCommandMemoryUsed` counts](../../../docs/reference/nvn-objects.md#what-getcommandmemoryused-counts), [Inlined SDK code writes into `NVNcommandBuffer` +0x10](../../../docs/reference/nvn-objects.md#inlined-sdk-code-writes-into-nvncommandbuffer-0x10), [Host records must fit the game's SDK object sizes](../../../docs/reference/nvn-objects.md#host-records-must-fit-the-games-sdk-object-sizes).
- **Exclusive windows** (fall-through exit first, Mario had 82): [Exit stub order in exclusive windows](../../../docs/reference/rewriter.md#exit-stub-order-in-exclusive-windows). **Ideal core −2**: [Ideal core −2 resolves to the process default core](../../../docs/reference/threads.md#ideal-core-2-resolves-to-the-process-default-core).
- **Present-interval holds** (`g_link_hold_until`, `nvn_game_fps`): [Frames of a dynamic present interval held on screen and at the acquire](../../../docs/reference/frame-rate.md#frames-of-a-dynamic-present-interval-held-on-screen-and-at-the-acquire).
- **Glitch detector before bisection**: [05, Intermittent rendering glitches](../../../docs/agents/05-debugging.md#intermittent-rendering-glitches); **offline pipeline link check**: [05](../../../docs/agents/05-debugging.md#offline-pipeline-link-check).

## Open

- On the Mac, Mario's menu and dialogue text render blank (fine on the iPhone), per development.md's known gaps.
- dm/games/README.md says the heap check turns "Wonder's 1.44 GB" request into a 0xc0000000 alias region; 03 says it asks for 1.47 GB, and the profile's alias region is 0xB8000000.
- Several addresses in 02 and 03 (`main+0x1070910`, `main+0x107dbc0`, `main+0x4da5c`, `main+0xb8dec`, `main+0xcfd24`, `main+0x36ceff0`) do not name their build; the present-interval ones are from 1.2.1.
- 2 agl post-effect compute shaders do not translate.


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

