# F32 — Answerability classifier and research handoff

**Milestone:** M8 Project chat and outputs · **Status:** ⬜ Not started · **Depends on:** [F24](../m06-async-research/f24-planner-checkpoint.md), [F31](f31-project-chat-loop.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

When the project can't answer a chat question, say so plainly and offer to research it. Chat never quietly searches the web. The handoff goes through the normal plan checkpoint. When the run finishes, the same chat thread picks up the new findings and answers.

## Scope

**In scope**

- A classifier that labels each chat question as answerable, partly answerable or out of scope for the project.
- A reply that says what is missing.
- Two offered actions: research it as a new related question in this project, or extend the original question's run.
- The `start_research` tool, which proposes a run and never starts one without approval.
- Linking the run back to the thread, so the thread answers once the run finishes.
- Passing the origin thread to the planner, so its brief ([H13](../mh-pipeline-core/h13-compaction-tool-disclosure.md)) is always included in the run's context ([F24](../m06-async-research/f24-planner-checkpoint.md)). This follows PLAN.md decision D11.

**Out of scope**

- The planner and checkpoint themselves: [F24](../m06-async-research/f24-planner-checkpoint.md).
- Push notifications for plan approval and run completion: [F7](../m01-deployable-shell/f07-progress-notifications.md).
- Merging new claims into the ledger: [F13](../m03-ledger-verification/f13-claim-ledger.md).

## Design notes

- When the store can't answer, chat says so and offers a research run instead of quietly searching. (Design: Project chat)
- A classifier labels each question as answerable, partly answerable or out of scope for the project. (Design: Project chat › How it works)
- `start_research` proposes a run; it never starts one without your approval. (Design: Project chat › How it works)
- **When the project can't answer:** the reply says what is missing and offers two actions: research it as a new related question in this project, or extend the original question's run. Both go through the normal plan checkpoint. New claims merge into the same ledger, and chat picks up the new findings when the run finishes. (Design: Project chat › When the project can't answer)
- Classifier role: small model, with rules first where possible. Answerability is listed as one of its jobs. (Design: Model routing › Roles)
- Plan checkpoint: the run pauses in `awaiting_approval` and notifies you. (Design: Research pipeline step 2)
- Starting runs requires the editor role. (Design: Teams and permissions › Roles)
- Chat feeds research (PLAN.md D11): the thread's running brief, holding its important points, is injected into the planner and research agents. A handoff guarantees that the origin thread's brief is included.
- Budget caps at instance, user and project level apply. A run that would exceed any of them asks first. (Design: Teams and permissions › Budgets and keys)

## Implementation sketch

Suggestions, not decisions:

1. Run the classifier as an input stage before the chat loop. Give it the question plus a retrieval preview: top claims and their scores from `search_claims`.
2. Answerable: normal F31 flow. Partly answerable: answer what's supported, then list the missing parts with the handoff offer. Out of scope: no answer, just the gap and the offer.
3. Implement `start_research(kind: new_question | extend_run, text, parent_question_id?)`. It creates a question or run in a proposed state and posts a card in the thread with "Plan it" buttons. It doesn't call the planner until you tap.
4. Store `origin_thread_id` and `origin_message_id` on the run. F24 always includes the origin thread's brief in the planner's context. Before handing off, force a brief refresh so the brief covers the latest messages.
5. On run completion (job queue event from F6), post a system message to the origin thread and re-answer the original question with the updated store. Stream it if the thread is open; otherwise it shows on next load and in the Inbox.
6. Hide the offer buttons for viewers.

## Acceptance criteria

- [ ] Out-of-scope question offers a run; when it finishes, the same thread answers it
- [ ] The classifier's label is recorded on each message and visible in the trace.
- [ ] A partly answerable question gets a cited partial answer plus a list of what's missing.
- [ ] `start_research` never creates a running job without an explicit approval in the plan checkpoint.
- [ ] "Extend the original run" adds to that question's runs rather than creating a new question.
- [ ] A viewer sees the "what's missing" reply but no research buttons.
- [ ] A constraint stated earlier in the thread (for example, "only studies in adults") appears in the handed-off run's planner context, through the thread brief.

## Open questions

- Does "extend the original question's run" create a new run under the same question, or resume the old run with added sub-questions? The design doesn't specify.
- How should the re-answer behave if the thread has moved on with other messages by the time the run finishes?
- For quick depth, the plan checkpoint is off (Design: Depth presets). Does a chat handoff at quick depth still require approval? The design says "never starts one without your approval," so likely yes.
