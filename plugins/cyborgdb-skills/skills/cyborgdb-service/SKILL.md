---
name: cyborgdb-service
description: Help the user start and run cyborgdb-service (the standalone CyborgDB HTTP server) end-to-end. Detects which cyborgdb-service version the user has (Docker image tag or pip package), points at the matching docs on docs.cyborg.co, launches the service with zero required configuration (no API key, disk storage by default), and verifies /v1/health. Use when the user wants to start the service, run the server, troubleshoot a service that won't start, or get the service up before pointing SDKs at it. Triggers like "start cyborgdb-service", "run cyborgdb server", "docker run cyborgdb", "set up cyborgdb-service with disk/S3/MinIO", "service won't start".
---

# cyborgdb-service — start it and verify it's running

Help the user **launch the service and confirm it's healthy**. **First detect which version they're targeting** (or pin one), then follow the matching quickstart on docs.cyborg.co. The inline commands below target **v0.18**; older versions differ (see *Version-specific gotchas*).

The service starts with **zero required configuration**: no API key, disk storage by default, auth off. Don't add env vars the user didn't ask for.

For a fully automated zero-to-first-query install (sample dataset included), use [[cyborgdb-install]]. After the service is healthy, hand off to [[cyborgdb-sdks]] for calling it from code. If the user wants in-process (no server), redirect to [[cyborgdb-core]].

## Step 0 — Detect the version (do this first)

Try in order; stop as soon as you have a version.

**Service already running:** the `/v1/health` response contains `version` — this is authoritative:

```bash
curl -s http://localhost:8000/v1/health
# {"status":"healthy","api_version":"v1","version":"0.18.0"}
```

**Docker (pulled but not running):** read the image tag — the image carries no version label, so the tag is the only offline signal, and `:latest` tells you nothing (start the container and use the health check instead):

```bash
docker images cyborginc/cyborgdb-service --format '{{.Tag}}'
```

**Python (pip-installed):**

```bash
pip show cyborgdb-service 2>/dev/null | awk '/^Version:/ {print $2}'
```

If **nothing is installed/running yet**, resolve the latest from PyPI and proceed with that version — don't pause to ask, and don't hardcode a minor:

```bash
curl -s https://pypi.org/pypi/cyborgdb-service/json | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])"
```

Pulling `cyborginc/cyborgdb-service:latest` also lands on that same latest minor. Only switch versions if the user explicitly asks to pin an older one. If PyPI is unreachable, fall back to **v0.18** so setup can still proceed offline. If PyPI returns a **newer minor than v0.18**, the inline commands below may be stale — fetch that version's quickstart page.

**Docs URLs** — the current release lives under `versions/latest/`; pinned older minors under `versions/v<X.Y>.x/`. Every page is fetchable as raw markdown by appending `.md`, and `https://docs.cyborg.co/llms.txt` indexes every page.

- **Docker quickstart:** `https://docs.cyborg.co/versions/latest/service/guides/intro/quickstart-docker`
- **Python quickstart:** `https://docs.cyborg.co/versions/latest/service/guides/intro/quickstart-python`
- **Older pinned version:** swap `latest` for `v<X.Y>.x` (e.g. `v0.17.x`)

For any non-trivial config (especially storage backend), **fetch the matching quickstart page** and follow it.

## Version-specific gotchas

- **v0.17.1+ / v0.18:** No API key required — the service starts in free-tier mode (1M items per index) with `CYBORGDB_API_KEY` unset. Storage backends: `disk` (default), `s3`. Disk data dir is `/app/cyborgdb_data` inside the container, `~/.cyborgdb/data` for the pip-installed service.
- **Older releases:** required `CYBORGDB_API_KEY` at startup — if the service exits complaining about a missing key, the user is on a pre-key-removal build; recommend upgrading rather than configuring a key. Pre-v0.17 also used older backend names (`standalone`, `postgres`, `redis`) and `CYBORGDB_CONNECTION_STRING` — fetch that version's quickstart instead of using the commands below.

## Before you start

- **No API key needed.** Unset `CYBORGDB_API_KEY` = free-tier mode, capped at 1M items per index. A key from https://cyborgdb.co (or a demo key via any SDK: Python `cyborgdb.get_demo_api_key()`, JS `getDemoApiKey()`, Go `cyborgdb.GetDemoAPIKey("my demo app")`) lifts the cap. It's a license key, not an auth credential.
- Storage backend defaults to **disk** with no configuration. Only reach for `s3` (AWS or S3-compatible) if the user asks for it.
- Service auth is **off by default**. To require API-key auth on calls, set `CYBORGDB_SERVICE_ROOT_KEY` on the service — SDK clients then must pass the matching `api_key`.

## Steps (Docker — default)

Default to a specific minor tag if the user gave one (e.g. `:v0.18.0`); otherwise pull `:latest`.

1. **Pull:**
   ```bash
   docker pull cyborginc/cyborgdb-service:latest
   ```
2. **Run.** No env vars required. The `-v` mount is what makes disk data survive container recreation. Use `-d` instead of `-it` when you're driving the install yourself and need the shell back.

   Linux (host networking):
   ```bash
   docker run -it --rm --network host \
     -v cyborgdb_data:/app/cyborgdb_data \
     cyborginc/cyborgdb-service:latest
   ```

   macOS (Docker Desktop is in a VM — needs port mapping):
   ```bash
   docker run -it -p 8000:8000 \
     -v cyborgdb_data:/app/cyborgdb_data \
     cyborginc/cyborgdb-service:latest
   ```

3. **For anything more than a one-off**, scaffold `docker-compose.yml`:
   ```yaml
   services:
     cyborgdb:
       image: cyborginc/cyborgdb-service:latest
       ports:
         - "8000:8000"
       # No API key needed; add CYBORGDB_API_KEY to lift the free-tier cap:
       # environment:
       #   - CYBORGDB_API_KEY=${CYBORGDB_API_KEY}
       volumes:
         - cyborgdb_data:/app/cyborgdb_data
         - ./logs:/app/logs
   volumes:
     cyborgdb_data:
   ```
   Run: `docker-compose up -d`

4. **Verify** (next section).

## Steps (Python service — pip alternative)

Same REST API, same env vars. Disk data lives at `~/.cyborgdb/data` by default.

1. **Python 3.10–3.14.** Strongly recommend a conda env or venv to avoid dependency conflicts.
2. **Install:**
   ```bash
   pip install cyborgdb-service
   # Optional managed embeddings:
   # pip install cyborgdb-service[embeddings]
   ```
3. **Run:**
   ```bash
   cyborgdb-service
   ```
   No env vars required. `cyborgdb-service --help` lists CLI flags.
4. **Verify** (next section).

## Picking a storage backend (v0.18 syntax)

Only needed when the user explicitly wants something other than the disk default. **For older versions, fetch the matching docs page — the var names differ.**

| Backend | Vars |
|---|---|
| **Disk** (default) | none — optionally `CYBORGDB_DISK_PATH` to move the data dir |
| **S3 (AWS)** | `CYBORGDB_DB_TYPE=s3`, `CYBORGDB_S3_BUCKET=...`, `CYBORGDB_S3_REGION=...`. On EC2/ECS/EKS the instance/task role is picked up; off-AWS set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`. |
| **S3-compatible (MinIO / R2)** | `CYBORGDB_DB_TYPE=s3`, `CYBORGDB_S3_BUCKET=...`, `CYBORGDB_S3_ENDPOINT=http://minio:9000`, `CYBORGDB_S3_ACCESS_KEY=...`, `CYBORGDB_S3_SECRET_KEY=...`. The custom endpoint bypasses the AWS credential chain — the endpoint-specific keys are required. |

Full env-var reference: `https://docs.cyborg.co/versions/latest/service/guides/advanced/env-vars`

## Verifying it works

Don't claim the service is running until the health check passes:

```bash
curl http://localhost:8000/v1/health
# Expect: {"status":"healthy","api_version":"v1","version":"0.18.0"}
```

The `version` field is also how you confirm the running service matches the version you targeted.

Open the interactive API docs at http://localhost:8000/v1/docs to confirm routes are live.

If the user runs the service somewhere you can't curl from, hand them the command and say so explicitly — don't assert healthy without evidence.

## Troubleshooting common failures

- **Port already in use:** another process on `:8000`. `lsof -i :8000` to find it, or change the port via `-p 8001:8000` (Docker) / `PORT=8001` (pip).
- **Service exits demanding `CYBORGDB_API_KEY`:** pre-key-removal build. Pull `:latest` / upgrade the pip package — don't configure a key to work around it.
- **S3 backend won't start:** check `CYBORGDB_S3_BUCKET` exists and credentials resolve. With `CYBORGDB_S3_ENDPOINT` set, the AWS default credential chain is bypassed — `CYBORGDB_S3_ACCESS_KEY` / `CYBORGDB_S3_SECRET_KEY` are required.
- **macOS Docker can't reach the host:** use `host.docker.internal` from inside the container instead of `localhost`.

## Pointers for follow-up questions

- HTTPS auto-enables when both `SSL_CERT_PATH` and `SSL_KEY_PATH` exist on disk.
- Authentication is opt-in: set `CYBORGDB_SERVICE_ROOT_KEY` on the service to require SDK `api_key`s.
- Production process management: run the pip-installed service under a supervisor (systemd unit, etc.) via its own entrypoint — `cyborgdb-service` (or equivalently `uvicorn cyborgdb_service.app:app --host 0.0.0.0 --port 8000`). The service is single-worker by design — don't front it with a multi-worker process manager. For Docker, use a container orchestrator or restart policy.
- KMS/BYOK, multi-tenancy, YAML config: `https://docs.cyborg.co/versions/latest/service/guides/advanced/`
