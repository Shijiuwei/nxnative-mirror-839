# 05 — Debugging recipes

## Reading a crash report

```
[dm] CRASH sig=11 addr=0 pc=main+0x373e04 lr=main+0x373dd8 sp=0x7066dd6370
x0=… x28=…
[dm] bt: main+0x373dd8 main+0x2ea2ac …          frame-pointer backtrace
[dm] fp=…  0x100 bytes below sp: …                 the 32 words below sp (the frame that was just popped)
[dm] stack scan sp=…: [-72]main+0x57a14 [+24]main+0x57618 …   every word near sp that resolves to a module or host symbol
[dm] thread 0x1001 'main thread' …                  all guest threads, what they wait on, their backtraces
```

- `pc` and `lr` are module-relative (`main+…`, `sdk+…`) or host symbols. Module offsets are the same as in the decompressed NSO and the ELF (Ghidra adds 0x100000).
- Look at the registers before the code: a pointer that is a valid pointer plus garbage high bits, an index of `0xfffffffe`, or `pc == lr == 0x100000000` each tell a different story. a fault at `0x74a740e114 = 0x70a740c570 + 0xfffffffe·4 + 0x1bac` was an index of −2 widened as unsigned; `pc == lr` on a nonsense value is a `ret` through an overwritten saved LR (look at the zeroed frame below `sp`).
- The handler takes SIGSEGV, SIGBUS, SIGILL, SIGTRAP, SIGALRM (the run's time limit) and SIGABRT (`abort()`: an assertion or an uncaught exception in the host or a framework; before it was handled, Big Brain Academy: Brain vs. Brain died once with exit 134, nothing in the log and no report). A run that ends with exit 137 and no `CRASH` line was killed with SIGKILL: macOS does that after a trap in a system library (`_os_unfair_lock_corruption_abort` destroying a corrupted mutex, the movie decoder entry [Input indices returned by a flush while a client holds them](../reference/movies.md#input-indices-returned-by-a-flush-while-a-client-holds-them)), and its report is in `~/Library/Logs/DiagnosticReports/dmhost-*.ips`, a JSON document whose `faultingThread` names the thread whose `frames` to read.
- When the frame-pointer backtrace is empty, the stack scan's negative offsets (below `sp`) show the return addresses of the calls the crashing function made last.
- `[dm] *xN:` lines show the first six words each register points at (read safely, so a bad pointer is skipped). A word inside a module prints as `main+…`: the first word of an object is its vtable, which names its class, and comparing it with the class's own vtable shows whether the object was destroyed (a base-class vtable) or rebuilt.

## Pipeline compile waits

Render-pipeline lookahead runs after the demanded pipeline is available and cached. It admits at most two predictions per scan and at most two scheduled or running predictions overall, so speculative attachment variants cannot create an unbounded compiler backlog ahead of real draws. Exact-key results are still recorded only when used; a full pending table may discard completed unused results, never running jobs. The independent startup warmer is not subject to this limit. A new pipeline can still stall on its intrinsic first-use compilation cost; compare pipeline-miss timings and live stacks with asset reads before attributing a hitch to loading. `dm/gfx/tests/t_pso_pending.m` exercises the queue and demand-before-prediction ordering with a gated fake compiler.

## Ghidra / objdump

```bash
# decompile the functions containing these module offsets
JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home \
  /opt/homebrew/opt/ghidra/libexec/support/analyzeHeadless $W/ghidra/proj <proj> -process <module>.elf \
  -noanalysis -readOnly -scriptPath <repo>/dm/tools/ghidra -postScript Decompile.java 0x373e04,0x2ea2ac

# raw disassembly (works where Ghidra defined no function)
objdump -d --start-address=0x2ea1c0 --stop-address=0x2ea2b0 $W/elf/<module>.elf
```

Prefix an offset with `@` (`... Decompile.java @0x49740`) to (re)create the function at exactly that address. Code reached only through vtables has no function, and Ghidra's analysis marked some `nn::os` imports (`LockMutex`) as non-returning, which cut many functions short right after the first lock. The script clears that flag and the call-return overrides on such calls before decompiling.

Map GOT slots to NVN names with the survey table (`census/*_nvn_slots.tsv`): a call through `[x8, #0x3e8]` after `adrp x8, 0x35ad000` is GOT entry 0x35ad3e8.

## Probes without a debugger (`DM_PROBE`)

`DM_PROBE=main:4290c0,main:9712c` (module:offset, comma-separated; `host/probe.c`, Mac only) replaces the instruction with `brk`, and on every hit prints the guest thread (`T=`), x0–x7 (and strings they point to), x19–x23, lr, x8 and a backtrace, then runs the original instruction from a trampoline. Put probes at function entries or where x16 is dead; the probed instruction must not be PC-relative (no `adrp`, `adr`, `b`, `bl`, literal `ldr`: probe the next instruction instead). Each hit prints as one block even when several threads hit at once. Probes slow the game, which can shift heap addresses between runs.

Per-probe suffixes, in this order:

| Suffix | Effect |
|---|---|
| `@off`, `@*off`, `@xN+off`, `@*xN+off`, `@*xN+off+add` | Print 16 bytes at x0+off (or xN+off); `*` dereferences that address first, and `+add` moves on that many bytes from the pointer read (`@*x0+48+8` on an `al::LiveActor` is its position, `al::getTrans`) |
| `?xN=val`, `?[xN+off]=val` | Only report hits where the register (or the 64-bit word at xN+off) equals `val` (hex); others pass silently. E.g. `main:4743c0?[x0+30]=781548` = only allocations into the heap whose size field is 0x781548 |
| `!xN` | Treat xN as a `sead::ExpHeap` and print its free and used blocks (offset+size, `p` + alignment pad) |

The `!xN` heap walk takes the layout of the game's sead version from `DM_PROBE_HEAP=<name>,<start>,<size>,<free list>,<used list>` (hex offsets in the heap object; default Mario's `0,28,30,b0,c8`, Breath of the Wild's `28,38,40,e0,f8`; find them in the heap's constructor and `tryAlloc`). `DM_PROBE_HEAPNAME=<text>` reports only hits whose heap name contains the text (resource heaps are named after their file), `DM_PROBE_HEAPWORDS=1` adds the first two words of each used block (a vtable names the object), `DM_PROBE_HEAPFRAME=<vtable offset in main>` prints heaps of that class as frame heaps (head, tail, free), and `DM_PROBE_HEAPRAW=1` dumps the heap object's first 0x130 bytes, to find another class's fields.

`DM_PROBE_ALL=1` adds x9–x18; `DM_PROBE_X2=<bytes>` dumps the memory x2 points to.
`DM_PROBE_FRAMES=3000-3001` restricts observations to those presented frames
(a single frame is also accepted); `DM_PROBE_LIMIT=20` caps printed hits per
probe. These controls bound a late-game observation without logging startup.

**Who writes a word: `DM_PROBE_WATCH=<address>[+<length>]`** (hex guest address, 8 bytes by default; the bytes must lie in one 16 KB host page). The host write-protects that page as soon as it is writable and keeps it protected: the fault handler carries out every store that lands on the page through a second, writable mapping of it (STR / STUR / STP of general and SIMD registers in their immediate, pre- and post-index forms, register-offset STR, DC ZVA), so the stores of a busy page all pass through it. A store that reaches the watched bytes prints `WATCH … store of N bytes at …: old → new`, the thread, pc, lr and a backtrace. Other instructions (atomics, exclusives) get a `not emulated` line once per pc and run on an unprotected page, protected again a millisecond later. A guest object sits at the same address in every run (the guest heap is laid out the same), so take the address from a probe's register dump first. STEINS;GATE RE:BOOT's render size setting (engine object + 0x6000): the static search for stores with that offset, conditional probes on 25 candidate stores and a wrapper around the host's `memcpy` / `memset` (the guest's go straight to the host's libc, `hle_table.c`, where guest probes cannot see them) all found nothing; a watch that unprotected the page for every fault missed the write among the bursts of stores to the engine object's page; this one found it on the first run (`main+0x389e90`, a setter its start-up code calls with 0).

An exclusive or atomic store (`STXR`, `STLR`, `CAS`, `SWP`, `LD<op>`) that reaches the watched bytes prints `WATCH … exclusive store` (or `atomic store`) with its pc and backtrace, every time, though it is not emulated. A page that takes many atomics stays unprotected most of the time, so even that can miss: Tears of the Kingdom's memory-shortage flag (a word its frame check compares with 1 and clears) shares its page with atomic counters, and the watch saw neither the write nor the check's own clear.

**When the watch misses: `DM_PROBE_POLL=<address>[+<length>]`** (hex guest address, 4 bytes by default, up to 8). A host thread spins on the bytes; on each change it stops every thread at once, reads each one's pc, lr and 8 frame-pointer frames, lets them run again, then prints `POLL thread …` for the threads with guest code in those frames, the change (`POLL … old → new`) and, for the first two changes, the full guest thread dump. The snapshot comes a few microseconds after the store; a writer that went on into a wait shows only its wait (for that flag the one thread in guest code was `PlacementMgr` in `main+0x5aaac0`, both times). It costs one core while it runs. A word that changes thousands of times before the bad store takes a value filter, `DM_PROBE_POLL=<address>+<length>=<value>`: every change is still logged, but the threads are snapshotted and dumped only at changes to that value (Mario vs. Donkey Kong: a heap block's list link, rewritten by every allocation beside it, until an output index landed in it, [AAC decoders offer at most 32 outputs](../reference/movies.md#aac-decoders-offer-at-most-32-outputs)).

**A heap block overwritten by something else (Unreal):** the crashes move around between runs, in allocator code or on pointers that look fine one run and bogus the next. Unreal's own `-purgatorymallocproxy` (on the game's command line: `DM_ROMFS_OVERLAY` on a folder with `UE4CommandLine.txt`, the game's line with the switch added) fills freed blocks with 0xDC, keeps them back for a few frames and checks them before the real free; a probe on the branch that reports a modified byte gives the block, the offset and the value. Making the parked blocks read-only (a temporary probe handler in `probe.c` on the proxy's park and release points, `mprotect` of the block's whole 16 KB pages) then faults the stray store itself, with its backtrace. Yoshi's Crafted World: a texture mip copy that wrote past its upload buffer ([Sparse textures of 8- and 16-byte blocks](../reference/rendering.md#sparse-textures-of-8--and-16-byte-blocks-blocks-at-most-8-gobs-high)), 3 of 3 runs.

A probe's backtrace reads each frame with `vm_read_overwrite`: code built without frame pointers keeps data in x29, and the walk used to fault in the host on such a value (x29 = 0x430 at `main+0x38921c` in the same game).

Use it when lldb is unavailable: after macOS revokes the debugger authorization (`DevToolsSecurity -status` says developer mode is disabled), `lldb` launches hang waiting for a GUI approval prompt. `sudo DevToolsSecurity -enable` (run by the user) restores it.

## lldb

The host uses SIGSEGV for dirty-page tracking, so lldb must pass it through without stopping. Address randomization is off under lldb, so guest addresses repeat across runs. Template (`$W/dbg/*.py`, run with `perl -e 'alarm 240; exec @ARGV' lldb -b -o "command script import x.py"`):

```python
import lldb
HOST = '/path/to/dm/host/dmhost'
def __lldb_init_module(dbg, _):
    dbg.SetAsync(False); ci = dbg.GetCommandInterpreter(); res = lldb.SBCommandReturnObject()
    ci.HandleCommand('settings set platform.plugin.darwin.ignored-exceptions EXC_BAD_ACCESS|EXC_BAD_INSTRUCTION', res)
    t = dbg.CreateTarget(HOST); t.BreakpointCreateByName('run_module_inits')
    li = lldb.SBLaunchInfo(['<patched dir>']); li.SetEnvironmentEntries(['DM_ROMFS=…', 'DM_SAVE=…', 'DM_TEMP=…'], True)
    p = t.Launch(li, lldb.SBError())
    for sig in ('SIGSEGV', 'SIGBUS', 'SIGUSR1', 'SIGUSR2', 'SIGPIPE'): ci.HandleCommand('process handle %s -n false -p true -s false' % sig, res)
    base = None
    while p.GetState() == lldb.eStateStopped:
        fr = p.GetSelectedThread().GetFrameAtIndex(0)
        if base is None and fr.GetFunctionName() == 'run_module_inits':
            base = t.EvaluateExpression('(unsigned long)g_mod[0].base').GetValueAsUnsigned()   # g_mod[0] = main
            t.BreakpointCreateByAddress(base + 0x2ea228)                                       # module offset
        elif base is not None and fr.GetPC() == base + 0x2ea228:
            print('x0=%#x' % fr.FindRegister('x0').GetValueAsUnsigned(), flush=True)          # flush: a timeout kill loses buffered output
        p.Continue()
    p.Kill()
```

Patterns that worked: print every call of one function with its arguments; stop when a return value is null and print the backtrace; trace the branch path through one function (one-shot breakpoints on every branch site). Hardware watchpoints on words touched by `ldxr`/`stxr` may stay silent.

## Scripted input and thread scheduling

- `DM_KEYS="<seconds>:<button>[+button][~hold],…"` presses buttons from host start (`hle_hid.c`). `<seconds>:<button>*<count>/<interval>` repeats a press, e.g. `42:A*170/2` presses A every 2 s from 42 s on, 170 times; at most 1,024 steps after expansion. Steps run in time order, so repeated groups interleave (`f17400:Left~0.3*120/500,f17480:A*120/500,f17560:Left~0.3*120/500,f17640:A*120/500` = Left, A, Left, A every 500 frames: Fire Emblem: Three Houses' confirm dialogs default to No, and this cycle confirms Yes whether a dialog is open or not when it starts; A, Left, A loops forever once it starts on an open dialog); a list that mixes seconds and frame numbers keeps its listed order. A long run through a game's story is then a single switch, such as Metroid Dread's title, new game and story recap. `LLeft` / `LUp` / `LRight` / `LDown` push the left stick fully (Super Mario 3D World moves on its map by stick only); `Shake` shakes the controller for the hold time, at least one whole shake (its six-axis sensor); combos such as `L+R~0.5` press buttons together. Scripted presses are player 1's, as is the keyboard, on a path of their own that is ORed with the keyboard and player 1's controller (`script_set`): a connected controller (a DualShock 4 publishes its sticks all the time) no longer overwrites them. An `f` before the time counts presented frames instead of seconds (`f2500:A`, repeat intervals in frames too): with another heavy job running, seconds drift away from the game's progress and a route stops lining up; frame numbers do not. Breath of the Wild from its title menu into the autosave: `DM_KEYS=f2500:A,f2800:A,f3100:A,f3400:A` (control at frame ~3900).
- `DM_CTL=<file>` (Mac) drives a running game step by step instead: the host polls the file every 100 ms and acts on each line appended to it (`hle_hid.c` `ctl_main`). A line is a press as in `DM_KEYS` (`A`, `LDown~0.2`, `L+R~0.5`), or `shot <path>` (the next presented frame as PNG), `passdump <n>` (the next n frames as `DM_NVN_PASSDUMP`), `texdump <path with %d>` (the next frame's colour targets as `DM_NVN_TEXDUMP`), `texraw <serial> <path>` (level 0 of the host texture behind a serial, raw, with a `width height format row` header), `texlist <NVN target>` (registered textures of that target, `2` = 3D, with storage address, size and block-linear layout), `memdump <address> <bytes> <path>` (guest memory to a file: a buffer texture's storage from its `B` line holds what the GPU wrote), `peek <address> <count>` (floats to the log), `cvarf` / `cvari <main offset> <value>` (an Unreal console variable's game- and render-thread copies, for a `TAutoConsoleVariable` found as for `hle_fpslock.c`), or `env KEY=VALUE` (for switches read at use). Look, press, look again: menus and loads whose timing moves between runs need no blind timed script. A shell helper that appends a line and waits for a shot to be written keeps the loop short (DRAGON QUEST VII Reimagined's work folder: `run/ctl.sh` with `wait S` / `snap NAME`, and route scripts on top of it into the opening cutscene).
- Frame numbers are a stable way to aim dumps (`DM_NVN_PASSDUMP=<frame>`, `DM_NVN_SHOT=…`) at the same scene in another run: the per-frame NVN call counts (`[nvn] frame N: … calls`) matched between runs of the same script. Times in seconds drift with load.
- **A route through a 3D level** (Captain Toad: Treasure Tracker's opening course, where Toad has to find his way up a diorama to the star): screenshots alone took a dozen runs and still lost Toad behind the scenery, and turning the camera mid-route made the same input land differently each run. What worked: read the player's position every frame with a probe and plan from the stage's own data. The guest heap is laid out the same in every run, so the player actor is at the same address. `DM_PROBE=main:1fb2e0@*x0+48+8` (`PlayerActor::control`, the actor's pose at +0x48, the position 8 bytes into it, as `al::getTrans` reads it) prints x, y (height), z as three floats per frame. The stage map (`StageData/<stage>.szs`, a BYML) lists every actor with its `Translate`, including the goal and the rail the guide character walks. Stick directions are camera-relative: on the default camera two directions together (`LUp+LRight`) move along one world axis. Hold against walls where possible, since a wall stops the player at the same place in every run; the route then reproduced in 5 of 5 runs.
- **Spin waits** (`patch_nso.py` "Spin waits", `dm_spin_wait` in `hle_thread.c`).
  - **Switches:** `DM_SPIN=0` makes the gate return at once, so the loops poll with no sleep; the gate itself stays. `DM_SPIN_POLL=<µs>` sets how long a wait polls before it yields: 2000 by default, which is where the yield back-off starts sleeping. `DM_SPIN_LIST=1` makes the rewriter print every loop it found.
  - **Logs:** `sites.tsv` lists each loop as `<head> S <back-edges>`. In the power line's top yielders, a thread's spin passes show as `spin passes N, last at main+0x…`; the site is the loop head.
  - **Checking a loop:** to tell whether one is a real wait, disassemble from its head to its last back-edge. It must reload what it compares on every pass.
- **Who runs during a hitch.** Guest threads carry their guest names as host thread names (`0x1001 main thread`, `0x1008 Worker`), so `sample <pid>` and `xcrun xctrace record --template 'Time Profiler' --attach <pid>` (no root needed) name them; the export (`xctrace export --xpath '/trace-toc/run[@number="1"]/data/table[@schema="time-profile"]'`) gives each sample's time, thread and backtrace, and guest frames map to `main+…` with the module bases at the top of the log. The trace starts at the TOC's `start-date`; the TSV's `@call_ms` counts from the process start (`kinfo_proc` `p_starttime`), which lines the two up. `DM_HITCH_DUMP=<ms>[:<s>]` logs every guest thread once whenever no frame has been presented for `<ms>` (from `<s>` seconds after the start on, at most 24 times): what it waits on, its last SVC, and where it is, with x0–x2 and x19–x25 when it runs guest code. Breath of the Wild's CPU hitches showed up there as the main thread and two workers in one guest loop with no SVC, and the registers told which thread the loop waited for (its item index in x0; [docs/development.md](../development.md), Spin loops).
- `DM_PREEMPT=0` / `1` disables / enables the additional check of already-runnable same-core peers after every thread SVC, overriding the profile's `core_preempt` (`hle_thread.c`). Wake-triggered handoffs apply with either setting; they wait only for the threads the wake made runnable (and those these wake in turn) until each blocks. The thread dump reports `same-core handoffs N, total T ms`; these waits have no timeout that releases unfinished guest work, and a total near or above the run's length means a handoff waits for threads it should not. See [Same-core wake handoffs](../reference/threads.md#same-core-wake-handoffs) for the runnable-state rule and regression test.

## Racy crashes: load, trace gates

A crash that needs an unlucky thread interleaving (one run in three, or once in a long session on the phone) gets much more frequent when the CPU is busy, because threads are descheduled at more places. 12 busy shell loops (`for i in $(seq 12); do (while :; do :; done) & done`) beside a scripted Mac run made Link's Awakening's physics crash happen in every run instead of 2 of 5, and the frame rate kept switching between 60 and 30 as on the phone. That is enough to test a fix: same load, same route, a few runs.

To see the order of events, put temporary observation gates (`OBS` in the profile's `patches.py`, only at function entries or right after a `bl`, on position-independent instructions) on the functions involved. Their host callbacks append `(time, thread, event, pointers)` to an in-memory ring, and the crash handler prints the last ~150 entries. Printing each event as it happens is too slow and changes the timing. A function whose first instruction is PC-relative (an `adrp`, typical of short destructors) cannot take a gate: a temporary cut can put `svc #0x8f` at its entry and move its body into the `udf` padding after it (inside the same 4 KB page, an `adrp` keeps its encoding), with `svc_host_call` handling the number. Inside the host callback, the caller's return address is on the guest stack, in the frame the gate pushed (`regs + 800 + 8`), because the gate's own call has replaced `x30`. Remove all of it afterwards.

## Threading cut regressions

`dm/tools/check_thread_cuts.py` replays the registered FFT 1.4.0 and Link's Awakening 1.1.0 cuts against original NSOs without storing game bytes in the repository. Use a Python with `lz4` and `unicorn` (on Apple silicon Unicorn's JIT needs execution outside the agent sandbox):

```bash
python3 dm/tools/check_thread_cuts.py "<FFT ExeFS>/main" "<Link's Awakening ExeFS>/main"
```

It verifies build IDs and original instructions, relocates vtables, then checks that the original wrong-worker wake and late destructor fail while the registered cuts pass. `--original` instead asserts the originals are safe and exits nonzero on these failures. This is a controlled instruction replay: FFT's SDK event is a FIFO model, and the physics scene getter supplies a test manager. It does not reproduce the full kernel or gameplay. `make -C dm/host/tests GAME=010038B015560000 preempt` independently checks actual host condition priority/FIFO, mutex reacquisition, start and wake handoffs. Keep the game-specific observations in their notes.

## Memory growth

1. The host's memory report line (`memory up to frame …`) every few hundred frames; `footprint -p <pid>` from outside. On a phone, a crash with no log usually means `headroom to the system memory limit` reached 0.
2. `vmmap -summary <pid>` twice a few seconds apart: which region grows. "owned unmapped (graphics)" = private Metal resources.
3. `DM_NVN_ALLOCPROF=1`: growth per replayed op type and per function (`tex_mtl` / pass begin / pipelines / argument buffers), command allocator sizes, and a log of new textures with serial, size, format and storage. A stream of new serials with the same sizes = objects re-created every frame.
4. GPU allocated climbing while the residency set, `shared textures` and the upload counts stay flat: retired objects kept alive by an autorelease pool that never drains. Any objc call on the guest submit thread outside `g_sub_pool` leaks what it autoreleases (Metroid Dread; Breath of the Wild, GitHub issue #9). Build the host with the suspect wrapped in `@autoreleasepool` and compare.
5. `serial objects` climbing by the same amount every frame with GPU memory flat: entries the serial table never deletes. A census of `g_by_serial` by class and size in `mtl_mem_stat` shows which (BotW: finalized textures replay resolved again; they now go to the small table counted as `finalized textures still referenced`).

6. Address space rather than memory (an iPhone has about 5 GB of it without the extended virtual addressing entitlement): `DM_ALIAS_WINDOWS=<MB>` logs every 30 s how much of each window of that size in the alias region is mapped (windows aligned to their size in absolute addresses). For an allocator that reserves fixed windows and commits each from its start, the windows are its pools: Unreal's FMallocBinned3 in Octopath Traveler 0 showed 74 pools of 512 MB holding 732 MB together, the largest 68 MB, so no smaller uniform window fitted ([memory.md](../reference/memory.md#unreals-fmallocbinned3-replaced-by-the-host-allocator)). Unreal's own heap on the host malloc zone (`hle_uemalloc.c`) logs its peak as `[mem] Unreal heap … in use`.

Instruments' Metal Resource Events didn't record this process from the command line (`xctrace record --launch`, with `--env` for the variables); the in-host probes were faster.

## Save failures

`DM_TEST_FS_FAILAT=<point>[:<count>][,…]` makes steps of the save transaction fail with ENOSPC (`mountclone`, `reclone`, `guestname`, `renameback`; `dm/host/tests/t_fs.c` section 6). The game must keep running: a mount that cannot clone its work directory falls back to the committed saves read-only, and the iOS front end shows a notice.

## Render-pass investigation

See [04](04-graphics.md) section 7. Notes: pass dumps are large and may contain non-UTF-8 bytes; filter with `LC_ALL=C grep -a "\[pass\] F600 " | grep -av " o cmd"`. A Python one-liner with PIL (the dm venv has it) gives mean colour and non-black share of each dumped PNG, which is quicker than opening 20 images.

## On-device diagnostics (iOS)

The phone runs an unsigned IPA the user sideloads; nothing is attached to a debugger. The loop is: build an IPA with switches, the user plays and sends `Documents/log.txt` (the previous launch is `log.prev.txt`), read it, repeat.

- **Switches**: `DM_ENV_FILE=<file> dm/ios/tools/build_ipa.sh` bakes `KEY=VALUE` lines into the app (`dm/env.txt`); `Documents/env.txt` overrides them without a rebuild. Every applied switch is logged at boot (`boot: env KEY=VALUE (source file)`). The app's working directory is `Documents`, so relative output paths (`DM_NVN_SHOT=3600:shot_%d.png`, `DM_NVN_TEXDUMP`) land where the Files app can reach them.
- **Which build ran**: the first lines name the modules (`main: … patch_off …`), the shader map size (`shader map N entries`) and, for builds that have it, the build-ID gate line. Compare them with the build you sent before reading further; users sometimes send an old log. The save path (`fs mount save:`) shows the data folder, which changes when a sideloading tool gives a reinstall a new one.
- **Reading the periodic lines**: `[longframe]` gives each long frame's submit / GPU / pipeline-compile / upload split; a burst of `new pipelines` in long frames after an install is first-use compilation (see "Prewarm lists" in [docs/development.md](../development.md)). In `display link up to tick N: …`, `empty ticks` (refreshes with no new frame) and `old frames dropped by merging` are totals since launch, not per window. Per frame, `DM_NVN_TSV` has the display-link tick that took it: `@link_target_ms` (same time base as `@pres_at_ms`, so their difference is how late the system showed the frame: 8.3 ms = one 120 Hz refresh), `@link_lead_ms` (the callback's lead before that target: about 50 ms in a window, 16 ms in fullscreen at frame latency 1), `@link_ticks` (callbacks since the previous frame's; above 1 = ticks without a new frame) and `@link_gap_ms` (the longest interval between them; 33 ms on a 60 Hz link = a refresh the link skipped). `on-screen up to frame N` windows give dropped frames (a frame on screen longer than 1.5 × its own hold, so a frame-step hold of 2 vsyncs at 33.3 ms is not dropped; discarded presents count too), p99 frame time and the physical footprint; `memory up to frame N` gives resident and Metal-allocated memory. Memory that steps up at level loads and stays flat in between is not a leak.
- **Power**: `power up to frame N` covers the window since the previous one, for the whole process: CPU energy in watts and per frame (`task_energy` from `TASK_POWER_INFO_V2`, CPU only; compare per frame when one build holds the frame rate better), CPU time (100% = one core) and the share on performance cores, wakeups per second, the thermal state (0 nominal … 3 critical; changes are also logged as `front end: thermal state → n`), and the two guest threads that yielded most with how many of those yields the back-off turned into sleeps (`DM_YIELD_BACKOFF`, see [docs/development.md](../development.md)), and its spin wait passes with the last loop head. A thread yielding hundreds of thousands of times per window with few back-off sleeps is a busy spin. A busy thread that never yields and is not in the list may poll in a loop the spin wait rule does not take: sample it and look at its loop. `residency set up to frame N` gives residency-set commits and their time, what was added by source line of `nvn_mtl.m` (count, MB), removals, how many draw passes began with a full barrier or without one (`DM_NVN_BARRIER=game`), and segment commits: a source line adding gigabytes per window is allocation churn. It ends with render-target traffic per frame: attachment bytes loaded and stored, passes that load and store everything, and the stores `DM_NVN_UNWRITTEN_DC` saved (the same counters as the TSV's `@rt_load_mb` / `@rt_store_mb`). `threads up to frame N` gives guest-thread CPU time in ms per second by Switch priority band (≤ 44, the main thread's level and above; 45–59; ≥ 60) and the three busiest threads with their priority; it shows what `DM_QOS_MAP` would move. On the Mac the same lines appear; energy readings vary by tens of percent between identical runs, so compare runs interleaved and over the same scene.
- **Screen recordings** from the user are useful for timing and visual symptoms; extract frames with `ffmpeg -ss <t> -i rec.mov -frames:v 1 out.png`.

## GPU time on the device

Which passes cost GPU time, on the iPhone or the Mac, without Xcode attached. The switches are off by default; on the phone they go in `Documents/env.txt` and the lines come back in `Documents/log.txt`.

- **A profile every n frames**: `DM_NVN_GPUPROF=every:600` profiles frames 600, 1200, … each on its own (`DM_NVN_GPUPROF=f` or `f1-f2` suits the Mac, where a scripted route reaches the same scene at the same frame). A profiled frame prints one `[gpuprof] F<frame>` line per encoder: its GPU time and start (µs after the frame's first encoder); for a render pass the size, colour attachments, depth, loads, draws, the command-buffer segment, what it loaded and stored of its attachments (`load` / `store` KB, the bandwidth ledger's view: a DontCare store counts nothing) and `fmt <colour 0>/<depth>` as `MTLPixelFormat` numbers; for a compute / copy encoder its copies, uniform updates and dispatches. Then `[gpuprof] frame N: render … ms, compute / copy … ms over … encoders (… render passes, attachments load … MB store … MB), first start to last end … ms`. The timestamps are relaxed, so encoders overlap: their sum is not the frame's GPU time, the span is closer. It costs two timestamps per encoder; the frame runs as usual. `DM_NVN_PASSDUMP=every:600` adds a pass dump halfway between two profiled frames, for the shader keys of an expensive pass shape.
- **Reading them**: `python3 dm/tools/gpuprof.py log.txt` prints per-frame totals (sum, span, idle time between encoders, draws, attachment traffic), then the mean per frame of each pass shape (size, colours, depth, formats), the most expensive first. `--by class` groups by size relative to the frame's largest colour pass instead (full, half, quarter … per axis; passes with four colours or more, a deferred renderer's G-buffer, apart), which compares games and resolutions; `--timeline <frame>` lists one frame's encoders in GPU order with the gaps between them; `--frames` keeps some frames. The GPU clock changes from frame to frame (on the iPhone with its thermal state), so compare means over many profiled frames.
- **A/B in one run**: two runs on a phone differ by scene, temperature and clock more than most changes do. `DM_AB=<frames>` alternates every switch set to `ab` in phases of that many frames (on in the odd phases); `ab:<k,…>/<n>` turns it on in the phases whose number modulo n is one of the k, so `ab:1,3/4` on one switch and `ab:2,3/4` on another run four configurations. Each phase prints `[ab] phase N <switches on, or base>` with GPU and whole frame time (p50 / p90 / mean without the phase's first quarter) and attachment traffic and passes per frame; `gpuprof.py log.txt --ab` gives the medians per configuration and their difference from the base. `DM_NVN_BARRIER=ab` is the first such switch; another becomes one by reading its mode with `sw_mode` and asking `sw_on` where it applies (`nvn_mtl.m`).
- **A frame for Xcode's Metal debugger** (per-draw cost, the shader profiler): `DM_GPU_CAPTURE=<frame>[:<frames>]`, or `gpu:<ms>[:<after frame>]` for the first frame, after that frame, that follows 60 frames in a row with more than `<ms>` of GPU time, writes `dm_<date>-<time>_f<frame>.gputrace` to `gpu_captures/` in the working directory (`Documents` on iOS, the run folder on the Mac; `DM_GPU_CAPTURE_DIR` names another folder). The iPhone needs an IPA built with `DM_METAL_CAPTURE=1` (`MetalCaptureEnabled` in its `Info.plist`), the Mac `MTL_CAPTURE_ENABLED=1`; on the Mac the `DM_CTL` line `env DM_GPU_CAPTURE=<frame>` sets it while the game runs. A trace takes 1.5–2 GB, so before each capture that folder alone is pruned to the newest `DM_GPU_CAPTURE_KEEP` traces (2, the new one included). Take it off the phone with Finder or the Files app and open it in Xcode. Such a run builds its pipelines through the `MTLDevice` ([GPU frame captures need pipelines built through the device](../reference/shaders.md#gpu-frame-captures-need-pipelines-built-through-the-device)), so time with the profile and look inside with a capture.

## Offline pipeline link check

Pipeline link errors depend only on the two libraries, so all vertex / pixel pairs can be checked on the Mac after a shader build, without the game: for each line of `shader_pairs.tsv`, load `<vs>_b.metallib` and `<ps>_b.metallib` from `b-<V>/`, build a vertex descriptor from the vertex function's `vertexAttributes` (format from each attribute's type), set colour attachment formats from the `[[color(n)]]` outputs in the pixel shader's `_b.metal` (uint / int / float formats to match), add a depth format when it writes `[[depth(...)]]`, and call `newRenderPipelineStateWithDescriptor:`. Count failures by message. Mario: 8,712 linked / 331 failed before the vertex output padding, 9,043 / 0 after, and 14,603 / 0 for the v0 + 1.2.1 tables. Pipelines built this way have the default blend state and sample count, so this checks linking only.

## Intermittent rendering glitches

A glitch that comes and goes between frames or runs needs a detector before bisection, or "no glitch in these 16 frames" proves nothing. What worked for Mario's fountain panels:

1. Screenshot a dense frame range (`DM_NVN_SHOT=$(seq -s, 4430 10 4590):x_%d.png`) and score each frame automatically. The first metric (texture busyness) missed the flat translucent variant of the glitch; a metric tuned to its shape (long vertical straight edges in a sky region) separated glitch frames (≈1.0) from clean ones (≈0.3) reliably. Look at a few frames by eye before trusting a metric.
2. Run a control with nothing changed to measure how often the glitch appears per run.
3. Bisect with `DM_NVN_SKIP_FS` (up to 16 fragment shader keys per run) until one draw remains, then `DM_NVN_HILITE` it to see its geometry.
4. Dump that draw's inputs across frames (`DM_NVN_CBDUMP` + `DM_NVN_CBDUMP_KEY`). If every input is byte-identical between good and bad frames, the difference is on the GPU side: follow the textures it samples (`DM_NVN_TEXINFO`, `DM_NVN_TEXRAW`) to whatever writes them.
5. For a GPU-side simulation, trace its state on the GPU timeline (`DM_NVN_T2T_TRACE`), re-implement the shader step in numpy with the logged uniform updates (the `Y update_ub` lines print the first four floats), and compare frame by frame. Frames that disagree show which input the GPU really used; for the fountain, a coefficient of −1 instead of 1.81 explained every bad texel and led to the missing intra-encoder barrier.
6. A dump read at encode, or after the submission completed, may not be what the draw read: later work of the submission can rewrite the buffer. When a draw's indices or vertices look wrong, list in command order every copy and inline update of the frame from the pass dump (`DM_NVN_PASSDUMP_ALL=1`, the `Y copy_b2b` / `Y update_ub` lines): a copy whose source an earlier GPU write of the same frame covers reads that write, not the bytes the CPU staged. Cuphead's broken runs had dozens of those a frame, and the clean runs none. The game's allocations overlapped, so the cause was in the game's allocator, which a host answer had broken ([Sizes the hardware never reports as 0](../reference/nvn-objects.md#sizes-the-hardware-never-reports-as-0-zcull-save-area)).


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

