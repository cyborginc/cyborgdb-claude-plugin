---
name: cyborgdb-migrate
description: Migrate vector data from Pinecone, Qdrant, Weaviate, ChromaDB, or Milvus into CyborgDB using the cyborgdb-migrate tool. Drives the whole migration end-to-end — installs the tool with the right source extra, writes a headless TOML config, runs the migration with checkpoint/resume, captures the generated encryption key, and verifies the result with spot checks. Use when the user says "migrate to CyborgDB", "move my Pinecone/Qdrant/Weaviate/Chroma/Milvus data", "import my vectors into CyborgDB", or asks how to switch from another vector database.
---

# cyborgdb-migrate — move vectors from another vector DB into CyborgDB

Run the migration for the user with tool calls wherever possible. Use the **headless CLI mode** (`--config`) — the interactive TUI exists (`cyborgdb-migrate` with no args) but is for humans at a terminal; only suggest it if the user explicitly wants to drive it themselves.

Supported sources: **Pinecone, Qdrant, Weaviate, ChromaDB, Milvus**.

## Non-negotiable safety rules

1. **Capture the encryption key.** When creating a new non-KMS index, the tool generates an encryption key and prints it once as `Generated encryption key (hex): …` — on stderr only, never in the log file, and not at all under `--quiet`. **If this key is lost, the migrated data is unrecoverable.** Never pass `--quiet` on a key-generating run, capture the key from stderr before backgrounding anything, and save it to a mode-600 file under `~/.cyborgdb/keys/` as a local recovery copy (see Step 4) while telling the user to store it in a secrets manager as the durable home. Never commit it.
2. **No secrets in the config file.** The TOML supports `${VAR_NAME}` expansion anywhere — reference API keys as environment variables (`api_key = "${PINECONE_API_KEY}"`), never paste raw keys into the file. Write the config to a path outside any git repo (or confirm it's gitignored).
3. **Never guess credentials.** If a source API key or host isn't available in the environment, ask the user for it (or where it lives) — don't invent values or probe speculatively.

## Step 1 — Establish the two endpoints

**Destination:** a running CyborgDB service URL. If CyborgDB isn't running yet, run the `cyborgdb-install` skill first (or point at their existing deployment). A service set up by the install skill has no authentication configured — no API key exists, and that's fine (see Step 3). The tool does a version-compatibility health check on connect and fails fast on mismatch — if that fires, upgrade the service or `pip install -U cyborgdb-migrate` rather than working around it.

**Source:** which database, and how to reach it. Per-source credential keys for the `[source]` section:

| Source | `type` | Credential keys (beyond `index`) |
|--------|--------|-------------------------------|
| Pinecone | `pinecone` | `api_key` (required); `namespace` optional |
| Qdrant | `qdrant` | `host` (default `http://localhost:6333`); `api_key` optional |
| Weaviate | `weaviate` | `host` (default `http://localhost:8080`); `api_key` optional |
| ChromaDB | `chromadb` | `mode` = `Remote` (default, uses `host`, default `localhost:8100`) or `Local` (uses `path` to the persist directory) |
| Milvus | `milvus` | `uri` (default `http://localhost:19530`); `token` optional; `database` (default `default`); `namespace` = partition, optional |

`index` is the source index/collection name in every case.

## Step 2 — Install the tool

```bash
pip install "cyborgdb-migrate[<source>]>=0.18"   # pinecone | qdrant | weaviate | chromadb | milvus
```

Prefer a venv if the environment isn't already one. `[all]` installs every source extra. The `>=0.18` floor matters: it upgrades any pre-existing older install, and versions before 0.18 **silently ignore** `text_fields` and `metadata_schema` — the migration would succeed but land without the full-text/metadata settings, which can't be added to the index afterward.

## Step 3 — Write the TOML config

Minimal new-index migration:

```toml
[source]
type = "pinecone"
api_key = "${PINECONE_API_KEY}"
index = "my-source-index"

[destination]
host = "http://localhost:8000"
api_key = "unused"
index_name = "my-cyborgdb-index"
create_index = true

[options]
batch_size = 200
checkpoint_every = 10
spot_check_per_batch = 4
```

**Destination `api_key`:** the field is required and must be non-empty, but a service with no authentication configured (the `cyborgdb-install` default) ignores its value — write the literal placeholder `api_key = "unused"`. Do **not** export `CYBORGDB_API_KEY` to fill it: that variable is read by the *service* as a license key and will make the server fail to start. Only if the user's deployment actually has auth set up should you use their real key — then reference it via a differently-named env var (e.g. `api_key = "${CYBORG_DEST_API_KEY}"`), per safety rule 2.

Decisions to make with the user before running (all are **fixed at index creation and immutable afterward**):

- **Key management:** default is a generated key (see safety rule 1). If they have a KMS slot registered on the service, set `kms_name = "<slot>"` instead — then no key is generated or needed.
- **Full-text search:** if they want BM25 over metadata fields, set `text_fields = ["title", "body"]` (optional `bm25_k1`, `bm25_b`).
- **Metadata indexing policy:** optional `[destination.metadata_schema]` with per-field `{ filterable = ..., pattern = ..., full_text = ... }`; unlisted fields are filterable by default.
- **Storage precision:** `storage_precision` defaults to `float32`. `float16` halves storage with small recall loss; TurboQuant tiers `tq12` / `tq8` / `tq6` / `tq4` compress further (`tq4` is ~8× smaller than `float32` at ~94% recall@100).
- **Embedding model:** optional `embedding_model = "<HuggingFace model>"` if they want the index to auto-embed new content after migration (upsert `contents` / query with `query_contents`). The migrated vectors themselves come from the source unchanged — this only affects data added later, but like everything else here it can only be set now.
- Dimension and distance metric are read from the source automatically — don't ask for them.

**Migrating into an existing index** instead: set `create_index = false` and supply exactly one of `index_key` (hex), `key_file`, or `kms_name`. The tool validates that source and destination dimensions match and exits 1 if not.

## Step 4 — Run it

```bash
cyborgdb-migrate --config migration.toml
```

Progress goes to stderr; a log lands at `./cyborgdb-migrate.log` (override with `--log-file`).

**Key capture comes before everything else on a new non-KMS index.** The `Generated encryption key (hex): …` line is printed **once, to stderr only** — it never appears in the log file, and `--quiet` suppresses it entirely. So:

- **Never pass `--quiet`** on a run that creates a non-KMS index — the key would be shown nowhere and the migrated data unrecoverable.
- **Don't background the run until the key is saved.** Capture stderr to a file and wait for the key line before leaving the process alone:

  ```bash
  cyborgdb-migrate --config migration.toml 2> migrate-stderr.log &
  until grep -m1 "Generated encryption key (hex):" migrate-stderr.log; do
    kill -0 $! 2>/dev/null || { echo "migrate exited before printing a key" >&2; break; }
    sleep 1
  done
  ```

  Then save the key to a protected file right away — don't rely on the user having stored it yet:

  ```bash
  mkdir -p ~/.cyborgdb/keys
  (umask 077; grep -m1 "Generated encryption key (hex):" migrate-stderr.log | awk '{print $NF}' > ~/.cyborgdb/keys/<index_name>.key)
  rm migrate-stderr.log   # safe mid-run: the process keeps its open handle
  ```

  Hand the key to the user and tell them two things: it's saved at `~/.cyborgdb/keys/<index_name>.key` (owner-only permissions, outside any git repo) as a local recovery copy, and they should still store it in a secrets manager — the file dies with this machine. Monitor `./cyborgdb-migrate.log` for the rest of the run. Future runs into the same index can set `create_index = false` and point `key_file` straight at this path.
- **For long migrations, prefer a key the user already holds:** create the destination index yourself first (SDK `create_index` with a key from their secret manager), then run migrate with `create_index = false` + `key_file` — no key is ever printed and backgrounding is safe from the start. (`key_file` is only read when `create_index = false`.)

Runs with `kms_name` or against an existing index generate no key — background those freely.

## Step 5 — Handle the outcome

| Exit code | Meaning | What to do |
|-----------|---------|------------|
| `0` | Success, spot check passed | Report vectors migrated and duration; suggest a test query against the new index |
| `1` | Config/connection/dimension/version error | Read the stderr message and `cyborgdb-migrate.log`, fix the config, retry |
| `2` | Migration finished but spot-check verification failed | Don't declare success — report it. Check the log for which vectors mismatched before re-running |

**Interrupted migration?** Checkpoints are saved every `checkpoint_every` batches to `./cyborgdb-migrate-checkpoints/<type>_<source-index>_<dest-index>.json`. Resume with:

```bash
cyborgdb-migrate --config migration.toml --resume
```

(`--resume` only works with `--config`; the checkpoint dir is relative to the working directory, so resume from the same directory.)

## Step 6 — Verify beyond the spot check

The built-in spot check samples `spot_check_per_batch` vectors per batch for vector + metadata integrity. After exit 0, close the loop for the user:

- Confirm the reported `Vectors: N / N` counts match the source's vector count.
- Run one similarity query against the new CyborgDB index (Python SDK: `cyborgdb.Client` → `load_index` with the saved key → `query`) and show non-empty results.
- Remind the user the source data was only read, never modified — they decommission the source on their own schedule.
