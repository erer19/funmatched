# Rules & Data Model (`docs/RULES_MODEL.md`)

All game content (battlefields, heroes, card decks) and match replays are stored as plain, human-readable JSON in `data/` and `tests/replays/`. No gameplay data is hardcoded in GDScript.

---

## 1. Map Graph Schema (`data/battlefields/<map_id>.json`)

Each battlefield is an undirected graph of discrete spaces. Every space has a unique string `id`, 3D tabletop coordinates `(x, y, z)` in meters/Godot units, an array of `zones` (color strings used for ranged line-of-sight and zone effects), and an array of `adjacent_ids`.

### Schema & Example (`data/battlefields/sanctum_courtyard.json` excerpt)

```json
{
  "id": "sanctum_courtyard",
  "name": "Sanctum Courtyard",
  "starting_spaces": {
    "0": "s01",
    "1": "s18"
  },
  "zone_colors": {
    "red": "#D9463E",
    "blue": "#3B82F6",
    "gold": "#EAB308",
    "purple": "#9333EA"
  },
  "spaces": [
    {
      "id": "s01",
      "position": { "x": -4.0, "y": 0.0, "z": 2.5 },
      "zones": ["red"],
      "adjacent_ids": ["s02", "s04"]
    },
    {
      "id": "s02",
      "position": { "x": -2.5, "y": 0.0, "z": 2.5 },
      "zones": ["red", "gold"],
      "adjacent_ids": ["s01", "s03", "s05"]
    }
  ]
}
```

### Graph Invariants (Validated on Load)
1. **Symmetry:** For every space `A`, if `B in A.adjacent_ids`, then `A in B.adjacent_ids`.
2. **No Self-Loops:** `A.id not in A.adjacent_ids`.
3. **Zone Non-Empty:** Every space belongs to at least 1 zone (`zones.size() >= 1`).
4. **Connectedness:** Every space is reachable from starting space `"0"`.

---

## 2. Hero & Sidekick Configuration (`data/heroes/<hero_id>.json`)

Defines a playable roster (Hero + Sidekicks + Passive Ability + Deck Reference).

```json
{
  "id": "arthur",
  "name": "King Arthur",
  "deck_id": "arthur_deck",
  "theme_color": "#C83E37",
  "hero": {
    "id": "arthur",
    "name": "King Arthur",
    "max_hp": 18,
    "move_range": 2,
    "attack_type": "MELEE",
    "model_path": "res://assets/models/heroes/arthur.glb",
    "passive": {
      "id": "noble_sacrifice",
      "trigger": "ON_ATTACK_DECLARE",
      "optional": true,
      "description": "When King Arthur attacks, you may boost that attack by discarding 1 card face-down. If effects on your attack card are cancelled, the boost value is ignored, but the boost card stays discarded."
    }
  },
  "sidekicks": [
    {
      "id": "merlin",
      "name": "Merlin",
      "count": 1,
      "max_hp": 7,
      "move_range": 2,
      "attack_type": "RANGED",
      "model_path": "res://assets/models/tokens/merlin_token.glb"
    }
  ]
}
```

### Second MVP Hero Example (`data/heroes/medusa.json`)

```json
{
  "id": "medusa",
  "name": "Medusa",
  "deck_id": "medusa_deck",
  "theme_color": "#2E9E5B",
  "hero": {
    "id": "medusa",
    "name": "Medusa",
    "max_hp": 16,
    "move_range": 3,
    "attack_type": "RANGED",
    "model_path": "res://assets/models/heroes/medusa.glb",
    "passive": {
      "id": "gorgon_gaze",
      "trigger": "START_OF_TURN",
      "optional": true,
      "effects": [
        {
          "op": "DEAL_DAMAGE",
          "target": "CHOOSE_ENEMY_IN_HERO_ZONE",
          "amount": 1
        }
      ],
      "description": "At the start of your turn, you may deal 1 damage to an opposing fighter in Medusa's zone."
    }
  },
  "sidekicks": [
    {
      "id": "harpy",
      "name": "Harpy",
      "count": 3,
      "max_hp": 1,
      "move_range": 3,
      "attack_type": "MELEE",
      "model_path": "res://assets/models/tokens/harpy_token.glb"
    }
  ]
}
```

---

## 3. Card Deck & Trigger Schema (`data/cards/<deck_id>.json`)

A deck file defines the 30 cards for a hero (`sum(card.copies) == 30`). During match setup, each copy is instantiated with a deterministic instance ID (`"<card_id>#1"`, `"<card_id>#2"`, etc.).

```json
{
  "deck_id": "arthur_deck",
  "owner_hero_id": "arthur",
  "cards": [
    {
      "id": "feint",
      "title": "Feint",
      "type": "VERSATILE",
      "character": "ANY",
      "value": 2,
      "boost": 1,
      "copies": 3,
      "art_path": "res://assets/textures/cards/feint.webp",
      "text": "IMMEDIATELY: Cancel all effects on your opponent's card.",
      "triggers": {
        "IMMEDIATELY": [
          {
            "op": "CANCEL_ENEMY_EFFECTS",
            "condition": "ALWAYS"
          }
        ]
      }
    },
    {
      "id": "excalibur",
      "title": "Excalibur",
      "type": "ATTACK",
      "character": "arthur",
      "value": 6,
      "boost": 3,
      "copies": 1,
      "art_path": "res://assets/textures/cards/excalibur.webp",
      "text": "",
      "triggers": {}
    },
    {
      "id": "regroup",
      "title": "Regroup",
      "type": "DEFENSE",
      "character": "ANY",
      "value": 1,
      "boost": 2,
      "copies": 3,
      "art_path": "res://assets/textures/cards/regroup.webp",
      "text": "AFTER COMBAT: Draw 1 card. If you won the combat, draw 2 cards instead.",
      "triggers": {
        "AFTER_COMBAT": [
          {
            "op": "DRAW_CARDS",
            "target": "SELF_PLAYER",
            "amount": 1,
            "condition": "IF_LOST_COMBAT"
          },
          {
            "op": "DRAW_CARDS",
            "target": "SELF_PLAYER",
            "amount": 2,
            "condition": "IF_WON_COMBAT"
          }
        ]
      }
    },
    {
      "id": "prophecy",
      "title": "Prophecy",
      "type": "SCHEME",
      "character": "merlin",
      "value": 0,
      "boost": 2,
      "copies": 2,
      "art_path": "res://assets/textures/cards/prophecy.webp",
      "text": "Draw 2 cards. Move each of your fighters up to 2 spaces.",
      "triggers": {
        "ON_SCHEME": [
          {
            "op": "DRAW_CARDS",
            "target": "SELF_PLAYER",
            "amount": 2,
            "condition": "ALWAYS"
          },
          {
            "op": "MOVE_FIGHTER",
            "target": "ALL_FRIENDLY_FIGHTERS",
            "max_steps": 2,
            "condition": "ALWAYS"
          }
        ]
      }
    }
  ]
}
```

### Trigger Phase Dictionary Keys
- `"IMMEDIATELY"` — Resolved first after reveal (Defender, then Attacker).
- `"DURING_COMBAT"` — Resolved before damage calculation (Defender, then Attacker).
- `"AFTER_COMBAT"` — Resolved after combat damage is applied, provided neither Hero has `0 HP` (Defender, then Attacker).
- `"ON_SCHEME"` — Resolved immediately when played as a `SCHEME` action.

---

## 4. Command & Event Protocol

### 4.1. Commands (Input to `TurnManager.execute_command`)
Every command is a plain `Dictionary` with `"type"` and `"player_id"`:

| Command `type` | Allowed `phase` | Payload Fields |
| :--- | :--- | :--- |
| `"UseStartTurnPassive"` | `"START_OF_TURN"` | `{ "skip": bool, "target_fighter_id": String }` |
| `"ActionManeuver"` | `"TURN_ACTION"` | `{ "boost_card_instance_id": String }` (`""` for no boost) |
| `"MoveFighter"` | `"MANEUVER_MOVE"` | `{ "fighter_id": String, "target_space_id": String }` |
| `"EndManeuver"` | `"MANEUVER_MOVE"` | `{}` |
| `"ActionScheme"` | `"TURN_ACTION"` | `{ "fighter_id": String, "card_instance_id": String }` |
| `"ActionAttack"` | `"TURN_ACTION"` | `{ "attacker_id": String, "defender_id": String, "card_instance_id": String, "boost_card_instance_id": String }` |
| `"SelectDefenseCard"` | `"COMBAT_DEFENSE_SELECT"` | `{ "card_instance_id": String }` (`""` to take undefended hit) |
| `"ResolveChoice"` | `"PENDING_CHOICE"` | `{ "choice_value": Variant }` (space ID, fighter ID, card ID, or `"SKIP"`) |
| `"DiscardExcessCard"` | `"END_TURN_DISCARD"` | `{ "card_instance_id": String }` |

### 4.2. Domain Events (Output from `TurnManager.execute_command`)
Every event is a plain `Dictionary` with `"type"`:

- `{"type": "TurnStarted", "player_id": 0, "turn_number": 1}`
- `{"type": "CardDrawn", "player_id": 0, "card_instance_id": "feint#1"}`
- `{"type": "FatigueDamageDealt", "player_id": 0, "fighter_id": "arthur", "amount": 2, "new_hp": 16}`
- `{"type": "ManeuverBoosted", "player_id": 0, "card_instance_id": "regroup#1", "boost_value": 2}`
- `{"type": "UnitMoved", "fighter_id": "arthur", "from_space": "s01", "to_space": "s03", "path": ["s01", "s02", "s03"]}`
- `{"type": "SchemePlayed", "player_id": 0, "fighter_id": "merlin", "card_instance_id": "prophecy#1"}`
- `{"type": "CombatDeclared", "attacker_id": "arthur", "defender_id": "medusa", "is_boosted": true}`
- `{"type": "CardsRevealed", "attack_card": "excalibur#1", "defense_card": "feint#2", "boost_card": "regroup#2"}`
- `{"type": "EffectTriggered", "phase": "IMMEDIATELY", "source_fighter_id": "medusa", "op": "CANCEL_ENEMY_EFFECTS"}`
- `{"type": "CombatDamageDealt", "attacker_id": "arthur", "defender_id": "medusa", "attack_total": 6, "defense_total": 2, "damage": 4, "new_hp": 12}`
- `{"type": "FighterDefeated", "fighter_id": "harpy_1"}`
- `{"type": "GameWon", "winner_player_id": 0, "reason": "ENEMY_HERO_DEFEATED"}`

---

## 5. Deterministic Replay Log Format (`tests/replays/<replay_name>.json`)

Because `GameState` uses a seeded `RandomNumberGenerator` and processes only discrete `Command` dictionaries, any match can be saved and re-verified in `< 10 ms` using a compact JSON log:

```json
{
  "format_version": 1,
  "seed": 987654321,
  "map_id": "sanctum_courtyard",
  "players": [
    { "player_id": 0, "hero_id": "arthur" },
    { "player_id": 1, "hero_id": "medusa" }
  ],
  "commands": [
    { "player_id": 0, "type": "ActionManeuver", "boost_card_instance_id": "" },
    { "player_id": 0, "type": "MoveFighter", "fighter_id": "arthur", "target_space_id": "s03" },
    { "player_id": 0, "type": "MoveFighter", "fighter_id": "merlin", "target_space_id": "s02" },
    { "player_id": 0, "type": "EndManeuver" }
  ],
  "expected_end_state": {
    "turn_number": 1,
    "active_player_id": 0,
    "actions_remaining": 1,
    "fighter_positions": {
      "arthur": "s03",
      "merlin": "s02"
    },
    "fighter_hp": {
      "arthur": 18,
      "merlin": 7,
      "medusa": 16
    }
  }
}
```

