# Data flow & security

Disclosures are pre-filing IP. Expect a security review before any pilot, and expect it to ask
specifically "what leaves our network, and to whom." This page answers that, per tier, and
explains the one gap that's easy to miss.

## What calls out, by category

| Category | Calls out to | Controlled by |
|---|---|---|
| **LLM (Normalize, Assess)** | Anthropic, or your own OpenRouter/self-hosted endpoint | `ANTHROPIC_API_KEY` vs. `LLM_BASE_URL` — see [Configuration reference](../handbook/configuration.md) |
| **Embeddings (Evidence map)** | Voyage, or your own endpoint | `VOYAGE_API_KEY` vs. `LLM_EMBEDDING_MODEL`/`LLM_EMBEDDING_BASE_URL` |
| **Prior-art search** | arXiv, Semantic Scholar, and whichever of PatentsView/EPO/Google Patents you've configured | Per-provider API keys; unconfigured providers are simply excluded from the fan-out |
| **Outbound email** | Whatever SMTP relay you point at (or nowhere, if `SMTP_HOST` is unset) | `SMTP_HOST`/`EMAIL_FROM` |

**BYO-LLM removes the LLM/embeddings egress** — point `LLM_BASE_URL` at your own endpoint and
those two rows go to infrastructure you control, or nowhere outside your network at all if that
endpoint is internal. This is what "your data never touches us" means concretely for the
BYO-cloud tier, and it's fully implemented today (see
[Components → Providers](../handbook/components/providers.md)).

## The gap that's easy to miss: query egress

BYO-LLM does **not** remove the prior-art search egress. Every search query in the fan-out is
built from a disclosure's own novel elements — the confidential, pre-filing substance of the
invention — and sent as a plaintext query string to whichever external search APIs are
configured, on every pipeline run. This is exactly the kind of thing a security review is
designed to catch, and BYO-cloud alone doesn't close it.

The mitigation is the **local prior-art corpus**: a Postgres full-text-search mirror
(`corpus_papers`) that joins the fan-out without leaving your network. It's opt-in
(`LOCAL_CORPUS_SEARCH=1`) and, as of today, covers arXiv — populated and kept current by an
OAI-PMH sync job you run on a schedule (daily cron is the intended pattern; upserts are
idempotent, so an overlapping sync window is safe):

```bash
uv run python -m patent_triage_providers.search.arxiv_sync --set cs --days 7
```

**Current state**: additive, not a replacement — it joins the external search APIs in the
fan-out rather than replacing them, so query egress is reduced, not eliminated, until the
external providers are explicitly turned off. Full replacement (zero query egress) is the bar
for the fully air-gapped tier; PatentsView bulk data is next on the roadmap since patents are the
core prior-art source for this use case, arXiv being a proof of concept. If zero query egress is a
hard requirement, the current registry needs one more engineering control before deployment:
keyless live arXiv is enabled by default, and anonymous Semantic Scholar is enabled unless
explicitly disabled. Leaving credentials unset is therefore **not** a local-only switch. The
roadmap tracks a fail-closed local-only profile and an egress-denied test; see
[Fully local deployment](fully-local.md).

## Trust surface, by tier

| Control | SaaS | BYO-cloud | Air-gapped |
|---|---|---|---|
| Data residency | Fixed to our infrastructure | Your cloud account, your region | Your network only |
| Encryption at rest/in transit | Yes | Yes (your infrastructure's) | Yes (your infrastructure's) |
| Audit log | Yes (`audit_log` table, every decision and job-state transition) | Yes | Yes |
| LLM egress | To our subprocessor list (in the DPA) | To your chosen provider/endpoint | None (self-hosted endpoint) |
| Search query egress | To external search APIs | Same, unless local corpus is enabled | None, once the corpus fully replaces external search |
| SOC 2 / pen test | Planned, follows first committed customer | Same | Same |

The output itself is structured, throughout the product, as **decision support, never a legal
opinion** — a human review committee makes the actual call
([ADR 0004](../adr/0004-triage-score-not-legal-opinion.md)). That framing matters for your own
internal sign-off as much as it does for UPL positioning: nothing in any tier is designed to make
an autonomous patentability determination.

## Next

- [Configuration reference](../handbook/configuration.md) for every environment variable
  mentioned above.
- [Components → Providers](../handbook/components/providers.md) for how the provider-swap
  mechanism is implemented, if your security review wants to read the actual code path rather
  than take this page's word for it.
