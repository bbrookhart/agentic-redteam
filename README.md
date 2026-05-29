# agentic-redteam

An **authorized, canary-based agentic red-teamer** for LLM applications and agents, built on **LangGraph**. It autonomously probes a target for the OWASP **LLM Top 10 (2025)** and **Agentic / ASI Top 10 (2026)** vulnerability classes, *adapts its payloads when a defense holds*, and produces an OWASP-mapped report with attack-success-rate (ASR) metrics.

> **Why this design is credible (and not an abuse kit).** Success is detected with **benign canaries** — a planted secret token or the harmless word `BANANA` — never by eliciting genuinely harmful content. A hard **authorization gate** refuses to run against any real target you have not affirmed you are permitted to test. This is the same model used by mature tools like NVIDIA `garak`, Microsoft PyRIT, and `promptfoo`.

---

## What it does

For each technique it runs a **feedback loop**: send a payload → observe → judge → if the control held, **refine** and try again, until it succeeds or the budget is spent. That adaptivity is the difference between an *agent* and a static scanner — naive payloads get blocked, adapted ones get through.

```
START ─▶ plan ─▶ execute ─▶ judge ──success / exhausted──▶ plan ─ … ─▶ END
                   ▲                    │
                   │                 (control held)
                   └─────── refine ◀────┘
```

```mermaid
flowchart LR
    START([start]) --> plan
    plan -->|next technique| execute
    plan -->|queue empty| END([report])
    execute -->|budget hit| END
    execute --> judge
    judge -->|exploited / exhausted| plan
    judge -->|control held| refine
    refine --> execute
```

## Quickstart

```bash
pip install -e .                 # core (LangGraph only) — runs offline
pip install -e ".[llm,dev]"      # + adaptive mutation, real targets, tests

# Offline demo against the built-in vulnerable sandbox — no API key needed:
python -m redteam.cli --target mock
```

Sample output:

```
ASR 33% | 3 technique(s) exploited | 9 calls | report -> report.md
```

The sandbox is deliberately vulnerable to 3 of 4 techniques and ships with naive
keyword defenses, so you can watch the adaptive loop bypass them on the 2nd attempt
while the encoded-injection control correctly **holds**.

### Adaptive mode (LLM-driven mutation)

```bash
REDTEAM_MODEL=anthropic:claude-sonnet-4-6 python -m redteam.cli --target mock
```

When the hand-written seeds are exhausted, the `refine` node asks the model for an
improved, still-benign payload using the technique's documented strategy.

### Testing a real target you are authorized to test

```bash
export REDTEAM_AUTHORIZED=1
export REDTEAM_SCOPE_NOTE="approved by sec-team, ticket SEC-1234, 2026-05-01"
export REDTEAM_MODEL=openai:gpt-4.1
python -m redteam.cli --target chat --target-model openai:gpt-4.1 \
    --system-prompt-file ./prompt.txt   # must contain the canary for leak detection
```

Without `REDTEAM_AUTHORIZED=1`, any non-sandbox target is **blocked**.

## OWASP coverage

| Technique | OWASP class | Detection (benign) |
|---|---|---|
| System-prompt / hidden-config leakage | LLM07 Prompt Leakage | planted canary token appears in output |
| Goal hijack via role-play framing | ASI01 Agent Goal Hijack | forbidden marker `BANANA` appears |
| Tool misuse via authority framing | ASI02 Tool Misuse | privileged tool returns the canary |
| Encoded-payload injection (control check) | LLM01 Prompt Injection | included to confirm a control *holds* |

## How it's built

| Layer | Choice | Rationale |
|---|---|---|
| Orchestration | **LangGraph** | explicit state + conditional edges map cleanly to a retry/refine loop; the production default in 2026 |
| Models | LangChain `init_chat_model` (`anthropic:` / `openai:`) | provider-agnostic; multi-model |
| Detection | **deterministic** canary/marker matching (+ optional LLM judge) | free, fast, and not itself foolable by a non-deterministic model |
| Targets | thin `Target.send(str) -> str` interface | same probes hit the sandbox, an OpenAI-compatible endpoint, Anthropic, or your own agent |
| Safety | authorization gate + benign canaries | authorized-testing model; proves the *vulnerability class* without producing harm |

## Talking points (for interviews)

- **Why canaries instead of harmful outputs?** You can prove an injection succeeded by leaking a planted secret; you never need the model to actually misbehave harmfully. Safer, and the detector is deterministic.
- **Why deterministic judging?** An LLM-as-judge adds cost, latency, and a second non-deterministic failure mode. Marker matching is O(1) and reliable; the LLM judge is a fallback for genuinely semantic cases.
- **What makes it agentic vs. a scanner?** The `refine` loop conditions the next payload on the target's last response — escalating weak→strong and adapting per defense — which is exactly why it finds what single-shot tools miss.
- **Metric that matters:** attack-success-rate per OWASP category, plus attempts-to-exploit, so a defender can see *which* controls held and *how hard* they were to break.

## Tests

```bash
pytest -q          # offline; asserts 3/4 techniques exploited, encoding control holds
```

## Roadmap

- Add indirect prompt injection (poisoned RAG/tool-output) and memory-poisoning (cross-session) probes.
- Persist runs and diff ASR across builds → wire into CI as a regression gate.
- Export findings as SARIF for security dashboards; emit OpenTelemetry traces.
- Pluggable target adapters for popular agent frameworks (LangGraph/CrewAI agents).

## Responsible use

Only run this against systems you own or are explicitly authorized to test.
Unauthorized testing of third-party systems is unethical and, in most
jurisdictions, illegal. The authorization gate is a guardrail, not a substitute
for written permission.
