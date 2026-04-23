# Commander Mission System — CK3 1.19 (Crane) Research Pass

**Scope:** Read-only validation against vanilla install `D:\Steam\steamapps\common\Crusader Kings III\game` plus public wiki/forum sources. No mod implementation code.

**Path note:** Vanilla hooks live under `common\on_action\` (singular), not `on_actions\`.

---

## Better Automated Armies mod — Stage 1 notes (post-research)

### V1 — `random_character_war` when attacker-side in multiple wars

**Context:** Stage 1 role assignment runs from `on_army_monthly` on the army owner (`root`). To recover a `scope:war` for the martial-fallback counter, the mod uses `any_character_war` / `random_character_war` with **`is_attacker = root`** inside the war scope (vanilla pattern: `war_on_actions.txt` `on_join_war_as_secondary` uses `scope:war = { is_attacker = root }`; jesec `Triggers_list.md` — `is_attacker` on war scope). That includes **primary attacker, co-attacker, and attacker-side allies**; defender-side wars do not match.

**Limitation:** If `root` is an **attacker-side participant in more than one simultaneous war**, `random_character_war` still returns **one** of those wars (selection is not player-facing and should be treated as arbitrary for V1).

**Impact:** **No script failure.** Commander discovery uses `every_army` / `army_commander` on the character; it is not `scope:war`-filtered. The ambiguous `scope:war` mainly affects where the temporary fallback counter variable (`baa_fb_i`) is stored and any future logic that mistakenly assumes `scope:war` is “the” war for all purposes.

**Triage:** Reports of “wrong war” while in several attacker-side wars at once are this known limitation until V2 selects a war deterministically (e.g. war start date, army-linked war, or explicit player scope).

### Stage 2 — shared `on_army_monthly` with Stage 1

**Plan:** Keep a single `baa_on_army_monthly_role_assignment` under `on_army_monthly`. Stage 2 appends tick / strategy effects **after** `baa_assign_all_roles_effect` in that block.

**Ordering:** Assignment executes first in the chain and applies `baa_role_assigned` where appropriate. Stage 2 tick logic should require stable role flags (and its own state) so the first post-raise pulse assigns roles and subsequent pulses (~30 days, staggered by army ID) handle ticks without racing assignment.

---

## High-Risk Items First (Q4 / Q9)

### Recurring war tick (Q4) — summary

**Finding:** There is **no** documented global `on_monthly_pulse` / `on_yearly_pulse` pair in `_on_actions.info`. Interval hooks from code are **yearly** (plus character-birthday-based pulses at quarter/year/multi-year granularity) and **per-army** `on_army_monthly` (~30 days, staggered by army ID). `on_war_started` exists and is war-scoped via standard war scopes.

**Evidence:**

- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\_on_actions.info` — lists `yearly_global_pulse`, `on_yearly_playable`, `quarterly_playable_pulse`, `three_year_playable_pulse`, `five_year_playable_pulse`, `random_yearly_playable_pulse`, etc.; **no monthly global pulse**.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\army_on_actions.txt` — `on_army_monthly` header: *"called for armies every 30 days; exact date dependent on army ID"*, `root` = army owner, `scope:army` = army.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\yearly_on_actions.txt` — `quarterly_playable_pulse` (lines ~2480–2506): *"Called from code once a quarter for playable (count+) characters"* with `scope:quarter` 1–4 relative to `on_yearly_playable`, not calendar quarters.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\war_on_actions.txt` — `on_war_started` at line ~312 with comment on CB-equivalent scopes.

**Risk:** **Yellow / Red for a strict “every 15–30 game days, war-only, all commanders” tick.** Closest native fits are `on_army_monthly` (30-day, per army, staggered) or chaining off `quarterly_playable_pulse` / `on_yearly_playable` with manual day counters on characters (extra script). `yearly_global_pulse` is Jan 1 only (`yearly_on_actions.txt` header).

**V2/V3 note:** Any pulse tied to **playable character** birthdays fires **per human player** for their characters, not “host only.” Staggered `on_army_monthly` can duplicate work across many armies unless carefully deduplicated.

### Multiplayer safety (Q9) — summary

**Finding:** Official modding intro states multiplayer requires **identical mods/load order** for all players; it does **not** document per-hook host vs client execution. `character_event` / `on_action` behavior in MP is not spelled out on the pages retrieved; **desync risk** remains standard for any scripted effect that changes state differently if evaluation order or RNG diverges.

**Evidence:**

- `https://productionwiki-ck3.paradoxwikis.com/Modding` — *"In multiplayer, all players must use the same mods in the same load order."*
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\_on_actions.info` — `on_action` `effect` blocks run **concurrently** with events fired from the same `on_action`, not before; scopes/locals in `effect` do not carry into those events (sync/design implication for chained logic).

**Risk:** **Yellow.** No authoritative wiki line was found stating `on_war_started` or `character_event` fires only on host; treat **all state-changing script** as needing deterministic, replicated-friendly patterns (same inputs → same effects on all clients).

**V2/V3 note:** Persisting mission state on **characters** (flags/variables) aligns with typical CK3 MP save replication; avoid logic that depends on hidden local scope only.

---

## Q1 — Army Movement Scripting

**Finding:** Script exposes **teleport-style** repositioning and commander assignment on **army** scope, not UI-equivalent “march here / attack this stack” orders. Documented effects include:


| Effect              | Scope → target   | Role                                                                                                                      |
| ------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `set_army_location` | army → province  | Teleports army to province; **not while in combat**; if hostiles present, **can initiate combat** as attacker (per docs). |
| `assign_commander`  | army → character | Sets army commander.                                                                                                      |
| `remove_commander`  | army             | Clears commander.                                                                                                         |
| `spawn_army`        | character        | Creates/spawns armies (large block in docs).                                                                              |
| `add_loot`          | army             | Raiding loot.                                                                                                             |
| Supply / depletion  | army             | `add_supply`, `subtract_supply`, `clear_supply`, `refill_supply`, `deplete_army_by_percentage` (EP3 localization file).   |


No separate documented effect named like `attack_army` / `order_move_to_province` was found in the effects list excerpt for army-move orders; **province targeting** for movement is `set_army_location = scope:province` (or fixed `province:id` as in vanilla).

**Evidence:**

- Community mirror of script docs: [jesec/ck3-modding-wiki `Effects_list.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md)` — rows for `set_army_location`, `assign_commander`, `remove_commander`, `spawn_army`, `every_army`, `random_army`, `ordered_army`.
- `D:\Steam\steamapps\common\Crusader Kings III\game\events\game_rule_events.txt` — `game_rule.1151` / `game_rule.1152` use `ordered_army` + `assign_commander` + `set_army_location = province:1588` / `province:1509` (Stamford Bridge / Hastings setup).
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\effect_localization\00_character_effects.txt` — `spawn_army` localization block (~1523).
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\effect_localization\07_ep3_effects.txt` — army supply / `deplete_army_by_percentage`.

**Risk:** `set_army_location` is a **hard teleport**, not pathing; may feel/exploit differently than player orders. Combat restriction means no mid-battle reposition. Forcing combat via hostile province is less controlled than explicit “attack army X.”

**V2/V3 note:** Teleporting **AI ally** armies from script may be politically/MP sensitive; confirm acceptability and OOS impact before V2.

---

## Q2 — Iterating Individual Armies

**Finding:** From **character** scope, `every_army`, `random_army`, and `ordered_army` iterate that character’s armies; inner scope is **army**. `any_army` appears in triggers. Additional iterators: `every_army_in_location`, `random_army_in_location` (e.g. `yearly_on_actions.txt`, `army_on_actions.txt`). Commander is accessed from army (vanilla uses `army_commander` / `knight_army` patterns in `army_on_actions.txt`).

**Evidence:**

- [jesec/ck3-modding-wiki `Effects_list.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md)` — `every_army`, `ordered_army`, `random_army` (character → army).
- `D:\Steam\steamapps\common\Crusader Kings III\game\events\game_rule_events.txt` — nested `ordered_army` under `scope:norway` / `scope:england`.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\script_values\03_dlc_fp2_script_values.txt` — `every_war_attacker` containing nested `every_army`.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\army_on_actions.txt` — `every_army_in_location`, references to `army_commander`, `knight_army`.

**Risk:** Multiple armies on one owner require explicit **limits/sorting** (`ordered_army` + `order_by`) to avoid applying the same instruction to the wrong stack.

**V2/V3 note:** Iteration is per **owner character**; allied stacks belong to other characters — must iterate allies separately (see Q8).

---

## Q3 — Persistent State Storage

**Finding:** **Character flags and character variables** are the standard persistence vehicles for per-commander mission roles and match the stated design (“authoritative on character”). Army-scoped data is weaker for “commander identity” because commanders are characters and armies are transient/rebuilt; vanilla also uses **army variables** for short-lived pulses (example: cooldown on `scope:army` in `on_army_monthly`).

**Evidence:**

- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\army_on_actions.txt` — `set_variable` on `scope:army` for `commander_death_events_pulse_cooldown` (illustrates army variables for pulse gating, not long-term design data).
- Event modding overview ([jesec Event_modding.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Event_modding.md)) describes `immediate` for setting variables for functional effects — general pattern; save files persist character state objects.

**Risk:** If a commander dies or loses assignment, **cleanup** must clear flags/variables. Army-only state can **orphan** when the army disbands.

**V2/V3 note:** Character-bound state travels with **MP saves** as part of character records; prefer one clear variable namespace per mod to avoid collisions.

---

## Q4 — Recurring War Tick (full)

**Finding:** Reliable **native** schedules:

- **~30 days, army-scoped:** `on_army_monthly` (`army_on_actions.txt`).
- **Quarter-year (per playable ruler, birthday-relative):** `quarterly_playable_pulse` (`yearly_on_actions.txt` + `_on_actions.info`).
- **Yearly (character, birthday):** `on_yearly_playable` and pulses derived from it.
- **Jan 1 global:** `yearly_global_pulse` / `yearly_on_actions.txt` (no root scope per `_on_actions.info`).
- **War start / end hooks:** `on_war_started`, `on_war_won_attacker`, etc. (`war_on_actions.txt`).

Custom **15-day** granularity is **not** listed as a built-in pulse; approximate with **character variables** counting days since last run, updated from an available pulse, or accept `on_army_monthly` cadence.

**Evidence:** Same as High-Risk section + `_on_actions.info` lines 4–21.

**Risk:** **Yellow.** Pulse choice drives performance and desync surface; `on_army_monthly` fires **per army** (could be hot in huge wars).

**V2/V3 note:** `quarterly_playable_pulse` runs per **playable** ruler — good for player commanders; V2 AI allies need scopes that still run for **AI** characters (also playable-tier rulers, not baronies).

---

## Q5 — Automated Army Conflict

**Finding:** Vanilla **script** in the scanned `common` text does **not** expose toggles or checks for the player’s **Automated Armies** feature (no matches for `automated` army controls in `.txt` under `common`). That system appears **engine/UI-driven** (icons under `common\defines\graphic\00_graphics.txt` for order types). Practical **risk** is unresolved in data files: automation may **override or fight** scripted `set_army_location` behavior until tested in-game.

**Evidence:**

- `D:\Steam\steamapps\common\Crusader Kings III\game\common\defines\graphic\00_graphics.txt` — `army_order_*` icon paths (orders exist at UI level).
- Forum discussion (player feature, not script API): [CK3 army automation thread](https://forum.paradoxplaza.com/forum/threads/ck3-army-automation-auto-command-auto-move-armies.1533915/).

**Risk:** **Yellow** until playtested on 1.19 with automation on/off. Mod may need to **document** “disable automation for intended behavior” if conflicts appear.

**V2/V3 note:** Other human players’ automation settings multiply uncertainty in MP.

---

## Q6 — Trait Internal Names

**Finding (1.19 install `00_traits.txt`):**


| User-facing / intent         | Internal key (`has_trait = …`) | Notes                                                                                                                                                                                           |
| ---------------------------- | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Aggressive (commander track) | `aggressive_attacker`          | `category = commander` at `D:\Steam\steamapps\common\Crusader Kings III\game\common\traits\00_traits.txt` ~12115. No separate `aggressive = {` personality trait found in the same file search. |
| Brave                        | `brave`                        | ~3259                                                                                                                                                                                           |
| Patient                      | `patient`                      | ~2804                                                                                                                                                                                           |
| Strategist                   | `strategist`                   | ~1448                                                                                                                                                                                           |
| Humble                       | `humble`                       | ~3006                                                                                                                                                                                           |
| Schemer                      | `schemer`                      | ~1679                                                                                                                                                                                           |
| Deceitful                    | `deceitful`                    | ~3082                                                                                                                                                                                           |


**Evidence:** `D:\Steam\steamapps\common\Crusader Kings III\game\common\traits\00_traits.txt` at line anchors above.

**Risk:** If designers meant generic “aggressive personality,” CK3 uses `**aggressive_attacker`** for the commander trait; confirm design intent.

**1.18 → 1.19 rename check:** **Not verified** in this pass (would need patch notes or a 1.18 file tree). Current Crane install uses the keys above.

**V2/V3 note:** Commander traits remain character-scoped; safe for MP replication.

---

## Q7 — Commander Council / War Screen UI

**Finding:** War UX is defined in `**.gui`** under `game\gui\` (e.g. `window_war_overview.gui`, `window_war_results.gui`). Mods typically **override or extend** these windows or add **decisions / character interactions** for player choices. No evidence was collected in this pass for a **supported plug-in API** specifically for “inject one button into war overview without copying vanilla window” — practical approach is **GUI modding** (higher maintenance on vanilla updates) or **off-war UI** (decisions, sidebar widgets).

**Evidence:**

- `D:\Steam\steamapps\common\Crusader Kings III\game\gui\window_war_overview.gui` — `name = "war_overview_window"`, `widgetid = "war_overview_window"`.
- Paradox modding intro: `https://productionwiki-ck3.paradoxwikis.com/Modding` — standard guidance to mirror vanilla folder structure when replacing assets.

**Risk:** **Yellow.** GUI overrides **conflict** between mods; patch updates can break selectors.

**V2/V3 note:** UI-only changes are client-local; underlying role state must still live in **script-synced** character data.

---

## Q8 — Allied Commander Iteration

**Finding:**

- `**every_war_ally`** — from **character** scope, iterates **direct war allies** (character → character). Documented in [Effects_list.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md).
- `**every_war_participant` / `every_war_attacker` / `every_war_defender` / `every_war_enemy`** — used from **war** scope in scripted effects (e.g. `00_war_effects.txt`, `war_on_actions.txt`, `10_dlc_tgp_natural_disaster_scripted_effects.txt`).
- `**every_character_war`** — iterate wars a character is in, then open war scope (`00_war_effects.txt`, `00_war.txt` interactions, etc.).
- **Combat-only commander iteration:** `every_side_commander` / `random_side_commander` (combat side scope) per Effects_list.

There is **no** single built-in iterator named `every_war_commander`; workable pattern: `every_war_participant` or `every_war_ally` → `every_army` (where applicable) → read `army_commander`.

**Evidence:**

- `D:\Steam\steamapps\common\Crusader Kings III\game\common\scripted_effects\10_dlc_tgp_natural_disaster_scripted_effects.txt` — `every_war_ally`, `every_war_enemy` blocks (~716–747).
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\war_on_actions.txt` — `every_war_participant` usage.
- [jesec Effects_list.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md) — `every_war_ally` definition line.

**Risk:** Allies without raised armies yield **no armies** to iterate; commanders might be idle courtiers.

**V2/V3 note:** Applying **orders** to AI ally armies hits **AI autonomy / player war contribution** expectations; technically scoped, politically different.

---

## Q9 — Multiplayer Event Safety (full)

**Finding:** Public documentation stresses **mod parity** across clients, not per-hook replication tables. `_on_actions.info` documents **ordering**: `effect` in an `on_action` does **not** run before linked events and does **not** share locals with them — relevant when splitting logic across `effect` vs `events`. No wiki page in this pass stated `on_war_started` is host-only.

**Evidence:**

- `https://productionwiki-ck3.paradoxwikis.com/Modding` — multiplayer same mods/load order.
- `D:\Steam\steamapps\common\Crusader Kings III\game\common\on_action\_on_actions.info` — `effect` / event concurrency note (~lines 95–97 in the inspected file).

**Risk:** **Yellow.** Desync comes from **non-deterministic** flows (unordered lists, unsynchronized random, effects that assume single-player UI). Hidden `character_event` still mutates global state visible to others.

**V2/V3 note:** Test MP early with **two human clients** and logging when touching war + army effects.

---

## Q10 — Prior Art Survey

**Finding:** Public discussions and Workshop listings (titles/claims only; **not** source-audited in this pass) around **AI army behavior**, automation complaints, and **army composition / recruitment** mods. Examples surfaced by web search (Steam Workshop IDs for CK3 app **1158310** where stated):


| Name (as listed)                   | Workshop ID  | Relevance / claimed patterns                                                                                                                |
| ---------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| CK3 Enhanced AI                    | `3650626004` | Broad AI war behavior (forum/marketing text: supply, targeting priorities). Likely AI script / weights, not necessarily player army orders. |
| Better AI Army Builder             | `3617312541` | Army composition / raising (workshop title).                                                                                                |
| Vassals To Arms                    | `2535753142` | Vassal military call-ups; may include war AI subcomponents per search snippets.                                                             |
| AI Recruitment & Army Compositions | `2789853654` | Army templates / recruitment (per title).                                                                                                   |


**Forum threads (automation / player pain):**

- `https://forum.paradoxplaza.com/forum/threads/ck3-army-automation-auto-command-auto-move-armies.1533915/`
- `https://forum.paradoxplaza.com/forum/threads/make-automated-armies-option-for-controling-the-army-lead-by-our-character.1732370/latest`
- `https://forum.paradoxplaza.com/forum/threads/automate-army.1418468/`
- User-supplied (not fully fetched here): `https://forum.paradoxplaza.com/forum/threads/modding-automated-army.latest`

**Vanilla “prior art” for scripted army repositioning:**

- `game_rule.1151` / `game_rule.1152` in `D:\Steam\steamapps\common\Crusader Kings III\game\events\game_rule_events.txt` — `ordered_army` + `set_army_location` pattern.

**Risk:** Workshop pages were not systematically downloaded for CK3; some search hits can point at **wrong games** — verify app ID and file contents before citing implementation.

**V2/V3 note:** Study **AI mod packs** for weight-based war decisions; study **vanilla war on_actions** for safe hook points.

---

## External sources consulted (fetch status)


| URL                                                                               | Result                            |
| --------------------------------------------------------------------------------- | --------------------------------- |
| `https://ck3.paradoxwikis.com/Army_modding`                                       | **404**                           |
| `https://ck3.paradoxwikis.com/On_actions`                                         | **404**                           |
| `https://ck3.paradoxwikis.com/Character_flags`                                    | **Challenge page** (no content)   |
| `https://productionwiki-ck3.paradoxwikis.com/Modding`                             | **OK** — general modding guidance |
| `https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md`  | **OK** (mirror of script docs)    |
| `https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Event_modding.md` | **OK**                            |


For authoritative effect names, Paradox recommends generating local `effects.log` via in-game `script_docs` (`Modding` page, lines ~62–64).

---

## Feasibility Summary


| Pillar                                               | Rating     | One-line justification                                                                                                                                                                                     |
| ---------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Role assignment** (traits/stats + manual override) | **Green**  | Traits and triggers are well-defined; **interactions/decisions** are proven patterns; commander assignment exists (`assign_commander`).                                                                    |
| **Strategy tick** (15–30d war pulse + orders)        | **Yellow** | No exact 15–30d global war pulse; `**on_army_monthly` ~30d** or **quarterly_playable_pulse** + counters are the native fits; `**set_army_location`** is teleport combat starter, not full order emulation. |
| **UI** (war screen controls)                         | **Yellow** | War UI is `**.gui` override`** territory; workable but **fragile** vs vanilla updates and mod conflicts.                                                                                                   |
| **MP safety**                                        | **Yellow** | Docs mandate **mod parity** only; no host-only guarantees found; standard **OOS** discipline applies (`_on_actions.info` concurrency caveat).                                                              |


---

## Closest alternatives when “ideal” is not native

- **Exact 15-day global war clock:** approximate with **variable-backed counters** from `quarterly_playable_pulse`, `on_yearly_playable`, or `on_army_monthly`, accepting skew vs calendar.
- **Player-style attack order on a specific enemy army:** no dedicated effect found; closest is `**set_army_location**` to a **hostile province/stack location** or rely on combat initiation side-effect — needs in-game validation.
- **Dedicated war-screen widget without GUI work:** use **character interaction** or **decision** with war/role filters.

---

## Follow-up: Army Movement Verdict

**VERDICT B — Walking is engine-only (C++)**

Script does **not** expose any effect that issues a player-style march/path order to a province. The only documented army→province repositioning effect is `**set_army_location**`, which the community wiki and vanilla usage both describe as a **teleport**, not pathing.

### Evidence (1.19 install `D:\Steam\steamapps\common\Crusader Kings III\game\`)

1. **Documented effects (external mirror, matches in-game API surface)**
  [jesec/ck3-modding-wiki `Effects_list.md](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md)` lists under army scope: `add_loot`, `assign_commander`, `remove_commander`, `**set_army_location**` (explicit text: *"Teleports the army to the given location"*; parameters: army scope → `scope:province` or fixed province id), plus iterators (`every_army`, `ordered_army`, `random_army`) and `spawn_army` (creates/spawns with a `location` / `origin`; not an order to an existing stack to walk). No separate effect name for “move order,” “path to,” or “attack move” appears in that list.
2. **Vanilla script usage of `set_army_location**`
  A full-text search of the CK3 install’s `game\` tree for `set_army_location` returns **only** `events\game_rule_events.txt` — hidden setup events `**game_rule.1151`** (Stamford Bridge) and **`game_rule.1152`** (Hastings): `ordered_army` → `assign_commander` → `set_army_location = province:1588` / `province:1509`. No other vanilla `.txt` in the install references this effect.
3. `**common\scripted_effects\`**
  No files under `common\scripted_effects\` contain `set_army_location` or any other army-movement order effect in this pass.
4. `**common\on_action\army_on_actions.txt**`
  Army hooks (`on_army_monthly`, `on_army_enter_province`, raid/siege hooks, etc.) apply **trait XP**, **events**, **variables**, **messages** — **no** scripted army repositioning and **no** movement order beyond reacting to armies **already** entering provinces.
5. `**common\game_rules\**`
  No matches for automated army / army movement scripting in `.txt` under `common\game_rules\` (this pass).

### Movement-adjacent effects (why each is not “walk to province”)


| Effect / hook                                                                     | Why it falls short                                                                                                                                   |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `**set_army_location**`                                                           | Instant teleport; wiki documents combat ban while in combat and possible forced attack if hostiles present — not incremental pathing.                |
| `**spawn_army**`                                                                  | Creates (or merges into) armies at a `location`; not an order for an existing army to march tile-by-tile.                                            |
| `**assign_commander` / `remove_commander**`                                       | Commander assignment only; no movement.                                                                                                              |
| `**add_loot**`                                                                    | Raid loot on army scope.                                                                                                                             |
| `**pan_camera_to_province**`                                                      | UI/camera only (`none` → province per wiki).                                                                                                         |
| `**on_army_enter_province**`                                                      | Reacts after movement; does not issue orders.                                                                                                        |
| **Defines `NArmy.MOVEMENT_SPEED` / `MOVEMENT_SPEED_RETREAT` / `MOVEMENT_LOCKED**` | Tunes **how** the engine moves armies once movement exists; not a script API to start a player-style order.                                          |
| `**NRetreat` (shattered retreat)**                                                | Parameterizes retreat **scoring** / province preferences; retreat pathing is still engine behavior, not a mod-callable “give this army a move line.” |


**Ceiling for script:** repositioning an existing army to a province in data-driven script is `**set_army_location**` (teleport). There is **no** second, pathing-based effect found in docs or vanilla text.

### Defines note (`automated_army` / `NMilitary`)

- Grep under `common\defines\` found **no** keys named `automated_army`, `NMilitary`, or similar automation toggles. Army automation surfaces in `**gui\**` (e.g. `window_military.gui`, `window_army.gui`) and **localization** (`ARMY_AUTOMATION_FEATURE_NAME`, `PROVINCE_TOOLTIP_NOT_ALLOWED_TO_MOVE_AUTOMATED_ARMY`) — consistent with a **UI/engine feature**, not a `00_defines.txt` script knob.
- Military movement tuning in data lives mainly under `**NArmy`** in `common\defines\00_defines.txt` (e.g. `MOVEMENT_SPEED = 3`, `MOVEMENT_SPEED_RETREAT = 4.5`, `MOVEMENT_LOCKED = 0.5`, `MOVEMENT_SPEED_BONUS_FRIENDLY_AREA = 0.2`, plus supply/reinforcement constants). `**NFleet**` includes `ATTRITION_AFTER_DAYS = 30` for embarked armies. `**NRetreat**` holds shattered-retreat weights (e.g. `SHATTERED_RETREAT_MAX_PROVINCES = 15`). These parameterize engine behavior once movement is underway; they do not expose scripted “issue move order.”

### VERDICT B: implications for the mod (Workshop / forum context)

Mods cited in Q10 (e.g. Enhanced AI, army composition / recruitment packs) align with **AI weights, templates, and raising logic** — areas that do not require issuing **player-identical map march orders** from script. Where mods “improve” war behavior without teleport hacks, they typically work **within** what the engine already decides (AI goals, composition) or use **teleport-style** setup only where vanilla does (historical scenarios). **Script cannot replicate** the Automated Armies feature’s walk-the-stack UX; that remains **C++/UI** unless Paradox adds a new effect.

### V1 movement UX recommendation (teleport framing)

Treat `**set_army_location`** as a **mission resolve / deployment** action, not a visible march: use **tooltips and brief interface copy** (“Deploy to mission objective,” “Strategic reposition,” “Operational jump”) so players expect an **instant** placement; optionally pair with **toast or map ping** on the target province so the outcome is visible without faking a path the engine cannot run from script. Avoid implying overland travel time unless you simulate it with **narrative delay** (timer + message) **without** moving the counter on the map until the pulse fires.

---

*End of research document.*