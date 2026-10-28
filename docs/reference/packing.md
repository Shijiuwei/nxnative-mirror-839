# Packing

## Build tools

### Homebrew .NET is on PATH but apphosts cannot find the runtime

- **Symptom:** the first extraction stops with `nx_extract exit code 131` and `.NET location: Not found`, even though `dotnet` is on `PATH`. The shader translator can fail the same way.
- **Cause:** standalone .NET apphosts search runtime roots and registered installations, not `PATH` ([.NET runtime discovery](https://daks.wtpuscm.cn/yunying/news-670953.html)). Homebrew's formula installs the runtime under `libexec` and its `bin/dotnet` can be a wrapper that sets `DOTNET_ROOT` only for its own child ([Homebrew caveat](https://pddn.yx-sf.com/chanpin/tool-33121081.html)). Resolving that wrapper's symlink gives `bin`, not the runtime root.
- **Fix:** `scripts/build_ipa_from_nsp.py` `configure_dotnet` derives the root from the `Microsoft.NETCore.App` entry in `dotnet --list-runtimes`, using the build's environment, before any extraction or survey. It exports `DOTNET_ROOT` both to that environment and to `os.environ`, for tools launched with an explicit environment and those that inherit it. Caller-supplied roots, including architecture-specific overrides, retain precedence. A failed or malformed probe leaves the environment unchanged rather than guessing the wrapper's directory.
- **Found on:** [Balatro](../../dm/games/0100CD801CE5E000/README.md), first extraction with Homebrew .NET ([PR #284](https://itei.wtpuscm.cn/keji/reminder-303333.html)). Applies to every game's build on that installation setup.

### The shader step stops with an empty `xlate_source`

- **Symptom:** the extraction and captures succeed, then the shader step stops with `translation chain tools do not match the lock file: xlate_source: lock … / actual` and nothing after `actual`, which reads like a stale translator.
- **Cause:** the translator (`dmsh_xlate_bin/dmsh_xlate`) is a universal framework-dependent apphost. When it finds no .NET 10 runtime for the slice it starts as, it exits 131 with an empty stdout. A Python under Rosetta (an Intel Homebrew's) starts it as x86_64, and it finds only the arm64 .NET; extraction is not affected, because `nx_extract` is built for the SDK's own architecture only. An Intel .NET on Apple silicon, or a `DOTNET_ROOT` that names one, fails the same way. `dmsh_chain.py` `tools_now` keeps only the stdout of `dmsh_xlate --source-sha`, without its exit code or stderr, so the apphost's own message is lost.
- **Fix:** the NSP script checks the environment before extracting anything: `check_native_python` stops under Rosetta (`sysctl.proc_translated`), and `check_prereqs` starts the translator once with `--source-sha` and stops with its first message line (`check_translator`). It also checks glslang and, for an iPhone build, the iOS SDK (`check_ios_sdk`, before the Metal check, whose message would blame the Metal toolchain). `own_compilers.py` no longer asks a Mac-only shader build (`DMSH_TARGETS=mac`) for the iOS Metal compiler. Open: `tools_now` still drops the translator's exit code and stderr, so `dmsh_build.py` run outside the NSP script shows the empty value. Fixing it changes `dmsh_chain.py`, a chain file, so it waits for the next chain change.
- **Found on:** NX2Apple builds ([LNPANDY/NX2Apple#31](https://opic.ai-hao123.com/kuangjia/music-26642762.html), item 3; [PR #477](https://vkms.wtpuscm.cn/fenxi/browser-424322.html), [PR #487](https://xzmn.wtpuscm.cn/jianzhan/layout-942414.html)); reproduced with `arch -x86_64`. Applies to every game.

## IPA archives

### A streamed file changes size and the IPA fails its CRC check

- **Symptom:** packing succeeds, but `unzip -t` or Python's `ZipFile.testzip()` reports a bad CRC for a large deflated file.
- **Cause:** `pack_ipa.py` wrote the uncompressed size collected before reading. Its streaming check compared the output size only for stored files, so a compressible file that grew or shrank still published an archive with the old size.
- **Fix:** count uncompressed bytes as they are read and compare them with the listed size before completing either streamed method. A mismatch stops the pack and removes the partial archive, preserving the previous IPA and its reuse index. Regression tests cover growth and shrinkage with both deflate and store.
- **Found on:** synthetic IPA inputs while reviewing [PR #279](https://rhmr.wtpuscm.cn/anli/goal-409504.html); no game files required.

## RomFS images

### RomFS folder transfers fail or leave stale game files

- **Symptom:** `devicectl` copying a large RomFS folder fails with POSIX error 24 (`EMFILE`) or spends most of its time on small files; deleting an old folder during iOS startup hits the launch watchdog. A push after changing extracted files can also keep sending old assets if an image is reused solely because it exists.
- **Cause:** a folder transfer performs hundreds of thousands of file operations. The image cache must track its source tree, and magic plus total length cannot establish that an image's index is valid. Accepting that weak check before deleting the working folder loses the fallback when mounting later rejects the index.
- **Fix:** `dm/tools/romfs_image.py` packs one `DMROMFS1` file: a 32-byte header, file bytes, then the index and name pool. Each directory has a sorted, contiguous run of child entries; the host binary-searches each path component and reads file ranges with `pread` through one descriptor per mount. Overlay folders keep precedence and directory listings merge both sources. The shared `dm/host/romfs_image.h` loader checks the full index before either mounting or iOS folder cleanup: length, entry and name bounds, file ranges, a rooted breadth-first directory tree, and sorted, unique child names. It supports an empty root too. Invalid images fall back to the folders on iOS; an explicit invalid `DM_ROMFS` image is refused with a `[RED]` line. The index check detects interrupted sequential transfers and structural damage, not same-length corruption inside file contents. The packer publishes by rename only after the source stays stable. Its reuse stamp tracks source names and file metadata including ctime, the packer and image identity; the push helper always checks it, and `--repack` forces a rebuild. Usage: [Quick reinstalls](../development.md#quick-reinstalls).
- **Found on:** [Tears of the Kingdom](../../dm/games/0100F2C0115B6000/README.md#files-and-shaders), whose 304,000-file transfer and startup deletion failures are reported in [PR #369](https://bukc.ai-hao123.com/wendang/faq-79851401.html). [ANIMAL WELL](../../dm/games/010020D01AD24000/README.md#files) provides the empty-image boot case. Synthetic packer/host tests cover actual file reads, case aliases, overlay merging, corrupt indexes and changed sources; the macOS Foundation test executes iOS's locator to check that rejected images preserve its folder.

### Separate RomFS deployment loses its image or folder fallback

- **Symptom:** a successful `--romfs separate --no-keep-extracted` build prints a push command pointing at deleted game files. On iOS, a complete image for another build can also prevent startup despite a valid `Documents/romfs` folder.
- **Cause:** separate mode omitted the extraction-retention check, so normal post-build cleanup deleted the image needed for transfer. The startup identity rejection returned before the existing folder lookup.
- **Fix:** reject separate mode without `--keep-extracted` before extraction or building. Reject a mismatching image while continuing to the validated folder fallback; move a folder aside for deletion only after accepting the image. Matching images and the default bundled route retain their behavior. The identity covers image size, header and index, not same-length file-content changes.
- **Found on:** synthetic CLI and Foundation startup tests while integrating [PR #493](https://ulzv.yx-sf.com/yinqing/calculator-95681044.html), after [PR #478](https://lkzb.tcti.cn/gongxiang/beauty-83609792.html); no game or device is needed to reproduce either failure.

### Building creates a full RomFS tree before packing its image

- **Symptom:** reducing the iPhone transfer to one image still creates thousands of resource files on the Mac, and build scanners require the expanded folder.
- **Cause:** folder extraction followed by a separate image pack duplicates the work; ordinary filesystem APIs cannot address files inside an image.
- **Fix:** the NSP pipeline passes `--romfs-image` to `nx_extract`. LibHac layers the update over the base and checks its integrity while the writer streams file bytes into a temporary `DMROMFS1` image, appends the sorted directory index and publishes by rename. ExeFS remains expanded; RomFS and each DLC remain one physical file. `dm/tools/romfs.py` gives the scanners read-only virtual paths, bounded seeks and archive mappings (Python 3.12+), without changing the process's standard filesystem APIs. The IPA packer maps the physical image to `Payload/DmIOS.app/romfs.img`; iOS and the Mac launcher validate and mount it in place. Neither installation nor startup expands the resource files. Documents overrides and legacy folder inputs remain supported. A source image is uncompressed; the IPA can deflate it, so file-count reduction does not imply a smaller installed byte count. Full images for separate game versions cannot share unchanged individual files through APFS clones as legacy folder extraction did.
- **Found on:** [ANIMAL WELL](../../dm/games/010020D01AD24000/README.md#files), whose direct image is an empty root (56 bytes); the local Baba Is You provisional build exercises 6,374 files in a 103,473,292-byte image, 131 shaders and an unsigned IPA containing only the single image entry. [Raging Loop](../../dm/games/010020400E1C2000/README.md#files-and-shaders) supplies Unity resources: all 1,473 source manifest entries match the image, and every registered shader remains in the scan. Synthetic integration tests call the actual LibHac writer, read unaligned archive ranges, and exercise IPA mapping, Xcode staging and bundled-image startup.

### The zstd command given a path inside the image

- **Symptom:** the NSP script stops at the shader split of a registered profile with `[RED] 13744 file(s) could not be read, their shaders are left out (first: AI/FileEntry/FileEntry.Product.100.byml.zs: CalledProcessError: Command ['zstd', '-d', '-c', '-q', '…/extract/v0/RomFS.img/AI/…'] returned non-zero exit status 1.)`.
- **Cause:** `shader_split.py`'s `unzs` read a `.zs` file through `romfs.open`, which reads inside the image, but then handed its path to the `zstd` command, which can only open real files (`Not a directory`).
- **Fix:** `unzs` feeds the command the bytes it read (`input=`). No other tool hands an external command a path inside the image. Test: `test_romfs_read.py` `test_zstd_file_inside_an_image`.
- **Found on:** [Super Mario Bros. Wonder](../../dm/games/010015100B514000/README.md#shaders): every file of its RomFS is zstd-compressed, 13,744 left out in both versions; the first image-backed build after the folder extraction.

## Launcher metadata

### A Mac front end's default settings change Metal execution paths

- **Symptom:** starting a Mac app through a front end with unchanged settings enables the encode thread, and can disable Metal 4 pipelines and the pipeline archive; starting the same app from Finder keeps the host's defaults.
- **Cause:** Mac `launcher.json` reused iOS defaults from `settings.tsv`. `rec_on` in `nvn_rec.h` defaults off on Mac and on on iOS; `pso4_on` in `nvn_mtl.m` defaults on on Mac and follows the profile on iOS. The exported defaults instead enabled the encode thread everywhere and followed the profile for Metal 4 everywhere. A front end applying those values changed the Mac host's behavior without a player override.
- **Fix:** a setting's default may be `<ios>|<mac>`, selected before resolving a profile macro or build variable. Metal 4 uses `DM_GAME_PSO_MTL4|1`, the encode thread `1|0`. The generator validates the platform and the pair. Tests cover a profile with Metal 4 on and one with it off, and both platforms; the host and shader chain are unchanged.
- **Found on:** synthetic metadata generation for [Super Mario Bros. Wonder](../../dm/games/010015100B514000/README.md) while reviewing [PR #352](https://eplt.tcti.cn/youhua/price-85131480.html); checked against the native defaults, without a gameplay run.

## External drives

### A game folder on an exFAT drive reads one update as two, or fails to remove a folder

- **Symptom:** with the NSP folder on an exFAT or FAT drive, the build leaves out an update whose name has no `[v…]` tag (the folder seems to hold two), or a stage stops with `FileNotFoundError` on a `._<name>` path inside `<id>_build/` and the next run fails on the half-deleted folder.
- **Cause:** these file systems have no extended attributes, so macOS keeps a file's attributes (a download's quarantine flag) in an AppleDouble file `._<name>` beside it, and deletes it with the file. `scan_nsps` took `._Update [<title ID>800].nsp`, 4 KB of metadata, for an update: its name carries the title ID. Deleting `x` takes `._x` with it, so `shutil.rmtree`'s later unlink of the listed `._x` found it gone; before Python 3.13 that error ended the removal.
- **Fix:** `scripts/build_ipa_from_nsp.py` `scan_nsps` takes a file only when its header is one nx_extract opens (`is_package`: `PFS0` at 0, or an XCI's `HEAD` at 0x100), and says which files it skips. The name tag alone no longer counts, so the same check also rejects any other tagged file that is not a package. Names that start with `._` are skipped without a line. The script's `rmtree` passes ENOENT from `shutil.rmtree` (the entry is gone, which is the goal) and raises every other error. `dm/tools/tests/test_nsp_external_drive.py` and `test_nsp_scan.py` cover both without game files.
- **Found on:** the NX2Apple fork's users, who keep games on external drives ([PR #476](https://oild.wtpuscm.cn/qiye/growth-951765.html), part of [issue #388](https://tuqm.ai-hao123.com/tuiguang/success-07156116.html)); reproduced on an exFAT disk image made with `hdiutil create -fs ExFAT`. Applies to every game.

## Compressed inputs

### NSZ decompression rejects `--keys`

- **Symptom:** `decompress NSZ` stops immediately with `nsz: error: unrecognized arguments: --keys …`.
- **Cause:** `nsz` 4.6.1 has no `--keys` argument. It loads `keys.txt` beside `sys.argv[0]`, or falls back to `~/.switch/prod.keys`, while importing its modules, before parsing arguments ([4.6.1 key loader](https://thsq.wtpuscm.cn/kuangjia/objective-651506.html)). Version 5.0 supports `--keys` ([argument parser](https://bnjx.yx-sf.com/pingtai/saving-07657152.html)); staging `keys.txt` also supports 4.6.1.
- **Fix:** `scripts/build_ipa_from_nsp.py` `decompress_nsz` prefers the current Python's module: it links the file selected by `-k` as `keys.txt` in the conversion's temporary folder and runs `nsz -D` there, with `sys.argv[0]` set to `nsz` before import. Without the module, `nsz_command` probes `nsz` and `nsz-cli-macos-arm64` on the build's PATH and passes `--keys` to the first whose successful help lists that option. An unsupported or failed candidate does not hide the next one; when none works, the error lists the reasons and installation options. Executable, input and output paths are absolute because conversion changes directories. The temporary folder and link are removed on success or failure; the user's default keys are untouched.
- **Found on:** [Super Mario Odyssey](../../dm/games/0100000000010000/README.md), converting its v0 NSZ. Applies to NSZ and XCZ inputs for every game. Synthetic tests during [PR #327](https://daks.wtpuscm.cn/yunying/news-670953.html) also reproduce an older console script hiding the release binary, a failed help probe containing `--keys`, and a relative PATH executable failing after the directory change; no game files are needed for those cases.

## Cartridge dumps

### A cartridge dump with its update on it extracts the update as the base game

- **Symptom:** the extraction of a game whose base is an XCI that also carries its update (a "1G+1U" dump) stops with `[RED] nx_extract exit code -6`. `extract_v<version>.log` lists the update's build IDs even for the base game alone, then ends in an unhandled exception from the RomFS (`ResultFsNonRealDataVerificationFailed (2002-4604): Hash error!` from `RomFsFileSystem..ctor`, or `ArgumentException` / `InvalidOperationException` from the patch layers).
- **Cause:** the XCI's secure partition holds the update's NCAs beside the base game's, and the update's Program NCA has the program's title ID too. `nx_extract` took the first Program NCA of the lowest title ID as the base, which is the update's when the dump lists it first. Its data section is a patch (BKTR) over the base RomFS, so it cannot be read alone or under another patch. The ExeFS still opens, with the update's build IDs, because an update's code section is complete. The Control NCA (display version, icon) was also whichever of the two came last.
- **Fix:** `dm/tools/nx_extract/Program.cs` `OpenNsp` reads the base package's content metas (`*.cnmt` in its Meta NCAs) and keeps only the NCAs that its Application content meta lists, and prints how many it left out. When none of them is a Program NCA (NCA files renamed, so their names are not content IDs), it prints a `[RED]` line and reads every NCA as before. An NSP or a base-only XCI keeps every NCA but its meta NCA, so it extracts as before (Balatro's NSPs and Monster Hunter Rise's XCI give identical manifests). The update is still taken from the update NSP in the folder; the one on the cartridge is not used, so an XCI alone packs the base game's version.
- **Found on:** [The Legend of Zelda: Tears of the Kingdom](../../dm/games/0100F2C0115B6000/README.md), a 1G+1U XCI (base v0 with update 1.4.3, whose 524 MB Program NCA is listed before the base's 17.1 GB one) beside the 1.4.3 update NSP ([PR #311](https://osqy.yx-sf.com/zhizhu/browser-21853867.html)). Reproduced with a package of Balatro's update NSP followed by its base NSP: the base alone and the base with the update now give the same manifests and control data as the separate NSPs. Applies to any game's XCI that carries its update.


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

