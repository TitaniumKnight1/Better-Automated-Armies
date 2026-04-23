# Contributing to Better Automated Armies

Thanks for helping improve this mod. The project is small and **stage-gated**; keeping changes focused makes review and multiplayer safety easier.

## Ground rules

1. **Do not modify the vanilla CK3 install** in pull requests. Reference paths in comments or docs only.
2. **Match the active stage** described in `README.md`. If you are unsure whether an idea belongs in Stage 2 or 3, open an issue first.
3. **Namespace:** new script identifiers, flags, variables, and loc keys should use the **`baa_`** prefix unless you are intentionally integrating a shared dependency (avoid for this repo).
4. **Traits and effects:** confirm trait internal names and effect scopes against vanilla files or the research document before relying on them in triggers.
5. **Localization:** English files under `localization/english/` should remain **UTF-8 with BOM** for CK3 compatibility unless Paradox tooling changes.

## How to contribute

1. **Fork** the repository and create a **feature branch** from `master` (or the default branch).
2. Keep commits **logical and reasonably scoped** — one clear intent per commit message when possible.
3. Open a **pull request** with:
   - What changed and **why** (player-facing or systems-level impact).
   - CK3 version you tested against (e.g. 1.19.x).
   - Any **save-compatibility** or **multiplayer** notes if relevant.
4. Respond to review feedback; maintainers may squash-merge to keep history readable.

## What we are likely to merge quickly

- Bug fixes for Stage 1 role assignment (scopes, lists, flags, loc).
- Documentation improvements, diagrams, or research notes that do not contradict validated vanilla behavior.
- Small refactors that reduce duplication **without** changing gameplay outcomes.

## What needs discussion first

- New hooks that fire outside `on_war_started` for Stage 1-only releases.
- Gameplay that affects **defenders** or **allies** without an agreed design (see research doc / roadmap).
- New dependencies, generated binaries, or bundled tools.

## Conduct

Contributors are expected to follow [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md). Harassment or bad-faith behavior will lead to removal from the project spaces.
