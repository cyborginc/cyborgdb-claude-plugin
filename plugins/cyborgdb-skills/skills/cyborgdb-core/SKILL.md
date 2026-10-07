---
name: cyborgdb-core
description: Help the user set up cyborgdb-core (the embedded, in-process CyborgDB engine) in their own project, matched to the version they're actually on. Detects the installed cyborgdb-core version, points at the matching docs on docs.cyborg.co, and scaffolds a minimal working integration following that version's quickstart — no server, no API key, local disk storage by default. Use when the user wants to use cyborgdb-core, integrate it into their Python code, evaluate it locally, or fix code that already uses it. Triggers like "set up cyborgdb-core", "use cyborgdb-core", "cyborgdb-core quickstart", "embedded cyborgdb", "add cyborgdb-core to my app".
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

- **v0.17 / v0.18:** `index_key` is supplied on every data op. Storage backends: `disk` (default), `memory`, `s3`. Free-tier cap is 1,000,000 items per index without an API key. Query `contents` live in separately encrypted item records — chain `get()` after `query()` to read them.
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
2. **Scaffold** a self-contained file (e.g. `quickstart.py`) using the snippet below. Use `StorageConfig.disk(...)` for default persistent storage; swap to `.memory()` for tests or `.s3("bucket")` for cloud.
3. **Run it once** to verify the setup works against the user's installed version. See *Verifying it works* below.

```python
# quickstart.py — cyborgdb-core v0.18 embedded quickstart
import cyborgdb_core as cyborgdb
import secrets

# Local disk backing store — no server required.
client = cyborgdb.Client(storage_config=cyborgdb.StorageConfig.disk("/tmp/cyborgdb"))

# 32-byte key. LOSING THIS LOSES THE DATA — persist it in a secret manager.
index_key = secrets.token_bytes(32)

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
    include=["distance", "metadata"],
    index_key=index_key,
)

# Contents live in separately encrypted item records — retrieve them with get().
top_items = index.get([r["id"] for r in results], index_key=index_key)
by_id = {item["id"]: item for item in top_items}
for r in results:
    contents = by_id[r["id"]]["contents"].decode()
    print(f"ID: {r['id']}  Distance: {r['distance']:.4f}  Contents: {contents}")
```

## Verifying it works

Don't claim the setup is done until the program has actually run and printed:

- 2 lines of `ID: item_N  Distance: ...  Contents: ...`, with `item_1` closest to itself.

If you can't run it (sandboxed env, remote machine), say so explicitly — don't assert success.

## Pointers for follow-up questions

- Storage backends: `StorageConfig.memory()`, `.disk(path)`, `.s3(bucket)`.
- Encryption model: an encrypted index does not auto-hide vectors from the key holder — the user must pass `index_key` on `get`/`query` to retrieve decrypted data, and anyone without the key gets nothing.
- API key (lifts the 1M-item cap): https://cyborgdb.co — pass as first argument: `cyborgdb.Client(api_key, storage_config=...)`.
- LangChain integration: `cyborgdb_core.integrations.langchain.CyborgVectorStore` (`pip install cyborgdb-core[langchain]`).
- Reference: `https://docs.cyborg.co/versions/latest/embedded/`
