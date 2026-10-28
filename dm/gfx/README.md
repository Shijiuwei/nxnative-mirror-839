# dm/gfx — graphics layer (NVN → Metal)

The Switch version of the game draws with NVN (the Switch's native graphics interface). The graphics layer reimplements this interface on top of Apple's Metal: every NVN call the game makes lands here, is recorded as a command, and is replayed as Metal commands at submit time. The game's shaders are not translated at run time; they are translated offline, once, on a computer, into Metal shader libraries (when the computer has no Apple Metal compiler, the final compile step moves to the phone; see below).

For the build steps of the whole project see [README.md](../../README.md) and [docs/development.md](../../docs/development.md) (the tools they need: Xcode, xcodegen, .NET SDK, zstd, and the Python packages lz4 / capstone / fonttools; spirv-cross is the in-repo build `tools/spirv_cross_bin`); this page covers only the graphics layer's own matters.

## How it works

### Taking over the NVN interface

- The game's main program imports only one NVN symbol, `nvnBootstrapLoader`; all other NVN functions are obtained through it by name. The host forcibly takes over this import (`nvn/hle_nvn.c`), and not a single instruction of the original graphics driver is executed.
- The game fetches the addresses of 523 names in total (`nvn/names.txt`). Each name has a thunk (`nvn/nvn_thunks.S`, a generated file): it first increments that function's call count and then jumps to the implementation.
- The functions games call have implementations, according to NVN semantics or empty (`g_nops` in `hle_nvn.c`); the rest are connected to a red-reporting stub that, when called, prints `[RED] NVN unimplemented function called: <name>` in the log.
- Every implementation gets a compile-time signature check against `nvn/nvn_protos.h` (a generated file). The calling conventions of Apple arm64 and the Switch differ for integer parameters narrower than 32 bits, so on the host side `NVNboolean` parameters are always received as `uint32_t` and only the low 8 bits are examined.

### Two layers: record and replay

- **Interface layer `nvn/hle_nvn.c`**: host-side objects are stored in place in the memory the game allocates for NVN objects; the layouts are in `nvn/nvn_obj.h`, and each has a static assertion guaranteeing it does not exceed NVN's reserved size. Commands are recorded into the command memory provided by the game, in the record format of `nvn_obj.h`.
- **Replay layer `nvn/nvn_mtl.m`**: at submit time it replays the record stream as Metal commands.
  - Memory pools are created as zero-copy `MTLBuffer`s, with the GPU address equal to the CPU address, so only unified-memory devices are supported.
  - The visible effects of "GPU completion" (sync objects becoming ready, events writing their value, counters) happen only after the Metal command buffer has actually completed.
  - Pipeline states and depth-stencil states are cached by state hash; at startup, shaders and pipelines are pre-built according to the warm-up manifests (the bundled seed `pso_manifest.tsv`, then this device's `pso_user.tsv`; see below): with a window, on every core before the game's first frame behind a "Preparing shaders" screen (docs/development.md, "The prepare step"), else in the background. A pipeline that fails to build is cached as failed and reported once with its shader keys.
  - Render pipelines are made through `MTL4Compiler` from a base whose colour state is left unspecialized, one base per pipeline less its colour state, specialized per colour state; the bases are kept in the per-game pipeline archive (`pso_archive/` beside `pso_user.tsv`) and taken from it at the next start (`pso_build`; see docs/development.md, "Pipeline bases and the pipeline archive"). A draw whose pipeline is not built yet draws with its shader pair's pulling pipeline (no vertex descriptor) while its own is built in the background (docs/development.md, "Hybrid pipelines").
  - The game's packaged textures are in Tegra block-linear layout and are deswizzled before upload (see `tex/README.md`).
  - Presentation: the game's swapchain texture is drawn to a `CAMetalLayer`, and presentation timing is scheduled by the pacer in `nvn/nvn_pace.h`.

For which Metal object each NVN object and command lands on, see [nvn_metal_mapping.md](nvn_metal_mapping.md).

### Shader translation chain

The game's shaders are GPU machine code for the Tegra X1 (the Maxwell instruction set). The translation from this machine code to Metal Shading Language is not done on the phone; all of it happens offline:

```
Original Switch shader package
  → tools/dmsh_xlate        produces SPIR-V with Ryujinx's shader translation library (a C# tool)
  → spirv-cross             converts SPIR-V to Metal Shading Language (MSL); the in-repo build tools/spirv_cross_bin
  → xcrun metal / metallib  compiles into Metal shader libraries
```

The whole chain is driven by `tools/dmsh_build.py`; `scripts/build_ipa_from_nsp.py` runs it as part of packing an IPA. Key points:

- **Key**: each shader is keyed by the first 16 hexadecimal digits of the sha256 of its control section (the first 0x880 bytes, or the section's own length at control +0x20 when that is shorter, as in SDK 10's 0x850-byte sections) followed by its code area (many of Mario's shaders share a control section and differ in code). At run time the game hands both to `nvnProgramSetShaders`, and the replay layer computes the same key to find the library.
- **Build outputs**: `<key>_b.metallib` (the shader library), `<key>_b.json` (reflection used by replay: each binding's kind, slot number, index within the argument buffer and so on), `<key>_b.metal` (the MSL source). Some fragment shaders additionally have `<key>_bf.metallib`, a "readable tile" version used for pass merging (see the file header of `tools/dmsh_fetch.py`). `_b_build.json` in the directory records the success or failure of each key in this run.
- **`dm/games/<id>/shader_map.tsv`**: the shader map, one row per shader variant. At run time only its "key → stage (vertex / fragment / compute)" is used.
- **`dm/games/<id>/shader_facts.tsv`**: the answer table for translation queries. The translator needs to know some facts that cannot be read from the machine code (the dimension type of a texture, image formats, the component types of vertex attributes and fragment outputs, a compute shader's thread group size and shared memory byte count), and this table gives the answers by key. The table ships with the repository, and building the shader libraries only reads it; the table is maintained by `tools/dmsh_facts.py`, whose file header has the details.
- **Translation chain version and the `b-<V>/` directory**: the version V is a string of 12 hexadecimal digits, obtained by hashing together the contents of the source files involved in the translation chain, the translation options and the tool fingerprints (`tools/dmsh_chain.lock`); `tools/dmsh_chain.py` computes it. Shader libraries are written into `<output directory>/b-<V>/`, and every reflection json also records V. When the host is compiled it takes V from `nvn/dmsh_chain.h`, one version for every game, loads only from `b-<V>/`, and checks the version in each json one by one; on a mismatch it reports red and does not load. So after changing any file on the chain, or changing a tool, the libraries must be rebuilt and the host recompiled. A profile's inputs (its shader tables and `shader_*` / `nvn_*` keys) are not part of V: they are in each shader's cache key, so a change rebuilds the shaders it affects, and a full build stamps the directory with the game and their digest (`b-<V>/inputs.tsv`, `dmsh_chain.py inputs`). A full build starts from no shader outputs and a partial one from none of the keys it builds, so a directory holds only what its last build made; it refuses a directory stamped for another game, so each game needs its own output directory.
  - `dmsh_chain.py`: prints V.
  - `dmsh_chain.py --check-tools`: checks that the local tools match the lock file.
  - `dmsh_chain.py --update-lock`: rewrites the lock file according to the local tools.
  - `dmsh_chain.py --header`: rewrites `nvn/dmsh_chain.h` (commit it with the chain change; `dm/tools/tests/test_nsp_chain.py` fails on a stale one).
  - `dmsh_chain.py --inputs`: prints the digest of the current profile's inputs (`DM_GAME`), as a full build stamps it.
  - The converter is `tools/spirv_cross_bin/spirv-cross`, built by `tools/spirv_cross_bin/build.sh` from a pinned upstream commit. `SPIRV_CROSS=` selects another executable, which must match the lock for NVN builds. Changing the pin requires rebuilding it, `dmsh_chain.py --update-lock` and `--header`. Moving to another Xcode also requires updating the documented tested version.
  - `tools/own_compilers.py <dmsh_build.py arguments>` requires the translator, SPIRV-Cross and both Metal compilers to match the lock. Another Metal compiler stops the build with installation instructions; `DM_ALLOW_OTHER_METAL=1` permits it and prints the actual version. Issue #242 reported corruption with an untested compiler, but the compiler's role remains unconfirmed. The override changes cache keys without changing the lock or chain version.
- **`dm/games/<id>/c1_full.tsv`**: the index list for the shaders' built-in constant section (c1), generated by `tools/c1_guard.py` scanning the translated MSL; it lists the shader keys whose "run-time index could not be proven bounded". An index counts as proven only when interval arithmetic over the whole expression (every variable followed to its single definition; literals, `& K`, `% K`, shifts, `min`, `max`, `clamp` and casts bound it) keeps it non-negative and inside the span a shared bank gives that shader: its c1 data rounded up to 256 bytes plus the 256-byte zero gap. Every stage counts (`vp_`, `fp_`, `cp_` …). Its first line records the `b-<V>` and the inputs stamp of the library it was made from. For the shaders on the list the replay layer allocates a separate whole 64 KiB buffer for c1; when the list is missing or does not name the loaded library's version and stamp, it reports red and conservatively treats all shaders this way. The list is not committed (git ignores it): each checkout makes it from its own build's Mac outputs (a full `dmsh_build.py` run does).
- **`dm/games/<id>/cb_extent.tsv`**: how far into each uniform slot (cbuf binding) every shader can read, generated by `tools/cb_extent.py` from the translated MSL with `c1_guard.py`'s interval arithmetic: per shader key, `reg:bytes` for each slot whose reads are all proven to stay inside its first bytes, and `s<reg>` for each storage slot the shader never writes (SPIRV-Cross types every storage buffer Ryujinx emits as writable, so the host counts the others as written). When the game updates part of a bound uniform block inside a render pass, the replay layer copies only that much of the binding to give the draws after the update their data without ending the pass (`nvn_mtl.m` `ubp_get`; Breath of the Wild binds a 64 KiB block and its vertex shaders read at most 544 bytes of it). Its first line records the `b-<V>` and inputs stamp, like `c1_full.tsv`'s; when it is missing or stale, bindings are copied whole (up to 64 KiB). Like `c1_full.tsv`, it is not committed, and a full `dmsh_build.py` run makes it.
- **`dm/games/<id>/shader_pairs.tsv` and `pso_pairs.tsv`**: the (vertex key, pixel key) of every BNSH variation (Unity: of every program) and agl BAHS program variation, written by `tools/shader_split.py`. Metal only builds a pipeline when every pixel shader input is a vertex shader output, while Maxwell gives an unwritten attribute a default; `dmsh_build.py` therefore adds the outputs a vertex shader's partners read but it doesn't write, set to (0, 0, 0, 1) (`vs_pad`). The profile tables are part of its inputs digest and the per-shader cache keys. A profile without `pso_pairs.tsv` takes the generated table beside the shader package.
- **Several game versions**: `shader_split.py` takes comma-separated RomFS and `main` lists (base game first) and writes the union; the map, facts and pairs tables currently cover Mario v0 and 1.2.1.
- **`dm/games/<id>/pso_manifest.tsv`**: the warm-up seed manifest bundled with the app: shaders (`S`), render pipelines (`R`, the hex of `pso_skey` in `nvn/nvn_pso_skey.h`) and compute pipelines (`C`) to pre-build at startup. The row order is the pre-build order (first use); do not sort it. Its companion is `pso_user.tsv`, which the host appends next to the save directory whenever it has to build something on the spot, and which is pre-built after the seed at the next launch. Both name shaders by key and describe pipeline settings only, so a list recorded on one device works on any other; lines a build cannot use are skipped. Recorded lines can be appended to the seed below its header (currently empty).
- **Shaders embedded in the program itself**: the main program additionally contains 4 compute shaders that are not in the shader package; they were hand-written as MSL from the disassembly and are in `nvn/nvn_builtin.h`.

Binding conventions followed during translation (the NVN driver's layout, the same in every game: `tools/dmsh_build.py`, filled at run time by `nvn/nvn_mtl.m`):

- uniform buffer r is constant section `c[3 + r]` in the machine code;
- the handle of combined texture r (`nvnCommandBufferBindTexture`) is at `c[2][0x20 + 8r]`, of image r at `c[2][0x120 + 8r]`, of separate texture r at `c[2][0x168 + 8r]` (the host's texture slot 128 + r) and of separate sampler r at `c[2][0x568 + 8r]`;
- the address of storage buffer r is at `c[0][base + 16r]`, with base: vertex 0x110, compute 0x310, fragment 0x510;
- the window origin is at the top left and the depth range is 0 to 1, the same as Metal, so nothing is flipped during translation.

## Directory structure

| Location | Contents |
|---|---|
| `nvn/hle_nvn.c` | Interface layer: NVN takeover, object implementations, command recording, presentation pacing |
| `nvn/nvn_mtl.m`, `nvn/nvn_mtl.h` | Replay layer: Metal device, memory pools, textures, shader loading, pipelines, drawing, presentation |
| `nvn/nvn_obj.h` | Host-side object layouts and the command record format, shared by both layers |
| `nvn/nvn_entry.S`, `nvn/nvn_thunks.S` | Entries for the unimplemented stub and the timing wrapper, per-function thunks |
| `nvn/nvn_tex.c`, `nvn/nvn_tex.h` | Storage size, alignment and per-level offsets of block-linear textures (ported from deko3d) |
| `nvn/nvn_dirty.c`, `nvn/nvn_dirty.h` | Tracks dirty pages of game memory with write protection, to decide whether texture contents have changed and need re-uploading |
| `nvn/nvn_refl.m`, `nvn/nvn_refl.h` | Converts shader reflection json into fixed-length C arrays, so that drawing only walks arrays |
| `nvn/nvn_hmap.h`, `nvn/nvn_cover.h`, `nvn/nvn_pace.h`, `nvn/nvn_pso_skey.h`, `nvn/nvn_primitive.h` | Hash table, geometric proof of full-coverage draws, presentation pacer, pipeline manifest key, indexed triangle fans and quads as triangle lists (primitive restart kept) |
| `nvn/nvn_builtin.h` | Hand-written MSL for the 4 compute shaders embedded in the main program |
| `nvn/include/nvn.h` | NVN interface declarations (third party; for the license see the file header) |
| `nvn/names.txt` | The NVN API name list. The per-game tables described above live in the game profile `dm/games/<id>/` (named after the base game's title ID; tools pick it with `DM_GAME`, make with `GAME`; there is no default game) |
| `tex/` | Texture format mapping, deswizzling, BC decompression, see [tex/README.md](tex/README.md) |
| `vk/hle_vk.m` | Vulkan on Metal, for games that render with Vulkan (Hades, Super Mario Galaxy); their SPIR-V shaders are translated by `tools/vk_shaders.py` |
| `gl/hle_gl.m` | OpenGL through EGL on Metal, for games that render with NVIDIA's GL driver (Digimon Story Cyber Sleuth: Complete Edition; Balatro, LÖVE on its core-profile paths); their GLSL sources are translated by `tools/gl_shaders.py` (glslang, then SPIRV-Cross) |
| `metal/` | A standalone minimal Metal offscreen example, not compiled into the host, see [metal/README.md](metal/README.md) |
| `tools/` | Offline tools, see below |
| `tests/` | Unit tests |
| `hle_vi.c`, `hle_nv.c` | Glue layer for display service and graphics driver initialization: returns host static objects or does nothing |

The scripts in `tools/`; usage is written in each file's header:

| Script | Purpose |
|---|---|
| `dmsh_xlate/` | The shader translation tool (C#), which references the translation library in `dm/third_party/ryujinx` |
| `dmsh_build.py` | Drives the whole translation chain to produce the shader libraries; when the environment variable `IOSOUT=directory` is set, it also produces the iOS version |
| `dmsh_chain.py`, `dmsh_chain.lock` | Translation chain version and tool fingerprints |
| `dmsh_facts.py` | Reads the answer table `shader_facts.tsv` (which `shader_split.py` writes) |
| `dmsh_fetch.py` | Generates the "readable tile" versions of fragment shaders |
| `c1_guard.py` | Generates `c1_full.tsv` |
| `cb_extent.py` | Generates `cb_extent.tsv` |
| `dmsh_dryrun.py` | Dry-run conversion of the whole library, measuring only "can it be converted" |
| `gen_protos.py` | Generates `nvn_protos.h` / the thunks |
| `tbdr_budget.py`, `tbdr_load_dead.py`, `tbdr_store_dead.py`, `tbdr_pass_merge.py` | Read the `DM_NVN_PASSDUMP` log and analyze the load / store volume of each render pass's attachments per frame |
| `passgpu_top.py` | Reads the `DM_NVN_PASSGPU` output and lists GPU time by pass |

## Building

The graphics layer has no separate library. `dm/host/Makefile` compiles `dm/gfx/*.c` and the source files under `dm/gfx/nvn/` and `dm/gfx/tex/` into the host `dmhost` by wildcard; the iOS App is built by `dm/ios/tools/build_ipa.sh`, which also copies the map, `pso_manifest.tsv`, `c1_full.tsv` and the `b-<V>/` shader libraries into the App. `tests/` and `metal/` do not go into the host.

The shader libraries are generated by `tools/dmsh_build.py` (the NSP script runs it; for building by hand see [docs/development.md](../../docs/development.md)):

- the Mac version is in `$DM_ROOT/build/cache/metal_shader_cache/b-<V>/`;
- the iOS version is in `$DM_ROOT/build/ios/dmsh_b/b-<V>/`.

After changing a file on the translation chain, the order for rebuilding manually is: `dmsh_chain.py --header` (once, for every game) → `dmsh_build.py` (a full build also makes `c1_full.tsv` and `cb_extent.tsv`, local tables, not committed) → recompile the host.

The per-shader cache key in `dmsh_build.py` (`shader_key`) covers the profile's `shader_*` rows and the shader's pairs, and a paired vertex shader's key also takes its pixel shaders' keys (`pair_keyed`), since `vs_pad` reads their MSL. Before, these were only in the chain version, so a build after changing only them hit the old per-shader entries and filed them under the new version; now the chain version leaves them out altogether. `$DM_ROOT` defaults to `~/dm-work`.

## Tests

```sh
make -C dm/gfx/tests run        # graphics layer unit tests
make -C dm/gfx/tex/tests run    # texture submodule unit tests; the real-data items need texture samples from both the Switch and PC versions, see tex/README.md
make -C dm/gfx/metal run        # offscreen example self-test, see metal/README.md
```

The tests in `dm/gfx/tests` compile the production code in directly and need a Metal device on the local machine; a few of them run under `MTL_DEBUG_LAYER=1` (the Metal validation layer). Most tests do not read the workspace; the exceptions are `t_refl` and `t_rt_range_ro`, which read the translated shader reflection json under `$DM_ROOT/build/cache/metal_shader_cache`, and `test_tbdr_budget.py`, which creates a temporary directory under `$DM_ROOT/run/gfx`. Main contents:

| Test | What it checks |
|---|---|
| `t_nvn` | Interface layer: fetches addresses from `nvnBootstrapLoader` the way the game does, all the way through to presentation |
| `t_builtin` | The 4 embedded compute shaders |
| `t_refl`, `t_hmap`, `t_cover`, `t_pace`, `t_pso_skey`, `t_dirty_slots` | Reflection parsing, hash table, full-coverage proof, pacer, pipeline manifest key, dirty page tracking |
| `t_pso_warm` | Historical manifest normalization and deduplication, draw-time cache reuse and GPU pixels for normal and merged passes, through MTLDevice and Metal 4 |
| `t_deswz_gpu` | GPU deswizzling |
| `t_present_crop` | Presentation cropping |
| `t_mip_extent` | Effective size when rendering to a mip level other than level 0 |
| `t_fuse_check`, `t_reorder_chain`, `t_defer_chain`, `t_zskip`, `t_xv`, `t_ub`, `t_pfv`, `t_rt_range`, `t_rt_range_shape`, `t_rt_range_ro`, `t_rt_depth_ro` | Compile the replay layer `nvn_mtl.m` in directly and check the various optimization and diagnostic logic inside the replay layer |
| `test_tbdr_budget.py` | The parsing in `tools/tbdr_budget.py` |

## Common run-time switches

All are environment variables. The iOS App sets `DM_NVN_SHMAP` and `DM_MSL_DIR` itself at startup.

Resource locations:

| Switch | Effect |
|---|---|
| `DM_NVN_SHMAP=path` | The shader map. The default is `../games/<id>/shader_map.tsv` relative to the executable's directory; `c1_full.tsv` and `pso_manifest.tsv` are taken from the same directory |
| `DM_MSL_DIR=directory` | The parent directory of the shader libraries; the host loads from `b-<V>/` beneath it. When running the host on a Mac, set it to `$DM_ROOT/build/cache/metal_shader_cache` |
| `DM_PSO_MANIFEST=path` or `off` | Use a different warm-up manifest, or turn warm-up off |
| `DM_PSO_PREPARE=0` or `1` | `0`: the warm-up runs in the background while the game starts, as without a window; `1`: the game waits for it before its first frame also without a window (`DM_PSO_PREPARE_THREADS=<n>` workers; docs/development.md, "The prepare step") |
| `DM_PSO_RECORD=path` | Appends the shaders and pipelines used in this run to a file, for re-recording the warm-up manifest |
| `DM_PSO_USER=path` | Where to keep the per-device list of shaders and pipelines built on the spot (default: `pso_user.tsv` in the parent of `DM_SAVE`) |
| `DM_PSO_ARCHIVE=folder` or `off` | Where to keep the per-game pipeline archive of pipeline bases (default: `pso_archive/` beside `pso_user.tsv`), or no archive |
| `DM_PSO_MTL4=0` / `=1` | `0`: build render pipelines through the `MTLDevice` as before, with no unspecialized bases and no archive (the Mac's default is on). `1`: turn them on on iOS, where they are off by default |
| `DM_PSO_HYBRID=0` | A draw whose pipeline is not built yet waits for it, instead of drawing with the pair's pulling pipeline meanwhile (docs/development.md, "Hybrid pipelines") |
| `DM_PSO_PREBUILD=0` or `tool` | No prebuild of the pulling bases into the archive (in the prepare step, or in the background without one); `tool`: run the prebuild alone and exit (`DM_PSO_PREBUILD_DEPTH=zero`, `DM_PSO_PREBUILD_THREADS=<n>`) |
| `DM_PSO_PAIRS=path` | The prebuild's second pair table (default: `pso_pairs.tsv` beside the shader map) |
| `DM_PSO_BUNDLE=folder` or `off` | The pipeline archive made offline (`dm/gfx/tools/pso_offline.py`): `<GPU architecture>.mtl4a` and `prebuilt.tsv`, read only; when it covers the shader pairs the device builds no prebuild bases. The iOS app sets it to its `pso/` folder when it has one (docs/development.md, "The pipeline archive made offline") |
| `DM_PSO_WAITLOG=1` | Log each draw that waited more than 20 ms for its pipeline, with why no stand-in served it |
| `DM_PSO_PULL_ALL=1` | Diagnostic: every draw that has a pulling pipeline draws with it |

Logging and capture:

| Switch | Effect |
|---|---|
| `DM_NVN_LOG=1` | Logs the creation of NVN objects |
| `DM_NVN_FRAMES=N` | Prints the call table frame by frame for the first N frames (default 3) |
| `DM_NVN_TSV=path` | Writes the per-frame call table and timings as TSV; adding `DM_NVN_TSV_CALLS=0` writes only the timing rows |
| `DM_SLOWFRAME=1` | Writes the timing breakdown into the log only for the frames that dropped |
| `DM_NVN_SHOT=frame[,frame…]:path` | Saves the swapchain texture as an image at the specified frames; `%d` in the path is replaced by the frame number |
| `DM_NVN_SHOT_OUT=WxH` | Makes the screenshot above capture the picture after upscaling to this resolution instead |
| `DM_NVN_TEXDUMP=frame:path` | Saves each color render target used in that frame as an image |
| `DM_NVN_PASSDUMP=frame` or `start-end` | Writes each render pass's attachments, samples and presentation for these frames to standard error entry by entry, for analysis by `tools/tbdr_*.py`; additionally setting `DM_NVN_PASSDUMP_ALL` logs every draw |
| `DM_NVN_PASSGPU=file` | Per-pass GPU time (requires device support for counter sampling at stage boundaries) |
| `DM_NVN_PROF=1` | Wraps the implemented NVN functions in a timing wrapper and collects the time each function takes during recording |

Comparison and testing:

| Switch | Effect |
|---|---|
| `DM_NVN_TBDR=1` | Turns on the pass optimizations of the replay layer (store / load elision, pass merging). They are off by default: they infer which textures a draw reads from its bound slots, which bindless textures defeat ([docs/agents/04-graphics.md](../../docs/agents/04-graphics.md) section 4) |
| `DM_NVN_UNWRITTEN_DC=0` | With `DM_NVN_TBDR=1`, turns off the single item "attachments not written in this pass are not stored" |
| `DM_NVN_FUSE_CHECK=1` | Same-frame check of pass merging; logs the comparison result every time |
| `DM_NVN_CS_HOIST=0` | A compute dispatch between the draws of a pass ends it (by default it runs ahead of the pass when that changes nothing the draws before it read: [reference](../../docs/reference/rendering.md#compute-dispatches-between-the-draws-of-a-pass-run-ahead-of-it)) |
| `DM_NVN_SLICE_MERGE=0` | Passes into other layers of one array or cube texture level stay apart (by default they run as one layered pass: [reference](../../docs/reference/rendering.md#passes-into-other-layers-of-one-texture-level-run-as-one-layered-pass)) |
| `DM_FPS=30` or `60` | Pins the target frame rate |
| `DM_NVN_BARRIER=game`, `DM_NVN_CUT` | Pass ordering and segment commits: see [docs/agents/04-graphics.md](../../docs/agents/04-graphics.md) section 7 |
| `DM_PRESENT_HZ=N` | The frequency of presentation pacing (default 60); 0 means no frame limit and is for performance profiling only |
| `DM_PRESENT_UPSCALE=metalfx` or `off` | How the present scales a picture smaller than its rectangle on the screen: `metalfx` through MetalFX's spatial scaler, `off` (the default) bilinearly. An option on both platforms (the iOS launcher's Upscaling; `env.txt` on the Mac); rendering at the screen's resolution stays the reference for sharpness ([reference](../../docs/reference/rendering.md#the-present-upscales-through-metalfx)) |

## Known limitations

The following paths are not implemented and report red in the log when reached (`RED_ONCE` in `nvn/nvn_mtl.m`):

- clears in the middle of a render pass, and color clears with a partial mask or partial scissor;
- indexed quad primitives (8-bit indices are widened to 16-bit at replay);
- some forms of texture-to-texture copy and texture clear.

Functions listed in `g_nops`, such as `MultiDraw*IndirectCount` and `SetDepthBounds`, are empty implementations: calling them neither reports red nor has any effect. `Downsample` / `DownsampleTextureView` are recorded as texture copies with a host flag (`DM_COPY_RESOLVE`) and replayed as a Metal multisample resolve.


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

