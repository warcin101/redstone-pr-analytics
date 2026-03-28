# RedStone Oracle Analytics — Sui

Dune-based analytics for RedStone price feed deployments on Sui mainnet: supply-side update monitoring and demand-side consumer detection.

---

## Overview

This folder contains Dune SQL queries and supporting documentation for monitoring RedStone oracles on the Sui blockchain.

Sui's architecture differs significantly from EVM. The key differences that affect oracle analytics:

- **All price feeds live inside a single shared object** — there is no per-feed contract address as on EVM. Feed identity is encoded in the `feed_id` field of the `PriceWrite` event.
- **Move function names are human-readable** — no 4-byte selector decoding needed.
- **Read-only access is invisible in `transaction_objects`** — only mutations are recorded there. Consumer detection requires parsing `transaction_json`.
- **Upgrades deploy a new package ID** — tracked via `sui.move_package` using `original_package_id`.

---

## Key Contracts and Addresses

| Role | Address |
|---|---|
| Price Adapter Package | `0xfa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6` |
| Relayer Package | `0xbd6c028d49d92e7e7f5cf268a2c91c0551f20a3d1f79d27d02482a71a9eb6ac3` |
| Shared Price Feed Object | `0x22794c3a37c5320e5acb6b9cdba6e256bc08867e9de8afd2a4b5d8ea7061fea3` |
| Relayer Wallet (primary) | `0xbd288ccf0f92df315f7b212e5481f4f2b469f6c61c0d58a16e616eb2e0341f9c` |
| Relayer Wallet (secondary) | `0xfcb82d9138f1aed43fd1259c94fe20890c2f48297dc673d354efd9f9572f0319` |

The shared price feed object is the central data structure. All feeds (BTC, ETH, LBTC_FUNDAMENTAL, etc.) are stored within it and distinguished by `feed_id`. Any consumer that reads a price must declare this object as an input in their transaction.

---

## Dune Queries

### Query 1 — Price Adapter Updates (Supply Side)

Monitors successful price writes to the RedStone price adapter. Shows update frequency per feed, latest values, and write timestamps. Uses the `PriceWrite` event from the price adapter package and filters for write-intent transactions (`mutability: Mutable`) to isolate relayer activity from consumer reads.

[View on Dune](https://dune.com/queries/6899856?category=canonical&namespace=sui&id=sui.objects)

```sql
WITH

-- Supply-side: all price update events with decoded feed names
price_writes AS (
  SELECT
    transaction_digest,
    TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))   AS feed_id,
    CAST(JSON_EXTRACT_SCALAR(event_json, '$.value') AS bigint)                 AS price_value,
    from_unixtime(
      CAST(JSON_EXTRACT_SCALAR(event_json, '$.timestamp') AS bigint) / 1000
    )                                                                          AS price_timestamp,
    from_unixtime(
      CAST(JSON_EXTRACT_SCALAR(event_json, '$.write_timestamp') AS bigint) / 1000
    )                                                                          AS write_timestamp,
    date
  FROM sui.events
  WHERE date >= CURRENT_DATE - INTERVAL '7' DAY
    AND event_type = '0xfa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6::price_adapter::PriceWrite'
)

SELECT
  feed_id,
  COUNT(DISTINCT transaction_digest)   AS total_updates,
  MIN(write_timestamp)                 AS first_update,
  MAX(write_timestamp)                 AS last_update,
  MAX(price_value)                     AS latest_raw_value
FROM price_writes
GROUP BY 1
ORDER BY total_updates DESC
```

**Standalone freshness check (latest value per feed):**

```sql
SELECT
  TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))   AS feed_id,
  from_unixtime(
    CAST(JSON_EXTRACT_SCALAR(event_json, '$.write_timestamp') AS bigint) / 1000
  )                                                                          AS last_write_time,
  CAST(JSON_EXTRACT_SCALAR(event_json, '$.value') AS bigint)                AS latest_value,
  COUNT(*) OVER (
    PARTITION BY TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))
  )                                                                          AS total_updates
FROM sui.events
WHERE date >= CURRENT_DATE - INTERVAL '1' DAY
  AND event_type = '0xfa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6::price_adapter::PriceWrite'
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))
  ORDER BY CAST(JSON_EXTRACT_SCALAR(event_json, '$.write_timestamp') AS bigint) DESC
) = 1
ORDER BY feed_id
```

---

### Query 2 — Full Analytics (Supply and Demand)

Combines supply-side update monitoring with demand-side consumer detection. Any wallet or protocol that references the shared price feed object in a transaction — whether to write or read — is captured. The `mutability` field in `transaction_json` distinguishes writers (relayers) from readers (DeFi consumers).

[View on Dune](https://dune.com/queries/6903784?sidebar=none)

```sql
WITH

-- Supply-side: all price update events with decoded feed names
price_writes AS (
  SELECT
    transaction_digest,
    TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))   AS feed_id,
    CAST(JSON_EXTRACT_SCALAR(event_json, '$.value') AS bigint)                 AS price_value,
    from_unixtime(
      CAST(JSON_EXTRACT_SCALAR(event_json, '$.timestamp') AS bigint) / 1000
    )                                                                          AS price_timestamp,
    from_unixtime(
      CAST(JSON_EXTRACT_SCALAR(event_json, '$.write_timestamp') AS bigint) / 1000
    )                                                                          AS write_timestamp,
    date
  FROM sui.events
  WHERE date >= CURRENT_DATE - INTERVAL '7' DAY
    AND event_type = '0xfa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6::price_adapter::PriceWrite'
),

-- Demand-side: all transactions referencing the shared price feed object
feed_interactions AS (
  SELECT
    transaction_digest,
    '0x' || to_hex(sender)                                AS caller,
    CASE
      WHEN transaction_json LIKE '%"mutability":"Mutable"%'   THEN 'write'
      WHEN transaction_json LIKE '%"mutability":"Immutable"%' THEN 'read'
      ELSE 'unknown'
    END                                                   AS interaction_type,
    date
  FROM sui.transactions
  WHERE date >= CURRENT_DATE - INTERVAL '7' DAY
    AND execution_success = true
    AND transaction_json LIKE '%22794c3a37c5320e5acb6b9cdba6e256bc08867e9de8afd2a4b5d8ea7061fea3%'
)

-- Combine: attach feed_id from events to each interaction
SELECT
  fi.interaction_type,
  COALESCE(pw.feed_id, 'unknown')         AS feed_id,
  fi.caller,
  COUNT(DISTINCT fi.transaction_digest)   AS txns,
  MIN(fi.date)                            AS first_seen,
  MAX(fi.date)                            AS last_seen
FROM feed_interactions fi
LEFT JOIN price_writes pw
  ON fi.transaction_digest = pw.transaction_digest
GROUP BY 1, 2, 3
ORDER BY interaction_type, txns DESC
```

**Reading the results:**

| `interaction_type` | Meaning |
|---|---|
| `write` | Relayer updating the price feed |
| `read` | External consumer reading the price (demand-side) |
| `unknown` | Edge case — investigate manually |

---

## How It Works

### Update flow

```
Relayer Package (0xbd6c028d...)
    └── calls try_write_price()
            |
            v
    Price Adapter Package (0xfa84e5...)
        └── module: price_adapter
             └── try_write_price()  →  mutates shared object, emits PriceWrite event
```

`sui.move_call` records the directly invoked package (the relayer). The price adapter is called internally and does not appear as a separate `move_call` row. To observe price writes, use `sui.events` filtered on `PriceWrite`.

### Consumer detection

Sui's consensus model requires every transaction to declare all shared object inputs upfront in `transaction_json`, regardless of whether the object is mutated or only read. This declaration includes a `mutability` field:

- `"mutability":"Mutable"` — the transaction intends to write (relayer)
- `"mutability":"Immutable"` — the transaction is read-only (consumer)

Because `sui.transaction_objects` only records mutations (`Created`, `Mutated`, `Deleted`), read-only access is invisible there. Scanning `transaction_json` via `LIKE` is the only reliable method for detecting consumers on Dune.

### Feed ID encoding

The `feed_id` in `PriceWrite` events is a Move `bytes32` field, null-padded to 32 bytes. Strip the padding before displaying or grouping:

```sql
TRIM(TRAILING CHR(0) FROM JSON_EXTRACT_SCALAR(event_json, '$.feed_id'))
```

---

## Table Reference

| Sui Table | EVM Equivalent | Key Columns |
|---|---|---|
| `sui.transactions` | `transactions` | `sender` (varbinary), `execution_success`, `timestamp_ms`, `transaction_json`, `date` |
| `sui.move_call` | `traces` (top-level only) | `package`, `module`, `function`, `transaction_digest`, `date` |
| `sui.events` | `logs` | `event_type` (human-readable), `event_json`, `sender`, `package`, `date` |
| `sui.objects` | *(no equivalent)* | `object_id`, `type_`, `owner_type`, `object_status`, `date` |
| `sui.transaction_objects` | *(no equivalent)* | `object_id`, `input_kind`, `object_status` — mutations only |
| `sui.move_package` | *(no equivalent)* | `package_id`, `original_package_id`, `package_version`, `date` |

### EVM vs Sui concept mapping

| EVM Concept | Sui Equivalent |
|---|---|
| Contract address (feed) | Shared Object ID |
| Implementation contract | Package ID |
| `traces.to` | `move_call.package` + `.module` |
| 4-byte function selector | `move_call.function` (already human-readable) |
| `traces.from` (caller) | `transactions.sender` — requires join |
| `tx_hash` | `transaction_digest` (varchar) |
| EVM logs | `sui.events` |
| Contract deployment | `sui.move_package` |

---

## Known Limitations

**Read-only access not in `transaction_objects`**
Only `Created`, `Mutated`, and `Deleted` statuses exist. A DeFi protocol reading the price without mutating the object leaves no trace there. `transaction_json` scanning is the workaround.

**`LIKE` on `transaction_json` is expensive at scale**
The text search is not indexed. Always include a `date` partition filter. For historical analysis, run in short time windows.

**`move_call` records the outer package only**
When the relayer calls into the price adapter internally, only the relayer package appears in `move_call`. Use `sui.events` to observe the inner execution.

**Upgrade tracking requires maintenance**
If the price adapter package is upgraded, the new `package_id` must be discovered via `sui.move_package` using `original_package_id`. Do not hardcode a single package ID in long-lived queries.

```sql
-- Enumerate all deployed versions of the price adapter
SELECT package_id, package_version, date
FROM sui.move_package
WHERE to_hex(original_package_id) = 'fa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6'
   OR to_hex(package_id) = 'fa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6'
ORDER BY package_version ASC
```

---

## Gotchas and Syntax Notes

**Table name is singular**
```sql
sui.move_call   -- correct
sui.move_calls  -- does not exist
```

**Reserved word column**
The object type column has a trailing underscore to avoid the SQL reserved word `type`:
```sql
sui.objects.type_   -- not .type
```

**Binary columns — filtering and display**
All address and ID columns are `varbinary`:
```sql
-- Filter (strip 0x prefix)
WHERE package = from_hex('fa84e5620fda505bbeb9178316a555e1db36f8932dee44210dd88a6464a76ef6')

-- Display
'0x' || to_hex(sender) AS sender
```

**Partition alignment on joins**
Always include `date` on both sides of a join between partitioned tables:
```sql
INNER JOIN sui.transactions t
  ON mc.transaction_digest = t.transaction_digest
  AND mc.date = t.date   -- prevents full scans
```

**Timestamps are in milliseconds**
```sql
from_unixtime(timestamp_ms / 1000) AS block_time
```

---

*Verified against live Sui mainnet data on Dune, March 2026.*
