# 9. Standalone-first packaging

## Status
Accepted

## Context
Deployment targets range from a design-partner pilot (wants the least infrastructure possible)
to enterprise air-gapped installs (wants a platform team to install it in a day via Helm).
Designing for Helm first and retrofitting a lightweight mode tends to produce a lightweight mode
that's lightweight in name only.

## Decision
Design standalone-first: a single container (embedded Postgres + in-process worker + statics),
then compose, then Helm, as the same artifact configured differently — not separate builds. This
forces a Postgres-only queue (no separate broker) and an embeddable worker. No single-binary
target: the container is the artifact.

## Consequences
The pipeline orchestrator (Design & Architecture, Components §2) must be built against a
Postgres-backed queue from the start. Anything that can't run in a single container without
external services either waits for the compose/Helm tiers or gets redesigned to fit.
