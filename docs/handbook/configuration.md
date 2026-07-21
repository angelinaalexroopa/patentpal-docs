# Configuration reference

Every environment variable the application reads, grouped by concern. `.env.example` in the repo
root is the canonical, always-current template — copy it to `.env.local` and fill in what you
need; this page explains the *why* behind each group.

## Required

| Variable | Required for | Notes |
|---|---|---|
| `DATABASE_URL` | everything | Postgres connection string |
| `AUTH_JWT_SECRET` | `services/api` | signs/verifies session tokens ([ADR 0010](../adr/0010-authentication.md)); the API refuses to start without it. Generate with `python3 -c "import secrets; print(secrets.token_hex(32))"` |

## LLM and embeddings

Three ways to get an `LLMProvider`, checked in this order by
`packages/providers/registry.py`'s `build_llm_provider` — see
[Components → Providers](components/providers.md) for the full precedence logic:

| Variable | Effect |
|---|---|
| `ANTHROPIC_API_KEY` | default backend: Anthropic direct |
| `VOYAGE_API_KEY` | default embeddings backend |
| `VOYAGE_REQUESTS_PER_MINUTE`, `VOYAGE_MAX_BATCH_SIZE`, `VOYAGE_MAX_TOKENS_PER_BATCH` | override Voyage's conservative free-tier pacing once a payment method is on file |
| `OPENROUTER_API_KEY` | routes reasoner/worker calls through OpenRouter instead |
| `OPENROUTER_REASONER_MODEL`, `OPENROUTER_WORKER_MODEL` | required alongside the key above — exact model slugs are never guessed; either may be a comma-separated fallback chain |
| `OPENROUTER_EMBEDDING_MODEL` | opts embeddings into OpenRouter too (not triggered by `OPENROUTER_API_KEY` alone) |
| `LLM_BASE_URL` | **BYO-LLM** — takes precedence over everything above; points the OpenAI-compatible client at your own endpoint (self-hosted vLLM, Azure/Bedrock gateway) |
| `LLM_REASONER_MODEL`, `LLM_WORKER_MODEL` | required alongside `LLM_BASE_URL` |
| `LLM_API_KEY` | optional — unauthenticated internal endpoints send no `Authorization` header at all if unset, rather than a dangling credential |
| `LLM_EMBEDDING_MODEL`, `LLM_EMBEDDING_BASE_URL` | BYO-LLM for embeddings; falls back to `LLM_BASE_URL` if the base URL isn't given separately |

## Search providers

| Variable | Effect |
|---|---|
| — | arXiv and Semantic Scholar work with no key; Semantic Scholar's anonymous tier is aggressively rate-limited, a free key gets a dedicated 1 RPS |
| `SEMANTIC_SCHOLAR_API_KEY` | recommended — removes anonymous-tier throttling |
| `PATENTSVIEW_API_KEY` | adds PatentsView to the fan-out |
| `EPO_OPS_CONSUMER_KEY`, `EPO_OPS_CONSUMER_SECRET` | adds EPO OPS |
| `ENABLE_GOOGLE_PATENTS_SEARCH` | opt-in — unofficial, unverified endpoint |
| `LOCAL_CORPUS_SEARCH` | adds `LocalCorpusSearchProvider` (Postgres FTS over `corpus_papers`) to the fan-out; requires the corpus to be populated via `arxiv_sync` first — see [Components → Providers](components/providers.md) |

## Email

| Variable | Effect |
|---|---|
| — | unset `SMTP_HOST` = emails only logged (`LoggingEmailSender`), not sent — every flow works without real delivery |
| `SMTP_HOST`, `EMAIL_FROM` | required together to enable `SmtpEmailSender` |
| `SMTP_PORT` | default 587 |
| `SMTP_USERNAME`, `SMTP_PASSWORD` | optional — an internal relay may need no auth |
| `SMTP_STARTTLS` | set to `0` to disable STARTTLS (internal port-25 relays) |
| `SMTP_TLS` | set to `1` for implicit TLS (port-465 style) |

## SSO (OIDC)

Per-org configuration lives in the `org_sso_config` table, not env vars ([ADR 0012](../adr/0012-oidc-sso.md)) — an org admin sets it via `PUT /org/sso-config`. This is the one process-wide setting SSO needs:

| Variable | Effect |
|---|---|
| `PUBLIC_APP_BASE_URL` | required if any org has SSO configured — the absolute, single-origin base URL this deployment is reachable at, used to build the OIDC `redirect_uri` and the post-login redirect back into the app |

## Observability

| Variable | Effect |
|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | unset = telemetry is a complete no-op; set = real metrics export over OTLP/HTTP every 15s. See [Workflows → Observability](workflows/observability.md) |

## Standalone image only

| Variable | Effect |
|---|---|
| `RUN_WORKER_IN_PROCESS` | runs the worker loop as a background task inside the API process instead of a separate `services/worker` process |
| `WEB_DIST_DIR`, `PGDATA`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | set by the Dockerfile itself; see [Deployment → Standalone](deployment.md#standalone) |

## Other

| Variable | Effect |
|---|---|
| `MIGRATIONS_DIR` | override where `run_migrations` looks for `.sql` files (defaults to `deploy/migrations`) |
