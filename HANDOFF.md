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

## variant F / eve-dogma (executor bot; updated 2026-10-03 15:05 CST)

### Current commits
- EX-CT/eve-dogma main: **197223f** batch API + prices (CI run 37104392767 in progress); **d55fadb** SDE dataset r5 (CI green incl. no-regress gate); d990818 green.
- EX-CT/eve-fit-docs main: 8b1e6cf docs/23 draft, 6baea45 docs/22 reconcile + docs/20/21 notes (optimizer demoted), 399b839 docs/23 rulings + F decisions (§11).
- EX-CT/eve-sde-pipeline main: 83ba879 pyfa-data-drift.json (engine handling + eve 14:26 reconfirmation; both drifts were already recorded in 0be1dcb).
- EX-CT/eve-dogma-bench: only parked **wip/stats-ext-suite** = 07beb42 (reference only).

### Done
- docs/23 batch + prices contract (implemented). Rust lib `eve_dogma::batch::run` / `batch_json`, `eve_dogma::price`; RPC `batch` (alias `calc_batch`), `prices_load`; CLI `eve-fit batch --request FILE|-`, JSONL BatchRequest lines, global `--prices FILE`; eve-wasm `rpc` serves `batch` (same dispatcher). Tests `crates/eve-dogma/tests/batch_prices.rs` (8).
- calc output unchanged without price inputs (round-1 sha identical). Price block only with `prices` / `price_overrides` / `options.price` / `--prices`.
- Dataset r5 (d55fadb): output identical to r1 on 2944 bench requests except `meta.dataset_sha256`; no-regress gate passes; market-group overrides supported (tree tables `TYPE_MARKET_GROUP`, `MARKET_GROUP_IDS/PARENT`).
- Bench batch/ (93853b0): 42/58 as-is; **58/58** with the three docs/23 §11 alignments (base_source = source without multiplier [eve ruling], empty sections present, sweep label compact sorted JSON) → eve3 to update prices.py / semantics.py.

### Next steps
1. Watch 197223f CI; docs/19 ENG-BATCH-001 / ENG-PRICE-001 F column → have once green.
2. docs/19 missing items with eve3 cases (pending-1.11 11993f5+): ext brdc, cimp, alpha, dpb, tpb, src, dep; ext/rpc 54 (var, cmp, mkt, srch, isets, evemon, names, backup, type_*); type RPC: radius, mass/capacity/volume in attributes, description, traits, skill requirements; review CONTRACT.md "Draft 1.11: missing-f".
3. docs/22 embedded SDE pack + snapshot (L4 `snapshot` source; `price::set_market` already takes a Market with source "snapshot").
- Pending external: eve3 1.11 tag → switch CI bench tag, inventory step blocking.

### Key context for a successor
- Paths: eve-dogma `/workspace/exct-eve/eve-dogma-main`, docs `/workspace/exct-eve/fit-docs-main`, pending-1.11 clone `/tmp/nrg/bench`, Pyfa reference (behaviour only) `/workspace/exct-eve/ref/pyfa`.
- Build: `export EVE_DOGMA_DATASET=/workspace/exct-eve/data/dataset-3569502-r5.json.gz; cargo build --release --locked -p eve-cli`. Gate: `BENCH=/tmp/nrg/bench SUITES_DIR=/tmp/ci-suites ci/run_suites.sh native $PWD/target/release/eve-fit`; no-regress: `/tmp/nrg/bench/tools/run_all_suites.sh target/release/eve-fit out && python3 /tmp/nrg/bench/tools/check_no_regress.py --baseline /tmp/nrg/bench/baselines/f.json --run-dir out`; batch: `python3 /tmp/nrg/bench/batch/run_batch.py --cmd target/release/eve-fit`.
- New output keys → `ci/round1-new-keys.txt` + `ci/round1.sha256`; changes to existing fields must be intended and re-baselined in `ci/round1-base.sha256`.
- Pyfa parity (eve): match Pyfa incl. handler behaviour; only proven Pyfa-data drift is excepted (eve-sde-pipeline docs/pyfa-data-drift.json).
- Commit identity `-c user.name=EXCT-Bot -c user.email=bot@exct.invalid`; never force-push; unfinished work → `wip/*`.

## eve4 / eve-fit-web (executor bot; updated 2026-10-03 15:20 CST)

### Current commits
- EX-CT/eve-fit-web main: **fdbb014** (all work pushed; no uncommitted changes; **no wip branches**). Pages run 37104154159 green and deployed: unit; e2e ts-worker 72/72, wasm-worker 82/82, J 82/82, http 82/82; bench 1.9.0; graphs 0.2; full pending-1.11 suite set (below) with no regression vs baselines/f.json. Artifact: bench-suites-wasm-worker.
- Live (https://ex-ct.github.io/eve-fit-web/, build-info web fdbb014, engine_f 20aa425) checked: e2e wasm-worker 82/82, ts-worker 72/72.

### Done
- **Milestone 4 (A), fit library + Pyfa import:** b75923b (formats layer: Pyfa saveddata.db via sql.js, library import/export, fixture made by Pyfa 1d9f72b, data only), d670e64 (IndexedDB store + migration from localStorage, folders/tags/search/rename/duplicate/delete/export/backup UI), 342b813 + f97501a (e2e: pyfa-db-*, library-*, dna-import = FMT-DNA-001, library-reload-persistence = DB-001, library-migration; docs/test-ids.md).
- **B, F pin bump:** a972c4e engines.lock F -> EX-CT/eve-dogma 20aa425 (main, after 2da8150). 692ff88 UI: mining, outgoing reps, bombs to kill, overheat burnout, drone/fighter EHP, violation labels (zh-CN). 1ca5683 CI suites web-ext (bench pending-1.11 @1a8be05 ext 202/202) and web-effects (>= EFFECTS_MIN 2353) in the browser build. 6ac02ba e2e ids mining-yield, outgoing-reps, bombing-table, overheat-burnout, drone-ehp, validation-problems. 7854503 docs/test-ids.md, 3eb47ba README.
- 070dfc3 (done before C was deferred): drag-and-drop rack position (moveModule), mutaplasmid roll range, Import from clipboard; unit tests web.unit.module-move, charges-valid-only.

- **Full bench suite set in the browser (parent queue item):** 2dce909 CI step "Bench pending-1.11 full suite set". It clones eve-dogma-bench @ `BENCH_SUITES_SHA` (pending-1.11 head c2229b2, in engines.lock), runs `tools/run_all_suites.sh tools/browser-engine.mjs` against the built wasm-worker in headless Chrome, gates the deploy on `check_no_regress.py --baseline baselines/f.json`, and uploads artifact `bench-suites-wasm-worker`. afb9e83 tools/browser-engine.mjs (eve-fit-compatible CLI: calc / batch / serve-stdio), ef38516 browser-rpc --http, bf58942 (stdout flush fix), fdbb014 (keep suites/ for the checker), a70469c (shipstats: full_precision stats as stats_json, byte-exact; browser-rpc computes shipstats stats like serve-stdio).
  CI result (run 37104154159, identical locally): core 339/339, ext 207/239, ext_rpc 0/54, batch 0/44 (the methods are not in the browser; baseline 0), effects 2353/2378, graphs 192/192, cap 150/150, mutated 93/93, formats 4779/4779. Result: no regression.

### Next steps
0. Bench owners may `check_no_regress.py --update` f.json: the browser run improves ext 204 -> 207 and effects 2352 -> 2353 (I don't edit the bench).
2. Moving the F pin to eve-dogma d990818+ should give effects 2378/2378.
3. Deferred (C): tests for the 27 docs/19 web items downgraded have->partial. Not started beyond 070dfc3. A draft of the checks (target profiles, damage pattern editor, fleet command fit, addition panes, show-info traits / required skills, probe size, MWD, utility modules, clipboard, dataset = engine sha) was written locally and dropped from the commit; redo from inventory/mcp-web-gaps.md.
4. Prices: leave the web price box alone. Engine-computed prices (price_overrides, docs/22) and a local "my prices" setting come later.

### Key context
- Repo: /workspace/exct-eve/eve-fit-web. Local preview: `npm run build && npx vite preview --port 4180`, then `node tools/e2e.mjs http://127.0.0.1:4180/eve-fit-web/ wasm-worker|ts-worker`. For http: `node tools/engine-bridge.mjs --stdio "<eve-fit> serve-stdio" --port <free port>`. Other agents' bridges hold 8787 to 8799.
- The F build for 20aa425 is in /tmp/ed20: wasm in target/wasm32-unknown-unknown/release-small, native eve-fit in target/release. It has been copied to public/engines/f.
- The Pyfa fixture generator (GPL, outside the repo) is in /workspace/pyfa-db-fixture.
- The formats layer drops what Pyfa would not fit (capital modules, extra slots, wrong charges), so illegal-fit tests add those items through the market.

## eve4 / eve-fit-mcp / eve-market-prices (executor bot; updated 2026-10-03 15:04 CST)

### Current commits
- EX-CT/eve-fit-mcp main: **d3786eb** = release **v0.4.0** (release run 37105109044 success, published 15:04 CST; assets eve-fit-mcp-0.4.0.tgz, eve-fit-mcp.tgz, SHA256SUMS; CI 37104953313 green: npm test 85 pass/1 todo/0 fail, mcp-bench ext 202/202, effects 2378/2378, cap 150/150). 462c759 engines.lock -> eve-dogma **197223f** (docs/23 batch + prices); CI must run the docs/23 checks (TODO fallback fails CI). Before it: 583a197 price_fit engine path (include_* toggles as a view over engine sections), e309e7e README, 73eb2b7 module /online|/active|/overheated, 7e8fa6b docs/test-ids (tools/test-ids.py), c1626ef price_fit via engine, aeb7b7e compute_batch, 1665e11 compute_fit price inputs. Previous release v0.3.1 (e7b794a). No wip branches.
- EX-CT/eve-market-prices main: **e9781a5** CLI `rule` (bench d22/price_rule transport) + band-edge fix + CI job price-rule-bench (bench pinned d151cb3; CI 37104912614 green, 21/21). wip/docs22-schema kept (same as 1a34266; no branch deletion). Snapshot release `prices-jita44-20261003T063856Z`.

### Done
- docs/23 MCP pass-through, checked against eve-dogma 197223f: compute_fit `price_overrides` / `prices` / `price` (names -> ids, engine block verbatim), compute_batch (fit sources normalised, engine `batch`), price_fit (market table + own isk injected as `prices.isk`, engine block; legacy sum only for old engines). npm test on 197223f: 88 pass, 1 todo (security-status effect 6871, engine). mcp-bench on 197223f: core 339/339, ext 202/202, effects 2378/2378 (0 lost through MCP), cap 150/150.
- Step 2 test gaps (inventory/mcp-web-gaps.md): mcp-bench in CI, silent skips hard-fail, stats/validation value assertions, get_ship traits, utility module states; docs/test-ids.{md,json} (89 ids with inventory item refs).
- eve-market-prices: d22/price_rule found a real bug (order prices were trimmed to 12 digits before the band compare, so an order one ulp above band_max counted); fixed per docs/22 §4.5.
- Earlier: mcp_batch fixes (error codes, graph passthrough), v0.3.1; eve-market-prices v1 updater + snapshot workflow.

### Next steps
1. eve-market-prices vs docs/22 rulings eccf455: check hash = RFC 8785 JCS (UTF-16 key order, ECMAScript numbers) and type coverage from CCP SDE (all published types with a market group; `coverage`); updater name `eve-market-prices`.
2. eve3: price_rule interface notes (rounding of exact-binary halves, price<=0 orders dropped, clamp) in the eve4 report.

## eve3 / bench (executor bot; updated 2026-10-03 15:15 CST)

### batch-suite, no-regress gate, optimizer-bench (shelved at d48bbab, score.py not started) — updated 2026-10-03 14:54 CST
**Current commits** (everything pushed)
- eve-dogma-bench pending-1.11:
  - Gate + baseline: `98419df`, `b3a0957`, `07245e8`, `c2229b2`, `88ed590`, `32981f5`, `d151cb3` (d22 suites).
    - `baselines/f.json` = eve-dogma d990818: core 339, ext 208/239, ext_rpc 0/54, batch 0/92, effects 2378, graphs
      192, cap 150, mutated 93, formats 4779, sde 0/17, price_inject 0/32. Full run_all + gate on d990818: no
      regression. price_rule runs only with `PRICE_RULE_CMD` (updater) and is not in f.json; `SDE_PACK` enables 6
      pending d22 cases.
  - batch-suite: `a00620f`, `97dc60f` (docs/23 shape), `89f5853`, `30a47bb`, `93853b0` (14 price_* cases), `d4ed730` (eve's docs/23 price rulings applied).
    `dfb4f49`: 34 gap cases (gap 15, gap_error 12, calc_price 6, calc_price_embedded 1) + `batch/data/` price files.
    92 cases; self-test 92/92; d990818 0/92 (no `batch` method; calc has no price block / no `--prices`).
  - `48b1cb6`: embedded-snapshot batch case sets options.price.
  - d22 suites `04f9ba5` (docs/22; provisional adapter `d22/adapter.py`): sde 22, price_inject 33, price_rule 21.
    d990818: sde 0/17 (+5 pending, need `SDE_PACK`), price_inject 0/32 (+1 pending); price_rule targets the
    updater (eve4), not F, and runs only with `PRICE_RULE_CMD`.
  - optimizer-bench WIP: `13cb0d9`..`d48bbab`.
- wip branches: `wip/eve3-batch-prices` (already merged).

**In progress:** none. eve's task (steps 0-3) is done: `d4ed730`, `dfb4f49`, `48b1cb6`, `04f9ba5`, `d151cb3`, plus
`32981f5` (ext 208).

**Next step:** wait for F's batch / price / docs/22 contract and change only `batch/adapter.py` and `d22/adapter.py`.
When eve-sde-pipeline publishes an `.edp`, run with `SDE_PACK`. eve4 can point `PRICE_RULE_CMD` at the updater.
Optimizer-bench is still shelved.

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

