# Frame rate

Frame pacing: the host's frame step, engine frame limiters, present intervals and display rates. Each entry: symptom, cause, fix, the games it was found on. The [symptom index](../agents/symptoms.md) lists every symptom.

## Frame step

### Frame cap checked and used from the same sample

- **Symptom:** clang reports division by zero in `fstep_tick` (`hle_nvn.c`).
- **Cause:** the present path checked `fps_step_cap()` through `fstep_on`, then `fstep_tick` sampled it again for `60 / cap`. The second sample could be zero even though the first enabled stepping.
- **Fix:** sample the cap once in `nvnQueuePresentTexture`, check it and pass the nonzero value into `fstep_tick`.
- **Found on:** clang static analysis ([issue #424](https://jrii.ai-hao123.com/ziyuan/forecast-71880394.html)); no gameplay failure reported.

### sead's dynamic present interval, `ksys::VFR` and the host's frame step

- **Symptom:** a 30 fps game to run at 60; after the floor was lowered, slow motion on a GPU-bound device (GitHub issue #9).
- **Cause:** find where the present interval comes from and how the logic time step follows it before patching anything. BotW's `sead::GameFrameworkNx` main loop (`main+0x112660c`) runs a dynamic interval: from the last frame's time it computes how many vertical syncs were needed, clamps that to [2, 3] (the maximum from the interval controller built at `main+0x1126c04`), steps up after three overrunning frames and down one level every six frames, and sets the result through `setWaitVsyncInterval` (vtable +0x130, `main+0x11269b8` → `nvnWindowSetPresentInterval`). The game logic follows on its own: `ksys::VFR` is initialized with a constant base interval of 2 (`main+0x127e740`), and every frame `VFR::updateInterval` (`main+0x151d3d0`) takes the framework's current interval, giving a frame ratio of interval / 2, so the engine's variable frame rate scales its time step by the present interval and the speed is right as long as the interval matches the frames. Then check what the controller measures: BotW's measured only the main thread's CPU time (about 3 ms a frame), so a GPU-bound device stayed at interval 1 and played in slow motion.
- **Fix:** now the host's frame step in whole vsyncs: a stub at the framework's `setWaitVsyncInterval` hands the game the host's step, and frames stay on screen exactly that long. The method before it cuts the game's frame-interval logic, and those cuts stay, for the interval the framework keeps while the game's VFR override holds (DM_FRAME_STEP=legacy, which ran this method on its own, was removed in 79d82f7b): five rewriter cuts lower the floor from 2 to 1 (its `patches.py`, build `cd57b23f…`): the candidate floor (two places) and the hysteresis targets. Two more cuts make the controller always take its GPU-time path, and the host's timestamp reports had to carry real GPU times for that path to see anything (profile key `gpu_timestamps`, [GPU timestamps per submission segment](#gpu-timestamps-per-submission-segment)).
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (1.9.0; its note has the interval controller and the legacy cuts).
- **Check:** probes confirmed the legacy result: the framework presents at interval 1, VFR reports interval 1 with ratio 0.5, and when a frame runs long the game steps back to 2 with ratio 1, so the game speed does not change.
- **Note:** the zeldaret/botw decompilation (`VFR.cpp`) gave the semantics; 1.9.0's addresses were found by the constants (`60 / interval`, the `fmov s11, #2.0` floor).

### A thermally serious iPhone holds 30 fps

- **Symptom:** on an iPhone, a game with the frame step at 60 fps drops a frame in most windows after a few minutes, and its GPU time per frame roughly doubles to triples (17-26 ms for the same scenes).
- **Cause:** at thermal state serious (`NSProcessInfo` `thermalState` 2) iOS caps the GPU clock to about a third: the median of the frame's tiny passes, a clock proxy from `DM_NVN_GPUPROF`, rose from about 6 to 22 µs. A frame with thousands of draws no longer fits 16.7 ms, and running the GPU flat out keeps the device hot.
- **Fix:** `thermal_capped` in `hle_nvn.c`: while the device is serious or worse, `fps_step_cap` holds 30 fps; it goes back to the cap after the device has been fair or better for 30 s (`THERMAL_CALM_NS`), so it does not flap. The thermal state is read once a second (`dm_thermal_now`). The log says `thermal state 2: the frame step holds 30 fps …` and `frame step cap 60 → 30 fps … (thermal state serious)`. `DM_THERMAL_CAP=0` turns it off. The Mac reports no thermal state, so it never applies there.
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (iPhone 17 Pro Max, docked at 60 fps: serious after 226 s from a cold start, after 64 s from a warm one; at 30 fps the frames held with no drops).
- **Resolved: movies interrupted thermal recovery, and acquire/present callers raced.** The cap getter short-circuited thermal sampling while a movie or a 30 fps target applied. A serious interval during a movie could leave an earlier calm timer intact and release the cap too soon; cooling during a movie instead added another 30 s hold after playback. Thermal sampling now runs independently of those caps, with a mutex protecting the sample, cached result and recovery timer across acquire/present threads.
- **Check:** `make -C dm/gfx/tests GAME=01007EF00011E000 check_t_thermal thermal-tsan` drives the production cap getter with a controlled clock and thermal source: serious/critical, recovery interrupted by heat, cooling and heating during movies or a 30 fps target, one-second sampling, disabled/inactive caps, the Mac's absent thermal state, and concurrent readers under ThreadSanitizer. No new device gameplay run or on-device recovery observation is claimed.

### The frame step after a 30 fps cap ends

- **Symptom:** after a 30 fps cap ends (a prerendered movie, the thermal cap), a GPU-bound game stays at 30 fps in a scene that held 60 before the cap; the log shows `frame step cap 30 → 60 fps` and a few frames later `frame step: hold 1 → 2 vsyncs at present … (3 late frames; …)`, with no probe after it.
- **Cause:** when the cap rose, `nvn_fstep_next` put the hold back to 1 with no grace, while the clocks were still set for 30 fps, so three late frames stepped it up again. After a movie, the cap change had dropped the cost that held hold 1 (`cref`), so the same-cost probe could not fire. After the thermal cap, the GPU reference (`nvn_fgpu`) had been learned during the cap, at the clock it left, so the same work at hold 2 did not look slowed. A cost near `FSTEP_FIT` at either hold kept the fit trial off in both cases.
- **Fix:** `nvn_fstep.h` keeps `cref` and the GPU reference for each kmin. Entering a cap also saves the cost of an outgoing run that held for at least 8 s, even if no earlier hitch had populated `cref`; a shorter run keeps the earlier proven reference. The frames after the cap rises are a counted probe with the usual 30 frames of grace, starting with the first frame at the shorter hold. The completed frame at the old cap consumes none of that grace, and a failed recovery doubles the hold-off as usual.
- **Found on:** reading the code; modelled in `t_fstep` with [The Legend of Zelda: Tears of the Kingdom](../../dm/games/0100F2C0115B6000/README.md)'s movie flag and [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md)'s iPhone GPU times (0.86 vsyncs at hold 1, 1.5 at hold 2). Not yet seen on these paths in a device log; first seen on a Mac with a 30 fps mode in a downstream fork.
- **Check:** `make -C dm/gfx/tests GAME=010015100B514000 t_fstep && dm/gfx/tests/t_fstep`: movie and thermal recovery, the first movie without a prior hitch, repeated caps after short and sustained runs, all 30 grace frames, failed-probe backoff and game-time debt. The original recovery cases fail on main before [PR #253](https://pnqp.wtpuscm.cn/jianzhan/metric-969534.html); its initial fix still failed the first-movie, repeated-cap and full-grace regressions. All pass with the corrections above.

### Engine-native frame caps also govern presentation

- **Symptom:** selecting 30 FPS changes the engine's interval field, but the screen and frame loop still run near 60 FPS.
- **Cause:** presentation took its cap only from the optional frame-step state page. Games with engine-native frame limiters have no such page, so a profile using display pacing continued to present every tick despite `DM_FPS_TARGET=30`.
- **Fix:** resolve a presentation cap from the launcher environment or profile when no frame-step page exists. Use it for the present interval and display-link target without enabling frame-step logic. Existing state-page, movie and game-override behavior keeps precedence; target zero retains the profile's interval semantics.
- **Check:** `t_nvn` verifies literal 33,333,333 / 16,666,666 ns periods for 30 / 60, display-link targets, disabled frame-step logic for native engines, target zero and movie/state-page precedence. All 134 NVN checks pass with Metal validation.
- **Found on:** [NieR:Automata iPhone controls](../../dm/games/010056B015FE8000/README.md#iphone-resolution-and-frame-rate-controls).

### Resolution policy is independent of the frame-rate cap

- **Symptom:** the game always renders at full resolution, or turning off dynamic resolution still allows the native controller to lower it.
- **Cause:** fixed-resolution code cuts bypassed native scale clamping and UV publication, while the frame-limit handler also overwrote scale and counters. Restoring only those instructions is insufficient when an additional mode-specific lower bound can override the selected preset.
- **Fix:** the verified build retains native scale/UV publication and makes its lower bound follow the selected preset. Apply the resolution choice independently of the FPS limiter. In dynamic mode the game owns current scale and timing counters; fixed mode selects the full-resolution minimum. Verify the build and required instructions before enabling it. Rewriter `CAPABILITIES` metadata offers the launcher control only for supported rewrites; older fixed-resolution images omit it.
- **Check:** production memory-policy tests cover 0/30/60 FPS, on/off/unset/invalid choices, older code and unmatched builds. An asset-backed fixture executes the actual native controller with only GOT addresses relocated, checking both console modes, scale reduction under load, correct UV publication, full-resolution recovery and the fixed-resolution path.
- **Found on:** [NieR:Automata iPhone controls](../../dm/games/010056B015FE8000/README.md#iphone-resolution-and-frame-rate-controls).

### GPU timestamps per submission segment

- **Symptom:** slow motion when the GPU is the bottleneck: a game's interval controller that times the GPU never steps up (GitHub issue #9).
- **Cause:** a game that times its GPU work as the difference of two timestamp reports in one submission (`TIMESTAMP_TOP` at the start of the frame's command buffer, `TIMESTAMP` at its end) read 0–6 ticks, because the host wrote all the reports of a submission with one time in its completion handler.
- **Fix:** the profile key `gpu_timestamps segment` makes each report take the GPU start (`TIMESTAMP_TOP`) or end (other types) of the `sub_cut` segment ([Submissions handed to the GPU in segments](#submissions-handed-to-the-gpu-in-segments)) it was encoded in (`rep_ticks` in `nvn_mtl.m`: MTL4CommitFeedback times converted to the guest clock, written in the last segment's handler as before). `DM_NVN_GPU_TS=submission` gives the old timestamps.
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (legacy frame-step method): BotW's frame has 5 reports in one submission (`main+0xe94584`, `main+0xe94664`); with two more cuts the controller always takes the GPU path, and the GPU time now reads the frame's GPU span, gaps where the GPU waits for the host's translation included. Measured on the Mac with two BotW runs sharing the GPU under a heavy CPU load (load average about 75, a shader rebuild running): GPU time p50 16–23 ms; the fixed run stayed at interval 2 (3 while the GPU time passed 1.9 vsyncs, 1 for under 2% of the frames) at 0.93–0.98 of real time, while a run with `DM_NVN_GPU_TS=submission` spent 10–70% of its frames at interval 1 and ran at 0.78–0.95. The same GPU time drives the game's dynamic resolution.

### Raw NVN timestamp units

- **Symptom:** GPU time reads 32 times too small when a game converts raw NVN counter timestamps itself.
- **Cause:** the host wrote reports in 19.2 MHz CPU ticks, while the raw NVN timestamp field uses 614.4 MHz ticks (384 ticks per 625 ns). Games calling `nvnDeviceGetTimestampInNanoseconds` saw correct time because its inverse conversion had the same wrong rate, but an inline conversion read a 16 ms span as 0.5 ms.
- **Fix:** `nvn_timestamp.h` shares the 614.4 MHz conversions between all report writes (`gpu_ticks`, `rep_ticks`) and `nvnDeviceGetTimestampInNanoseconds`. Wide integer products preserve long uptimes. The guest clock epoch stays the same; CPU system counters stay at 19.2 MHz, and the API still returns nanoseconds.
- **Found on:** [NieR:Automata The End of YoRHa Edition](../../dm/games/010056B015FE8000/README.md), whose dynamic-resolution controller subtracts two raw report timestamps and converts the difference inline.
- **Check:** `t_timestamp` checks known vectors, long uptimes, round-trip precision and the game's exact inline arithmetic. `t_nvn` checks the API's known units and bounds a completed report between submit and fence wait.

## Engine frame limiters

### Unity's integer target frame rate

- **Symptom:** menus reach 60 fps with `present_interval display`, but entering gameplay returns to 30 fps despite spare GPU time.
- **Cause:** the game sets Unity's `Application.targetFrameRate` to 30. Changing NVN's present pacing does not change the engine's own limiter.
- **Fix:** `hle_fpslock.c` supports integer FPS fields alongside float FPS, milliseconds and vsync counts. Its build-gated row keeps the verified field at the profile's `fps_target` (`DM_FPS_TARGET` overrides it), preserving unset values such as 0 and -1. `present_interval display` removes the separate interval-2 present cap.
- **Found on:** [Pokémon Brilliant Diamond](../../dm/games/0100000011D90000/README.md), 1.3.0; its note records the setter, field and gameplay timing checks.
- **Check:** `t_fpslock` exercises the production write path, build mismatch, disabled override, repeated game writes, unset values and 30 fps fallback, plus the existing float, millisecond and two-copy vsync paths.

### A render-thread limiter with a constant and a fixed time step

- **Symptom:** the game presents at interval 1, yet every frame takes exactly 33.3 ms of "frame work"; after only the limiter was raised, everything ran twice as fast on the iPhone.
- **Cause:** the cap is the render thread's own limiter, `main+0x707090(this, fps)`, which sleeps out 1000 / fps ms since its previous call, called once with the constant 30 (`main+0x706fa4`). Found from the render thread's backtrace in the thread dump (`HGCommandDispatcher`), not from the NVN calls. The frame update takes a fixed step, the float 1/30 at `main+0x167a6ec`, loaded at `main+0x667904` and passed to `main+0x6a337c`.
- **Fix:** two cuts: the limiter's constant, and a second cut that loads the 1/60 at `main+0x167a834` (the community's 60 fps cheat for this build changes the same two words). Some 20 other loads of that 1/30 turn parameters counted in 30 fps frames into seconds or step single systems; they stay.
- **Found on:** [DARK SOULS: REMASTERED](../../dm/games/01004AB00A260000/README.md) (30 on the Switch, 60 now).
- **Check:** before changing the constant, check how the game steps time, and not by eye: a 0.8 s walk that looked the same at 30 and 60 fps on the Mac was taken for a time-based game, and on the iPhone everything ran twice as fast. Probing every site that loads such a constant (`DM_PROBE`, 16 per run) tells per-frame steps from one-off conversions, but which of them sets the game's speed only shows by timing the game.

### A frame-end wait polled in 2 ms sleeps

- **Symptom:** 55 fps with every frame 18.2 ms, the main thread asleep most of the time; every 11th frame stays on screen two vsyncs.
- **Cause:** the game's own code. KTGL's frame end (`main+0x443e8c`) presents, acquires the next texture, then waits until N / 60 s have passed since the frame's start tick, polling `GetSystemTick` and sleeping 2 ms a round (`main+0x4c9f24`). N is the game's vsyncs per frame, a global. The start tick is read after the scene's update, so each frame lasts N / 60 s plus the time from the last poll to the next start, rounded up to the rounds: never exactly one vsync, on any hardware. The host's `SleepThread` and tick are exact; a model of the loop gives 54 fps with the host's acquire and with the Switch's double-buffered one alike. Found by sampling the host (`sample <pid>`: the main thread in `dm_guest_sleep_ns` under `main+0x443d74`, and in `nvnWindowAcquireTexture` under the present wrapper).
- **Fix:** a build cut on the wait's threshold (its `patches.py`): 0 when N = 1, where the game presents at interval 1 and the host's present pacing (the acquire waits) holds the frame one vsync; frame time p50 16.67 ms, wakeups 2,050 → 300 a second. Scenes with N = 2 still present at interval 1, so the wait stays theirs. Skipping the wait for every N, the first cut, presented 60 frames a second in them (28.8 with the wait, N forced to 2), for code that counts 60 / N frames a second.
- **Check:** before cutting a game's frame limiter, find what sets its target and whether anything else holds those frames. A limiter that also enforces a slower mode (here N = 2) has to keep it.
- **Found on:** [Taiko Risshiden V DX](../../dm/games/0100346017304000/README.md), in its menus and town; its movies, paced by their own player, ran at 60.

## Present interval and display rate

### Present thread rescues a silent display link

- **Symptom:** the app is active and the game keeps running, but the picture freezes while display-link callbacks stop; the log repeatedly drops queued frames.
- **Cause:** dropping frames releases the game's swapchain textures but never displays them. A display link may stop while the app is active, including an occluded Mac window.
- **Fix:** after 500 ms of silence while active, `mtl_present_wait` requests rescue. `link_sync` invalidates the display link on its run loop before releasing the layer to the present thread, which takes its own drawables in mode 0. Retry delays are 3, 6, 12, 24, 48 and then at most 60 seconds. Rescue state changes take `g_pq_mu`; the requested `g_dm_present_link` stays unchanged, and a new runtime mode ends rescue immediately. A retry resets stall timing. Inactive and background time do not count towards the 500 ms threshold, including a focus timestamp newer than the watchdog's clock sample.
- **Found on:** d2r-mobile commit `afad65c` (an iPhone picture froze for 80 seconds), carried through MacNativeGames `cf1e629` and [PR #198](https://zcml.ai-hao123.com/yanjiu/education-96985273.html). That upstream title has no profile in this repository. Mac window occlusion also affected [ANIMAL WELL's image validation](../../dm/games/010020D01AD24000/README.md#files): foreground rescue and out-of-focus fallback produced incomparable timings; its note records the controlled comparison. [Xenoblade Chronicles 2's controller motion review](../../dm/games/0100E95004038000/README.md#controller-motion-review) also records stalled-link timing on both hosts and the paired diagnostic runs.
- **Check:** `make -C dm/gfx/tests GAME=010015100B514000 t_present_rescue` and `dm/gfx/tests/t_present_rescue` exercise the production wait and retry paths with a controlled clock and a silent link double, without a GPU. They cover foreground grace, backoff saturation, runtime mode changes, and layer release only after run-loop invalidation. The test also runs in CPU graphics CI. `make -C dm/gfx/tests GAME=010015100B514000 check_t_present_rescue` adds a paused real Metal display link on an offscreen Mac layer: rescue invalidates it, then `nextDrawable` succeeds (47 checks, zero failures with Metal validation). `DM_TEST_LINK_STALL=<start seconds>,<duration seconds>` pauses a real window's display link for manual verification.
- **Not checked:** gameplay during stalled-link recovery or the iPhone; the PR's gameplay runs were headless.

### A present command allocator is reused only after its own present completes

- **Symptom:** with the display link driving presents, a presented frame comes out black when three presents are on the GPU at once, or two after a dropped frame; on an M6 the same reuse also logs `[RED] display command buffer error` with `kIOGPUCommandBufferCallbackErrorPageFault`.
- **Cause:** `present_now` cycles three Metal 4 command allocators, each with its command buffer and UV buffer, by ticket, and before resetting one it waited for `g_pq_done` to reach the ticket that last used it. `g_pq_done` is not GPU completion: every display-link callback sets it to `ticket - 1` (the previous present was committed, not run), and a present that was not submitted sets it to its own ticket. So a display-link present never waited there. With three presents in flight, or two after a dropped frame, the allocator was reset and its command buffer encoded again under a present the GPU had not run, and that present's drawable read back black.
- **Fix:** each allocator keeps the newest ticket its own commit feedback has reported (`g_present_alloc_done`), and its reuse waits for that. The display-link callback can now block in `present_now` until the GPU has run the allocator's previous present. `g_pq_done` keeps its meaning for the game's swapchain acquire.
- **Found on:** no game. Found by reading `NvMetalDisplayLinkDelegate`, then reproduced in `t_present_alloc` (below).
- **Check:** `make -C dm/gfx/tests GAME=010015100B514000 check_t_present_alloc` holds the Metal 4 queue behind a shared event and calls the display-link callback by hand, each present with a drawable of its own offscreen layer. It covers a dropped frame (tickets 1, 3, then 4 on allocator 1 again) and one frame per tick (5, 6, 7, then 8 on allocator 2 again). Before the fix, in both cases the reusing present returns while the earlier one is held and the earlier drawable reads back 0 in all 64 pixels (4 of 13 checks fail; on an M6 every run also logs the GPU page fault); with it the reusing present waits until the gate opens, and all 13 checks pass with Metal validation.
- **Not checked:** a game run hitting it, the iPhone, and the not-submitted path, which the test does not exercise.

### Frames of a dynamic present interval held on screen and at the acquire

- **Symptom:** "sometimes the speed increases when jumping" (Mario 1.2.1 on the iPhone): in `display` mode interval-2 frames were shown for 16.7 ms and the game visibly jumped ahead. "When pressing B, Link's move pace is speeding up" (Breath of the Wild on the iPhone); `old frames dropped by merging`.
- **Cause:** Mario times every frame (`main+0xcfd24`); after a frame that ran past one vertical sync it sets present interval 2 for the next frame *and* sets its own time step to the interval (a float at `main+0x36ceff0`), so that frame catches up 2/60 s. A user's 2-minute iPhone session switched 123 times, mostly for one frame. On the game side, the acquire waited for one display-link tick per frame whatever the interval, so the game kept producing frames every 16.7 ms while the screen held each interval-2 frame for 33 ms and dropped the one in between. BotW sprinting flaps between interval 1 and 2 for runs of 6 or more frames, and its VFR time step at interval 2 is a whole 30 fps tick, so those runs went at double speed.
- **Fix:** both sides, fixed:
  - the screen: in `game` mode (the default since 2026-10-15) the display-link callback keeps a frame whose period is over 25 ms on screen for its whole period before taking the next one (`g_link_hold_until` in `nvn_mtl.m`; the `display link up to tick` log line counts these ticks as `held for the present interval`), and the 30 fps target that drops the display link to 30 Hz (`nvn_game_fps` in `hle_nvn.c`) waits until interval 2 has held for a full second, so one-frame switches no longer flip the display-link mode;
  - the game: `mtl_link_wait_tick` now waits for as many ticks as the game's present interval (times 2 on the 120 Hz link, half rounded up on the 30 Hz link), as the Switch blocks the next acquire for N vertical syncs.
- **Found on:** [Super Mario Bros. Wonder](../../dm/games/010015100B514000/README.md) (1.2.1, iPhone: the display-link hold); [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (iPhone: the acquire).
- **Check:** `DM_TEST_PRESENT_LAG=90,30` (sleep 30 ms before every 90th present) makes Mario do this on the Mac; run it with `DM_WINDOW=1` to get the display link. `DM_NVN_TSV` writes each frame's interval as `@itv`.
- **Note:** the windowless Mac host was never affected by the acquire: there the acquire sleeps until the interval's slot.

### One frame per tick of a 120 Hz display link

- **Symptom:** with `present_interval display`, `DM_PRESENT_HZ=120` and `DM_PRESENT_LINK=2` (a 120 Hz display link) a game stays at 60 fps although its frame takes 8 ms; the display link logs about two ticks per present. With a requested 90 Hz, truncating the tick conversion instead allows 120 fps (reproduced through the real acquire with simulated link ticks).
- **Cause:** the acquire waits for the present interval's vsyncs on the display link (`mtl_link_wait_tick`), counted in 60 Hz vsyncs, so two ticks of a 120 Hz link per frame; display mode's interval is 1. Dividing the link rate by the requested rate once per acquire loses fractional ticks (120 / 90 becomes 1).
- **Fix:** in display mode above 60 Hz the acquire waits as many link ticks as one tick of the present rate takes (`link_wait` in `hle_nvn.c`, `mtl_link_wait_ticks`): one at 120 Hz. Fractional ticks carry across acquires: at 90 Hz, three frames wait four link ticks (1, 1, 2); a rate or pacing-mode change starts a new phase. Rates at or above the link's rate wait one tick.
- **Found on:** Batman: Arkham Knight (`0100ACD0163D0000`; its profile and note are in a PR of their own): 119.8 fps in the opening cutscene on an M2 Max.
- **Check:** `t_link_ticks` (also run in CPU-only CI): with a controlled clock and one tick delivered per timed wait, a 60 Hz vsync waits two ticks of a 120 Hz link and one tick of a 60 Hz one, and a wait in the link's own ticks takes one tick per frame. The real acquire checks display pacing at 60, 75, 90, 120 and 240 Hz, game intervals and frame-step caps, including the 60 and 30 Hz link modes. Exact tick counts, already elapsed ticks, spurious wakes and silent-link timeouts are checked without depending on OS scheduling: a sleeping ticker could miss the acquire's wall-clock timeout on CI. These checks do not measure a game's frame rate on a screen.

### Present snapshots: the game's swapchain texture free at once

- **Symptom:** frames miss the screen while the GPU stays at its lowest clock (444 MHz on an M2 Max); more work (a higher render scale) only makes the frames slower.
- **Cause:** a game with two swapchain textures gets one back only when the display link has taken its previous present, so each frame's CPU work starts on a tick and the GPU idles from the end of one frame to the first work of the next. macOS reads that idle time as room and keeps the clock low.
- **Fix:** present snapshots (`DM_PRESENT_SNAP=<n>`, a ring of n host textures, 2 to 4; off by default on iOS and Mac; `nvn_mtl.m` `snap_take`): each present is copied on the game's queue, right after the frame's work, into the next ring slot; the display link shows the copy, and the game's texture is free at once. The game runs up to a frame ahead of the screen, and a frame whose work runs long is absorbed instead of shown late. Acquire waits for ring capacity and counts it as pacing (`mtl_snap_acquire_wait`). Present never waits for a snapshot slot: if it is still busy (several textures acquired ahead of time, or a copy not finished), the original texture is queued with its usual acquire ticket. Waiting inside present held the game's locks while the screen was stalled.
- **Setting:** the iOS launcher's **Frame buffering** toggle is offered for NVN builds and remembers the choice. Its default is the profile's `present_snap` (off unless a game was tested with it on: Breath of the Wild, a steady 60 fps on an iPhone 17 Pro Max where it fell to 30 without). `Documents/env.txt` overrides it (`DM_PRESENT_SNAP=1` on, `0` off); restart the app after editing the file. The Mac uses the same environment variable and the same profile default.
- **Fix:** the frame step judges lateness by what the screen shows while snapshots are on: the on-screen interval of the frame 8 presents back, against its hold (`fstep_tick`, `nvn_fstep.screen`). Judged by its cost, a frame whose CPU time holds the game's waits on its own GPU work (which now runs a frame later) looked late at a GPU time of 0.75 vsyncs.
- **Note:** the cost is up to a frame more latency.
- **Note:** with Metal encoding on its own thread (`DM_NVN_ENC_THREAD`, the default on iOS) the copy is a block of the encode thread's own queue (`rec_own_push`), behind everything submitted before the present, so a present from another thread than the submitting one gets a snapshot too; the present waits only until that block is encoded (`snap_commit_wait`). The wait ledger `DM_NVN_WAITS=1` counts every CPU wait for the GPU or the encode thread (`[waits]` every 300 frames).
- **Fix:** suspend drains snapshot commits and their GPU event even when their display tickets were discarded (`snap_drain` in `mtl_drain`). Submission fences and upload completion do not cover these copies; previously suspend could return while the encoder still held a copy to submit.
- **Check:** `make -C dm/gfx/tests GAME=01007EF00011E000 check_t_present_snap` uses a gated encoder and GPU to check suspend, and a delayed present consumer to check nonblocking present, acquire pacing, fallback tickets and copied pixels after ring reuse. Runs with Metal validation on the Mac, with and without the encode thread; does not measure iPhone frame rate or latency.
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (Mac; on the iPhone 17 Pro Max with the encode thread a steady 60 fps at docked resolution, present to screen about 74 ms with a ring of 2 against about 46–60 ms without).

### Vulkan acquire and vsync event follow the display link

- **Symptom:** a steady game whose frame-rate counter wanders between 50 and 60 on the iPhone only; the `display link up to tick` lines show about as many empty ticks as frames dropped by merging: frames arrive two in one tick and none in the next.
- **Cause:** the Vulkan layer's acquire and the `vi::GetDisplayVsyncEvent` thread each kept a 60 Hz clock of their own, and Galaxy runs the Wii's VI retrace on that event, so its frames drifted against the screen.
- **Fix:** while the display link presents, both follow its ticks (`hle_vk.m` acquire → `mtl_link_wait_tick`, as NVN's; `hle_sdk16.c` vsync thread → `mtl_link_wait_vsync`), with the old clocks as the fallback.
- **Found on:** [Super Mario Galaxy 2](../../dm/games/0100FD8022DAA000/README.md) (Vulkan).
- **Note:** a Mac window measures nothing here unless it is on screen: macOS fires an occluded window's display link about once a second (`display link recovered: stalled 1017 ms`).

### Virtual 60 Hz ticks on screens that are not a multiple of 60

- **Symptom:** asked for 60 Hz, the system fires the display link on every third refresh of a 165 Hz screen, 55 Hz, so games paced by it ran at 55 fps (27.5 at 30).
- **Fix:** on 144 or 165 Hz Mac displays `link_range` (`nvn_mtl.m`) runs the link at the screen's rate and counts as ticks only the refreshes nearest a steady 1/60 s clock: 60 a second on average, each frame shown 12.1 or 18.2 ms at 165 Hz. A tick lies up to half a refresh off its slot, so two ticks can be one refresh nearer each other than their slots: the hold of frames over 25 ms ([Frames of a dynamic present interval held on screen and at the acquire](#frames-of-a-dynamic-present-interval-held-on-screen-and-at-the-acquire)) tolerates one refresh more (in a simulation of the callback without it, 30 fps frames held a third tick: 22.5 fps at 165 Hz, 24 at 144), and the frame step lets a frame on time show one refresh longer than its hold (`shown_slack`, `nvn_fstep.h`; a cheap 50 ms frame shows 54.5 ms at 165 Hz, and counted as late it kept Breath of the Wild at 20 fps for good). `DM_LINK_VTICK=0` keeps the system's choice.
- **Note:** iOS screens run at multiples of 60 and are not affected.
- **Found on:** PR #27 (its author measured at 165 Hz); [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md) (the 20 fps without `shown_slack`).
- **Not checked:** not yet run on such a screen here.

### Fullscreen switches the screen to a rate the frame rate divides

- **Symptom:** virtual ticks keep the average, but a frame still stays up two or three refreshes in turn (13.9 / 20.8 ms at 144 Hz), and Apple's HUD showed the frame interval jumping between them.
- **Fix:** `front_mac.m` `front_screen_match`, called from `link_sync` with the link's rate (60, 30, or 120 with `DM_PRESENT_LINK=2`): when the window is fullscreen on a screen whose mode is not a multiple of the link's rate, the host switches that screen to the mode of the same size with the highest rate that is (an XB273U at 2560x1440: 144 → 120 Hz for 60 fps), with `kCGConfigureForAppOnly`, so macOS switches it back when the host quits, crashes included (the PR's author checked: 144 Hz again after quitting); leaving fullscreen, or the screen, switches it back at once. A window keeps the screen's rate and the virtual ticks: the switch changes the whole screen for every app, and most external screens go black for a second or more. The link sets its range again whenever the screen's rate changes (`link_sync`; before, a move to another screen kept the first screen's range), with no virtual ticks after a switch. A screen with no such mode keeps the virtual ticks; `DM_SCREEN_MATCH=0` keeps the screen's rate.
- **Found on:** PR #79: the PR's author ran [Hades](../../dm/games/0100535012974000/README.md) at 144 Hz (switched in a window then): the HUD's frame interval line steady at 16.7 ms, where it jumped before.
- **Check:** here (a built-in 120 Hz screen, a multiple of every link rate) a harness drove `front_mac.m` asked for 50 fps: no switch in the window, 120 → 50 Hz in fullscreen, 120 Hz again on leaving it, and 120 Hz after quitting in fullscreen; at 60 fps and with `DM_SCREEN_MATCH=0` nothing switched.
- **Not checked:** the link's new range after a switch has not run here (Hades windowed and fullscreen paced at 60 as before).

### Frame latency 2 for the display link in fullscreen

- **Symptom:** on a ProMotion built-in screen, only in fullscreen, the frame step went 1 → 2 → 3 vsyncs on frames costing 0.1–0.6 vsyncs (`3 late frames; cost 0.32 vsyncs`) and the game presented 15 frames a second; a window, or the screen set to a fixed 60 Hz, did not do it. The `DM_NVN_TSV` rows show presents shown one 120 Hz refresh late (`@pres_at_ms` − `@link_target_ms` = 8.3 ms) with a skipped tick after each (`@link_gap_ms` 33 ms), and `@link_lead_ms` about 16 ms against about 50 ms in a window.
- **Cause:** in fullscreen the system shows the layer directly, and at frame latency 1 the display link calls back only one 60 Hz period before the target display time. There it shows some presents one refresh late and skips the link's next tick, though the callbacks run on time (16.2–16.6 ms ahead), the present takes about 0.1 ms and the misses do not follow GPU load (the late presents had less GPU work in flight than the on-time ones). The frames that wait for the skipped tick start late, and the frame step reads three of them in a row as load. Why the system misses at 16 ms of lead is **open**.
- **Fix (a workaround):** `link_latency` (`nvn_mtl.m`) asks for frame latency 2 while the window is fullscreen (`front_mac.m` `front_fullscreen`) and 1 in a window; `link_sync` sets it again when the window enters or leaves fullscreen. Latency 2 gives the system 33 ms, for one frame more input lag in fullscreen only. `DM_PRESENT_LATENCY=<n>` sets one latency for both. iOS keeps latency 1.
- **Found on:** issue #347 (and #348, Mario Kart 8 Deluxe on an M4, closed into it); [The Legend of Zelda: Tears of the Kingdom](../../dm/games/0100F2C0115B6000/README.md) on an M4 Pro: in fullscreen at latency 1, 68 to 1,640 of about 9,600 presents a refresh late per 180 s run, and in one run the 1 → 2 → 3 steps and 15 presents a second twice; at latency 2, 17 and 18 in two runs, no cheap step-ups. The reporter's M5 Max: 49 skipped ticks in 2,306 frames at latency 1, 4 in 5,328 at latency 2.
- **Note:** a 120 Hz link instead (`DM_PRESENT_LINK=2`, a frame every other tick, latency 1: 16.6 ms of lead) kept the frame step at its cost in three runs, but the system still showed 102–209 of about 9,600 presents one refresh late, against 17–19 at latency 2; so a steady panel rate does not remove the misses.
- **Not checked:** other games here; Mario Kart 8 Deluxe on the M4 with latency 2; screens other than the built-in ProMotion ones; entering and leaving fullscreen while a game runs (only `t_present_rescue`); `scripts/regress.py`.

## CPU cost and pacing

### Arena buffer ownership on allocation

- **Symptom:** autorelease work takes a measurable share of replay time, with temporary arena-buffer ownership traffic a few times per draw.
- **Cause:** under ARC an unqualified `id<MTLBuffer> *` out-parameter is `__autoreleasing`, so assigning the arena buffer in `arena_alloc` autoreleases it. Reading `buf.length` also sends an Objective-C message on each allocation and while selecting or trimming arenas.
- **Fix:** both declarations of `arena_alloc` use a `__strong` buffer out-parameter, matching every caller's strong local. `NvArena` caches its buffer length when created; allocation, reuse selection and idle trimming use that value. Buffer ownership and fence-based reuse remain the same.
- **Found on:** MacNativeGames commit `0531f28`, ported in [PR #195](https://jrii.ai-hao123.com/ziyuan/forecast-71880394.html), reports autorelease work falling from 2.63% to 1.73% of replay time; the PR does not identify the game for that measurement. It has not been repeated here.
- **Check:** `t_ub`, `t_vbuf` and `t_geometry` exercise arena-backed uniform shadows, vertex conversion and geometry scratch on the Mac with Metal validation.

### Registry lock priority inheritance

- **Symptom:** the submit thread waits on the texture / sampler registry lock while a lower-priority registration thread is preempted holding it.
- **Cause:** game threads register textures and samplers under `g_rmu`, which the submit thread also takes for argument encoding. The previous mutex allowed those owners to delay the higher-priority submitter.
- **Fix:** `g_rmu` is an `os_unfair_lock`, lending a waiter's priority to its owner. All registration, lookup, cache, diagnostic and test callers use its lock API; the critical sections and lock order stay the same.
- **Found on:** [The Legend of Zelda: Tears of the Kingdom](../../dm/games/0100F2C0115B6000/README.md), measured upstream in MacNativeGames commit `5f5de5e` and ported in [PR #195](https://www.siseo.cn/index.html): in a shrine, game threads at priority 46 delayed the submit thread at priority 97 by about 1.8 ms a frame. Waiting fell from 155 to about 0 ms/s there, with unchanged frame time. Those game measurements have not been repeated here.
- **Check:** `t_stencil_view`, `t_view_levels` and `t_view_swizzle` exercise production registry lookups and cached views with the new lock on the Mac.

### Hash-table lookups of 8-byte keys

- **Symptom:** in a replay profile, `_platform_memcmp` is most of the time spent in serial lookups (`tex_mtl` through `g_by_serial`).
- **Cause:** `nvn_hm_get` compared each probed key with `memcmp` and the table's run-time key length, a libc call per probe. The serial, live-serial, ghost, view-cache and compute-pipeline tables have 8-byte keys and are looked up several times per draw.
- **Fix:** for 8-byte keys `nvn_hm_get` loads the key as one word and compares words. The hash, probe order and answers are the same; puts and deletes are unchanged.
- **Found on:** MacNativeGames commit `7431eb7` (38 of 55 samples of the serial lookup were `memcmp` in a Tears of the Kingdom profile). Here, a lookup in a 3,000-entry table of 8-byte keys takes 1.5 ns instead of 5.9 ns on an M4 Pro (`clang -O2`, 60 million lookups); no game frame has been measured.
- **Check:** `t_hmap` checks 8- and 20-byte keys against a naive model through random puts, overwrites, deletes and clears.

### Submissions handed to the GPU in segments

- **Symptom:** the GPU sits idle each frame while the host translates the submission (a game that does not submit frame N+1 until frame N's GPU work is done).
- **Cause:** on the Switch the submit costs nothing; here it is the host's translation, and the GPU got nothing until the whole submission was encoded.
- **Fix:** the replay commits the command buffer encoded so far every 64 draws, between two passes (`sub_cut` in `nvn_mtl.m`, called after `rp_flush` in `OP_SET_RT`; `DM_NVN_CUT=<draws>`, 0 = off). It cuts only with no open encoder and nothing deferred, merged or hoisted, commits the pending upload batch first, and continues on the same command allocator. The fence signal, the completion work (`post` ops, `g_done`) and the allocator's fence stay on the last segment: the queue runs command buffers in commit order, and every encoder starts with a queue barrier.
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md): a stack sample in the shrine had the main thread in `nvnSyncWait` (the fence of the frame just submitted) for 59% of the time and the render thread in `nvnQueueSubmitCommands` for 40%; the translation was about 3,600 draws and 347 passes, 7 ms, so per frame the GPU ran 13–14 ms and then sat idle 7.7 ms (`DM_NVN_TSV` rows `@gpu_beg_ms` / `@gpu_end_ms` against `@call_ms`). GPU idle per frame went from 7.7 to 3.8 ms; most of the rest is the game's own work between its fence wait and the next submit (about 2.5 ms). The other five games' frame times were unchanged; Metroid Dread's p95 dropped from 19.6 to 16.9 ms.
- **Note:** after this, BotW's docked frame p50 was 16.7 ms (was 20.8 ms before segmenting). Host translation is 8 ms per frame (22,000 instructions per draw), spread over Metal's encoder creation and draw calls, argument encoding, Objective-C retain / release and pool lookups. The per-pass GPU timer (`DM_NVN_PASSGPU`) is switched off under Metal 4 (`pg_render` returns at once), so GPU time cannot yet be attributed to passes. Render-target traffic per frame is in `DM_NVN_TSV`'s `@rt_load_mb` / `@rt_store_mb`.
- **Note:** with the encode thread (`DM_NVN_ENC_THREAD`) a segment's commit record is handed to it at once (`rec_publish` in `sub_cut`), not when its block of records fills. Breath of the Wild on an M2 Max (encode thread and snapshots, `DM_FPS_TARGET=120`): the frame's wait on its newest fence 12.5 ms with `DM_NVN_CUT=0`, 5.6 ms with 64, 5.35 ms with 16.
- **Note:** MacNativeGames moved its cut to 30 draws for the first segment and 40 after (`2fa27d7`; uncapped Tears of the Kingdom in a shrine, 10.77 → 10.60 ms a frame). At this repository's 60 fps cap the size changes nothing where the GPU finishes inside the frame: Breath of the Wild outdoors on an M4 Pro (2026-10-26, the outdoor seed of the [BotW Mac routes](../../dm/games/01007EF00011E000/README.md#mac-routes), frames 3,001–5,700, cut 64) waited on its fence 0.02 ms a frame on average (`@sync_ms`), with GPU span 10.8 ms. The default stays 64; a GPU-bound device (an iPhone) is where a smaller cut could show, through `DM_NVN_CUT`.

### The set of CPU-written textures hashed by serial

- **Symptom:** the submit thread's CPU samples half in `__CFBasicHashFindBucket_Linear` under `CFSetContainsValue` (from `tex_mtl_` and `tex_by_id`), a few hundred thousand to over a million instructions of translation per draw, the submission growing with the game's CPU-written textures.
- **Cause:** the set of textures the CPU writes (`tex_dirty_upload`, `mtl_tex_cpu_written`) was a CFSet with no callbacks, which hashes a serial as the number it is; serials come in order, so they filled one run of neighbouring buckets, and each lookup of a texture not in the set (nearly every one) probed to the end of the run. Released textures stayed in it.
- **Fix:** the host's own table (`nvn_hmap`, which mixes the key). `mtl_tex_release` removes live membership and the `packaged textures` log line counts live entries. Recorded commands and registered descriptors can still use a finalized texture, including one written after recording: its CPU-write classification moves to a retired serial-to-pool table and remains until that pool is finalized. Both marking and release lock `g_smu` before `g_cpu_tex_mu`, so a dirty-page poll marking an already finalized snapshot cannot put it back in the live set. Retired metadata can accumulate while a pool remains alive; earlier eviction requires tracking snapshot ownership.
- **Found on:** Batman: Arkham Knight (`0100ACD0163D0000`; profile not present here), reported in [PR #321](https://auqc.tcti.cn/ziyuan/revenue-15211718.html). The author's gameplay measurements have not been independently reproduced here.
- **Check:** on an M4 Pro, three runs of one million misses with 6,442 consecutive serials and the same mutex took 1,693–1,707 ms with the callback-free CFSet and 9.6–10.0 ms with `nvn_hmap`. `t_cpu_tex` checks membership, duplicates, release, late dirty-page marking, pool cleanup and lookup cost without a GPU; `--metal` also records a texture-to-buffer copy and descriptor registration, writes before or after recording, finalizes and overwrites the API object, and verifies the pixels on replay. The PR's unconditional deletion failed four pixel checks; preserving retired classification passes them.

### Geometry boundaries submit prepared work early

- **Symptom:** several milliseconds of CPU translation and geometry preparation separate one frame's GPU end from the next frame's GPU start, even though the frame already contains complete geometry passes.
- **Cause:** segmented submission was checked only at the game's render-target changes. Geometry replay creates additional capture/compute/raster encoder boundaries inside one target binding, but their commands stayed on the CPU while the next run was prepared.
- **Fix:** after closing the previous geometry run's encoders and landing deferred updates, check the existing `sub_cut` policy before preparing the next run. Its draw threshold, deferral exclusions, upload flush, final-fence lifetime and segment timestamp handling remain in force. This adds submissions at existing encoder boundaries without adding render passes. `DM_NVN_GEOMETRY_CUT=0` disables the additional checks; `DM_NVN_CUT=0` still disables draw-count cuts overall. GPU dependency and pixel checks pass with either policy.
- **Found on:** [NieR:Automata US](../../dm/games/010056B015FE8000/README.md#library-shadow-sequence-and-geometry-ordering). In an alternating warmed workload of 1,024 geometry/query draws at 1080p, median CPU-encode-to-GPU-completion time falls from 11.605 to 7.800 ms; render passes stay at 17 and all query results agree. GPU execution is essentially unchanged (4.004 versus 4.007 ms), while CPU encoding rises from 6.971 to 7.268 ms: the gain is overlap, not reduced shader work. Gameplay improvement remains unmeasured.

### Reusing upload staging after a slot completes

- **Symptom:** CPU submission spikes during texture updates, with the residency log showing tens of MB of new private staging allocations per frame.
- **Cause:** an upload slot retained a block sized for the first texture in its batch. A small first texture left it at 4 MB, so later textures repeatedly allocated and retired 32 MB overflow blocks even when the same batch shape repeated.
- **Fix:** the slot's next primary block grows to its previous batch demand (capped at 32 MB), and it also retains one 32 MB overflow block. Reuse still follows `up_begin`'s wait for that slot's previous GPU batch. Each retained block is used at most once per batch; larger bursts and individual uploads over 32 MB keep the temporary, deferred-retirement path. Retained capacity is bounded at 64 MB per slot / 192 MB overall. `DM_NVN_UPLOAD_REUSE=0` restores the original policy. `t_upload_slots.m` checks distinct GPU-copied bytes across pending batches, an event-gated slot wait, warm allocation reuse, overflow beyond the budget and oversized allocations; both policies pass.
- **Found on:** [NieR:Automata US](../../dm/games/010056B015FE8000/README.md#upload-stability-follow-up). This removes allocation churn in the controlled workload; a gameplay comparison with the updated build is pending.

### Small array texture uploads without per-face staging

- **Symptom:** periodically rewritten textures cause CPU submission spikes even with no new staging allocations or shader compilation; a small cube-array image produces thousands of deswizzle dispatches and buffer-to-texture copies.
- **Cause:** the staged upload path encodes a deswizzle dispatch and a copy for every face and mip. Face count can dominate encoding cost even when total pixel data is small.
- **Fix:** mapped, non-sparse RGBA16F 2D/cube arrays with at least 32 faces and dimensions at most 64×64 write all faces in one dispatch per mip. An RGBA16Uint view preserves the exact stored component bits, including half-float NaN payloads, without floating-point conversion. Temporary views live until GPU completion; direct targets participate in dropped-upload recovery. Initial fills retain their original texture and usage contract. Large images, other formats, linear/compressible layouts, multisampling and unaligned sources keep the staged path. `DM_NVN_DIRECT_UPLOAD=0` restores it for all uploads. `t_direct_upload.m` compares all mip/face bits after CPU edits and spare reuse, checks fallback and initial fills, and injects a lost direct upload; Metal validation passes 26,384 checks normally and 26,388 with recovery injection.
- **Found on:** [NieR:Automata US](../../dm/games/010056B015FE8000/README.md#small-light-probe-array-uploads). Its 16×16×768, five-mip RGBA16F array needs five direct dispatches instead of 3,840 staged dispatches and 3,840 copies. In an alternating warmed benchmark on M4 Pro, CPU upload encoding plus commit falls from 3.916 to 0.0275 ms and upload GPU time from 1.802 to 0.0355 ms. Gameplay and terminal-screen improvement remain unmeasured.

### Address to memory pool lookups by binary search

- **Symptom:** the submit thread's CPU samples dominated by `mtl_addr` / `mtl_gpu_addr` (vertex buffers in `draw`, uniform buffers in `encode_args`).
- **Cause:** with many memory pools alive, the lookups scanned them all for every buffer binding of every draw.
- **Fix:** they try the last hit, then a binary search over pools sorted by start address (`pool_find`, rebuilt when pools are added or removed).
- **Found on:** [The Legend of Zelda: Breath of the Wild](../../dm/games/01007EF00011E000/README.md): about 1,760 memory pools alive; a CPU sample of the submit thread had about 1,000 of its samples under those scans. Host submit time in the shrine went from 21 ms to 7 ms per frame (p50), the median frame from 38 ms to 21 ms.

### Main guest thread keeps inherited default QoS

- **Symptom:** gameplay has low CPU clocks, large render submission wall time compared with thread CPU time, and default QoS (`0x15`) on the main guest thread while workers report user-interactive (`0x21`).
- **Cause:** worker initialization applied the guest QoS policy, but `svc_thread_main_init` only recorded the boot pthread's inherited class. On iOS that class can be default. A time-constraint policy may hide this difference until it is disabled for measurement, and does not establish the intended underlying QoS.
- **Fix:** main-thread registration calls the same `qos_apply` used by worker threads, before recording the actual class, and logs the resulting class. With diagnostics enabled, `DM_NVN_CPUPROF=1` measures inclusive CPU/wall time in pass begin/end, pipeline lookup, argument encoding, texture lookup, draw, geometry, and submission indexing. These nested phase times overlap. The thread regression confirms `0x21`; on-device performance remains to be validated.
- **Found on:** [NieR:Automata US 1.0.3](../../dm/games/010056B015FE8000/README.md), where the slow gameplay captures show 159–160 million host translation instructions per frame, considerable scheduling delay, nominal thermal state, and ample memory headroom.

### Time-constraint computation declared below the thread's peaks

- **Symptom:** stalls of 30–60 ms about once a second, at 30 fps and at 60 (99th percentile 63–127 ms); the main thread waits 20–34 ms for the render thread once a second.
- **Cause:** the thread that acquires the window (here the render thread) runs under a time-constraint policy whose computation the host re-declares every 120 frames; it was 1.25 × the 90th percentile of that thread's CPU time per frame. This render thread needs 1.9–2.0 ms a frame with one or two frames in 120 near 4.4 ms, against 2.5 ms declared, and each frame past the declaration stalled the thread. Probes on both threads' waits showed the main thread's waits; adding probes slowed the render thread and the stalls vanished, and `DM_RT=0` or a fixed `DM_RT_COMP=5` / `10` removed them too (99th percentile 18–21 ms) at the same CPU energy.
- **Fix:** the declaration also covers 1.1 × the window's peak (`rt_comp_tick` in `hle_nvn.c`): 5.6 ms here, 60.0 fps, 99th percentile 21 ms.
- **Found on:** [DARK SOULS: REMASTERED](../../dm/games/01004AB00A260000/README.md), in the Asylum cell.


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

