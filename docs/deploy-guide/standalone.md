# Standalone quickstart

The fastest path to a working pilot: one image, one container, no external services. Embedded
Postgres and the worker both run inside the same process as the API
([`deploy/standalone/`](https://github.com/rnalex/ip-workflow/blob/main/deploy/standalone)).
This is what BYO-cloud and air-gapped deployments are built from too — it's the same image,
configured differently, not a separate build.

## Build and run

```bash
docker build -f deploy/standalone/Dockerfile -t patent-triage-standalone .
docker run -d --name patent-triage --env-file .env.local -p 8000:8000 patent-triage-standalone
curl localhost:8000/health   # {"status": "ok"}
```

Open `http://localhost:8000/` — the API serves the built web app directly, no separate frontend
deployment.

## What to put in `.env.local`

At minimum, credentials for whichever LLM path you're using — see
[Configuration reference](../handbook/configuration.md) for the full set:

```bash
ANTHROPIC_API_KEY=...
VOYAGE_API_KEY=...
```

or, to use your own OpenAI-compatible endpoint instead (self-hosted vLLM, an Azure/Bedrock
gateway):

```bash
LLM_BASE_URL=https://your-endpoint/v1
LLM_REASONER_MODEL=...
LLM_WORKER_MODEL=...
```

If you want SSO ([ADR 0012](../adr/0012-oidc-sso.md)), also set `PUBLIC_APP_BASE_URL` to this
deployment's actual reachable origin (e.g. `https://patent-triage.your-company.example`) — the
API serves the web app itself here, so unlike the split dev setup there's no proxy/origin
mismatch to worry about, just the real public URL. The rest of an SSO config (issuer, client ID,
secret, allowed domain) is entered once by an org admin in-app, not an env var.

Note: `docker run --env-file .env.local` (used here) really does inject every variable in the
file into the container's environment — that's different from `docker compose --env-file`, which
only uses the file for `${VAR}` substitution *inside* `docker-compose.yml` and still requires
each variable to be listed explicitly under a service's `environment:` block (see the
[Compose & Kubernetes](compose-and-kubernetes.md) page, or ADR 0012's Consequences for the bug
this caused).

Without either, the container still starts and serves the web app and API — intake and the
review queue work fine — but the worker's pipeline calls fail per-job rather than crashing the
process, so submitted disclosures won't progress past intake.

## Data persistence

Postgres runs as a real embedded server (`initdb` on first boot, not an in-process/emulated
engine), with its data directory under `$PGDATA` inside the container's writable layer by
default. **Mount a volume there**, or data is lost on `docker rm`:

```bash
docker run -d --name patent-triage --env-file .env.local \
  -v patent-triage-data:/var/lib/postgresql/standalone-data \
  -p 8000:8000 patent-triage-standalone
```

`AUTH_JWT_SECRET` behaves the same way: if you don't supply one, the entrypoint generates one at
boot, but it isn't persisted across restarts of a fresh container — fine for a demo, but supply
your own explicitly (`python3 -c "import secrets; print(secrets.token_hex(32))"`) for anything
where sessions need to survive a restart.

## Sizing

There's no published benchmark yet — treat this as a pilot/demo tier for a handful of concurrent
users, not a sized production recommendation. The worker processes one pipeline job at a time by
design (see [Job queue mechanics](../handbook/workflows/job-queue.md)), so throughput under load
is a single-worker-in-one-container ceiling; the [Compose & Kubernetes](compose-and-kubernetes.md)
path is the one to move to once you need more than that.

## Next

- [Compose & Kubernetes](compose-and-kubernetes.md) for a multi-container, horizontally-scalable
  shape.
- [Data flow & security](data-flow-and-security.md) for what this tier sends outbound and how to
  reduce it.
