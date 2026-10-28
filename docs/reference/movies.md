# Movies

The SDK's movie library (`movie::Decoder`, `movie::Extractor`), which the host takes over in `dm/host/hle_movie.c`, and the games' movie players. Each entry: symptom, cause, fix, the games it was found on. The [symptom index](../agents/symptoms.md) lists every symptom.

## Decoder lifecycle

### The SDK's hardware decoder opens nvdrv

- **Symptom:** abort in NVIDIA's video path (`NvRmChannelOpen` → `NvOsDrvOpen`).
- **Cause:** the SDK movie library's hardware decoder opens nvdrv directly.
- **Fix:** the host takes over `movie::Decoder` (`hle_movie.c`, [`movie::Decoder` taken over as a MediaCodec-style protocol](#moviedecoder-taken-over-as-a-mediacodec-style-protocol)); nvdrv opens fail (`hle_nvdrv.c`).

### `movie::Decoder` taken over as a MediaCodec-style protocol

- **Symptom:** the game waits forever at a movie (Engage's 94 s opening `S01F.webm`; frames keep presenting, nothing happens).
- **Cause:** Unity demuxes the WebM itself and feeds VP9 packets to `movie::Decoder`; with no working decoder the movie never ends.
- **Fix:** `movie::Decoder` is a thin wrapper over Android's MediaCodec, and its vtable entries are symbol relocations, so `HLE_FORCE` on each method also catches Unity's virtual calls. The host implements the protocol: four `nn::os` events (`DecoderEvents` = input available, output ready, format changed, error), input indices (`AcquireInputBufferIndex`, none = 0x1011; `GetInputBuffer` fills the client's `movie::Buffer`; `SendInputBufferForDecode(index, offset, size, µs, flags)`), output indices (`AcquireOutputBufferIndex`, none = 0x100e; one NV12 frame per packet copied into the buffer Unity allocated, w × h × 1.5; `ReleaseOutputBufferIndex`). `Buffer`, `MediaData` and events are handled through the SDK's own exported functions (`SetDataAndCapacity`, `SetRange`, `SetInt32`, `SetRect`, `SignalEvent`). Unity reads `width`, `height`, `crop`, `nv12-y-stride`, `nv12-y-offset`, `nv12-uv-offset` and `slice-height`. The guest thread that sends a packet or returns an output index waits for the decode and signals the events itself, because `SignalEvent` is guest code and cannot run on a host thread.
- **Found on:** [Fire Emblem Engage](../../dm/games/0100A6301214E000/README.md): when the movie ends the game continues to its title screen.
- **Note:** output slots, VP9 decoding and the output format keys are in their own entries ([Packets wait while every output slot is taken](#packets-wait-while-every-output-slot-is-taken), [VP9 on host threads with libvpx](#vp9-on-host-threads-with-libvpx), [Output format keys (`md_format`)](#output-format-keys-md_format)).

### Output format keys (`md_format`)

- **Symptom:** a movie in grey with vertical bands of colour, its luma sharp; or a read far outside guest memory at `main+0x344f74` on the first movie picture (CRI Sofdec2's NV12 → RGB conversion).
- **Cause:** the host never set `nv12-uv-stride` in the output format (Unity reads `nv12-uv-offset` only). CRIWARE's Sofdec hands its VP9 video to `movie::Decoder` in mode 1, NV12 into its own buffers, and draws the planes with its shaders; it reads `nv12-uv-stride` from the output format, and the bands were the top row's chroma: every chroma row was read from the first one. Catherine's Sofdec2 sends its H.264 through `movie::Decoder`, mode 1, output format 4; the player reads `nv12-y-stride`, `nv12-y-offset`, `nv12-uv-stride` and `nv12-uv-offset` from the output format and checks none of the results (`main+0x34e694`), so the chroma stride was stack garbage.
- **Fix:** the host now sets everything the SDK's `DecoderImpl::GetOutputBufferFormatFromDecoder` (SDK 19 `subsdk0+0x121f50`) does: also `nv12-uv-stride`, the luma (Y) stride as for any NV12 picture, `nv12-width` and `nv12-height` (`md_format` in `hle_movie.c`).
- **Found on:** [DRAGON QUEST VII Reimagined](../../dm/games/0100A9D01C446000/README.md) (the grey picture); [Catherine: Full Body](../../dm/games/0100BF00112C0000/README.md) (SDK 8.2.1, CRI middleware; one boot: the crash).

### SDK 4's `movie::Decoder` constructor without a priority

- **Symptom:** a call through a null pointer in the host's `movie::Decoder::Configure` right after `nnMain` starts the logos.
- **Cause:** SDK 4's `Decoder` constructor has no priority argument (`_ZN5movie7DecoderC1ENS_11DecoderTypeENS_19DecoderOutputFormatENS_11DecoderModeE`). The host took over only the four-argument one, so the SDK built its own decoder and the host's `Configure` read that object as its own.
- **Fix:** both forms are taken over now (`hle_movie.c`).
- **Found on:** [DARK SOULS: REMASTERED](../../dm/games/01004AB00A260000/README.md) (SDK 4.5.0; one boot).

### The complete-object destructor

- **Cause:** the game calls the complete-object destructor `movie::Decoder::~Decoder` (`D1`); the SDK's own one destroyed the host object as a `DecoderImpl`.
- **Fix:** the host now takes it over too.
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md).

### Client events read through the registered pointer

- **Symptom:** the first deferred signal crashed in `InternalCriticalSectionImplByHorizon::Enter` (DRAGON QUEST XI S).
- **Cause:** `RegisterClientEvents` copied the four event pointers, because Dread's struct had been overwritten by its second movie. DRAGON QUEST XI S (Unreal's Switch movie player, VP9 with 6-channel Vorbis) does the opposite: it registers its struct right after constructing the decoder (`main+0x39edab0`, through the decoder's vtable) and fills it later, so the copy held leftovers (`0xe00000010`). The SDK keeps only the pointer (`DecoderImpl::RegisterClientEvents` stores it at +0xd0) and reads the entry each time it signals, skipping a null one (`SendOutputBufferEventAvailableToClient` and the others, `subsdk0+0x1feb0`).
- **Fix:** the host now does the same (`md_event` in `hle_movie.c`). When the client's entry is not a readable, 8-byte aligned pointer, it falls back to the entry's value at registration if that was one (Dread's case), and otherwise logs a red line.
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md); [DRAGON QUEST XI S](../../dm/games/01006C300E9F0000/README.md).

### `Flush`, `Stop` and `Start`

- **Fix:** as with MediaCodec, `Flush` announces nothing, and no input or output index is handed out while the decoder is stopped. The "input available" after `Start` is signalled about 2 ms later from a guest thread, as the SDK's codec thread would. Pending signals are dropped at `Stop`, `Flush` and destruction.
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md): the game ends a movie with `Flush`, `Flush`, `Stop`, destroy, and calls `Start` after a `Flush` when it loops.

### Input indices returned by a flush while a client holds them

- **Symptom:** the process dies when an activity starts, 2 runs in 5, with no `CRASH` line: the run ends with exit 137 (SIGKILL), and `~/Library/Logs/DiagnosticReports/dmhost-*.ips` has `EXC_BREAKPOINT` in `_os_unfair_lock_corruption_abort` ← `pthread_mutex_destroy` ← `md_free` (the movie decoder's destructor).
- **Cause:** each activity's preview is an H.264 movie (`movie/Movie/*.mp4`) the game loops with Flush and Start, from four threads at once (`DM_MOVIE_TRACE=1` logs every decoder call with its thread). Its event thread acquired an input index while the control thread's Flush waited for the packet being decoded; the flush then returned that index to the free list, it was handed out a second time, two packets shared one buffer (VideoToolbox -12909 on the overwritten one, in exactly the runs that died), and a fifth pending packet wrote past `pend[]` into the decoder's `npend` and `dmu`.
- **Fix** (`hle_movie.c`): a send now claims an index a flush returned, and `pend[]` cannot overflow. Handing such an index out last was not enough (with three hosts running at once one run still sent an index twice): an index the client held across a flush is now withheld for 250 ms unless the client sends it first, and an acquire refused only for that announces input when the first one comes back, so a client that abandons its indices, as MediaCodec's rules expect, loses none.
- **Found on:** [Big Brain Academy: Brain vs. Brain](../../dm/games/0100620012D6E000/README.md) (SDK 11.4.3, its activity previews).

### End of stream is the last input until `Flush` or `Stop`

- **Symptom:** when the prologue movie was skipped, 2.25 GB were mapped within seconds, past the application pool, and the game crashed in its allocator (DRAGON QUEST XI S).
- **Cause:** DRAGON QUEST XI S's movie player sends an end-of-stream packet (length 0, time 0) for every input it gets once its extractor is done, until the game stops the movie. Each one became an end-of-stream output that the player turned into an audio sample.
- **Fix:** also as with MediaCodec, an input marked end-of-stream is the last until `Flush` / `Stop`: no input index is handed out or announced after it (`in_eos` in `hle_movie.c`).
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md) (movie sub-list); [DRAGON QUEST XI S](../../dm/games/01006C300E9F0000/README.md).

### End-of-stream packets with no data, and deleted decoders

- **Symptom:** decoding the end-of-stream packet gave VideoToolbox −12909 and AudioToolbox −50.
- **Cause:** the game ends a stream with time −1 and the whole 4 MB input buffer as its length: that packet carries no data. It also deletes its decoders (`D0`).
- **Fix:** for `D0` the host frees the object with the game's `operator delete` after its own state.
- **Found on:** [Monster Hunter Rise](../../dm/games/0100B04011742000/README.md).

### H.264 drain at the end of a stream goes on with released slots

- **Symptom:** the game never leaves a movie that has played to its end; the last picture stays on screen.
- **Cause:** `DM_MOVIE_TRACE=1`: the end-of-stream packet came with H.264 pictures still in the reorder queue, its output went out as a black picture without the end-of-stream flag, and the queued pictures were to follow one per free output slot (the drain). Unity held all 16 video slots by then. `ReleaseOutputBufferIndex` decoded again only when packets were queued, and none were, so the drain never went on: no flag, and the game waited for the movie's end for good.
- **Fix:** a released slot now goes on with a drain too, and the end-of-stream packet's own output takes the first queued picture (`jflush`), so no black picture shows before a movie's last frames (`hle_movie.c`).
- **Found on:** [AI: THE SOMNIUM FILES](../../dm/games/010089B00D09C000/README.md) (Unity 2017.4: the 6 s Spike Chunsoft logo `StreamingAssets/Movie/al_titel_cl_sc_1k_30.mp4`, H.264 High, before the title).
- **Note:** the audio decoder was never affected: AAC has no reorder queue.

## Output slots

### Output slot counts per decoder kind

A decoder hands out as many outputs as it has slots, and while every slot is taken it takes no input ([Packets wait while every output slot is taken](#packets-wait-while-every-output-slot-is-taken)). The counts now (`hle_movie.c`): `MD_NOUT` 64 for a Vorbis decoder (the most any decoder has), `MD_NOUT_AAC` 32 for AAC and `MD_NOUT_VIDEO` 16 for video; a slot is allocated when first used. Every decoder had 8 at first. The entries below raised them one game at a time, and each gives the counts of its time: audio 32 ([Audio decoders keep more outputs than a pre-decoding player fills](#audio-decoders-keep-more-outputs-than-a-pre-decoding-player-fills)), audio 64 ([Audio output slots for players that hold every output](#audio-output-slots-for-players-that-hold-every-output)), video 16 ([Video output slots: how far the extractor gets ahead](#video-output-slots-how-far-the-extractor-gets-ahead)) and AAC 32 ([AAC decoders offer at most 32 outputs](#aac-decoders-offer-at-most-32-outputs)).

### Packets wait while every output slot is taken

- **Fix:** while every output slot is taken, packets wait and their input indices are not returned, as with MediaCodec; dropping them instead lost frames.
- **Found on:** [Fire Emblem Engage](../../dm/games/0100A6301214E000/README.md).

### Audio decoders keep more outputs than a pre-decoding player fills

- **Symptom:** the opening movie never starts, in about half the boots (DRAGON QUEST XI S: a black screen and no sound from boot on).
- **Cause:** Square Enix's `UMediaPlayer` has two natives of its own, `StartPreDecoding` and `IsPreDecodingCompleted`. The opening's Blueprint polls the second once a frame and only then plays its Matinee, whose event key calls `Play`. Its Switch player (vtable `main+0x77921b8`, `IsPreDecodingCompleted` at `main+0x42d4d20`) is complete when every selected track has at least one decoded output queued. The player starts its clock and its decoders on the first `TickInput` after `StartPreDecoding`. The render thread's first fetch (`main+0x39f07d0`, the `+0x5c7` flag) takes every decoded picture and shows the newest. The host decoded all eight video outputs (to 236 ms) within 36 ms. Each Vorbis decoder's 8 outputs were full by then, because nothing consumes audio before `Play`, so the extractor waited for an audio input and no picture followed the ones the render thread took. The movie started only when the game thread's poll came between the first decoded picture and that fetch.
- **Fix:** audio decoders then kept 32 outputs (`MD_NOUT`) and video decoders 8, so video keeps flowing after the first fetch. Since then audio decoders have 64 (Monster Train 2), AAC ones 32 (`MD_NOUT_AAC`, Mario vs. Donkey Kong) and video ones 16 (`MD_NOUT_VIDEO`, Animal Crossing: New Horizons); see [Output slot counts per decoder kind](#output-slot-counts-per-decoder-kind).
- **Found on:** [DRAGON QUEST XI S](../../dm/games/01006C300E9F0000/README.md): 8 boots out of 8 start; [Monster Train 2](../../dm/games/010051701FB46000/README.md) (64).
- **Note:** to find such a gate: probe the player's `SetRate` (`DM_PROBE=main:<off>`) to see whether the game ever asks to play, then follow the Blueprint native that calls it (the `FNameNativePtrPair` tables give the names).

### Audio output slots for players that hold every output

- **Symptom:** a movie stays black after its first picture, the sound never starts, and the game keeps presenting frames.
- **Cause:** the player takes every output a decoder offers and starts playing only once it holds about half a second of audio. The host then gave each decoder 8 output slots (before DRAGON QUEST XI S's 32 for audio) and takes input only while one is free (as MediaCodec does). `DM_MOVIE_TRACE=1` (each line now names the decoder): Unity held all 8 audio outputs, the audio decoder kept the next 4 packets and handed out no input index, the video packets behind them in the file never came, and the video decoder had given out one picture (the host releases H.264 pictures once more than 4 are queued).
- **Fix:** slot counts per decoder kind, measured: 8 audio / 8 video and 8 audio / 32 video hang; 64 audio / 8 video plays, starting with 27 audio and 8 video outputs held (peak 30 audio). 32 audio slots hold that peak with two to spare, so audio decoders got 64 (`MD_NOUT`; a PCM slot is 128 KB, allocated when first used), and video kept 8 (`MD_NOUT_VIDEO`: a 1080p picture is 3 MB, 8 MB as RGBA).
- **Found on:** [Monster Train 2](../../dm/games/010051701FB46000/README.md) (its 90 s intro, `StreamingAssets/intro_fmv.mp4`: H.264 High with 2 reorder frames and AAC interleaved packet by packet, played by Unity 2022.3's movie player: `movie::Extractor` in `subsdk0` feeds both decoders from one thread in file order). The game's A, B and + do not skip this intro.
- **Note:** later changes: video decoders have 16 slots ([Video output slots: how far the extractor gets ahead](#video-output-slots-how-far-the-extractor-gets-ahead)) and AAC decoders 32 ([AAC decoders offer at most 32 outputs](#aac-decoders-offer-at-most-32-outputs)); Vorbis keeps 64. `hle_movie.c` now has `MD_NOUT` 64, `MD_NOUT_AAC` 32, `MD_NOUT_VIDEO` 16.

### Video output slots: how far the extractor gets ahead

- **Symptom:** a movie shows as a black screen with the loading icon, one picture every few seconds, while its sound decoder keeps up.
- **Cause:** this player also holds every output it is offered (16 of 16 video, 59 of 64 audio), so how far its extractor can get ahead of the picture on screen is the video slots plus the 4 queued packets. With 8 video slots that was 0.4 s, and the audio behind the next video packet, whose clock the player follows, ran dry: 86 pictures in 25 s. With 16 or 32 slots all 1,960 play at 30 fps.
- **Fix:** a video decoder now has 16 (`MD_NOUT_VIDEO`); a slot is allocated when first used, so the 8 more cost 24 MB of NV12 at 1080p only in a player that holds them all.
- **Found on:** [Animal Crossing: New Horizons](../../dm/games/01006F8002326000/README.md) (the in-flight movie `Movie/OpeningMovie.webm`, VP9 1080p and Vorbis through `movie::Extractor`).

### AAC decoders offer at most 32 outputs

- **Symptom:** a crash in the game's heap free (`sead::ExpHeap`) on a movie thread, a second into the first movie, at the same address every boot (`pc=main+0x42f980`, a free-list unlink through a link of 0x3f, `addr=0x47`).
- **Cause:** the game's movie player keeps the outputs it holds in a ring of 32 entries {u32 output index, u32 flags, u64 time} per track (`main+0xe93d4c`: 0x200 bytes) and pushes with no bounds check (`main+0xe94140`), and the host's audio decoder offered 64 outputs, so the 33rd outstanding one, index 63, went past the ring. The console's AAC decoder cannot offer more than 32, or the game would overflow there too.
- **Fix:** AAC decoders now have 32 outputs (`MD_NOUT_AAC`; Monster Train 2 peaked at 30 held), Vorbis keeps 64.
- **Found on:** [Mario vs. Donkey Kong](../../dm/games/0100B99019412000/README.md) (1.0.1, SDK 15.3.0: its opening movie `Movie/Cutscene01.mp4`, H.264 and AAC through `movie::Extractor`).
- **Check:** `DM_PROBE_WATCH` on the link broke the movie's file reads (the page also held a 64 KB read buffer, and `read()` into a protected page fails with EFAULT), so the movie stopped and nothing crashed; `DM_PROBE_POLL=<address>+8=3f` (a snapshot only at the change to that value) showed it being written right after the block was relinked, and the values around it were the ring's pairs.

### AAC output buffer size is one frame

- **Symptom:** the movie then froze on its first second, its sound decoder full and the game taking no more audio.
- **Cause:** the player opens an AudioOut of its own (`main+0xe92c2c`), sizes its buffers by the audio decoder's `GetOutputBufferSize`, decodes straight into them (`SetDataAndCapacity` with that size) and appends each whole, advancing its audio clock by size / 4 samples (`main+0xe92a74`). The host answered 128 KB, a bound for any Vorbis packet, so every 4 KB AAC frame played as 0.68 s and the clock crawled (a temporary log of `AppendAudioOutBuffer` showed 131,072-byte buffers).
- **Fix:** an AAC decoder now reports one frame: 1024 samples per channel, 2048 with SBR (object type 5 or 29 in the AudioSpecificConfig), which is what each output holds; Vorbis keeps the bound, its packets vary in length.
- **Found on:** [Mario vs. Donkey Kong](../../dm/games/0100B99019412000/README.md) (1.0.1, the same movie as in [AAC decoders offer at most 32 outputs](#aac-decoders-offer-at-most-32-outputs)).

## Native-texture output

### Decoding into the client's NVN texture

- **Cause:** the decoder is created with output format 3 and mode 2 (`DecoderMode_NativeTexture`). The client's `movie::Buffer` then carries the `NVNtexture` object itself as its data, 8 bytes of capacity (an `RGBX8_SRGB` texture embedded in the game's texture pool entry, `main+0x226c98`), plus `SetNvnDev` / `SetNvnSync`.
- **Fix:** the host converts the VP9 picture to RGBA (BT.709 / BT.601 and range from the stream) and writes it into the texture's storage like `WriteTexels`, then marks the sync as signalled (`nvn_host_write_texture` / `nvn_host_signal_sync` in `hle_nvn.c`). The host writes the texture's block-linear storage on the CPU and the replay layer re-uploads the dirtied pages, as for `WriteTexels` ([nvn-objects.md](nvn-objects.md)).
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md).

### SDK 4's `movie::Buffer` has no sync object

- **Symptom:** `FATAL movie: missing SDK symbol _ZNK5movie6Buffer10GetNvnSyncEv` at the first frame of the native-texture movie (H.264 into the game's RGBA texture, as in Monster Hunter Rise).
- **Cause:** SDK 4's `movie::Buffer` carries the texture object but no sync object.
- **Fix:** the lookup is optional now; the game reads the texture once `GetOutputBuffer` returns, which the CPU write satisfies.
- **Found on:** [DARK SOULS: REMASTERED](../../dm/games/01004AB00A260000/README.md) (SDK 4.5.0; one boot).

### Parameter sets in `Configure` made the H.264 decoder without RGBA

- **Symptom:** a native-texture movie shows as a strip of small grey copies at the top of the screen.
- **Cause:** the texture held NV12 bytes. In RGBA8, 405 rows of 1920 pixels are exactly a 1920×1080 NV12 frame, rows 0–270 the luma at video black (16) and rows 270–405 the chroma (128). This game's MP4 demuxer hands the parameter sets to `Configure` (`csd-0` / `csd-1`), which made the H.264 decoder there without switching it to RGBA, and the first packet then found it made.
- **Fix:** both paths now go through `md_avc_open` (`hle_movie.c`).
- **Found on:** [Absolum](../../dm/games/0100A83022A22000/README.md) (H.264 logos, FNA3D's `nx_video_player.cpp`: type 1, output 3, mode 2). Monster Hunter Rise's movies never brought parameter sets to `Configure`.

### Last picture kept across a flush and at end of stream

- **Symptom:** looping movie previews flash black at the end of every loop.
- **Cause:** the game reads the buffers of the end-of-stream output (no picture) and of an output it acquired before the flush, and the host painted its texture black for both.
- **Fix** (`hle_movie.c`): a flush now keeps each slot's last picture, and in native-texture mode an output with no picture leaves the texture as it was, as the SDK's GPU consumer has nothing to draw.
- **Found on:** [Big Brain Academy: Brain vs. Brain](../../dm/games/0100620012D6E000/README.md) (the same looped activity previews, Flush and Start).

## Demuxers and codecs

### `movie::Extractor` runs natively

- **Symptom:** the game crashed on a null right after the failed open.
- **Cause:** `movie::Extractor` (libwebm in `subsdk0`) runs natively. The host used to force `SetDataSource` to fail because Engage's Unity demuxes by itself.
- **Fix:** the host no longer forces `SetDataSource` to fail.
- **Found on:** [Metroid Dread](../../dm/games/010093801237C000/README.md).

### VP9 on host threads with libvpx

- **Fix:** VP9 packets are decoded with libvpx (`dm/third_party/libvpx`) on one host thread per decoder: guest threads run on the stacks the SDK gives them, too small for libvpx. Packets are decoded in arrival order from one queue.
- **Found on:** [Fire Emblem Engage](../../dm/games/0100A6301214E000/README.md).

### VP8 uses the same libvpx worker and output paths

- **Symptom:** VP8 WebM movies play black; `movie::Decoder` is created with decoder type 2.
- **Cause:** the host offered output slots for VP8 without decoding the packets, and its vendored libvpx included only the VP9 decoder.
- **Fix:** the same libvpx revision now includes its VP8 decoder. Type 2 selects `vpx_codec_vp8_dx`, type 3 selects VP9; both use the existing worker, NV12 conversion and native RGBA output. VP8 keyframe dimensions come from its ten-byte uncompressed header. Only decoded pictures and end-of-stream packets make an output ready.
- **Found on:** [Pokémon Brilliant Diamond](../../dm/games/0100000011D90000/README.md): logos and opening WebM movies. Decoding 3,182 packets over 106 seconds averaged about 1 ms per packet; making the picture visible also needed [zero initialization of depth textures](rendering.md#new-textures-start-as-zero). [Silver Pines](../../dm/games/0100D5B026844000/README.md): the Team17 logo and the title screen's animated background, 1920x1080.
- **Check:** `t_movie_vpx` decodes synthetic VP8 and VP9 keyframes through the real decoder, checking NV12 and RGBA pixels and a truncated VP8 header.

### H.264 into an RGBA texture and AAC through AudioToolbox

- H.264 (type 1) into the client's NVN texture (output 3 = RGBA, mode 2), with AAC 5.1 sound (type 4, PCM16). VideoToolbox decodes to BGRA for that mode (`mavc_set_rgba`), and AAC goes through AudioToolbox's converter with an MPEG-4 ES descriptor built from the AudioSpecificConfig in `csd-0` (as FFmpeg's AudioToolbox decoder does). Found on [Monster Hunter Rise](../../dm/games/0100B04011742000/README.md); its end-of-stream packets and decoder deletes are in [End-of-stream packets with no data, and deleted decoders](#end-of-stream-packets-with-no-data-and-deleted-decoders).

### Vorbis movie sound

- Movie sound in [The Legend of Zelda: Link's Awakening](../../dm/games/01006BB00C6F0000/README.md) (SDK 8.2.0): the decoder is asked for type 5 (Vorbis) with output format 1 (PCM16); see the other entries in this file and `hle_movie.c`.

## Movie players

### Unity's video planes and checking the movie path

- Unity's `Hidden/VideoDecode` samples luma as `.w` of an `R8` texture with the swizzle `(0, 0, 0, R)` (Unity's Alpha8) and chroma from an `RG8` texture at half size; both are CPU-written block-linear textures ([nvn-objects.md](nvn-objects.md)).
- **Check:** to check the movie path apart from the decoder, fill the frames with a gradient in `hle_movie.c`; it should appear on screen.
- **Found on:** [Fire Emblem Engage](../../dm/games/0100A6301214E000/README.md).


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

