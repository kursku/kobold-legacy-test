# Stellar Legacy — Space Genre Fork Design

**Date:** 2026-03-19
**Base:** Kobold Legacy (kursku/kobold-legacy-main)
**Approach:** Fork + design for extraction later (Approach B)

---

## Overview

Fork Kobold Legacy into a space exploration/colonization Discord bot game called **Stellar Legacy**. The engine (`kobold.py`) is preserved mostly intact. A genre system is introduced so all game content lives in swappable genre folders — making a future Wild West or other genre game a matter of creating a new `genres/<name>/` folder rather than forking the codebase again.

---

## Tone & Setting

**Space Adventure + Military/Conquest.** Exploration is exciting, alien ruins hold treasure, crew bonds matter — but there are real military stakes. Think *Firefly* meets *Starship Troopers*. Not gritty survival, not cosmic horror.

---

## Core Character Unit

**Colonists/Crew members** — mixed specialists (engineers, pilots, scientists, fighters). Everyone starts as a colonist and specializes via the existing skill system. **Soldiers/Marines** are a specialization role, not a separate unit type. New colonists arrive via supply ship arrivals and births — mapping directly to the existing kobold reproduction system with minimal code change.

---

## Factions

| Role | Fantasy equivalent | Space faction |
|---|---|---|
| Constant threat | Goblins | **Alien Hive/Swarm** — insectoid, no negotiation, expand and raid |
| Complex faction | Humans | **Megacorporations** — funded your colony, neutral until interests conflict |
| Player groups | Other tribes | Other player colonies |

Additional factions (pirates, rogue AI, alien civilizations) can be added later via `genre.json` — no engine changes required.

---

## Ability System

Two tracks replacing the magic/spell system:

| Track | Resource | Who uses it | Maps from |
|---|---|---|---|
| **Psionics** | Psi Points (MP renamed) | Rare specialist crew | Spells / Arcana skill |
| **Tech abilities** | Power Cells (activated items) | All crew | Item system — zero new code |

The existing MP pool and arcana skill are renamed via `genre.json`. Tech abilities use the existing item activation system. No new code paths required.

---

## Project Structure

```
kobold-legacy/
├── kobold.py                    ← engine, mostly unchanged
├── genres/
│   ├── active                   ← plain text file: "space" or "fantasy"
│   ├── space/
│   │   ├── genre.json           ← labels, faction names, resource names
│   │   ├── items.json           ← EVA suits, plasma weapons, gadgets, meds
│   │   ├── creatures.json       ← alien fauna, hive soldiers, mercenaries
│   │   ├── spells.json          ← 30 psionic + 22 tech abilities
│   │   ├── traits.json          ← Void-hardened, Hive-touched, Corp-trained, Psionic
│   │   ├── research.json        ← tech tree: life support → FTL → terraforming
│   │   ├── buildings.json       ← colony modules: docking bay, med bay, barracks
│   │   ├── landmarks.json       ← alien ruins, derelicts, mining sites, anomalies
│   │   ├── crafts.json          ← fabrication: gear, meds, gadgets, ship components
│   │   ├── skills.json          ← Piloting, Engineering, Xenobiology, Psionics, Tactics
│   │   ├── liquids.json         ← stims, coolants, medical compounds
│   │   ├── dungeons.json        ← alien hive, derelict vessel, corporate black site
│   │   └── tribe_names.txt      ← colony station names
│   └── fantasy/
│       └── ...                  ← original data files, untouched
├── docs/
│   └── superpowers/specs/
│       └── 2026-03-19-space-genre-fork-design.md
└── .env
```

### genre.json schema

```json
{
  "name": "Stellar Legacy",
  "character": "colonist",
  "characters": "colonists",
  "faction": "colony",
  "factions": "colonies",
  "home_base": "station",
  "threat_faction": "Hive",
  "complex_faction": "Megacorp",
  "ability_resource": "Psi Points",
  "tech_resource": "Power Cells",
  "character_name_prefix": "space",
  "role_names": {
    "red":    "Vanguard",
    "yellow": "Specialist",
    "green":  "Medic",
    "blue":   "Engineer",
    "purple": "Psionic",
    "white":  "Pilot",
    "black":  "Infiltrator",
    "grey":   "Colonist"
  }
}
```

`ability_resource` and `tech_resource` are the display strings shown in Discord messages (e.g. "12 Psi Points"). `role_names` replaces the hardcoded `ROLENAMES` dict in `kobold.py` (e.g. "Mudscale", "Bloodscale") with genre-appropriate titles. `character_name_prefix` tells the name generator which phoneme table to use — a `space` table will be added alongside the existing `kobold` table in Phase 2.

---

## Engine Changes (kobold.py)

### Bug fixes (Phase 1)
- Line 9726: remove extra closing paren `main_loop()))`
- Line 9727: `clive.start('token')` → `clive.start(TOKEN)`
- Line 9730: remove unreachable `clive.run(TOKEN)`

### Genre system (Phase 2)
- On startup: read `genres/active` → set active genre name. If file is missing or genre folder does not exist, fall back to `"fantasy"` and log a warning.
- Replace all `data/` path references → `genres/{active_genre}/`, including the standalone `open('data/tribe_names.txt')` call in `tribe_name()` (line 67)
- Load `genre.json` into a global `GENRE` config dict
- Replace ~50–100 hardcoded `"kobold"` / `"tribe"` strings in Discord messages with `GENRE["character"]` / `GENRE["faction"]`
- Replace hardcoded `ROLENAMES` dict with `GENRE["role_names"]`
- Replace `kobold_name()` phoneme logic with a genre-aware name generator driven by `GENRE["character_name_prefix"]`

### What does NOT change
- Combat logic
- Crafting system
- Research system
- Save/load (shelve)
- Discord event handling
- All 154 command implementations
- Breeding/reproduction system (becomes colonist births + supply ship arrivals via content, not code)

---

## Space Content Plan

Content volume mirrors the original. Each file is replaced wholesale — the JSON schema stays identical so the engine needs no changes to consume it.

| File | Scope |
|---|---|
| `skills.json` | 26 skills — Piloting, Engineering, Xenobiology, Psionics, Tactics, Medicine, etc. |
| `traits.json` | 91+ traits — adapted to space: Void-hardened, Hive-touched, Corp-trained, Psionic Sensitive |
| `creatures.json` | 49 creatures — alien fauna, hive warriors, hive queens, mercenaries, rogue drones |
| `items.json` | 150+ items — EVA suits, plasma rifles, med-injectors, tech gadgets, ship components |
| `crafts.json` | 210 recipes — fabrication of gear, meds, gadgets, structural components |
| `research.json` | 52 topics — life support → weapons → xenobiology → FTL → terraforming |
| `buildings.json` | Colony modules — docking bay, med bay, barracks, reactor, greenhouse, comms array |
| `landmarks.json` | 30+ POIs — alien ruins, derelict ships, mining sites, anomaly zones, corporate outposts |
| `dungeons.json` | 3 types — alien hive, derelict vessel, corporate black site |
| `spells.json` | 52 abilities — 30 psionic (telekinesis, mind control, precognition) + 22 tech (hacking, drones, EMP) |
| `liquids.json` | Stims, coolants, medical compounds, alien fluids |
| `tribe_names.txt` | Colony station names |

---

## Phase Plan

| Phase | Branch | Goal | Estimate |
|---|---|---|---|
| 1 — Fix & Run | `fix/startup-bug` | Fix startup bug, verify bot connects, merge | Days |
| 2 — Genre System | `feat/genre-system` | Move data to `genres/fantasy/`, add genre loading, verify fantasy works | 1–2 weeks |
| 3a — Space Skeleton | `feat/space-genre-skeleton` | Stub `genres/space/` files — bot boots and plays a basic session, merge independently | 1 week |
| 3b — Space Content | `feat/space-genre-content` | Fill all 12 content files, one commit per file | 3–4 weeks |
| 4 — OSS Release | `chore/oss-release` | README, contribution guide, TASKS.md, license review, tag v1.0 | 1 week |

**Pacing rule:** every session ends with a merged commit. No half-finished work left in the working tree.

**Bootable space game:** end of Phase 3a — the bot runs with stub content, enough to validate the genre system. **Fully playable space game:** end of Phase 3b, when all 12 content files are complete.

**Total estimate:** 6–9 weeks at part-time pace.

---

## Future Extraction Path

When a second genre (Wild West, etc.) is ready:
1. Create `genres/wildwest/` with its own content files and `genre.json`
2. Change `genres/active` to `wildwest`
3. The engine runs the new genre with zero code changes

Full framework extraction (splitting `kobold.py` into modules) is deferred until after shipping and test coverage is established.
