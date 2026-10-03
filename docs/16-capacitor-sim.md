# 16 — Capacitor simulation: suite, contract draft, scoring, current pass rates

**中文摘要**：本文介绍"电容模拟"难题：测试集、合约草案、计分方法，以及各引擎目前的通过率。

- **测试集**：位于 eve-dogma-bench 分支 `cap-suite`（`cap/`），共 150 个用例，全部计分。
  覆盖面包括：本地装备、超载、电容注电器（含装填）、己方及来袭的中和器/掠能器（含无人机）、远程电容传输、
  渐进武器、装填、错峰、`cap_sim` 选项，以及 14 个困难用例。
- **期望值**：全部由 Pyfa 作为黑盒 oracle 生成，测试集中不含 Pyfa 代码。
- **合约**：`cap/CONTRACT-CAP.md` 0.1 草案，规定消耗列表、分组/错峰、事件循环、结果和容差。
- **计分**：一个用例的全部电容指标都在容差内才算通过；得分 = 通过数 / 用例数。
- **eve 的裁定**：
  - `cap_sim.stagger` 已弃用，一律忽略（始终错峰，同 Pyfa）；
  - 注电器不足的情况未定义，不计分；
  - `nos_no_target_cap` 不在范围内。
- **当前通过率**：H 150/150，F 147，E 145，A/B/C/D/G/I/J/K 均为 141。
- **"34 个全超载差异"**：H 与 Pyfa 34/34 一致，A 仅 1/34。原因是模拟器会把以毫秒计的周期（浮点数，如 7649.999…）
  向下取整，而 A 得到的是 7650。

Status: draft, 2026-10-03 ~09:45 CST; eve's rulings applied. Informational suite; it is not part of the frozen 1.8.0 scoring.

- **Suite:** `EX-CT/eve-dogma-bench`, branch **`cap-suite`**, directory `cap/`.
- **Contract:** `cap/CONTRACT-CAP.md` (0.1 draft).
- **Results:** `cap/RESULTS.md`.
- **Oracle:** `oracle/pyfa_cap_oracle.py`. It is a GPL test tool that imports Pyfa, like `oracle/pyfa_oracle.py`. No
  Pyfa code is in the suite or in this document. The rules below describe observed behaviour in our own words.

## 1. Problem

The main contract already returns `capacitor.{capacity, recharge_time_s, peak_recharge_gj_s, use_gj_s,
injected_gj_s, delta_gj_s, stable, stable_percent, depletes_in_s}`. The 1.8.0 cases only lightly exercise the
simulator behind `stable_percent` / `depletes_in_s`. With every module overheated, H and A disagreed on 34 bench fits,
and eve left those differences unscored. This problem pins the simulator down.

## 2. Suite (`cap/`)

| category | cases | what it exercises |
|---|---|---|
| baseline | 12 | local modules on frigates to battleships; skills 0 / 3 / 5 |
| modifiers | 6 | rigs, implants and skills that change capacity, recharge and cap need |
| overheat | 15 | overheated reps, propulsion, neutralizers, weapons; 5 bench fits with every module overheated |
| injectors | 14 | capacitor boosters: overshoot postponement, mixed charges, reload, ASB, capital, already-stable fits |
| own_neut_nos | 10 | the fit's own neutralizers (drain) and nosferatu (counted as a gain) |
| incoming | 16 | projected neutralizers / nosferatu: range falloff, signature-resolution factor, drones (optimal cut-off), void bomb, projected fits |
| remote_cap | 9 | incoming and outgoing remote capacitor transmitters, range limit |
| spool | 6 | spool weapons (cap need per cycle) |
| reload | 8 | `factor_reload`: charges with clip and reload in the simulation |
| stagger | 6 | grouping identical modules: non-turrets staggered (duration / n), turrets x n, clip-based offsets |
| light | 14 | mostly stable fits (the `stable_percent` path) |
| sim_options | 16 | `options.cap_sim.{reload, max_time_s}`; 8 cases send the deprecated `stagger: false`, which must be ignored |
| edge | 4 | micro jump drive, drones only, tiny drains |
| hard | 14 | fractional clip-stagger start times; long LCM period; boosters topping up before a big need; void bombs on a battleship (single and grouped); command bursts; smartbombs; mixed skill levels; nos under incoming neuts; injectors kept waiting by incoming transfer |

Requests are ordinary FitRequests (dataset 3569502), generated from item names by `cap/tools/gen_cases.py`.
Expected values come from `cap/tools/make_expected.py`, which runs Pyfa. Each expected file also carries the
simulator's drain list and end time, as diagnostics.

Modules are only overheated when they have an overload effect. The main contract keeps a requested state the module
cannot take, while Pyfa falls back to online, so such requests would test state handling, not the simulator.

## 3. CONTRACT-CAP 0.1 in brief

1. **Drain list.** Own active modules with non-zero cap need come first, then incoming sources one entry each. An
   entry is (duration, need, clip, turret, reload, booster).
   - Duration is **⌊cycle + reactivation⌋ ms**, taken from the double as computed.
   - Nosferatu on the fit are a gain (−transfer amount).
   - Incoming need = amount × resistance × range factor × signature-resolution factor. Transmitters fill (negative
     need).
2. **Preparation.**
   - Without reload in the simulation, clip and reload are cleared, except for boosters.
   - Identical entries are grouped. Boosters fire separately at t = 0.
   - Staggered non-turrets without a clip become one event of ⌊duration/n⌋. With a clip, they get n offset starts.
   - Turrets: need × n. `cap_sim.stagger` is deprecated and ignored.
   - Period = LCM of the durations; none if any clip.
3. **Event loop.**
   - Events are ordered by (time, duration, need, shot, clip, reload, booster).
   - Recharge between events: C·(1 + (√(c/C) − 1)·e^(−Δt/τ))², with τ = recharge / 5.
   - At a period boundary, the run is stable if cap ≥ the previous boundary's cap (rounded to 0.1) and the same
     boosters are waiting.
   - A booster that would overfill waits. Waiting boosters top up before a need larger than the current cap and after
     spending.
   - Cap < 0 ⇒ unstable at that event's time. Time limit 6 h (or `max_time_s`) ⇒ stable.
4. **Results.**
   - Unstable: `depletes_in_s` = failing event time / 1000.
   - Stable: `stable_percent` = min(100, (lowest after + lowest before activations) / 2C × 100).
   - `eve_stable_percent` and `sim_iterations` are report-only.
5. **Rulings (eve, 2026-10-03).**
   - `cap_sim.stagger` is deprecated and ignored: the simulation always staggers, like Pyfa. Main bench 1.8.0
     stays frozen and is not regenerated.
   - When no waiting booster covers a shortfall, the behaviour is undefined and unscored (Pyfa raises an error).
   - `nos_no_target_cap` is out of scope.

## 4. Scoring method

- **Metrics:** `capacity`, `recharge_time_s`, `peak_recharge_gj_s`, `use_gj_s`, `injected_gj_s`, `delta_gj_s`,
  `stable`, plus `stable_percent` (stable cases) or `depletes_in_s` (unstable cases).
- **Tolerance:** max(1e-3, 1e-4·|want|). `depletes_in_s` must be within 0.0005 s, i.e. the same millisecond.
  `stable` must be equal.
- **Case pass:** every scored metric passes.
- **Score:** passed / cases (all 150 scored), with per-category and per-metric breakdowns (`cap/results/<engine>.json`).
- **Proposal if adopted as a round problem:** the cap-suite score is gated on the main bench staying 326/326.
- **Reproduce:** `python3 cap/run_cap.py --work-root <bench checkout>` (all built variants), or
  `--batch-cmd "<engine> --dataset D batch" --name X`. The whole suite runs in < 1 s per engine.

## 5. Current pass rates (after the rulings, built bench binaries, ~09:45 CST)

| engine | passed | failing groups |
|---|---|---|
| H (variant-h b1c7852 code) | **150/150 (100 %)** | — |
| F | 147/150 (98.0 %) | void bomb ×3 |
| E | 145/150 (96.7 %) | void bomb ×3; `cap_sim.reload` with ancillary repairers ×2 |
| A (eve-dogma-rs 659737b), B, C, D, G, I, J, K | 141/150 (94.0 %) | cycle truncation ×6; void bomb ×3 |

Per category, every engine passes baseline, modifiers, injectors, own_neut_nos, remote_cap, stagger and edge. The
differences sit in overheat, reload, spool and light: the truncation cases (overheated fits) appear in several
categories. The rest are in incoming / hard (void bombs) and sim_options (E).

## 6. The "34 all-overheated" differences — verdict

- **H matches Pyfa on 34/34. A matches on 1/34** (tolerance 1e-4 relative / 1e-3 absolute; checked with
  `oracle/pyfa_oracle.py` on the 34 requests).
- **Cause.** The overheat bonus is applied in doubles. For example, a Medium Armor Repairer II's overheated cycle
  comes out as 7649.999… ms. The simulator floors full cycle times to integer milliseconds, so it runs at 7649 ms.
  H does the same; A computes 7650.
- **Effect.** The difference is small per cycle (573.675 vs 573.75 s depletion in the reported example). It changes
  depletion times and stable percentages of overheated fits.
- **Scoring.** These cases are now scored in the cap suite. The same truncation explains the 6 failures of
  A, B, C, D, G, I, J and K.

## 7. Next steps

1. Engines other than H: floor full cycle times to integer ms as computed; add incoming void bombs (H, E, F show
   where).
2. If adopted: freeze the suite as CONTRACT-CAP 1.0 and merge `cap-suite` into a bench release.
