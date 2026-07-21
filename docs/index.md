# PatentPal — Customer Documentation

PatentPal is an invention-disclosure triage system: engineers submit disclosures, and PatentPal
produces an evidence-grounded **patentability triage report** — a calibrated score,
per-dimension legal reasoning (§101/§102/§103), cited prior art, and a routing recommendation.
A human review committee makes the final call. **Decision support, never a legal opinion.**

## Where to start

- **[Deploying PatentPal](deploy-guide/index.md)** — evaluating which deployment tier fits your
  organization (SaaS, BYO-cloud, or air-gapped)? Start here. Includes quickstarts for each tier.
- **[Configuration reference](handbook/configuration.md)** — full list of environment variables
  and what each one controls.
- **[Data flow & security](deploy-guide/data-flow-and-security.md)** — where data lives, what
  leaves your network, and what controls are available per tier.

## Deployment tiers at a glance

| Tier | Who hosts it | Data boundary |
|------|-------------|---------------|
| **SaaS** | PatentPal hosts, multi-tenant | PatentPal infrastructure + LLM/search subprocessors listed in DPA |
| **BYO-cloud** *(recommended start)* | Our software, your cloud account | Your cloud account; your LLM provider |
| **Air-gapped** | Fully on-prem, no outbound | Nowhere — no network egress required |

## Contact & support

To start a pilot or request a security questionnaire, contact us at
**[hello@patentpal.io](mailto:hello@patentpal.io)** (placeholder — update before publishing).
