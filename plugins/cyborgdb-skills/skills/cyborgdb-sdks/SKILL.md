---
name: cyborgdb-sdks
description: Help the user set up a CyborgDB SDK (Python, JavaScript/TypeScript, or Go) in their project, matched to the version they're actually on. Detects the installed SDK version, points at the matching docs on docs.cyborg.co, and scaffolds a minimal working integration against a running cyborgdb-service — using the hosted sample dataset so the first query returns non-empty results. Also covers metadata filtering (MongoDB-style operators on encrypted metadata) and full-text/hybrid BM25 search. Use when the user wants to use the SDK in their code, connect an app to cyborgdb-service, or fix code that already uses the SDK. Triggers like "set up cyborgdb sdk", "use the cyborgdb client", "cyborgdb python/js/ts/go example", "connect my app to cyborgdb-service", "filter cyborgdb query by metadata", "cyborgdb full-text / hybrid / bm25 search".
---

# cyborgdb-sdks — set the SDK up in the user's project

Help the user get a working CyborgDB SDK integration in their code. **First detect which SDK version they're on** (or want to install), then follow the matching quickstart on docs.cyborg.co. The inline code below targets **v0.18**; if the user is on an older minor, fetch and follow that version's quickstart — call shapes have shifted.

If the service isn't running yet, do [[cyborgdb-service]] first (or [[cyborgdb-install]] for the fully automated zero-to-first-query path). If the user wants in-process / no service, redirect to [[cyborgdb-core]].

## Step 0 — Detect the SDK version (do this first)

Try the matching command for the user's language. Stop as soon as you have a version.

**Python:**

```bash
pip show cyborgdb 2>/dev/null | awk '/^Version:/ {print $2}'
```

**JavaScript / TypeScript** (from the project root):

```bash
npm ls cyborgdb --depth=0 2>/dev/null
# Or read it directly:
node -p "require('cyborgdb/package.json').version" 2>/dev/null
# Or check the lockfile:
grep -A1 '"cyborgdb"' package-lock.json | head -3
```

**Go** (from the module root):

```bash
go list -m github.com/cyborginc/cyborgdb-go 2>/dev/null
# Or check go.mod:
grep cyborgdb-go go.mod
```

Also useful — check the **running service's** version, which the SDK should be compatible with:

```bash
curl -s http://localhost:8000/v1/health   # has a "version" field
```

If **nothing is installed yet**, resolve the latest from PyPI and proceed with that version — don't pause to ask, and don't hardcode a minor (all three SDKs release on the same minor as `cyborgdb` on PyPI):

```bash
curl -s https://pypi.org/pypi/cyborgdb/json | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])"
```

Only switch versions if the user explicitly asks to pin an older one. SDK and service should be on the **same minor**. If PyPI is unreachable, fall back to **v0.18** so setup can still proceed offline. If PyPI returns a **newer minor than v0.18**, the inline code below may be stale — fetch that version's quickstart page.

**Docs URLs** — current release under `versions/latest/`, pinned older minors under `versions/v<X.Y>.x/`. Append `.md` to any page for raw markdown; `https://docs.cyborg.co/llms.txt` indexes every page.

- **Client setup:** `https://docs.cyborg.co/versions/latest/service/guides/encrypted-indexes/create-client`
- **Create index:** `https://docs.cyborg.co/versions/latest/service/guides/encrypted-indexes/create-index`
- **Upsert / query:** under `https://docs.cyborg.co/versions/latest/service/guides/data-operations/`
- **Sample dataset:** `https://docs.cyborg.co/versions/latest/service/{python-sdk,js-ts-sdk,go-sdk}/load-sample-dataset`
- **Per-language SDK reference:** `https://docs.cyborg.co/versions/latest/service/{python-sdk,js-ts-sdk,go-sdk}/`

For any non-trivial code, **fetch the matching page** for the user's version and follow it.

## Version-specific gotchas

- **v0.18:** `load_sample_dataset` / `loadSampleDataset` / `LoadSampleDataset` ships in all three SDKs — prefer it for demos so queries are never empty. Full-text/hybrid (BM25) query knobs exist on `query`. `create_index` accepts exactly one of `index_key` or `kms_name`.
- **v0.17:** Optional knobs (`metric`, `dimension`, `embedding_model`, `storage_precision`, `kms_name`) are flat keyword args on `create_index`. No sample-dataset helper. `upsert` may trigger background training once `num_vectors > n_lists * RETRAIN_THRESHOLD` (default 10,000) — response carries `training_triggered`.
- **v0.16 and earlier:** the polymorphic `index_config` argument and `IndexIVFFlat` / `IndexIVFPQ` / `IndexIVFSQ` constructors exist. **Do not use the v0.18 calls below verbatim on older versions — fetch their quickstart.**

## Before you scaffold

- Confirm `cyborgdb-service` is reachable from where the code will run (default `http://localhost:8000`). Hit `/v1/health` first; if it fails, fix the service before writing client code.
- The SDK package on PyPI / npm is just **`cyborgdb`** — *not* `cyborgdb-py` / `cyborgdb-js`. Don't confuse it with `cyborgdb-core` (separate embedded library).
- **No API key by default.** The SDK's `api_key` is required **only when the service has auth turned on** (`CYBORGDB_SERVICE_ROOT_KEY` set on the service). With auth off (default), omit it (Go: pass `""`).
- Each index is encrypted with a 32-byte `index_key`. Use the SDK helper: `client.generate_key()` (Python), `client.generateKey()` (JS/TS), `cyborgdb.GenerateKey()` (Go). **Losing the key means losing the data** — persist it in the user's secret manager, don't hardcode.
- For a demo with guaranteed non-empty results, load the hosted sample dataset (`quickstart-75k`: 75k items, 128-dim, euclidean; downloads once, cached locally) instead of inventing toy vectors.

## Pick the SDK

| Language | Install | Import |
|---|---|---|
| Python | `pip install cyborgdb` | `import cyborgdb` |
| JavaScript / TypeScript | `npm install cyborgdb` | `import { Client, loadSampleDataset } from 'cyborgdb'` |
| Go | `go get github.com/cyborginc/cyborgdb-go` | `import "github.com/cyborginc/cyborgdb-go"` |

## Steps (same shape for every language)

1. **Install the SDK** in the user's existing project (don't create a new one unless they ask).
2. **Scaffold one self-contained file** from the matching snippet below. Drop it next to existing code, or in `examples/`.
3. **Run it once** to verify the setup. See *Verifying it works*.

### Python (v0.18)

```python
# quickstart.py — cyborgdb Python SDK v0.18 quickstart
import cyborgdb

client = cyborgdb.Client(base_url="http://localhost:8000")
# api_key only needed if the service sets CYBORGDB_SERVICE_ROOT_KEY.

print(client.get_health())  # {"status": "healthy", ...}

# Hosted sample dataset — downloads on first use, cached in ~/.cache/cyborgdb.
dataset = cyborgdb.load_sample_dataset()  # quickstart-75k: 75k items, 128-dim

# 32-byte key. save=True caches under ~/.cyborgdb/ for local eval only.
index_key = client.generate_key(save=False)

index = client.create_index(
    index_name="my_index",
    index_key=index_key,
    dimension=dataset.dimension,
    metric=dataset.metric,
)

index.upsert(dataset.items)

results = index.query(
    query_vectors=dataset.sample_queries[0],
    top_k=10,
    include=["distance"],
)
for r in results:
    print(r["id"], r["distance"])

# Curated metadata filter, guaranteed to match:
example = dataset.example_filters[0]
print(index.query(query_vectors=dataset.sample_queries[0], filters=example["filter"]))
```

Run: `python quickstart.py`

### JavaScript / TypeScript (v0.18)

```ts
// quickstart.ts — cyborgdb JS/TS SDK v0.18 quickstart
import { Client, loadSampleDataset } from "cyborgdb";

const client = new Client({
  baseUrl: "http://localhost:8000",
  // apiKey only needed if the service sets CYBORGDB_SERVICE_ROOT_KEY.
});

console.log(await client.getHealth());

const dataset = await loadSampleDataset(); // quickstart-75k: 75k items, 128-dim

const indexKey = client.generateKey();

const index = await client.createIndex({
  indexName: "my_index",
  indexKey,
  dimension: dataset.dimension,
  metric: dataset.metric,
});

await index.upsert({ items: dataset.items });

const response = await index.query({
  queryVectors: dataset.sampleQueries[0],
  topK: 10,
  include: ["distance"],
});
console.log(response.results);
```

Run: `tsx quickstart.ts` (or `node quickstart.js` after compiling).

### Go (v0.18)

```go
// main.go — cyborgdb Go SDK v0.18 quickstart
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/cyborginc/cyborgdb-go"
)

func main() {
    ctx := context.Background()

    // Second arg is the service-auth key — "" when auth is off (the default).
    client, err := cyborgdb.NewClient("http://localhost:8000", "")
    if err != nil {
        log.Fatal(err)
    }

    if health, err := client.GetHealth(ctx); err == nil {
        fmt.Printf("service: %+v\n", health)
    }

    // Hosted sample dataset — "" loads the default ("quickstart-75k").
    dataset, err := cyborgdb.LoadSampleDataset("")
    if err != nil {
        log.Fatal(err)
    }

    indexKey, err := cyborgdb.GenerateKey()
    if err != nil {
        log.Fatal(err)
    }

    // CreateIndexParams.Dimension is *int32; the dataset stores it as int.
    dimension := int32(dataset.Dimension)

    index, err := client.CreateIndex(ctx, &cyborgdb.CreateIndexParams{
        IndexName: "my_index",
        IndexKey:  indexKey,
        Dimension: &dimension,
        Metric:    &dataset.Metric,
    })
    if err != nil {
        log.Fatal(err)
    }

    if err := index.Upsert(ctx, dataset.Items); err != nil {
        log.Fatal(err)
    }

    results, err := index.Query(ctx, cyborgdb.QueryParams{
        QueryVector: dataset.SampleQueries[0],
        TopK:        10,
        Include:     []string{"distance"},
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(results)
}
```

Run: `go run main.go`

For exact Go field names and types, fetch `https://docs.cyborg.co/versions/latest/service/go-sdk/load-sample-dataset.md` — the struct shapes above follow it, but verify before shipping.

## Verifying it works

Don't claim the setup is done until the program has actually run and produced:

- A `healthy` response from the health check.
- **Non-empty** query results (ids + distances) against the sample dataset.

If you can't run the user's code (sandboxed env, remote machine), say so explicitly rather than asserting success.

## Metadata filtering

Metadata is end-to-end encrypted like the vectors. Tag items at upsert, filter at query.

**Python:**

```python
index.upsert([
    {"id": "item_1", "vector": [0.1, 0.1, 0.1, 0.1], "metadata": {"category": "dog", "views": 42}},
    {"id": "item_2", "vector": [0.2, 0.2, 0.2, 0.2], "metadata": {"category": "cat", "views": 7}},
])

results = index.query(
    query_vectors=[0.5, 0.9, 0.2, 0.7],
    top_k=10,
    filters={"category": {"$in": ["dog", "cat"]}, "views": {"$gte": 10}},
)
```

**JavaScript / TypeScript** (same filter objects, camelCase params; `query()` returns `{results: [...]}`):

```ts
await index.upsert({ items: [
  { id: "item_1", vector: [0.1, 0.1, 0.1, 0.1], metadata: { category: "dog", views: 42 } },
  { id: "item_2", vector: [0.2, 0.2, 0.2, 0.2], metadata: { category: "cat", views: 7 } },
]});

const response = await index.query({
  queryVectors: [0.5, 0.9, 0.2, 0.7],
  topK: 10,
  filters: { category: { $in: ["dog", "cat"] }, views: { $gte: 10 } },
});
console.log(response.results);
```

**Go** (filters are `map[string]interface{}`; single-vector results land in `Results.ArrayOfQueryResultItem`):

```go
items := cyborgdb.VectorItems{
    {Id: "item_1", Vector: []float32{0.1, 0.1, 0.1, 0.1},
        Metadata: map[string]interface{}{"category": "dog", "views": 42}},
    {Id: "item_2", Vector: []float32{0.2, 0.2, 0.2, 0.2},
        Metadata: map[string]interface{}{"category": "cat", "views": 7}},
}
if err := index.Upsert(ctx, items); err != nil { log.Fatal(err) }

resp, err := index.Query(ctx, cyborgdb.QueryParams{
    QueryVector: []float32{0.5, 0.9, 0.2, 0.7},
    TopK:        10,
    Filters: map[string]interface{}{
        "category": map[string]interface{}{"$in": []string{"dog", "cat"}},
        "views":    map[string]interface{}{"$gte": 10},
    },
})
if err != nil { log.Fatal(err) }
for _, r := range *resp.Results.ArrayOfQueryResultItem {
    fmt.Println(r.Id)
}
```

Things to get right:

- **Types:** boolean, string, list of strings, numbers (stored as fp64).
- **Operators** (MongoDB-style subset): `$and`, `$or`, `$not`, `$exists`, `$eq`, `$ne`, `$in`, `$nin`, `$gt`, `$gte`, `$lt`, `$lte`.
- **Case-sensitive:** field names and string values must match exactly between upsert and filter.
- **Indexing:** every metadata field is encrypted-indexed by default, so filters are fast. Fields can be opted out via `metadata_schema` at index creation; filters on non-indexed fields still work but evaluate after decryption (slower) — same for `$regex`/`$contains` on fields not pattern-indexed.
- The sample dataset's `example_filters` are curated, guaranteed-to-match examples — use them for demos instead of inventing filters.

Full reference: `https://docs.cyborg.co/versions/latest/service/guides/data-operations/metadata-filtering`

## Full-text & hybrid search (BM25, v0.18)

BM25 runs over **designated metadata fields** (not `contents`), and the text index is encrypted like everything else. Enablement is by designation at creation — it cannot be added to an existing index:

```python
index = client.create_index(
    index_name="articles",
    index_key=index_key,
    dimension=4,
    text_fields=["title", "body"],   # marks each field full_text=True
)
# JS/TS: createIndex({..., textFields: ["title", "body"]})
# Go:    CreateIndexParams{..., TextFields: []string{"title", "body"}}
index.upsert([{
    "id": "a1",
    "vector": [0.1, 0.2, 0.3, 0.4],
    "metadata": {"title": "Encrypted vector search", "body": "Searching encrypted embeddings.", "author": "ann"},
}])
```

Three query paths, picked by which arguments you set:

```python
# 1. Pure text (no vector) — query_metadata, ranks by BM25 score:
index.query_metadata(text="encrypted search", top_k=10)
# [{"id": "a1", "score": 2.63}, ...]

# 2. Hybrid (BM25 + vector) — query with text; rank fusion via weighted RRF:
index.query(query_vectors=qv, text="encrypted search", top_k=10, alpha=0.5)
# alpha: 0 = pure BM25, 1 = pure vector (default 0.5). Results carry `score`, never `distance`.

# 3. Three-way — add filters; the filter resolves first, both legs rank its survivors:
index.query(query_vectors=qv, text="encrypted search", filters={"author": "ann"}, top_k=10)
```

**JavaScript / TypeScript** — same three paths; note the return-shape split (`queryMetadata` returns a bare array, `query` returns `{results: [...]}`):

```ts
const pure = await index.queryMetadata({ text: "encrypted search", topK: 10 });
// pure is the array itself: [{id: "a1", score: 2.63}, ...]

const hybrid = await index.query({ queryVectors: qv, text: "encrypted search", topK: 10, alpha: 0.5 });
const threeWay = await index.query({ queryVectors: qv, text: "encrypted search", filters: { author: "ann" }, topK: 10 });
console.log(hybrid.results, threeWay.results);
```

**Go** — optional params use pointer helpers; read single-vector results via the union accessor:

```go
pure, err := index.QueryMetadata(ctx, cyborgdb.QueryMetadataParams{
    Text: cyborgdb.String("encrypted search"),
    TopK: 10,
})
// pure.Results rows: row.Id, row.GetScore()

hybrid, err := index.Query(ctx, cyborgdb.QueryParams{
    QueryVector: qv,
    Text:        cyborgdb.String("encrypted search"),
    TopK:        10,
    Alpha:       cyborgdb.Float64(0.5),
    // Three-way: add Filters: map[string]interface{}{"author": "ann"}
})
for _, r := range *hybrid.Results.ArrayOfQueryResultItem {
    fmt.Println(r.Id, r.GetScore())   // hybrid rows carry score; r.HasDistance() is false
}
```

Text-leg knobs: `text_fields` (search a subset of designated fields), `text_field_weights` (parallel weights), `require_all_terms` (AND instead of the default OR). Creation-time scoring constants: `bm25_k1` (default 1.2), `bm25_b` (default 0.75).

Gotchas that produce errors or silent misses — check these before writing code:

- `full_text: true` implies `filterable: false` and is incompatible with `pattern: true` — a field is analyzed for BM25 **or** exact-match indexed, never both. Need both? Store the value twice under two names.
- A hybrid `query()` still **requires a query vector even at `alpha=0`** — for text-only search use `query_metadata`.
- With `text` set, only filters the metadata index resolves exactly are accepted — `$regex` on a non-`pattern` field or predicates on non-`filterable` fields are rejected (HTTP 400), and `order_by` can't combine with `text`.
- A `full_text` field's value must be a **single string** — numbers, booleans, and lists of strings are skipped for that item (upsert still succeeds). English-only analysis for now; no phrase queries or NOT/exclusion.
- Multi-tenant: use index-per-tenant — BM25 corpus statistics are global per index, never scoped to a filter.

Full guide (tuning table for `alpha`/`rrf_k`/`window_mult`, error reference): `https://docs.cyborg.co/versions/latest/service/guides/data-operations/bm25-search`

## Pointers for follow-up questions

- **Batched queries.** Pass a list of vectors instead of one; results come back nested (one list per query).
- **Automatic embeddings.** Create the index with `embedding_model="all-MiniLM-L6-v2"` (any HuggingFace model), upsert `contents` only, query with `query_contents` instead of `query_vectors`. Requires the Docker service or `pip install cyborgdb-service[embeddings]`.
- **REST API directly** (no SDK): `POST /v1/indexes/create`, `POST /v1/vectors/upsert`, `POST /v1/vectors/query`, `POST /v1/vectors/query_metadata`, with `X-API-Key` header when auth is on. Reference: `https://docs.cyborg.co/versions/latest/service/rest-api/`
