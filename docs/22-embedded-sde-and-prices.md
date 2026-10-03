# 22 — Embedded SDE, frozen prices and the price snapshot format

**中文摘要**：
- **内嵌 SDE**：每个发布版本在构建时把对应版本的 SDE 紧凑二进制包（`eve-sde-pipeline` 产出的 `.edp`）用 `include_bytes!` 编进程序。原生、WASM、CLI、RPC 默认都只用内嵌数据，从不联网。输出和 `version` 命令报告 `sde_build` 与数据哈希。可选开关 `--sde <文件>` / RPC `sde_override` 在不发新版的情况下加载更新的 SDE 包。
- **冻结价格**：每个发布版本内嵌一份价格快照（含来源、时间、规则参数）。快照由独立的更新工具生成，引擎只读。定价规则：Jita 4-4（空间站 60003760）卖单；剔除数量少于 `min_units` 的订单（按数量过滤，不按订单数比例）；p0 = 剩余最低价；快照价 = 价格落在 [p0, p0×1.05] 内的全部卖单按数量加权的均价；带宽（默认 5%）和 `min_units` 可配置。
- **价格注入**：请求可带 `prices`（价格表）和 `price_overrides`（按类型 / 市场分组 / 分组 / 类别的固定价或倍数，见 docs/23），或用 `--prices <文件>` 加载；都没有时用内嵌快照。输出给出 `price_source` 和快照时间。优化器的价格上限和价格目标都用这套价格。
- **价格更新工具**（eve4 负责）独立于引擎，数据源可插拔（ESI 市场订单、Fuzzwork 等），输出带版本的快照文件；快照格式由本文 §4 定义。

**Status: DECIDED (design from the user via eve, 2026-10-03 14:15 CST); written up 2026-10-03.** Implementation in
`EX-CT/eve-dogma` comes after the effects step and before the docs/19 missing/partial items. eve3 writes bench cases
for the embedded-version checks, the pricing rule and the price injection overrides. The price updater is a separate
tool built by eve4 against §4.

## 1. Principles

1. **The engine never touches the network.** Not in native, WASM, CLI or RPC, and not as a fallback. Everything it
   needs is compiled in or passed in by the caller (file path, RPC value, request field).
2. **One release = one SDE build + one price snapshot.** Both are embedded at build time. The release reports
   exactly which (`sde_build`, revision, content hashes, snapshot id and time).
3. **Updates without a release are opt-in.** `--sde` / `sde_override` and `--prices` / request `prices` replace or
   override the embedded data for that process or request only. Output always says which data was used.
4. **The pricing rule lives in the updater.** The engine reads `price` per type from a snapshot and never re-derives
   it from orders. The rule and its parameters are recorded in the snapshot for provenance.

## 2. Embedded SDE (A)

### 2.1 Today and the reconciliation

Today `eve-sde-pipeline` publishes `dataset-<build>-r<rev>.json.gz` (deterministic JSON, ~0.7 MB gz, format
`exct-eve-dataset` v1, docs/04 §3). `eve-dogma` reads it at build time (`EVE_DOGMA_DATASET`) in
`crates/eve-dogma-codegen`, which generates static Rust tables (`eve-sde`: types, attributes, names, …) and compiled
modifier code (`eve-dogma`: `apply_local`, folded skills, Pyfa quirk table). So the data is already compiled in, but:
there is no binary pack, nothing at runtime can load another SDE, and the SDE version is tied to whatever file the
builder pointed `EVE_DOGMA_DATASET` at (the `meta` command reports `sde_build` and `dataset_sha256` of that file).

Decision:
- The pipeline gets a second, binary artifact per release: the **EVE data pack** `sde-<build>-r<rev>.edp` (§2.2),
  built deterministically from the same dataset JSON (`python -m sdepipe pack dist/dataset-*.json.gz`). The JSON
  stays (diffs, MCP, humans); the pack is what engines embed.
- `eve-dogma` builds from the pack: `EVE_DOGMA_SDE_PACK=path/to/sde-….edp` (CI and release download it from the
  pipeline release matching the pinned `SDE_TAG`). The codegen reads the pack instead of the JSON, and the engine
  embeds the very same bytes with `include_bytes!`. A build-time check asserts that the pack's content hash equals
  the hash baked into the generated tables, so compiled tables and embedded pack can never come from different data.
- **Default data path (embedded):** the generated static tables, i.e. zero-copy `'static` data compiled from the
  embedded pack; nothing is parsed at startup. The embedded pack bytes are kept for `version`, hash checks and as
  the reference for the override path.
- **Override path (`--sde` / `sde_override`):** the pack is loaded at runtime into a `PackData` view: the section
  directory is read at load, each section is sliced zero-copy from the buffer and decoded lazily on first use.
  Modifiers are applied by an interpreter over the pack's modifier rows, with the same Pyfa quirk table and the same
  `sp_*` specials (keyed by effect name) as the compiled path. CI checks that the interpreter fed the embedded pack is
  byte-identical to the compiled path on the round-1 corpus and all bench suites.
- Optional build feature `embed-sde` is on by default; a build without it is not a release build (it only exists
  for size experiments and must be given `--sde`).

### 2.2 Pack format `edp` v1

Little-endian. All offsets are from the start of the file. Deterministic: same dataset JSON → byte-identical pack.

Header (64 bytes):

| offset | type | field |
|---|---|---|
| 0 | `[u8;4]` | magic `EDPK` |
| 4 | u16 | `format_major` = 1 (engines refuse an unknown major) |
| 6 | u16 | `format_minor` (additive changes; engines ignore unknown sections) |
| 8 | u32 | `sde_build` (CCP build, e.g. 3569502) |
| 12 | u16 | `revision` (pipeline revision `r<rev>`) |
| 14 | u16 | `section_count` |
| 16 | i64 | `sde_release_unix` (CCP release time, seconds UTC) |
| 24 | `[u8;32]` | `content_sha256`: SHA-256 of bytes 64..EOF |
| 56 | u64 | reserved, 0 |

Section directory: `section_count` entries of 24 bytes `{tag: [u8;4], flags: u32, offset: u64, len: u64}`, sorted by
tag, followed by the sections, each 8-byte aligned. `flags` bit 0 = section is zlib-compressed (only used for the
text sections `TRAI`, `ENVI`, `NAMZ`).

| tag | content (row layout; strings are `(u32 offset, u32 len)` into `STRS`) |
|---|---|
| `META` | UTF-8 JSON: pipeline version, dataset `format_version`, applied patch ids, `dataset_sha256` of the source JSON |
| `STRS` | string blob (UTF-8) |
| `ATTR` | attributes sorted by id: `{id u32, name str, default f64, flags u32 (stackable, high_is_good, published), max_attr u32, min_attr u32, unit u16}` |
| `EFCT` | effects sorted by id: `{id u32, name str, category u8, flags u8, duration/range/falloff/tracking/discharge/resist/usage-chance attr u32 ×7, mod_first u32, mod_count u32}` |
| `MODS` | modifier rows `{func i8, domain i8, op i8, pad, modified u32, modifying u32, group_or_skill u32}` (docs/04 tuple order) |
| `TYPE` | types sorted by id: `{id u32, group u32, category u32, market_group u32, meta_group u16, meta_level i16, tech_level u8, published u8, variation_parent u32, name str, mass/volume/capacity/radius f64, attr_first u32, attr_count u32, eff_first u32, eff_count u32, skill_first u32, skill_count u16}` |
| `TATR` | `{attr u32, value f64}` rows, per type sorted by attr |
| `TEFF` | `{effect u32, is_default u8}` rows |
| `RSKL` | required skills `{skill u32, level u8}` rows |
| `GRUP`, `CATG`, `MKTG`, `METG`, `UNIT` | `{id, parent/category, name str, flags}` |
| `DBUF` | warfare buffs `{id u32, aggregate u8, op i8, name str, item/location/location_group/location_skill row ranges}` + their rows |
| `MUTA` | mutaplasmids: `{id, input/output ranges, attribute ranges {attr, min f64, max f64}}` |
| `FABL` | fighter abilities and per-type ability slots |
| `CLON` | clone grades (alpha skill caps) `{grade u32, skill u32, level u8}` |
| `TRAI`, `ENVI`, `NAMZ` | traits text (en/zh), environment (wormhole classes, beacons, system effects), zh names: zlib JSON, same shape as the dataset JSON sections |

### 2.3 Version reporting

- New top-level output key **`provenance`** on every `calc` / `batch` / RPC result (strip-listed like the other new
  keys: added to `ci/round1-new-keys.txt`, so the round-1 identity check runs on the output minus `provenance`, and
  `ci/round1.sha256` is updated once):

```json
"provenance": {
  "engine": "eve-dogma 0.2.0",
  "sde_build": 3569502, "sde_revision": 5, "sde_release": "2026-10-02T11:08:57Z",
  "sde_hash": "sha256:6c12…", "sde_source": "embedded",
  "price_source": "embedded", "price_snapshot_id": "jita44-20261003T060000Z",
  "price_time": "2026-10-03T06:00:00Z", "price_hash": "sha256:…"
}
```

  `sde_source`: `embedded` | `override` (then `sde_override_path` or `"rpc"`). `price_source`: `embedded` | `file` |
  `request` (full table in the request) | `request+embedded` / `request+file` (partial override on top). With
  `price_source = request`, `price_snapshot_id` / `price_time` / `price_hash` are null.
- `eve-fit version` and RPC `version` return the same object without per-request fields, plus `pack_format`
  (`1.0`), `snapshot_schema_version` and the build target (`native` / `wasm32-wasip1` / `wasm32-unknown-unknown`).
  The existing `meta` command keeps its fields (features are only added).

### 2.4 Override switch

- CLI: `eve-fit --sde FILE.edp <command>` (all commands). The file is checked (magic, `format_major`,
  `content_sha256`) before use; failure is an error, never a silent fallback to the embedded pack.
- RPC (serve-stdio, WASM `rpc`): method `sde_override` `{"path": "…"}` (native) or `{"pack_b64": "…"}` (any), which
  switches the session; `{"reset": true}` returns to the embedded pack. Result: the new `version` object.
- An override pack with a newer `sde_build` than the engine knows is accepted; effects the pack contains whose
  names are not in the engine's specials / quirk table are applied from their modifier rows only and listed once in
  `warnings` (`"sde_override: N effects without engine handlers: …"`).
- The override never changes the embedded price snapshot; prices are independent (§3).

## 3. Frozen prices (B)

### 3.1 Snapshot embedded per release
- The release workflow takes the newest snapshot published by the updater (§4) at release time, checks it
  (schema, `content_hash`), and embeds it (`EVE_DOGMA_PRICES=path`, `include_bytes!`, gzip JSON). It is parsed
  lazily on first price use.
- The engine reads `types[*].price` only. It never sees orders and has no pricing rule in code.

### 3.2 Pricing rule (implemented by the updater; recorded in every snapshot)
Per type, rule `jita_sell_band_weighted` v1:
1. Orders: **sell orders only, at Jita IV - Moon 4 - Caldari Navy Assembly Plant, `location_id` 60003760**
   (region 10000002 The Forge). Buy orders and other stations are ignored.
2. Drop every order whose remaining units (`volume_remain`) are **fewer than `min_units`**. This filters by unit count
   per order, not by a percentage of the order count.
3. `p0` = the lowest price among the remaining orders.
4. Band: all remaining orders with price in **[p0, p0 × (1 + band)]**, `band` default **0.05**.
5. `price` = unit-weighted average over the band: Σ(price_i × units_i) / Σ units_i, with units = `volume_remain`.
6. If no order remains after step 2, the type gets no price (listed in `missing`).
- Parameters `band` and `min_units` are configurable in the updater and written to the snapshot. Default
  `min_units`: **10** (proposed; eve may change it, it is a parameter, not engine code).

### 3.3–3.5 Price injection, precedence, request fields and output — superseded by docs/23
**Reconciled 2026-10-03 with [docs/23](23-batch-api-and-prices.md) §5–§6**, which is now normative:
- Request inputs: `price_overrides` (type / market group incl. children / group / category; fixed price or multiplier)
  and `prices` (`{"isk": {type_id: isk}, "use_snapshot": bool}`; the earlier `mode: override|replace` is an accepted
  alias for `use_snapshot: true|false`).
- Layers, highest first: variant overrides > request overrides > request `prices` > market snapshot (`--prices FILE` /
  RPC `prices_load` if given, replacing the embedded snapshot completely; else the embedded snapshot). Within an
  override layer the most specific entry wins (type > deepest market group > group > category; tie → lower id); a
  multiplier applies to the price resolved by the next lower layer; a fixed price stops the chain.
- Output: top-level `price` block with total, per-section lines down to each item, `source` per line
  (`override:type|market_group|group|category`, `injected`, `snapshot`), `snapshot_time`, and a `missing` list
  (supersedes the `*_isk` / `unpriced` sketch). Emitted only when price inputs are present or `options.price` is set,
  so round-1 outputs stay unchanged; `price` is strip-listed for the round-1 identity check anyway.
- A type without a price in any layer is missing (reported, never guessed).

### 3.6 Optimizer
- `constraints.price.max_isk` (price cap) and the `price` objective/metric (price_fit, minimise price) use the same
  resolved prices as `calc` (§3.3). `constraints.price.prices` (docs/21) keeps working and is merged into the request
  `prices.isk` table (docs/23 §5.3).
- `constraints.price.missing`: `error` (default; `OPT_MISSING_PRICE` lists the unpriced types) or `zero`.
- Results carry the base fit's `provenance.price_*` once in the response.

## 4. Price snapshot schema `eve-price-snapshot` v1 (C)

Produced by the updater (eve4), read by engines. UTF-8 JSON, optionally gzip (`.json.gz`, mtime 0).

### 4.1 File naming and versioning
- File: `prices-<market>-<YYYYMMDDTHHMMSSZ>.json.gz`, e.g. `prices-jita44-20261003T060000Z.json.gz`, where the
  time is `market_time` (§4.2) in UTC.
- `schema_version` (integer) follows the rules: additive optional fields → same version; any change to the meaning
  or type of an existing field, or a new required field → version + 1. Engines accept versions they know and
  reject others with `PRICE_SNAPSHOT_VERSION`.
- `snapshot_id` is unique per snapshot: `<market>-<YYYYMMDDTHHMMSSZ>`. Snapshots are immutable; a corrected one gets a
  new id.

### 4.2 Top-level fields

| field | type | required | meaning |
|---|---|---|---|
| `schema` | string | yes | `"eve-price-snapshot"` |
| `schema_version` | integer | yes | `1` |
| `snapshot_id` | string | yes | §4.1 |
| `market` | string | yes | `"jita44"` |
| `market_time` | string, RFC 3339 UTC (`Z`), seconds | yes | the market state the prices describe: the time the orders were fetched (ESI: the response's `Last-Modified`, max over pages and types) |
| `generated_at` | string, RFC 3339 UTC | yes | when the updater wrote the file |
| `source` | object | yes | §4.3 |
| `rule` | object | yes | §4.4 |
| `currency` | string | yes | `"ISK"` |
| `sde_build` | integer | yes | SDE build whose type list the updater priced |
| `type_count` | integer | yes | number of entries in `types` |
| `types` | object | yes | type id as decimal string → §4.5 |
| `missing` | array of integer | yes | type ids requested but without a qualifying order (sorted ascending) |
| `updater` | object | yes | `{ "name": string, "version": string }` |
| `content_hash` | string | yes | `"sha256:<64 lowercase hex>"`, §4.6 |

### 4.3 `source`
| field | type | meaning |
|---|---|---|
| `kind` | string | `"esi"` \| `"fuzzwork"` \| other registered plug-in names |
| `endpoint` | string | base URL or dataset name used |
| `region_id` | integer | 10000002 |
| `location_id` | integer | 60003760 |
| `fetched_from` / `fetched_to` | string RFC 3339 UTC | fetch window |
| `notes` | string, optional | free text (e.g. aggregate source limitations) |

A source that cannot supply per-order data (an aggregate feed) must still fill §4.5 per the rule; if it can only
approximate the rule it says so in `notes` and sets `rule.exact: false`.

### 4.4 `rule`
| field | type | default | meaning |
|---|---|---|---|
| `name` | string | `"jita_sell_band_weighted"` | rule id |
| `version` | integer | 1 | rule version |
| `order_side` | string | `"sell"` | |
| `location_id` | integer | 60003760 | |
| `min_units` | integer ≥ 1 | 10 | orders with `volume_remain < min_units` are dropped |
| `band` | number ≥ 0 | 0.05 | band upper bound = p0 × (1 + band) |
| `weighting` | string | `"units"` | weight = `volume_remain` |
| `exact` | boolean | true | false if the source only approximates the rule |

### 4.5 Per-type entry
| field | type | unit | meaning |
|---|---|---|---|
| `price` | number | ISK per unit | the snapshot price, rounded to 0.01 ISK (round half to even) |
| `p0` | number | ISK per unit | lowest price after the `min_units` filter |
| `band_max` | number | ISK per unit | p0 × (1 + band) |
| `units` | integer | units | Σ volume_remain of the orders in the band (the weights) |
| `orders` | integer | orders | number of orders in the band |
| `units_considered` | integer | units | Σ volume_remain of all orders left after the `min_units` filter |
| `orders_considered` | integer | orders | orders left after the `min_units` filter |
| `orders_total` | integer | orders | sell orders at the location before filtering |

Invariants (validated by engines on load, `PRICE_SNAPSHOT_INVALID` otherwise): `p0 ≤ price ≤ band_max`,
`0 < units ≤ units_considered`, `0 < orders ≤ orders_considered ≤ orders_total`, all numbers finite.

### 4.6 `content_hash`
SHA-256 over the canonical JSON of the whole object without the `content_hash` key: keys sorted, no insignificant
whitespace, UTF-8, integers without exponent, numbers as the shortest representation that round-trips an IEEE
double (Python `repr` / Rust `ryu`), no trailing zeros. Engines recompute it on load.

### 4.7 Example
```json
{
  "schema": "eve-price-snapshot", "schema_version": 1,
  "snapshot_id": "jita44-20261003T060000Z", "market": "jita44",
  "market_time": "2026-10-03T06:00:00Z", "generated_at": "2026-10-03T06:04:12Z",
  "source": { "kind": "esi", "endpoint": "https://esi.evetech.net/latest/markets/10000002/orders/",
              "region_id": 10000002, "location_id": 60003760,
              "fetched_from": "2026-10-03T05:58:40Z", "fetched_to": "2026-10-03T06:00:00Z" },
  "rule": { "name": "jita_sell_band_weighted", "version": 1, "order_side": "sell", "location_id": 60003760,
            "min_units": 10, "band": 0.05, "weighting": "units", "exact": true },
  "currency": "ISK", "sde_build": 3569502, "type_count": 1,
  "types": { "2889": { "price": 1251234.56, "p0": 1240000.0, "band_max": 1302000.0, "units": 412, "orders": 9,
                       "units_considered": 2310, "orders_considered": 31, "orders_total": 37 } },
  "missing": [],
  "updater": { "name": "eve-prices", "version": "0.1.0" },
  "content_hash": "sha256:…"
}
```

## 5. Errors and warnings
| code | when |
|---|---|
| `SDE_PACK_INVALID` | `--sde` / `sde_override`: bad magic, unknown `format_major`, hash mismatch, truncated section |
| `PRICE_SNAPSHOT_VERSION` | unknown `schema` / `schema_version` |
| `PRICE_SNAPSHOT_INVALID` | `content_hash` mismatch or an invariant of §4.5 broken |
| `BAD_PRICES` | request `prices`: unknown `mode`, non-numeric / negative / non-finite value, non-integer key |
| warning `price snapshot for SDE build X, engine data is Y` | snapshot `sde_build` ≠ data `sde_build` (allowed) |

## 6. Tests and gates
- eve3 (bench): embedded-version checks (`version` / `provenance` fields vs the release's pack and snapshot), the
  pricing rule on synthetic order books (for the updater), and price injection precedence (request override /
  replace, `--prices`, embedded) on `price` totals and the optimizer's price cap.
- eve-dogma CI: pack hash = generated-table hash; interpreter on the embedded pack byte-identical to the compiled
  path; `provenance` and `price` strip-listed for the round-1 identity check; no network access in tests (the
  engine has no HTTP client dependency; `cargo tree` check like the crate-boundary check).
- Updater (eve4): rule unit tests (min_units by units, band edges inclusive, ties, single order, no order),
  canonical-hash test vectors, schema validation of every published snapshot.

## 7. Open points for eve
1. Default `min_units` (10 proposed).
2. Whether `price` totals should include cargo by default (proposed: yes, separate line).
3. Snapshot cadence for releases (proposed: newest snapshot at release time; no snapshot older than 7 days in a
   release, else the release workflow fails).
