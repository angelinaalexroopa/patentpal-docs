# 4. Triage score + routing, not legal opinions

## Status
Accepted

## Context
An invention-disclosure triage tool that produces something read as a legal opinion creates
unauthorized-practice-of-law exposure and false confidence. Humans (committee, later counsel)
must remain the decision-makers.

## Decision
The system outputs a calibrated score, per-dimension reasoning, and a routing recommendation
only. Humans decide. "Not legal advice" framing is structural, not a disclaimer bolted on later —
it appears on every LLM-facing surface (UI and API responses).

## Consequences
No component may present a finding as a legal conclusion. This shapes UI copy, API response
schemas, and eventually the legal shell of the productization roadmap (UPL line, MSA IP clause).
