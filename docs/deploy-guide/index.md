# Deploying patent-triage

This guide is for a customer's platform, security, or IT team evaluating how to run patent-triage
in your own environment — not for engineers working on this codebase (that's the
[engineering handbook](../handbook/index.md)). It answers three questions: which tier fits you,
what you need to provide, and how to stand each one up.

Disclosures are pre-filing IP. Every tier below is designed around that: the software runs on the
same codebase everywhere ([ADR 0009](../adr/0009-standalone-first-packaging.md)), and moving to a
tier with a stronger trust story is a configuration change, never a rewrite or a fork.

## Choose a tier

| Tier | What it is | Trust story | Where your data goes |
|---|---|---|---|
| **SaaS** | We host it, multi-tenant | DPA + SOC 2 + subprocessor list | Our infrastructure, plus whichever LLM/search subprocessors are in the DPA |
| **BYO-cloud** *(recommended starting point)* | Our software, deployed into your cloud account; your LLM tenancy | "Your data never touches us," at roughly 30% of air-gap cost | Your cloud account; your LLM provider |
| **Private / air-gapped** | Fully on-prem bundle, no outbound dependency | Complete isolation | Nowhere — no network egress required |

Recommended sequencing for a first pilot: **SaaS pilot → BYO-cloud → air-gap only once a funded
deployment specifically requires it.** Air-gap is the most expensive tier to operate and support,
and most of its trust benefits are already available from BYO-cloud once you also enable the
local prior-art corpus (see [Data flow & security](data-flow-and-security.md)).

## What you'll need to provide

Every tier needs the same things; only *where they come from* changes.

| Requirement | SaaS | BYO-cloud | Air-gapped |
|---|---|---|---|
| **Session signing secret** (`AUTH_JWT_SECRET`) | We generate it | You generate it (`python3 -c "import secrets; print(secrets.token_hex(32))"`) — mandatory, the API refuses to start without one | Same as BYO-cloud |
| **LLM + embeddings credentials** | We provide (in the DPA) | Your Anthropic/Voyage keys, your OpenRouter account, or your own OpenAI-compatible endpoint (self-hosted vLLM, Azure/Bedrock gateway) | Your own self-hosted endpoint — no calls leave your network |
| **Outbound mail (SMTP)** | Handled for you | Point at your own mail infrastructure | Point at your own mail infrastructure, or leave unset — invite/verification flows are DB-state-driven and function without delivery, they just won't email anyone |
| **Prior-art search** | External APIs (arXiv, Semantic Scholar, and whichever of PatentsView/EPO you configure) | Same, optionally joined by a local corpus mirror to cut egress | Local corpus only — no external search calls |
| **Postgres** | Managed by us | A Postgres instance in your account (or the embedded one in the standalone image, for a pilot) | Embedded Postgres, no external dependency |
| **SSO** *(optional; [ADR 0012](../adr/0012-oidc-sso.md))* | We configure it if requested | `PUBLIC_APP_BASE_URL` set to your deployment's single reachable origin, plus your IdP's issuer/client ID/secret — entered once by an org admin in-app (Org → Single sign-on), not an env var | Same as BYO-cloud, pointed at an internal IdP if you have one |

Password login works out of the box with nothing beyond the session signing secret — SSO is
additive, not a prerequisite. None of the above requires code changes — every one is either an
environment variable read once at startup or, for SSO specifically, in-app admin configuration.
See [Configuration reference](../handbook/configuration.md) for the full env var list, or jump
straight to a specific path:

- **[Standalone quickstart](standalone.md)** — one container, fastest way to a working pilot.
- **[Compose & Kubernetes](compose-and-kubernetes.md)** — multi-container and Helm paths for
  production-shaped deployments.
- **[Data flow & security](data-flow-and-security.md)** — what leaves your network, per tier, and
  how to close the gaps that matter most to a security review.
- **[Fully local deployment](fully-local.md)** — the local LLM/corpus target, known gaps,
  one-machine experiment, and measured cost break-even method.

## Current state, honestly

The **standalone single-container image is the only fully productized artifact today** — build
it, run it, everything (embedded Postgres, in-process worker, the web app) is in one container.
The `deploy/compose/` and `deploy/helm/` packaging layers referenced in the roadmap are placeholders
right now; see [Compose & Kubernetes](compose-and-kubernetes.md) for what that means in practice
and what to use in the meantime. If your evaluation depends on Kubernetes packaging on a specific
timeline, raise it — it's a known, tracked gap, not an oversight.
