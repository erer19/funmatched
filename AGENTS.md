# AGENTS.md — AI Development Rules & Architecture Mandates

> This file defines persistent rules for Google Antigravity (`AGENTS.md` / `GEMINI.md`). Read and enforce these constraints on every task.

## 1. Project Stack & Hardware Constraints

- **Engine & Language:** Godot 4.7, GDScript (static typing required on all functions and properties: `var x: int = 0`, `func foo() -> void:`).
- **Target Hardware:** **4 GB VRAM budget**.
  - Prefer procedural shaders (`assets/shaders/mini_wash.gdshader`) over multi-map 4K PBR textures for miniatures.
  - Cap environment/board textures at **1K–2K (512px–2048px)**.
  - Use Godot `Tween` and `AnimationPlayer` for tabletop miniature motion (parabolic arcs, combat lunges, hit shake) instead of heavy skeletal rigs.
- **MCP Integrations:**
  - `godot-ai` (`godot-mcp`): Inspect scene trees, attach scripts, run scenes, capture screenshots, and read console errors.
  - `blender-mcp`: Run headless/automated `bpy` scripts for mesh decimation, base attachment, cavity/AO baking, and `.glb` export.
- **Version Control:** Git + Git LFS (for `*.glb`, `*.png`, `*.webp`, `*.ogg`, `*.wav`).

---

## 2. Directory Structure

```text
fun_matched/
├── AGENTS.md                 # Canonical AI agent rules (mirrored in GEMINI.md)
├── GEMINI.md                 # Entry pointer/mirror for Gemini agent rules
├── ASSETS.md                 # Mandatory registry of all 3D, 2D, and audio assets + licenses
├── project.godot             # Godot 4.7 project configuration
├── docs/
│   ├── GDD.md                # Game Design Document & MVP ruleset
│   ├── ARCHITECTURE.md       # GameCore, EventBus, and Presentation architecture
│   ├── RULES_MODEL.md        # JSON schemas (Map, Heroes, Cards, Commands, Replay Log)
│   ├── ASSET_PIPELINE.md     # 2D -> 3D -> Blender MCP -> Godot Mini-Wash shader workflow
│   └── MILESTONES.md         # Ordered milestones and acceptance gates
├── data/                     # Pure declarative JSON data (NO scene/visual references)
│   ├── battlefields/         # Map graphs (spaces, 3D coords, adjacent_ids, zones)
│   ├── heroes/               # Hero & sidekick stats, starting health, passive rules
│   └── cards/                # 30-card deck definitions and phase triggers
├── scripts/
│   ├── core/                 # HEADLESS GAME CORE (Pure RefCounted GDScript, NO Node3D/Control)
│   │   ├── game_state.gd
│   │   ├── turn_manager.gd
│   │   ├── movement_rules.gd
│   │   ├── deck_manager.gd
│   │   ├── combat_resolver.gd
│   │   └── effect_resolver.gd
│   ├── ai/                   # Bot controllers (Random, Greedy Heuristic, MCTS)
│   ├── presentation/         # 3D board, miniature tokens, Tween animations, camera
│   └── ui/                   # Programmatic Control-based cards, hand fan, HUD, health dials
├── scenes/
│   ├── main_menu.tscn
│   ├── game.tscn
│   ├── battlefield.tscn
│   └── ui/
├── assets/
│   ├── models/               # .glb meshes (Git LFS)
│   ├── shaders/              # .gdshader files (mini_wash.gdshader, space_highlight.gdshader)
│   ├── textures/             # 1K-2K textures and clean card illustrations (Git LFS)
│   └── audio/                # Music and SFX (Git LFS)
└── tests/
    ├── run_tests.gd          # Headless CLI test runner (extends SceneTree)
    ├── unit/                 # Unit tests for movement, combat, triggers, fatigue, victory
    └── replays/              # Deterministic seed + command log regression tests
```

---

## 3. Non-Negotiable Architectural Rules

1. **Strict Headless Core Separation:**
   - Files in `scripts/core/` MUST inherit from `RefCounted` (or be static utility classes).
   - `scripts/core/` MUST NEVER reference `Node`, `Node3D`, `Control`, `SceneTree`, `Tween`, `AnimationPlayer`, or visual resources.
   - All state mutations happen exclusively via `TurnManager.execute_command(state, command) -> Dictionary`, which mutates `GameState` and returns an ordered `Array[Dictionary]` of visual events.
2. **Deterministic Randomness:**
   - Never call global `randi()`, `randf()`, `randomize()`, or `Array.shuffle()` inside `scripts/core/`.
   - All randomness (deck shuffles, random discards) MUST use `GameState.rng` (`RandomNumberGenerator` initialized with `GameState.seed`).
   - Given the same initial `seed` and sequence of commands, `GameState` must produce the exact same state hash every time.
3. **Data-Driven Content (No Hardcoded Cards):**
   - Never write card-specific `if card.id == "..."` branches inside `combat_resolver.gd` or `turn_manager.gd`.
   - All cards are defined declaratively in `data/cards/*.json` using phase triggers (`IMMEDIATELY`, `DURING_COMBAT`, `AFTER_COMBAT`, `ON_SCHEME`) and atomic effect operations resolved by `effect_resolver.gd`.
4. **Programmatic Card UI:**
   - Card illustrations in `assets/textures/cards/` must be clean 2D art **without** baked borders, numbers, or text.
   - Card frames, attack/defense/scheme icons, values, boost numbers, and ability text are rendered dynamically in Godot `Control` nodes from JSON data.
5. **Minimal Code (YAGNI):**
   - Write the smallest, clearest GDScript that solves the current milestone task. Do not add speculative abstractions, wrapper classes, or unused configuration flags.

---

## 4. Workflow, Testing & Verification Mandates

- **One Task = One Atomic Commit:** Keep changes scoped to a single milestone task.
- **No Unverified Logic Changes:** Every change to `scripts/core/` or `data/` MUST be accompanied by a headless unit test in `tests/` and verified via:
  ```bash
  godot --headless -s tests/run_tests.gd
  ```
- **Visual Verification for Presentation/UI:** Every change to `scenes/`, `scripts/presentation/`, or `scripts/ui/` must be launched via `godot-mcp` (or headless smoke-check) to confirm zero script/shader errors.
- **No Invented Game Rules:** Follow the combat timing and movement rules in `docs/GDD.md` and `docs/RULES_MODEL.md` (specifically: **Defender resolves triggers before Attacker** in all combat phases; friendly fighters can pass through friendly fighters but never through enemies; empty deck draw deals 2 fatigue damage per friendly fighter).
- **Asset Provenance Tracking:** Every 3D model, texture, card illustration, or audio file added to `assets/` must be logged immediately in `ASSETS.md` with its generator tool, date, tier, and license.

