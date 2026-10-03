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

## eve4 / eve-fit-web (executor bot; updated 2026-10-03 14:40 CST)

### Current commits
- EX-CT/eve-fit-web main: **e1afe8d** (all work pushed; no uncommitted changes; **no wip branches**).
  Pages run 37102734646 (6ac02ba) green: unit, e2e wasm-worker 82/82, http 82/82, ts-worker 72/72, bench 331/331, graphs 178/178, ext 202/202, effects 2353/2378. J (optional) failed only validation-problems there; fa206d1 fixes it. The e1afe8d run is pending.
- Live (https://ex-ct.github.io/eve-fit-web/) checked at f97501a: e2e wasm-worker 76/76, ts-worker 66/66.

### Done
- **Milestone 4 (A), fit library + Pyfa import:** b75923b (formats layer: Pyfa saveddata.db via sql.js, library import/export, fixture made by Pyfa 1d9f72b, data only), d670e64 (IndexedDB store + migration from localStorage, folders/tags/search/rename/duplicate/delete/export/backup UI), 342b813 + f97501a (e2e: pyfa-db-*, library-*, dna-import = FMT-DNA-001, library-reload-persistence = DB-001, library-migration; docs/test-ids.md).
- **B, F pin bump:** a972c4e engines.lock F -> EX-CT/eve-dogma 20aa425 (main, after 2da8150). 692ff88 UI: mining, outgoing reps, bombs to kill, overheat burnout, drone/fighter EHP, violation labels (zh-CN). 1ca5683 CI suites web-ext (bench pending-1.11 @1a8be05 ext 202/202) and web-effects (>= EFFECTS_MIN 2353) in the browser build. 6ac02ba e2e ids mining-yield, outgoing-reps, bombing-table, overheat-burnout, drone-ehp, validation-problems. 7854503 docs/test-ids.md, 3eb47ba README.
- 070dfc3 (done before C was deferred): drag-and-drop rack position (moveModule), mutaplasmid roll range, Import from clipboard; unit tests web.unit.module-move, charges-valid-only.

### Next steps
1. Confirm the 3eb47ba Pages run is green (unit, e2e on all backends, bench 331/331, graphs 178/178, ext 202/202, effects) and run the live e2e (wasm-worker, ts-worker) against the deployed site.
2. EFFECTS_MIN can go to 2378 once the pin moves to eve-dogma d990818+ (effects 2378/2378).
3. Deferred (C): tests for the 27 docs/19 web items downgraded have->partial. Not started beyond 070dfc3. A draft of the checks (target profiles, damage pattern editor, fleet command fit, addition panes, show-info traits / required skills, probe size, MWD, utility modules, clipboard, dataset = engine sha) was written locally and dropped from the commit; redo from inventory/mcp-web-gaps.md.
4. Prices: leave the web price box alone. Engine-computed prices (price_overrides, docs/22) and a local "my prices" setting come later.

### Key context
- Repo: /workspace/exct-eve/eve-fit-web. Local preview: `npm run build && npx vite preview --port 4180`, then `node tools/e2e.mjs http://127.0.0.1:4180/eve-fit-web/ wasm-worker|ts-worker`. For http: `node tools/engine-bridge.mjs --stdio "<eve-fit> serve-stdio" --port <free port>`. Other agents' bridges hold 8787 to 8799.
- The F build for 20aa425 is in /tmp/ed20: wasm in target/wasm32-unknown-unknown/release-small, native eve-fit in target/release. It has been copied to public/engines/f.
- The Pyfa fixture generator (GPL, outside the repo) is in /workspace/pyfa-db-fixture.
- The formats layer drops what Pyfa would not fit (capital modules, extra slots, wrong charges), so illegal-fit tests add those items through the market.

## eve4 / eve-fit-mcp / eve-market-prices (executor bot; updated 2026-10-03 14:40 CST)

### Current commits
- EX-CT/eve-fit-mcp main: **477726d** (all pushed, nothing uncommitted, no wip branches). Green runs through a3ef6d0/8dfb5b1 (37102094656, 37102267465); 477726d run 37102448844 (mcp-dogma-bench effects+cap suites).
- EX-CT/eve-market-prices (new, public, MIT): main **6a59568** (license + README stub); all code on **wip/docs22-schema** (latest 5444990).

### Done
- eve-fit-mcp step 1 (engine bump): engines.lock → eve-dogma 2da8150, compute_fit exposes mining / remote_repair / bombing / heat / validation / probe_size etc., get_ship traits, `tools/mcp-dogma-bench.py` in CI (core 339/339, ext 202/202, effects 2353/2378 = engine, cap 150/150), stats + validation tests.
- eve-market-prices (TS/Node ≥20, zero runtime deps, lib + CLI): rule `jita_sell_band_weighted` v1, ESI source (Forge sell orders, Jita 4-4 filter, X-Pages, Expires/ETag/304, error-limit pause, 420/5xx retry, Last-Modified consistency, User-Agent with contact), Fuzzwork source (`exact:false`), source registry, snapshot = docs/22 §4 `eve-price-snapshot` v1 (canonical JSON + sha256 `content_hash`, §4.5 fields + invariants, `prices-<market>-<time>.json[.gz]`).

### Paused (by priority change)
- eve-fit-mcp step 2 (test gaps) stopped at: `docs/test-ids.md/json` update for new ids, README section for mcp-bench/new outputs. v0.4.0 release (VERSION bump + tag) not done.

### Next steps
1. eve-market-prices: fix tests for the docs/22 fields, JSON Schema file, README, CI (Node 20/22 + non-gating live smoke), daily snapshot release workflow; merge wip → main.
2. Then (after F's contract, docs/23): MCP `compute_batch` + `price_overrides` / `prices` pass-through only (no pricing math in MCP).
