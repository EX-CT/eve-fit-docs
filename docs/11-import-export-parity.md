# 11 — Import / export format parity with Pyfa

**中文摘要**：本文列出 Pyfa `service/port/` 的全部导入/导出格式（EFT、EFT 配置文件、DNA/聊天链接、XML、ESI JSON、
Multibuy、舰船属性文本、变异装备文本、附加列表、自动识别、EFS），说明用 Pyfa 自身代码生成的往返测试集
（eve-dogma-bench 分支 `formats-suite`：3260 条导出、1304 条往返导入、16 条边界用例），并给出 Pyfa 的实际行为
与 variant F 的实现/未实现对照表。目前只有 EFT 导出（326/326）已在引擎中实现；EFT 导入有 6 处与 Pyfa 不同的行为。

Status: 2026-10-03, Pyfa client db 3532181, bench cases 1.8.0 (326 fits). Suite: `EX-CT/eve-dogma-bench`
branch **`formats-suite`** (`formats/`, `oracle/pyfa_formats.py`, `tools/make_formats.py`, `tools/check_formats.py`).
The suite is not part of the frozen 1.8.0 scoring.

## 1. Pyfa's formats (`service/port/`)

| # | Format | Pyfa entry points | Direction | Notes on Pyfa behaviour (from the generated cases) |
|---|---|---|---|---|
| F1 | **EFT text** | `eft.py exportEft`, `importEft` | export + import | Export options: implants, mutations, loaded charges, boosters, cargo (each switchable). It writes `[Empty X slot]` lines and `/offline` (import also accepts `/OFFLINE`). Mutated modules get a `[n]` reference plus a trailing block (base type, mutaplasmid, `attr value, …`). T3D: **no mode line is written**, and on import a mode line is not read: the fit gets the hull's *first* mode. |
| F1a | EFT, multi-fit paste | `importEft` (via `importAuto`) | import | Two `[ship, name]` blocks in one paste are **merged into one fit** (the first header wins). The second header is treated as a stub line. |
| F1b | **EFT config file** (`<Ship>.cfg`) | `eft.py importEftCfg` | import (multi-fit) | The ship comes from the file stem, and each `[name]` header starts a fit. It reads `Drones_Active=` / `Drones_Inactive=` (active count), `Implant_*=`, `Booster_*=`, `Cargohold=` and `Description=`, and `module,charge` with no space. |
| F2 | **DNA** | `dna.py exportDna`, `importDna` | export + import | `ship:sub;1…:mod;n…:drone;n:fighter;n:charge;n::`. Subsystems are sorted by `subSystemSlot`. Modules are grouped by type, and loaded charges are summed (`numCharges`, scripts count 1) and appended with cargo charges. Implants and boosters are never exported, and module state is lost. Import: charges go to **cargo** (not loaded), every module is set to its highest allowed state (`activeStateLimit`), and the name is `"<Ship> - DNA Imported"`. An unknown type id aborts the import. |
| F2a | DNA chat link | `exportDna` with FORMATTING, `importAuto` | export + import | `<url=fitting:DNA::>name</url>`. On import the link text becomes the fit name. |
| F2b | DNA alt (`DNA:ship:id*n`) | `importDnaAlt` | import | Same as DNA, with `*` as the amount separator. |
| F3 | **XML** (EVE client fittings) | `xml.py exportXml`, `importXml` | export + import (multi-fit) | It writes `<hardware slot="low slot 0">`, drone bay and cargo `qty`, and mutated `base_type` / `mutaplasmid` / `mutated_attrs`. Loaded charges are exported as cargo, the description is limited to 400 chars and newlines become `<br>`. Import: charges stay in cargo, implants and boosters are not supported, and every `<fitting>` becomes a fit. |
| F4 | **ESI fitting JSON** | `esi.py exportESI`, `importESI` | export + import | The name is cut to 50 chars (47 plus `...`). Flags: low 11+, mid 19+, high 27+, rig 92+, subsystem = `subSystemSlot`, service 164+, cargo 5, drone bay 87, fighter 158. Charges (as an option) go to cargo, and so do implants and boosters. A fit with no items raises an error. Import: items are sorted by flag, unpublished types are skipped, fighters take the **default squadron size**, and a drone sent with the fighter flag is dropped. |
| F5 | **Multibuy** | `multibuy.py exportMultiBuy` | export | The ship line comes first, then items sorted by (category, group, name) with ` xN` when N > 1. Options: loaded charges, cargo, implants, boosters, and price optimisation (needs the market service, not generated). **Mutated modules are omitted.** |
| F6 | **Ship stats text** | `shipstats.py exportFitStats` | export | `name (ship)`, then DPS/volley, EHP plus per-layer resists, reps, and misc (speed, sig, cap, targeting). It uses Pyfa's `formatAmount` (3 significant digits, `k`/`M` suffixes). |
| F7 | Mutated item text | `muta.py parseMutant` (via `importAuto` with an active fit) | import | 3 lines: base type, mutaplasmid, attributes. The result is an item, not a fit. |
| F8 | Additions lists | `eft.py isValid{Drone,Fighter,Implant,Booster,Cargo}Import` | import | `Name xN` lists pasted into an open fit. The result is a list, not a fit. |
| F9 | Format auto-detection | `port.py Port.importAuto` | import | The order is: XML header, then `{` → ESI, then `[...]` plus a file path → EFT config, then `[a, b]` → EFT, then DNA, then DNA link, then `DNA:` alt. After that it tries the dynamic-item ESI link (needs the network), then the mutant text, then the additions lists. |
| F10 | EFS (Eve Fitting Simulator JSON) | `efs.py EfsPort.exportEfs` | export | This needs Pyfa's GUI fit commands and stats tables, so it is not generated. |
| F11 | Killmail / ESI fittings fetch | `service/esi.py` | import | This needs the network and SSO, so it is out of scope for an offline engine. |

## 2. Test suite (EX-CT/eve-dogma-bench `formats-suite`)

`oracle/pyfa_formats.py` loads Pyfa's own `service/port/*.py` **unmodified**. The only stand-ins are for GUI-only
imports: the Market singleton (it uses Pyfa's `conversions` and `ITEMS_FORCEPUBLISHED` read from Pyfa's source),
`service.fit` recalc/fill, and `activeStateLimit` (executed from Pyfa's `helpers.py` source). For every bench case
the oracle builds the fit, exports it in every format, imports each importable export back, and re-exports.
Hand-written edge inputs go through Pyfa's `Port.importAuto`.

| Set | Rows | Content |
|---|---|---|
| Export | **3260** | 326 cases × {eft, eft_min, dna, dna_formatted, esi, esi_min, xml, multibuy, multibuy_min, shipstats}. 2 rows are expected errors (esi_min on an empty fit). |
| Round-trip import | **1304** | 326 × {eft, dna, esi, xml}. The input is Pyfa's export and the expected result is Pyfa's imported fit (FitRequest-shaped). Of the 326 cases, 132 are over-fitted validation cases (more modules than slots or hardpoints), and Pyfa's importers drop the excess modules there. The other 194 are legal fits. |
| Edge | **16** | `/offline` and `/OFFLINE`; mutated blocks; all EFT sections and drone-stack merging; CRLF, an unknown item and a charge on a module that takes none; T3D mode line; T3C subsystems; structure with services; multi-fit EFT paste; `.cfg` multi-fit; XML with 2 fits and a mutation; DNA link, alt and unknown id; ESI with wrong flags; additions list; single mutant. |

Checker: `python3 tools/check_formats.py --rpc "<variant> serve-stdio" --name X`. It sends `format_export` and
`format_import` RPC calls (proposed contract extension), falling back to `eft_export` and `eft_parse`.

### Pyfa's own round-trip stability (194 legal fits)

| Format | Pyfa re-export identical | Import identical to the source fit | What the format loses (count of fits) |
|---|---|---|---|
| EFT | 190/194 | 133/194 | drone active count 48, active→online by `activeStateLimit` 15 (MJD, cloak, WCS, …), T3D mode set to the first 5, invalid charge dropped 3, drone stacks merged 1 |
| DNA | 183/194 | 0/194 | name 193, loaded charges → cargo 121, active counts 47, state 15, subsystem/module order 5, implants and boosters 6, fighters 1 (plus one unknown-type crash) |
| ESI | 181/194 | 52/194 | charges → cargo 121 (cargo 125), active counts 48, state 15, fighter squadron size 6, implants and boosters 6 |
| XML | 185/194 | 54/194 | charges → cargo 121, active counts 48, state 15, implants and boosters 6 |

## 3. Parity table — variant F (`EX-CT/eve-dogma-lab` `variant-f`, commit beefcfd)

Status: ✅ implemented and matching Pyfa · 🟡 implemented with differences · ⬜ not implemented.

| Format | Direction | F status | Suite result | Differences vs Pyfa |
|---|---|---|---|---|
| EFT (all options) | export | ✅ | 326/326 | Only the known data divergence: T3 cruisers get `maxSubSystems` 5 in SDE 3569502 and 4 in Pyfa's db, so there is one extra `[Empty Subsystem slot]`. |
| EFT (options off) | export | ⬜ (options ignored) | 80/326 | `eft_export` has no option switches. |
| EFT | import (`eft_parse`) | 🟡 | 138/326 (legal fits 138/194) | (1) No `activeStateLimit`: MJD, cloak, WCS, etc. import as active (Pyfa: online), and modules that can't be activated without a charge import as online (Pyfa: active). (2) Duplicate drone lines are not merged. (3) An invalid charge (one the module can't load) is kept. (4) Over-fitted modules are kept (Pyfa drops them; F reports them as violations at calc time instead). (5) A T3D mode is left null (= first mode by contract, a soft difference). (6) Drone `active` = quantity (Pyfa: 0 on EFT import, a soft difference). |
| EFT config (`.cfg`) | import | ⬜ | – | |
| DNA / DNA link / DNA alt | export, import | ⬜ | – | |
| XML | export, import | ⬜ | – | |
| ESI JSON | export, import | ⬜ | – | |
| Multibuy | export | ⬜ | – | |
| Ship stats text | export | ⬜ | – | Needs Pyfa `formatAmount`. The stats themselves are already 21051/21051. |
| Mutant text / additions lists / auto-detect | import | ⬜ | – | |
| EFS | export | ⬜ (not in suite) | – | |

Other variants can be scored with the same checker. As of this writing only EFT export and import exist in the
1.8.0 contract.

## 4. Recommendations for the contract (proposed, not yet agreed)

1. Add `format_export {fit, name, format, options}` and `format_import {text, format|auto, path?}` RPC methods.
   Pyfa's option names are in `formats/expected/export.jsonl`.
2. Decide whether `eft_parse` should copy Pyfa's import-time normalisation: `activeStateLimit`, merging drone
   stacks, dropping invalid charges and over-fitted modules. An alternative is to keep the request faithful and
   report violations, with a `pyfa_import: true` option for strict parity.
3. Treat T3D modes explicitly: Pyfa neither writes nor reads a mode line in EFT. DNA, ESI and XML carry no mode
   either.
