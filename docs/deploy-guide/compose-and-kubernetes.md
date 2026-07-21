# Compose & Kubernetes

For anything beyond a single-container pilot — separate API/worker scaling, an external managed
Postgres, or a Kubernetes-native rollout.

!!! note "The productized packaging for these paths isn't built yet"
    `deploy/compose/` and `deploy/helm/` exist as placeholders in the repo today — the actual
    multi-container compose file and Helm chart are on the roadmap under "Private-deployment
    additions," not yet implemented. What follows is the current, honest state of each path: what
    exists, what you'd build on top of it, and what's still open.

## Multi-container (Compose)

The root [`docker-compose.yml`](https://github.com/rnalex/ip-workflow/blob/main/docker-compose.yml)
is this project's own **development** setup — `api`, `worker`, `db`, and `otel-lgtm` as separate
containers against an external (not embedded) Postgres:

```bash
cp .env.example .env.local
docker compose --env-file .env.local up --build
```

It's a real, working multi-container topology, and everything in it (the API image, the worker
image, the migration-on-startup behavior) is the same code the standalone image and any future
production compose packaging would use — but it isn't hardened or documented as a production
artifact (no resource limits, no restart policies tuned for production, dev-oriented defaults
throughout). Treat it as a reference for the shape a production compose file would take, not as
something to point at customer traffic directly.

`deploy/compose/` is meant to hold the separate, productized version of this — layered on the
standalone image rather than built from source, with production-appropriate defaults. It's a
placeholder today.

## Kubernetes (Helm)

`deploy/helm/` is meant to hold the chart for private/air-gapped tiers, including the fully
air-gapped bundle: a signed OCI archive with images, model weights, and the prior-art corpus,
sized so a platform team can install it in a day per the roadmap's target. It's a placeholder
today — no chart exists yet.

If a Kubernetes deployment is a requirement for your evaluation now, the standalone image is
still deployable as a single-replica Deployment with a PersistentVolumeClaim for
`/var/lib/postgresql/standalone-data` in the interim, at the same single-worker throughput ceiling
described in the [standalone guide](standalone.md#sizing). This isn't a substitute for the
planned chart (no external managed-Postgres story, no horizontal worker scaling), but it's a
reasonable bridge for a pilot.

## What "production-shaped" will mean once built

Per the roadmap, so you know what to expect and can plan around it:

- External managed Postgres rather than embedded, for the same reasons any production Postgres
  workload wants a managed instance (backups, PITR, connection pooling at scale).
- API and worker scaled independently — the API is stateless HTTP, the worker is a
  `FOR UPDATE SKIP LOCKED` polling loop that can safely run multiple replicas.
- Signed license keys (features/seats/expiry) and versioned releases with automated migrations
  for the private-deployment tiers.
- A diagnostics bundle (sanitized, never disclosure content) for support without requiring log
  access to customer infrastructure.

## Next

- [Data flow & security](data-flow-and-security.md) for the egress picture that matters most for
  a Kubernetes/air-gap evaluation.
