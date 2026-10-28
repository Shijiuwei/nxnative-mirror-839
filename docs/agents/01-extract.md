# 01 — Extracting an NSP

## Keys

- Needs the user's own `prod.keys` and the title key from the NSP's ticket. Keep both under the work folder, e.g. `$W/build/keys/.switch/{prod,title}.keys`, and run hactool with `HOME` pointed there so it never reads anything else:

  ```bash
  # $W/build/keys/ht — hactool with a private key home
  HOME="$(cd "$(dirname "$0")" && pwd)" exec "$(dirname "$0")/../../tools/hactool/hactool" --disablekeywarns "$@"
  ```

- Dumped `prod.keys` files may contain malformed lines (seen: 33 `*master_kek_source_*` lines with the wrong length) that make hactool refuse the whole file. Drop lines whose value length is wrong for the key type; keep a backup of the original.
- Title key: the NSP carries `<rights id>.tik`. For a common ticket the encrypted title key is the first 16 bytes of the title-key block at 0x180; append `<rights id> = <key hex>` to `title.keys`. Never print the value.

## NSP → NCA → ExeFS / RomFS

```bash
ht -t pfs0 --outdir=$W/build/nsp game.nsp > $W/build/nsp.log         # NSP is a PFS0: NCAs + cnmt + tik/cert
ht -t nca $W/build/nsp/<id>.nca | head -40                             # identify the Program NCA (content type Program, has ExeFS + RomFS)
ht -t nca --listromfs $W/build/nsp/<program>.nca > romfs_list.txt     # quick look before extracting 3+ GB
ht -t nca --exefsdir=$W/data/v0/ExeFS --romfsdir=$W/data/v0/RomFS $W/build/nsp/<program>.nca
```

- ExeFS holds `main.npdm`, `rtld` (usually), `main`, `sdk` and sometimes `subsdk*` modules (NSO). D2R and Mario both had only `main` + `sdk` worth loading.
- Update NSPs: the update's Program NCA is a patch NCA; extract it with `--basenca=<base program nca>` so RomFS is the merged view. Keep base and update in separate directories (`data/v0`, `data/v<ver>`); the rewriter and the host take one ExeFS at a time.
- Newer patch NCAs (seen: Mario 1.2.1, key generation 17) make hactool stop with `Invalid BKTR layout!`. `dm/tools/nx_extract` (C#, LibHac 0.19) handles them: `nx_extract <prod.keys> <base.nsp> <update.nsp> <out dir> [<base RomFS dir>]`. It reads the NCAs straight out of both NSPs (no 3+ GB intermediate NCA), imports the common tickets itself, prints the display version and module build IDs, and with a base RomFS dir writes identical files as APFS clones.
- After an update, find every address the host or rewriter hard-codes for the old build (`CUTS` / `OBS` / `FRAME_STEP` in the profile's `patches.py` or `dm/tools/sdk_patches.py`, `->base + 0x…` in the host) and carry it over with `dm/tools/nso_fnmap.py <old.orig.bin> <new.orig.bin> 0x…`: it matches functions with address bits masked and maps globals through their `adrp` references; check each result by disassembly (`sdis.py`). Also diff the imported SDK symbols: new imports that reach a system service need host implementations, or the real SDK aborts after connecting to `sm:`.
- **A cartridge dump (`.xci`) as the base game** (Monster Hunter Rise): the card header's magic `HEAD` sits at 0x100, and its root HFS0 holds the `update`, `logo`, `normal` and `secure` partitions; the game's NCAs are in `secure` (here the 7 GB program NCA, control, legal information, HTML manual and the meta NCA). Cartridge NCAs carry their key in the key area, so there is no ticket. `nx_extract` opens either kind as the base game (`OpenNsp`), and the NSP script takes `*.xci` beside `*.nsp`; the update and DLC stay NSPs.
- Decompressed NSO images for analysis: `dm/tools/nso.py <nso> <out.bin>` (lz4; use a venv that has `lz4`). A tool that reads a module's code loads it through `nso.load` / `nso.read` there, never a copy of its own. `dm/tools/nso2elf.py` makes an ELF for Ghidra / REA / objdump.

**A damaged NSP** (an incomplete copy, a broken download) shows as a hash error while the RomFS is read. `nx_extract` lists every file whose data fails the NCA's hash check and exits with code 3; `--allow-damaged` (the NSP script's `ALLOW_DAMAGED` / `--allow-damaged`) writes them as read and only lists them. To check a whole NCA, its file name is the first 16 bytes of its SHA-256: Super Mario Galaxy's base program NCA hashed differently, with 12 MB runs of zeros near its end (147 MB of damaged RomFS: the two WebM prologue movies), while its update and the small NCAs matched their names.

## Workspace layout used so far

The work folder `$W` sits in the game's NSP folder, beside the NSP script's `<title>_build/`, never in the home folder:

```
<NSP folder>/<title>-work/
  build/nsp/            NCAs + tik/cert + nsp.log
  data/v0/{ExeFS,RomFS} extracted base game (v<ver> for updates)
  probe/                decompressed images (main.orig.bin, sdk.orig.bin), small experiments
  elf/                  nso2elf output
  census/               survey tables (imports, svc, NVN slots)
  build/patched_v0/     rewriter output (what the host loads)
  shader/               shader package directory + map + facts produced by shader_split.py
  build/cache/metal_shader_cache/b-<chain version>/   translated metallibs
  build/gfx/dmsh_xlate/ the shader translator build
  ghidra/proj           persistent Ghidra project (analysis takes ~35 min for a 28 MB main)
  run/                  boot.sh, logs, saves, screenshots
  dbg/                  lldb scripts
```

## Compressed RomFS content

- Newer EPD titles store most files as `*.zs` (plain zstd, no dictionary in Mario; Zelda TotK needs `ZsDic.pack` dictionaries). Check for `*dic*` files before assuming plain zstd.
- Older EPD titles (Super Mario 3D World + Bowser's Fury, a Wii U port) store almost everything as `*.szs`: Yaz0-compressed SARC. `shader_split.py` decompresses Yaz0 itself (`yaz0`, pure Python, about 20–45 MB/s per core; the whole 4.7 GB decompressed RomFS scans in about a minute). One of its SARCs (`SystemData/LayoutShader.szs`) is big-endian (BOM `FE FF`), which the SARC reader does not handle; it holds no shader container the game uses.
- `*.sarc` / `*.pack` / `*.blarc` / `*.bfarc` are SARC archives; `*.byml` / `*.bgyml` are BYML (version 7 in Mario). A minimal reader for both is short enough to write inline when needed (see `shader_split.py` for SARC). Unity titles: `dm/gfx/tools/unityfs.py` reads UnityFS bundles, serialized files and objects by type tree (`bundle_files`, `serialized_objects`).
- Capcom's RE Engine (Monster Hunter Rise) keeps everything in PAK archives: `re_chunk_000.pak` plus `re_chunk_000.pak.patch_001.pak` … `patch_016` (19.6 GB in all). A PAK is `KPKA`, u8 version 4, u8 0, u16 features (0: a plain entry table), u32 count, u32 fingerprint, then one 0x30-byte entry per file: u32 murmur hashes of the lower- and upper-case UTF-16 path, u64 offset, u64 stored length, u64 length, u64 attributes (bits 0–3: 0 stored, 1 raw deflate, 2 zstd), u64 checksum. File names are only hashes. The game reads all of them as one view: a path in a later patch replaces the same hash in earlier PAKs (`kpka_jobs` in `shader_split.py` takes only the newest copy). The first bytes of each file name its format (`MESH`, `TEX\0`, `SDF\0` shader definitions, `BKHD` / `AKPK` Wwise, `USR\0`, `PFB\0`, …).
- Koei Tecmo's KTGL (Hyrule Warriors: Age of Calamity) keeps everything in RDB archives: `asset/<name>.rdb` is the index (`_DRK0000` header, then one `IDRK0000` record per file), and `asset/<name>.rdb.bin*` plus `asset/data/0x<KTID>.file` hold the data as a run of `IDRK0000` entries: u64 entry length, u64 stored length, u64 length, u32 0, u32 file KTID, u32 type KTID, u32 flags, a few parameter words, then the stored data at the end of the entry. Entries follow each other directly, some after padding to 64 KB. Flag 0x100000 = zlib in chunks: u32 stored length + a zlib stream each, up to 16 KB inflated per chunk, a zero length ends. The first bytes of the inflated data name the format (`GT1G` textures, `_M1G` models, `_S2G` shaders, `KTSR` / `TSRS` / `ASRS` sound). The DLC folders add `.rdb.bin3_0` files of the same layout.
- A newer RDB (ROMANCE OF THE THREE KINGDOMS 8 REMAKE, KTGL on SDK 17.5.3): `0002/root.rdb` and `0002/system.rdb` are the indexes (`_DRK0000`, with a `.rdx` beside each), and the data is in 296 `0002/0x<KTID>.fdata` files (12.2 GB) plus `0002/data/0x<KTID>.file` (and `jaJP/`, `zhCN/` copies). An `.fdata` starts with `PDRK0000` and u64 0x10, then the same `IDRK0000` entries (the u32 after the lengths is 1 or 0, the padding 16 bytes). The compression is new: flag 0x400000 = chunks of u16 stored length, 8 bytes, then a zlib stream; with 0x100000 as well (flags 0x500000) the streams are LZ4 blocks. Each chunk inflates to at most 16 KB and a zero length ends. Of the 51,655 entries, 49,620 are LZ4, 2,026 zlib and 9 stored; all decode to the length in their header (`shader_split.py` `idrk_data`). The shaders are `_S2G5000` / `6000` / `7000` (G2S) and `_DOK0000` containers again: 59,214 GLSLC outputs beside 149 in `main`.
- **An update can carry more than one program** (ROMANCE OF THE THREE KINGDOMS 8 REMAKE 1.1.3): a multi-program application has one Program and one Control NCA per program index, title IDs `…0000`, `…0001`, … Its update NSP holds the game's patch (`…0000`, 1.5 GB) and a whole second program (`…0001`, 34 MB, a full NCA, not a patch) with its own Control NCA; `main` starts it with `nn::oe::ExecuteProgram(1, …)`. `nx_extract` used to take the first Program NCA it found, the second program, and failed with `NCA section 1 does not exist` layering it over the base game; it now takes the lowest title ID of the base NSP (program index 0) and the update's NCAs of that title ID. The host runs program 0 only (`hle_oe.c` logs the call).
- Unreal Engine (Octopath Traveler 0, UE5: the paged global distance field shaders and the LowLevelTasks task system) keeps everything in pak files: `Octopath_Traveler0/Content/Paks/pakchunk0-Switch.pak` (1.5 GB, 159,757 files) next to loose CRIWARE `.awb` sound banks and one `.usm` movie. A version 11 pak has its footer in the last 0xdd bytes (encryption key GUID, encrypted-index flag, magic `0x5A6F12E1`, version, index offset and size, SHA-1, five compression method names) and an AES-256 (ECB) encrypted index: mount point, entry count, path hash seed, path hash index, full directory index (a second encrypted block: directories, then file names with offsets into the encoded entries) and the encoded entries (one bit-packed u32 per file: block size, block count, encrypted, compression method, then offset and sizes). The key is in `main`: the one 32-byte window of its read-only or data segment that decrypts the index start to the mount point `../../../` (Octopath Traveler 0: `main+0x57c7f70`, found among 116,287 candidates in under a second). Never print it. Most files are Oodle-compressed in 64 KB blocks; the two shader code archives are stored. `dm/gfx/tools/uepak.py` reads all of this (AES through macOS CommonCrypto, no Python package) and lists a pak (`uepak.py list <pak> <main NSO> <out tsv>`).
- Unreal Engine 4.27 with IoStore (DRAGON QUEST VII Reimagined: PhysX 3.4, so UE4; the source paths say `UEngine/4.27-source`) keeps its cooked packages in IoStore containers (`DOLL/Content/Paks/pakchunk0-Switch.utoc` / `.ucas`, 2.8 GB, and smaller chunks beside it) and everything else in `pakchunk0-Switch.pak` (1.5 GB, 3,972 files: configs, the asset registry, fonts, CRIWARE `.awb` banks, the two shader code archives); the other `pakchunk*-Switch.pak` files are 339-byte empty paks. Its version 11 footer is the same 0xdd bytes, but the magic sits 0xcc from the end, not 0xc8 as in Octopath Traveler 0's (five whole 32-byte compression method names, `Zlib` and `Oodle`); `uepak.py` takes either. The index is not encrypted. The update (v131072) is a delta over the base RomFS, so the merged extraction holds the whole game.
- Unreal Engine 4.26.2 with IoStore (Pikmin 4 1.1.1: `SDK MW+EpicGames+UnrealEngine-4_26_2` in `main`) keeps its cooked packages in `Carrot4/Content/Paks/Carrot4-Switch.utoc` / `.ucas` (7.5 GB, TOC version 2) and everything else in `Carrot4-Switch.pak` (3.5 GB, version 11 with the magic 0xcc from the end as DRAGON QUEST VII Reimagined's, 10,046 files, Zlib, index not encrypted): the configs, 1,294 Wwise `.wem` files, the WebM movies (stored) and the two shader code archives (stored, every shader an LZ4 block: `uepak.py` `lz4_shaders`). The update (v262144, 0.9 GB) is a delta over the base RomFS.
- Unreal Engine 5.6.1 (FINAL FANTASY RESONANCE, Square Enix: `SDK MW+EpicGames+UnrealEngine-5_6_1` in `main`) keeps everything in one IoStore container, `FFRS/Content/Paks/FFRS-Switch.utoc` / `.ucas` (6 GB, 63,910 chunks, Oodle in 64 KB blocks, not encrypted), beside `FFRS-Switch.pak` (2.9 GB, 2,363 files: configs, fonts, sound banks; version 11, index not encrypted, Oodle), a `global.utoc` holding only the script objects, and the RomFS's `Manifest_UFSFiles_Switch.txt`, which lists every file the engine sees, the shader archives among them. The pak has no shader code archive: the cooker converted both libraries into the container (EIoChunkType 8 `ShaderCodeLibrary`, one chunk each, the tables; type 9 `ShaderCode`, one chunk per shader group). A TOC (version 8) is a 144-byte header, the chunk IDs (12 bytes: hash, index, type), offsets and lengths (5 + 5 bytes big-endian), perfect hash seeds, the compression blocks (5-byte offset, 3-byte stored and raw sizes, method) and the method names; a library chunk's ID is CityHash64 of `<library>-<format>` lowercased in UTF-16 (`global-sf_glsl_switch-sp_glsl_switch`). Its tables and each group are Oodle-compressed, two layers, so `uepak.py` (`Toc`, `io_archive`) reads them through the game's own decoder in the Mac host's tool mode (`OodleHost`, `DM_OODLE_SERVE`; served before the SDK starts, no boot) and writes each library as a version 2 archive with every shader decoded: 3,542 + 70,702 shaders, 707 MB, in about 40 s. The pak stores block size 0 for a file of one block (`Pak.entry`).
- Unreal Engine 4.18 (DRAGON QUEST XI S, Square Enix) writes legacy paks, version 4: `JackGame/Content/Paks/pakchunk0-Switch.pak` (6.6 GB, 264,844 files) and `pakchunk1-Switch.pak` (7.2 GB, 147,824 files), and the update adds `*_0_P.pak` patch paks, which the engine mounts over the others (a path in a `_P` pak wins). The footer is the last 0x2d bytes (encrypted index flag, magic, version, index offset and size, SHA-1) and the index is not encrypted: the mount point, the file count, then each file's name and its FPakEntry (offset, stored and raw size, compression flags, SHA-1, the blocks' start and end, which are file offsets before version 5, the encrypted flag and the block size, 64 KB). The compression flags are no standard value: 3, which is LZ5 2.0 (LZ5 v2, later renamed Lizard; `main` exports `LZ5_*`, `LZ5_versionNumber` = 20001, and `FCompression::UncompressMemory` calls `LZ5_decompress_safe` for flags 3, `main+0x16f5c70`). Every block starts with its level, 0x2d = 45 (LIZv1 with Huffman). The repository has no LZ5 decoder: the game's own runs in the Mac host's tool mode (`hle_oodle.c`), which `uepak.py` (`Pak.read` with a decode function) and the shader split use. It takes 512 KB of work memory from the game's `malloc` on every call, so the host serves it only after `nninitStartup`. Scanning both paks (13.8 GB) with 8 workers takes 28 s.
- Capcom's MT Framework (The Great Ace Attorney Chronicles) keeps everything under `nativeNXx64/`: 447 ARC archives (`*.arc`, 17,910 files) beside loose `.tex` textures, Opus / ADPCM sound and H.264 `.mp4` movies. An ARC is `ARC\0`, u16 version 7, u16 file count, then one 0x90-byte entry per file: the path (128 bytes, `\`-separated, no extension), u32 type hash, u32 stored length, u32 length in the low 29 bits (flags above), u32 offset. Each file is one zlib stream (stored when the two lengths are equal), and its first bytes name the format (`TEX\0`, `MOD\0`, `MRL\0`, `GMD\0`, `SDL\0`, `SPK\0` the shader package, …). `shader_split.py` reads them (`mtarc_members`).
- NintendoWare Bezel Engine (Big Brain Academy: Brain vs. Brain) keeps nearly everything in Bezel Engine archives: 603 `.bea` files (`Archive/<Permanent|Temporary>~<name>.nx[.<language>].bea`, plus the engine's own `System.nx.bea` at the RomFS root), 7,665 files in all, 2.6 GB unpacked from 0.38 GB. A `.bea` is a NintendoWare binary file with magic `SCNE` (version 0x10100, BOM `FF FE`): +0x20 u64 entry count, +0x28 u64 offset of an array of u64 entry offsets, +0x30 the name dictionary (`_DIC`), +0x40 the archive name. Each entry is a 0x30-byte `ASST` record: +0x10 u16 kind (2; 258 for the one sound project), u16 alignment shift, +0x14 u32 stored length, +0x18 u32 length, +0x20 u64 data offset, +0x28 u64 name offset (u16 length, then the path). Every entry is one zstd frame, even the two whose stored length equals their length. Inside: layouts (`.lyt`, SARCs with their own `__ArchiveShader.bnsh`), BNTX textures, `.bfsha` shader archives, models (`.fmdb`), MSBT text and fonts; the sound is loose `stream/*.bfstm` and the movies loose `movie/Movie/*.mp4` (H.264). `shader_split.py` reads the archives itself (`bea_members`).
- Game Freak's engine before Trinity (Pokémon Sword 1.3.2, SDK 7.3.2) keeps its models, textures, effects and maps in 7,699 GFPAK archives under `bin/archive` (7.9 GB of the 12.1 GB RomFS), beside loose message tables, battle data, Wwise banks and BNSH files. A GFPAK is `GFLXPACK`, u32 version 0x1000, u32 0, u32 file count, u32 folder count, u64 offset of the file table, u64 offset of the file hashes (FNV-1a 64 of each path; no names are stored), u64 offset of the folder table pointers; the file table holds 0x18 bytes per file {u16 level, u16 compression, u32 length, u32 stored length, u32 0, u64 data offset}, compression 0 stored, 1 zlib, 2 LZ4 (one block), 3 Oodle Kraken (pkNX). All 201,635 entries of Sword's are LZ4, so the split needs no Oodle (`shader_split.py` `gfpak_members`). Pokémon Legends: Arceus's 678 GFPAKs are Oodle Kraken in every entry, which the split decodes with the game's own decoder ([Oodle entries of GFPAK archives](../reference/shaders.md#oodle-entries-of-gfpak-archives)). Inside: BNTX textures, BNSH shader archives, models (`.gfbmdl`) and animations.
- Game Freak's Trinity engine (Pokémon Legends: Z-A) keeps almost everything in one archive, `arc/data.trpfs` (4.4 GB, 13,181 packs, 238,546 files, 17.7 GB unpacked), indexed by `arc/data.trpfd`; both are FlatBuffers. The descriptor's root table holds the file hashes (u64, sorted), the pack names (`arc/<the first file's path without its slashes>.trpak`), one info table per file (its pack index) and one per pack (size, file count). The file system starts with `ONEPACK\0` and the u64 offset of its own root table: the pack name hashes and the pack offsets. The name hash is not a plain FNV-1a 64 of the descriptor's name, so a pack is named through its first file's hash. A pack is a table of file hashes and file tables `{1 compression i8, 2 level u8, 3 length u64, 4 data}`; compression 3 is Oodle (every file but 15,528 small stored ones), and every Oodle stream is Mermaid (header `8C 0A`, `CC 0A` for an uncompressed chunk). Oodle is linked into `main` and there is no Mermaid decoder with a licence fit for this repository, so the shader split runs the game's own `OodleLZ_Decompress` in the Mac host's tool mode (`dm/host/hle_oodle.c`, `DM_OODLE_SERVE=1`: requests on stdin, answers on stdout). The host finds it in `main` by the one function that references its name string (as for Unreal's archives), and calls it as the game does (`main+0x1a9b7bc` on 2.0.2: fuzz safe, no CRC check, thread phase 3). The shader split's full scan (every file decoded, 17.7 GB) takes 5 minutes with 12 workers on a 12-core Apple M6.
- The older KTGL of Fire Emblem: Three Houses uses a LINKDATA archive instead: `DATA0.bin` is an index of 32-byte entries `{u64 offset, u64 length, u64 stored length, u64 compressed}` into `DATA1.bin` (6.9 GB), and `patch1`–`patch4` hold loose overrides listed by `patchN/INFO0.bin` (0x120-byte entries ending in a `rom:/patchN/…` path). Its compressed entries and `*.gz` files are another chunked zlib: u32 chunk size (0x10000), u32 chunk count, u32 length, the stored length of each chunk, then the chunks at 0x80 alignment, each a u32 length + one zlib stream, or the plain bytes when the stored length equals the chunk's length (`kt_gz` in `shader_split.py`).
- **One archive of hashed names** (Super Smash Bros. Ultimate 13.0.5): the RomFS is `data.arc` (17.6 GB) and 14 NRRs. Its header (magic `0xABCDEF9876543210`) gives the offsets of the music, file and shared-file sections and of the file system table (one zstd frame of 50 MB); paths are Hash40s (CRC-32 of the path, its length in the top byte). A path leads to its current file through its info index, file info, info-to-data entry and file data, at file section + directory offset + 4 × its offset, zstd or stored (`dm/gfx/tools/smash_arc.py` documents the layout). The archive also keeps every earlier version of updated files, which no path leads to any more: 183,882 data blobs, 169,939 of them current (the 98 fighter NROs, 764 with their old versions, which the NRRs of older releases still list). A file's first bytes name its format: SSBH `HBSS` resources (`RDHS` shader databases, `XFUN` effect libraries, `LDOM` / `HSEM` / `LTAM` / `MINA` / `LEKS` models, meshes, materials, animations, skeletons), `EFFN` effects, `SARC` UI layouts, `BNTX`, `NUS3` sound and the NROs (`NRO0` at +0x10).
- Next Level Games' engine (Luigi's Mansion 3) keeps everything but its Wwise banks in archives of its own: `<name>.dict` indexes `<name>.data` (`init`, `global`, `Hotel/<floor>`, … 98 of them). The `.dict` is 3,203 bytes in every archive: u32 magic `0xa9f32458`, u8 4, u8 1, u8 compressed, u8 0, u32 the largest stored chunk, u8 chunk count (120), u8 record count (52), u8 name count (3), u8 0; then the records (0x18 bytes: u32 hash, u16 the top-level entry count of its table, u16, and one chunk index for each of 16 sections), the chunks (0x10 bytes: u32 offset, length, stored length (one zlib stream; 0 in an archive stored as is), u16 alignment as log2, u8 file, u8 GPU memory) and the file names `.data`, `.debug`, `.nxpc`. Only `.data` ships: the 13 even records are the game's variants (languages), the odd ones point into `.nxpc`. Record v's resource table is chunk v: 12-byte entries {u32 kind (bit 31: a list of w1 entries from entry w2; bits 27–30 the section; bits 22–26 the alignment as log2), u32 length, u32 offset in the section}. The game reads an archive in `main+0x1ba00` (1.4.0). `nlg_dict` checks this layout before reading; Mario Strikers: Battle League's archives are another revision it does not read yet ([A file no reader reads](../reference/shaders.md#a-file-no-reader-reads)).
- An update to that engine (Luigi's Mansion 3 1.4.0) leaves `.dict` and `.data` as they were and adds `<name>.patch`: u64 a bitmask of the records it patches, u32 offset, stored and inflated length of its body (one zlib stream: 16 u32 section offsets, 16 u16 alignments, then each section's new data), then 0xd0 bytes per patched record {u32 offset, stored length and entry count of its new resource table (one zlib stream), u16 its top-level entry count, u16, u32 the new length of each of 16 sections, 16 u32 end and 16 u32 start of the old data that is overwritten before it is read}. After loading a section the game walks the new table and packs that section's entries from its start at their alignment, each from the old section (an offset below its length) or from the body (the rest, counted from the old length), and writes the new offset into the entry (`main+0x1c8d0`, `main+0x1cbd0`); it does this in place, keeping a copy of the old data in [start, end). `shader_split.py` `nlg_patch_section` does the same. The base game's `init` has 5,149 shaders, 1.4.0's 5,317; 46 of the 103 the title draws exist only in the patched sections.
- PlatinumGames' engine (Bayonetta 3) keeps everything but its WebM movies in 72 PKZ archives in the RomFS root (14 GB; `root_000.pkz`, `core_000.pkz`, `pl_000.pkz`, `st1_000.pkz`, `sound_000.pkz` … `sound_017.pkz`, …). A PKZ is `pkzl`, u32 version 0x20000, u64 archive length, u32 file count, u32 entry table offset (0x20), u32 name table length, u32 alignment (0x1000), then one 0x20-byte entry per file: u32 name offset, u32 compression name offset (both into the name table that follows the entries), u64 length, u64 offset, u64 stored length. The compression is named per entry, `OodleKraken` or `None` here (11,760 and 5,159 entries); `main` knows `ZStandard` too (`main+0xcbdf80` maps the three names to 0xc9, 0x65 and 0). Inside are DAT archives: `DAT\0`, u32 file count, then the offsets of the file offset table (u32 each), the extension table (4 bytes each), the name table (u32 name length, then one name of that length each), the file length table (u32 each) and a name hash map. Their members are the engine's own formats (`.wmb` models, `.wtb` / `.wta` textures, `.bxm`, `.eff`, …) and, in `root_000.pkz`'s `shader*.dat` and `autoshadereff.dat`, its shader objects ([04](04-graphics.md)). There is no Kraken decoder fit for this repository, and this `main` reads its archives only through Oodle's streaming decoder (`main+0xcbc698` creates one, `main+0xcbc7f8` feeds it 256 KB at a time into a 10 MB window), so `OodleLZ_Decompress` was never linked in and its name string is not there: `hle_oodle.c` finds `OodleLZDecoder_MemorySizeNeeded`, `Create`, `Destroy` and `DecodeSome` instead (by the strings of Oodle's `oodlelzcompressors.cpp` and their order) and decodes each entry with them in `DM_OODLE_SERVE`. The shader split decodes every entry of every PKZ (8 workers, one Mac host each) in about 20 s; only `root_000.pkz` holds shaders.
- The same engine in its Wii U form (Bayonetta 2, SDK 3.6.0) keeps everything but its USM movies and Wwise sound in six CRI CPK archives in the RomFS root (`data000.cpk` … `data006.cpk`, there is no `data004.cpk`; 9.5 GB, 6,091 files). A CPK is `CPK `, u32 0xff, u32 the length of the @UTF table at +0x10 that names `TocOffset` and `ContentOffset`; at `TocOffset` comes `TOC `, 12 bytes and another @UTF table with one row per file: `DirName`, `FileName`, `FileSize` (stored), `ExtractSize`, `FileOffset` (relative to the smaller of `TocOffset` and `ContentOffset`). An @UTF table is big-endian: `@UTF`, u32 length, then from +8 u16 version, u16 rows offset, u32 strings offset, u32 data offset, u32 table name, u16 column count, u16 row width, u32 row count, and one flag byte (storage 0x10 zero, 0x30 constant, 0x50 per row; type in the low nibble) and a name offset per column. Vanillaware's CPKs (Unicorn Overlord) store their @UTF tables masked: each byte XOR the low byte of a 16-bit value that starts at 0x655f and is multiplied by 0x4115 after every byte (`cpk.py` `unmask`); the split reads their 7,744 files, 6.5 GB unpacked, in 38 s and finds no shader there, so Unicorn Overlord's tables are unchanged. 5,693 of the files (6.9 GB stored, 12 GB unpacked) are CRILAYLA: `CRILAYLA`, u32 size, u32 compressed length, the compressed bits, then the output's first 0x100 bytes stored; the bits are read from the end backwards and the output written from its end backwards (`dm/gfx/tools/cpk.py`, about 29 MB/s in Python with the bits read from a string). Inside are PlatinumGames DAT archives, as in Bayonetta 3; the shaders are BNSH files (`.vso` / `.pso` members of `shader.dat`, `shader2.dat` and `shadereff.dat` in `data000.cpk`), which the split finds by decoding every entry of every CPK (`shader_split.py` `cpk_jobs`, about 2.5 minutes with 12 workers). Bayonetta (SDK 3.6.0) has the same layout in nine CPKs (`data00.cpk` … `data03.cpk`, `data10.cpk` … `data14.cpk`, 4.25 GB), its shaders in `shader.dat` of `data00.cpk`; `cpk.py` reads them as they are.


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

