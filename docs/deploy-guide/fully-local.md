# Fully local inference and prior-art search

This is the handoff plan for running the complete decision-support path inside one controlled
network: application, LLM, embeddings, patent and scholarly indexes, and database. It is a target
private-deployment profile, **not a claim that the current build is air-gapped**.

## What works today

The application already speaks OpenAI-compatible APIs. A local vLLM service can therefore replace
hosted generation and embeddings without changing pipeline code:

```dotenv
LLM_BASE_URL=http://local-llm:8000/v1
LLM_REASONER_MODEL=<served-model-name>
LLM_WORKER_MODEL=<served-model-name>
LLM_API_KEY=

LLM_EMBEDDING_BASE_URL=http://local-embeddings:8001/v1
LLM_EMBEDDING_MODEL=<served-embedding-model-name>
```

The arXiv corpus proof of concept can be synchronized into Postgres and searched with
`LOCAL_CORPUS_SEARCH=1`. This proves the provider shape, incremental ingestion, and local lexical
retrieval. It does not yet provide a patent-complete corpus.

## Current gaps before calling it local-only

- `build_search_providers()` always includes keyless live arXiv and includes live Semantic Scholar
  unless explicitly disabled. `LOCAL_CORPUS_SEARCH=1` adds a source; it does not replace the live
  fan-out. An explicit allowlist/profile such as `SEARCH_PROFILE=local-only` is required, with a
  startup failure if no local corpus is available.
- Patent bulk ingestion is not implemented. Add PatentsView applications, grants, claims, long
  descriptions, citations, families and legal-status data before judging patent retrieval.
- Postgres FTS is only the first retrieval stage. Measure lexical, vector, and hybrid recall@K on
  reviewed patent queries before selecting an index architecture.
- The release bundle needs pinned model weights, tokenizer/config, corpus snapshot, migrations,
  licenses, checksums, storage sizing, incremental update media, and backup/restore instructions.
- Add an egress-denied integration test and content-free freshness/capacity metrics. An environment
  variable alone is not evidence of isolation.

Public access to a dataset does not necessarily grant commercial redistribution. Record the
license and provenance of every model and corpus component, and have counsel review the intended
distribution mode (download-at-install versus redistribution in our bundle).

## A practical one-machine experiment

Do not start with the largest model names in the current hosted matrix. Mixture-of-experts “active
parameter” counts describe compute per token, not how few weights must be stored; the full model
still has to be resident across accelerator and host memory. Begin with a permissively licensed
7B-32B instruction model quantized to 4 or 8 bits, served by vLLM on a Linux workstation with one
24-48 GB GPU. Treat these as experiment bounds, not production sizing: long context, KV cache,
concurrency, quantization format, and guided decoding all change memory use.

Keep prompts small by retrieving a compact top-K evidence set rather than asking the model to hold
the corpus in context. Start with one worker and low concurrency, record peak VRAM, tokens/second,
queue time, power, and error/retry rates, then increase concurrency. Use a separate small local
embedding service only when the retrieval evaluation needs semantic search; lexical retrieval is
a valid first baseline.

The experiment is successful only when the same eval command used for hosted candidates can point
at the local endpoint and pass the quality, stability, structured-output, citation, and sensitive-
data gates. Extend that harness with retrieval and end-to-end cases before making a product claim.

## Cost and break-even

Compare total cost, not API price against GPU purchase price:

```text
hosted/month = disclosures × hosted variable cost per disclosure

local/month = hardware amortization + power + storage + operations
              + local variable cost per disclosure

break-even disclosures/month = local fixed monthly cost
  / (hosted variable cost/disclosure - local variable cost/disclosure)
```

Use a 36-month hardware life as a starting assumption and sensitivity-test utilization, electricity
price, support time, replacement capacity, corpus storage/backup, and model-quality-driven human
review. Idle GPU capacity is still a cost. If the local model causes more retries or weaker reports,
the additional reviewer time can overwhelm inference savings.

The first nine-case assessment smoke run spent less than half a cent per candidate on hosted model
calls. That makes it unlikely that a dedicated GPU beats hosted inference on **pure dollar cost at
pilot volume**. Local can still be the right choice for confidentiality, contractual no-egress,
predictable capacity, or a GPU that is already owned and highly utilized. It becomes financially
competitive as sustained workload rises, the hardware is shared across useful workloads, hosted
models are materially more expensive, and the smaller model preserves quality.

## Implementation and evaluation checklist

1. Add and test the fail-closed `local-only` search profile.
2. Package one quantized model behind vLLM and verify schema-constrained responses.
3. Run `assessment_v1`; record actual served model, peak VRAM, throughput, latency, power, retries,
   and operator time alongside token/cost results.
4. Implement PatentsView bulk ingestion, then benchmark retrieval recall before adding an arXiv or
   Semantic Scholar subset.
5. Run end-to-end reviewed cases against hosted and local stacks, blinded where possible.
6. Fill in the break-even formula with measured values and test low/base/high utilization.
7. Only then choose local, hosted, or a policy-controlled hybrid per pipeline role.

Start with [Components → Providers](../handbook/components/providers.md), the
[eval-harness handoff](../handbook/components/scoring-assess-evalharness.md), and the
[productization roadmap](../superpowers/specs/2026-07-12-patentability-triage-roadmap.md).
