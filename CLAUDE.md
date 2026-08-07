# CLAUDE.md

Guidance for Claude Code working in this repository.

## Project

**Soccer Course** — a 2D arcade soccer game built in **Godot 4.4** (GDScript, GL Compatibility renderer). This repo doubles as a *course* for building a 2D soccer game, so code is written to be readable and instructive rather than maximally terse.

Low-res pixel art: internal viewport is **280×180**, integer-scaled up to 1400×900 (`window/stretch/mode="viewport"`, `scale_mode="integer"`). `default_texture_filter=0` (nearest) keeps pixels crisp — never apply smoothing/filtering to art assets.

## Running

Open in the Godot 4.4 editor and press Play, or from CLI:

```sh
godot --path .          # open editor
godot --path . --main   # run the game (if your godot build supports --main)
```

There is no test suite, linter, or build step. `.godot/` and `.import/` are gitignored (generated).

## Architecture

The codebase is built around a **state-machine pattern applied uniformly** to four domains: the game/match (`GameManager`), the ball, the player, and the screens. Each domain follows the same triad:

- **`*State`** (extends `Node`) — one class per state. Receives context via `setup(...)`, holds a `state_transition_requested` signal, and calls `transition_state(Enum.VALUE, data)` to switch. Override hooks like `on_animation_complete()`, `can_carry_ball()`, etc. default to no-op/`false`.
- **`*StateFactory`** — maps an `enum` → state class in a `Dictionary`, instantiated fresh via `.new()` in `get_fresh_state()`.
- **`*StateData`** — a plain object carrying transition payload, built fluently: `PlayerStateData.build().set_shot_direction(v).set_shot_power(p)`.

The owner (e.g. `Player`, `Ball`, `GameManager`) holds `switch_state(enum, data)` which `queue_free()`s the old state, creates the new one via the factory, wires its `state_transition_requested` signal back to `switch_state` (using `.bind()`), and `call_deferred("add_child", ...)`s it. States are **Nodes added as children**, not plain objects — this is intentional and consistent.

### Screens

`SoccerGame` (root, `scenes/soccer_game.gd`) is itself a screen state machine over `ScreenType {MAIN_MENU, TEAM_SELECTION, TOURNAMENT, IN_GAME}`. `Screen` (base) carries a `screen_transition_requested` signal and a `ScreenData` payload; `ScreenFactory` builds fresh instances. `WorldScreen` is the in-game screen and kicks off `GameManager.start_game()`.

### Autoloads (singletons, see `project.godot` `[autoload]`)

| Name | File | Role |
|------|------|------|
| `DataLoader` | `utils/data_loader.gd` | Loads `res://assets/json/squads.json` once at `_init()`. Holds `countries` and `squads` (country → `[PlayerResource]`). |
| `GameEvents` | `scenes/screens/world/game_events.gd` | **Global signal bus** — the decoupling layer. Signals: `ball_possessed`, `ball_released`, `game_over`, `kickoff_ready`, `kickoff_started`, `impact_received`, `score_changed`, `team_reset`, `team_scored`. Prefer emitting/connecting here over direct cross-node references. |
| `GameManager` | `scenes/game_manager/game_manager.gd` | Owns `current_match` (`Match`), `player_setup` (2 country strings), the game state machine, the match timer (2 min), and hitstop (impact pause). `process_mode = ALWAYS` so it can unpause the tree. |
| `SoundPlayer` | `scenes/audio/sound_player.gd` | SFX via a pool of 4 `AudioStreamPlayer`s. `SoundPlayer.play(SoundPlayer.Sound.X)`. |
| `MusicPlayer` | `scenes/audio/music_player.gd` | Per-screen music; `Screen._enter_tree()` calls `MusicPlayer.play_music(music)`. |

### Match & tournament

- `Match` (`scenes/screens/tournament/match.gd`) — tracks home/away countries, goals, winner, final score. `increase_score(country_scored_on)` increments the *opposite* side. CPU-only matches `resolve()` with random scores.
- `Tournament` (`scenes/screens/tournament/tournament.gd`) — 8-country single-elim bracket: `QUARTER_FINALS → SEMI_FINALS → FINAL → COMPLETE`. `advance()` resolves the current stage and seeds the next from winners.
- `ActorsContainer` (`scenes/screens/world/actors_container.gd`) — spawns both 11-player squads from `DataLoader`, wires goals/control schemes, and runs CPU "on-duty" steering weights + player-swap logic (closest CPU to ball takes control when a human requests a swap).

### Players & ball

- `Player` (CharacterBody2D): 11 per squad — 1 `GOALIE` + 10 outfield in a 4-3-3 across `DEFENSE/MIDFIELD/OFFENSE`. `ControlScheme {CPU, P1, P2}`. `SkinColor` and team color are applied via the palette shader (not separate sprites). Has its own state machine (15 states incl. MOVING, TACKLING, SHOOTING, PASSING, HEADER, VOLLEY_KICK, BICYCLE_KICK, CELEBRATING, MOURNING…). AI is delegated to an `AIBehavior` (field vs goalie) built by `AIBehaviorFactory`.
- `Ball` (AnimatableBody2D): 3 states (CARRIED, FREEFORM, SHOT). Custom `height`/`height_velocity` for a 2.5D bounce; sprite is offset `Vector2.UP * height`. `pass_to()` computes intensity from distance and friction; high passes arc. `scoring_raycast` points along velocity to detect the scoring area.
- **Hitstop**: high-impact collisions `emit impact_received(pos, true)` → `GameManager` pauses the tree for `DURATION_IMPACT_PAUSE` (100ms), then unpauses in `_process`.

### Rendering: palette shader

`shaders/replace_color.gdshader` swaps pixel colors from two palette textures (`skin_palette`, `team_palette`) — one sprite serves all skin tones and all team colors. `team_color` is the country's index in `DataLoader.countries`. When adding teams/players, keep palette indices consistent.

## Conventions

- **GDScript style**: `class_name` PascalCase at top of file. Variables/functions `snake_case`. Constants and enum members `UPPER_SNAKE_CASE`. Types use `:=` inference or explicit `: Type`.
- **Node access**: `%UniqueNodeName` (`@onready`) for scene-internal refs; `@export` for editor-wired dependencies (ball, goals, control scheme).
- **Files**: each script gets its own `.gd` + an auto-generated `.gd.uid` (Godot 4 UID). Never hand-edit `.uid`/`.import` files.
- **Communication**: prefer the `GameEvents` bus and the `state_transition_requested`/`setup` pattern over ad-hoc `get_node()`/signal wiring. Context is passed *into* states via `setup(...)`, not read from the parent tree.
- **Assets**: `res://assets/{art,sfx,music,fonts,json}/`. Squads data is `assets/json/squads.json` (array of `{country, players:[{name, skin, role, speed, power}]}` × 11 players in canonical order `[GOALIE, DEFENSE×4, MIDFIELD×3, OFFENSE×3]`, indices map to `Player.SkinColor`/`Player.Role` enums). Flags: `assets/art/ui/flags/flag-<COUNTRY_LOWER>.png`, cached by `FlagHelper`.
- **Physics layers** (see `[layer_names]`): `PitchWalls`, `Player`, `Ball`, `InvisibleWalls`, `ScoringArea`, `GoalKeeperHands`.
- **Input map** (`project.godot` `[input]`): P1 = arrow keys + `[` (pass) / `]` (shoot); P2 = WASD + `0` (pass) / `1` (shoot). Game modes: single-player (P2 empty), versus, co-op (both pick the same country).

## When making changes

- Adding a new **state**: create the `*State` subclass, register `Enum → Class` in the matching `*StateFactory._init()`, add the enum value to the owner's `State` enum. If it needs payload, extend the `*StateData` builder.
- Adding a **screen**: add to `SoccerGame.ScreenType` + `ScreenFactory`, extend `Screen`, set its `music` export.
- Adding a **team/country**: add to `squads.json` (11 players, canonical role order), drop a `flag-<name>.png`, and note that `Tournament` slices `countries[1..9]` (index 0 is `"DEFAULT"` placeholder) — keep 8 real teams after DEFAULT for the bracket.
- Keep the viewport/scaling and nearest-filter settings intact — they define the pixel-art look.
