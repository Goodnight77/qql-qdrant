# Managing Collections

---

## SHOW COLLECTIONS — list collections

Lists all collections in the connected Qdrant instance.

```sql
SHOW COLLECTIONS
```

**Output:**
```
✓ 3 collection(s) found
┌──────────────────┐
│ Collection       │
├──────────────────┤
│ articles         │
│ notes            │
│ products         │
└──────────────────┘
```

---

## CREATE COLLECTION — create a collection

Explicitly creates a new empty collection. Collections are also created automatically on the first INSERT, so this command is optional — use it when you want to pre-create a collection before inserting data.

**Syntax:**
```
CREATE COLLECTION <collection_name>
CREATE COLLECTION <collection_name> HYBRID
CREATE COLLECTION <collection_name> USING MODEL '<model_name>'
CREATE COLLECTION <collection_name> USING HYBRID
CREATE COLLECTION <collection_name> USING HYBRID DENSE MODEL '<model>'
```

Any of the above forms can be followed by an optional `QUANTIZE` clause — see [Quantization](#quantization--quantize-clause) below.

**Examples:**

Dense-only collection (standard, uses default model dimensions):
```sql
CREATE COLLECTION research_papers
```

Dense-only collection pinned to a specific model (768-dimensional):
```sql
CREATE COLLECTION research_papers USING MODEL 'BAAI/bge-base-en-v1.5'
```

Hybrid collection (dense + sparse BM25, default models):
```sql
CREATE COLLECTION research_papers HYBRID
```

Hybrid collection with a custom dense model:
```sql
CREATE COLLECTION research_papers USING HYBRID DENSE MODEL 'BAAI/bge-base-en-v1.5'
```

When `USING MODEL` is omitted, the collection uses the **default embedding model's dimensions** (384 for `all-MiniLM-L6-v2`). If the collection already exists, the command succeeds with a message and does nothing.

---

## Quantization — QUANTIZE clause

Quantization reduces the memory footprint of vector collections and speeds up search at the cost of a small, controllable accuracy loss. QQL supports all three Qdrant quantization strategies via an optional `QUANTIZE` clause appended to `CREATE COLLECTION`.

**Three strategies:**

| Type | Compression | Accuracy Loss | Best For |
|---|---|---|---|
| `SCALAR` | 4× (float32 → int8) | < 1% | Most collections — best balance |
| `BINARY` | 32× (float32 → 1-bit) | Higher | High-dimensional vectors (768+), speed priority |
| `PRODUCT` | 4× (configurable) | Variable | Memory-constrained deployments |

**Full syntax:**
```
CREATE COLLECTION <name> ... QUANTIZE SCALAR [QUANTILE <0.0–1.0>] [ALWAYS RAM]
CREATE COLLECTION <name> ... QUANTIZE BINARY  [ALWAYS RAM]
CREATE COLLECTION <name> ... QUANTIZE PRODUCT [ALWAYS RAM]
```

- **`QUANTILE <float>`** — (scalar only) calibration quantile for the INT8 conversion; defaults to Qdrant's built-in default (0.99) when omitted.
- **`ALWAYS RAM`** — keep the **original** (unquantized) vectors in RAM for rescoring, sacrificing memory savings but preserving accuracy during re-ranking. Supported by all three types.
- **`QUANTIZE`** always appears **after** all other clauses (`HYBRID`, `USING MODEL`, etc.).
- For `PRODUCT`, the compression ratio is fixed at **4×** in this version.
- When used with `HYBRID` collections, quantization applies only to the **dense** vector.

**Examples:**

Scalar quantization (recommended default):
```sql
CREATE COLLECTION research_papers QUANTIZE SCALAR
```

Scalar with explicit calibration and original vectors kept in RAM:
```sql
CREATE COLLECTION research_papers QUANTIZE SCALAR QUANTILE 0.95 ALWAYS RAM
```

Binary quantization for large high-dimensional embeddings:
```sql
CREATE COLLECTION research_papers QUANTIZE BINARY
```

Product quantization for maximum memory savings:
```sql
CREATE COLLECTION research_papers QUANTIZE PRODUCT ALWAYS RAM
```

Combined with hybrid collection:
```sql
CREATE COLLECTION research_papers HYBRID QUANTIZE SCALAR
```

Combined with a pinned model:
```sql
CREATE COLLECTION research_papers USING MODEL 'BAAI/bge-base-en-v1.5' QUANTIZE SCALAR QUANTILE 0.99
```

**Valid combinations:**

| Base form | + QUANTIZE SCALAR | + QUANTIZE BINARY | + QUANTIZE PRODUCT |
|---|---|---|---|
| `CREATE COLLECTION name` | ✓ | ✓ | ✓ |
| `... HYBRID` | ✓ | ✓ | ✓ |
| `... USING MODEL 'x'` | ✓ | ✓ | ✓ |
| `... USING HYBRID` | ✓ | ✓ | ✓ |
| `... USING HYBRID DENSE MODEL 'x'` | ✓ | ✓ | ✓ |

> INSERT and SEARCH on quantized collections work exactly the same as on non-quantized ones — no changes to INSERT or SEARCH syntax are needed.

---

## CREATE INDEX — create a payload index

Creates a payload index on a collection field. Payload indexes speed up `WHERE` clause filtering by allowing Qdrant to efficiently match on indexed fields.

**Syntax:**
```
CREATE INDEX ON COLLECTION <collection_name> FOR <field_name> TYPE <schema_type>
```

**Supported schema types:**

| Type | Description |
|---|---|
| `keyword` | Exact string match (e.g. status, category) |
| `integer` | Whole numbers |
| `float` | Decimal numbers |
| `bool` | Boolean values |
| `text` | Full-text search (enables `MATCH` operators) |
| `geo` | Geospatial coordinates |
| `datetime` | Date/time values |

**Examples:**

```sql
CREATE INDEX ON COLLECTION articles FOR category TYPE keyword
CREATE INDEX ON COLLECTION articles FOR year TYPE integer
CREATE INDEX ON COLLECTION articles FOR title TYPE text
CREATE INDEX ON COLLECTION articles FOR meta.author TYPE keyword
```

**Rules:**
- The collection must already exist. Raises an error otherwise.
- Indexes are idempotent — creating the same index twice succeeds silently.

---

## DROP COLLECTION — delete a collection

Permanently deletes a collection and **all points inside it**. This operation is irreversible.

```sql
DROP COLLECTION old_experiments
```

Raises an error if the collection does not exist.

---

## DELETE — remove points

Deletes one or more points from a collection by specific ID or by a `WHERE` filter.

**Syntax:**
```
DELETE FROM <collection_name> WHERE id = '<point_id>'
DELETE FROM <collection_name> WHERE id = <integer_id>
DELETE FROM <collection_name> WHERE <filter>
```

**Examples:**

```sql
-- Delete by UUID
DELETE FROM articles WHERE id = '3f2e1a4b-8c91-4d0e-b123-abc123def456'

-- Delete by integer ID
DELETE FROM articles WHERE id = 42

-- Delete all points matching a filter
DELETE FROM articles WHERE category = 'archived'

-- Delete with a compound filter
DELETE FROM articles WHERE year < 2020 AND status = 'draft'
```

**Notes:**
- If no points match the filter or ID, the operation succeeds silently with a count of 0.
- The collection itself must exist; deleting from a non-existent collection raises an error.
