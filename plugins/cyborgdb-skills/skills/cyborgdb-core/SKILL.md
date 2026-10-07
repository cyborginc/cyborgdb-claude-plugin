---
name: cyborgdb-core
description: Help the user set up cyborgdb-core (the embedded, in-process CyborgDB engine) in their own project, matched to the version they're actually on. Detects the installed cyborgdb-core version, points at the matching docs on docs.cyborg.co, and scaffolds a minimal working integration following that version's quickstart — no server, no API key, local disk storage by default. Also covers embedded metadata filtering (MongoDB-style operators on encrypted metadata) and full-text/hybrid BM25 search. Use when the user wants to use cyborgdb-core, integrate it into their Python code, evaluate it locally, or fix code that already uses it. Triggers like "set up cyborgdb-core", "use cyborgdb-core", "cyborgdb-core quickstart", "embedded cyborgdb", "add cyborgdb-core to my app", "filter/bm25/hybrid search with cyborgdb-core".
---

# cyborgdb-core — set it up in the user's project

Help the user get a working `cyborgdb-core` integration in their code. **First detect which version of cyborgdb-core they're on** (or want to install), then follow the matching quickstart on docs.cyborg.co. The inline code below targets **v0.18**; if the user is on an older version, fetch and follow that version's quickstart instead — APIs have shifted between minors.

`cyborgdb-core` is **Python only**. For other languages, the user wants [[cyborgdb-sdks]] against a running [[cyborgdb-service]]. For a fully automated zero-to-first-query install with the hosted sample dataset, use [[cyborgdb-install]] (the sample-dataset helper ships in the service SDK, not in cyborgdb-core).

## Step 0 — Detect the version (do this first)

Try these in order until one resolves a version. Stop as soon as you have one.

```bash
pip show cyborgdb-core 2>/dev/null | awk '/^Version:/ {print $2}'
# Or, if already imported in their code:
python -c "import cyborgdb_core; print(cyborgdb_core.__version__)"
```

If **nothing is installed yet**, resolve the latest from PyPI and proceed with that version — don't pause to ask, and don't hardcode a minor:

```bash
curl -s https://pypi.org/pypi/cyborgdb-core/json | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])"
```

Only switch versions if the user explicitly asks to pin an older one. If PyPI is unreachable, fall back to **v0.18** so setup can still proceed offline. If PyPI returns a **newer minor than v0.18**, the inline code below may be stale — fetch that version's quickstart page and follow it instead.

**Docs URLs** — current release under `versions/latest/`, pinned older minors under `versions/v<X.Y>.x/`. Append `.md` to any page for raw markdown; `https://docs.cyborg.co/llms.txt` indexes every page.

- **Quickstart:** `https://docs.cyborg.co/versions/latest/intro/quickstart` (embedded flow, maintained as the primary onboarding page)
- **Embedded guides:** `https://docs.cyborg.co/versions/latest/embedded/guides/intro/quickstart`

For any non-trivial code change, **fetch the matching quickstart page** and follow it — don't rely on memory for older versions.

## Version-specific gotchas

- **v0.17 / v0.18:** `index_key` is supplied on every data op. Storage backends: `disk` (default) and `s3` — there is **no `StorageConfig.memory()`**; for memory-tier reads use the cache flags on disk, e.g. `StorageConfig.disk(path, cache_vectors=True, cache_metadata=True, cache_ids=True, cache_metadata_index=True)`. Free-tier cap is 1,000,000 items per index without an API key. Query `contents` live in separately encrypted item records — chain `get()` after `query()` to read them.
- **v0.16 and earlier:** polymorphic index constructors (`IndexIVFFlat`, `IndexIVFPQ`, `IndexIVFSQ`) exist; in v0.17 they were collapsed into flat keyword arguments on `create_index`. The old storage backend names (`standalone`, `postgres`, `redis`, `CYBORGDB_CONNECTION_STRING`) also apply pre-v0.17. **If the user is on v0.16 or older, do not use the v0.18 code below verbatim — fetch their version's quickstart.**

## Before you scaffold

- The `index_key` is a 32-byte secret that encrypts the index. **Losing it means losing the data — there is no recovery.** Tell the user to store it in their secret manager; never hardcode a real key.
- No API key needed: without one the client runs in free-tier mode (1M items/index cap). For a real key, point them to https://cyborgdb.co.
- The hosted sample dataset (`load_sample_dataset`) lives in the **service SDK** (`pip install cyborgdb`), not here — if the user wants a pre-populated demo, that's the [[cyborgdb-install]] / [[cyborgdb-sdks]] path.

## Steps (Python, v0.18)

1. **Install** into the user's existing environment:
   ```bash
   pip install cyborgdb-core
   # Optional extras:
   #   pip install cyborgdb-core[embeddings]   # automatic embedding generation
   #   pip install cyborgdb-core[langchain]    # LangChain integration
   #   pip install cyborgdb-core[all]
   ```
2. **Scaffold** a self-contained file (e.g. `quickstart.py`) using the snippet below. Use `StorageConfig.disk(...)` for default persistent storage; swap to `.s3("bucket")` for cloud. For tests, point `.disk(...)` at a temp dir — there is no in-memory backend.
3. **Run it once** to verify the setup works against the user's installed version. See *Verifying it works* below.

**Re-running either script fails** with `ValueError: Index name my_index already exists` (or `articles` for the snippet below — both share `/tmp/cyborgdb`) — the index persists on disk, and since the demo key was ephemeral, the old index is unrecoverable (`load_index` with a different key fails to unwrap the DEK). For a fresh start, delete the storage dir (`rm -rf /tmp/cyborgdb`). For real usage, save the key and reopen with `client.load_index(index_name, index_key)` instead of `create_index`.

```python
# quickstart.py — cyborgdb-core v0.18 embedded quickstart
import cyborgdb_core as cyborgdb
import secrets

# Local disk backing store — no server required.
client = cyborgdb.Client(storage_config=cyborgdb.StorageConfig.disk("/tmp/cyborgdb"))

# 32-byte key. LOSING THIS LOSES THE DATA — persist it in a secret manager.
index_key = secrets.token_bytes(32)

# dimension is optional here — it's inferred from the first upsert.
index = client.create_index(index_name="my_index", index_key=index_key)

# index_key is required on every data operation.
items = [
    {"id": "item_1", "vector": [0.1, 0.2, 0.3, 0.4], "contents": "Hello!"},
    {"id": "item_2", "vector": [0.5, 0.6, 0.7, 0.8], "contents": "Bonjour!"},
    {"id": "item_3", "vector": [0.9, 0.10, 0.11, 0.12], "contents": "Hola!"},
]
index.upsert(items, index_key=index_key)

results = index.query(
    query_vectors=[0.1, 0.2, 0.3, 0.4],
    top_k=2,
    include=["distance"],  # add "metadata" when your items carry metadata
    index_key=index_key,
)

# Contents live in separately encrypted item records — retrieve them with get().
top_items = index.get([r["id"] for r in results], index_key=index_key)
by_id = {item["id"]: item for item in top_items}
for r in results:
    contents = by_id[r["id"]]["contents"].decode()
    print(f"ID: {r['id']}  Distance: {r['distance']:.4f}  Contents: {contents}")
```

## Metadata filtering & full-text search (embedded, v0.18)

Both work embedded with the same shapes as the service SDK, plus `index_key` on every call. BM25 must be enabled at creation by designating `full_text` fields — it cannot be added to an existing index.

```python
import secrets
import numpy as np
import cyborgdb_core as cyborgdb

client = cyborgdb.Client(storage_config=cyborgdb.StorageConfig.disk("/tmp/cyborgdb"))
index_key = secrets.token_bytes(32)

index = client.create_index(
    "articles",
    index_key,
    dimension=128,
    metadata_schema={
        "title":  {"full_text": True},   # analyzed for BM25
        "body":   {"full_text": True},   # (shorthand: text_fields=["title", "body"])
        "author": {"filterable": True},  # ordinary exact-match filterable field
    },
)

rng = np.random.default_rng(0)
index.upsert([
    {"id": "a1", "vector": rng.random(128, dtype=np.float32),
     "metadata": {"title": "Encrypted vector search",
                  "body": "Searching over encrypted embeddings.", "author": "ann"}},
    {"id": "a2", "vector": rng.random(128, dtype=np.float32),
     "metadata": {"title": "Index tuning",
                  "body": "Choosing n_lists for recall.", "author": "bob"}},
], index_key=index_key)

# Pure BM25 (no vector) — ranks by text score:
index.query_metadata(text="encrypted search", top_k=10, index_key=index_key)
# [{"id": "a1", "score": 2.67}]

# query_metadata also works with filters alone (no text) — fetch by metadata, no vector needed:
index.query_metadata(filters={"author": "bob"}, top_k=10, index_key=index_key)
# [{"id": "a2"}]

# Hybrid (BM25 + vector, fused by weighted RRF; results carry score, never distance):
qv = rng.random(128, dtype=np.float32)
index.query(query_vectors=qv, text="encrypted search", top_k=10, alpha=0.5, index_key=index_key)
# alpha: 0 = pure BM25, 1 = pure vector (default 0.5)

# Metadata filter (MongoDB-style operators), with or without text.
# Vector/filter query() returns bare IDs unless you pass include=[...] (hybrid always carries score):
index.query(query_vectors=qv, filters={"author": {"$in": ["ann", "bob"]}},
            top_k=10, include=["distance", "metadata"], index_key=index_key)
# [{"id": "a1", "distance": ..., "metadata": {...}}, {"id": "a2", ...}]
index.query(query_vectors=qv, text="encrypted search", filters={"author": "ann"},
            top_k=10, index_key=index_key)   # filter resolves first; both legs rank its survivors
```

A `metadata_schema` is only **required for BM25** (`full_text` fields must be declared at creation). Plain metadata filtering works without any schema — upsert `metadata` on a schemaless index and `filters=` just works.

Gotchas (same as the service SDK — see [[cyborgdb-sdks]] for the full list): `full_text` fields can't also be `filterable`/`pattern` (store the value twice under two names if needed); hybrid `query()` requires `query_vectors` or `query_contents` even at `alpha=0` — the error names `query_contents`, the auto-embedding input from the `[embeddings]` extra (use `query_metadata` for text-only); `full_text` values must be single strings; field names and string values are case-sensitive in filters.

Full guides: `https://docs.cyborg.co/versions/latest/embedded/guides/data-operations/{metadata-filtering,bm25-search}`

## Verifying it works

Don't claim the setup is done until the program has actually run and printed:

- 2 lines of `ID: item_N  Distance: ...  Contents: ...`, with `item_1` closest to itself.

If you can't run it (sandboxed env, remote machine), say so explicitly — don't assert success.

## Pointers for follow-up questions

- Storage backends: `StorageConfig.disk(path)` and `.s3(bucket)` only. No `.memory()` — for memory-tier warm reads set the four `cache_*` flags on `.disk(...)` to `True`.
- Existing indexes: `client.list_indexes()` to enumerate, `client.load_index(name, index_key)` to reopen (needs the original key), `index.delete_index()` to remove one.
- Filter operators: `$eq`, `$ne`, `$in`, `$nin`, `$gt`, `$gte`, `$lt`, `$lte`, `$and`, `$or`, `$not`, `$exists` (MongoDB-style subset; see the section above for upsert/query shapes).
- Encryption model: an encrypted index does not auto-hide vectors from the key holder — the user must pass `index_key` on `get`/`query` to retrieve decrypted data, and anyone without the key gets nothing.
- API key (lifts the 1M-item cap): https://cyborgdb.co — pass as first argument: `cyborgdb.Client(api_key, storage_config=...)`.
- LangChain integration: `cyborgdb_core.integrations.langchain.CyborgVectorStore` (`pip install cyborgdb-core[langchain]`).
- Reference: `https://docs.cyborg.co/versions/latest/embedded/`
