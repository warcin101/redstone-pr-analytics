# RedStone Oracle Analytics — Stellar

Dune-based analytics for RedStone price feed deployments on Stellar mainnet: supply-side update monitoring and demand-side consumer detection.

---

## Overview

This folder contains Dune SQL queries and supporting documentation for monitoring RedStone oracles on the Stellar blockchain.

Stellar's architecture differs significantly from EVM. The key differences that affect oracle analytics:

- **All price writes go through a single shared adapter contract** — there is no per-feed write target as on EVM. The relayer calls `write_prices()` on the shared adapter, passing feed names as arguments.
- **Per-feed contracts exist but are read-only** — the 19 individual `C...` contract addresses from the RedStone API config are the consumer-facing interfaces. As of April 2026 no consumers have been observed calling them on-chain.
- **Stellar uses Soroban for smart contracts** — Soroban is Stellar's smart contract layer, written in Rust and compiled to WebAssembly. All oracle activity appears in `stellar.history_operations` as `invoke_host_function` operations.
- **The `function` column is useless** — it returns the host function type enum (`HostFunctionTypeHostFunctionTypeInvokeContract`), not the actual contract function name. The real function name lives at `parameters_decoded[2].value`.
- **Multiple relayer accounts** — four relayer accounts were discovered during exploration. All call `write_prices` on the shared adapter. Only filtering by a single known relayer will produce false consumer positives.

---

## Key Contracts and Addresses

| Role | Address |
|---|---|
| Shared Price Adapter | `CA526Y2NQWGWVVQ7RFFPGAZMU66PSYJ3UC2MTVAV4ZU7OM5BOPHDXUSG` |
| Relayer (primary) | `GASNOA72CECDUZ5GEUK6WFINSASEG6R3WYZB2DE2CGDU7YI7GC2QPSFX` |
| Relayer (secondary) | `GARZ4YWUOMCVPFTYI57N3TQEU6PM52RGG3Y46DVOBB4TG3TG7JORFQCK` |
| Relayer (tertiary) | `GBES67CMQHFXTDNO7NTW7IP5GHJ2K5NSLWXN5JP7BVQR7JLDIKBT3NTF` |
| Relayer (quaternary) | `GAKPQB2X5IL25YLG3YYESHI7EAXSU4FTMUECEQGJUJY3KBPVLYZY74U6` |

### Per-Feed Contracts (demand side)

| Feed | Contract Address |
|---|---|
| ETH | `CBC3S2VH7CM3DLGC62QXM4UD4YKCQULRLENAEEO7ZZ6U4OMZIQ5B777O` |
| BTC | `CBLEPHLNPTCW3DY2NF5BYCZVECIP2K6YWBA3JDTU3HMQV6AKXB5GKTXA` |
| PYUSD | `CB3PNR7FSAHQH3UAVYKSJVPLWEQRPPYVFL2UIR4P3RYEPZLTOF7PL2L4` |
| USDC | `CC2GB2TZTX24XIAM375YDWUDXA3PR2GDMNFUL2Z5VNXUMKPPOBES3CXM` |
| EUROC/EUR | `CD4H2CN6ROCXVZFDTMLFCSU4O5CKLJP5BCFECNMNZEATNABEDRVGFKNI` |
| iBENJI_ETHEREUM_FUNDAMENTAL | `CCFFK56RHKVSJF6ADIB35TLLRG7IN76H2WHO7P4SFVNYMLZOSVLLSDI2` |
| SolvBTC.BBN_FUNDAMENTAL | `CBCVJGBJGNPMYCEU3YVYQIH5Y6KSNATRLRLCZCMS26QTCUV367AD54BE` |
| SolvBTC_FUNDAMENTAL | `CCE4HNAVDIJJAJQYETON5CCER57MDYXLW45JEXYXI2QMASWFAHAPL5PT` |
| BENJI_ETHEREUM_FUNDAMENTAL | `CCDM2U5E3SBMPJUQCBIER4BSFAVSJTY5I5EQEZIRDZSA3YOJH3YM4TW5` |
| SPXU | `CBKC3LPRNTAOOJLNLG2GFKSMLLSDNHEUBVJO22Y2MY2RJLLYIXCRPK53` |
| CETES | `CCYGBQZ5JKE2TGK2XNNVIZRKFEIT27NZADS3XJFSO4CCIH3LHIUP34LG` |
| KTB | `CDF24LSSKG55VEN2MRKNHEWK3XT5YKQ6XRXNVTNSRBMFL2FOMAC2MZ5O` |
| USTRY | `CAEGYZFAJ3HLSDBUIVJ77SK5HLLS4MPE4LO5OB2W3L2TEDMAS6RCTMZ4` |
| TESOURO | `CAO3JDDO52PV5SNIR4WRLBNTE72CY6B4JO5PNSSRRM4TC5DZ5X5ZSJVC` |
| EUROB | `CAY3Q4DVBMS6OXWQWZWNOCVG7M3JNO6GL2PNZSQ6YH3NJOLW7O7IUWES` |
| GILTS | `CDIBUKDR3AUJFOG3Z5PH2UIKTNK6FQEUBODVH4FA7DST4NITPKKYHMAP` |
| MXNe | `CDYTKTOHHMMZQSBGEVTFSOAMN6ABFCRZ6TCNLZNFQZ6OK3ZGDB3666IV` |
| XLM | `CAXRDLYBL7ZNE754UX33XTVGTA7MNTIFWURTFJBTRKHPTNP2P36TU2ZF` |
| SolvBTC | `CA5ZDXNDNOR7RA6M6HAB5ZPITD2NU3DIW2AFJCBTV2MJHRUPQXNI2LSE` |

---

## Dune Queries

### Query 1 — Price Feed Updates (Supply Side)

Saved as: **PR calls Stellar | write view** — [View on Dune](https://dune.com/queries/7383755)

A proof of usage of the `write_prices` function by the RedStone relayer. Monitors calls on the shared adapter, broken down by relayer account. Used as a relayer health check — shows update cadence per relayer and will surface any new functions that appear on the adapter over time.

```sql
SELECT
  parameters_decoded[2].value AS function_name,
  source_account,
  COUNT(*) AS calls,
  MIN(closed_at) AS first_seen,
  MAX(closed_at) AS last_seen
FROM stellar.history_operations
WHERE closed_at_date >= CURRENT_DATE - INTERVAL '30' DAY
  AND type_string = 'invoke_host_function'
  AND contract_id = 'CA526Y2NQWGWVVQ7RFFPGAZMU66PSYJ3UC2MTVAV4ZU7OM5BOPHDXUSG'
GROUP BY 1, 2
ORDER BY calls DESC
```

**Expected output (as of April 2026):**

| function_name | source_account | calls |
|---|---|---|
| write_prices | GASN...PSFX | ~28000 |
| write_prices | GARZ...QCKK | ~1849 |
| write_prices | GBES...NNTF | ~153 |
| write_prices | GAKP...74U6 | ~32 |

---

### Query 2 — Price Feed Activity (Supply & Demand)

Saved as: **PR calls Stellar** — [View on Dune](https://dune.com/queries/7383739)

The universal view. Monitors both the shared adapter (supply side, visible as `write_prices` rows) and all 19 per-feed contracts (demand side, currently empty). This query will become the primary analytics view as RedStone adoption on Stellar grows -- any consumer calling a per-feed contract will appear here automatically. No filters applied -- relayer writes are intentionally visible as a Dune data sanity check, consistent with the EVM query approach.

```sql
SELECT
  contract_id,
  parameters_decoded[2].value AS function_name,
  source_account,
  COUNT(*) AS calls,
  MIN(closed_at) AS first_seen,
  MAX(closed_at) AS last_seen
FROM stellar.history_operations
WHERE closed_at_date >= CURRENT_DATE - INTERVAL '30' DAY
  AND type_string = 'invoke_host_function'
  AND contract_id IN (
    -- Shared price adapter (supply side / sanity check)
    'CA526Y2NQWGWVVQ7RFFPGAZMU66PSYJ3UC2MTVAV4ZU7OM5BOPHDXUSG',
    -- Per-feed contracts (demand side)
    'CBC3S2VH7CM3DLGC62QXM4UD4YKCQULRLENAEEO7ZZ6U4OMZIQ5B777O', -- ETH
    'CBLEPHLNPTCW3DY2NF5BYCZVECIP2K6YWBA3JDTU3HMQV6AKXB5GKTXA', -- BTC
    'CB3PNR7FSAHQH3UAVYKSJVPLWEQRPPYVFL2UIR4P3RYEPZLTOF7PL2L4', -- PYUSD
    'CC2GB2TZTX24XIAM375YDWUDXA3PR2GDMNFUL2Z5VNXUMKPPOBES3CXM', -- USDC
    'CD4H2CN6ROCXVZFDTMLFCSU4O5CKLJP5BCFECNMNZEATNABEDRVGFKNI', -- EUROC/EUR
    'CCFFK56RHKVSJF6ADIB35TLLRG7IN76H2WHO7P4SFVNYMLZOSVLLSDI2', -- iBENJI_ETHEREUM_FUNDAMENTAL
    'CBCVJGBJGNPMYCEU3YVYQIH5Y6KSNATRLRLCZCMS26QTCUV367AD54BE', -- SolvBTC.BBN_FUNDAMENTAL
    'CCE4HNAVDIJJAJQYETON5CCER57MDYXLW45JEXYXI2QMASWFAHAPL5PT', -- SolvBTC_FUNDAMENTAL
    'CCDM2U5E3SBMPJUQCBIER4BSFAVSJTY5I5EQEZIRDZSA3YOJH3YM4TW5', -- BENJI_ETHEREUM_FUNDAMENTAL
    'CBKC3LPRNTAOOJLNLG2GFKSMLLSDNHEUBVJO22Y2MY2RJLLYIXCRPK53', -- SPXU
    'CCYGBQZ5JKE2TGK2XNNVIZRKFEIT27NZADS3XJFSO4CCIH3LHIUP34LG', -- CETES
    'CDF24LSSKG55VEN2MRKNHEWK3XT5YKQ6XRXNVTNSRBMFL2FOMAC2MZ5O', -- KTB
    'CAEGYZFAJ3HLSDBUIVJ77SK5HLLS4MPE4LO5OB2W3L2TEDMAS6RCTMZ4', -- USTRY
    'CAO3JDDO52PV5SNIR4WRLBNTE72CY6B4JO5PNSSRRM4TC5DZ5X5ZSJVC', -- TESOURO
    'CAY3Q4DVBMS6OXWQWZWNOCVG7M3JNO6GL2PNZSQ6YH3NJOLW7O7IUWES', -- EUROB
    'CDIBUKDR3AUJFOG3Z5PH2UIKTNK6FQEUBODVH4FA7DST4NITPKKYHMAP', -- GILTS
    'CDYTKTOHHMMZQSBGEVTFSOAMN6ABFCRZ6TCNLZNFQZ6OK3ZGDB3666IV', -- MXNe
    'CAXRDLYBL7ZNE754UX33XTVGTA7MNTIFWURTFJBTRKHPTNP2P36TU2ZF', -- XLM
    'CA5ZDXNDNOR7RA6M6HAB5ZPITD2NU3DIW2AFJCBTV2MJHRUPQXNI2LSE'  -- SolvBTC
  )
GROUP BY 1, 2, 3
ORDER BY calls DESC
```

**Reading the results:**

| `contract_id` | `function_name` | Meaning |
|---|---|---|
| Shared adapter | `write_prices` | Relayer updating prices (supply side) |
| Per-feed contract | anything | DeFi consumer reading a price (demand side) |

---

## How It Works

### Update flow

```
Relayer Account (GASN...PSFX, GARZ...QCKK, etc.)
    └── calls write_prices(relayer_account, [feed_names], payload)
            |
            v
    Shared Price Adapter (CA526Y2N...)
        └── stores price data, accessible per feed via individual C... contracts
```

Unlike EVM where each adapter is a separate contract, Stellar uses a single shared adapter. Feed names are passed as string arguments in the call, so one transaction can update multiple feeds (e.g. `["ETH", "BTC"]`).

### Consumer detection

On EVM, consumers call per-feed adapter contracts directly -- visible in `ethereum.traces` with `to` = the adapter address. On Stellar, the equivalent is a consumer calling one of the 19 per-feed `C...` contracts via `invoke_host_function`. This will appear in `stellar.history_operations` with `contract_id` = the per-feed address and `source_account` = the consumer wallet or protocol.

As of April 2026 no consumers have been observed. The demand-side query is in place and will surface activity automatically when it appears.

### Why the `function` column cannot be used

On EVM, the function selector is decoded from `input` bytes. On Stellar, the `function` column in `history_operations` returns the Soroban host function type enum -- always `HostFunctionTypeHostFunctionTypeInvokeContract` for contract calls -- which is useless for distinguishing `write_prices` from a read function. The actual function name is encoded in the `parameters` array and must be extracted via:

```sql
parameters_decoded[2].value AS function_name
```

The `parameters_decoded` array structure for a Soroban contract invocation is:
- `[1]` — contract address (Address type)
- `[2]` — function name (Sym type)
- `[3+]` — function arguments

---

## Exploration Log

This section documents what was tried during the initial analytics build, including dead ends, so future work starts from the right place.

**Attempt 1 — Query per-feed contracts directly**
Queried all 19 individual `C...` feed addresses in `history_operations`. Returned zero results. Confirmed that the relayer never writes to individual feed contracts.

**Attempt 2 — Add shared adapter, filter one relayer**
Added the shared adapter to the `IN` list and excluded `GASN...PSFX` as the only known relayer. Returned 2034 calls from `GARZ...` and `GBES...`, which appeared to be consumer activity. This was a false positive -- those accounts are also relayers.

**Attempt 3 — Remove source_account filter, group by function**
Dropped all account filters and grouped by `parameters_decoded[2].value`. Revealed 4 relayer accounts, all calling `write_prices` exclusively. Confirmed no consumer activity exists yet.

**Dead end — `function` column**
Early queries used the `function` column expecting a human-readable name as on Sui. Returns only the host function enum. Must use `parameters_decoded[2].value` instead.

**Dead end — per-feed contracts as write targets**
The RedStone internal API (`/feeds/push/all`) returns 19 individual `C...` contract addresses. Initial assumption was these were per-feed adapters the relayer writes to (EVM mental model). They are read-only consumer interfaces. The relayer only ever touches the single shared adapter.

---

## Table Reference

| Stellar Table | EVM Equivalent | Key Columns |
|---|---|---|
| `stellar.history_operations` | `ethereum.traces` | `contract_id`, `source_account`, `function`, `parameters`, `parameters_decoded`, `type_string`, `closed_at`, `closed_at_date` |
| `stellar.history_contract_events` | `ethereum.logs` | `contract_id`, `topic`, `value`, `transaction_hash`, `closed_at` |
| `stellar.history_transactions` | `ethereum.transactions` | `transaction_hash`, `account`, `successful`, `closed_at` |
| `stellar.history_ledgers` | `ethereum.blocks` | `sequence`, `closed_at` |

### EVM vs Stellar concept mapping

| EVM Concept | Stellar Equivalent |
|---|---|
| Contract address (per-feed adapter) | Per-feed `C...` contract (read only) |
| Shared backing store | Shared price adapter (`CA526Y2N...`) |
| `traces.to` | `history_operations.contract_id` |
| 4-byte function selector | `parameters_decoded[2].value` (human-readable) |
| `traces.from` (caller) | `history_operations.source_account` |
| `tx_hash` | `transaction_hash` (varchar) |
| EVM logs | `stellar.history_contract_events` |
| `updateDataFeedsValues()` | `write_prices()` |
| `latestRoundData()` / `latestAnswer()` | Unknown — no consumer observed yet |

---

## Known Limitations

**No consumer activity observed yet**
As of April 2026 the per-feed contracts have never been called by a non-relayer account. The demand-side query is ready but will return zero rows until adoption begins.

**Read function name unknown**
Because no consumer has called a per-feed contract, the read function name(s) have not been observed in the wild. When the first consumer appears, check `parameters_decoded[2].value` on the per-feed contract call to identify it.

**Multiple relayer accounts**
Four relayer accounts were discovered, with the primary handling ~93% of volume. This list may grow. If a new relayer is added and only the `source_account NOT IN (...)` pattern is used, it will surface as fake consumer activity. The combined query avoids this by including no source_account filter and relying on `contract_id` to distinguish supply from demand.

**`parameters_decoded` index is 1-based**
Array indexing in Trino/Dune is 1-based. `parameters_decoded[2]` is the second element (function name), not the third.

**`closed_at_date` required for partition pruning**
Always include `closed_at_date` in the `WHERE` clause. Without it the query will full-scan the table.

---

*Verified against live Stellar mainnet data on Dune, April 2026.*
