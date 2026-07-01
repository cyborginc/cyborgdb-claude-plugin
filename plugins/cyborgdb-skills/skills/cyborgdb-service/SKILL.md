---
name: cyborgdb-service
description: Help the user start and run cyborgdb-service (the standalone CyborgDB HTTP server) end-to-end. Detects which cyborgdb-service version the user has (Docker image tag or pip package), points at the matching v0.X.x docs on docs.cyborg.co, launches the service, and verifies /v1/health. Use when the user wants to start the service, run the server, troubleshoot a service that won't start, or get the service up before pointing SDKs at it. Triggers like "start cyborgdb-service", "run cyborgdb server", "docker run cyborgdb", "set up cyborgdb-service with disk/S3/MinIO", "service won't start".
---

# cyborgdb-service — start it and verify it's running

Help the user **launch the service and confirm it's healthy**. **First detect which version they're targeting** (or pin one), then follow the matching quickstart from `docs.cyborg.co/versions/v<MAJOR.MINOR>.x/...`. The inline commands below target **v0.17**; older versions have different storage-backend env vars (see *Version-specific gotchas*).

After the service is healthy, hand off to [[cyborgdb-sdks]] for calling it from code. If the user wants in-process (no service), redirect to [[cyborgdb-core]].

## Step 0 — Detect the version (do this first)

Try in order; stop as soon as you have a version.

**Docker (already pulled):**

```bash
docker image inspect cyborginc/cyborgdb-service:latest \
  --format '{{ index .Config.Labels "org.opencontainers.image.version" }}' 2>/dev/null
# Or, if a specific tag is in use:
docker images cyborginc/cyborgdb-service --format '{{.Tag}}'
```

**Docker (service already running):** the `/v1/health` response contains `version`:

```bash
curl -s http://localhost:8000/v1/health
# {"status":"healthy","api_version":"v1","version":"0.17.0"}
```

**Python (pip-installed):**

```bash
pip show cyborgdb-service 2>/dev/null | awk '/^Version:/ {print $2}'
```

If **nothing is installed/running yet**, resolve the latest from PyPI and proceed with that version — don't pause to ask, and don't hardcode a minor:

```bash
curl -s https://pypi.org/pypi/cyborgdb-service/json | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])"
```

Pulling `cyborginc/cyborgdb-service:latest` also lands on that same latest minor. Only switch versions if the user explicitly asks to pin an older one. If PyPI is unreachable, fall back to **v0.17** so setup can still proceed offline. If PyPI returns a **newer minor than v0.17**, the storage-backend flags below may be stale — fetch that version's quickstart page.

Once you know `vX.Y.Z`:

- **Docs base:** `https://docs.cyborg.co/versions/v<X.Y>.x/`
- **Docker quickstart:** `https://docs.cyborg.co/versions/v<X.Y>.x/service/guides/intro/quickstart-docker`
- **Python quickstart:** `https://docs.cyborg.co/versions/v<X.Y>.x/service/guides/intro/quickstart-python`

For any non-trivial config (especially storage backend), **fetch the matching quickstart page** and follow it.

## Version-specific gotchas

- **v0.17:** Storage backends are `disk` (default), `memory`, `s3`. New env vars: `CYBORGDB_DB_TYPE`, `CYBORGDB_DISK_PATH`, `CYBORGDB_S3_*`. Disk data dir is `/app/cyborgdb_data` inside the container, `~/.cyborgdb/data` for the pip-installed service.
- **v0.16 and earlier:** older backend names (`standalone`, `postgres`, `redis`) and the catch-all `CYBORGDB_CONNECTION_STRING` are still in play. **Do not use the v0.17 storage flags below verbatim on older versions — fetch their quickstart.**

## Before you start

- The service requires `CYBORGDB_API_KEY`. Production keys: https://cyborgdb.co. For evaluation, any SDK mints a demo key — Python: `cyborgdb.get_demo_api_key()`; JS: `getDemoApiKey()`; Go: `cyborgdb.GetDemoAPIKey("my demo app")`. Surface this if the user doesn't already have one.
- Storage backend defaults to **disk**. Other options: `memory` (ephemeral, tests only), `s3` (AWS or S3-compatible). Ask which they want; default to disk for evaluation.
- Service auth is **off by default**. To require API-key auth on calls, set `CYBORGDB_SERVICE_ROOT_KEY` on the service — SDK clients then must pass the matching `api_key`.

## Steps (Docker — default)

Default to a specific minor tag if the user gave one (e.g. `:v0.17.0`); otherwise pull `:latest`.

1. **Confirm `CYBORGDB_API_KEY`** with the user. Offer the demo-key route above if they don't have one.
2. **Pull:**
   ```bash
   docker pull cyborginc/cyborgdb-service:latest
   ```
3. **Run.** Pick the line that matches the host. The `-v` mount is what makes disk data survive container recreation.

   Linux (host networking):
   ```bash
   sudo docker run -it --rm --network host \
     -e CYBORGDB_API_KEY=cyborg_your_api_key_here \
     -v cyborgdb_data:/app/cyborgdb_data \
     cyborginc/cyborgdb-service:latest
   ```

   macOS (Docker Desktop is in a VM — needs port mapping):
   ```bash
   sudo docker run -it -p 8000:8000 \
     -e CYBORGDB_API_KEY=cyborg_your_api_key_here \
     -v cyborgdb_data:/app/cyborgdb_data \
     cyborginc/cyborgdb-service:latest
   ```

4. **For anything more than a one-off**, scaffold `docker-compose.yml`:
   ```yaml
   services:
     cyborgdb:
       image: cyborginc/cyborgdb-service:latest
       ports:
         - "8000:8000"
       environment:
         - CYBORGDB_API_KEY=${CYBORGDB_API_KEY}
       volumes:
         - cyborgdb_data:/app/cyborgdb_data
         - ./logs:/app/logs
   volumes:
     cyborgdb_data:
   ```
   Run: `docker-compose up -d`

5. **Verify** (next section).

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
   export CYBORGDB_API_KEY=cyborg_your_api_key_here
   cyborgdb-service
   ```
   `cyborgdb-service --help` lists CLI flags.
4. **Verify** (next section).

## Picking a storage backend (v0.17 syntax)

Add these env vars in addition to `CYBORGDB_API_KEY`. **For older versions, fetch the matching docs page — the var names differ.**

| Backend | Vars |
|---|---|
| **Memory** (ephemeral, tests only) | `CYBORGDB_DB_TYPE=memory` |
| **S3 (AWS)** | `CYBORGDB_DB_TYPE=s3`, `CYBORGDB_S3_BUCKET=...`, `CYBORGDB_S3_REGION=...`. On EC2/ECS/EKS the instance/task role is picked up; off-AWS set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`. |
| **S3-compatible (MinIO / R2)** | `CYBORGDB_DB_TYPE=s3`, `CYBORGDB_S3_BUCKET=...`, `CYBORGDB_S3_ENDPOINT=http://minio:9000`, `CYBORGDB_S3_ACCESS_KEY=...`, `CYBORGDB_S3_SECRET_KEY=...`. The custom endpoint bypasses the AWS credential chain — the endpoint-specific keys are required. |

Full env-var reference for the user's version: `https://docs.cyborg.co/versions/v<X.Y>.x/service/guides/advanced/env-vars`

## Verifying it works

Don't claim the service is running until the health check passes:

```bash
curl http://localhost:8000/v1/health
# Expect: {"status":"healthy","api_version":"v1","version":"0.17.0"}
```

The `version` field is also how you confirm the running service matches the version you targeted.

Open the interactive API docs at http://localhost:8000/v1/docs to confirm routes are live.

If the user runs the service somewhere you can't curl from, hand them the command and say so explicitly — don't assert healthy without evidence.

## Troubleshooting common failures

- **Port already in use:** another process on `:8000`. `lsof -i :8000` to find it, or change the port via `-p 8001:8000` (Docker) / `PORT=8001` (pip).
- **`CYBORGDB_API_KEY` missing/invalid:** service exits at startup; surface the exact log line, then ask the user for a fresh key or generate a demo key with an SDK.
- **S3 backend won't start:** check `CYBORGDB_S3_BUCKET` exists and credentials resolve. With `CYBORGDB_S3_ENDPOINT` set, the AWS default credential chain is bypassed — `CYBORGDB_S3_ACCESS_KEY` / `CYBORGDB_S3_SECRET_KEY` are required.
- **macOS Docker can't reach the host:** use `host.docker.internal` from inside the container instead of `localhost`.

## Pointers for follow-up questions

- HTTPS auto-enables when both `SSL_CERT_PATH` and `SSL_KEY_PATH` exist on disk.
- Authentication is opt-in: set `CYBORGDB_SERVICE_ROOT_KEY` on the service to require SDK `api_key`s.
- Production process management: gunicorn for the pip-installed service (`gunicorn cyborgdb_service.main:app --port 8000`), or a systemd unit / container orchestrator for Docker.
- KMS/BYOK, multi-tenancy, YAML config: `https://docs.cyborg.co/versions/v<X.Y>.x/service/guides/advanced/`
