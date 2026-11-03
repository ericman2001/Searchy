# F8 — Model registry and role routing

**Milestone:** M2 Local knowledge · **Status:** ⬜ Not started · **Depends on:** [F5](../m01-deployable-shell/f05-projects-data-model.md), [H2](../mh-pipeline-core/h02-provider-adapters.md), [H12](../mh-pipeline-core/h12-error-recovery-reasoning.md) · **Unblocks:** [F9](f09-passage-store.md), [F26](../m06-async-research/f26-datasets-analyst.md)

[← Back to plan](../PLAN.md)

## Goal
Every model call in Searchy goes through one router that maps a *role* to a configured model. The router runs inside the hub's **model gateway** (D4), which is also how no-network workers reach models. A large cloud model can then direct research while small, cheap or local models do the high-volume work. The router also enforces the privacy rule that local-only notes and datasets never reach a cloud model, and it records cost per role so you can see where the money goes.

## Scope
**In scope**
- Model registry: named endpoints for Anthropic, OpenAI and OpenAI-compatible servers (LM Studio, Ollama, llama.cpp), plus a local embedding model.
- Role assignments for all nine roles, with the layered override order.
- Verifier escalation (`first` / `escalate` / `when = "uncertain"`).
- Fallback chains on rate limits and outages, recording which model actually ran.
- The model gateway (D4): a hub endpoint on the internal model network that workers call instead of any provider. It holds the API keys, runs the router, and reaches LAN, localhost and cloud models. Workers never get keys or an internet route.
- Local-only enforcement at the gateway: reroute to a local model, or skip the step and flag it.
- Per-role cost tracking, written per call with user and project attribution.
- A "test-call" action per role in Settings.
- The model-backends part of the first-run flow (Design: Deployment › First run, step 2).

**Out of scope**
- Budget caps and the budget counselor → [F25](../m06-async-research/f25-fan-out-budget.md).
- Domain-profile and depth-preset override layers. This feature defines the layer slots, and they are filled by [F41](../m10-profiles-media/f41-profile-system.md) (profiles) and [F24](../m06-async-research/f24-planner-checkpoint.md) (question-level overrides in plan review).
- The "no self-grading" reviewer rule in practice → [F38](../m09-review/f38-review-job.md). The router only needs to expose model family.
- Embedding storage and retrieval → [F9](f09-passage-store.md).

## Design notes
**Roles** (Design: Model routing › Roles)

| Role | Does | Suggested default |
| --- | --- | --- |
| Director | Scopes questions, writes plans, decides when to stop | Large cloud model |
| Classifier | Profile selection, source type, claim kind, answerability | Small model, rules first where possible |
| Extractor | Passages → structured claims | Small or mid; highest volume |
| Summarizer | Condenses sources, names clusters | Small |
| Writer | Drafts reports and outputs | Large |
| Verifier | Checks each passage supports its claim | Small first, escalate unclear cases |
| Reviewer | Adversarial review and debate | Large, ideally a different family from the writer |
| Chat | Project chat | Mid or large |
| Embeddings | Vectors for retrieval and clustering | Local model |

**Assignment layers**, each overriding the one before: (1) instance defaults set by the admin, (2) domain profile, (3) depth preset, (4) project, (5) question in plan review.

**Config shape** (Design: Model routing)
```toml
[models]
large = { provider = "anthropic", model = "<large-model-id>" }
small = { provider = "openai_compat", base_url = "http://lmstudio:1234/v1", model = "<small-model-id>" }
embed = { provider = "local", model = "<embedding-model-id>" }

[roles]
director   = "large"
writer     = "large"
reviewer   = "large"
extractor  = "small"
summarizer = "small"
classifier = "small"
verifier   = { first = "small", escalate = "large", when = "uncertain" }
embeddings = "embed"
```

**Rules** (Design: Model routing › Rules)
- Model gateway (D4). Every model call goes through a gateway in the hub. Workers sit on an internal Podman network with no internet route, where the gateway is the only reachable service. The gateway holds the keys, enforces local-only routing and records per-role cost.
- Local-only data stays local. If a role would send a local-only document or dataset to a cloud model, the call is rerouted to a local model, or the step is skipped and flagged.
- Escalation, not blind trust. Small models miss subtle mismatches, so the verifier escalates anything it is unsure of.
- Fallback chains. Rate limits or outages move to the next model in the role's list. The trace and report record which model actually ran.
- No self-grading. The reviewer avoids the writer's model when another is configured.
- Per-role costs feed the budget counselor.

**Keys and spending** (Design: Teams and permissions › Budgets and keys): cloud API keys are set once by the admin. Spending is tracked per user and per project. Only admins set keys and endpoints (Design: Roles table).

**First run** (Design: Deployment): cloud APIs work out of the box with a key. Local endpoints are optional. Each role gets a sensible default you can change later.

## Implementation sketch
Suggestions only.
- Crate `searchy-models`. Build on the `ChatProvider` trait and the Anthropic and OpenAI-compatible adapters from [H2](../mh-pipeline-core/h02-provider-adapters.md). This feature adds the registry of endpoints, role assignments and an `embed` call. Retry, backoff and the fallback mechanism come from [H12](../mh-pipeline-core/h12-error-recovery-reasoning.md); F8 only configures the fallback chain for each role.
- `Router::resolve(role, ctx) -> Vec<ModelRef>`, where `ctx` carries the layers (instance, profile, depth, project, question) plus a `contains_local_only` flag on the payload.
- Mark every input chunk with a `local_only` bit from its source. The router checks the bit before dispatch. This makes enforcement structural, not a convention.
- Tables: `model_endpoints`, `role_assignments(scope, scope_id, role, spec_json)`, and `model_calls(id, role, model, user_id, project_id, run_id, tokens_in, tokens_out, cost, fell_back_from, escalated, at)`.
- Store API keys encrypted at rest, or in a file on the data volume readable only by the hub. Never render them back in full in the UI.
- Gateway endpoint: e.g. `POST /internal/model` (and `/internal/embed`) bound only on the internal network interface. Workers authenticate with a per-job token issued by the hub at launch, so calls are attributed to the job's user, project and run. The hub's own pipeline calls the router in-process.
- Settings page: endpoints, role table, and a "Test" button per role that sends a tiny prompt and shows latency, model and cost.

## Acceptance criteria
- [ ] Each role test-calls its model; a local-only note never reaches a cloud model in the trace
- [ ] A rate-limit (429) from the first model in a chain falls back to the next, and the call record names the model that ran.
- [ ] A verifier call that returns "uncertain" escalates to the configured escalation model, and the record shows the escalation.
- [ ] A project-level override beats the instance default for the same role.
- [ ] Each `model_calls` row carries role, user, project and cost, and a per-role cost sum is queryable.
- [ ] Non-admins cannot view or change keys or endpoints.
- [ ] A worker on the internal network gets a model reply through the gateway using its job token, while a direct request from the worker to a cloud API host fails (no route).
- [ ] A gateway request without a valid job token is rejected.

## Open questions
- How is "uncertain" signalled? Options include a structured verdict field, a confidence threshold, or both. F14 is the first heavy user, so the decision should be made with it in mind.
- Resolved (D4): no-network workers reach models through the hub's gateway over the internal network.
- If no local model is configured, a local-only step can't be rerouted, so it must be skipped and flagged. Settings should warn about this case.
- Pricing tables for cost estimates drift. Decide whether they live in config or are maintained in code.
- Model calls are recorded by the tracer from [H3](../mh-pipeline-core/h03-turn-store-tracer.md). Decide whether `model_calls` is a separate table or a view over trace events.
