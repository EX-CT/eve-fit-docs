# HANDOFF

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
