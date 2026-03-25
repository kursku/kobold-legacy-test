# Stellar Legacy — Phase 1 & 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the broken bot startup and introduce the genre folder system so all game content lives under `genres/{active}/` and the engine reads labels (character names, faction names, resource names) from `genre.json`.

**Architecture:** `refresh_data()` is the single function that loads all JSON — patching it to use a genre path covers 12 of 14 `data/` references. The remaining two (`tribe_name()` line 67 and the `ROLENAMES` dict line 27) are patched individually. A new `GENRE` global dict is loaded from `genre.json` on startup and used everywhere a hardcoded `"kobold"` or `"tribe"` label appears in Discord output.

**Tech Stack:** Python 3, discord.py, python-dotenv, shelve (save system), JSON

---

## Scope Note

This plan covers **Phase 1 (Fix & Run)** and **Phase 2 (Genre System)** from the spec. Phase 3a (space skeleton content) and Phase 3b (full space content) each need their own plan — they are content design work, not engineering.

---

## File Map

| File | Action | What changes |
|---|---|---|
| `kobold.py` | Modify | Fix 3 startup bugs; add `GENRE` global; update `refresh_data()`, `tribe_name()`, `kobold_name()`, `ROLENAMES` |
| `genres/active` | Create | Plain text file containing `"fantasy"` |
| `genres/fantasy/` | Create | Copy of all original `data/` files |
| `genres/fantasy/genre.json` | Create | Labels matching original kobold/tribe terminology |
| `data/` | Keep | Untouched — engine will stop reading from here after Phase 2 |

No new modules. No test infrastructure exists in this codebase — verification is manual bot startup + command checks. New genre-loading helper functions are simple enough to verify via startup logs.

---

## Task 1: Fix Startup Bug

**Branch:** `fix/startup-bug`
**Files:** Modify `kobold.py` lines 9724–9730

- [ ] **Step 1: Read the broken section**

  Open `kobold.py` and locate lines 9724–9730:
  ```python
  async def main():
      async with clive:
          clive.loop.create_task(main_loop()))   # ← extra paren
          await clive.start('token')             # ← literal string, not TOKEN var

  asyncio.run(main())
  clive.run(TOKEN)                               # ← unreachable dead code
  ```

- [ ] **Step 2: Apply the three fixes**

  Replace those lines with:
  ```python
  async def main():
      async with clive:
          clive.loop.create_task(main_loop())
          await clive.start(TOKEN)

  asyncio.run(main())
  ```

- [ ] **Step 3: Verify syntax is valid**

  Run: `python -c "import ast; ast.parse(open('kobold.py').read()); print('OK')`
  Expected: `OK` with no errors.

- [ ] **Step 4: Create a `.env` file if it doesn't exist**

  The bot requires `DISCORD_TOKEN` and `DISCORD_GUILD`. Create `.env`:
  ```
  DISCORD_TOKEN=your_token_here
  DISCORD_GUILD=your_guild_name_here
  ```
  This file is already in `.gitignore` — do not commit it.

- [ ] **Step 5: Create branch then commit**

  Create the branch *before* any edits are staged to avoid touching main directly:
  ```bash
  git checkout -b fix/startup-bug
  git add kobold.py
  git commit -m "fix(startup): remove extra paren, use TOKEN var, drop unreachable clive.run"
  ```

- [ ] **Step 6: Open PR and merge to main**

  ```bash
  git push -u origin fix/startup-bug
  gh pr create --title "fix(startup): fix bot startup sequence" --body "Fixes extra closing paren on main_loop(), replaces literal 'token' string with TOKEN variable, and removes unreachable clive.run(TOKEN) after asyncio.run()."
  gh pr merge --merge
  git checkout main && git pull
  ```

---

## Task 2: Create Genre Folder Structure

**Branch:** `feat/genre-system`
**Files:** Create `genres/`, `genres/active`, `genres/fantasy/`

- [ ] **Step 1: Create branch**

  ```bash
  git checkout -b feat/genre-system
  ```

- [ ] **Step 2: Create the genres directory and active file**

  ```bash
  mkdir -p genres/fantasy
  echo "fantasy" > genres/active
  ```

- [ ] **Step 3: Copy all data files into `genres/fantasy/`**

  ```bash
  cp data/items.json genres/fantasy/
  cp data/creatures.json genres/fantasy/
  cp data/spells.json genres/fantasy/
  cp data/traits.json genres/fantasy/
  cp data/research.json genres/fantasy/
  cp data/buildings.json genres/fantasy/
  cp data/landmarks.json genres/fantasy/
  cp data/crafts.json genres/fantasy/
  cp data/skills.json genres/fantasy/
  cp data/liquids.json genres/fantasy/
  cp data/dungeons.json genres/fantasy/
  cp data/commands.json genres/fantasy/
  cp data/tribe_names.txt genres/fantasy/
  ```

- [ ] **Step 4: Create `genres/fantasy/genre.json`**

  Create the file with this exact content:
  ```json
  {
    "name": "Kobold Legacy",
    "character": "kobold",
    "characters": "kobolds",
    "faction": "tribe",
    "factions": "tribes",
    "home_base": "lair",
    "threat_faction": "Goblins",
    "complex_faction": "Humans",
    "ability_resource": "Mana",
    "tech_resource": "Mana",
    "character_name_prefix": "kobold",
    "role_names": {
      "brown":  "Mudscale",
      "red":    "Bloodscale",
      "yellow": "Goldscale",
      "green":  "Jadescale",
      "blue":   "Silkscale",
      "white":  "Marblescale",
      "black":  "Coalscale",
      "orange": "Copperscale",
      "purple": "Violetscale",
      "silver": "Silverscale"
    }
  }
  ```

- [ ] **Step 5: Commit the folder structure**

  ```bash
  git add genres/
  git commit -m "feat(genre): create genre folder structure with fantasy pack"
  ```

---

## Task 3: Wire Genre Loading into the Engine

**Branch:** `feat/genre-system` (continued)
**Files:** Modify `kobold.py`

This task adds a `GENRE` global and patches `refresh_data()` and `tribe_name()` to read from the active genre folder.

- [ ] **Step 1: Add the `GENRE` global and `load_genre()` function**

  Find the block of globals at the top of `kobold.py` (around line 15, after the imports). Add after the existing globals:

  ```python
  GENRE = {}

  def load_genre():
    global GENRE
    try:
      with open('genres/active') as f:
        active = f.read().strip()
    except Exception:
      active = 'fantasy'
      console_print('WARNING: genres/active missing or unreadable, defaulting to fantasy')
    genre_path = f'genres/{active}'
    if not os.path.isdir(genre_path):
      console_print(f'WARNING: Genre folder {genre_path} not found, defaulting to fantasy')
      genre_path = 'genres/fantasy'
      active = 'fantasy'
    genre_file = f'{genre_path}/genre.json'
    loaded = get_json(genre_file)
    if loaded is None:
      console_print(f'WARNING: Could not load {genre_file}, using empty genre config')
      loaded = {}
    GENRE.update(loaded)
    GENRE['_path'] = genre_path
    console_print(f'Genre loaded: {GENRE.get("name", active)} from {genre_path}')
  ```

  Note: `get_json()` is defined at line 94. `load_genre()` must be placed after it.

- [ ] **Step 2: Patch `refresh_data()` to use the genre path**

  Find `refresh_data()` at line 9137. Replace every `'data/'` prefix with `GENRE['_path'] + '/'`:

  ```python
  def refresh_data():
    global item_data,item_cats,research_data,building_data,craft_data,cmd_data,creature_data,spell_data,liquid_data,landmark_data,trait_data,skill_data,dungeon_data
    base = GENRE.get('_path', 'genres/fantasy') + '/'
    item_data=get_json(base+'items.json')
    # ... rest of existing code, all data/ prefixes → base
    research_data=get_json(base+'research.json')
    building_data=get_json(base+'buildings.json')
    craft_data=get_json(base+'crafts.json')
    cmd_data=get_json(base+'commands.json')
    creature_data=get_json(base+'creatures.json')
    spell_data=get_json(base+'spells.json')
    liquid_data=get_json(base+'liquids.json')
    landmark_data=get_json(base+'landmarks.json')
    trait_data=get_json(base+'traits.json')
    skill_data=get_json(base+'skills.json')
    dungeon_data=get_json(base+'dungeons.json')
  ```

  Leave all post-load processing (the loops checking for "Default" entries, warning prints) untouched.

- [ ] **Step 3: Patch `tribe_name()` to use the genre path**

  Find `tribe_name()` at line 66. Replace the hardcoded path:

  ```python
  def tribe_name():
    path = GENRE.get('_path', 'genres/fantasy') + '/tribe_names.txt'
    try: f = open(path)
    except:
      console_print('ERROR: Cannot find tribe name list at ' + path)
      return "Erroneously-named Tribe"
    # rest unchanged
  ```

- [ ] **Step 4: Call `load_genre()` before `refresh_data()` in startup**

  Find `load_game()` at line 9015:
  ```python
  def load_game(path='klsave'):
    refresh_data()
    ...
  ```

  Change to:
  ```python
  def load_game(path='klsave'):
    load_genre()
    refresh_data()
    ...
  ```

  The standalone `refresh_data()` at line 8339 is inside `cmd_refresh` (the admin `!refresh` command), not a second startup call. Leave it unchanged — genre is already loaded at startup via `load_game()`, and re-running `load_genre()` on `!refresh` is unnecessary.

- [ ] **Step 5: Verify syntax**

  ```bash
  python -c "import ast; ast.parse(open('kobold.py').read()); print('OK')"
  ```
  Expected: `OK`

- [ ] **Step 6: Smoke test — bot should start and log the genre**

  Run the bot: `python kobold.py`
  Expected in console output:
  ```
  Genre loaded: Kobold Legacy from genres/fantasy
  ```
  The bot should connect to Discord exactly as before.

- [ ] **Step 7: Commit**

  ```bash
  git add kobold.py
  git commit -m "feat(genre): wire genre loading into engine startup and data refresh"
  ```

---

## Task 4: Replace `ROLENAMES` and `kobold_name()`

**Branch:** `feat/genre-system` (continued)
**Files:** Modify `kobold.py` lines 27 and 37–64

- [ ] **Step 1: Replace `ROLENAMES` dict with genre config**

  Find line 27:
  ```python
  ROLENAMES={"brown":"Mudscale","red":"Bloodscale", ... }
  ```

  Replace with:
  ```python
  def get_role_name(color):
    return GENRE.get('role_names', {}).get(color, color.capitalize())
  ```

  Then search for all uses of `ROLENAMES` in `kobold.py`:
  ```bash
  grep -n "ROLENAMES" kobold.py
  ```
  Replace each `ROLENAMES[x]` or `ROLENAMES.get(x, ...)` call with `get_role_name(x)`.

- [ ] **Step 2: Make `kobold_name()` genre-aware**

  The existing function generates kobold-style phoneme names. Rename it to `_kobold_name()` (private) and add a dispatcher:

  ```python
  def _kobold_name():
    # existing kobold_name() body — unchanged
    ...

  def _space_name():
    # Space-flavored names: short, hard consonants, sci-fi feel
    prefixes = ['Ax', 'Cor', 'Dex', 'Fen', 'Kor', 'Lyx', 'Mav', 'Nex', 'Orv', 'Pax', 'Ryn', 'Sev', 'Tav', 'Vex', 'Zyn']
    suffixes = ['an', 'ar', 'en', 'er', 'in', 'is', 'on', 'or', 'us', 'yn']
    return (random.choice(prefixes) + random.choice(suffixes)).capitalize()

  def kobold_name():
    prefix = GENRE.get('character_name_prefix', 'kobold')
    if prefix == 'space':
      return _space_name()
    return _kobold_name()
  ```

  The function name `kobold_name()` stays the same so all callers work unchanged.

- [ ] **Step 3: Verify syntax**

  ```bash
  python -c "import ast; ast.parse(open('kobold.py').read()); print('OK')"
  ```

- [ ] **Step 4: Commit**

  ```bash
  git add kobold.py
  git commit -m "feat(genre): replace ROLENAMES dict and make kobold_name() genre-aware"
  ```

---

## Task 5: Replace Hardcoded Label Strings in Discord Output

**Branch:** `feat/genre-system` (continued)
**Files:** Modify `kobold.py`

- [ ] **Step 1: Find all hardcoded "kobold"/"tribe" strings in Discord messages**

  ```bash
  grep -n '"kobold\|"tribe\|"Kobold\|"Tribe' kobold.py | grep -v "^\s*#" | wc -l
  ```
  This gives the count to track progress. Expect 50–100 matches.

- [ ] **Step 2: Replace label strings systematically**

  Work through the grep results. For each Discord-facing message string, replace:
  - `"kobold"` → `GENRE.get("character", "kobold")`
  - `"Kobold"` → `GENRE.get("character", "kobold").capitalize()`
  - `"kobolds"` → `GENRE.get("characters", "kobolds")`
  - `"tribe"` → `GENRE.get("faction", "tribe")`
  - `"Tribe"` → `GENRE.get("faction", "tribe").capitalize()`
  - `"tribes"` → `GENRE.get("factions", "tribes")`

  **Do NOT replace:**
  - Variable names (`kobold_list`, `tribe_name`, etc.)
  - Comments
  - Internal logic strings (e.g., save keys)
  - Class/function names

  Focus only on strings that appear in `await chan.send(...)` or similar Discord output calls.

- [ ] **Step 3: Verify syntax after replacements**

  ```bash
  python -c "import ast; ast.parse(open('kobold.py').read()); print('OK')"
  ```

- [ ] **Step 4: Smoke test**

  Start the bot and run a basic command (e.g., `!status`). Confirm Discord output still reads correctly for the fantasy genre (all labels should appear as "kobold"/"tribe" since that's what `genre.json` returns).

- [ ] **Step 5: Commit in batches as you go**

  After every ~25 replacements, run the syntax check and commit to limit blast radius:
  ```bash
  python -c "import ast; ast.parse(open('kobold.py').read()); print('OK')"
  git add kobold.py
  git commit -m "feat(genre): replace kobold/tribe label strings in Discord output (batch N)"
  ```
  Repeat until all matches are replaced. Use a final commit to mark completion:
  ```bash
  git commit -m "feat(genre): complete label string replacement — all Discord output uses GENRE config"
  ```

---

## Task 6: Final Verification and Merge

**Branch:** `feat/genre-system`

- [ ] **Step 1: Confirm `data/` has no remaining references in the engine**

  ```bash
  grep -n "data/" kobold.py
  ```
  Expected: zero results (or only inside comments).

- [ ] **Step 2: Confirm fantasy game still works end-to-end**

  With `genres/active` set to `"fantasy"`, start the bot and verify:
  - Bot connects and logs `Genre loaded: Kobold Legacy from genres/fantasy`
  - A player can join and create a kobold
  - Basic commands work (`!status`, `!look`, `!inventory`)
  - Tribe name generation works
  - Character name generation produces kobold-style names

- [ ] **Step 3: Test genre switching works**

  Change `genres/active` to a nonexistent genre name (e.g., `"invalid"`).
  Restart bot — expected console output:
  ```
  WARNING: Genre folder genres/invalid not found, defaulting to fantasy
  Genre loaded: Kobold Legacy from genres/fantasy
  ```
  Bot should start normally.

- [ ] **Step 4: Open PR and merge**

  ```bash
  git push -u origin feat/genre-system
  gh pr create --title "feat(genre): add genre folder system (Phase 2)" --body "$(cat <<'EOF'
  ## Summary
  - Introduces genres/ folder with genres/active switcher
  - Moves all data files into genres/fantasy/ (original content preserved)
  - Adds genre.json config driving character/faction labels, role names, and name generator
  - Patches refresh_data(), tribe_name(), ROLENAMES, and kobold_name() to use genre config
  - Replaces ~50-100 hardcoded kobold/tribe strings in Discord output with GENRE config values
  - Fantasy game behavior is unchanged

  ## Test plan
  - [ ] Bot starts and logs correct genre name
  - [ ] Fantasy game plays normally end-to-end
  - [ ] Invalid genre name falls back to fantasy with warning
  EOF
  )"
  gh pr merge --merge
  git checkout main && git pull
  ```

---

## What's Next

After this plan is complete, create a new plan for **Phase 3a: Space Genre Skeleton** which will:
- Create `genres/space/` with stub versions of all 12 content files
- Create `genres/space/genre.json` with the Stellar Legacy config from the spec
- Switch `genres/active` to `"space"` and verify the bot boots

Phase 3b (full space content — 12 JSON files) and Phase 4 (OSS release) each need their own plans.
