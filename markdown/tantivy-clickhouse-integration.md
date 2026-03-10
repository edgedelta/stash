# How Tantivy Integrates with ClickHouse

## ClickHouse Storage Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ClickHouse Table                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Table = Collection of "Parts"                                          │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │
│  │   Part 1    │  │   Part 2    │  │   Part 3    │  ...                │
│  │ (immutable) │  │ (immutable) │  │ (immutable) │                     │
│  └─────────────┘  └─────────────┘  └─────────────┘                     │
│                                                                         │
│  Each Part contains:                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Part Directory: all_1_1_0/                                      │   │
│  │  ├── primary.idx          (primary key index)                    │   │
│  │  ├── timestamp.bin        (column data)                          │   │
│  │  ├── message.bin          (column data)                          │   │
│  │  ├── service.bin          (column data)                          │   │
│  │  ├── skp_idx_fts.meta     (Tantivy index metadata)    ← NEW      │   │
│  │  ├── skp_idx_fts.data     (Tantivy index data)        ← NEW      │   │
│  │  └── ...                                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What is a Granule?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Part Structure                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  A Part is divided into "Granules" (default: 8192 rows each)            │
│                                                                         │
│  Part (e.g., 1 million rows)                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Granule 0: rows 0-8191        │ primary.idx[0] = min(timestamp) │   │
│  │ Granule 1: rows 8192-16383    │ primary.idx[1] = min(timestamp) │   │
│  │ Granule 2: rows 16384-24575   │ primary.idx[2] = min(timestamp) │   │
│  │ ...                           │ ...                              │   │
│  │ Granule N: rows ...           │ primary.idx[N] = min(timestamp) │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Granules are the UNIT of skipping in ClickHouse                        │
│  Skip index says: "Can I skip this entire granule?"                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ClickHouse's Full-Text Search Problem

```
Without Tantivy (native ClickHouse):

Query: SELECT * FROM logs WHERE message LIKE '%connection refused%'

Execution:
  Part 1: Scan ALL rows, check each message  → SLOW
  Part 2: Scan ALL rows, check each message  → SLOW
  Part 3: Scan ALL rows, check each message  → SLOW

Problem: O(n) scan of every row, even if only 10 rows match!
```

---

## How Tantivy Helps

```
With Tantivy Skip Index:

┌─────────────────────────────────────────────────────────────────────────┐
│                    What Tantivy Stores                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Inverted Index (per Part):                                             │
│                                                                         │
│  Term            →  Document IDs (row numbers in this part)             │
│  ─────────────────────────────────────────────────────────              │
│  "connection"    →  [0, 5, 102, 8195, 8200, 50000]                      │
│  "refused"       →  [5, 102, 8200]                                      │
│  "timeout"       →  [10, 20, 8192, 8193]                                │
│  "error"         →  [0, 1, 2, 5, 10, 102, 8192, ...]                    │
│                                                                         │
│  Files stored in Part directory:                                        │
│  ├── skp_idx_fts.meta   (term dictionary, offsets)                      │
│  └── skp_idx_fts.data   (posting lists, positions)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Query Execution Flow

```
Query: WHERE message LIKE '%connection refused%'

┌─────────────────────────────────────────────────────────────────────────┐
│                      Execution Flow                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Step 1: Query Tantivy Index                                            │
│  ─────────────────────────────                                          │
│  "connection" AND "refused"                                             │
│       ↓                                                                 │
│  Tantivy returns: Row IDs [5, 102, 8200]  (Roaring Bitmap)              │
│                                                                         │
│  Step 2: Map Row IDs to Granules                                        │
│  ───────────────────────────────                                        │
│  Row 5     → Granule 0  (rows 0-8191)                                   │
│  Row 102   → Granule 0  (rows 0-8191)                                   │
│  Row 8200  → Granule 1  (rows 8192-16383)                               │
│                                                                         │
│  Result: Only need to read Granules 0 and 1!                            │
│          Skip Granules 2, 3, 4, ... N                                   │
│                                                                         │
│  Step 3: Read Only Matching Granules                                    │
│  ───────────────────────────────────                                    │
│  Read Granule 0 → filter rows [5, 102]                                  │
│  Read Granule 1 → filter rows [8200]                                    │
│  Skip all other granules!                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   CREATE TABLE logs (                                                   │
│       timestamp DateTime64,                                             │
│       service String,                                                   │
│       message String,                                                   │
│       INDEX fts_idx message TYPE fts('{}')   ← Tantivy index            │
│   ) ENGINE = MergeTree()                                                │
│   ORDER BY timestamp;                                                   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   INSERT (writes to ClickHouse):                                        │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  1. ClickHouse writes column data (.bin files)                  │  │
│   │  2. For each row, extract "message" column                      │  │
│   │  3. Call Tantivy: index(row_id, message_text)                   │  │
│   │  4. On commit: write skp_idx_fts.meta + skp_idx_fts.data        │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   SELECT (reads from ClickHouse):                                       │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  1. Parse WHERE clause, find text search predicates             │  │
│   │  2. Query Tantivy: search("connection refused")                 │  │
│   │  3. Get back: Roaring Bitmap of matching row IDs                │  │
│   │  4. Convert row IDs → granule IDs to read                       │  │
│   │  5. Read only those granules from column files                  │  │
│   │  6. Apply remaining filters on the reduced dataset              │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What Gets Stored Where

| Component | Storage | Contents |
|-----------|---------|----------|
| **ClickHouse columns** | `.bin` files | Actual log data (compressed columnar) |
| **Primary index** | `primary.idx` | Min/max of ORDER BY key per granule |
| **Tantivy index** | `skp_idx_fts.data` | Inverted index (term → row IDs) |
| **Tantivy metadata** | `skp_idx_fts.meta` | Term dictionary, segment info |
| **Granule mapping** | `.idx` file | Which row IDs belong to which granule |

---

## Integration Options

### Option 1: Add Tantivy as Skip Index (MyScaleDB approach)

```sql
-- Modify your existing table
ALTER TABLE logs ADD INDEX fts_idx (message) TYPE fts('{}');

-- Queries automatically use it
SELECT * FROM logs
WHERE TextSearch(message, 'connection refused')
AND timestamp > now() - INTERVAL 1 HOUR;
```

### Option 2: External Tantivy Index (DIY approach)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   ClickHouse                         Tantivy (external)                 │
│   ┌─────────────┐                   ┌─────────────────┐                │
│   │ logs table  │                   │ Inverted Index  │                │
│   │ - timestamp │                   │                 │                │
│   │ - service   │    Sync row_id    │ term → row_ids  │                │
│   │ - message   │ ←───────────────→ │                 │                │
│   │ - row_id    │                   │                 │                │
│   └─────────────┘                   └─────────────────┘                │
│                                                                         │
│   Query flow:                                                           │
│   1. Search Tantivy for "error" → get row_ids [5, 102, 8200]            │
│   2. Query ClickHouse: SELECT * FROM logs WHERE row_id IN (5,102,8200)  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Question | Answer |
|----------|--------|
| **What's stored in Tantivy files?** | Inverted index: term → list of row IDs |
| **What's a granule?** | 8192 rows, the unit ClickHouse can skip |
| **How does it help?** | Returns row IDs that match, so ClickHouse only reads relevant granules |
| **Where are files stored?** | Inside each Part directory as `skp_idx_*.data` and `skp_idx_*.meta` |

---

## Key Insight

**Tantivy doesn't replace ClickHouse storage, it adds an inverted index alongside it to enable fast text search and granule skipping.**

---

## Related Files (MyScaleDB Implementation)

- `src/Storages/MergeTree/MergeTreeIndexTantivy.h` - Skip index definition
- `src/Storages/MergeTree/MergeTreeIndexTantivy.cpp` - Indexing and query logic
- `src/Storages/MergeTree/TantivyIndexStore.h` - Tantivy index file management
- `src/Storages/MergeTree/TantivyIndexStore.cpp` - FFI calls to Rust
- `src/Interpreters/TantivyFilter.h` - Query filter implementation
- `rust/supercrate/libs/tantivy_search/` - Rust Tantivy wrapper (FFI)
