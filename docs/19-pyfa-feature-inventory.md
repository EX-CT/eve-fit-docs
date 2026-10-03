# 19 — Pyfa feature inventory (scope baseline for the Rust mainline)

**Status: DRAFT 2 (2026-10-03, CST).** 200 items. Statuses are evidence-backed; every item gets a test ID in docs/20 §5.

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

## Method

1. **Walk the Pyfa source** (read-only), directory by directory:
   - `eos/saveddata/*`: every public method of `fit.py`, `module.py`, `drone.py`, `fighter.py`, `ship.py`,
     `character.py` and the other models.
   - `eos/capSim.py`, `eos/modifiedAttributeDict.py`, and `eos/effects.py` (2402 effect classes).
   - `graphs/data/*`: the X/Y definitions of all 10 graphs.
   - `service/*`, including `port/*` and `marketSources/*`.
   - `gui/`:
     - the main menu (`mainMenuBar.py`, every label), `builtinStatsViews` (every label), `builtinContextMenus` (51)
     - `builtinPreferenceViews` (11), `builtinAdditionPanes`, `builtinItemStatsViews`, `builtinViewColumns`
     - the editors.
   - The settings defaults in `service/settings.py` and `service/fit.py`.

   A GUI action that is only a different way into the same engine feature is folded into that feature. For example,
   `itemProject` maps to ENG-PROJ-001.
2. **F column.** Everything was probed on the box with the F binary from `lab-fp` (`variant-f-perf` f5b709f, built
   11:10 CST). F means the Rust mainline base.

   | suite | result |
   |---|---|
   | bench v1.9.0 | **330/331** cases, 22044/22046 values. One failure: `breacher_kestrel` `weapon_pure_dps` is 0, want 250. |
   | cap-suite d80cc38 | **147/150**. The failures are void bombs (`in_void_bomb`, `hard_void_bomb_bs`, `hard_three_void_bombs`). This is fixed on `variant-f-capfix` 9bf10a4 (150/150, per F's bot). |
   | mutated-suite 2ac7c00 | **86/93**, 6148/6184 values. Seven implant/booster slot-conflict and two-booster cases fail (ENG-IMP-004). |
   | pending-1.10 b4deddd `e_fz_*` | **8/8**, 442/442 values on the metrics the 1.9 scorer knows. The not-yet-scored `weapon_pure_dps` was skipped. |
   | formats-suite 7c716e7 | 4779/4779 native+wasm (F's own gate log in `variant-f-perf`). |
   | graphs-round2 84f7c2e | 178/178 on `graphs-g4`, the F + graph-layer branch. It is **not merged** into `variant-f-perf`, so every GRF item is `partial` for F. |

   Also checked:
   - `rg` over `variant-f/src` (output keys in `stats.rs`, validation codes, RPC methods).
   - F's bot's list (eve-dogma-lab `variant-f-perf` 102f4d4, `variant-f/docs/MISSING-PYFA-FEATURES.md`, cited as
     MPF). It was used as a cross-check only. This inventory adds about 40 items that MPF does not list, such as alpha
     clones, character implants, XML/EVEMon character import, skill-plan export, HTML export, price optimisation,
     drone/fighter EHP, the dependants view, item compare, conversions and the preference/UI catalog.
3. **MCP and WEB columns** come from two sources:
   - `git grep` on `origin/main` of EX-CT/eve-fit-mcp f223517 and EX-CT/eve-fit-web 4a4afd8 (read-only).
   - eve4's evidence draft (eve-fit-docs a08d943 `drafts/mcp-web-features.md`), including MCP test names and web
     `tools/e2e.mjs` check names.

   Both front-ends compute nothing themselves. An engine feature is `have` in MCP/WEB when the front-end exposes it
   and it is tested. It is `partial` when it is only passed through untested, or only works on some backends.

   Caveats on engines:
   - MCP's default engine is still the frozen eve-dogma-rs.
   - WEB defaults to `ts-worker`. F is selectable as `wasm-worker` or `wasm-g4-worker`.
4. **`n/a`** means the item is not that layer's job, e.g. UI tabs for the engine. A missing UI feature that is not
   the engine's job is not counted against F. It is still a product gap, and docs/20 assigns it to web.

**Beyond Pyfa:** the product already has features Pyfa lacks. These are not counted here because the rule is "never
less", and they are tracked in the front-end READMEs:
- MCP: `compare_fits`, `what_if`, `suggest_*`, `optimize_fit` (stat goals), `evaluate_profiles`, jargon search,
  prompts.
- WEB: share links, WASM in-browser engine, zh UI.

## Top gaps

These are ordered by user impact. Effort and owners are in docs/20 §4.

1. **Graphs are not in mainline F** (GRF-*, 10 graph types). They exist and pass on `graphs-g4` (GR2 178/178). Merging
   them is the largest single coverage gain. The MCP has no graph tool at all. The web has no ECM-burst graph and no
   multi-fit overlay or target fits (GRF-ECM-001, GRF-UI-001).
2. **Stats Pyfa shows that F does not compute:**
   - mining yield (ENG-OFF-006)
   - outgoing remote reps and cap transfer (ENG-PROJ-006)
   - drone and fighter EHP and regen (ENG-DRN-003, ENG-FTR-004)
   - bombing panel (ENG-OFF-007)
   - heat (ENG-MOD-007)
   - breacher pure DPS (ENG-MOD-012, bench `breacher_kestrel`; fixed on `variant-f-features` 7878a41, not merged)
   - the min/max spool pair (ENG-OFF-005)
   - lock times vs ship classes (ENG-TGT-002)
   - special holds (ENG-TGT-006)
3. **Engine correctness and coverage:**
   - implant/booster slot conflicts (ENG-IMP-004, MUT 7 cases; fix on `variant-f-features` 2046757, not merged)
   - void bomb in cap sim (ENG-PROJ-007, fixed on `variant-f-capfix` 9bf10a4, to merge)
   - no per-effect test map (ENG-CORE-003). The probe `tools/probe_pyfa_effects.py` found the following:
     - Dataset types use 2378 Pyfa effect classes.
     - 2266 of them carry SDE modifierInfo, which F's codegen handles.
     - 112 are handler-only. F's source names 75 of them.
     - The other 37 are not named in F's source. Most are handled generically: weapons, the weather/cloud dbuffs and
       the warfare links all pass B19. The rest is unverified: mining, breacher-DC, titan generator, point defense
       and lightning weapon.
4. **Modifier transparency:** "Affected by", dependants and skill affectors (ENG-CORE-007/008). They are missing
   everywhere. The engine must record modifier sources.
5. **Character model:**
   - alpha clones (ENG-CORE-009)
   - character implants (ENG-IMP-002, CHR-006)
   - EVEMon/XML import and skill-plan export (CHR-004/005)
   - ESI/SSO characters (CHR-003, ESI-001). Ruling of 2026-10-03: ESI login and skill/character fetch belong to the
     web frontend eve-fit-web (owner eve4, **later**; other clients may follow). The engine owns the
     character/skill input format (`character.skills`/`security`), which the web passes in after login. These items
     are therefore `n/a` for F and MCP and `missing (later)` for WEB.
6. **Prices:**
   - The MCP and engine have none.
   - The web uses only the ESI average price. Pyfa's sources and system choice are missing (PRC-001).
   - Also missing: the price column, price options and Optimize Fit Price (PRC-003..005).
7. **Formats and integrations:**
   - Missing everywhere: EFS (FMT-EFS-001) and HTML export (FMT-HTML-001).
   - ESI fittings browse/upload/delete (ESI-002/003) are missing too. They are web frontend work, later (eve4).
   - XML import is missing in MCP and web (F has it).
   - EVE item-rename conversions (SVC-005).
8. **Persistence and libraries outside the web:** saved fits, profile libraries and backups (DB-001/003/008). The
   MCP is stateless, and an app or CLI layer needs a persistence crate.
9. **Market and item info:**
   - market tree and variations (MKT-001, ENG-MOD-013), missing in F and MCP
   - item compare (MKT-004), missing everywhere
   - jargon search in F/web (MKT-002)
   - full localisation (MKT-005)

<!-- BEGIN GENERATED -->
<!-- GENERATED by tools/render_inventory.py from the YAML; do not edit by hand -->

## Counts (200 items)

| column | have | partial | missing | n/a |
|---|---|---|---|---|
| F engine/CLI/WASM | 78 | 52 | 37 | 33 |
| eve-fit-mcp | 70 | 52 | 53 | 25 |
| eve-fit-web | 104 | 55 | 35 | 6 |

Per area (have/partial/missing, n/a not counted):

| area | items | F | MCP | WEB |
|---|---|---|---|---|
| ENG-CORE Engine core | 9 | 5/1/3 | 4/2/3 | 6/0/3 |
| ENG-MOD Modules | 14 | 7/3/2 | 7/5/2 | 8/4/2 |
| ENG-SHIP Ships, modes, subsystems, structures | 6 | 5/1/0 | 3/3/0 | 3/3/0 |
| ENG-DRN Drones | 5 | 2/1/1 | 2/2/1 | 2/2/1 |
| ENG-FTR Fighters | 4 | 3/0/1 | 2/1/1 | 3/0/1 |
| ENG-IMP Implants and boosters | 5 | 2/1/2 | 2/2/1 | 3/1/1 |
| ENG-CAP Capacitor | 6 | 4/2/0 | 5/1/0 | 6/0/0 |
| ENG-DEF Defense / tank | 7 | 7/0/0 | 7/0/0 | 6/1/0 |
| ENG-OFF Offense / mining | 8 | 4/1/2 | 4/1/3 | 4/2/2 |
| ENG-NAV Navigation | 4 | 4/0/0 | 4/0/0 | 4/0/0 |
| ENG-TGT Targeting / sensors / misc | 6 | 4/2/0 | 4/2/0 | 5/1/0 |
| ENG-PROJ Projection / remote effects | 8 | 5/2/1 | 2/4/2 | 4/3/1 |
| ENG-FLT Fleet / command | 2 | 2/0/0 | 0/2/0 | 1/1/0 |
| ENG-ENV Environment | 4 | 4/0/0 | 0/4/0 | 4/0/0 |
| ENG-VAL Validation / restrictions | 7 | 6/1/0 | 6/1/0 | 5/2/0 |
| ENG-MISC Other engine items | 5 | 1/3/1 | 2/3/0 | 2/3/0 |
| GRF Graphs | 19 | 0/18/0 | 0/1/17 | 9/7/3 |
| CHR Characters and skills | 9 | 2/3/3 | 3/2/3 | 4/2/3 |
| PRC Prices | 5 | 0/0/5 | 0/0/5 | 1/1/3 |
| FMT Import / export | 14 | 7/4/2 | 3/5/5 | 5/3/6 |
| ESI ESI / SSO | 3 | 0/0/0 | 0/0/0 | 0/0/3 |
| DB Persistence / fit library | 9 | 0/1/3 | 0/2/3 | 5/3/0 |
| PRF Damage patterns / target profiles | 4 | 0/0/2 | 2/2/0 | 4/0/0 |
| MKT Market / item info | 6 | 0/2/3 | 2/2/1 | 1/4/1 |
| UI GUI panes, columns, menus, preferences | 26 | 4/4/5 | 5/4/5 | 8/12/4 |
| SVC Services / misc | 5 | 0/2/1 | 1/1/1 | 1/0/1 |

## Items

### ENG-CORE: Engine core

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-CORE-001 | **Dogma modifier engine**: Apply SDE effects/modifiers (ItemModifier, LocationModifier, LocationGroup/RequiredSkill, OwnerRequiredSkill) with all operators (preAssign, preMul, preDiv, modAdd, modSub, postMul, postDiv, postPercent, postAssign) | `eos/modifiedAttributeDict.py; eos/effects.py; eos/gamedata.py` | ✅ have: B19 330/331 (22044/22046 values), probe lab-fp f5b709f | ✅ have: compute_fit passes the full FitRequest to the engine backend (src/schemas.ts) | ✅ have: FitStats from engine worker (src/engine/adapter.ts) |
| ENG-CORE-002 | **Stacking penalties**: Penalised multiplier chains per attribute/operator group, stackable-attribute exemptions, penalty groups | `eos/modifiedAttributeDict.py (calculateValue, penalized)` | ✅ have: B19 esf_stacking_per*, e_fz_cloak_wcs_penalty_group_ninazu (FZ 8/8) | ✅ have: compute_fit passes the full FitRequest to the engine backend (src/schemas.ts) | ✅ have: FitStats from engine worker (src/engine/adapter.ts) |
| ENG-CORE-003 | **Hand-written effects (eos/effects.py, 2402 classes)**: Pyfa overrides/complements SDE expressions with hand-written handlers (incl. runTime early/normal/late, projected-only, activeByDefault quirks) | `eos/effects.py (class EffectNNNN)` | 🟡 partial: probe tools/probe_pyfa_effects.py: 2378 Pyfa effect classes used by dataset-3569502 types; 2266 have SDE modifierInfo (F codegen); 112 are handler-only, 75 named in F src (build.rs/engine.rs/stats.rs), 37 not named (weather_*/aoe clouds/warfare links handled via dbuffs and pass B19; mining*, salvaging, hacking, tractor, pointDefense, lightningWeapon, doomsdayAOEBubble, moduleTitanEffectGenerator, moduleBonusBreacherPodDamageControl unverified). Needs a per-effect test map. | ✅ have: engine pass-through | ✅ have: engine pass-through |
| ENG-CORE-004 | **Skills as modifier sources**: Skill levels drive skill effects; per-skill level; character-wide skill bonuses | `eos/saveddata/character.py (Skill.calculateModifiedAttributes)` | ✅ have: B19 skills0/2/3/4_* (70 cases) | ✅ have: compute_fit character.skills; skill_requirements tool | ✅ have: src/ui/Character.tsx |
| ENG-CORE-005 | **Attribute overrides**: User-set base attribute value overrides per type (Attribute Overrides menu/editor) | `eos/saveddata/override.py; gui/propertyEditor.py; mainMenuBar 'Attribute Overrides'` | ✅ have: request.overrides (src/request.rs, engine.rs) | 🟡 partial: schema overrides only (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: show-info override editor (e2e tools/e2e.mjs: 'attribute override raises weapon dps') |
| ENG-CORE-006 | **Full modified attribute dump per item**: All modified attributes of ship/module/charge/drone/fighter (item stats Attributes tab) | `gui/builtinItemStatsViews/itemAttributes.py` | ✅ have: options.include_attributes (stats.rs attributes) | 🟡 partial: get_type gives base attrs; fitted values only via compute_fit include_attributes | ✅ have: Market show info with fitted values |
| ENG-CORE-007 | **Attribute modifier sources (Affected by)**: Per attribute: which items/skills modified it, operator, value; skill affectors context menu | `gui/builtinItemStatsViews/itemAffectedBy.py; gui/builtinContextMenus/skillAffectors.py` | ❌ missing: options.sources parsed but unused (MPF A6; rg '.sources' src/ 0 hits) | ❌ missing: no tool exposes modifier sources (src/ grep 'affected' 0) | ❌ missing: no Affected-by tab (src/ grep 'affected' 0) |
| ENG-CORE-008 | **Dependants view**: Which attributes/items an item affects (itemDependants) | `gui/builtinItemStatsViews/itemDependants.py` | ❌ missing: no reverse-modifier output | ❌ missing: grep dependants 0 | ❌ missing: grep dependants 0 |
| ENG-CORE-009 | **Alpha clone restrictions**: Alpha clone skill caps (alphaCloneID) on character | `eos/saveddata/character.py (alphaCloneID); gui/characterEditor.py` | ❌ missing: request has no clone state; rg alpha src/ 0 | ❌ missing: only pirate-set names match 'alpha' | ❌ missing: Character.tsx has All 0/4/5/custom only |

### ENG-MOD: Modules

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-MOD-001 | **Module states**: offline/online/active/overheated with per-state effect application, canHaveState/isValidState/getMaxState | `eos/saveddata/module.py (isValidState, canHaveState, getMaxState, getProposedState)` | ✅ have: B19 overheat_order*, e_fz_offline_* (FZ) | ✅ have: schema modules[].state | 🟡 partial: click/right-click state cycle; no e2e check (eve4 a08d943) |
| ENG-MOD-002 | **Charges / ammo**: Loaded charge modifies module; charge capacity, numCharges, valid charges per launcher/turret | `module.py (charge, getValidCharges, isValidCharge, numCharges)` | ✅ have: B19 esf_charges*, validation CHARGE_GROUP/SIZE/CAPACITY (stats.rs validate) | ✅ have: suggest_charges tool | ✅ have: charge picker |
| ENG-MOD-003 | **Change ammo for all same modules**: moduleAmmoChange (ammoChangeAll setting) | `gui/builtinContextMenus/moduleAmmoChange.py; settings ammoChangeAll` | – n/a: UI feature | 🟡 partial: what_if can set charges per module | 🟡 partial: per-module only (Fitting.tsx) |
| ENG-MOD-004 | **Reload time / factor reload**: DPS/cap with reload factored (factorReload pref + context menu) | `module.py (reloadTime, forceReload); gui/builtinContextMenus/factorReload.py` | ✅ have: options.factor_reload; B19 sustain_reload_*; CAP reload 8/8 | ✅ have: options passthrough | 🟡 partial: options tab; no e2e (a08d943) |
| ENG-MOD-005 | **Spool-up weapons/remote reps**: Triglavian disintegrators, mutadaptive reps: spool type/amount; default spool pref; per-module spool menu | `module.py (getSpoolData); gui/builtinContextMenus/moduleSpool.py; eos/utils/spoolSupport.py` | ✅ have: B19 esf_spool*, CAP spool 6/6, modules[].spool, options.default_spool | 🟡 partial: FitRequest spool (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: per-module spool (e2e tools/e2e.mjs: 'per-module spool 0% lowers disintegrator dps') |
| ENG-MOD-006 | **Overheat bonuses**: Overheated state bonuses (heat damage modelled separately) | `module.py; effects overload*` | ✅ have: B19 overheat_order*, CAP overheat 15/15, FZ e_fz_capsim_overheat_* | ✅ have: state overheated | ✅ have: state toggle |
| ENG-MOD-007 | **Heat column (heat generation/damage)**: Heat per module column (heatDamage, heatAbsorbtionRateModifier) | `gui/builtinViewColumns/heat.py` | ❌ missing: no heat model (MPF A7) | ❌ missing: not exposed | ❌ missing: no heat column |
| ENG-MOD-008 | **Mutated / abyssal modules**: Mutaplasmid rolls per attribute, base type, mutator; mutated drones | `eos/saveddata/mutatedMixin.py, mutator.py` | ✅ have: MUT 86/93 (mm_*, mwd_*, edge_*, drone_* pass; 7 combo_/slot_ booster/implant cases fail, see ENG-IMP-004) | 🟡 partial: FitRequest mutation (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: sliders (e2e tools/e2e.mjs: 'mutated module imported', 'mutated module EFT round trip') |
| ENG-MOD-009 | **Mutation stats view**: Item stats Mutator tab: roll range/percentage per attribute | `gui/builtinItemStatsViews/itemMutator.py; gui/builtinContextMenus/itemMutations.py` | 🟡 partial: F returns modified values; no roll-range output | ❌ missing: no mutaplasmid range output (src/ grep mutaplasmid 0) | ✅ have: slider UI shows ranges |
| ENG-MOD-010 | **Module ranges**: Optimal/falloff, missile range (incl. Pyfa missileMaxRangeData lower/upper chance) | `module.py (maxRange, falloff, missileMaxRangeData, calculateRange)` | 🟡 partial: stats.rs optimal_m/falloff_m/max_range_m; missile range-chance pair not output | ✅ have: via compute_fit | ✅ have: maxRange column |
| ENG-MOD-011 | **Capacitor use per module**: capUse incl. reactivation delay, cycle time | `module.py (capUse, rawCycleTime, reactivationDelay, disallowRepeatingAction)` | ✅ have: stats.rs modules cycle_time_ms/use; CAP 147/150 | ✅ have: via compute_fit | ✅ have: capacitorUse column |
| ENG-MOD-012 | **Breacher pods**: Pure-damage breachers (% HP + flat cap) DPS | `module.py (isBreacher, getVolleyParameters)` | 🟡 partial: B19 breacher_kestrel fails on variant-f-perf f5b709f (weapon_pure_dps 0 vs 250); fixed on variant-f-features 7878a41 (B19 331/331, contract 1.4.4 'pure' key), not yet merged | 🟡 partial: depends on engine | 🟡 partial: depends on engine |
| ENG-MOD-013 | **Module variations / meta swap**: Switch module to meta variation (itemVariationChange), market group siblings | `gui/builtinContextMenus/itemVariationChange.py; service/market.py (getVariationsByItems)` | ❌ missing: no variation lookup over RPC (MPF E2) | ✅ have: get_type same_group variations (src/server.ts:145); suggest_modules | ❌ missing: no variations window (a08d943 §1 gaps) |
| ENG-MOD-014 | **Module position / rack handling**: Slot positions, rack labels, empty slot placeholders | `module.py (modPosition, buildEmpty, buildRack)` | – n/a: fit model concern | 🟡 partial: order preserved in EFT | ✅ have: Fitting.tsx slots |

### ENG-SHIP: Ships, modes, subsystems, structures

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-SHIP-001 | **T3D tactical modes**: Mode item applies bonuses; shipModeChange menu | `eos/saveddata/mode.py; ship.py (modes, validateModeItem); gui/builtinContextMenus/shipModeChange.py` | ✅ have: B19 esf_tactical*, request.mode | 🟡 partial: mode_type_id (eve4 draft a08d943: works, no dedicated MCP test) | 🟡 partial: mode selector; no e2e (a08d943) |
| ENG-SHIP-002 | **T3C subsystems**: Subsystem slot, slot/hardpoint additions | `eos/effects.py (subsystem effects); fit.py getNumSlots` | ✅ have: B19 esf_slots*, Slot::Subsystem in validate | 🟡 partial: subsystems (eve4 draft a08d943: works, no dedicated MCP test) | 🟡 partial: subsystem slots; no e2e (a08d943) |
| ENG-SHIP-003 | **Structures / citadels**: Citadel fits, service slots, structure rigs, standup modules, fuel | `eos/saveddata/citadel.py; fit.py isStructure` | ✅ have: B19 esf_structure*, standup_* | ✅ have: structures supported (src/ grep citadel 8 files) | ✅ have: structures |
| ENG-SHIP-004 | **Pilot security status**: Security-status dependent effects (e.g. Pirate faction / Concord) | `fit.py getPilotSecurity; gui/builtinContextMenus/fitPilotSecurity.py` | ✅ have: B19 esf_security_status; character security | 🟡 partial: character.security_status (eve4 draft a08d943: works, no dedicated MCP test) | 🟡 partial: Character page; no e2e (a08d943) |
| ENG-SHIP-005 | **System security**: High/low/null/WH system effects (e.g. Cynosural, Upwell modules) | `fit.py getSystemSecurity; gui/builtinContextMenus/fitSystemSecurity.py` | ✅ have: environment.system_security; B19 esf_security | ✅ have: schema environment | ✅ have: environment security |
| ENG-SHIP-006 | **Ship traits / role bonuses**: Traits text per ship | `gui/builtinItemStatsViews/itemTraits.py` | 🟡 partial: type RPC returns data; traits text not checked | ✅ have: get_ship | ✅ have: Market show info traits |

### ENG-DRN: Drones

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-DRN-001 | **Drones: active/inactive stacks**: Drone stacks with amount/active count, bandwidth, bay, release/store limits | `eos/saveddata/drone.py; fit.py getReleaseLimitForDrone, getStoreLimitForDrone` | ✅ have: B19 drones_*, DRONE_BANDWIDTH validation | ✅ have: schema drones | ✅ have: drone pane |
| ENG-DRN-002 | **Drone DPS/volley/mining/RR**: Per-drone damage, mining, remote reps | `drone.py getDps, getMiningYPS, getRemoteReps` | 🟡 partial: dps/volley have; drone mining and drone RR outgoing missing (MPF A1/A2) | 🟡 partial: engine | 🟡 partial: engine |
| ENG-DRN-003 | **Drone EHP and shield regen columns**: droneEhp/droneRegen view columns | `gui/builtinViewColumns/droneEhp.py, droneRegen.py; drone.py hp/ehp/calculateShieldRecharge` | ❌ missing: drone rows have no hp/ehp (MPF A4) | ❌ missing: not exposed | ❌ missing: no column |
| ENG-DRN-004 | **Drone speed/range/sig**: Drone max velocity, optimal, falloff, control range | `drone.py maxRange, falloff` | ✅ have: FZ e_fz_drone_speed_* 2/2; stats.rs control_range_m | ✅ have: engine | ✅ have: drone pane |
| ENG-DRN-005 | **Split/merge drone stacks**: droneAddStack/SplitStack | `gui/builtinContextMenus/droneAddStack.py, droneSplitStack.py` | – n/a: fit-model op | 🟡 partial: what_if edits | 🟡 partial: amount edit only |

### ENG-FTR: Fighters

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-FTR-001 | **Fighters squadrons**: Fighter tubes, squadron size, light/heavy/support slots, bay | `eos/saveddata/fighter.py; fit.py fighterTubesUsed` | ✅ have: B19 fighters_*, stats.rs fighter_tubes/squadron_size | ✅ have: schema fighters | ✅ have: fighter pane |
| ENG-FTR-002 | **Fighter abilities**: Per-ability on/off (attack, missiles, bombs, MWD, MJD, evasive, ECM, web, point, neut) | `eos/saveddata/fighterAbility.py; gui/builtinContextMenus/fighterAbilities.py` | ✅ have: B19 fighters_*, pfighter_* | 🟡 partial: FitRequest abilities (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: Pyfa-default abilities (e2e tools/e2e.mjs: 'disabling an attack ability lowers fighter dps') |
| ENG-FTR-003 | **Fighter DPS per effect**: Volley/DPS per ability incl. bomb cycles | `fighter.py getDpsPerEffect, getCycleParametersPerEffect*` | ✅ have: stats.rs fighter_dps; B19 fighters_* | ✅ have: engine | ✅ have: engine |
| ENG-FTR-004 | **Fighter EHP / regen**: fighter.py hp/ehp/calculateShieldRecharge | `fighter.py hp, ehp` | ❌ missing: not in output | ❌ missing: - | ❌ missing: - |

### ENG-IMP: Implants and boosters

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-IMP-001 | **Implants**: Implant slots, set bonuses (pirate sets multiply) | `eos/saveddata/implant.py` | ✅ have: B19 implants_* | ✅ have: schema implants; implant_set presets | ✅ have: implant pane |
| ENG-IMP-002 | **Character implants vs fit implants**: implantSource (fit or character), useCharacterImplantsByDefault | `fit.py implantSource, appliedImplants` | ❌ missing: implants only per request | ❌ missing: - | ❌ missing: implants per fit only |
| ENG-IMP-003 | **Boosters and side effects**: Booster slots, side effects toggles (boosterSideEffects menu) | `eos/saveddata/booster.py, boosterSideEffect.py; gui/builtinContextMenus/boosterSideEffects.py` | ✅ have: B19 booster_se_* (7), booster_* | 🟡 partial: FitRequest side_effects (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: side effects (e2e tools/e2e.mjs: 'booster side effect lowers armor HP') |
| ENG-IMP-004 | **Implant/booster slot conflicts**: Same-slot implants/boosters: Pyfa keeps first; boosters slot rules | `fit.py; service/fit.py addImplant/addBooster (replace by slot)` | 🟡 partial: MUT 86/93 on f5b709f: combo_two_boosters_dda, slot_booster_* (3), slot_implant_* (2) fail; fix on variant-f-features 2046757 (first entry per slot wins), not yet merged | 🟡 partial: engine | 🟡 partial: engine |
| ENG-IMP-005 | **Implant sets (saved + precalculated)**: Implant Set Editor, apply/save set to fit, precalc faction sets | `eos/saveddata/implantSet.py; service/implantSet.py; service/precalcImplantSet.py; gui/setEditor.py; gui/builtinContextMenus/implantSetApply.py, implantSetSave.py` | ❌ missing: implants: [type_id] only (MPF C2) | ✅ have: list_presets implant_set (src/profiles.ts) | ✅ have: SDE + saved sets (e2e tools/e2e.mjs: 'implant set applied (6 Snake + kept RP-905, faster)') |

### ENG-CAP: Capacitor

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-CAP-001 | **Capacitor capacity/recharge/peak**: capacitorCapacity, recharge time, peak recharge, capUsed, capDelta | `fit.py capUsed, capRecharge, capDelta, calculateCapRecharge` | ✅ have: CAP capacity/recharge/peak 150/150 | ✅ have: engine | ✅ have: Stats capacitor |
| ENG-CAP-002 | **Capacitor simulation (stable %, lasts)**: Event sim with stagger, reload, injectors, max time | `eos/capSim.py; fit.py simulateCap, getCapSimData` | ✅ have: CAP 147/150 (f5b709f); 150/150 on variant-f-capfix 9bf10a4 | ✅ have: options.cap_sim | ✅ have: Stats capacitor |
| ENG-CAP-003 | **Cap injectors / boosters**: Charges inject cap, reload | `capSim.py; module charges` | ✅ have: CAP injectors 14/14 | ✅ have: engine | ✅ have: engine |
| ENG-CAP-004 | **Own neuts/nos in cap sim**: Own neutralizer use, nosferatu gain (incl. no-target option) | `fit.py addDrain/getCapRegenGainFromMod` | ✅ have: CAP own_neut_nos 10/10; options.nos_no_target_cap | ✅ have: engine | ✅ have: engine |
| ENG-CAP-005 | **Incoming neut / remote cap in sim**: Projected neuts/nos/cap transfers into cap sim | `fit.py addDrain, iterDrains` | 🟡 partial: CAP incoming 15/16, hard 12/14 (void bombs) on f5b709f; 150/150 on variant-f-capfix 9bf10a4 / variant-f-features be35345 | ✅ have: engine | ✅ have: engine |
| ENG-CAP-006 | **Capacitor graph data (cap over time)**: capacitor vs time series | `graphs/data/fitCapacitor` | 🟡 partial: graphs-g4 only (GRF-CAP-*) | 🟡 partial: sweep tool not time-series | ✅ have: Graphs capacitor |

### ENG-DEF: Defense / tank

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-DEF-001 | **HP and resists per layer**: Shield/armor/hull HP and EM/Th/Kin/Ex resonance | `fit.py hp; resistancesViewFull` | ✅ have: B19 all cases (hp.*, resonance) | ✅ have: engine | ✅ have: Stats defense |
| ENG-DEF-002 | **EHP vs damage pattern**: Effective HP vs selected damage pattern; raw/effective toggle | `fit.py ehp; damagePattern` | ✅ have: B19 dmgpattern_* | ✅ have: damage_pattern | ✅ have: Profiles |
| ENG-DEF-003 | **Tank: active/sustained/reinforced**: Active shield boost, armor/hull repair, passive regen; sustainable (cap) and effective | `fit.py tank, effectiveTank, sustainableTank, calculateSustainableTank; rechargeViewFull` | ✅ have: stats.rs tank raw/sustained/sustained_effective; B19 sustain_* | ✅ have: engine | ✅ have: Stats |
| ENG-DEF-004 | **Reactive armor hardener sim**: RAH adaptation to damage pattern; resistMode/moduleRahPattern | `eos/effects.py (Effect4928 adaptive armor hardener); gui/builtinContextMenus/moduleRahPattern.py` | ✅ have: B19 esf_reactive_armor, dmgpattern_rah; options.rah | ✅ have: engine | 🟡 partial: options tab RAH; no e2e (a08d943) |
| ENG-DEF-005 | **Ancillary repairers (charged)**: AAR/ASB with paste/charges, reload | `effects; module charges` | ✅ have: B19 esf_ancillary*, proj_aar_* | ✅ have: engine | ✅ have: engine |
| ENG-DEF-006 | **Shield recharge peak**: calculateShieldRecharge peak passive regen | `fit.py calculateShieldRecharge` | ✅ have: stats.rs passive_shield | ✅ have: engine | ✅ have: Stats |
| ENG-DEF-007 | **Incoming remote reps in tank**: Projected RR applied to sustained tank | `fit.py getRemoteReps (incoming via projected)` | ✅ have: stats.rs incoming RR (MPF); B19 proj_ars/rar/rsb | ✅ have: engine | ✅ have: engine |

### ENG-OFF: Offense / mining

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-OFF-001 | **Weapon DPS/volley**: Turrets, missiles, smartbombs, bombs per module; with spool and reload | `module.py getVolley/getDps/getCycleParametersForDps; fit.py getWeaponDps` | ✅ have: B19 (weapon_dps/volley all) | ✅ have: engine | ✅ have: Stats offense |
| ENG-OFF-002 | **Drone and fighter DPS totals**: fit.py getDroneDps/Volley, total | `fit.py getTotalDps` | ✅ have: B19 drone_dps, fighter_dps | ✅ have: engine | ✅ have: Stats |
| ENG-OFF-003 | **DPS vs target profile (applied)**: Resist-aware DPS vs target profile | `fit.py calculateWeaponDmgStats; targetProfile` | ✅ have: stats.rs vs_target_profile | ✅ have: evaluate_profiles | ✅ have: Profiles |
| ENG-OFF-004 | **Doomsdays / superweapons**: Doomsday damage | `effects; stats` | ✅ have: stats.rs doomsday (engine.rs); covered in B19 exct_avatar? | ✅ have: engine | ✅ have: engine |
| ENG-OFF-005 | **Spool min/max/current DPS display**: firepowerViewFull spool-up tooltip (min/max) | `gui/builtinStatsViews/firepowerViewFull.py` | 🟡 partial: single spool value per request; min/max need two calls | 🟡 partial: what_if | 🟡 partial: spool option |
| ENG-OFF-006 | **Mining yield (modules + drones)**: m3/s mining and drain (residue), mining panel | `fit.py minerYield, droneYield, minerDrain, totalYield; miningyieldViewFull` | ❌ missing: no mining key in calc output (MPF A1, rg mining stats.rs 0) | ❌ missing: no mining metric (src/ grep mining 0) | 🟡 partial: Stats.tsx shows st.mining only if backend provides (D ts-worker), not F |
| ENG-OFF-007 | **Bombing panel**: Bombs needed per bomb type vs target | `gui/builtinStatsViews/bombingViewFull.py` | ❌ missing: MPF A3 | ❌ missing: - | ❌ missing: - |
| ENG-OFF-008 | **Ammo to damage pattern**: Create damage pattern from loaded ammo | `gui/builtinContextMenus/ammoToDmgPattern.py` | – n/a: UI over type attrs | ❌ missing: - | ❌ missing: grep ammo pattern 0 |

### ENG-NAV: Navigation

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-NAV-001 | **Speed, align, agility, mass, sig**: Max velocity, align time, agility, signature | `fit.py maxSpeed, alignTime` | ✅ have: B19 max_velocity/align_time_s | ✅ have: engine | ✅ have: Stats navigation |
| ENG-NAV-002 | **Warp speed and max warp distance**: warpSpeed, maxWarpDistance (cap) | `fit.py warpSpeed, maxWarpDistance` | ✅ have: stats.rs warp_speed_au_s, max_warp_distance_au | ✅ have: engine | ✅ have: Stats |
| ENG-NAV-003 | **Warp core strength / scram status**: Warp scramble status | `targetingMiscViewMinimal (Warp Core Strength)` | ✅ have: stats.rs warp_scramble_status | ✅ have: engine | ✅ have: Stats |
| ENG-NAV-004 | **Propulsion mods (AB/MWD/MJD/bastion/siege)**: Speed boosts, sig bloom, MJD; cloaks/WCS penalties | `effects` | ✅ have: B19 + FZ e_fz_cloak_wcs_* | ✅ have: engine | ✅ have: engine |

### ENG-TGT: Targeting / sensors / misc

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-TGT-001 | **Targeting stats**: Max targets, lock range, scan res, sensor strength/type | `fit.py maxTargets, maxTargetRange, scanStrength, scanType` | ✅ have: stats.rs targeting | ✅ have: engine | ✅ have: Stats targeting |
| ENG-TGT-002 | **Lock time vs ship classes**: Lock times vs Frigate…Carrier sigs (tooltip) | `targetingMiscViewMinimal; fit.py calculateLockTime` | 🟡 partial: lock_time_s vs target-profile sig only | 🟡 partial: engine | ✅ have: Stats lock time |
| ENG-TGT-003 | **ECM jam chance**: % chance to be jammed (incoming ECM) | `fit.py jamChance` | ✅ have: stats.rs jam_chance_percent; B19 ecm_* | ✅ have: engine | ✅ have: Stats |
| ENG-TGT-004 | **Probe size / scan**: Probe size | `fit.py probeSize` | ✅ have: stats.rs probe_size | ✅ have: engine | ✅ have: Stats |
| ENG-TGT-005 | **Drone control range**: droneControlRange | `targetingMisc` | ✅ have: stats.rs control_range_m | ✅ have: engine | ✅ have: Stats |
| ENG-TGT-006 | **Cargo and special holds**: Cargo, ore/fuel/fleet/ammo/... holds volumes | `targetingMiscViewMinimal (holds list)` | 🟡 partial: stats.rs cargo capacity; special holds not all output | 🟡 partial: engine | 🟡 partial: cargo only |

### ENG-PROJ: Projection / remote effects

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-PROJ-001 | **Projected modules**: Remote modules (webs, TP, neut, nos, ECM, damp, TD, GD, RR, cap transfer) on the fit | `fit.py projectedModules; gui/builtinContextMenus/itemProject.py` | ✅ have: B19 proj_* (30) | 🟡 partial: FitRequest projected (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: Projected tab (e2e tools/e2e.mjs: 'projected web slows the ship') |
| ENG-PROJ-002 | **Projected drones / fighters**: Drones/fighters projected, amounts | `fit.py projectedDrones/Fighters` | ✅ have: B19 proj_*drone*, pfighter_* (6) | 🟡 partial: FitRequest projected (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: projected |
| ENG-PROJ-003 | **Projected fits**: Whole fits projected (with amount, active) | `fit.py projectedFits, getProjectionInfo` | ✅ have: B19 projfit_* (10) | 🟡 partial: projected[kind=fit] (eve4 draft a08d943: works, no dedicated MCP test) | 🟡 partial: nested toRequest (fit/model.ts); no e2e (a08d943) |
| ENG-PROJ-004 | **Projection range (distance)**: Range-dependent strength (falloff) for projected items | `itemProjectionRange context menu; projectionRange column` | ✅ have: projected distance; B19 *_falloff | ✅ have: engine | ✅ have: projected distance |
| ENG-PROJ-005 | **Burst projectors / AoE / doomsday AoE**: Burst jammers, AoE structure modules | `effects` | ✅ have: B19 esf_burst*, aoe_* (5), ecm_burst | ✅ have: engine | ✅ have: engine |
| ENG-PROJ-006 | **Outgoing remote reps / cap transfer stats**: Outgoing shield/armor/hull/cap per s (with spool) | `fit.py getRemoteReps; gui/builtinStatsViews/outgoingViewFull.py` | ❌ missing: no outgoing key in calc (MPF A2); only shipstats export uses remote_reps (formats.rs:1529) | ❌ missing: - | 🟡 partial: Stats 'Remote assistance' section only when backend provides |
| ENG-PROJ-007 | **Bombs vs cap (void bomb)**: Void bomb neut projection | `effects` | 🟡 partial: CAP in_void_bomb, hard_*void_bomb* fail on f5b709f; fixed on variant-f-capfix 9bf10a4 / variant-f-features be35345 (CAP 150/150), not yet merged | 🟡 partial: engine | 🟡 partial: engine |
| ENG-PROJ-008 | **Target resists column / ignore resists**: targetResists column | `gui/builtinViewColumns/targetResists.py` | 🟡 partial: projected resist applied; no per-row output | ❌ missing: - | ❌ missing: - |

### ENG-FLT: Fleet / command

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-FLT-001 | **Command bursts / warfare buffs**: Command burst modules + charges → warfare buffs (shield/armor/skirmish/info/mining) | `fit.py addCommandBonus; effects warfare*` | ✅ have: B19 fleet_*, esf_burst | 🟡 partial: fleet.buffs (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: buff editor (e2e tools/e2e.mjs: 'manual fleet buff raises shield resist') |
| ENG-FLT-002 | **Command fits (booster fits)**: Other fits as command boosters; strongest-wins | `fit.py commandFits, getCommandInfo; gui/builtinContextMenus/commandFitAdd.py` | ✅ have: booster_fits; B19 fleet_two_boosters; stats strongest | 🟡 partial: fleet.booster_fits (eve4 draft a08d943: works, no dedicated MCP test) | 🟡 partial: booster-fit picker; no e2e (a08d943) |

### ENG-ENV: Environment

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-ENV-001 | **Wormhole effects**: Pulsar/Magnetar/Black hole/WR/Cataclysmic/Red giant classes 1-6 | `fit.py systemEffect; envEffectAdd` | ✅ have: B19 env_* (8) | 🟡 partial: environment (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: environment |
| ENG-ENV-002 | **Abyssal weather**: Electric/Caustic/Darkness/Infernal/Xenon/… + tiers | `envEffectAdd` | ✅ have: B19 weather_* (6) | 🟡 partial: environment (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: environment |
| ENG-ENV-003 | **Incursion / Drifter / metaliminal / filament clouds**: Incursion effects, Drifter, cloud filaments | `envEffectAdd` | ✅ have: B19 incursion_* (3), cloud_* (3) | 🟡 partial: environment (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: environment |
| ENG-ENV-004 | **Beacons / location bonus**: Beacons and location effects | `effects` | ✅ have: B19 esf_beacons, esf_location_bonus_not_on | 🟡 partial: environment (eve4 draft a08d943: works, no dedicated MCP test) | ✅ have: beacon selector (e2e tools/e2e.mjs: 'environment beacon applied') |

### ENG-VAL: Validation / restrictions

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-VAL-001 | **Resources overflow**: CPU/PG/calibration/bandwidth over limit | `fit.py cpuUsed, pgUsed, calibrationUsed, droneBandwidthUsed` | ✅ have: validate CPU/POWER/CALIBRATION/DRONE_BANDWIDTH; FZ e_fz_cpu_round_tie_* | ✅ have: validate_fit | ✅ have: Problems section |
| ENG-VAL-002 | **Slots and hardpoints**: Slots per rack, turret/launcher hardpoints | `fit.py getSlotsUsed, getHardpointsUsed` | ✅ have: SLOTS_EXCEEDED, TURRET/LAUNCHER_HARDPOINTS | ✅ have: validate_fit | ✅ have: Fitting |
| ENG-VAL-003 | **canFit restrictions**: canFitShipGroup/Type, rig size, maxGroupFitted/Online/Active, maxTypeFitted | `module.py fits; fit.py canFit` | ✅ have: SHIP_RESTRICTION, RIG_SIZE, MAX_GROUP_*, MAX_TYPE_FITTED | ✅ have: validate_fit | ✅ have: Problems |
| ENG-VAL-004 | **Missing skills (prereqs)**: Required skills per item vs character | `service/prereqsCheck.py; itemRequirements view` | ✅ have: MISSING_SKILL | ✅ have: skill_requirements | ✅ have: missing/train-required skills |
| ENG-VAL-005 | **Disable fitting restrictions**: Toggle ignore restrictions (mainMenu) | `gui/mainMenuBar.py 'Disable Fitting Restrictions'` | ✅ have: violations reported, never block | ✅ have: allow_violations | 🟡 partial: fits always allowed; no strict mode toggle found |
| ENG-VAL-006 | **Charge validity**: Charge group/size/capacity | `module.py isValidCharge` | ✅ have: CHARGE_GROUP/SIZE/CAPACITY | ✅ have: validate_fit | ✅ have: charge picker filtered |
| ENG-VAL-007 | **Fit-level special restrictions**: e.g. one cloak, max 1 per group, structure-only modules, T3D/T3C subsystem completeness | `fit.py isInvalid; ship.py validate` | 🟡 partial: generic max-group checks; subsystem completeness not checked | 🟡 partial: engine | 🟡 partial: engine |

### ENG-MISC: Other engine items

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ENG-MISC-001 | **Smartbombs and bombs**: Smartbomb DPS, bomb launchers (incl. fighter bombs) | `module.py getVolleyParameters` | ✅ have: stats.rs smartbomb (l.283/421); fighterAbilityLaunchBomb | ✅ have: engine | ✅ have: engine |
| ENG-MISC-002 | **Utility modules without stats**: Salvagers, tractor beams, hacking/data analyzers, cynos, cloaks, point defense, lightning weapon — Pyfa effects with no stat output | `eos/effects.py (salvaging, tractorBeamCan, doHacking, pointDefense, lightningWeapon)` | 🟡 partial: probe tools/probe_pyfa_effects.py: not named in F src; no stats expected except pointDefense/lightningWeapon damage | ✅ have: engine | ✅ have: engine |
| ENG-MISC-003 | **Titan/doomsday AoE bubbles and effect generators**: moduleTitanEffectGenerator, doomsdayAOEBubble | `eos/effects.py` | 🟡 partial: not named in F src (probe); B19 exct_avatar passes for stats | 🟡 partial: engine | 🟡 partial: engine |
| ENG-MISC-004 | **Breacher pod damage control interaction**: moduleBonusBreacherPodDamageControl | `eos/effects.py` | ❌ missing: not named in F src (probe); see ENG-MOD-012 | 🟡 partial: engine | 🟡 partial: engine |
| ENG-MISC-005 | **Industrial / compression / ore holds**: Rorqual/Porpoise compression, industrial core, mining holds | `effects; targetingMisc holds` | 🟡 partial: effects apply; hold capacities not output | 🟡 partial: engine | 🟡 partial: - |

### GRF: Graphs

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| GRF-DMG-001 | **Damage stats graph**: dps/volley/damage vs distance, time, target speed (m/s, %), target sig (m, %); inputs attacker/target speed+angle, target profile or fit, ignore resists, apply projected, drone mobile mode | `graphs/data/fitDamageStats/*; gui/builtinContextMenus/graphDmg*` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 dmg_* 63 cases | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-APP-001 | **Ammo application profile graph**: best ammo dps/volley vs distance, ammo picker, ammo quality | `graphs/data/fitApplicationProfile; graphFitAmmoPicker, graphAmmoOptimal*` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 app_* 10 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-CAP-001 | **Capacitor graph**: cap amount (GJ/%) and regen vs time / cap amount | `graphs/data/fitCapacitor` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 cap_* 13 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-ECM-001 | **ECM/burst/scan-res/damps graph**: source damage / target lock time / lock uptime vs target DPS or scan res | `graphs/data/fitEcmBurstScanresDamps` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 ecm_* 10 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ❌ missing: no ecm_burst graph / extra inputs in web (a08d943 §9 gaps) |
| GRF-EWAR-001 | **EWAR stats graph**: neut, web %, ECM, damp lock range %, TD optimal %, GD range %, TP % vs distance | `graphs/data/fitEwarStats` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 ewar_* 21 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-LOCK-001 | **Lock time graph**: lock time vs target sig | `graphs/data/fitLockTime; graphLockRange` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 lock_* 7 | 🟡 partial: sweep target_signature + lock metric | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-MOB-001 | **Mobility graph**: speed, distance, momentum, bump speed/distance vs time | `graphs/data/fitMobility` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 mob_* 10 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-RR-001 | **Remote reps graph**: RR HP/s and total vs distance/time (spool) | `graphs/data/fitRemoteReps` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 rr_* 9 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-SHLD-001 | **Shield regen graph**: shield amount/regen (HP/EHP) vs time / amount | `graphs/data/fitShieldRegen` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 shield_* 5 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-WARP-001 | **Warp time graph**: warp time vs distance (AU/km) | `graphs/data/fitWarpTime` | 🟡 partial: graphs-g4 (F + graph layer) GR2 178/178 native+wasm; not merged into variant-f-perf (MPF B1: 'eve-dogma-f graph' → usage); GR2 warp_* 8 | ❌ missing: no graph tool; sweep (src/server.ts:578) gives x=sig/velocity/skill/distance series for calc metrics only | ✅ have: src/ui/Graphs.tsx engine graph on wasm-g4-worker (browser GR2 178/178 CI; e2e 'graph * (engine)'); default ts-worker backend uses UI approximations (a08d943 §9) |
| GRF-UI-001 | **Graph frame features**: multiple source fits and targets in one graph, target profiles as targets, legend, colors/line styles, export? (Pyfa: copy/save image) | `graphs/gui/frame.py, lists.py, canvasPanel.py` | – n/a: UI | – n/a: - | ❌ missing: no multi-fit overlay or graph target fits (a08d943 §9 gaps) |
| GRF-OPT-001 | **Damage graph options**: Attacker/Target lists (fits or target profiles); inputs: distance, time, target speed/sig (abs and %), attacker speed/angle, target angle; checkboxes: apply projected (webs/TPs of source), ignore resists, drone mobile mode, (graphDmgApplyProjected/DroneMode/IgnoreResists), drone control range limit | `graphs/data/fitDamageStats/graph.py; gui/builtinContextMenus/graphDmg*.py, graphDroneControlRange.py` | 🟡 partial: graphs-g4: params ignore_resists, apply_projected, mobile_drone_mode (CONTRACT-GRAPHS 0.2 BAD_REQUEST enums); GR2 dmg_* | ❌ missing: no graph tool (a08d943 §9) | 🟡 partial: Graphs.tsx: no attacker/target lists, no per-graph extra inputs (a08d943 §9 gaps) |
| GRF-OPT-002 | **Capacitor graph options**: Starting cap amount, use capacitor simulator checkbox (useCapsim) | `graphs/data/fitCapacitor/graph.py` | 🟡 partial: graphs-g4 GR2 cap_* | ❌ missing: - | 🟡 partial: Graphs.tsx capacitor; options not exposed |
| GRF-OPT-003 | **Mobility graph target inputs**: Target mass / inertia factor for bump speed and distance | `graphs/data/fitMobility/graph.py` | 🟡 partial: graphs-g4 GR2 mob_bump* | ❌ missing: - | 🟡 partial: speed/distance only |
| GRF-OPT-004 | **Remote reps graph options**: Reload ancillary RRs (ancReload), spool | `graphs/data/fitRemoteReps/graph.py` | 🟡 partial: graphs-g4 GR2 rr_* | ❌ missing: - | 🟡 partial: engine rr graph on g4 |
| GRF-OPT-005 | **ECM/burst graph options**: applyDamps, applyDrones | `graphs/data/fitEcmBurstScanresDamps/graph.py` | 🟡 partial: graphs-g4 GR2 ecm_*, err_ecm* | ❌ missing: - | ❌ missing: - |
| GRF-OPT-006 | **EWAR graph target resistance option**: Target resistance checkbox | `graphs/data/fitEwarStats/graph.py` | 🟡 partial: graphs-g4 GR2 ewar_resist* | ❌ missing: - | 🟡 partial: - |
| GRF-OPT-007 | **Shield regen graph options**: Starting shield amount, effective HP toggle (isEffective) | `graphs/data/fitShieldRegen/graph.py` | 🟡 partial: graphs-g4 GR2 shield_* | ❌ missing: - | 🟡 partial: shield_pct/regen |
| GRF-OPT-008 | **Lock range / ammo picker graph menus**: graphLockRange, graphFitAmmoPicker, graphAmmoOptimalApplyProjected/IgnoreResists | `gui/builtinContextMenus/graph*.py` | 🟡 partial: graphs-g4 app_* params | ❌ missing: - | 🟡 partial: - |

### CHR: Characters and skills

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| CHR-001 | **All 0 / All 5 built-in characters**: Default characters with all skills 0 or 5 | `service/character.py (all0, all5)` | ✅ have: character.skills.default_level | ✅ have: default all V; skills param | ✅ have: All 5/4/0 |
| CHR-002 | **Custom characters (per-skill levels, save/rename/copy/delete)**: Character editor | `gui/characterEditor.py; service/character.py` | 🟡 partial: per-request levels only (MPF C1) | 🟡 partial: skills map per call; no saved characters | ✅ have: Character.tsx custom levels (e2e tools/e2e.mjs: 'custom character (all 0) lowers dps') |
| CHR-003 | **Import character from ESI (SSO)**: Fetch skills via ESI for logged-in character *Owner: eve-fit-web (eve4), later; engine owns the character/skill input format.* | `service/esi.py; eos/saveddata/ssocharacter.py; gui/ssoLogin.py` | – n/a: ruling 2026-10-03: ESI login/character fetch is a web-frontend job (owner eve4, later); engine input = character.skills/security (request.rs) — have | – n/a: web frontend first; MCP may later accept the same skills input (already does: skills param) | ❌ missing: missing — web frontend, later (eve4) |
| CHR-004 | **Import character file (EVEMon/XML)**: Import skills from XML (EVEMon / API char sheet) | `service/character.py importCharacter; gui/mainMenuBar.py 'Import Character File'` | ❌ missing: - | ❌ missing: - | ❌ missing: - |
| CHR-005 | **Export skills needed (EVEMon plan)**: Export required skills for fit as EVEMon plan | `eos/saveddata/character.py / service/character.py exportText; mainMenu 'Export Skills Needed'` | 🟡 partial: MISSING_SKILL list | 🟡 partial: skill_requirements (list, no plan export) | 🟡 partial: missing skills list, no export |
| CHR-006 | **Character implants**: Implants on the character applied to all fits | `eos/saveddata/character.py implants` | ❌ missing: - | ❌ missing: - | ❌ missing: - |
| CHR-007 | **Security status on character**: secStatus | `character.py secStatus` | ✅ have: character security | ✅ have: schema | ✅ have: Character.tsx |
| CHR-008 | **Skill prerequisites tree / requirements view**: Item stats Requirements tab with levels | `gui/builtinItemStatsViews/itemRequirements.py` | 🟡 partial: type RPC? | ✅ have: skill_requirements | ✅ have: Market show info required skills |
| CHR-009 | **Skill planner needs (train time)**: Pyfa shows missing skills; train time via SP/attributes (planner-adjacent) | `characterEditor skill tree` | ❌ missing: no SP/train time | ❌ missing: - | 🟡 partial: train-required skills list |

### PRC: Prices

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| PRC-001 | **Market price fetch with sources**: fuzzwork, evemarketdata, evetycoon, cevemarket; price system (Jita…); caching, timeouts | `service/price.py; service/marketSources/*` | ❌ missing: offline engine (MPF A5: price → UNKNOWN_METHOD) | ❌ missing: src/ grep price 0 | 🟡 partial: src/data/prices.ts ESI average/adjusted only; no source/system choice |
| PRC-002 | **Price stats panel**: Ship/fittings/drones/cargo/character totals; include drones/cargo/character toggles | `gui/builtinStatsViews/priceViewFull.py; settings MarketPriceSettings` | ❌ missing: - | ❌ missing: - | ✅ have: PriceBox.tsx (e2e tools/e2e.mjs: 'fit price from ESI') |
| PRC-003 | **Price column per item**: price column | `gui/builtinViewColumns/price.py` | ❌ missing: - | ❌ missing: - | ❌ missing: no per-row price |
| PRC-004 | **Optimize fit price**: Replace items by cheaper equivalents (Fit menu) | `gui/mainFrame.py optimizeFitPrice; gui/copySelectDialog.py (multibuy OPTIMIZE_PRICES)` | ❌ missing: - | ❌ missing: optimize_fit optimizes stats, not price | ❌ missing: - |
| PRC-005 | **Price options context menu**: priceOptions (what to include) | `gui/builtinContextMenus/priceOptions.py` | ❌ missing: - | ❌ missing: - | ❌ missing: - |

### FMT: Import / export

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| FMT-EFT-001 | **EFT import/export**: EFT text incl. offline, charges, drones/fighters x N, implants, boosters, cargo, mode, mutated | `service/port/eft.py` | ✅ have: FMT 4779/4779 native+wasm; eft_parse/eft_export | ✅ have: parse_fit/export_fit | ✅ have: src/fit/formats.ts parseEft/exportEft |
| FMT-EFT-002 | **EFT config (.cfg) import**: EFT cfg file import | `service/port/eft.py importEftCfg` | ✅ have: format_import eftcfg | ❌ missing: not in parse_fit | ❌ missing: - |
| FMT-EFT-003 | **EFT export options**: Implants/mutations/cargo/loaded charges toggles (copySelectDialog) | `gui/copySelectDialog.py; port/eft.py options` | 🟡 partial: fixed export shape | 🟡 partial: - | 🟡 partial: - |
| FMT-DNA-001 | **DNA import/export (incl. alt, link)**: ship DNA string, dna alt, fitting link | `service/port/dna.py` | ✅ have: FMT; format_import dna/dna_alt/dna_link | ✅ have: parse_fit/export_fit DNA | ✅ have: parseDna/exportDna |
| FMT-ESI-001 | **ESI JSON import/export**: ESI fitting JSON (exportCharges/Implants/Boosters settings) | `service/port/esi.py` | ✅ have: format_export/import esi | 🟡 partial: lenient JSON in parse_fit, export_fit JSON (a08d943 §10) | ✅ have: exportEsi/parseEsi |
| FMT-XML-001 | **XML import/export (EVE client)**: Fittings XML, multiple fits, backup | `service/port/xml.py` | ✅ have: format_import/export xml | ❌ missing: - | ❌ missing: src/ grep xml 0 |
| FMT-MB-001 | **Multibuy export**: Multibuy text | `service/port/multibuy.py` | ✅ have: format_export multibuy | ✅ have: export_fit multibuy | ✅ have: exportMultibuy |
| FMT-SS-001 | **Ship stats (exportFitStats) text**: Stats summary export | `service/port/shipstats.py` | ✅ have: format_export shipstats | 🟡 partial: compute_fit summary (src/summary.ts) differs | ❌ missing: - |
| FMT-EFS-001 | **EFS export**: Eve Fitting Stats JSON | `service/port/efs.py` | ❌ missing: UNSUPPORTED_FORMAT efs (MPF D1) | ❌ missing: - | ❌ missing: - |
| FMT-MUTA-001 | **Mutated module text export/import**: Copy mutated module text | `service/port/muta.py; gui/builtinContextMenus/moduleMutatedExport.py` | 🟡 partial: inside EFT only (MPF D2) | 🟡 partial: inside EFT | 🟡 partial: inside EFT |
| FMT-HTML-001 | **HTML export of all fits**: Export fits to HTML (prefs path, options) | `gui/utils/exportHtml.py; mainMenu 'Export All Fittings to HTML'; HTMLExport prefs` | ❌ missing: - | ❌ missing: - | ❌ missing: - |
| FMT-AUTO-001 | **Auto-detect import, multi-fit buffers, files/folders**: importAuto, importFitFromBuffer, importFitsFromFile (threaded) | `service/port/port.py` | 🟡 partial: auto-detect one text (MPF D5) | 🟡 partial: parse_fit auto | 🟡 partial: multi-EFT paste; no file/folder |
| FMT-CLIP-001 | **Clipboard to/from**: Fit menu To/From Clipboard | `gui/mainMenuBar.py` | – n/a: host | – n/a: - | ✅ have: ImportExport.tsx clipboard |
| FMT-ADD-001 | **Additions export/import**: Export all/selection of drones/cargo/implants etc., import additions | `gui/builtinContextMenus/additionsExportAll.py, additionsExportSelection.py, additionsImport.py` | 🟡 partial: EFT sections | ❌ missing: - | ❌ missing: - |

### ESI: ESI / SSO

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| ESI-001 | **SSO login, multiple characters**: ssoLogin, Manage ESI Characters, token refresh, custom app/server settings *Owner: eve-fit-web (eve4), later; engine owns the character/skill input format.* | `gui/ssoLogin.py; service/esiAccess.py; Esi prefs` | – n/a: ruling 2026-10-03: ESI login/character fetch is a web-frontend job (owner eve4, later); engine input = character.skills/security (request.rs) — have | – n/a: web frontend first; MCP may later accept the same skills input (already does: skills param) | ❌ missing: missing — web frontend, later (eve4) |
| ESI-002 | **Browse/import ESI fittings**: List in-game fittings, import *Owner: eve-fit-web (eve4), later; engine owns the character/skill input format.* | `gui/esiFittings.py` | – n/a: ruling 2026-10-03: ESI login/character fetch is a web-frontend job (owner eve4, later); engine input = character.skills/security (request.rs) — have | – n/a: web frontend first; MCP may later accept the same skills input (already does: skills param) | ❌ missing: missing — web frontend, later (eve4) |
| ESI-003 | **Export fit to ESI / delete ESI fittings**: Upload fit, delete fitting *Owner: eve-fit-web (eve4), later; engine owns the character/skill input format.* | `gui/esiFittings.py; service/esi.py` | – n/a: ruling 2026-10-03: ESI login/character fetch is a web-frontend job (owner eve4, later); engine input = character.skills/security (request.rs) — have | – n/a: web frontend first; MCP may later accept the same skills input (already does: skills param) | ❌ missing: missing — web frontend, later (eve4) |

### DB: Persistence / fit library

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| DB-001 | **Saved fits database**: Persist fits (sqlite saveddata), rename, delete, copy | `eos/db/saveddata; service/fit.py` | ❌ missing: stateless (MPF C5) | ❌ missing: stateless | ✅ have: localStorage fit library (FitBrowser.tsx) |
| DB-002 | **Fit browser tree (by ship group/race, search)**: shipBrowser categories, race filter, fit search | `gui/shipBrowser.py; builtinShipBrowser/*` | – n/a: UI | – n/a: - | ✅ have: FitBrowser grouped by ship group |
| DB-003 | **Backup all fits / restore**: Backup all to XML | `service/port/port.py backupFits` | ❌ missing: - | ❌ missing: - | 🟡 partial: JSON library backup/restore, no e2e; no XML (a08d943 §11) |
| DB-004 | **Fit links (projected/command fits by reference)**: Projected/command fits reference saved fits; updates propagate | `eos/saveddata/fit.py projectedFits/commandFits` | 🟡 partial: inline booster_fits/projected fits (MPF F2) | 🟡 partial: inline | ✅ have: projected/booster from library |
| DB-005 | **Fit notes**: Notes pane per fit | `gui/builtinAdditionPanes/notesView.py` | – n/a: n/a | ❌ missing: src/fit.ts 'notes' are name-resolution notes, not fit notes | 🟡 partial: notes field, EFT comment export (a08d943 §2) |
| DB-006 | **Tabs / multiple open fits**: Open fits in tabs; open in new tab | `gui/mainFrame.py; fitOpenNewTab` | – n/a: UI | – n/a: - | 🟡 partial: single active fit + browser |
| DB-007 | **Undo/redo**: fitCommands undo stack | `gui/fitCommands/*` | – n/a: UI | – n/a: - | ✅ have: App.tsx undo/redo |
| DB-008 | **Saved damage patterns / target profiles / implant sets / characters**: User-editable profile libraries | `service/damagePattern.py, targetProfile.py, implantSet.py, character.py` | ❌ missing: inline only | 🟡 partial: list_presets read-only | ✅ have: Profiles.tsx custom + localStorage |
| DB-009 | **Database prefs (saveddata path, cleanup)**: Database pref view | `gui/builtinPreferenceViews/pyfaDatabasePreferences.py` | – n/a: - | – n/a: - | – n/a: - |

### PRF: Damage patterns / target profiles

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| PRF-DMG-001 | **Built-in damage patterns**: Uniform, EM/Th/Kin/Ex, NPC factions, ammo patterns | `eos/saveddata/damagePattern.py BUILTINS` | ❌ missing: numbers only (MPF C3) | ✅ have: list_presets damage | ✅ have: presets |
| PRF-DMG-002 | **Damage pattern editor / change**: patternEditor, damagePatternChange | `gui/patternEditor.py; gui/builtinContextMenus/damagePatternChange.py` | – n/a: - | 🟡 partial: pass custom numbers | ✅ have: Profiles custom |
| PRF-TGT-001 | **Built-in target profiles**: NPC/ship-class target profiles incl. infinite sig | `eos/saveddata/targetProfile.py getBuiltinList` | ❌ missing: numbers only (MPF C4) | ✅ have: list_presets target | ✅ have: presets |
| PRF-TGT-002 | **Target profile editor (resists, sig, speed, radius)**: targetProfileEditor | `gui/targetProfileEditor.py` | – n/a: - | 🟡 partial: custom numbers | ✅ have: Profiles custom |

### MKT: Market / item info

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| MKT-001 | **Market browser tree**: Market groups, meta buttons (T1/T2/faction/…), empty group modes | `gui/marketBrowser.py; builtinMarketBrowser/*` | ❌ missing: search/type only (MPF E2) | ❌ missing: search only, no market tree (a08d943 §1) | 🟡 partial: Market.tsx tree from market groups; no e2e (a08d943 §1) |
| MKT-002 | **Search with jargon/aliases**: jargon (mwd, lse, ab) | `service/jargon/*` | ❌ missing: name match only (MPF E1) | ✅ have: search_types jargon/fuzzy | 🟡 partial: EN/ZH substring search; no jargon (src/ grep jargon\|alias 0) |
| MKT-003 | **Item stats: description/traits/attributes/effects/properties**: itemDescription, itemTraits, itemAttributes, itemEffects, itemProperties | `gui/builtinItemStatsViews/*` | 🟡 partial: type RPC returns attributes/effects | ✅ have: get_type/get_ship | ✅ have: Show info |
| MKT-004 | **Item compare view**: Compare attributes across variations | `gui/builtinItemStatsViews/itemCompare.py` | ❌ missing: - | 🟡 partial: compare_fits (fits, not items) | ❌ missing: - |
| MKT-005 | **Localized item names**: Pyfa locales | `locale/; service/market.py` | 🟡 partial: EN+ZH search (MPF E3) | 🟡 partial: EN/ZH | 🟡 partial: zh switch covers names + main labels, not every string (a08d943 §13) |
| MKT-006 | **Market jump / ship jump**: itemMarketJump, shipJump | `gui/builtinContextMenus/itemMarketJump.py, shipJump.py` | – n/a: UI | – n/a: - | 🟡 partial: not found |

### UI: GUI panes, columns, menus, preferences

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| UI-PANE-001 | **Addition panes**: Drones, fighters, cargo, implants, boosters, projected, command, notes | `gui/builtinAdditionPanes/*` | – n/a: - | – n/a: - | ✅ have: Fitting.tsx panes |
| UI-PANE-002 | **Cargo add/fill ammo**: cargoAdd, cargoAddAmmo, cargoFill | `gui/builtinContextMenus/cargo*.py` | – n/a: - | – n/a: - | 🟡 partial: cargo edit; no add-all-ammo |
| UI-PANE-003 | **Item amount change/fill/remove**: itemAmountChange, itemFill, itemRemove | `gui/builtinContextMenus/item*.py` | – n/a: - | 🟡 partial: what_if | ✅ have: amount edit |
| UI-STAT-001 | **Stats panels with Minimal/Full/hidden modes**: StatView prefs per panel | `service/settings.py StatViewSettings` | – n/a: - | – n/a: - | 🟡 partial: fixed layout |
| UI-COL-001 | **Fitting view columns**: abilities, ammo, capUse, dampScanRes, droneEhp, heat, maxRange, misc, price, projectionRange, sideEffects, state, targetResists… | `gui/builtinViewColumns/*` | – n/a: - | – n/a: - | 🟡 partial: subset of columns |
| UI-PREF-001 | **Preferences**: General, ContextMenu, Engine (global reload, spool default, RAH, strict skill levels), StatView, Market, Network/proxy, HTML export, ESI, Update, Logging, Database | `gui/builtinPreferenceViews/*` | 🟡 partial: engine options per request | 🟡 partial: options passthrough | 🟡 partial: EngineSettings.tsx reload/spool/RAH |
| UI-CTX-001 | **Context menu catalog (51 actions)**: See builtinContextMenus; most mapped to other items | `gui/builtinContextMenus/*` | – n/a: - | – n/a: - | 🟡 partial: subset |
| UI-MISC-001 | **Fleet/command pane, command link add**: commandLinkAdd | `gui/builtinContextMenus/commandLinkAdd.py` | – n/a: - | – n/a: - | ✅ have: fleet booster fits |
| UI-STAT-RES | **Resources panel**: CPU, PG, calibration, drone bandwidth/bay, fighter bay, fighter squadrons active, drones active, turret/launcher hardpoints, cargo | `gui/builtinStatsViews/resourcesViewFull.py` | ✅ have: stats.rs resources/hardpoints/cargo | ✅ have: compute_fit summary (a08d943 §6) | ✅ have: Stats.tsx Resources |
| UI-STAT-RST | **Resistances panel**: Resist % per layer and type, HP per layer, EHP, raw/effective toggle, damage pattern selector, resist multiplier | `gui/builtinStatsViews/resistancesViewFull.py` | ✅ have: stats.rs resonance/hp/ehp | ✅ have: compute_fit detail full | ✅ have: Stats.tsx Defense |
| UI-STAT-RCH | **Recharge panel**: Passive shield recharge, active shield boost, armor/hull repair: reinforced vs sustained | `gui/builtinStatsViews/rechargeViewFull.py` | ✅ have: stats.rs tank raw/sustained | ✅ have: compute_fit | ✅ have: Stats.tsx |
| UI-STAT-FP | **Firepower panel**: Weapon/drone/total DPS and volley, spool tooltip (min/current/max), toggle to mining yield | `gui/builtinStatsViews/firepowerViewFull.py` | 🟡 partial: no min/max spool pair, no mining (ENG-OFF-005/006) | 🟡 partial: same | 🟡 partial: same |
| UI-STAT-CAP | **Capacitor panel**: Total GJ, lasts / stable %, recharge time, delta, extra stats (use/regen) | `gui/builtinStatsViews/capacitorViewFull.py` | ✅ have: stats.rs capacitor (stable_percent, depletes_in_s, delta_gj_s) | ✅ have: compute_fit | ✅ have: Stats.tsx Capacitor |
| UI-STAT-TGT | **Targeting & misc panel**: Targets, range, scan res, sensor str/type, drone range, speed, align, signature, warp speed, max warp distance, warp core strength, probe size, cargo + all special holds, lock times per class, jam chance | `gui/builtinStatsViews/targetingMiscViewMinimal.py` | 🟡 partial: special holds / lock-time table partial (ENG-TGT-002/006) | 🟡 partial: same | 🟡 partial: same |
| UI-STAT-OUT | **Outgoing (Remote reps) panel**: Shield/armor/hull/cap restored per s | `gui/builtinStatsViews/outgoingViewFull.py, outgoingViewMinimal.py` | ❌ missing: ENG-PROJ-006 | ❌ missing: - | 🟡 partial: Remote assistance section depends on backend |
| UI-STAT-MIN | **Mining yield panel**: Mining m3/s total, modules/drones | `gui/builtinStatsViews/miningyieldViewFull.py` | ❌ missing: ENG-OFF-006 | ❌ missing: - | 🟡 partial: backend-dependent |
| UI-STAT-PRC | **Price panel**: Ship, fittings, drones, cargo, character, total | `gui/builtinStatsViews/priceViewFull.py, priceViewMinimal.py` | ❌ missing: PRC-002 | ❌ missing: - | ✅ have: PriceBox.tsx |
| UI-STAT-BMB | **Bombing panel**: Bombs to kill per bomb type, covert-ops skill level | `gui/builtinStatsViews/bombingViewFull.py` | ❌ missing: ENG-OFF-007 | ❌ missing: - | ❌ missing: - |
| UI-PREF-GEN | **General preferences**: global character / damage pattern, character implants default, color by slot, rack separation/labels, open in new page, reopen fits on startup, ammo change all, additions tab quantity, mutated names, language, theme | `gui/builtinPreferenceViews/pyfaGeneralPreferences.py` | – n/a: - | – n/a: - | 🟡 partial: language switch; others not present |
| UI-PREF-ENG | **Engine preferences**: Global default spool %, strict skill level requirements, global force reload, RAH | `gui/builtinPreferenceViews/pyfaEnginePreferences.py` | 🟡 partial: options.default_spool, factor_reload, rah; strict skill mode not separate (MISSING_SKILL always reported) | ✅ have: options passthrough | 🟡 partial: EngineSettings.tsx; no e2e (a08d943) |
| UI-PREF-MKT | **Market & price preferences**: Default price source/system, totals include drones/cargo/implants, meta button modes, search delay, market shortcuts | `gui/builtinPreferenceViews/pyfaMarketPreferences.py` | ❌ missing: - | ❌ missing: - | ❌ missing: - |
| UI-PREF-CTX | **Context menu preferences**: Enable/disable individual context menus | `gui/builtinPreferenceViews/pyfaContextMenuPreferences.py` | – n/a: - | – n/a: - | – n/a: no configurable menus |
| UI-PREF-STV | **Stat view preferences**: Per-panel hidden/minimal/full | `gui/builtinPreferenceViews/pyfaStatViewPreferences.py` | – n/a: - | – n/a: - | ❌ missing: - |
| UI-PREF-DB | **Database preferences**: Paths, delete all damage/target profiles, delete prices | `gui/builtinPreferenceViews/pyfaDatabasePreferences.py` | – n/a: - | – n/a: - | 🟡 partial: localStorage; clear via browser |
| UI-PREF-ESI | **ESI preferences**: SSO server, auto-login, token expiry, export charges/implants/boosters *Owner: eve-fit-web (eve4), later; engine owns the character/skill input format.* | `gui/builtinPreferenceViews/pyfaEsiPreferences.py; service/settings.py EsiSettings` | – n/a: engine owns only the character/skill input format | – n/a: - | ❌ missing: web frontend, later (eve4) |
| UI-PREF-NET | **Network/Update/Logging/HTML prefs**: Proxy, update channel, debug logs, HTML export path/minimal | `gui/builtinPreferenceViews/pyfaNetworkPreferences.py, pyfaUpdatePreferences.py, pyfaLoggingPreferences.py, pyfaHTMLExportPreferences.py` | – n/a: - | – n/a: - | – n/a: browser handles network |

### SVC: Services / misc

| id | feature | Pyfa source | F | MCP | WEB |
|---|---|---|---|---|---|
| SVC-001 | **Update check**: Check GitHub releases (prerelease setting) | `service/update.py; gui/updateDialog.py` | – n/a: n/a for engine | – n/a: engine_info versions | – n/a: web deploys |
| SVC-002 | **Network/proxy settings**: Proxy modes | `service/network.py; Network prefs` | – n/a: - | – n/a: - | – n/a: browser |
| SVC-003 | **Logging / dev tools**: Logging prefs, widget inspect | `gui/builtinPreferenceViews/pyfaLoggingPreferences.py; gui/devTools.py` | 🟡 partial: stderr | 🟡 partial: metrics resource | – n/a: - |
| SVC-004 | **Data update (new SDE)**: Pyfa ships with SDE dumps per release | `eos/db/gamedata; scripts/` | 🟡 partial: dataset compiled in at build (eve-sde-pipeline) | ✅ have: eve://dataset/meta | ✅ have: dataset.ts |
| SVC-005 | **Effect/attribute conversions**: service/conversions (renamed items) | `service/conversions/*` | ❌ missing: no rename map for imports | ❌ missing: - | ❌ missing: - |

<!-- END GENERATED -->
