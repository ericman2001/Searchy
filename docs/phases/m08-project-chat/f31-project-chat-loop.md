# F31 — Project chat loop

**Milestone:** M8 Project chat and outputs · **Status:** ⬜ Not started · **Depends on:** [F14](../m03-ledger-verification/f14-verification-stage.md), [H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md), [H14](../mh-pipeline-core/h14-interruption-branching-replay.md) · **Unblocks:** [F32](f32-answerability-handoff.md), [F33](f33-scoped-chat.md), [F34](f34-outputs.md)

[← Back to plan](../PLAN.md)

## Goal

Let you talk to a project's findings. Answers come from the project's own store, cite passages, and are checked by the same turn-end verifier as reports. No new research happens and the network is never touched. This is the Ask screen. It can start as soon as M3 is done.

## Scope

**In scope**

- A chat loop running in the hub, scoped to one project.
- Retrieval tools: `search_claims`, `get_passage`, `get_source`.
- Answers that cite passage ids, rendered as tappable citations like the Report page.
- The turn-end verifier from F14, applied to every answer.
- Token streaming over SSE into an htmx-swapped message list.
- Threads saved per project and searchable later.
- The Chat model role.
- Pin, edit or reject a claim from chat. These are the only ways chat changes the ledger.
- Compaction for long threads, and a running thread brief per thread, both through [H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md) (PLAN.md decision D11). The brief is shown at the top of the thread, collapsed, so you can see what research will be told about this conversation.

**Out of scope**

- The answerability classifier and `start_research` handoff: [F32](f32-answerability-handoff.md).
- Scoping to a report, claim or cluster, and the idea-map tools: [F33](f33-scoped-chat.md).
- `query_dataset`, `make_chart`, `make_table`, `save_output`: [F34](f34-outputs.md).
- Saving answers as derived notes: [F35](f35-output-staleness-derived.md).

## Design notes

- Once a project has findings, you can ask questions, challenge claims and generate outputs from the project's own store, with no new research. When the store can't answer, chat says so and offers a run instead of quietly searching. (Design: Project chat)
- **Ask vs. research run** (Design: Project chat › Two modes):

  |  | Ask | Research run |
  | --- | --- | --- |
  | Answers from | Claim ledger, passages, archived sources, your notes and datasets | Local corpora and web, then ledger |
  | Network | None | Fetcher only |
  | Time | Seconds, one model loop | Minutes to hours |
  | Plan checkpoint | No | Yes |
  | Changes the ledger | Only when you pin, edit or reject a claim | Adds and merges claims |

- "Project chat is the chat harness itself, pointed at one project. It runs in the hub with project-scoped tools." `search_claims`, `get_passage` and `get_source` use the same hybrid retrieval, limited to the project. (Design: Project chat › How it works)
- Harness stages carry over. Answers cite passage ids, and the turn-end verifier checks them exactly as it checks reports. (Design: Project chat › How it works)
- Chat role: mid or large model. (Design: Model routing › Roles)
- Local-only data is rerouted to a local model or skipped and flagged. (Design: Model routing › Rules)
- Chat streaming: SSE token stream into an htmx-swapped message list, the same transport as run progress. (Design: Tech stack)
- Chat threads are kept per project and are searchable later. (Design: Project chat › Outputs)
- Viewers can read chat threads; editors can chat. (Design: Teams and permissions › Roles)
- Chat is part of ongoing research (PLAN.md D11). Threads are compacted, and their important points are kept as a running brief (H13). The brief is injected into planner and research-agent context ([F24](../m06-async-research/f24-planner-checkpoint.md)). So constraints and preferences you state in chat shape later runs.

## Implementation sketch

Suggestions, not decisions:

1. Tables: `chat_threads (project_id, created_by, scope_kind, scope_id, title)` and `chat_messages (thread_id, role, content, citations_json, model_used, verifier_result, created_at)`. Include `scope_*` now so F33 doesn't need a migration.
2. Use the tool loop from [H5](../mh-pipeline-core/h05-tool-loop.md) over the routing layer from F8 (Chat role). Register the three retrieval tools in H5's tool registry; each calls F9's hybrid retrieval with a project filter. Context assembly and compaction come from [H4](../mh-pipeline-core/h04-context-assembler.md) and [H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md).
3. Register F14's verifier as a turn-end validator. The retry loop and the warning when the limit is reached come from [H9](../mh-pipeline-core/h09-turn-end-validation.md).
4. Stream tokens over SSE, using H9's streaming modes (buffered or stream-then-verify). Show a verifying state, then swap in the verified, citation-linked message. Stop, cancel and branching come from [H14](../mh-pipeline-core/h14-interruption-branching-replay.md).
5. Index messages in tantivy for thread search.
6. Thread brief: show H13's `thread_brief(thread_id)` as a collapsible panel at the top of the thread. Editors can correct a wrong point by telling the chat; there's no direct editing in v1.
7. Prove "no network": the hub process issues no outbound HTTP except to model endpoints, and the trace records every tool call. Assert this in a test.

## Acceptance criteria

- [ ] Question about a finished project is answered with passage citations and no network calls in the trace
- [ ] Retrieval tools return only passages and claims from the current project, and only personal sources shared to it.
- [ ] A planted unsupported sentence in an answer is caught by the turn-end verifier and removed or flagged.
- [ ] Tokens stream to a phone browser. Reloading mid-answer re-renders the thread from the database.
- [ ] Threads persist per project and can be found by keyword search.
- [ ] Pinning or rejecting a claim from chat updates the ledger and records who did it.
- [ ] A long thread compacts without losing citations, and its brief panel lists the constraints stated earlier in the thread.

## Open questions

- Resolved (D10): a verifier retry rewrites only the failing sentences of the final answer. It doesn't re-run the whole tool loop.
- "No network calls" must exclude calls to cloud model APIs. Define the trace check so it means "no Fetcher, no web."
