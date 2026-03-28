# RedStone Oracle Analytics -- Solana

Dune-based analytics for RedStone price feed deployments on Solana mainnet: supply-side update monitoring and demand-side consumer detection.

---

## Overview

This folder contains Dune SQL queries and supporting documentation for monitoring RedStone oracles on the Solana blockchain.

Solana's architecture differs significantly from both EVM and Sui. The key differences that affect oracle analytics:

- **Each price feed has its own PDA (Program Derived Address)** -- unlike Sui's single shared object, and unlike EVM's per-feed contract address. The PDA is derived from the program and feed_id, and stores the price data for that feed.
- **Feed identity is encoded in instruction data** -- the feed_id is a bytes32 null-padded UTF8 string embedded in the `WritePrice` instruction payload, starting at byte 9 (after the 8-byte Anchor discriminator).
- **No structured events** -- Solana programs emit log messages, not typed event objects. The RedStone program logs `"Instruction: WritePrice"` but does not emit decoded event data like Sui's `PriceWrite` event. Feed IDs and values must be parsed from raw instruction data.
- **Read-only account access is captured in `account_activity`** -- unlike Sui's `transaction_objects` (mutations only), Solana's `account_activity` table records both read (`writable = false`) and write (`writable = true`) interactions. This is the key enabler for demand-side detection.
- **Addresses are base58 strings** -- no hex encoding, no `0x` prefix, no `from_hex()`/`to_hex()` conversions needed.

---

## Key Addresses

| Role | Address |
|---|---|
| RedStone Oracle Program | `REDSTBDUecGjwXd6YGPzHSvEUBHQqVRfCcjUVgPiHsr` |
| Relayer Wallet (primary) | `DBAs8rAXQUpBZC2FBgxaYNEmLGynpAhMq8qhtCmzamCP` |
| Relayer Wallet (secondary) | `9kNw7uYTMiWmQhFk25dbWt9wNF4C88EszCpnJnU1pKTx` |

### Active Price Feed PDAs

| Feed ID | PDA |
|---|---|
| ETH | `HPmPoq3eUTPePsDB5U4G6msu5RpeZHhMemc5VnqxQ9Lx` |
| BTC | `74o5fhuMC33HgfUqvv2TdpYiKvEWfcRTS1E8zxK6ESjN` |
| ACRED_SOLANA_FUNDAMENTAL | `6sK8czVw8Xy6T8YbH6VC8p5ovNZD2mXf5vUTv8sgnUJf` |
| KTB | `7LpHkWowUSxwqpYEX71juUpNszR7EZ3ZiFnwt2fj3ubc` |
| MXNe | `8QuwResPgdQdja4gdTbvNxeUznc58XX8baUM3cCwfVW2` |
| CETES | `BBQbMAJxEjJQcPigdTDbMphzGoasBDKfsGKDNjuw41sn` |
| TESOURO | `Fx7SbwDDiir4C42Q1ESH4cm8NGcDjQucgA7QAHFso8ob` |
| GILTS | `7MNCZy8ucFRLzRitc24tBeXRNuzmfTKu469riUYSACq1` |
| EUROB | `HJpgmbTmVG49iLtYqQWacBqBs1XxVbAGAmpcB2ANhK4e` |
| VBILL_SOLANA_DAILY_ACCRUAL | `8aMjLzsmjSEGmgybHWELTEinTw4kn3okfos5zHrt2TJG` |
| BUIDL_SOLANA_DAILY_ACCRUAL | `CPKJ57Kvxf8Xrz1o3hqBK52SqqEUAPp1NVdCK94bDGSX` |
| BUIDL_SOLANA_FUNDAMENTAL | `ESxdEASDcYRN4ybnYNCJJuPHcF2SGJN1MypQq1yfY9Kz` |
| LBTC_FUNDAMENTAL | `EjN4MxDvAbBEofxFwpvCsz2u8wZ96kHMsLmk2N5Bvomt` |
| USTRY | `FQfn746CH96toaGvFAP4FaN1rhvcEPrH4AGjWcSMhP1P` |
| VBILL_SOLANA_FUNDAMENTAL | `H9ckR9pZtPZCjRYz9RXBPhc2X6m4e3ndpxgVe7HgYVMd` |

The deployment is heavily RWA-focused, reflecting RedStone's partnership with Securitize and integration with Drift and Kamino.

---

## Dune Queries

### Query 1 -- Supply Side (WritePrice Monitoring)

Monitors all `WritePrice` instructions to the RedStone Oracle Program. Extracts the feed_id from instruction data, identifies the relayer wallet, and counts update frequency per feed.

```sql
SELECT
  account_arguments[2]                                                        AS feed_pda,
  TRIM(TRAILING CHR(0) FROM FROM_UTF8(bytearray_substring(data, 9, 32)))      AS feed_id,
  tx_signer                                                                    AS relayer,
  COUNT(*)                                                                     AS total_writes,
  MIN(block_time)                                                              AS first_write,
  MAX(block_time)                                                              AS last_write
FROM solana.instruction_calls
WHERE block_date >= CURRENT_DATE - INTERVAL '7' DAY
  AND executing_account = 'REDSTBDUecGjwXd6YGPzHSvEUBHQqVRfCcjUVgPiHsr'
  AND tx_success = true
  AND NOT is_inner
GROUP BY 1, 2, 3
ORDER BY total_writes DESC
```

**Performance note:** Grouping by `account_arguments[2]` is expensive due to per-row array indexing. If you only need feed_id and relayer without the PDA address, drop column 1 from the SELECT and GROUP BY for a faster query.

---

### Query 2A -- Full Analytics: Read/Write Volume per Feed

Shows read vs write interaction counts per feed across all PDAs. Uses `account_activity` joined to the feed PDA lookup from Query 1. This is the primary monitoring query.

[View on Dune](https://dune.com/queries/6918613)

**Why two queries instead of one:** On Solana, combining the volume overview with consumer program resolution in a single query consistently times out or hits Trino's optimizer limits. The `contains(account_arguments, ...)` join required to resolve which program performed a read is extremely expensive when applied across all feeds and all of `instruction_calls`. Dune's Trino optimizer specifically fails on `ReorderJoins` for this pattern, and `COUNT(DISTINCT tx_id)` combined with `MIN`/`MAX` triggers `OptimizeMixedDistinctAggregations` timeouts. The solution is to split the work: Query 2A gives you the full volume picture fast, and Query 2B resolves consumer programs per feed individually.

```sql
WITH feed_pdas AS (
  SELECT
    account_arguments[2]                                                        AS feed_pda,
    TRIM(TRAILING CHR(0) FROM FROM_UTF8(bytearray_substring(data, 9, 32)))      AS feed_id
  FROM solana.instruction_calls
  WHERE block_date >= CURRENT_DATE - INTERVAL '7' DAY
    AND executing_account = 'REDSTBDUecGjwXd6YGPzHSvEUBHQqVRfCcjUVgPiHsr'
    AND tx_success = true
    AND NOT is_inner
  GROUP BY 1, 2
)

SELECT
  fp.feed_id,
  CASE WHEN aa.writable THEN 'write' ELSE 'read' END   AS interaction_type,
  COUNT(*)                                               AS txns,
  MIN(aa.block_time)                                     AS first_seen,
  MAX(aa.block_time)                                     AS last_seen
FROM solana.account_activity aa
INNER JOIN feed_pdas fp ON aa.address = fp.feed_pda
WHERE aa.block_date >= CURRENT_DATE - INTERVAL '7' DAY
  AND aa.tx_success = true
GROUP BY 1, 2
ORDER BY fp.feed_id, txns DESC
```

---

### Query 2B -- Consumer Program Resolution (per feed)

Identifies which programs and wallets are consuming a specific price feed. Run this separately for each feed that shows `read` activity in Query 2A. Replace the PDA address with the target feed's PDA.

[View on Dune](https://dune.com/queries/6918696)

```sql
-- Example: ACRED_SOLANA_FUNDAMENTAL consumers
SELECT
  ic.executing_account          AS program,
  ic.tx_signer                  AS caller,
  COUNT(*)                      AS txns
FROM solana.instruction_calls ic
WHERE ic.block_date >= CURRENT_DATE - INTERVAL '7' DAY
  AND ic.tx_success = true
  AND contains(ic.account_arguments, '6sK8czVw8Xy6T8YbH6VC8p5ovNZD2mXf5vUTv8sgnUJf')
  AND ic.executing_account != 'REDSTBDUecGjwXd6YGPzHSvEUBHQqVRfCcjUVgPiHsr'
  AND ic.executing_account != 'ComputeBudget111111111111111111111111111111'
GROUP BY 1, 2
ORDER BY txns DESC
```

To check a different feed, swap the PDA address. Refer to the feed PDA table above for the mapping.

**Combining into Dune's double query view:** These two queries are designed to sit side by side in Dune's split view. Query 2A on the left shows the volume overview across all feeds. Query 2B on the right shows consumer detail for whichever feed you're investigating. This split approach runs in seconds rather than timing out.

**Reading the results:**

| `interaction_type` | Meaning |
|---|---|
| `write` | Relayer updating the price feed (supply side) |
| `read` | External consumer reading the price (demand side) |

---

## How It Works

### Update flow

```
Relayer Wallet (DBAs8rAX...)
    └── signs transaction containing WritePrice instruction
            |
            v
    RedStone Oracle Program (REDSTBDUe...)
        └── WritePrice instruction
             ├── reads feed_id from instruction data
             ├── writes price value to feed PDA
             └── logs "Instruction: WritePrice"
```

Each feed has its own PDA. The relayer calls the program directly -- there is no inner CPI, so `is_inner = false` cleanly isolates these writes. The program ID appears as `executing_account` in `instruction_calls`.

### Consumer detection

On Solana, any program that reads price data must include the feed PDA in the transaction's account list. The `account_activity` table records every account referenced in a transaction, along with a `writable` flag:

- `writable = true` -- the transaction mutates the account (relayer writing a price)
- `writable = false` -- the transaction reads the account without mutating it (consumer)

This is analogous to Sui's `"mutability":"Mutable"` vs `"mutability":"Immutable"` distinction in `transaction_json`, but with a critical advantage: `account_activity` is indexed on `address`, making lookups fast. On Sui, consumer detection requires an expensive `LIKE` scan on `transaction_json`.

### Feed ID encoding

The feed_id is embedded in the `WritePrice` instruction data as a bytes32 null-padded UTF8 string, starting at byte offset 9 (after the 8-byte Anchor discriminator):

```sql
TRIM(TRAILING CHR(0) FROM FROM_UTF8(bytearray_substring(data, 9, 32)))
```

Note the order: `FROM_UTF8` converts the varbinary to varchar first, then `TRIM` removes the null padding. Reversing this order fails because `TRIM` cannot operate on varbinary. This is the same encoding pattern as Sui, with the same fix.

### Anchor discriminator

All `WritePrice` instructions share the same 8-byte Anchor discriminator: `0x10ba78e076b2a198`. This can be used as an additional filter if the program ever supports multiple instruction types:

```sql
AND bytearray_substring(data, 1, 8) = 0x10ba78e076b2a198
```

Currently only one instruction type has been observed, so this filter is not required but may be useful for future-proofing.

---

## Table Reference

| Solana Table | EVM Equivalent | Sui Equivalent | Key Columns |
|---|---|---|---|
| `solana.instruction_calls` | `traces` | `sui.move_call` + `sui.events` | `executing_account`, `data`, `account_arguments`, `tx_signer`, `tx_id`, `block_date` |
| `solana.account_activity` | *(no equivalent)* | *(no equivalent -- `transaction_objects` is mutations only)* | `address`, `writable`, `tx_id`, `block_date` |
| `solana.transactions` | `transactions` | `sui.transactions` | `signer`, `id`, `success`, `account_keys`, `block_date` |

### EVM vs Solana concept mapping

| EVM Concept | Solana Equivalent |
|---|---|
| Contract address (feed) | PDA (Program Derived Address) per feed |
| Implementation contract | Program ID (`REDSTBDUe...`) |
| `traces.to` | `instruction_calls.executing_account` |
| 4-byte function selector | 8-byte Anchor discriminator |
| `traces.from` (caller) | `instruction_calls.tx_signer` -- no join needed |
| `traces.input` | `instruction_calls.data` (varbinary) |
| `tx_hash` | `tx_id` (base58 string) |
| EVM logs / events | `log_messages` array (unstructured text) |
| `latestRoundData()` / `getValueForDataFeed()` | Direct PDA account read (no function call) |

### Sui vs Solana concept mapping

| Sui Concept | Solana Equivalent |
|---|---|
| Shared Object (single, all feeds) | Per-feed PDA |
| `PriceWrite` event in `sui.events` | `WritePrice` instruction in `instruction_calls` |
| `event_json` field parsing | `bytearray_substring(data, ...)` parsing |
| `transaction_json LIKE '%object_id%'` | `account_activity.address = '<pda>'` (much faster) |
| `"mutability":"Mutable"` / `"Immutable"` | `account_activity.writable` = true / false |
| `transaction_objects` (mutations only) | `account_activity` (both reads and writes) |
| `move_call.package` (outer only) | `instruction_calls.executing_account` (all levels) |
| Binary addresses with `from_hex()` / `to_hex()` | Base58 strings, no conversion needed |

---

## Approaches Tried and Lessons Learned

### What worked

**`account_activity` for demand-side detection.** This was the key discovery. Unlike Sui's `transaction_objects` (which only records mutations), Solana's `account_activity` records both read and write interactions. The `writable` column directly distinguishes consumers from relayers, and the table is indexed on `address` for fast lookups. This is a major improvement over the Sui approach, which requires expensive `LIKE` scans on `transaction_json`.

**Feed ID extraction from instruction data.** The bytes32 null-padded UTF8 encoding is identical to Sui. The `FROM_UTF8` then `TRIM` pattern works the same way.

**`instruction_calls` for supply-side monitoring.** Filtering by `executing_account` is fast and gives all the data needed: feed PDA from `account_arguments[2]`, feed_id from `data`, relayer from `tx_signer`.

### What didn't work

**`account_activity` for ETH feed consumers.** Initial testing against the ETH feed PDA returned only `writable = true` rows, which initially suggested the table only captures mutations (like Sui's `transaction_objects`). Testing against ACRED -- which has an active consumer (Kamino) -- revealed that read-only access IS captured. ETH simply had no consumers. Lesson: always test assumptions against a feed you know has active demand.

**`call_is_inner` column name.** Does not exist. The correct column is `is_inner` (boolean). The `inner_instruction_index IS NULL` workaround also works but `NOT is_inner` is cleaner.

**`FROM_UTF8(TRIM(TRAILING 0x00 FROM ...))` ordering.** `TRIM` operates on varchar, not varbinary. The conversion must happen first: `TRIM(TRAILING CHR(0) FROM FROM_UTF8(...))`.

**`account_arguments[2]` in GROUP BY.** Array indexing on every row is expensive on Solana's massive instruction_calls table. For aggregation queries, extract feed_id from instruction data instead and map to PDAs separately if needed.

**`contains(account_arguments, '<pda>')` on full `instruction_calls` scan.** Running this against a one-day window took 8+ minutes with no results for ETH (because there were no consumers). The `account_activity` approach is dramatically faster because it's indexed on address. Use `contains()` only after narrowing the dataset via `account_activity` first.

**Combined supply+demand+program query in a single pass.** Multiple attempts to combine the volume overview (from `account_activity`) with consumer program resolution (from `instruction_calls`) into one query failed due to Trino optimizer limits:

- `COUNT(DISTINCT tx_id)` combined with `MIN`/`MAX` triggers `OptimizeMixedDistinctAggregations` timeout (240s limit). Fix: replace with `COUNT(*)` where possible.
- Even after fixing that, the `contains(account_arguments, feed_pda)` join between the interactions CTE and `instruction_calls` triggers `ReorderJoins` timeout. Trino cannot plan an efficient join when the condition is an array containment check across a massive table.
- Splitting writes (no join needed) from reads (join needed) via `UNION ALL` did not help -- the optimizer still choked on the reads portion.

The solution is to run two separate queries: one for volume (Query 2A, fast) and one for program resolution per feed (Query 2B, targeted). This is a fundamental Dune/Solana constraint, not a query design issue.

### Performance hierarchy

From fastest to slowest for demand-side detection:

1. **`account_activity` filtered by `address`** -- indexed, sub-minute for 7-day windows
2. **`account_activity` joined to `transactions`** -- adds signer resolution, moderate cost
3. **`instruction_calls` with `contains()` on a single PDA** -- targeted program resolution, minutes but workable
4. **`instruction_calls` with `contains()` joined across multiple PDAs** -- times out on Dune, not viable as a single query

---

## Known Limitations

**No structured events.** Unlike Sui's `PriceWrite` events with typed JSON fields, Solana's RedStone program only emits log messages. Price values must be parsed from raw instruction data. If the instruction data layout changes, the byte offsets in queries will break.

**IDL not decoded on Dune.** If Dune adds decoded tables for the RedStone Oracle Program (namespace `redstone_solana`), the raw byte parsing can be replaced with named columns. Check periodically or submit the IDL for decoding.

**`account_activity` does not identify the consumer program.** It shows which transactions reference the PDA and whether the access was read or write, but not which program within the transaction performed the read. Resolving the consumer program requires joining to `instruction_calls` with `contains(account_arguments, '<pda>')`, which is significantly slower.

**Solana query volume.** Solana produces far more transactions than Sui or most EVM chains. Always use `block_date` partition filters. Avoid full-table scans on `instruction_calls` or `transactions`. Prefer `account_activity` for address-based lookups.

---

## Gotchas and Syntax Notes

**Addresses are base58 strings, not hex.**
```sql
-- Solana: no conversion needed
WHERE executing_account = 'REDSTBDUecGjwXd6YGPzHSvEUBHQqVRfCcjUVgPiHsr'

-- Sui: binary columns require conversion
WHERE package = from_hex('fa84e5620fda...')
```

**`is_inner` not `call_is_inner`**
```sql
AND NOT is_inner           -- correct
AND NOT call_is_inner      -- does not exist
```

**`tx_signer` on `instruction_calls`, `signer` on `transactions`**
```sql
-- instruction_calls
SELECT tx_signer FROM solana.instruction_calls

-- transactions
SELECT signer FROM solana.transactions
```

**Transaction ID column differs by table**
```sql
-- instruction_calls, account_activity
WHERE tx_id = '...'

-- transactions
WHERE id = '...'
```

**Partition alignment on joins.**
Always include `block_date` on both sides of a join between partitioned tables:
```sql
INNER JOIN solana.transactions t
  ON i.tx_id = t.id
  AND i.block_date = t.block_date   -- prevents full scans
```

**Array indexing is 1-based.**
```sql
account_arguments[1]   -- first element (signer)
account_arguments[2]   -- second element (feed PDA)
```

**`FROM_UTF8` before `TRIM` for feed_id decoding.**
```sql
-- Correct: convert to string first, then trim
TRIM(TRAILING CHR(0) FROM FROM_UTF8(bytearray_substring(data, 9, 32)))

-- Wrong: TRIM cannot operate on varbinary
FROM_UTF8(TRIM(TRAILING 0x00 FROM bytearray_substring(data, 9, 32)))
```

**`COUNT(*)` instead of `COUNT(DISTINCT tx_id)` for supply-side.**
With `NOT is_inner`, each row in `instruction_calls` is one instruction per transaction. `COUNT(*)` is sufficient and faster.

---

*Verified against live Solana mainnet data on Dune, March 2026.*
