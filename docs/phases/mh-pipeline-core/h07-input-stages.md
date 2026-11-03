# H7 — Input stages

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F11 · **Depends on:** [H6](h06-stage-runner-classifiers.md) · **Unblocks:** [H11](h11-calculator-cross-check.md), [F12](../m02-local-knowledge/f12-quick-depth-e2e.md)

[← Back to plan](../PLAN.md)

## Goal

Ship the first real stages: validators and transformers that run on input before any model call. In Searchy, the "input" is a user's question, a chat message, or a research step's task. These stages catch problems early, such as an oversized input, a pasted secret, or a slash command to expand. They also inject the context a step needs.

## Scope

**In scope**

- Built-in input stages, registered with H6:
  - `max_length`: estimate tokens against the model's context budget. Block or warn.
  - `attachment_limits`: type and size limits on uploaded material in chat.
  - `secret_scan`: needs a `secrets` rules classifier. Configure it to `redact` or `block`.
  - `normalize`: whitespace and Unicode normalization.
  - `slash_commands`: expand commands into full instructions from a TOML table.
  - `inject_context`: date and time, the user, the project, and the active domain profile name.
  - `tool_hint`: a hidden note, an optional `force_tool`, and a `when` gate. H11 uses it.
- Storing the original and sent text for every transformation (H3).
- Tests for each stage, including `when` gating.

**Out of scope**

- Injection scanning of pasted or fetched content:
  - The rules version is a small stage here.
  - The judge version → [H10](h10-judges.md).
  - Fetched-page scanning → [F20](../m04-polite-web/f20-injection-counselor.md).
- Model-based query rewriting for retrieval → [F9](../m02-local-knowledge/f09-passage-store.md) or [F12](../m02-local-knowledge/f12-quick-depth-e2e.md).
- Research scope stages (scope check, ledger lookup, clarifying question, plan checkpoint):
  - [F12](../m02-local-knowledge/f12-quick-depth-e2e.md).
  - [F13](../m03-ledger-verification/f13-claim-ledger.md).
  - [F24](../m06-async-research/f24-planner-checkpoint.md).

## Design notes

- Input stages run in a configured order on the raw message and return a verdict. The first `Block` stops the turn before any model call. Validators and transformers share one interface. (Harness: Stage 1)
- Starter validators: empty or too long, attachment limits, secret detection with redact or block, and a prompt-injection scan on pasted content. (Harness: Stage 1)
- Starter transformers: normalize whitespace and encoding, expand slash commands, inline attachments with clear delimiters, inject context, and an optional query rewrite. (Harness: Stage 1)
- The transcript stores the original message and what was sent. (Harness: Stage 1 › Rules)
- Searchy's input-stage uses are the scope check, ledger lookup, clarifying question and plan checkpoint. These are research stages built later on this framework. (Design: Research pipeline › How the chat harness stages map)

```toml
[[input]]
stage = "max_length"
max_tokens = 8000

[[input]]
stage = "secret_scan"
needs = ["secrets"]
on_match = "redact"            # or "block"

[[input]]
stage = "tool_hint"
needs = ["math"]
when = "math:arithmetic"
note = "Use the calculator tool for all arithmetic. Do not compute mentally."
force_tool = "calculator"      # optional: provider tool_choice
```

## Implementation sketch

Suggestions, not decisions:

1. Put one module per stage in `searchy-pipeline::stages::input`. Each parses its knobs from TOML and is registered by name.
2. `secret_scan` redacts to a placeholder like `[REDACTED:api_key]`, and records what kind was found but never the value in the trace.
3. `slash_commands` reads a `[slash]` table from the pipeline profile: `summarize = "Summarize the project's findings for …"`.
4. `tool_hint` sets `ctx.tool_choice` when `force_tool` is set. The provider adapter (H2) maps it.
5. Add a default chat pipeline profile and a default research-step profile that use these stages.

## Acceptance criteria

- [ ] Each stage has a test. `when` gating skips correctly, and the transformed text appears in the trace.
- [ ] A message containing an `sk-…` key is redacted before any model call, and the raw key never appears in `trace_events`.
- [ ] An over-length input is blocked, with a readable reason shown to the user.
- [ ] `/summarize` expands to its configured instruction, and the original text is still stored.
- [ ] `tool_hint` with `force_tool` produces a request whose `tool_choice` names that tool, in the mock provider's captured request.

## Open questions

- Secret scanning also matters for notes being ingested (F11). Those notes might be sent to cloud models. Should F11 reuse `secret_scan` at ingestion time? Proposed: yes, and flag the note rather than redact it.
