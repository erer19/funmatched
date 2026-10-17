# Game Design Document (GDD) — Unmatched-Inspired 3D Tactical Tabletop

## 1. High Concept & Vision

An asymmetric 3D turn-based tactical card-and-miniature duel game built in **Godot 4.7**. The game brings the tactile feel of a physical board game to life: sculpted plastic hero miniatures with procedural ink-wash shading, circular sidekick tokens, 3D health dials, and an orbital tabletop camera, paired with a crisp, programmatic 2D card interface.

---

## 2. Strict MVP Scope

### Included in MVP
- **Game Modes:**
  1. **Local Hot-Seat (1v1 Pass & Play)** on a single PC.
  2. **Single-Player vs. Local AI** (progressing from Random to Greedy Heuristic and Deterministic MCTS).
- **Battlefield (1 Map):**
  - `sanctum_courtyard` — A 1v1 arena graph with **18 discrete spaces**, bidirectional adjacency links, **4 colored zones** (`red`, `blue`, `gold`, `purple`), and designated starting spaces `1` and `2`.
- **Playable Rosters (2 Public-Domain / Archetype Heroes + Sidekicks):**
  1. **Arthur & Merlin (Melee Hero + 1 Ranged Sidekick)**
     - **Arthur (Hero):** `18 HP`, `MELEE`, Move `2`.
       - *Passive (`Noble Sacrifice`):* When Arthur attacks, he may discard 1 card face-down to boost the attack value (`+boost` value). If opponent cancels effects (e.g., *Feint*), the boost card remains discarded without adding its boost value.
     - **Merlin (Sidekick):** `7 HP`, `RANGED`, Move `2`.
     - **Deck:** 30 cards (`attack`, `defense`, `versatile`, `scheme`).
  2. **Medusa & The Harpies (Ranged Hero + 3 Melee Sidekick Tokens)**
     - **Medusa (Hero):** `16 HP`, `RANGED`, Move `3`.
       - *Passive (`Gorgon Gaze`):* At the start of Medusa's turn, she may deal `1` direct damage to an opposing fighter in her zone.
     - **Harpies (3x Sidekick Tokens):** `1 HP` each (defeated on any `>= 1` damage), `MELEE`, Move `3`.
     - **Deck:** 30 cards (`attack`, `defense`, `versatile`, `scheme`).

### Explicit Non-Goals for MVP (Cut / Deferred)
- Online multiplayer / matchmaking (deferred until post-MVP; architecture remains command-driven so it can be added later without changing `scripts/core/`).
- Free-roam 3D physics movement or skeletal character animations.
- 3–4 player free-for-all or 2v2 team modes.
- Custom deck building (each Hero uses a fixed, balanced 30-card signature deck).

---

## 3. Core Game Components

1. **Board Spaces & Colored Zones:**
   - The board is a graph of circular spaces connected by path lines.
   - Each space belongs to **1 to 3 colored zones** (indicated by segmented colored rings around the space).
   - **Occupancy Rule:** A space can hold at most **1 fighter** at the end of any movement or effect.
2. **Fighters (Heroes & Sidekicks):**
   - Each player controls **1 Hero** (miniature) and **1 or more Sidekicks** (plastic disc tokens).
   - Sidekicks with `1 HP` are defeated immediately upon taking 1 or more damage; multi-HP sidekicks track HP on a secondary health dial.
   - When a fighter reaches `0 HP`, their token/miniature is immediately removed from the board.
3. **30-Card Signature Deck:**
   - Every card specifies:
     - **Card Type:** `ATTACK` (red), `DEFENSE` (blue), `VERSATILE` (purple — usable for either Attack or Defense), or `SCHEME` (yellow lightning bolt).
     - **Character Restriction:** `HERO`, `SIDEKICK`, `ANY`, or a specific fighter ID (e.g., `"arthur"`, `"merlin"`).
     - **Combat Value:** Base numeric strength (`value`, required on `ATTACK`, `DEFENSE`, `VERSATILE`).
     - **Boost Value:** Numeric boost circle (`boost`, `0`–`4`) used when discarding the card to boost movement or special effects.
     - **Phase Triggers:** `IMMEDIATELY`, `DURING_COMBAT`, `AFTER_COMBAT` (for combat cards) or `ON_SCHEME` (for scheme cards).

---

## 4. Exact Match & Turn Loop

### 4.1. Match Setup
1. Initialize `GameState` with a deterministic 64-bit integer `seed`.
2. Player 1 places their Hero on Starting Space `1`, then places their Sidekick(s) on any empty spaces in the **same colored zone** as Space `1` (or adjacent if the zone is full).
3. Player 2 places their Hero on Starting Space `2`, then places their Sidekick(s) in the same colored zone as Space `2`.
4. Each player shuffles their 30-card deck using `GameState.rng` and draws **5 starting cards**.
5. Player 1 starts Turn 1 with `actions_remaining = 2`.

### 4.2. Start of Turn Phase
1. Resolve any `START_OF_TURN` hero passive abilities (e.g., Medusa's *Gorgon Gaze*: optional 1 damage to an enemy fighter sharing at least one zone with Medusa).
2. Set `actions_remaining = 2`.

### 4.3. Action Phase (Mandatory 2 Actions)
On their turn, the active player **MUST** take **2 actions** (cannot pass early). They may choose any combination of the 3 action types below (including the same action twice):

#### Action A: Maneuver (Draw 1 -> Optional Boost -> Move Fighters)
1. **Mandatory Draw:** Draw the top card of your deck.
   - **Exhaustion / Fatigue Rule:** If your deck has `0` cards when you must draw (from a Maneuver or any mandatory card effect), **do not reshuffle your discard pile**. Instead, every one of your living fighters (Hero + living Sidekicks) immediately takes **2 damage**.
2. **Optional Movement Boost:** You may discard 1 card from your hand to add its `boost` value to your fighters' base `move_range` for this Maneuver action.
3. **Move Friendly Fighters:** Move each of your living fighters one by one, from `0` up to `(base_move_range + boost_bonus)` spaces along connected edges:
   - Fighters **MAY** move through spaces occupied by **friendly** fighters.
   - Fighters **CANNOT** move through spaces occupied by **enemy** fighters.
   - Fighters **CANNOT** end their movement on an occupied space.

#### Action B: Scheme (Play Lightning Card)
1. Select a `SCHEME` card from your hand and declare which living friendly fighter matching the card's `character` restriction is executing it.
2. Reveal the card, resolve its `ON_SCHEME` effect sequence, and place the card face-up in your discard pile.

#### Action C: Attack (Declare -> Defend -> Reveal -> 3 Trigger Phases)
1. **Declare Attack:**
   - Choose a living friendly fighter (`attacker_id`) and a valid `ATTACK` or `VERSATILE` card from hand whose `character` restriction matches the attacker.
   - Choose a valid living enemy fighter (`defender_id`) within range:
     - **`MELEE` Attacker:** Target space must be in `attacker_space.adjacent_ids`.
     - **`RANGED` Attacker:** Target space must either be in `attacker_space.adjacent_ids` **OR** share at least one colored zone (`zone_a ∩ zone_b != ∅`) with `attacker_space`.
   - Place the attack card face-down (plus any optional pre-reveal passive boost, such as Arthur's *Noble Sacrifice*).
2. **Defender Response:**
   - The defending player may select 1 `DEFENSE` or `VERSATILE` card from their hand matching the defending fighter's `character` restriction and place it face-down, **or** decline to play a defense card (effective defense value `0`).
3. **Reveal & Combat Resolution Sequence:**
   - Both cards are revealed simultaneously.
   - **Tie-Breaker Rule (CRITICAL):** Whenever both Attacker and Defender have triggers in the same phase (`IMMEDIATELY`, `DURING_COMBAT`, `AFTER_COMBAT`), **the Defender's trigger ALWAYS resolves first, followed by the Attacker's trigger.**
   - **Step 1 — `IMMEDIATELY` Phase:**
     - Resolve Defender's `IMMEDIATELY` effect (if not cancelled).
     - Resolve Attacker's `IMMEDIATELY` effect (if not cancelled).
     - *Note:* If a card cancels "all effects on your opponent's card" (e.g., *Feint*), the opponent's `DURING_COMBAT` and `AFTER_COMBAT` effects are marked cancelled, though the card's printed `value` still applies.
   - **Step 2 — `DURING_COMBAT` Phase:**
     - Resolve Defender's `DURING_COMBAT` effect (value overrides, value modifiers, Blind Boost).
     - Resolve Attacker's `DURING_COMBAT` effect.
   - **Step 3 — Combat Damage Step:**
     - Compute `combat_damage = max(0, final_attack_value - final_defense_value)`.
     - Reduce Defender's HP by `combat_damage`.
     - **Who Won the Combat?**
       - If the Defender took **at least 1 combat damage** (`combat_damage >= 1`), the **Attacker won the combat**.
       - If the Defender took **0 combat damage** (`combat_damage == 0`), the **Defender won the combat**.
   - **Step 4 — `AFTER_COMBAT` Phase:**
     - Check if either Hero reached `0 HP` (if so, the match ends immediately!).
     - Otherwise, resolve Defender's `AFTER_COMBAT` effect, then Attacker's `AFTER_COMBAT` effect.
     - *Defeated Fighter Rule:* Even if a Sidekick was defeated during Step 3, their card's `AFTER_COMBAT` effect still resolves unless the effect requires moving or healing that defeated fighter.
   - **Step 5 — Cleanup:**
     - Move played attack, boost, and defense cards to their respective owners' discard piles.

### 4.4. End of Turn Phase
1. Once `actions_remaining == 0`, check the active player's hand size.
2. **7-Card Hand Limit:** If `hand.size() > 7`, the active player must choose and discard cards until `hand.size() == 7`.
3. Pass active turn to the opponent.

---

## 5. Victory Conditions

- **Instant Hero Defeat:** The moment a player's **Hero** drops to `0 HP` (whether from combat damage, scheme damage, after-combat damage, or exhaustion/fatigue damage), the game ends immediately and the opposing player wins.
- **Simultaneous Hero Defeat Edge Case:** If an effect causes both Heroes to reach `0 HP` simultaneously, the **active turn player** wins.

