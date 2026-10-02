# 09 — Engine round 1 evaluation (variants A–K)

> **Status: DRAFT SKELETON.** Numbers are placeholders (`⟨…⟩`) until the unified scoring run on 2026-10-03 10:20 CST
> (bench 1.8.0, frozen at `0969967`). Filled in from `eve-dogma-bench/results/evaluation.{md,json}`.
>
> 中文摘要：第一轮 11 个 dogma 引擎方案（A–K）的统一评测。正确性（bench 1.8.0 全部 326 个 case 与 Pyfa 一致）是门槛，
> 过门槛后按 速度 40%、可维护性 35%、功能覆盖 15%、可移植性 10% 计分。本文记录方法、规则、结果、决定与合并计划。

## 1. Goal

Eleven teams built the same stateless engine contract (`FitRequest` JSON → `FitStats` JSON, CONTRACT.md revision 1.4.3,
same dataset `dataset-3569502.json.gz`) with different languages and architectures. Round 1 decides which design (or
which combination of ideas) becomes the core of the EXCT toolkit (CLI, MCP server, web UI, WASM).

## 2. Method

All measurements come from one command in [EX-CT/eve-dogma-bench](https://github.com/EX-CT/eve-dogma-bench):

```bash
python3 tools/evaluate.py --as-of 2026-10-03T10:15:00+08:00 --runs ⟨N⟩ --fresh-clones   # results/evaluation.{md,json}
```

0. **Version rule:** every variant is evaluated at its branch HEAD as of **2026-10-03 10:15 CST** (last commit at or
   before the cutoff). Self-reported "final" versions are reference only. A commit that fails the correctness gate is
   **disqualified** for the round; there is no fallback to an older commit.
1. **Fetch** the head as of the cutoff for each variant: A = `EX-CT/eve-dogma-rs@main`; B–K = `EX-CT/eve-dogma-lab@variant-<x>`
   (directory `variant-<x>/`, commands from its `bench.yaml`). Full commit SHAs are recorded.
2. **Build** with the variant's own `build` command (time recorded; fresh clone ⇒ fresh build).
3. **Correctness + speed** with the official scorer (`run.py`, the same code `bench.py` uses), ⟨N⟩ runs per variant,
   after one untimed warm-up batch; median of perf numbers, `loadavg` before/after every run. Variants run one at a time. Every child run has a hard
   timeout, so one broken variant cannot stall the evaluation.
4. **Features** through the variant's `serve-stdio` RPC: EFT export (Pyfa byte-exact check), `eft_parse` round-trip,
   `calc` over RPC equal to the CLI, `meta`, unknown-method error, `search` (interim spec) and `type` probes.
5. **Maintainability** from the source tree: LOC per language (core / test / tooling / generated / vendored), own test
   suite run and result, direct dependencies, docs (README / DESIGN / LICENSE), license, and a heuristic count of
   effects special-cased by name in hand-written code (per-effect hard-coding vs data-driven).
6. **Portability**: evidence of a WASM / browser build in code (1), only documented (0.5), none (0).

Caveat: the box is a shared 8-CPU machine that was under load (loadavg ≈ ⟨…⟩) during the run. Perf numbers are only
comparable within the same run; the log scale in the speed formula softens noise.

## 3. Scoring rules

- **Version:** branch HEAD as of 10:15 CST; failing head ⇒ disqualified (no fallback).
- **Gate:** a variant is ranked only if it builds, runs and passes **all 326 cases** (21 051 values) in every run.
- **Total = 0.40·Speed + 0.35·Maintainability + 0.15·Features + 0.10·Portability** (each sub-score in [0, 1]).
- `L(x, best, span) = clamp(1 − log10(x / best) / log10(span), 0, 1)` for lower-is-better `x`.
- **Speed** = 0.5·L(latency ms/calc, 100) + 0.3·L(1 / batch fits·s⁻¹, 100) + 0.2·L(cold start ms, 100).
- **Maintainability** = 0.25·Tests + 0.20·DataDriven + 0.20·Size + 0.15·Docs + 0.10·Deps + 0.10·Build
  (Tests: passing suite 0.6 + 0.4·min(1, log10(1+n)/2), failing 0.2, timeout 0.3, none 0; DataDriven: L(h+10, h_min+10, 10)
  with h = effects special-cased by name; Size: L(core LOC, min, 10); Docs: 0.4 README + 0.4 DESIGN + 0.2 LICENSE;
  Deps: 1/(1+n/5); Build: L(build s, best, 100), only when all builds are fresh, else dropped and weights renormalised).
- **Features** = mean(EFT, RPC, search, type).
- **Portability** = WASM/browser build in code 1 · documented only 0.5 · none 0.

The authoritative text is the docstring of `tools/evaluate.py`; if this section and the tool disagree, the tool wins.

## 4. Variants

| | Variant | Language | Core idea | Branch / commit | License |
|---|---|---|---|---|---|
| A | eve-dogma-rs (reference) | Rust | lazy memoised modifier graph | `main` @ ⟨sha⟩ | LGPL-3.0-or-later |
| B | data-oriented | Rust | compile a flat CSR modifier graph, then evaluate | `variant-b` @ ⟨sha⟩ | ⟨…⟩ |
| C | Go | Go | pull-based modifier registry with selectors | `variant-c` @ ⟨sha⟩ | ⟨…⟩ |
| D | TypeScript | TypeScript | pull-based attribute graph + typed modifier pipeline | `variant-d` @ ⟨sha⟩ | ⟨…⟩ |
| E | Pyfa-faithful | Rust | Pyfa eos transpiled to Rust | `variant-e` @ ⟨sha⟩ | GPL-3.0-or-later |
| F | codegen | Rust (+WASM) | SDE compiled into Rust code at build time | `variant-f` @ ⟨sha⟩ | ⟨…⟩ |
| G | batch | Python + NumPy | vectorised dogma over many fits | `variant-g` @ ⟨sha⟩ | ⟨…⟩ |
| H | ECS | Rust (hecs) | entities/components/systems | `variant-h` @ ⟨sha⟩ | ⟨…⟩ |
| I | incremental | Rust (salsa) | memoised demand-driven query graph | `variant-i` @ ⟨sha⟩ | ⟨…⟩ |
| J | C++20 | C++20 | mmapped POD dataset image, flat attribute tables | `variant-j` @ ⟨sha⟩ | ⟨…⟩ |
| K | .NET | C# (Native AOT) | typed rule book + binary dataset cache | `variant-k` @ ⟨sha⟩ | ⟨…⟩ |

### Approaches (from each variant's README / DESIGN.md)

- **A — eve-dogma-rs (reference).** The baseline Rust engine. It builds an object graph with a hash map of attributes per
  item, resolves each modifier to concrete target items at registration time, and evaluates lazily with per-attribute
  memo cells. Modifiers come from SDE `modifierInfo`; effects CCP ships without it are covered by small data patches in
  the pipeline plus a few documented engine specials. Bincode dataset cache for faster cold start. Library + CLI +
  JSONL RPC (calc, EFT parse/export, search, type, meta).
- **B — data-oriented Rust.** "Compile, then evaluate": items get sorted base-attribute patches, all modifiers are
  appended to one flat list, sorted by a packed `(item, attr, op, seq)` key into a CSR graph whose sources resolve to
  node indexes or constants, then evaluated by an iterative DFS that visits each node once. Same formulas as A for the
  stats layer; adds `calc_many`.
- **C — Go.** Zero-dependency, stdlib-only Go. The dataset is loaded once (in parallel) into an immutable shared
  structure; each fit keeps a modifier registry keyed by attribute and tagged with a *selector* (ship location, group,
  required skill, owner…) instead of expanding modifiers to targets (pull, not push), with a memoised value cache and an
  explicit cycle stack. Skill modifiers for all-V characters are templated once per dataset. Library + CLI + HTTP.
- **D — TypeScript.** Zero runtime dependencies, pure ESM, runs in Node and in the browser (dataset decompressed with
  `DecompressionStream`). A pull-based attribute graph with lazily materialised cells, an operator table as data, target
  resolution through prebuilt indexes, and a registry of special effects keyed by effect name ("adding a special effect
  is one registry entry"). Goal: the most maintainable and web-friendly engine, within ~2–3× of Rust.
- **E — Pyfa-faithful Rust port.** Matches Pyfa by not re-deriving dogma: a Python-AST transpiler (`tools/pyfa2rs.py`)
  turns Pyfa's `eos/effects.py` handlers (2 359 of 2 402) into Rust in their original order, with names resolved to ids
  at transpile time; Pyfa's quirks (stacking sort order, `round(x, 2)`, cap-sim heap order) are kept on purpose. A few
  hand ports (RAH, remote reps/neuts…). GPL-3.0 because it is derived from Pyfa.
- **F — codegen Rust/WASM.** `build.rs` compiles the SDE into ~5 MB of generated Rust: every effect becomes a match arm
  of straight-line modifier calls, domains/operators/stacking decisions resolved at build time, skill effects
  precomputed for levels 0–5, dense attribute tables. No dataset parsing at runtime (dataset baked into the binary),
  so cold start is ~2 ms; also builds to WASM.
- **G — Python + NumPy batch.** The readability and batch-throughput baseline: fits are parsed per fit in Python, then
  modifier registration (gather/filter/equi-join on CSR template tables) and evaluation (levelised dependency graph,
  per-stage aggregation with stacking penalties via lexsort, caps, rounding) run as NumPy array operations over the
  whole batch. Special effects (propulsion, projected EWAR, RAH, fleet buffs) in a small per-fit Python layer.
- **H — Rust ECS.** Dogma as an Entity-Component-System on `hecs`: one entity per ship/character/skill/module/charge/…,
  attributes and modifiers as components, effect application and attribute calculation as systems run in an explicit,
  documented order (`Fit::run`) on a fresh world per request; derived dataset cache for start-up.
- **I — Rust salsa incremental.** Dogma as a demand-driven memoised query graph (salsa 0.28): inputs per item identity,
  derived queries for structure index, outgoing/incoming modifiers and attribute values, with backdating so a
  persistent session only recomputes what an edit touched. The CLI stays stateless and byte-identical (checked forward
  + reverse vs fresh database). Aimed at interactive fitting UIs.
- **J — C++20.** Performance-first: the gz JSON dataset is converted once (libdeflate + simdjson) into a relocatable POD
  image with flat index arrays and precomputed relevance tables, cached and `mmap`ped read-only (cold start ~2 ms);
  requests are parsed with simdjson, evaluated on flat attribute tables and written with a streaming JSON writer;
  `batch` spreads the JSONL across all cores in order. Byte-level fidelity to A's output.
- **K — C# / .NET 8 Native AOT.** "Rules that read like a rulebook": strong typed ids (`readonly record struct`),
  exhaustive switches, a `RuleBook` layer holding all dogma special cases separate from the attribute graph, a
  reflection-free deterministic JSON writer, binary dataset cache; Native AOT gives a single native executable with
  millisecond-scale start-up.

## 5. Results

Source: `eve-dogma-bench/results/evaluation.md` (commit ⟨sha⟩), measured ⟨date time⟩ CST, loadavg ⟨…⟩, ⟨N⟩ runs.

| rank | variant | cases | values | ms/calc | fits/s | cold ms | speed | maint. | features | port. | **total** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| ⟨…⟩ | A | ⟨…⟩/326 | ⟨…⟩/21 051 | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ | ⟨…⟩ |
| ⟨…⟩ | B | | | | | | | | | | |
| ⟨…⟩ | C | | | | | | | | | | |
| ⟨…⟩ | D | | | | | | | | | | |
| ⟨…⟩ | E | | | | | | | | | | |
| ⟨…⟩ | F | | | | | | | | | | |
| ⟨…⟩ | G | | | | | | | | | | |
| ⟨…⟩ | H | | | | | | | | | | |
| ⟨…⟩ | I | | | | | | | | | | |
| ⟨…⟩ | J | | | | | | | | | | |
| ⟨…⟩ | K | | | | | | | | | | |

Maintainability detail (core LOC per language, tests, deps, build time, docs, license, hard-coded effects) and the
per-run table with loadavg: see `results/evaluation.md`.

Not ranked (failed the gate) and why: ⟨…⟩

## 6. Decision

⟨To be written after the 10:20 run.⟩ Questions to answer:
- Which variant becomes the core engine (`eve-dogma-rs` main) for the CLI / MCP / web UI?
- Is the winner also the WASM/browser engine, or is a second engine kept for the browser (D or F)?
- Do we keep a second independent implementation as a cross-check oracle (differential testing)?

## 7. Merge plan

⟨To be written.⟩ Template:
1. Ideas to port into the chosen core (per idea: source variant, expected gain, owner, acceptance = bench 326/326 + no
   perf regression).
2. Repository moves (which branch becomes which repo / crate; licensing check — E and any variant importing GPL
   code stay GPL; the core stays LGPL-3.0-or-later).
3. Variants archived (branch kept, README pointer to this document).
4. Bench: unfreeze, apply `pending-1.9.0.md`, re-run the evaluation for the merged engine.

## 8. Lessons learned

⟨To be written.⟩ Prompts:
- Correctness: which Pyfa quirks were hardest, and how did the shared oracle/bench shape the work?
- Performance: what actually mattered (dataset loading/caching, allocation, skill pruning, batch parallelism)?
- Process: 11 parallel bots, frozen bench versions (1.5 → 1.8), contract rulings; what to change for round 2?
- Measurement: shared loaded machine; next time pin CPUs or run the evaluation on an idle host.
