# 19 — Pyfa feature inventory (scope baseline for the Rust mainline)

**Status: SKELETON (11:15 CST, 2026-10-03). Item tables follow in later commits.**

**Rule (from the user, 2026-10-03):** the product must cover **everything Pyfa does, and may do more, never less**. The
early reference engine and the bench corpus are incomplete, so they do **not** define scope. This inventory does.

- **Reference:** Pyfa `v2.69.0` (github.com/pyfa-org/Pyfa, local clone `/workspace/exct-eve/ref/pyfa`, master
  `1d9f72b` = v2.69.0 + 1 commit). It is used **read-only as a reference**: no Pyfa code is copied (Pyfa is GPL-3.0;
  the mainline is LGPL-3.0-or-later).
- **Machine-readable copy:** [`19-pyfa-feature-inventory.yaml`](19-pyfa-feature-inventory.yaml) is the source of truth.
  The tables below are rendered from it.
- **Status columns:**
  - **F**: the mainline base, eve-dogma-lab `variant-f-perf` (F's engine/CLI/WASM; graph layer from `graphs-g4`).
  - **MCP**: EX-CT/eve-fit-mcp `main`.
  - **WEB**: EX-CT/eve-fit-web `main`.

  Each column is `have` / `partial` / `missing` / `n/a`, with evidence: suite case ids, test names, file paths, or a
  probe. A `missing` entry says what was checked.
- **Evidence suites:**
  - eve-dogma-bench `v1.9.0` (331 cases)
  - `cap-suite` `d80cc38`
  - `mutated-suite` `2ac7c00`
  - `formats-suite` `7c716e7`
  - `graphs-round2` `84f7c2e`
  - `pending-1.10` fuzz repros `e_fz_*` (`b4deddd`)

## Areas and ID scheme

IDs are stable: `<AREA>-<SUB>-<NNN>`. New items get the next free number, and removed items keep their ID, marked
`dropped`.

| prefix | area | Pyfa source |
|---|---|---|
| ENG-CORE | modifier engine: operators, stacking, skills, attribute model, overrides | `eos/modifiedAttributeDict.py`, `eos/saveddata/fit.py`, `eos/effects.py` |
| ENG-MOD | modules: states, charges/ammo, overheat, spool-up, reload, mutated/abyssal | `eos/saveddata/module.py`, `mutatedMixin.py`, `mutator.py` |
| ENG-SHIP | ships: T3D modes, T3C subsystems, structures/citadels, security status | `eos/saveddata/ship.py`, `mode.py`, `citadel.py` |
| ENG-DRN / ENG-FTR | drones / fighters and fighter abilities | `eos/saveddata/drone.py`, `fighter.py`, `fighterAbility.py` |
| ENG-IMP | implants, implant sets, boosters and side effects | `eos/saveddata/implant.py`, `booster.py`, `boosterSideEffect.py` |
| ENG-CAP | capacitor and capacitor simulation | `eos/capSim.py`, `fit.py` (`simulateCap`, `addDrain`) |
| ENG-DEF | HP, resists, RAH, EHP, tank (raw/sustained/effective) | `fit.py`, `eos/effects.py` (adaptive armor hardener) |
| ENG-OFF | DPS/volley, weapons, doomsdays, breachers, mining, application | `module.py` (`getVolleyParameters` / `getDps`), `fit.py` |
| ENG-NAV / ENG-TGT | navigation and warp; targeting, sensors, EWAR strength, lock times | `fit.py`, `ship.py` |
| ENG-PROJ | projected modules, drones, fighters, fits; remote reps, cap, neut/nos, EWAR | `fit.py` (projected), `eos/effects.py` |
| ENG-FLT | command bursts / fleet boosts, command fits, warfare buffs | `fit.py` (`commandFits`, `addCommandBonus`), `eos/effects.py` |
| ENG-ENV | system effects: wormholes, abyssal weather, incursion, metaliminal, FW, system security | `fit.py` (`systemEffect`), `eos/effects.py` |
| ENG-VAL | fitting restrictions and validation | `module.py` (`fits`, `isValidState`, `canHaveState`), `fit.py` |
| GRF | graphs: every type, axis and option | `graphs/data/*`, `graphs/gui/*` |
| CHR | characters and skills (All 5/0, custom, ESI import, prerequisites) | `service/character.py`, `eos/saveddata/character.py` |
| PRC | prices and price sources | `service/price.py`, `service/marketSources/*` |
| FMT | import/export formats, clipboard | `service/port/*` |
| ESI | SSO, ESI fittings, character skills | `service/esi.py`, `service/esiAccess.py`, `gui/esiFittings.py`, `gui/ssoLogin.py` |
| DB | persistence: saved fits, fitting tree, booster/command/projected links, backups | `eos/db/*`, `service/fit.py` |
| MKT | market and ship browser, search, item stats views | `gui/marketBrowser.py`, `gui/shipBrowser.py`, `gui/builtinItemStatsViews/*`, `service/market.py` |
| UI | fitting window, addition panes, stats panels, context menus, preferences, misc tools | `gui/*` |
| SVC | settings, update check, network, logging, i18n | `service/settings.py`, `service/update.py`, `gui/builtinPreferenceViews/*` |

## Counts

(filled in when the item tables are complete)

## Items

(per area, rendered from the YAML)
