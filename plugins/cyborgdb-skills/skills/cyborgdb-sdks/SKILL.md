---
name: cyborgdb-sdks
description: Help the user set up a CyborgDB SDK (Python, JavaScript/TypeScript, or Go) in their project, matched to the version they're actually on. Detects the installed SDK version, points at the matching docs on docs.cyborg.co, and scaffolds a minimal working integration against a running cyborgdb-service — using the hosted sample dataset so the first query returns non-empty results. Use when the user wants to use the SDK in their code, connect an app to cyborgdb-service, or fix code that already uses the SDK. Triggers like "set up cyborgdb sdk", "use the cyborgdb client", "cyborgdb python/js/ts/go example", "connect my app to cyborgdb-service".
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

    index, err := client.CreateIndex(ctx, &cyborgdb.CreateIndexParams{
        IndexName: "my_index",
        IndexKey:  indexKey,
        Dimension: &dataset.Dimension,
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

## Pointers for follow-up questions

- **Metadata + filtering.** Add `metadata: {...}` on upsert, pass `filters` to `query` (operators: `$gt`, `$lt`, `$gte`, `$lte`, `$eq`, `$ne`). The sample dataset's `example_filters` are curated, guaranteed-to-match examples.
- **Batched queries.** Pass a list of vectors instead of one; results come back nested (one list per query).
- **Automatic embeddings.** Create the index with `embedding_model="all-MiniLM-L6-v2"` (any HuggingFace model), upsert `contents` only, query with `query_contents` instead of `query_vectors`. Requires the Docker service or `pip install cyborgdb-service[embeddings]`.
- **Full-text / hybrid search (v0.18).** Index with `text_fields`, query with `text=...` for BM25 + vector fusion: `https://docs.cyborg.co/versions/latest/service/guides/data-operations/bm25-search`
- **REST API directly** (no SDK): `POST /v1/indexes/create`, `POST /v1/vectors/upsert`, `POST /v1/vectors/query`, with `X-API-Key` header when auth is on. Reference: `https://docs.cyborg.co/versions/latest/service/rest-api/`
