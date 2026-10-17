# GEMINI.md — Antigravity Workspace Instructions

All persistent architectural constraints, directory conventions, hardware budgets (4 GB VRAM), and testing mandates for this repository are maintained in [AGENTS.md](./AGENTS.md).

## Quick Mandates Summary

1. **Read First:** Always follow [AGENTS.md](./AGENTS.md), [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md), and [docs/RULES_MODEL.md](./docs/RULES_MODEL.md).
2. **Headless Core:** `scripts/core/` must remain 100% independent of `Node3D`, `Control`, and `SceneTree`. All gameplay flows through `Command -> GameCore -> Array[Event]`.
3. **Deterministic RNG:** Use `GameState.rng` (seeded `RandomNumberGenerator`) exclusively; never use global `randi()` or `Array.shuffle()`.
4. **Data-Driven Design:** Define maps, heroes, and cards in `data/*.json`. Render card UI programmatically over clean 2D artwork.
5. **Verify Every Step:** Run `godot --headless -s tests/run_tests.gd` after any core change, and record every imported asset in [ASSETS.md](./ASSETS.md).

