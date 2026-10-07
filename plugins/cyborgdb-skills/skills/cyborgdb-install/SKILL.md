---
name: cyborgdb-install
description: Fully automated CyborgDB install — take the user from zero to a working first query with no manual steps. Installs the CyborgDB service with the default disk backing store (no API key, no storage config), loads the hosted sample dataset, and runs a first similarity query that returns non-empty results. Drives the install from the published docs at docs.cyborg.co so it stays accurate as docs change. Runs non-interactively, so it also works headless in CI against a clean container. Use when the user says "install CyborgDB", "set up CyborgDB", "try CyborgDB", "get CyborgDB running", or wants a zero-to-first-query demo.
---

# cyborgdb-install — zero to a working first query, no manual steps

Take the user from nothing installed to a **verified, non-empty first query** without asking them to edit files, copy-paste, or configure anything. Do every step yourself with tool calls. **Do not pause to ask questions** — every choice below has a default; use it. Only stop if the environment is genuinely missing a prerequisite you cannot install (no Docker *and* no Python ≥ 3.10, or no network).

Three invariants for this flow — they override anything older docs or memory suggest:

1. **No API key.** The service and SDKs run without `CYBORGDB_API_KEY` (free tier, 1M items per index). Never block the install on a key, never prompt for one.
2. **Default backing store.** Do not set `CYBORGDB_DB_TYPE`, connection strings, or any storage env vars. The default is local disk and it needs zero config.
3. **Non-empty first query.** Use the hosted sample dataset (`load_sample_dataset()`), never an empty index, so the first query provably returns results.

## Drive the install from the published docs

The install logic lives in the docs, not in this skill — fetch and follow the published quickstart so behavior tracks the current release without skill edits:

- Docs index (all pages): `https://docs.cyborg.co/llms.txt`
- Docker service quickstart: `https://docs.cyborg.co/versions/latest/service/guides/intro/quickstart-docker.md`
- Python service quickstart: `https://docs.cyborg.co/versions/latest/service/guides/intro/quickstart-python.md`
- Sample dataset reference: `https://docs.cyborg.co/versions/latest/service/python-sdk/load-sample-dataset.md`

Every docs page is fetchable as raw markdown by appending `.md`. **Fetch the quickstart for the path you pick and follow its commands.** The snippets in this skill are the verification contract and offline fallback — if they ever disagree with the fetched docs, the docs win.

If the docs are unreachable (offline/CI without egress to docs.cyborg.co), proceed with the fallback snippets below — they target v0.18.

## Step 1 — Pick the install path (silently)

Probe the environment; don't ask.

```bash
docker info >/dev/null 2>&1 && echo docker-ok
python3 -c 'import sys; print("py-ok" if sys.version_info >= (3,10) else "py-old")' 2>/dev/null
```

- **Docker daemon available** → Docker service path (default; closest to production).
- **No Docker, Python ≥ 3.10** → pip service path (`pip install cyborgdb-service`). Use a venv to avoid polluting the user's environment.
- **Neither** → report exactly what's missing and stop; that's the one legitimate blocker.

Either path also needs Python ≥ 3.10 for the client step (the `cyborgdb` SDK runs the first query).

## Step 2 — Start the service (detached, no config)

Follow the fetched quickstart. The shape it lands on (fallback if docs unreachable):

**Docker** — run detached, not interactive, so the flow continues without a human watching a foreground container:

```bash
docker pull cyborginc/cyborgdb-service:latest
docker run -d --name cyborgdb -p 8000:8000 \
  -v cyborgdb_data:/app/cyborgdb_data \
  cyborginc/cyborgdb-service:latest
```

No `CYBORGDB_API_KEY`, no storage env vars — the container starts with zero configuration on the disk backend. (On Linux `--network host` also works; `-p 8000:8000` works everywhere, so prefer it for an automated flow.)

**pip** (no Docker):

```bash
python3 -m venv .cyborgdb-venv && .cyborgdb-venv/bin/pip install cyborgdb-service
nohup .cyborgdb-venv/bin/cyborgdb-service > cyborgdb-service.log 2>&1 &
```

**Wait for health** — poll rather than sleep-and-hope; don't proceed until this passes:

```bash
for i in $(seq 1 60); do
  curl -sf http://localhost:8000/v1/health && break
  sleep 2
done
# Expect: {"status":"healthy","api_version":"v1","version":"0.18.0"}
```

If port 8000 is taken, remap (`-p 8001:8000` / `PORT=8001`) and carry the new port through the rest of the flow — don't stop to ask.

## Step 3 — First query against the sample dataset

Install the client SDK and run one self-contained script. Write it to a file yourself and run it — the user touches nothing.

```bash
pip install cyborgdb
```

```python
# first_query.py — CyborgDB first query against the hosted sample dataset
import cyborgdb

client = cyborgdb.Client(base_url="http://localhost:8000")
print(client.get_health())

# Hosted sample dataset: downloads on first use, cached in ~/.cache/cyborgdb.
dataset = cyborgdb.load_sample_dataset()
print(f"dataset: {dataset.name}  count={dataset.count}  dim={dataset.dimension}  metric={dataset.metric}")

# 32-byte per-index encryption key. For a throwaway demo this is fine in-memory;
# for anything the user wants to keep, tell them to store it in a secret manager —
# data encrypted with a lost key is unrecoverable.
index_key = client.generate_key(save=False)

index = client.create_index(
    index_name="cyborgdb_quickstart",
    index_key=index_key,
    dimension=dataset.dimension,
    metric=dataset.metric,
)

index.upsert(dataset.items)

results = index.query(query_vectors=dataset.sample_queries[0], top_k=5, include=["distance"])
assert len(results) > 0, "first query returned no results"
for r in results:
    print(f"ID: {r['id']}  Distance: {r['distance']:.4f}")

# Bonus: a curated metadata filter that is guaranteed to match.
example = dataset.example_filters[0]
filtered = index.query(query_vectors=dataset.sample_queries[0], top_k=5, filters=example["filter"])
print(f"filter demo ({example['name']}): {len(filtered)} results")
```

Run it: `python first_query.py`. The full dataset is 75k 128-dim vectors — a ~40MB download on first use (cached after), and the upsert typically completes in well under a minute. If the single upsert call times out on a slow host, retry in batches (e.g. 10k items per `index.upsert(...)` call) rather than shrinking the dataset.

## Step 4 — Verify before claiming success

Do not tell the user CyborgDB is installed until all three hold, with output as evidence:

1. `GET /v1/health` returned `"status": "healthy"` (note the `version` field — report it).
2. The script printed the dataset banner (name/count/dimension).
3. The first query printed **at least one result row** — non-empty is the whole point.

Then tell the user, concretely: what's running (image/package + version), where data lives (Docker volume `cyborgdb_data` or `~/.cyborgdb/data`), that no API key was needed (free tier, 1M items/index — a key from https://cyborgdb.co lifts the cap), and that the demo `index_key` was ephemeral — for real data they must generate and store one in a secret manager. Offer [[cyborgdb-sdks]] as the next step for integrating their own app — including metadata filtering (the filter demo above already ran one; the dataset's `example_filters` has more, covering equality, ranges, and lists) and full-text/hybrid BM25 search (needs `text_fields` designated at index creation — see the sdks skill). Or [[cyborgdb-core]] if they'd rather embed in-process Python with no server.

## Headless / CI mode

The same flow runs unattended in a clean container — it's already non-interactive. The CI contract:

- Exit non-zero if health never passes within the poll window, if the dataset download fails, or if the first query returns zero results (the `assert` above does this).
- Clean up when asked to leave no state: `docker rm -f cyborgdb && docker volume rm cyborgdb_data` (Docker) or kill the `cyborgdb-service` process and remove `~/.cyborgdb/data` + the venv (pip path).
- The sample dataset needs egress to its public S3 bucket; cache it (`~/.cache/cyborgdb`) between runs if the CI image supports it.

## Troubleshooting

- **Service exits at startup complaining about a missing API key** → that's a pre-key-removal release. Pull `cyborginc/cyborgdb-service:latest` / upgrade `cyborgdb-service` and rerun; do not work around it by configuring a key.
- **Port 8000 in use** → `lsof -i :8000`; remap and carry the port through.
- **`load_sample_dataset` download fails** → check egress; retry with `force_download=True` if a cached copy is corrupt (integrity is SHA-256-verified, so a bad cache is rejected automatically).
- **Health never passes under Docker** → `docker logs cyborgdb` and surface the actual error; on macOS remember the service is only reachable via the mapped port, not `--network host`.
