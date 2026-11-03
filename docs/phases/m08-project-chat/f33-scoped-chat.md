# F33 — Scoped chat and idea-map tools

**Milestone:** M8 Project chat and outputs · **Status:** ⬜ Not started · **Depends on:** [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md), [F31](f31-project-chat-loop.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Start a chat from a report, a single claim or an idea-map cluster, so retrieval only sees that subset. Also give the chat the idea-map views as tools, so it can list contradictions and gaps and read clusters. This is how "Ask about this cluster" works, and how you challenge one claim without the rest of the project leaking in.

## Scope

**In scope**

- Chat scopes: project (the F31 default), report, claim and cluster.
- Entry points: "Ask" from the Report page and from selected report text, from a claim in the ledger, and from a cluster in the idea map.
- Scope-aware filtering in `search_claims`, `get_passage` and `get_source`.
- Tools: `list_contradictions`, `list_gaps`, `get_cluster`.
- A visible scope label on the thread, with a way to widen it to the whole project.

**Out of scope**

- The base chat loop: [F31](f31-project-chat-loop.md).
- Graph data, clusters, contradictions and gaps: [F28](../m07-idea-clouds-export/f28-claim-graph-clustering.md).
- The Review action on selected text: [F38](../m09-review/f38-review-job.md).

## Design notes

- **Scoping:** start a chat from the project, a report, a single claim or an idea-map cluster, and retrieval is limited to that subset. (Design: Project chat › Scoping)
- Tools: `list_contradictions`, `list_gaps`, `get_cluster` are "the idea-map views as tools." (Design: Project chat › How it works)
- Ask screen: "Chat with the project's findings, scoped to the project, a report, a claim or a cluster." (Design: Web interface › Screens)
- Report screen: "select text to Ask or Review." (Design: Web interface › Screens)
- Idea map: "with 'Ask about this cluster'." (Design: Web interface › Screens; Idea clouds)
- Contradiction view: claim pairs with a contradicts edge, side by side with their sources. Gap view: sub-questions with few or no supporting claims, and clusters with only one source. (Design: Idea clouds)
- Answers cite passage ids, and the turn-end verifier checks them as it checks reports. This applies unchanged inside a scope. (Design: Project chat › How it works)

## Implementation sketch

Suggestions, not decisions:

1. Use the `scope_kind` and `scope_id` columns that F31 added to `chat_threads`.
2. Resolve each scope to a set of allowed claim ids and passage ids:
   - **Report:** the claims and passages the report cites.
   - **Claim:** the claim plus its supporting and contradicting passages.
   - **Cluster:** the cluster's member claims and their passages.
3. Pass that set as a filter into F9's hybrid retrieval. Filter before ranking, not after, so a scope doesn't return an empty top-k.
4. Wrap F28's query functions as the three tools. Inside a scope, filter their output to the scope too.
5. For "Ask" on selected text, store the selection as the thread's first context message and scope to the report.
6. Snapshot cluster membership when the thread starts. Later re-clustering then doesn't silently change what an old thread could see. Record the snapshot id.

## Acceptance criteria

- [ ] "Ask about this cluster" retrieves only that cluster's claims
- [ ] A claim-scoped thread cites only that claim's supporting or contradicting passages, verified by the trace.
- [ ] A report-scoped thread can't retrieve a passage the report doesn't cite.
- [ ] `list_contradictions` and `list_gaps` return the same items the idea-map UI shows for the same scope.
- [ ] Selecting report text and tapping Ask opens a thread with the selection as context.
- [ ] Widening a thread to project scope is explicit and shown in the thread.

## Open questions

- Should a report scope include the report's claims' other passages, or only the passages actually cited?
- Does a scoped thread's handoff ([F32](f32-answerability-handoff.md)) treat "not in this cluster" as out of scope, or first suggest widening to the project?
- Snapshotting cluster membership is an engineering suggestion, not in the design. Confirm it's wanted.
