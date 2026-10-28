# Game profiles

Each game the port knows has a folder `dm/games/<id>/`, named after the base game's title ID: Mario's is `010015100B514000`, Engage's `0100A6301214E000`. The folder holds:

- **`game.tsv`**: the game's constants, one `key<TAB>value` row each, with comments on their own lines.
- **The shader tables** that `shader_split.py` writes: `shader_map.tsv` (+ `.conflicts.tsv`), `shader_facts.tsv`, `shader_pairs.tsv`, and `pso_pairs.tsv` (supplemental agl BAHS program variations and legacy Unity pass-input covers). Both pair tables feed vertex output padding and the pipeline prebuild; for a profile without `pso_pairs.tsv`, the NSP script keeps the generated table beside the shader package.
- **`shader_facts_manual.tsv`** (optional): hand-written translation facts in `shader_facts.tsv`'s row format, for what the machine code cannot tell (Link's Awakening: a vertex input the integer-input guess gets wrong; Breath of the Wild and Tears of the Kingdom: integer fragment outputs, which a game without a PC version has no other source for). `shader_split.py` merges it over the generated rows; since the table is on the chain, changing it changes only that profile's chain version.
- **`pso_manifest.tsv`**: the prewarm seed list.
- **`bench.tsv`** (optional): benchmark routes for `scripts/bench.py` ([A benchmark on the Mac](../../docs/development.md#a-benchmark-on-the-mac)): a name, the scripted input (`DM_KEYS`), the measured window and a description, one route per line.
- **`vk_capture.tsv`** (a Vulkan game that generates its shaders at run time, Super Mario Galaxy): the keys of the SPIR-V modules it generates.
- **`gl_capture.tsv`** (an OpenGL game that puts its GLSL together at run time, Balatro): the files (`<key>.vert` / `<key>.frag`) of the shaders it compiles.
- **`bf_want.tsv`**.
- **`validated.tsv`** (optional): the chain this game was last green on, written by `scripts/regress.py check --record` (or `baseline --record`) when its comparison status is `PASS` or `CHANGED` and every tested boot run succeeds. A boot failure shared with the baseline still prevents recording. `quick` / `full` rows (the latest of each), `mode<TAB>chain=<DMSH_CHAIN> commit=<short SHA>[+dirty] date=<UTC day> machine=<model> macos=<version>`; a red result leaves the rows as they were. A build using a derived profile does not validate the repository profile. `regress.py list` shows it and whether that chain is the current one. It holds no game data; commit it with the pull request.
- **`geo_mesh.tsv`** (optional): vertex / geometry shader pairs drawn with a passthrough geometry shader; `dmsh_build.py` builds a mesh library for each (`dmsh_mesh.py`), and the host draws such a pair's triangle lists as mesh draws ([Passthrough geometry shaders drawn by the vertex shader](../../docs/reference/shaders.md#passthrough-geometry-shaders-drawn-by-the-vertex-shader)). A pair not listed takes the compute path.
- **`README.md`**: the game's notes: what its profile needs and why, what was found on it, how to reach a scene on the Mac, what is still open. Listed in [Games](#games).
- **`patches.py`** (optional): the rewriter's patches of the game's own builds (`main`), keyed by NSO build ID: `CUTS` (byte cuts, each original asserted), `OBS` (observation gates) and `FRAME_STEP` (frame-step stubs). `dm/tools/patch_nso.py` loads every profile's file and applies those of a module's build, wherever it comes from; a build claimed twice stops the rewrite. A provisional profile (in `DM_GAMES_DIR`) may have one too, which the rewriter and the NSP script's rewrite step take as well. Cuts of Nintendo's `sdk` modules, which several games ship, are in `dm/tools/sdk_patches.py`. Opt-in `MATCH_CUTS` can locate verified cuts in unregistered builds of the selected game by function fingerprints (a patch of several functions applies whole or not at all); see [Function-matched cuts](../../docs/reference/rewriter.md#function-matched-cuts-for-unregistered-builds) for enrollment, profile precedence and limits. A game without patches has no file, and a derived profile uses its repository profile's rules. `dm/tools/tests/test_patches.py` checks that a file names only its own profile's builds. `GENERALIZE` (a note, not read by the rewriter) says whether a general rule could replace the file's patches; after changing a patch file, a build-keyed host table or a profile key, run `dm/tools/docs_patches.py` to update the [patch index](../../docs/agents/game-patches.md).

The chain version is not in the profile: one version serves every game (`dm/gfx/nvn/dmsh_chain.h`, written by `dmsh_chain.py --header`). Two more tables live in the profile folder but are not committed (git ignores them). A full `dmsh_build.py` run makes both from its Mac outputs, so each checkout makes its own, and stamps its libraries with the digest of the profile's inputs (`b-<V>/inputs.tsv`, `dmsh_chain.py --inputs`). The host takes a table only when its header names its library's chain version and that digest.
- **`c1_full.tsv`**: written by `c1_guard.py`. Without it every shader's c1 gets its own 64 KiB.
- **`cb_extent.tsv`**: how far into each uniform slot every shader can read, written by `cb_extent.py`. Without it a uniform block the game updates in part inside a render pass is copied whole.

Everything game-specific belongs here. Code reads the profile; it does not hard-code a game.

| Who | Picks the profile with | Reads |
|---|---|---|
| Shader tools (`shader_split.py`, `dmsh_chain.py`, `dmsh_build.py`, `dmsh_facts.py`) | `DM_GAME=<id>` | tables, `shader_*` keys |
| Host (`make -C dm/host`, `dm/gfx/tests`) | `GAME=<id>` | `gen/<id>/game_profile.h`, generated from `game.tsv` by `game.py header` |
| `dm/ios/tools/build_ipa.sh` | `GAME=<id>` | app name, bundle ID, IPA name, modules, RomFS marker, tables |
| `scripts/build_ipa_from_nsp.py` | the NSPs' title ID | everything above. With no matching profile it writes a survey, then a provisional profile outside the repository and a best-effort IPA (`--survey-only` stops after the survey) |

None of them has a default game: without one they stop and list the profiles. To query a profile:
- `python3 dm/games/game.py list` lists the profiles.
- `python3 dm/games/game.py <id> <key>` prints one key.

**Launcher settings.** `settings.tsv` lists the player-facing settings once (the environment variable each sets, its kind, default, the condition under which a game offers it and its section on the single settings screen, **Game**, **Controls** or **Performance**; Reset to tested defaults sets the screen back to the profile's defaults, and Play then forgets the launcher choices; its header describes the columns). A profile key a setting defaults to (`DM_GAME_<NAME>`) is the value the game was tested with, which the player page [docs/settings.md](../../docs/settings.md#tested-defaults) explains. `build_ipa.sh` runs `settings.py <id> <rewritten modules> <control.nacp> <out.json>`, which keeps the rows that hold for the game: a profile macro (`DM_GAME_FPS_TARGET!=0`), a key in `main`'s rewriter table (`RENDER_SCALE`, written by `patch_nso.py` for the builds with a render-scale stub), the game's languages. It writes them into the app as `dm/launcher.json`, from which the iOS launcher (`dm/ios/Sources/Launcher.swift`) draws its screen; the NSP script packs the Mac app's copy (`SETTINGS_PLATFORM=mac`, without the rows marked `platform:ios`, `Contents/Resources/dm/launcher.json`) for the Mac app's settings window (`dm/mac/Launcher.swift`) and Mac front ends. A row's optional last column, `meta`, carries `key=value` fields front ends read (`level=basic`, `needs=<VAR>=<value>`; [Front ends](../../docs/frontends.md#the-settings-list), with the JSON Schema in `docs/launcher.schema.json`), and a profile's `render_base` goes into the render scale's row as `base`, so a front end can show that setting in pixels. A new player setting is one row; diagnostic switches stay environment-only.

Defaults may use `<ios>|<mac>` to select the platform's value before resolving a profile macro or build variable. For example, `DM_GAME_PSO_MTL4|1` follows the profile on iOS and the Mac host's always-on default on Mac; `1|0` matches the encode thread's iOS-on, Mac-off defaults. `SETTINGS_PLATFORM` accepts `ios` (the default) or `mac`; other values are rejected.

## Games

One row per profile; the notes are each profile's `README.md`.

| Game | Title ID | Build | SDK | Engine | Graphics |
|---|---|---|---|---|---|
| [13 Sentinels: Aegis Rim](01003FC01670C000/README.md) | `01003FC01670C000` | 1.10 | 11.4.3 | Vanillaware's engine | NVN |
| [13 Sentinels: Aegis Rim, Asian release](01008D7016438000/README.md) | `01008D7016438000` | 1.00 (base) | 11.4.3 | Vanillaware's engine | NVN |
| [13 Sentinels: Aegis Rim, Japanese release](010045C014650000/README.md) | `010045C014650000` | 1.00 (base) | 10.8.2 | Vanillaware's engine | NVN |
| [Absolum](0100A83022A22000/README.md) | `0100A83022A22000` | 1.1.2 | 18.3.2 | FNA / .NET NativeAOT (FNA3D's NVN driver) | NVN |
| [Ace Attorney Investigations Collection](010005501E68C000/README.md) | `010005501E68C000` | 1.0.0.2 | 17.5.3 | Unity 2022.3.17f1 IL2CPP | NVN |
| [Advance Wars 1+2: Re-Boot Camp](0100300012F2A000/README.md) | `0100300012F2A000` | 1.0.0 (base) | 10.4.1 | Unity 2020.2.1f3 IL2CPP | NVN |
| [AI: THE SOMNIUM FILES](010089B00D09C000/README.md) | `010089B00D09C000` | 1.0.1 | 6.4.2 | Unity 2017.4.17f1 IL2CPP | NVN |
| [ASTRAL CHAIN](01007300020FA000/README.md) | `01007300020FA000` | 1.0.1 | 6.4.0 | PlatinumGames' engine | NVN |
| [AI: THE SOMNIUM FILES - nirvanA Initiative](0100713016CCC000/README.md) | `0100713016CCC000` | 1.0.2 | 13.3.5 | Unity 2020.3.30f1 IL2CPP | NVN |
| [ANIMAL WELL](010020D01AD24000/README.md) | `010020D01AD24000` | 1.0.10 | 20.5.7 | its own (not named) | NVN |
| [Animal Crossing: New Horizons](01006F8002326000/README.md) | `01006F8002326000` | 3.0.3 | 20.5.4 | Nintendo EPD (sead / agl / gsys) | NVN |
| [BALL x PIT](010086A022444000/README.md) | `010086A022444000` | 1.293 | 20.5.19 | Unity 6000.0.62f1 IL2CPP | NVN |
| [Balatro](0100CD801CE5E000/README.md) | `0100CD801CE5E000` | 1.1.3 | 18.3.2 | LÖVE 11.4 (love-nx, LuaJIT, SDL) | OpenGL |
| [Bayonetta](010076F0049A2000/README.md) | `010076F0049A2000` | 1.1 | 3.6.0 | PlatinumGames' engine (Wii U form) | NVN |
| [Bayonetta 2](01007960049A0000/README.md) | `01007960049A0000` | 1.2 | 3.6.0 | PlatinumGames' engine (Wii U form) | NVN |
| [Bayonetta 3](01004A4010FEA000/README.md) | `01004A4010FEA000` | 1.2.0 | 13.3.2 | PlatinumGames' engine | NVN |
| [Big Brain Academy: Brain vs. Brain](0100620012D6E000/README.md) | `0100620012D6E000` | 1.2.0 | 11.4.3 | NintendoWare Bezel Engine 1.56 | NVN |
| [Blasphemous](0100698009C6E000/README.md) | `0100698009C6E000` | 1.0.8 | 8.3.1 | Unity 2017.4.28f1 IL2CPP, FMOD | NVN |
| [Blasphemous 2](010040801A4C8000/README.md) | `010040801A4C8000` | 1.0.8 | 15.3.2 | Unity 2021.3.20f1 IL2CPP, FMOD | NVN |
| [Brain Memory Training Academy](0100F5D01DA3A000/README.md) | `0100F5D01DA3A000` | 1.0.0 (base) | 16.2.4 | Unity 2022.3.5f1 IL2CPP | NVN |
| [Captain Toad: Treasure Tracker](01009BF0072D4000/README.md) | `01009BF0072D4000` | 1.4.2 | 8.2.0 | Nintendo EPD Tokyo (`al` over sead / agl) | NVN |
| [Catherine: Full Body](0100BF00112C0000/README.md) | `0100BF00112C0000` | 1.0.1a | 8.2.1 | Atlus GFD (CRI middleware) | NVN |
| [Celeste](01002B30028F6000/README.md) | `01002B30028F6000` | 1.4.1.1 | 15.3.2 | MonoGame (C# → C++ by BRUTE) | NVN |
| [Cuphead](0100A5C00D162000/README.md) | `0100A5C00D162000` | 1.3.7 | 5.5.0 | Unity 2017.4.9f1 IL2CPP | NVN |
| [DARK SOULS: REMASTERED](01004AB00A260000/README.md) | `01004AB00A260000` | 1.0.3 | 4.5.0 | FromSoftware's engine | NVN |
| [Digimon Story Cyber Sleuth: Complete Edition](010014E00DB56000/README.md) | `010014E00DB56000` | 1.0.3 | 6.4.0 | Media.Vision MVGL | OpenGL |
| [Donkey Kong Country Returns HD](01009D901BC56000/README.md) | `01009D901BC56000` | 1.1.0 | 18.3.5 | Unity 6000.0.28f1 IL2CPP | NVN |
| [Donkey Kong Country: Tropical Freeze](0100C1F0051B6000/README.md) | `0100C1F0051B6000` | 1.0.2 | 4.4.0 | Retro Studios' engine | NVN |
| [DRAGON QUEST VII Reimagined](0100A9D01C446000/README.md) | `0100A9D01C446000` | 1.2.0 | 19.3.1 | Unreal Engine 4.27 | NVN |
| [DRAGON QUEST XI S: Echoes of an Elusive Age – Definitive Edition](01006C300E9F0000/README.md) | `01006C300E9F0000` | 1.0.4 | 7.3.2 | Unreal Engine 4.18 | NVN |
| [FINAL FANTASY CRYSTAL CHRONICLES Remastered Edition](0100CE4010AAC000/README.md) | `0100CE4010AAC000` | 1.0.3 | 8.2.1 | Unity 2018.4.2f1 IL2CPP, FMOD | NVN |
| [FINAL FANTASY RESONANCE](0100606026F12000/README.md) | `0100606026F12000` | 1.0.1 | 20.5.17 | Unreal Engine 5.6.1 | NVN |
| [FINAL FANTASY RESONANCE, Asian release](0100B8A026F14000/README.md) | `0100B8A026F14000` | 1.0.1 | 20.5.17 | Unreal Engine 5.6.1 | NVN |
| [FINAL FANTASY TACTICS - The Ivalice Chronicles](010038B015560000/README.md) | `010038B015560000` | 1.4.0 | 19.3.9 | Square Enix's engine | NVN |
| [FINAL FANTASY X/X-2 HD Remaster](0100BC300CB48000/README.md) | `0100BC300CB48000` | 1.0.0 (base) | 5.4.0 | PhyreEngine (Virtuos) | NVN |
| [Fire Emblem Engage](0100A6301214E000/README.md) | `0100A6301214E000` | 2.0.0 | 12.3.7 | Unity 2020.3.18f1 IL2CPP, Wwise | NVN |
| [Fire Emblem: Three Houses](010055D009F78000/README.md) | `010055D009F78000` | 1.2.0 | 6.5.0 | Koei Tecmo's KTGL | NVN |
| [Ghost Trick: Phantom Detective](010029B018432000/README.md) | `010029B018432000` | 1.0.0 (base) | 13.3.0 | Capcom's RE Engine, Wwise | NVN |
| [Hades](0100535012974000/README.md) | `0100535012974000` | 1.0.38233 | 9.3.0 | Supergiant Games' engine (The Forge) | Vulkan |
| [Hades II](0100A00019DE0000/README.md) | `0100A00019DE0000` | base game | 19.3.9 | Supergiant Games' engine | NVN |
| [Hollow Knight: Silksong](010013C00E930000/README.md) | `010013C00E930000` | 1.0.30000 | 20.5.8 | Unity 6000.0.50f1 IL2CPP | NVN |
| [Hyrule Warriors: Age of Calamity](01002B00111A2000/README.md) | `01002B00111A2000` | 1.3.0 | 9.6.0 | Koei Tecmo's KTGL | NVN |
| [Kirby and the Forgotten Land](01004D300C5AE000/README.md) | `01004D300C5AE000` | 1.1.0 | 11.4.0 | HAL Laboratory's engine (NintendoWare) | NVN |
| [Luigi's Mansion 3](0100DCA0064A6000/README.md) | `0100DCA0064A6000` | 1.4.0 | 7.3.2 | Next Level Games' engine | NVN |
| [Mario Kart 8 Deluxe](0100152000022000/README.md) | `0100152000022000` | 4.0.0, 4.0.1 | 4.4.0 | Nintendo (sead / agl / gsys, NintendoWare, Pia) | NVN |
| [Mario + Rabbids Kingdom Battle](010067300059A000/README.md) | `010067300059A000` | 1.9.589692 | 4.4.0 | Ubisoft Snowdrop | NVN |
| [Mario + Rabbids Sparks of Hope](0100317013770000/README.md) | `0100317013770000` | 1.6.2225577 | 13.3.5 | Ubisoft Snowdrop | NVN |
| [Mario vs. Donkey Kong](0100B99019412000/README.md) | `0100B99019412000` | 1.0.1 (cartridge base) | 15.3.0 | Nintendo (sead / agl / gsys) | NVN |
| [Metroid Dread](010093801237C000/README.md) | `010093801237C000` | 2.1.0 | 11.4.3 | MercurySteam's engine | NVN |
| [Monster Hunter Rise](0100B04011742000/README.md) | `0100B04011742000` | 16.0.2 | 11.4.3 | Capcom's RE Engine | NVN |
| [Monster Train 2](010051701FB46000/README.md) | `010051701FB46000` | 1.1.1 | 18.3.5 | Unity 2022.3.54f1 IL2CPP | NVN |
| [NieR:Automata The End of YoRHa Edition](010056B015FE8000/README.md) | `010056B015FE8000` | 1.0.3 | 12.3.9 | PlatinumGames' engine | NVN |
| [NieR:Automata The End of YoRHa Edition, Asian release](0100B8E016F76000/README.md) | `0100B8E016F76000` | 1.0.3 | 12.3.9 | PlatinumGames' engine | NVN |
| [OCTOPATH TRAVELER](010057D006492000/README.md) | `010057D006492000` | 1.0.5 | 3.5.1 | Unreal Engine 4.18 | NVN |
| [Octopath Traveler 0](01005270232F2000/README.md) | `01005270232F2000` | 1.0.5 | 19.3.13 | Unreal Engine 5 | NVN |
| [OCTOPATH TRAVELER II](0100A3501946E000/README.md) | `0100A3501946E000` | 1.1.1 | 12.3.2 | Unreal Engine 4.27 | NVN |
| [Paper Mario: The Thousand-Year Door](0100ECD018EBE000/README.md) | `0100ECD018EBE000` | 1.0.1 | 15.3.2 | Intelligent Systems' engine on NintendoWare | NVN |
| [Patrick's Parabox](0100E44016C22000/README.md) | `0100E44016C22000` | 1.0.1 | 13.3.7 | Unity 2020.3.30f1 IL2CPP | NVN |
| [Persona 4 Golden](010062B01525C000/README.md) | `010062B01525C000` | 1.0.2 | 14.3.0 | Atlus in-house (CRI middleware) | NVN |
| [Persona 5 Royal](01005CA01580E000/README.md) | `01005CA01580E000` | 1.0.2 | 13.3.2 | Atlus GFD / Needle (CRI middleware) | NVN |
| [Pikmin 4](0100B7C00933A000/README.md) | `0100B7C00933A000` | 1.1.1 | 10.4.8 | Unreal Engine 4.26.2 | NVN |
| [Pokémon Brilliant Diamond](0100000011D90000/README.md) | `0100000011D90000` | 1.3.0 | 11.4.3 | Unity 2019.4.27f1 IL2CPP | NVN |
| [Pokémon Legends: Z-A](0100F43008C44000/README.md) | `0100F43008C44000` | 2.0.2 | 19.3.11 | Game Freak's Trinity engine | NVN |
| [Pokémon Scarlet](0100A3D008C5C000/README.md) | `0100A3D008C5C000` | 1.0.0 | 13.3.5 | Game Freak's Trinity engine | NVN |
| [Pokémon Shield](01008DB008C2C000/README.md) | `01008DB008C2C000` | 1.3.2 | 7.3.2 | Game Freak's engine (before Trinity) | NVN |
| [Pokémon Sword](0100ABF008968000/README.md) | `0100ABF008968000` | 1.3.2 | 7.3.2 | Game Freak's engine (before Trinity) | NVN |
| [Portal](01007BB017812000/README.md) | `01007BB017812000` | 1.0.3 | 13.3.0 | Valve's Source engine (28 NROs) | NVN |
| [Raging Loop](010020400E1C2000/README.md) | `010020400E1C2000` | 1.0.0 (base) | 7.3.2 | Unity 2018.3.10f1 IL2CPP | NVN |
| [Return of the Obra Dinn](010032E00E6E2000/README.md) | `010032E00E6E2000` | 1.0.1 | 8.2.0 | Unity 2017.4.28f1 IL2CPP | NVN |
| [Rhythm Heaven Groove](0100D9F01D474000/README.md) | `0100D9F01D474000` | 1.0.0 (base) | 21.4.0 | Nintendo's own 2D engine | NVN |
| [ROMANCE OF THE THREE KINGDOMS 8 REMAKE](0100C2A01A730000/README.md) | `0100C2A01A730000` | 1.1.3 | 17.5.3 | Koei Tecmo's KTGL | NVN |
| [Sea of Stars](01008C0016544000/README.md) | `01008C0016544000` | 2.0.59001 | 16.2.0 | Unity 2021.3.25f1 IL2CPP | NVN |
| [SHIN MEGAMI TENSEI V: Vengeance](010069C01AB82000/README.md) | `010069C01AB82000` | 1.0.3 | 15.3.2 | Unreal Engine 4.27 | NVN |
| [Silver Pines](0100D5B026844000/README.md) | `0100D5B026844000` | 1.0.0 (base) | 21.4.7 | Unity 6000.3.15f1 IL2CPP | NVN |
| [STEINS;GATE RE:BOOT](01008BC024FDE000/README.md) | `01008BC024FDE000` | 1.0.3 | 20.5.17 | M2's ADV engine | NVN |
| [Stardew Valley](0100E65002BB8000/README.md) | `0100E65002BB8000` | 1.6.15.13 | 20.5.27 | MonoGame (C# → C++ by BRUTE) | NVN |
| [Super Mario 3D World + Bowser's Fury](010028600EBDA000/README.md) | `010028600EBDA000` | 1.2.2 | 10.4.0 | Nintendo EPD (sead / agl, NintendoWare) | NVN |
| [Super Mario Bros. Wonder](010015100B514000/README.md) | `010015100B514000` | 1.2.1 (and 1.0.0) | 16.2.12 (1.0.0: 16.2.0) | Nintendo EPD (sead / agl) | NVN |
| [Super Mario Galaxy](010099C022B96000/README.md) | `010099C022B96000` | 1.3.1 | 18.3.5 | NERD's Hagi Wii layer (recompiled PowerPC) | Vulkan |
| [Super Mario Galaxy 2](0100FD8022DAA000/README.md) | `0100FD8022DAA000` | 1.4.0 | 18.3.5 | NERD's Hagi Wii layer (recompiled PowerPC) | Vulkan |
| [Super Mario Odyssey](0100000000010000/README.md) | `0100000000010000` | 1.4.1 | 7.3.2 | Nintendo EPD (sead / agl, NintendoWare) | NVN |
| [Super Smash Bros. Ultimate](01006A800016E000/README.md) | `01006A800016E000` | 13.0.5 | 8.2.1 | Bandai Namco / Sora's engine | NVN |
| [Taiko Risshiden V DX](0100346017304000/README.md) | `0100346017304000` | 1.1.1 | 12.3.3 | Koei Tecmo's KTGL | NVN |
| [The Great Ace Attorney Chronicles](010036E00FB20000/README.md) | `010036E00FB20000` | 1.0.0 (base) | 10.4.1 | Capcom's MT Framework | NVN |
| [The Legend of Zelda: Breath of the Wild](01007EF00011E000/README.md) | `01007EF00011E000` | 1.9.0 | 20.5.22 | Nintendo EPD (sead / agl / gsys / ksys, Havok) | NVN |
| [The Legend of Zelda: Echoes of Wisdom](01008CF01BAAC000/README.md) | `01008CF01BAAC000` | 1.1.0 | 20.5.1 | NintendoWare (G3d / Ui2d / Vfx) | NVN |
| [The Legend of Zelda: Link's Awakening](01006BB00C6F0000/README.md) | `01006BB00C6F0000` | 1.1.0 | 8.2.0 | Grezzo's engine on NintendoWare | NVN |
| [The Legend of Zelda: Tears of the Kingdom](0100F2C0115B6000/README.md) | `0100F2C0115B6000` | 1.4.3 | 20.5.22 | Nintendo EPD (sead / agl / gsys) | NVN |
| [Unicorn Overlord](010069401ADB8000/README.md) | `010069401ADB8000` | 1.05 | 16.2.4 | Vanillaware's engine | NVN |
| [Unicorn Overlord, Asian release](010054B01AD92000/README.md) | `010054B01AD92000` | 1.05 | 16.2.4 | Vanillaware's engine | NVN |
| [Xenoblade Chronicles: Definitive Edition](0100FF500E34A000/README.md) | `0100FF500E34A000` | 1.1.2 | 9.3.1 | Monolith's engine | NVN |
| [Xenoblade Chronicles 2](0100E95004038000/README.md) | `0100E95004038000` | 2.1.0 | 3.5.0 | Monolith's engine | NVN |
| [Xenoblade Chronicles 3](010074F013262000/README.md) | `010074F013262000` | 2.2.1 | 12.3.9 | Monolith's engine | NVN |
| [Yoshi's Crafted World](01006000040C2000/README.md) | `01006000040C2000` | 1.0.1 | 4.4.1 | Unreal Engine 4.18 era (Good-Feel) | NVN |

## Keys

| Key | Meaning |
|---|---|
| `title_id`, `program_id` | Identity. `program_id` is what `svcGetInfo(ProgramId)` reports. The game's name is not stored: the NSP script reads it from the NSP's control data (NACP) |
| `name` | The game's name, only a label for `game.py list`, error messages and the Mac window title. The iOS app's name is the game name in the base NSP's control data and its bundle ID is `org.nxnative.<title ID in lowercase>`; the NSP script's `APP_NAME` / `BUNDLE_ID` and `build_ipa.sh`'s `DM_DISPLAY_NAME` / `DM_BUNDLE_ID` override them |
| `ipa`, `app_category` | iOS app: the default IPA file name and the App Store category |
| `build <ver> <display version> <module>=<build ID>…` | One row per version the game was brought up on. The NSP script builds any version (the base game alone, any update): a registered one from the profile as it is, any other from a profile derived from it (`<id>_build/profile`: the same keys, that version's own shader tables; `derive_profile`), with a red line naming the modules whose build no row registers, which the per-build cuts and host fixes do not reach. The host reports the display version by `main`'s build ID |
| `package` | The version the profile's tables, prewarm list and notes were made on. The NSP script packs the newest version in the folder (`-p` / `PACKAGE` picks another) |
| `shader_versions` | Versions whose union the shader tables are (base game first). A registered version is built from the profile as it is only when all of them are in the folder; otherwise from a derived profile, with tables from the packed version alone |
| `modules` | Code modules rewritten and loaded, in load order. `rtld` is replaced by the host |
| `romfs_marker` | A path that must exist in the RomFS; empty checks the directory itself for a program without a data section (ANIMAL WELL) |
| `zip_store` | Extensions the IPA stores without compressing |
| `mem_reserve`, `mem_alias`, `mem_stack`, `app_pool` | Guest address-space partition and application pool (`hle_svc.c`) |
| `mem_used` | Memory `GetInfo(UsedMemorySize)` reports beside the heap (default `0x4000000`, 64 MB). `SetHeapSize` checks a request against the real size of the loaded modules, so a game that sizes its heap as total − used and has more than 64 MB of modules needs at least their size here. Hyrule Warriors: Age of Calamity: 66.6 MB of modules, so the default made `SetHeapSize(pool − 64 MB)` fail and `nnMain` return at once; it sets 72 MB. Cuphead: 80 MB of modules, 84 MB |
| `sys_resource` | The system resource size the NPDM declares (main.npdm META +0x14), which `GetInfo(TotalSystemResourceSize)` reports (`hle_svc.c`). The SDK's `IsVirtualAddressMemoryEnabled` is "that size is not 0", and some games choose their memory setup by it: Big Brain Academy: Brain vs. Brain's `nninitStartup` asks StandardAllocator for a 60 GB address region with virtual address memory and sets a heap without it, and its NPDM declares none (`0`). The survey writes it from the NPDM for every new game; a profile without it gets the host's earlier answer, 32 MB, for every game |
| `game_args` | Arguments the game gets when the command line gives none (the iOS app, the NSP script's Mac runs), split at spaces; `dmhost <dir> -- …` overrides them. The SDK hands them to the game before `nninitStartup`, as on hardware (`main.c`). Octopath Traveler 0 caps the address region Unreal's `nninitStartup` reserves (62 GB by default) with `-SwitchVirtualAddressSpaceSize=2048` (MB) |
| `game_env` | `KEY=VALUE` pairs, separated by spaces, that the game's `getenv` returns (`hle_os.c`; any other name goes to the SDK's own `getenv`, whose environment is empty as on Horizon). For runtimes that take their settings from the environment. Absolum (.NET NativeAOT) sets `DOTNET_GCRegionRange=60000000`: the GC reserves 5 × its heap hard limit as its region range otherwise, 3.75 GB, which only the Switch's 64 GB alias region holds. `DM_GAME_ENV` overrides it, also from `Documents/env.txt` |
| `mem_map_limit` | `on` / `off` (the default): `MapPhysicalMemory` fails with ResultLimitReached when it would take the process past the application pool (the pool less the used memory `GetInfo` reports), as on Horizon (`hle_svc.c`). Pokémon Legends: Z-A allocates and frees everything it can at startup and the SDK keeps the freed pages mapped until a map fails, then unmaps them and retries; without the limit they stay mapped and the game maps past the pool. Off by default: Ace Attorney Investigations Collection maps a little more than the 1 GB pool its profile reports |
| `stdalloc_cap` | With virtual address memory, the address space `StandardAllocator::Initialize(nullptr, size)` gets when `size` is more than the alias region has room for (`hle_svc.c` "StandardAllocator's address space", a gate the rewriter places by symbol in every `sdk`). Default `0`: the room, the alias region less the regions already made (at least the 62 MB node area) and 256 MB for the regions made after it. Needed when those need more (Absolum's .NET garbage collector takes 1.5 GB after its `malloc`): the NSP script's address region check sets it in a provisional or derived profile from what it measured, and checks a value set by hand. A request that fits is never changed |
| `present_interval` | `game` (the default): frames are paced by the present interval the game sets, as on the Switch (a frame presented with interval 2 stays on screen for two display ticks; a game that keeps asking for 2 for a second gets the 30 fps target, and on iOS the display link then runs at 30 Hz; Mario 1.2.1 switches to 2 for single frames after a slow one; Engage asks for 2 on the Switch, and a rewriter cut on its 2.0.0 `main` makes it ask for 1, see `0100A6301214E000/game.tsv`). `display`: one frame per display tick whatever the game asks, which is only right for games whose logic does not count frames (D2R's runs on its own 25 Hz clock; Monster Train 2 asks for 2 and steps by Unity's frame time, so it runs at 60 fps: screenshots at the same moments match at 30 and 60 fps). `DM_PRESENT_INTERVAL=game` / `display` overrides it, also from `Documents/env.txt` |
| `present_snap` | `on` / `off` (the default): present snapshots by default (`nvn_mtl.m` `mtl_snap_on`, a ring of 2 host textures): each present is copied and the game's swapchain texture is free at once, so the GPU does not idle between frames ([Present snapshots](../../docs/reference/frame-rate.md#present-snapshots-the-games-swapchain-texture-free-at-once)). A setting the game was tested with: the iOS launcher's **Frame buffering** defaults to it, and `DM_PRESENT_SNAP` / `Documents/env.txt` override it. Breath of the Wild sets it on |
| `pso_mtl4` | `on` / `off` (the default): on iOS, render pipelines through MTL4Compiler by default, with the pipeline archive, the prepare step, the hybrid pipelines and the prebuild (on by default on the Mac; `DM_PSO_MTL4=1` / `0` overrides). A setting the game was tested with on an iPhone. It also gives the app the [pipeline archive made offline](../../docs/development.md#the-pipeline-archive-made-offline) (`build_ipa.sh` builds it for `g18p` unless `DM_PSO_GPUS` says otherwise; without the key, none) and the pulling libraries (`_bp`). Breath of the Wild sets it on |
| `fps_target` | Frame rate cap of the rewriter's frame step, for builds that have one (`FRAME_STEP` in the profile's `patches.py`: Tears of the Kingdom 1.4.3, Breath of the Wild 1.9.0, Hyrule Warriors: Age of Calamity 1.3.0, Paper Mario: The Thousand-Year Door 1.0.1, ASTRAL CHAIN 1.0.1), or the engine's own frame limit the host writes into game memory (`hle_fpslock.c`: Monster Hunter Rise 16.0.2, which also needs `present_interval display`; Octopath Traveler 0 1.0.5, Unreal's `t.MaxFPS` and `rhi.SyncInterval`, so the game asks for present interval 1 itself; Luigi's Mansion 3 1.4.0, its frame limiter's milliseconds per frame, with `present_interval display`; Pokémon Brilliant Diamond 1.3.0, Unity's integer `Application.targetFrameRate`, also with `present_interval display`). `0` (the default): the game's own step. `30` / `60`: the host decides each frame's step in whole vsyncs and keeps the frame on screen that long (`hle_nvn.c` `fstep_tick`, `nvn_fstep.h`): 1 at a 60 cap, 2 when 60 does not hold or at a 30 cap (and while a prerendered movie plays), plus one vsync while the game is behind real time after a late frame. Presents follow that hold instead of the interval the game sets. `DM_FPS_TARGET` overrides it, also from `Documents/env.txt`; the log line `frame step: game time … of real time` checks the game's speed |
| `fps_hold` | `on` (the default): with `fps_target`, three late frames in a row raise the hold above the cap's vsyncs (30 or 20 fps), and steady frames that fit try the lower hold again. `off`: frames are never held longer than the cap's vsyncs; each is shown as soon as it is ready. For a game whose updates do not wait for the presents and step by the cap whatever the hold (Age of Calamity): there the hold only paces the screen. `DM_FPS_HOLD=on` / `off` overrides it, also from `Documents/env.txt` Paper Mario: The Thousand-Year Door sets it off too: its frame waits for three render worker threads, and holding frames left them idle, so a frame cost more the longer the hold (the hold stuck at 3 vsyncs, 20 fps, on a loaded Mac; 32–58 fps without holds) |
| `core_preempt` | `on` / `off` (the default): additionally check higher-priority runnable threads pinned to the caller's core after every thread SVC (`hle_thread.c`). `DM_PREEMPT=0` / `1` overrides it. Wake-triggered same-core handoffs apply to every game with either setting and wait until the threads the wake made runnable block, with no wall-clock cap. Metroid Dread no longer needs this key or its movie request-ring gates; see [Same-core wake handoffs](../../docs/reference/threads.md#same-core-wake-handoffs) |
| `alloc_big_align` | `on` / `off` (the default): `nn::mem::StandardAllocator::Allocate` aligns blocks of 64 KB or more to 64 KB (`hle_os.c`). Vanillaware's engine (13 Sentinels: Aegis Rim, Unicorn Overlord) takes a memory pool's storage from malloc and cuts ring heaps out of it at 64 KB alignment relative to the pool's GPU address. On the Switch that address is big-page aligned; the host's is the CPU address, only 4 KB aligned, so the last ring heap does not fit and keeps base 0, and the game writes through a null pointer (13 Sentinels entering its first battle, Unicorn Overlord past the title screen) |
| `shared_font` | Optional, decided by the game files without it (the SDK version the host reads from the `sdk` module, `loader.c` `dm_sdk_major`, and `main`'s imports, `dm_main_imports`): what the shared-font address points at. `bfttf` (SDK 11 and later, when `main` imports `ScalableFontEngineHelper::Decode`) = the 8-byte BFTTF header (with the "already decoded" marker) followed by the TTF, which Mario passes to `Decode`; `ttf` = the TTF itself, which SDK 10 and older games hand straight to `ScalableFontEngine::LoadFont` (Super Mario 3D World + Bowser's Fury: with the header the engine fails with error 403 and the game crashes looking up a glyph; Odyssey, Link's Awakening, Age of Calamity and Three Houses alike) and Unity's player to FreeType (Patrick's Parabox: with the header its Japanese, Korean and Chinese text drew nothing). No profile sets it |
| `device_api` | Optional, decided by the SDK version without it: the NVN API version `major.minor` that `nvnDeviceGetInteger` reports, 55.15 for SDK 11 and later (what Mario requires) and 53.313 for SDK 10 and older (`hle_nvn.c` `nvn_api`). Games check it and skip creating their device on a mismatch: Super Mario 3D World + Bowser's Fury (SDK 10) requires major 53 and minor ≥ 313. No profile sets it. `DM_NVN_API` overrides. Not part of the shader chain version (only `shader_*` keys are) |
| `depth_mode` | `gl` (the default, NVN's near = −w) or `zero` (near = 0): the depth mode the game sets with `nvnDeviceSetDepthMode`, for the pipeline bases made before the game runs: the [pipeline archive made offline](../../docs/development.md#the-pipeline-archive-made-offline) for the iPhone (`main0` or `main0_z` vertex entries; built for a game with `pso_mtl4 on` or with `DM_PSO_GPUS`) and the NSP script's Mac prebuild (`DM_PSO_PREBUILD_DEPTH`). The host itself follows the mode the game sets; a wrong value only costs the device its own prebuild. Great Ace Attorney Chronicles, 13 Sentinels (both releases), Big Brain Academy, Link's Awakening, Engage and ASTRAL CHAIN set `zero` |
| `render_base` | The 3D views' base size, docked then handheld (`1600x900 1280x720`): what `DM_RENDER_SCALE` scales. `settings.py` writes it into the render scale's row of `launcher.json` as `base`, so a front end can show the setting in pixels. Only for a game with a render scale stub, and only a size its note documents |
| `op_mode` | The operation mode the game sees at start: `console` (the default, docked) or `handheld`. Super Mario Galaxy and Captain Toad: Treasure Tracker use `handheld`: docked, their pointer is the Joy-Con gyro, which the host does not have; handheld, it is the touch screen, which the iPhone front end feeds with the touches no gamepad control takes (in the 3D layout, outside the right third of the screen, which is the right stick; `DmGamepad.m`). Handheld also renders at 720p. `DM_OPMODE=0` / `1` overrides, also from `Documents/env.txt` |
| `perf_mode` | `follow` (the default): the performance mode follows the operation mode (Boost docked, Normal handheld); `boost`: Boost whatever the operation mode (`hle_oe.c`), a combination a Switch 1 never reports, so a profile choice. Rhythm Heaven Groove plays handheld with Boost: docked means a TV to the game, which then asks for a latency calibration, and only Boost gives its 1920x1080 draw size (handheld Normal draws 1280x720) |
| `pad_layout` | The iPhone virtual gamepad's layout on first launch: `simple` (the default) = 2D (left stick, A/B/X/Y, shoulders, − / +), `full` = 3D (adds a D-pad, a right stick and L3 / R3; the right stick is not drawn: a touch landing in the right third of the screen on no control holds it, centred where it lands), `touch` = Tap (no control but the layout button: every touch goes to the game's touch screen; needs `op_mode handheld`, Balatro). The layout button at the top centre goes to the next layout in game (Tap only in handheld mode) and the app remembers the player's choice (`DmGamepad.m`) |
| `feedback_snap` | Fragment shader keys (prefixes) whose draws sample a colour attachment of their own render pass and must read a snapshot instead (`fb_prepare` in `nvn_mtl.m`). Writing and sampling one attachment in a render pass is undefined in Metal: the Mac usually happens to read the old content, the iPhone sometimes reads whole 16×16 blocks as magenta. Each such draw costs a pass split and a copy, so only the shaders seen to break are listed (Super Mario 3D World + Bowser's Fury: its final composite; Captain Toad: Treasure Tracker: the same agl program, whose blocks showed on the Mac too). `DM_NVN_FB_SNAP=0` / `all` / `<keys>` overrides, also from `Documents/env.txt` |
| `tex_flush_upload` | `on` / `off` (the default): a texture the game fills from the CPU in place (no packaged layout, no `WriteTexels`) and publishes with `nvnMemoryPoolFlushMappedRange` is uploaded from guest memory like a packaged one, and re-uploaded when its pages change (`ft_add` in `hle_nvn.c`). Tears of the Kingdom streams all its textures this way; without it the 3D scene is black. Off by default because a flushed range in another game could cover an image a shader writes, whose GPU content the upload would replace. Without the key, every game still gets a texture's storage once when the game flushed it before the texture's first use (`nvn_flushed` in `hle_nvn.c`: Breath of the Wild's sky tables); the key is for textures the CPU writes again after that |
| `render_cond` | `on` / `off` (the default): `SetRenderEnableConditional` / `SetRenderEnable` are recorded and evaluated at replay, and a timestamp report's counter field is 0 as on the GPU (`OP_RENDER_COND` in `nvn_mtl.m`). Tears of the Kingdom wraps its clears in conditions that skip clearing a target nothing has drawn into since its last clear; without the key every clear runs and its 3D scene is black. Animal Crossing: New Horizons too: four clear-only passes wiped its composite before the window copy, and every frame was black. Off by default: other games draw everything, as before |
| `tex_dirty_upload` | `on` / `off` (the default), needs `tex_flush_upload`: a registered texture whose pages the CPU writes after `nvnTextureInitialize` without any NVN flush is uploaded as well (its pages are write-protected at registration, checked on first use; `nvn_ft_written` in `hle_nvn.c`). Tears of the Kingdom fills its ASTC terrain material arrays this way; without it the cave is magenta. FINAL FANTASY TACTICS - The Ivalice Chronicles loads its UI textures straight into CPU-uncached pools, which need no flush, after creating them; without both keys its screen stays black. The Great Ace Attorney Chronicles copies every texture into a CPU-cached pool after creating it and never flushes (its `main` has no cache maintenance at all): black too |
| `tex_dyn_slots` | `on` / `off` (the default): a shader whose texture handle the translator could not trace to one slot gets Ryujinx's fallback arrays (`<stage>_<t|tb|s>_cb0_40000000`: 32 textures and samplers indexed by the handle's texture / sampler ID). With the key the replay puts slot indices in that shader's c2 texture and sampler tables (texture slot r → r, sampler slot r → r << 20) and binds slot k at element k, so the shader's own arithmetic picks the binding (`NVN_RB_TEXD` / `NVN_RB_SAMPD` in `nvn_mtl.m`). Right only where the handle comes from c2's tables: Octopath Traveler 0's 61 such vertex shaders read separate texture slots 0-10 and samplers 0-4 there and nothing else (FINAL FANTASY RESONANCE's 2,077 the texture table words 0x168-0x1c0 and sampler table words 0x568-0x590); Paper Mario: The Thousand-Year Door's 2,386 pixel shaders read combined texture slots 0, 7, 8, 13 and 28 (slot r → r \| r << 20). Off, such shaders do not load. Not part of the shader chain version |
| `gpu_timestamps` | What a timestamp report (`nvnCommandBufferReportCounter`) records: `submission` (the default) = the time the whole submission completed, the same for every report in it; `segment` = the GPU start (`TIMESTAMP_TOP`) or end (the other types) of the segment the report was encoded in (the host commits a submission in segments of 64 draws, `sub_cut` in `nvn_mtl.m`), so two reports of one submission bracket the GPU time between them, as on hardware. Breath of the Wild needs `segment`: its frame-interval controller and its dynamic resolution read the GPU time between the frame's first and last report, and with one time for both they read 0 (the game kept 60 fps in slow motion when the GPU could not keep up). `DM_NVN_GPU_TS=segment` / `submission` overrides it, also from `Documents/env.txt` |
| `vconv_range` | `on` / `off` (the default): vertex attributes the replay converts on the CPU (integer shader inputs fed float-type formats and the like, `vconv_bind` in `nvn_mtl.m`) are converted only over the vertices each draw reads (first / count, or the smallest and largest index of a direct indexed draw, kept while the index pages stay unwritten) instead of the whole bound range. Age of Calamity binds whole mesh pools: in a battle that was millions of vertices a frame. Indirect draws, per-instance streams and indices with a GPU write queued on them keep the whole range. `DM_NVN_VCONV_CHECK=1` scans the indices again at every reuse and reports differences; `DM_NVN_VCONV_RANGE=0` / `1` overrides it |
| `shader_groups`, `shader_note` | `shader_split.py` source groups and the map header note (Engage: `engine` for `data.unity3d` and `Resources/`, `assets` for the Addressables bundles) |

### What the game files decide at every build

These are not profile keys: the tools read them from the game files each time, by rules checked against every profile (`dm/games/profile_guess.py` `shader_sources`).

- **Graphics API.** A game whose `main` imports Vulkan functions and not `nvnBootstrapLoader` renders with Vulkan, which the host implements on Metal (`dm/gfx/vk/hle_vk.m`; Hades, Super Mario Galaxy). Its shaders are its SPIR-V files, translated by `dm/gfx/tools/vk_shaders.py <RomFS> <shader dir> [--ios <dir>]` into `<dir>/vk/`, and `build_ipa.sh` ships `$MSL_IOS/vk` instead of the NVN chain libraries (`GFXAPI`, which the NSP script passes; by hand the shader libraries there tell). The NVN shader tables stay empty.
- **SPIR-V generated at run time.** A Vulkan game without SPIR-V files in its RomFS generates its shaders at run time (Super Mario Galaxy: Hagi's GX shaders, 3,089 modules created from its pipeline database while it shows "Initializing system..."). The NSP script captures them on the Mac with `DM_VK_CAPTURE=<dir>` (`dm/gfx/vk/hle_vk.m`), booting the game in the Mac host until every key the profile's `vk_capture.tsv` lists is there, and `vk_shaders.py` translates the capture folder with the RomFS. An OpenGL game with no GLSL source in its RomFS or `main` (Balatro: LÖVE puts its shaders together in Lua) is captured the same way with `DM_GL_CAPTURE=<dir>` (`dm/gfx/gl/hle_gl.m`) until every file in `gl_capture.tsv` is there, and `gl_shaders.py` translates the capture.
- **Shaders compiled at run time.** A game whose `main` holds GLSL sources and calls `glslcCompile` in a module it loads compiles some shaders at run time with its own GLSLC (Fire Emblem: Three Houses: 9 built-in KTGL shaders at boot), so no game file holds them. The NSP script captures them on the Mac with `DM_GLSLC_CAPTURE=<dir>` (`dm/host/hle_glslc.c`), booting the game in the Mac host until every capture file the profile's `shader_map.tsv` names is there, and `shader_split.py` reads `<version dir>/glslc_capture/*.glslc` (beside that version's `RomFS`) or `$DM_GLSLC_CAPTURE`; without the folder it prints a red line, those shaders get no translation and the tables differ from the profile's.
- **Unreal paks.** An Unreal Engine game keeps its shaders in the shader code archives of its paks (`*.ushaderbytecode`; a pak is Unreal's when its footer has Unreal's magic, `uepak.py` `unreal_paks`), each shader Oodle-compressed, which only the game's own Oodle decodes. The NSP script (`decode_ue_shaders`) takes the archives out of the paks with `dm/gfx/tools/uepak.py` (the pak index is AES-encrypted with a key it finds in `main`), boots the game in the Mac host with `DM_OODLE_JOB` (`dm/host/hle_oodle.c`: the game's `OodleLZ_Decompress`, found as the one function that references the library's own name string, decodes every shader right before `nnMain`, then the host exits) and writes the decoded archives to `<version dir>/ue_shaders`, where `shader_split.py` reads them (`DM_UE_SHADERS` overrides; without them it stops). Octopath Traveler 0: 10,596 shaders in 2 s. Unreal Engine 4.26 and later compress shaders with LZ4 unless the project picks another format (DRAGON QUEST VII Reimagined, 4.27: 64,949 shaders, its packages in IoStore containers beside the pak); `uepak.py` decodes those itself (`lz4_shaders`) and the NSP script skips the host run. A pak whose index lists cooked packages and no `.ushaderbytecode` (`uepak.py` `inline_shaders`: legacy version 4 paks, and OCTOPATH TRAVELER II's version 11 pak with `bShareMaterialShaderCode` off) keeps its shaders inline in the packages; `shader_split.py` reads those file by file (`ue_legacy_jobs`) and the NSP script decodes no archive.
- **Trinity packs.** Game Freak's Trinity engine keeps nearly every file in Oodle-compressed packs (`arc/data.trpfs`, files starting `ONEPACK`; Pokémon Legends: Z-A). `shader_split.py` decodes them with the game's own `OodleLZ_Decompress`, run by the Mac host in its tool mode (`dm/host/hle_oodle.c`, `DM_OODLE_SERVE`). The split needs `DM_OODLE_HOST` (the Mac host) and `DM_OODLE_MODULES` (the rewritten modules); the NSP script builds the host and sets both when the split runs.
- **GLSLC outputs in `main` and loose files.** `shader_split.py` reads GLSLC compiler outputs (magic `0x19866891`, each one checked whole) in every game's `main`: 13 Sentinels and Unicorn Overlord keep all their shaders there, the KTGL games some; Unity's player has 11 of its own beside the game's shaders in its bundles (Engage, Ace Attorney Investigations Collection). It reads them as well in any RomFS file no other reader takes: FINAL FANTASY TACTICS - The Ivalice Chronicles keeps 257 in its technique files (`resources_nx/release/master/shader/*.tec`), Absolum (FNA3D) 163 in `shaders.nx.bin`.
- **MonoGame effects.** `shader_split.py` reads MonoGame's MGFX effects wherever they are (`mgfx_effect`): Celeste keeps 13 in its XNB files (`Content/Effects/*.xnb`) and SpriteBatch's built-in one is in `main`.
- **Single-stage programs.** A GLSLC output, an Unreal shader record or a MonoGame effect's shader is a program of one stage, combined with others per draw (a MonoGame pass may name no vertex shader and keep SpriteBatch's), so its real vertex / pixel pairs exist only at run time. `shader_split.py` pairs each such vertex shader with a cover of pixel shaders that together read every input any pixel shader reads, and `dmsh_build.py`'s `vs_pad` makes it link with any pixel shader (13 Sentinels, Unicorn Overlord, the KTGL games, Octopath Traveler 0, Celeste). BNSH variations, Unity programs and the MSHD and SDF files pair their own shaders.
- **NVN driver tables.** The driver's constant segments have one layout in every game (`dmsh_build.py`, filled by `nvn_mtl.m`): c2 holds the handles bound with `nvnCommandBufferBindTexture` at `0x20 + 8 × slot`, images at `0x120 + 8 × slot`, separate textures at `0x168 + 8 × slot` and separate samplers at `0x568 + 8 × slot`; c0 the storage buffer tables (vertex 0x110, compute 0x310, pixel 0x510). Separate texture binds are texture slots 128 + n of the host's table (`NVN_SEPTEX_SLOT0`), so a game may bind both kinds in one frame (Monster Hunter Rise: combined slot 0 beside separate slots 0-4), and a combined slot past 31 (Hades II binds slot 34) lies in the image table, where a bound texture wins over an unbound image. The translator (`dmsh_xlate/Program.cs`) numbers slots the same way when it looks up per-slot texture / image types in `shader_facts.tsv`.

## Adding a game

1. **Survey.** Run `scripts/build_ipa_from_nsp.py` on the game's NSP folder. Without a profile it extracts only the code modules and a RomFS list, then writes `<NSP folder>/<title ID>_survey/survey.md`. The survey covers modules and build IDs, SDK, engine and graphics API markers, RomFS containers, the rewrite-site census, and imports compared with the host hooks.
   Unless told `--survey-only`, the script then writes a **provisional profile** to `<NSP folder>/<title ID>_build/profile/<title ID>/` (outside the repository) and carries on to a best-effort IPA. `dm/games/profile_guess.py` fills its `game.tsv` from the game files, each key under the reason for it (the survey lists them too):
   - identity and name, the build row it found, the largest RomFS file as `romfs_marker`, big file types in `zip_store`, `shader_groups` by RomFS top-level folder;
   - `modules`: a subsdk is loaded only when `main` imports something from it that the host does not supply itself, or, once the rewrite stage has collected the game's NROs, when one of them does (`nro_modules`: Portal's Source engine NROs use the movie module and GLSLC, which its `main` does not). Under the Vulkan layer the Vulkan loader and GLSLC are dropped; movie modules and a GLSLC that `main` calls stay. NVIDIA's OpenGL and Vulkan drivers are never loaded: they program the GPU through NvRm, which the host does not have (Absolum imports EGL for SDL3's GL support and renders with NVN);
   - the memory partition. Without a system resource in the NPDM: the 4 GB layout, its heap region the pool − 64 MB − the bss of giant modules (Metroid Dread: 0x90000000). When `main` never sets a heap: a 1 GB block (Super Mario Galaxy). With a system resource: Engage's 5 GB block with the hardware pool, which the heap check corrects when the game sets a heap (without one the game maps everything in the alias region itself: Hollow Knight: Silksong, Absolum, Pokémon Legends: Z-A);
   - Engine presets: Hagi (`hagi.config`) gets `op_mode handheld` and `pad_layout full`.

   `python3 dm/games/profile_guess.py --check` runs these rules on every profile in `dm/games` (with the extractions under `/Volumes/T7/nsp`) and lists the keys where the guess differs.

   After the rewrite, `mem_used` is set when the rewritten modules exceed the default 64 MB (a game without a system resource); keep it when the profile moves into `dm/games` (Cuphead without it: a 0.03 GB heap and an abort at the first allocation). A game with a system resource that sets a heap then gets a **heap check**: it boots in the Mac host (headless, no shader libraries) until its first heap request, or 5 s into `nnMain` without one, at most 30 s (`HEAP_CHECK_SECS`). A request the heap region cannot hold (`SetHeapSize OOM`) sets the alias / heap split (Tears of the Kingdom asks for 0x86000000, which gives its hand-made 0x98000000 alias region; Wonder's 1.44 GB gives 0xc0000000), and a pool check that fails for good sets `mem_used`. A run with no heap set whose alias region is used up to within 0.25 GB of its end sets a 1 GB pool: `nninitStartup` handed pool − used to StandardAllocator as one address region and the game reserves regions of its own beside it (Unity 2020 and 2022: AI: THE SOMNIUM FILES - nirvanA Initiative reached +4.26 GB of 4.38 in 5 s with the hardware pool; Unity 2018.3's Raging Loop maps only inside StandardAllocator's region and keeps it). The 1 GB is a starting point, not a measurement: Advance Wars 1+2: Re-Boot Camp (Unity 2020.2) ran its malloc dry at its first loading screen with it and needs 2 GB. The profile gets a `# heap check:` line, and later runs keep its keys (delete the line to check again; `--no-heap-check` / `HEAP_CHECK = False` skips it). Other games skip the check, so it costs nothing for them and a few seconds plus a Mac host build for the rest.

   No short run decides `render_cond`, `tex_flush_upload`, `tex_dirty_upload`, `core_preempt`, `gpu_timestamps`, `feedback_snap`, `vconv_range` or `present_interval`, nor `op_mode` / `pad_layout` outside the presets: they keep their defaults for a developer. A minute's boot without input counted Wonder's clears under render conditions (65,536 or more; it needs no `render_cond`) as readily as Tears of the Kingdom's, saw Breath of the Wild flush 32 CPU-filled textures with data (it needs no `tex_flush_upload`) and Age of Calamity none (its movie frames come later), and the old host's Metroid Dread movie crash showed in about half the runs (now resolved by [shared wake handoffs](../../docs/reference/threads.md#same-core-wake-handoffs)). On the iPhone, `Documents/env.txt` can still set `DM_OPMODE`, `DM_NVN_GPU_TS` and `DM_NVN_FB_SNAP`, and the pad layout switches in game.

   The shader tables, `c1_full.tsv` and `cb_extent.tsv` are written into the same folder. A second run reuses that `game.tsv` (delete it to survey again), so it can be edited by hand. `--provisional` builds from a provisional profile even for a game that has one in `dm/games`, to test the guesses: `DM_GAMES_FIRST=1` makes every tool search `DM_GAMES_DIR` first, and the IPA gets `-provisional` in its name. Tools find profiles outside `dm/games` through `DM_GAMES_DIR` (folders separated by `:`, searched after `dm/games`); `make -C dm/host` reaches such a profile through the symlink `dm/host/gen/<id>/profile`. To keep a game, move the folder into `dm/games/` and bring it up as below.
2. **Profile.** Copy `010015100B514000/game.tsv` (or the provisional profile) to `dm/games/<title ID>/game.tsv` (the base game's title ID, as the folder name) and fill in the identity, `build` rows and `modules`. Leave the memory values as Mario's until the bring-up measures them ([docs/agents](../../docs/agents/README.md)).
3. **Shader tables.** Run `DM_GAME=<id> shader_split.py … dm/games/<id>/shader_map.tsv dm/games/<id>/shader_facts.tsv dm/games/<id>/shader_pairs.tsv dm/games/<id>/pso_pairs.tsv`. Add an empty `shader_map.tsv.conflicts.tsv`, `bf_want.tsv` and `pso_manifest.tsv` (copy Mario's headers).
4. **Shaders.** `DM_GAME=<id> [IOSOUT=…] dmsh_build.py <pkg> <cache>`, with a cache directory of the game's own. A full build also writes `c1_full.tsv` and `cb_extent.tsv` (local files, not committed). The chain version needs nothing: it is the same for every game.
5. **Host bring-up.** Run `make -C dm/host GAME=<id>`, then work through the bring-up loop in docs/agents. Code that only fits one game is gated by its build ID (`dm_main_build()`), never left to run on every game.
6. **IPA.** Use `GAME=<id> dm/ios/tools/build_ipa.sh`, or the NSP script, which now finds the profile.


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

