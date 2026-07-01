---
name: cyborgdb-core
description: Help the user set up cyborgdb-core (the embedded, in-process CyborgDB engine) in their own project, matched to the version they're actually on. Detects the installed cyborgdb-core version, points at the matching v0.X.x docs on docs.cyborg.co, and scaffolds a minimal working integration following that version's quickstart. Use when the user wants to use cyborgdb-core, integrate it into their Python or C++ code, evaluate it locally, or fix code that already uses it. Triggers like "set up cyborgdb-core", "use cyborgdb-core", "cyborgdb-core quickstart", "embedded cyborgdb", "add cyborgdb-core to my app".
---

# cyborgdb-core — set it up in the user's project

Help the user get a working `cyborgdb-core` integration in their code. **First detect which version of cyborgdb-core they're on** (or want to install), then follow the matching quickstart from `docs.cyborg.co/versions/v<MAJOR.MINOR>.x/...`. The inline code below targets **v0.17**; if the user is on an older version, fetch and follow that version's quickstart instead — APIs have shifted between minors.

`cyborgdb-core` is **Python and C++ only**. For other languages, the user wants [[cyborgdb-sdks]] against a running [[cyborgdb-service]].

## Step 0 — Detect the version (do this first)

Try these in order until one resolves a version. Stop as soon as you have one.

**Python:**

```bash
pip show cyborgdb-core 2>/dev/null | awk '/^Version:/ {print $2}'
# Or, if already imported in their code:
python -c "import cyborgdb_core; print(cyborgdb_core.__version__)"
```

**C++ (Conan):**

```bash
grep -E 'cyborgdb[-_]core' conanfile.txt conanfile.py 2>/dev/null
# Or check the lockfile:
grep -A1 'cyborgdb_core' conan.lock 2>/dev/null
```

If **nothing is installed yet**, resolve the latest from PyPI and proceed with that version — don't pause to ask, and don't hardcode a minor:

```bash
curl -s https://pypi.org/pypi/cyborgdb-core/json | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])"
```

Only switch versions if the user explicitly asks to pin an older one. If PyPI is unreachable, fall back to **v0.17** so setup can still proceed offline. If PyPI returns a **newer minor than v0.17**, the inline code below may be stale — fetch that version's quickstart page and follow it instead.

Once you know the version `vX.Y.Z`:

- **Docs base:** `https://docs.cyborg.co/versions/v<X.Y>.x/`
- **Quickstart:** `https://docs.cyborg.co/versions/v<X.Y>.x/embedded/guides/intro/quickstart`

For any non-trivial code change, **fetch the matching quickstart page** and follow it — don't rely on memory for older versions.

## Version-specific gotchas

- **v0.17:** `index_key` is supplied on every data op. Storage backends: `disk` (default), `memory`, `s3`. Free-tier cap is 1,000,000 items per index without an API key.
- **v0.16 and earlier:** polymorphic index constructors (`IndexIVFFlat`, `IndexIVFPQ`, `IndexIVFSQ`) exist; in v0.17 they were collapsed into flat keyword arguments on `create_index`. The old storage backend names (`standalone`, `postgres`, `redis`, `CYBORGDB_CONNECTION_STRING`) also apply pre-v0.17. **If the user is on v0.16 or older, do not use the v0.17 code below verbatim — fetch their version's quickstart.**

## Before you scaffold

- The `index_key` is a 32-byte secret that encrypts the index. **Losing it means losing the data — there is no recovery.** Tell the user to store it in their secret manager; never hardcode a real key.
- Without an API key, the client runs in free-tier mode (1M items/index cap on v0.17). For a real key, point them to https://cyborgdb.co.
- C++ install requires a Conan token from Cyborg — surface this early so the user can request one before you proceed.
- Default to **Python** unless the user is clearly in C++.

## Steps (Python, v0.17 — default)

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
# quickstart.py — cyborgdb-core v0.17 embedded quickstart
import cyborgdb_core as cyborgdb
import secrets

client = cyborgdb.Client(
    storage_config=cyborgdb.StorageConfig.disk("/tmp/cyborgdb"),
)

# 32-byte key. LOSING THIS LOSES THE DATA — persist it in a secret manager.
index_key = secrets.token_bytes(32)

index = client.create_index(
    index_name="my_index",
    index_key=index_key,
)

index.upsert(
    [
        {"id": "item_1", "vector": [0.1, 0.2, 0.3, 0.4], "contents": "Hello!"},
        {"id": "item_2", "vector": [0.5, 0.6, 0.7, 0.8], "contents": "Bonjour!"},
        {"id": "item_3", "vector": [0.9, 0.10, 0.11, 0.12], "contents": "Hola!"},
    ],
    index_key=index_key,
)

results = index.query(
    query_vectors=[0.1, 0.2, 0.3, 0.4],
    include=["distance"],
    index_key=index_key,
)
for r in results:
    print(f"ID: {r['id']}, Distance: {r['distance']}")

got = index.get(["item_1", "item_2", "item_3"], index_key=index_key)
for item in got:
    print(f"ID: {item['id']}, Vector: {item['vector']}, Contents: {item['contents']}")
```

## Steps (C++, v0.17)

1. **Install** via Conan (replace `<token>` with the Cyborg-provided token):
   ```bash
   conan remote add cyborgdb https://dl.cloudsmith.io/<token>/cyborg/cyborgdb/conan
   conan remote login cyborgdb -p <token> cyborg
   conan install cyborgdb_core -r cyborgdb
   ```
2. **Scaffold** `quickstart.cpp` and wire it into the user's CMake / Conan project.
3. **Build and run** to verify.

```cpp
// quickstart.cpp — cyborgdb-core v0.17 embedded quickstart
#include "cyborgdb_core/client.hpp"
#include "cyborgdb_core/encrypted_index.hpp"
#include <array>
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>
#include <openssl/rand.h>

int main() {
    cyborg::Client client(
        /*api_key=*/"",
        cyborg::StorageConfig::Disk("/tmp/cyborgdb"),
        /*device_id=*/0,
        cyborg::kNone);

    std::array<uint8_t, 32> index_key;
    if (RAND_bytes(index_key.data(), index_key.size()) != 1) {
        throw std::runtime_error("Failed to generate secure random key");
    }

    auto index = client.CreateIndex("my_index", index_key);

    std::vector<std::string> ids = {"item_1", "item_2", "item_3"};
    cyborg::Array2D<float> vectors{
        {0.1f, 0.2f, 0.3f, 0.4f},
        {0.5f, 0.6f, 0.7f, 0.8f},
        {0.9f, 0.10f, 0.11f, 0.12f},
    };
    std::vector<std::vector<uint8_t>> contents = {
        {'H','e','l','l','o','!'},
        {'B','o','n','j','o','u','r','!'},
        {'H','o','l','a','!'},
    };
    index->Upsert(ids, vectors, contents, /*metadata=*/{}, index_key);

    cyborg::Array2D<float> query_vectors{{0.1f, 0.2f, 0.3f, 0.4f}};
    cyborg::QueryResults results = index->Query(query_vectors, cyborg::QueryParams{}, index_key);
    auto view = results[0];
    for (uint32_t i = 0; i < view.num_results; ++i) {
        std::cout << "ID: " << view.ids[i] << ", Distance: " << view.distances[i] << "\n";
    }

    std::vector<cyborg::ItemFields> include = {cyborg::ItemFields::kContents};
    std::vector<cyborg::Item> items = index->Get(ids, include, index_key);
    for (const auto& item : items) {
        std::cout << "ID: " << item.id
                  << ", Contents: " << std::string(item.contents.begin(), item.contents.end())
                  << "\n";
    }
}
```

## Verifying it works

Don't claim the setup is done until the program has actually run and printed:

- 3 lines of `ID: item_N, Distance: ...` from the query, and
- 3 lines of `ID: item_N, ..., Contents: ...` from the get.

If you can't run it (sandboxed env, remote machine), say so explicitly — don't assert success.

## Pointers for follow-up questions

- Storage backends: `StorageConfig.memory()`, `.disk(path)`, `.s3(bucket)`.
- API key (lifts the 1M-item cap): https://cyborgdb.co.
- Reference for the user's version: https://docs.cyborg.co/versions/v<X.Y>.x/embedded/
