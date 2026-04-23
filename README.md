# Better Automated Armies

A **Crusader Kings III** mod (1.19 / Crane) that improves how **army commanders** are used in war by assigning them **operational roles** once armies are raised (first pass on the **`on_army_monthly`** pulse), so later stages of the mod can drive automation and missions without ad hoc guessing.

**Status:** Early development — **Stage 1** (role scaffolding) is implemented; strategy ticks, movement, deployment UI, and manual overrides are planned for later stages.

---

## What it does today (Stage 1)

After armies exist, on each ruler’s **`on_army_monthly`** pulse (~30 days per army), the mod runs for the **primary war attacker** when they are at war and:

- Collects **unique commanders** from that ruler’s **raised armies** (`every_army` → `army_commander`).
- **Skips** anyone with the character flag **`baa_role_override`** (reserved for a future manual-override path; the flag is never written in Stage 1).
- Assigns a **character variable** `baa_role` on each processed commander using `flag:field`, `flag:siege`, `flag:reserve`, or `flag:raider`, and sets **`baa_role_assigned`**.
- Handles **commander count** edge cases (1 / 2 / 3+) and a **trait/stat priority** pass for 3+, with a **martial-ordered fallback** for anyone who did not match a priority rule.

Trait keys and hooks follow validated patterns documented in [`research/commander_mission_system_research.md`](research/commander_mission_system_research.md).

---

## Roadmap

| Stage | Focus |
|-------|--------|
| **1** (current) | `on_army_monthly` hook (primary attacker), scripted effects/triggers, `baa_role` / flags, localization, GitHub docs |
| **2** | Strategy tick / army movement / deployment events (out of scope for Stage 1) |
| **3** | Character interaction UI and writing **`baa_role_override`** (out of scope for Stage 1) |

Non-goals for the public roadmap as stated in design docs: allied AI iteration (V2 idea), multiplayer-specific logic (V3 idea).

---

## Requirements

- **Crusader Kings III** patch **1.19.x** (matches `supported_version` in `descriptor.mod`).
- A **legal** copy of the game; this mod does not redistribute Paradox assets or game files.

---

## Installation

The Paradox launcher discovers mods from **`.mod` files** in your user **CK3 `mod` folder**, each paired with a **folder** that holds `descriptor.mod` and the rest of the mod. See the [official modding article](https://productionwiki-ck3.paradoxwikis.com/Modding) (“Every mod in it must have a .mod file and a folder”).

### Windows (typical)

1. Open your CK3 mod directory (default):
   - `Documents\Paradox Interactive\Crusader Kings III\mod\`
2. Ensure you have **both** of the following next to each other:
   - **`Better Automated Armies.mod`** — launcher entry (included at the **repository root**; **copy** that file into the `mod\` folder above, not inside the `Better Automated Armies\` subfolder).
   - **`Better Automated Armies\`** — mod content folder whose root contains **`descriptor.mod`**, `common\`, `localization\`, etc.

   The included `.mod` file uses a **portable** path:

   ```txt
   path="mod/Better Automated Armies"
   ```

   That path is relative to your Crusader Kings III user directory (`Documents\Paradox Interactive\Crusader Kings III\`), so it resolves to `mod\Better Automated Armies` under that folder. If you move the mod folder elsewhere, edit `path=` in the `.mod` file to an **absolute** path (forward slashes work), per the wiki.

3. **Development layout:** you can keep the project in another drive or folder and use a **directory junction** so the launcher still sees `mod\Better Automated Armies\` with `descriptor.mod` inside. Example (run in `cmd`):

   ```bat
   mklink /J "%USERPROFILE%\Documents\Paradox Interactive\Crusader Kings III\mod\Better Automated Armies" "C:\VSCode\Better Automated Armies"
   ```

   You still need **`Better Automated Armies.mod`** sitting in `mod\` (sibling to that folder), not only the junction.

   *True symbolic links* on Windows may require Developer Mode or an elevated shell; a **junction** is usually enough for local development.

4. Restart the **Paradox launcher** if it was already open, then enable **Better Automated Armies** under **Mods** and launch a **1.19**-compatible game version.

### If the mod still does not appear

- Confirm the `.mod` filename ends in **`.mod`** and lives in **`...\Crusader Kings III\mod\`**, not inside `Better Automated Armies\`.
- If your **Documents** folder is redirected (e.g. OneDrive), use the actual resolved path for junctions and for troubleshooting.
- As a last resort, Paradox forum threads suggest clearing launcher cache (`launcher-v2.sqlite` / `mods_registry.json` under the same CK3 user folder) after fixing paths — back up first if you care about other local mod metadata.

---

## Repository layout

| Path | Purpose |
|------|---------|
| `Better Automated Armies.mod` | **Launcher stub** — must live in `Documents\...\Crusader Kings III\mod\` next to the mod folder; points at the mod via `path=` |
| `descriptor.mod` | Mod metadata read from the **mod content folder** |
| `common/on_action/baa_war_on_actions.txt` | `on_army_monthly` → role assignment on army owner (primary attacker, at war) |
| `common/scripted_effects/baa_role_assignment_effects.txt` | `baa_assign_all_roles_effect`, `baa_score_and_assign_role_effect` |
| `common/scripted_triggers/baa_role_triggers.txt` | Trait/stat triggers for scoring |
| `localization/english/baa_l_english.yml` | English role names (UTF-8 BOM) |
| `research/commander_mission_system_research.md` | Design research and vanilla references |

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for branching, scope, and pull request expectations.

---

## Security

See [`SECURITY.md`](SECURITY.md) for how to report sensitive issues.

---

## Code of conduct

This project follows the [**Contributor Covenant**](https://www.contributor-covenant.org/) — see [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

---

## License

The mod’s **original** scripts and documentation in this repository are licensed under the **MIT License** — see [`LICENSE`](LICENSE).

**Crusader Kings III** and related trademarks are property of **Paradox Interactive AB**. This project is a fan work and is **not** affiliated with or endorsed by Paradox.

---

## GitHub “About” and metadata (maintainers)

Configure on GitHub under **Repository → ⚙ Settings** (or the gear on the main page **About** sidebar):

| Field | Suggested value |
|-------|------------------|
| **Description** | CK3 1.19 mod: automated commander roles at war start — foundation for smarter armies. |
| **Website** | *Leave blank* or set to this repo URL if you add GitHub Pages later. |
| **Topics** | `crusader-kings-3` `ck3` `ck3-mod` `paradox-interactive` `grand-strategy` `mod` |

Enable **Issues** and **Discussions** (optional) under **Settings → General → Features** if you want community feedback there.

---

## Links

- **Repository:** [github.com/TitaniumKnight1/Better-Automated-Armies](https://github.com/TitaniumKnight1/Better-Automated-Armies)
- **CK3 modding:** [Paradox Wikis — Modding](https://productionwiki-ck3.paradoxwikis.com/Modding)
- **Script reference (community mirror):** [jesec/ck3-modding-wiki — Effects list](https://github.com/jesec/ck3-modding-wiki/blob/HEAD/wiki_pages/Effects_list.md)
