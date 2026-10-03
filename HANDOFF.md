# HANDOFF / STATUS — EVE fitting project (EX-CT)

Anyone resuming work starts here. **Rule: commit + push after every small step; unfinished work goes to a `wip/*` branch. Update your own section below after each push.** Coordinator: eve. Last full update: 2026-10-03 14:25 CST.

## Standing decisions (user, 2026-10-03)
- Mainline engine: Rust, EX-CT/eve-dogma (built on variant F). J (C++) kept only as backup/speed reference (eve-dogma branch `j-backup`, lab tags `j-backup-2026-10-03`, `j-graphs-wip-2026-10-03`). All other engine lines stopped.
- **Feature completeness and correct calculation first; coverage may only grow, never shrink; optimization after.** Must cover ALL Pyfa features (docs/19); extras (beyond Pyfa) counted separately.
- Pyfa parity wins: Pyfa special behaviours are implemented as observed (documented, no copied code). Only proven Pyfa-bundled-data vs SDE drift goes to eve-sde-pipeline docs/pyfa-data-drift.json (eve approves).
- Engine/server/MCP never call ESI. ESI login/skills sync is web-client only (later). Engine defines character/skills input.
- Formats (EFT/DNA/XML/ESI/EFS/HTML/Pyfa saved fits) live in crate `crates/eve-fit-formats` inside the eve-dogma workspace (no separate repo); engine takes structured JSON only.
- Optimizer: v1 exists (eve-dogma 20aa425) but is **deprioritized** — keep as is, no regressions, no further work (AI will optimize via MCP).
- No-regression gate: bench pending-1.11 `tools/check_no_regress.py` + `baselines/f.json` (baseline 2da8150); pass counts, passed case ids and docs/19 have-counts may never drop.

## New core requirements (2026-10-03 14:14–14:23) — see docs/20, docs/21, docs/22
1. **Batch API (extra, high priority):** many fits, or base fit + variant patches (modules, ammo, states, skills, implants, boosters, fleet boosts, projected, targets), cartesian product and parameter sweeps (with a combination cap); shared data/caches, deterministic; output `fields` selection, delta vs baseline (abs/%), `sort_by`, `filter`, top-N, per-variant id/label, per-item errors. Entry points: Rust lib, RPC/CLI `eve-fit batch`, WASM, MCP `compute_batch`. Test: batch == one-by-one, in gate.
2. **Prices are engine core:** request input `price_overrides` by type_id / market_group_id (incl. children) / group_id / category_id; value fixed (0 for self-made/stock), or multiplier. Precedence: request override (type > market group, most specific first > group > category) > injected prices (`prices` / `--prices`) > embedded release Jita snapshot. Output total + per-item breakdown with `source` per price and missing-price list. Batch variants may carry their own overrides; sort/filter by price.
3. **Embedded SDE:** each release embeds its SDE build (no external download); report sde_build + hash; optional update switch `--sde` / `sde_override`. Engine never goes online.
4. **Frozen prices:** each release embeds a price snapshot. Rule: Jita 4-4 (60003760) sell orders only; drop orders with fewer than `min_units` units (filter by unit count, not by a percentage of orders); p0 = lowest remaining; price = unit-weighted average of orders within [p0, p0×1.05]; both parameters configurable.
5. **Market price updater:** separate tool, repo EX-CT/eve-market-prices (MIT), pluggable sources (ESI orders, Fuzzwork, more later), emits versioned snapshot files (schema in docs/22).

## Owners and queues
### F bot — EX-CT/eve-dogma (main 20aa425 at 13:56, CI green)
Queue: (1) eve-fit-formats unit tests + CI; (2) remaining 26 effects/ failures, Pyfa behaviour; (3) batch API + price overrides/breakdown (docs/20/21/22 contract first); (4) embedded SDE + price snapshot + injection; (5) close docs/19 F-column missing (26) and partial (24) items incl. alpha clones; wire no-regress gate into CI.
Scores 20aa425: core 1.10 339/339, effects 2352/2378, ext 116/116 (pending-1.11 ext 187/187+ fehp 15/15), cap 150, mutated 93, formats 4779, graphs 178 (0.3: 192).
_Status:_ see per-bot section "variant F / eve-dogma" below.

### eve3 — EX-CT/eve-dogma-bench (pending-1.11), docs/19 owner
Queue: (1) Pyfa-generated cases for the 26 missing + full cases for partial items; (2) batch-suite incl. price-override cases; (3) docs/22 cases (embedded SDE version/hash, price rule pure-function cases, injection precedence); docs/19 extras ENG-BATCH-001, ENG-PRICE-001. optimizer-bench parked as WIP (13cb0d9..d48bbab).
_Status:_ eve3: add a section "eve3 / bench" below.

### eve4 — eve-fit-web, eve-fit-mcp, eve-sde-pipeline, eve-market-prices
Queue: web fit library + Pyfa saved-fit import (with engine bump to latest and showing new outputs) ‖ eve-market-prices; then MCP `compute_batch` + price_overrides pass-through (right after F's contract); web "my prices" setting; then mcp/web test gaps (inventory/mcp-web-gaps.md). MCP has no optimize tool. ESI login last.
_Status:_ see per-bot section "eve4 / eve-fit-web" below.

## docs/19 counts (have/partial/missing/n/a), 200 items
- f (2da8150): 104/24/26/46 · mcp: see latest docs/19 (07341e3 upgraded 51 engine-backed items) · web (40e6044): 84/81/29/6 · formats: 10/1/2/1 · extras: UI-CMP-001 (+ENG-BATCH-001, ENG-PRICE-001 pending)

---
# Per-bot status sections (each bot updates only its own)

## variant F / eve-dogma (executor bot; updated 2026-10-03 14:25 CST)

### Current commits
- EX-CT/eve-dogma main: **d990818** (CI run in progress at handoff; previous green: 37fcaa1, 20aa425; 1ebaa1f failed only because the new gate's clone dir `bench` collided, fixed in d990818).
- EX-CT/eve-fit-docs main: 44891c8 docs/22 (mine), plus others' later commits.
- EX-CT/eve-dogma-bench: nothing of mine on main; parked branch **wip/stats-ext-suite** = 07beb42 (old stats-ext mining cases, superseded by eve3's ext suite; keep for reference only).
- No other wip branches: all eve-dogma work is on main.

### Done (all on eve-dogma main unless noted)
- 37fcaa1 eve-fit-formats unit tests (`crates/eve-fit-formats/tests/roundtrip.rs`, 18 tests: per-format import→export / export→import round trips + error paths), CI step added.
- 8011d64 + d990818 CI blocking no-regression gate (eve3: bench pending-1.11 `tools/run_all_suites.sh` + `check_no_regress.py --baseline baselines/f.json`), cloned to `_nrg_bench`.
- 1ebaa1f effects 2352 → **2378/2378** (Pyfa quirks Q1–Q12 in codegen `pyfa_mod_quirk` + engine `sp_*`, documented in DESIGN.md "Pyfa parity quirks"); gate.json effects floor 2378; round-1 shas updated (only one structure fit changed, Q8).
  Local no-regress run: core 339, ext 207/239 (baseline 204), ext_rpc 0/31, effects 2378 (baseline 2352), graphs 192, cap 150, mutated 93, formats 4779 → eve3 can raise the baseline.
  Data-drift candidates found (not excluded, for eve): remoteCapacitorImpedance(+Bonus) absent in Pyfa eve.db on siege/triage/industrial cores + some hulls; Proteus maxSubSystems 4 (Pyfa) vs 5 (SDE). Neither changes a scored value.
- 20aa425 optimizer v1 (crate eve-optimizer, CLI `optimize`, RPC `optimize` in serve-stdio + eve-wasm); frozen at v1 per user; tests must keep passing.
- Earlier today: mining, outgoing RR/cap, drone/fighter EHP, bombing, heat, overrides global, validation (SHIP_RESTRICTION capital, module_index, MISSING_SKILL skill_type_id/recursive), fighter EHP 15/15 confirmed locally on pending-1.11.
- eve-fit-docs 44891c8 **docs/22-embedded-sde-and-prices.md** (edp pack v1, provenance key, `--sde`/`sde_override`, price precedence, `eve-price-snapshot` v1 schema for eve4).

### In progress (nothing uncommitted)
- **Batch contract draft** (new doc, planned path `docs/23-batch-api-and-prices.md`): not written yet. Must cover: RPC method `batch` served by `eve-fit serve-stdio` and the eve-wasm `rpc` export (+ Rust lib); forms many-fits / base+variants / cartesian / sweeps; fields, deltas, sort, filter, top-N; per-variant `price_overrides`; sort/filter by price.
- **price_overrides** (eve 14:22) must also be written into docs/20, 21, 22: request-level input next to fit/skills; targets type_id > market_group_id (deepest wins, includes children) > group_id > category_id; value fixed (0 allowed) or multiplier; layers: request overrides > injected (`prices` / `--prices`) > embedded snapshot; output `price` block with total, per-section and per-item lines, per-price source (`override:type|override:market_group|override:group|override:category|injected|snapshot`) + snapshot time, unpriced items listed. Market-group tree needs dataset r5+ (`market_groups` with `parent`; current build dataset 3569502 r1 lacks it).
  docs/22 currently says request `prices {mode, isk}`; reconcile it with `price_overrides`. Proposed multiplier rule: a multiplier applies to the price from the next lower layer that has one (fixed overrides stop the chain); within one layer the most specific target wins, ties broken by the lower id.

### Next steps (order from eve/user)
1. ~~formats tests~~ → 2. ~~no-regress gate~~ → 3. batch contract + prices (docs/23 + docs/20/21/22 edits) → 4. ~~26 effects~~ → 5. batch API implementation incl. price input/output (no embedded snapshot yet: unpriced = missing) in Rust lib, RPC/CLI, WASM → 6. docs/22 embedded SDE pack + snapshot implementation → 7. docs/19 F-column missing (26) / partial (24) items incl. alpha clones, plus ext new missing-feature cases (37, 5 pass) and ext_rpc (31, 0 pass); update docs/19 F column per item.
- Pending external: eve3 1.11 tag → switch CI bench tag and make the inventory step blocking.

### Key context for a successor
- Repo paths on the box: eve-dogma `/workspace/exct-eve/eve-dogma-main`, docs `/workspace/exct-eve/fit-docs-main`, bench `/workspace/exct-eve/bench-main`, pending-1.11 clone `/tmp/nrg/bench`, Pyfa reference (behaviour only, never copy GPL code) `/workspace/exct-eve/ref/pyfa` incl. `eve.db`.
- Build: `export EVE_DOGMA_DATASET=/workspace/exct-eve/data/dataset-3569502.json.gz; cargo build --release --offline -p eve-cli`. Gate: `BENCH=/workspace/exct-eve/bench-main SUITES_DIR=/tmp/ci-suites2 ci/run_suites.sh native target/release/eve-fit`; no-regress: `/tmp/nrg/bench/tools/run_all_suites.sh target/release/eve-fit out && python3 /tmp/nrg/bench/tools/check_no_regress.py --baseline /tmp/nrg/bench/baselines/f.json --run-dir out`.
- New output keys: add to `ci/round1-new-keys.txt` and update `ci/round1.sha256`; any change to existing fields must be intended and re-baselined in `ci/round1-base.sha256`.
- Pyfa-parity rule (eve): match Pyfa including hand-written handler behaviour; only proven Pyfa-data vs SDE drift is an exception, listed for eve, never excluded unilaterally.
- Commit identity `-c user.name=EXCT-Bot -c user.email=bot@exct.invalid`; never force-push; unfinished work → `wip/*`.

## eve4 / eve-fit-web (executor bot; updated 2026-10-03 15:25 CST)

### Current commits
- EX-CT/eve-fit-web main: **b2b57e5** (all work pushed; no uncommitted changes; **no wip branches**). Pages run 37103998542 (b2b57e5) is pending; a70469c run 37103721084 is in progress.
  The last green run was 37102734646 (6ac02ba): unit, e2e wasm-worker 82/82, http 82/82, ts-worker 72/72, bench 331/331, graphs 178/178, ext 202/202, effects 2353/2378.
- Live (https://ex-ct.github.io/eve-fit-web/) checked at f97501a: e2e wasm-worker 76/76, ts-worker 66/66.

### Done
- **Milestone 4 (A), fit library + Pyfa import:** b75923b (formats layer: Pyfa saveddata.db via sql.js, library import/export, fixture made by Pyfa 1d9f72b, data only), d670e64 (IndexedDB store + migration from localStorage, folders/tags/search/rename/duplicate/delete/export/backup UI), 342b813 + f97501a (e2e: pyfa-db-*, library-*, dna-import = FMT-DNA-001, library-reload-persistence = DB-001, library-migration; docs/test-ids.md).
- **B, F pin bump:** a972c4e engines.lock F -> EX-CT/eve-dogma 20aa425 (main, after 2da8150). 692ff88 UI: mining, outgoing reps, bombs to kill, overheat burnout, drone/fighter EHP, violation labels (zh-CN). 1ca5683 CI suites web-ext (bench pending-1.11 @1a8be05 ext 202/202) and web-effects (>= EFFECTS_MIN 2353) in the browser build. 6ac02ba e2e ids mining-yield, outgoing-reps, bombing-table, overheat-burnout, drone-ehp, validation-problems. 7854503 docs/test-ids.md, 3eb47ba README.
- 070dfc3 (done before C was deferred): drag-and-drop rack position (moveModule), mutaplasmid roll range, Import from clipboard; unit tests web.unit.module-move, charges-valid-only.

- **Full bench suite set in the browser (parent queue item):** 2dce909 CI step "Bench pending-1.11 full suite set". It clones eve-dogma-bench @ `BENCH_SUITES_SHA` (pending-1.11 head c2229b2, in engines.lock), runs `tools/run_all_suites.sh tools/browser-engine.mjs` against the built wasm-worker in headless Chrome, gates the deploy on `check_no_regress.py --baseline baselines/f.json`, and uploads artifact `bench-suites-wasm-worker`. afb9e83 tools/browser-engine.mjs (eve-fit-compatible CLI: calc / batch / serve-stdio), ef38516 browser-rpc --http, bf58942 (stdout flush fix), a70469c (shipstats: full_precision stats as stats_json, byte-exact; browser-rpc computes shipstats stats like serve-stdio).
  Local result at a70469c: core 339/339, ext 207/239, ext_rpc 0/54, batch 0/44 (the methods are not in the browser; baseline 0), effects 2353/2378, graphs 192/192, cap 150/150, mutated 93/93, formats 4779/4779. Result: no regression.

### Next steps
0. Confirm Pages run 37103998542 (b2b57e5) is green, read its step summary, then do the live e2e.
1. Confirm the 3eb47ba Pages run is green (unit, e2e on all backends, bench 331/331, graphs 178/178, ext 202/202, effects) and run the live e2e (wasm-worker, ts-worker) against the deployed site.
2. EFFECTS_MIN can go to 2378 once the pin moves to eve-dogma d990818+ (effects 2378/2378).
3. Deferred (C): tests for the 27 docs/19 web items downgraded have->partial. Not started beyond 070dfc3. A draft of the checks (target profiles, damage pattern editor, fleet command fit, addition panes, show-info traits / required skills, probe size, MWD, utility modules, clipboard, dataset = engine sha) was written locally and dropped from the commit; redo from inventory/mcp-web-gaps.md.
4. Prices: leave the web price box alone. Engine-computed prices (price_overrides, docs/22) and a local "my prices" setting come later.

### Key context
- Repo: /workspace/exct-eve/eve-fit-web. Local preview: `npm run build && npx vite preview --port 4180`, then `node tools/e2e.mjs http://127.0.0.1:4180/eve-fit-web/ wasm-worker|ts-worker`. For http: `node tools/engine-bridge.mjs --stdio "<eve-fit> serve-stdio" --port <free port>`. Other agents' bridges hold 8787 to 8799.
- The F build for 20aa425 is in /tmp/ed20: wasm in target/wasm32-unknown-unknown/release-small, native eve-fit in target/release. It has been copied to public/engines/f.
- The Pyfa fixture generator (GPL, outside the repo) is in /workspace/pyfa-db-fixture.
- The formats layer drops what Pyfa would not fit (capital modules, extra slots, wrong charges), so illegal-fit tests add those items through the market.

## eve4 / eve-fit-mcp / eve-market-prices (executor bot; updated 2026-10-03 14:55 CST)

### Current commits
- EX-CT/eve-fit-mcp main: **e7b794a** = release **v0.3.1** (release run 37103909901 success; CI 37103809811 green: engine + test 20/22). No wip branches, nothing uncommitted.
- EX-CT/eve-market-prices main: **1a34266** (CI 37103816237 green). wip/docs22-schema is merged (same commit) and can be deleted. First snapshot release: `prices-jita44-20261003T063856Z` (snapshot run 37103826084; 4910 priced, 2836 missing, sha256:3dd62f6c…).

### Done
- eve-fit-mcp fixes for eve3's mcp_batch findings (ffc642a, schemas e7b794a). Bugs 1 (empty nested booster_fits) and 2 (projected fighter quantity) were already fixed in 70d463b and are now pinned by tests. Bug 3: tool errors are "Error: CODE: msg" (engine codes verbatim; MCP input errors BAD_REQUEST; unknown types UNKNOWN_TYPE). compute_graph passes explicit x.values / y / resist_mode through to the engine.
  mcp_batch 8c6b93d -> e7b794a: core 322 -> 339/339; ext 201 -> 207/239 (= engine direct); graphs 176 -> 189/192. err_missing_x, err_missing_x_values and err_missing_y still fail because compute_graph defaults omitted x/y on purpose; suggest marking them mcp n/a.
- eve-market-prices: TS library + CLI with no dependencies. Rule jita_sell_band_weighted v1 (min_units default 10, band 0.05, both configurable), ESI and Fuzzwork sources, eve-price-snapshot v1 per docs/22 §4, JSON Schema, 26 tests, non-gating live smoke, daily snapshot release workflow (03:17 UTC + manual). docs/22 gaps are listed in the README (hash number form `1240000` vs `1240000.0`, key order, band_max, sde_build required, aggregate clamp).

### Paused
- eve-fit-mcp step 2 (test gaps): stopped before updating docs/test-ids.md/json for the new ids and the README mcp-bench section. The v0.4.0 release is not done.

### Next steps
1. After F's docs/23 lands: MCP compute_batch plus price_overrides / prices pass-through, with no pricing math in MCP.
2. Resume step 2 (docs/test-ids, README), then v0.4.0.

## eve3 / bench (executor bot; updated 2026-10-03 15:15 CST)

### batch-suite, no-regress gate, optimizer-bench (shelved at d48bbab, score.py not started) — updated 2026-10-03 14:46 CST
**Current commits** (everything pushed)
- eve-dogma-bench pending-1.11:
  - Gate + baseline: `98419df`, `b3a0957`, `07245e8`, `c2229b2`, `88ed590`, `32981f5`.
    - `baselines/f.json` = eve-dogma d990818: core 339, ext 208/239, ext_rpc 0/54, batch 0, effects 2378, graphs
      192, cap 150, mutated 93, formats 4779.
  - batch-suite: `a00620f`, `97dc60f` (docs/23 shape), `89f5853`, `30a47bb`, `93853b0` (14 price_* cases), `d4ed730` (eve's docs/23 price rulings applied).
    `dfb4f49`: 34 gap cases (gap 15, gap_error 12, calc_price 6, calc_price_embedded 1) + `batch/data/` price files.
    92 cases; self-test 92/92; d990818 0/92 (no `batch` method; calc has no price block / no `--prices`).
  - optimizer-bench WIP: `13cb0d9`..`d48bbab`.
- wip branches: `wip/eve3-batch-prices` (already merged).

**In progress (eve's current task)**
0. Align batch/ with eve's docs/23 rulings:
   - multiplier = 1 and base_source = source when no multiplier applies;
   - L3 = request `prices` (the fit's own table wins over the batch-wide one);
   - L4 = `--prices` file (labelled injected) or the embedded snapshot.
1. Cover the gaps: `in` / `not_null` filters, delta_ref, numeric sweeps, swap_type, BATCH_TOO_LARGE,
   BAD_PRICE_OVERRIDE, `--prices` files, calc's own price block, use_snapshot, snapshot prices.
2. docs/22 suites: embedded SDE version/hash and `--sde`; the price rule on synthetic order books; injection
   precedence. Then add them to run_all_suites and the gate.

Steps 0 (`d4ed730`) and 1 (`dfb4f49`) done. **Next step:** 2 (docs/22 suites).

### docs/19 + missing/partial cases (updated 2026-10-03 15:00 CST)
**Current commits:** eve-dogma-bench pending-1.11: 443ee69, 5b6051c, d7ba4d9, 113415b, 5dc739d, a319f0f, 11993f5, 583f912, b34ebb9 (tip b34ebb9). eve-fit-docs: ef6cdb6, 07341e3, eccb194, b6b8a6d, 295694a. Nothing uncommitted; **no wip branches**.

**Progress**
- 26 f-missing items: 16 Pyfa-generatable with 68 cases. `ext:` brdc_ (ENG-MISC-004), cimp_ (ENG-IMP-002, CHR-006), alpha_ (ENG-CORE-009), dpb_ (PRF-DMG-001), tpb_ (PRF-TGT-001), src_ (ENG-CORE-007), dep_ (ENG-CORE-008). `ext-rpc:` var_, cmp_, mkt_, srch_, isets_, evemon_, names_, backup_ (ENG-MOD-013, MKT-004, MKT-001, MKT-002, ENG-IMP-005, CHR-004, SVC-005, DB-003). Not generatable: CHR-009, PRC-001..005, UI-STAT-PRC, UI-PREF-MKT, DB-001, DB-008. Oracle opt-in `ORACLE_EXTRA=drafts,sources` + `oracle/pyfa_lookup.py` (default output byte-identical). Draft fields: CONTRACT.md "Draft 1.11: missing-f".
- F d990818 (binary /workspace/exct-eve/bin/eve-fit-d990818): ext 208/239 (breacher_dc 4/4, char_implants 2/6, others 0), rpc 0/54, effects 2378/2378. Flipped f: ENG-MISC-004 missing -> have, ENG-CORE-003 partial -> have. Results: bench results-1.11/ED-d990818.md.
- f partial: `ext-rpc:type_*` (23) for MKT-003 / ENG-SHIP-006 / CHR-008 (F 0/23); other f partials are implementation gaps.
- mcp partial: `tools/mcp_batch.py` (bench suites through compute_fit / compute_graph), suites mcp-bench/-ext/-ext-unit/-cap/-mut/-graphs; 59 mcp items partial -> have (eve-fit-mcp 8c6b93d + F 2da8150). MCP bugs: nested empty `booster_fits: []` rejected (fit.ts:209), projected fighter quantity defaults to 1 (fit.ts:202).
- check_inventory: 0 problems.

**Next step:** (1) re-run mcp-* on the MCP's current main (engine bumped by eve4) and flip the 6 remaining engine items when fixed; (2) web partials need a web-side runner (wasm in headless Chrome, eve-fit-web CI); (3) re-score new F commits with `ext/tools/score.py` + `ext/tools/score_rpc.py` and flip f items at full pass.
**wip branches:** none.

