# Mario Party Superstars (`01006FE013472000`)

Update 1.1.1 (v131072) over the base game, SDK 11.4.3. NintendoWare Bezel Engine (threads `Bezel:*`) on NVN, with NEX / Pia online. Modules `main`, `subsdk0` (GLSLC), `subsdk1` (the movie module) and `sdk`; the boards, minigames and menus are 126 NROs loaded at run time. On the Mac (Apple M5): the opening, the menus, a board (`hsbd01`) and a minigame (`hsmg402`) twice, back to the board, 6.7 minutes in all with no red line but one, at about 60 fps (frame time p50 16.8 ms, p95 19 ms over 73 windows of 300 frames; 966 of 21,566 frames dropped on screen) and 2.3 GB resident. The log names no controller connected in that run, so physical controllers and more than one player were not part of it. Not tested on the iPhone.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `MAX_MOD` 160 | `dm/host/dm.h` (every game) | 4 modules and 126 NROs; with 128 the host stopped at start: [More NROs than module slots](../../../docs/reference/services.md#more-nros-than-module-slots) |
| Argument buffer members with a raster order group | `dm/gfx/tools/dmsh_build.py` (every game, a chain change) | the game stopped when a match started: [Argument buffer members with a raster order group](../../../docs/reference/shaders.md#argument-buffer-members-with-a-raster-order-group) |
| `tex_dyn_slots neutral` | [`game.tsv`](game.tsv) | the characters were drawn without their materials; see [Lights and characters](#lights-and-characters) |
| `depth_mode zero` | `game.tsv` | the game sets the near = 0 depth mode (log: `depth mode: near=0, vertex shaders use the main0_z entry`), for the pipeline bases made before it runs |
| `sys_resource 0x0`, a 4 GB block with a 3.25 GB heap region (`mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000`) | `game.tsv` | the NPDM declares no system resource; `main` sets a heap of the pool − 64 MB (3.14 GB), as the provisional profile guessed |

No game-specific rewriter cut and no `patches.py`.

## Lights and characters

- **Resolved by a workaround: characters drawn without their materials** (the NPCs Koopa Troopa, Toad, Kamek and Shy Guy, and parts of the boards), with `[RED] shader … reflection conversion failed: a texture handle the translator could not trace to one slot (profile key tex_dyn_slots off)` and `failed to load`. 6,370 of the 82,790 shaders (among them the materials of `npc003_nokonoko.bnbshpk`, `npc022_kinopio.bnbshpk` and `hsmn_bg.bnbshpk`) take their clustered lights' shadow maps and projected textures through handles the fallback arrays cannot reach, 6,364 of them from a storage buffer. With `tex_dyn_slots neutral` they all load, and the lights draw as lights without a shadow or a projected texture: [Texture handles from the game's own data](../../../docs/reference/shaders.md#texture-handles-from-the-games-own-data).
- The game's two Bezel PBR pixel shaders (`System.nx.bea/_BezelSystemResources/pbr/shader/bezel_pbr.bfsha`) have 106 bindings, over the old limit of 96 (`NVN_REFL_MAXB`, now 128), and the clustered-light materials up to 247 argument buffer entries (`NVN_REFL_MAXARG`, now 256).
- Checked offline: every shader of the build, against the host's load rules (`nvn_refl_from_json`, the argument buffer layout check), loads.

## Pipelines

- A first start with no pipeline archive builds the bases of all 140,036 shader pairs before the first frame (`pipeline prebuild … (prepare step, before the first frame)`): about 85 a second on an M5, about 27 minutes, about 7.7 GB of archive at 110 MB per 2,000 bases. The progress is kept, so a restart goes on from there. `DM_PSO_PREBUILD=0` in the app's `env.txt` skips it; the bases are then built as the game first draws with them, served meanwhile by the hybrid pipelines.
- The pipeline archive made offline (`--gpu`) failed for `applegpu_g17g` (`metal-tt`: `air-tt: applegpu-nt command failed`, no further detail); the app was packed with `--gpu none`.

## Mac routes

- The keyboard reached a board and a minigame. No `DM_KEYS` route is recorded yet.

## Found on this game

General host and tool changes first needed here; each applies to every game:

- **`MAX_MOD` 160**: [More NROs than module slots](../../../docs/reference/services.md#more-nros-than-module-slots).
- **Raster order groups in the reflection** (`dmsh_build.py` `reflect`): [Argument buffer members with a raster order group](../../../docs/reference/shaders.md#argument-buffer-members-with-a-raster-order-group).
- **`tex_dyn_slots neutral`**, `NVN_REFL_MAXB` 128 and `NVN_REFL_MAXARG` 256: [Texture handles from the game's own data](../../../docs/reference/shaders.md#texture-handles-from-the-games-own-data).

## Open

- Controllers: two or more players with physical controllers, and rumble (an earlier run logged `[RED] haptics: controller left handle create player: Couldn't communicate with a helper application`).
- The clustered lights' shadows and projected textures: a texture pool the shader indexes by the full ID would need a translator change.
- `SetScissors first=0 n=2: the host uses only index 0` (logged at start); whether a split-screen minigame draws its second viewport is not checked.
- `[RED] sampler ID 4095 not registered`, once at start.
- The other boards, the minigames beyond `hsmg402`, the iPhone, and an empty `pso_manifest.tsv` (not recorded yet).
