# Chat Harness Design

Oct 30, 2026

## Overview

The harness is a chat loop wrapped in a pipeline of pluggable stages: every user message passes through input stages, the model runs a tool loop watched by counselors, and the draft reply passes a turn-end validator before the user sees it. It is a learning project, so every stage is observable, swappable and logged.

**Goals**

- See exactly what a modern chat interface sends to the model each turn (system prompt, history, reminders, tools).
- Make the three stages you asked about first-class: input validation/transformation, mid-turn counselors, turn-end validation.
- Support at least two backends behind one interface: a local OpenAI-compatible server (LM Studio) and a hosted API (Anthropic).
- Record every turn as a trace you can replay and diff.
- Be buildable in small, independently testable features by Claude Code.

**Non-goals (for v1)**

- Multi-user auth, billing, or hosting for others.
- Fine-tuning or training anything.
- A polished consumer UI; a Ratatui TUI is enough.

## Prior art: yes, all three exist

All three ideas are established patterns, usually called guardrails, hooks, rails or classifiers. Production chat products typically run several of them per turn, mostly invisible to the user.

| Your idea | Common names | Where you'll see it |
| --- | --- | --- |
| Input validation & transformation | Input guardrails, input rails, prompt classifiers, query rewriting | OpenAI Agents SDK input guardrails (with "tripwires" that halt the run); NVIDIA NeMo Guardrails input rails; Llama Guard / safety classifiers; Claude Code `UserPromptSubmit` hooks; RAG query rewriters |
| Mid-turn counselors | Execution/tool rails, pre/post-tool hooks, critics, injected system reminders | Claude Code `PreToolUse` / `PostToolUse` hooks; NeMo execution rails; LangGraph interrupts and reflection nodes; chat apps injecting reminder text into long conversations |
| Turn-end validation | Output guardrails, output rails, stop hooks, validators with re-ask, self-critique | Claude Code `Stop` hooks (can block the turn and send the model back to work); OpenAI Agents SDK output guardrails; Guardrails AI validators with automatic re-ask; Self-Refine / Reflexion research |

Two things worth knowing going in:

- **Counselors come in two flavors.** Deterministic checks (regex, schema, permission rules) are cheap and run every time. Model-based checks (a small "judge" model) catch fuzzier problems but add latency and cost, so they usually run selectively.
- **Turn-end validation interacts with streaming.** If you stream tokens to the user, a validator can only flag or retract after the fact. Many products buffer, or validate in chunks, or only validate tool-heavy turns.

Names above are from general knowledge as of this writing; check each project's current docs before relying on specific APIs.

## Architecture: the turn pipeline

One user turn flows through input stages, a context assembler, a model/tool loop watched by counselors, and a turn-end check that can send the model back for another pass.

&#91;embedded content: turn pipeline · 3 stages, 1 loop\]

Only the three highlighted stages are new ideas; the rest is the standard chat loop every interface already has. The tracer (not drawn) records each box's input, output and verdict.

## Stage 1: Input validation & transformation

Input stages run in a configured order on the raw user message and return a verdict; the first `Block` stops the turn before any model call. Validators and transformers share one interface, so a stage can do both.

**Shared stage interface (used by all three stages)**

```
trait Stage {
    fn name(&self) -> &str;
    fn kind(&self) -> StageKind;          // Deterministic | Model
    async fn run(&self, ctx: &mut TurnContext) -> Verdict;
}

enum Verdict {
    Pass,
    Transform { note: String },          // stage edited ctx in place
    Annotate { text: String },           // hidden note for the model
    Warn { reason: String },             // shown to user, turn continues
    Block { reason: String },            // stop, explain to user
    Retry { feedback: String },          // turn-end only: send model back
}
```

**Starter validators**

- Empty / too long (token estimate against the model's context budget).
- Attachment type and size limits.
- Secret detection (API keys, connection strings) with a choice to redact or block.
- Prompt-injection scan on pasted or attached content, flagging text that tries to issue instructions.

**Starter transformers**

- Normalize whitespace and encoding.
- Expand slash commands (`/summarize`, `/sql`) into full instructions.
- Inline attached file contents with clear delimiters.
- Inject context: current date/time, user profile, relevant notes.
- Optional query rewrite for retrieval (model-based).

**Rules**

- The transcript stores both the original message and what was actually sent, so you can see every transformation.
- Model-based input stages use a small, fast model and a hard timeout; on timeout they `Pass` and log, rather than stall the chat.

## Stage 2: Mid-turn counselors

Counselors subscribe to events inside the model/tool loop and can allow, deny, inject a note, ask the user, or abort. They never edit the model's output directly; they influence it through what goes into the next model call.

**Hook points**

| Event | Fires when | Typical uses |
| --- | --- | --- |
| `BeforeModelCall` | Each iteration, just before the request is sent | Inject reminders, trim context, enforce budget |
| `BeforeToolCall` | Model has requested a tool | Permission checks, argument validation, ask user |
| `AfterToolResult` | Tool returned | Scan results for injected instructions, truncate huge output |
| `AfterModelResponse` | Model step completed | Loop detection, drift check |
| `EveryNSteps` | Every N loop iterations | Model-judged progress check on long agentic turns |

**Counselor actions**

- `Allow` — no change.
- `Deny { reason }` — tool call is not run; the reason is returned to the model as the tool result so it can adapt.
- `Inject { text }` — a hidden note is appended to the next model request (the "system reminder" pattern).
- `AskUser { question }` — pause the loop for a yes/no or choice from the user.
- `Abort { reason }` — end the turn and show the reason.

**Starter counselors**

1. **Permission** — per-tool allow / ask / deny rules from config.
2. **Loop guard** — same tool with same arguments 3 times in a row triggers an `Inject`, then `Abort`.
3. **Budget** — caps tokens, tool calls and wall-clock time per turn.
4. **Untrusted-content** — marks tool results as data and warns the model if they contain instructions.
5. **Drift judge** (model-based, opt-in) — every N steps, asks a small model "is this still working toward the user's request?" and injects a nudge if not.

All counselor activity, including notes the user never sees, appears in the trace view.

## Stage 3: Turn-end validation

When the model stops calling tools and produces a final reply, validators review the draft against the original request and the turn's tool results; a `Retry` sends the model back with feedback, up to a configured limit (default 2).

**Inputs each validator sees**: the original user message, the transformed message, the draft reply, and the turn's tool calls and results.

**Outcomes**

- `Pass` — deliver the reply.
- `Warn` — deliver with a visible flag (e.g. "unverified claim").
- `Retry { feedback }` — feedback is added as a hidden message; the loop resumes. Counts against the retry limit.
- `Block` — replace the reply with a fallback message and the reason.
- Retry limit reached — deliver the last draft with a `Warn`, never loop forever.

**Starter validators**

1. **Format** — if the user or a slash command asked for JSON, a table or code, check it parses.
2. **Completeness judge** (model-based) — did the reply address every part of the request?
3. **Grounding** — numbers and file names in the reply should appear in tool results; flag ones that don't.
4. **Promise check** — catch replies that say "I'll now do X" but end the turn without doing it (a classic agent failure).
5. **Length / policy** — simple caps and banned-content rules.

**Streaming modes** (configurable per session)

- **Buffered** — show a "checking" indicator, reveal only validated text. Simplest and most honest; feels slower.
- **Stream then verify** — stream live, then validate; on failure, mark the reply as superseded and stream the retry below it.
- **Off** — no turn-end stage, for comparing behavior with and without it.

## Core services

Five services sit under the pipeline; the stages only talk to them through `TurnContext`, which keeps stages testable with fakes.

| Service | Responsibility | Key decisions |
| --- | --- | --- |
| Conversation store | Sessions, messages, tool calls, stage verdicts | SQLite; append-only message table; store original and sent text |
| Context assembler | Builds each model request: system prompt, history, reminders, tool schemas | Token budgeting; oldest-first trimming, then summarization of dropped turns |
| Provider adapter | One `ChatProvider` trait over different backends | Normalize to a common message format; streaming events; tool-call parsing |
| Tool registry | Tool definitions, JSON-schema args, execution | Start with 3 tools: read file, web fetch (or SearXNG search), run SQL against a sandbox DB |
| Tracer | Structured event log for every stage, model call and tool call | JSON lines per turn; a trace viewer shows the exact request sent each step |

**Core types**

- `Session { id, title, config, created_at }`
- `Message { id, session_id, role, content_original, content_sent, hidden: bool }` — `hidden` marks injected notes and validator feedback.
- `TurnContext { session, user_msg, history, pending_notes, tool_calls, draft, budget }`
- `TraceEvent { turn_id, stage, event, verdict, latency_ms, tokens }`

**Config**: one TOML file per session profile listing provider, model, enabled stages in order, counselor hook bindings, retry limit and streaming mode. Swapping profiles is how you run A/B comparisons.

## Configuration

Code defines what each stage and classifier can do; a TOML profile decides which ones run, in what order, and with what settings. Real branching logic stays in code or a script hook, never in config.

**Three levels of configurability**

1. **Built-in, parameterized** — a Rust type registered by name; TOML picks it and sets its knobs.
2. **Prompt-defined judges** — a model-based check is a prompt, a model and an output format, so it lives entirely in TOML.
3. **Script hooks** — the runner executes a command, sends `TurnContext` as JSON on stdin, and reads a verdict (or labels) as JSON on stdout. Hooks are small Rust binaries in the workspace's hooks crate, so a new check never touches the harness core.

**Classifiers are a separate layer**

Classifiers label the input once per turn; stages read those labels instead of classifying for themselves. The relationship is many-to-many: one validator can depend on several classifiers, and one classifier can feed many stages.

- Each classifier is declared once with a name, a kind (`rules`, `judge`, `script`) and its label set.
- Stages list the classifiers they `need` and can be gated with `when`, a simple label match (`"intent:sql"`), not an expression language.
- The runner computes only the classifiers that enabled stages need, lazily, and caches results on `TurnContext.labels` for the rest of the turn.
- Counselors can re-run a classifier on tool results (for example, injection detection on fetched web pages).
- Every label, its score and its latency appear in the trace.

**Example profile**

```toml
[profile]
provider = "lmstudio"
model = "qwen3-14b"

# --- Classifiers ---
[[classifier]]
name = "intent"
kind = "judge"
model = "qwen3-4b"
labels = ["question", "sql", "code", "chitchat"]
prompt = "Classify the user's message into exactly one label."

[[classifier]]
name = "secrets"
kind = "rules"
patterns = { api_key = "sk-[A-Za-z0-9]{20,}", conn_string = "Server=.*;Password=" }

[[classifier]]
name = "injection"
kind = "script"
command = "bin/hook-injection"
labels = ["clean", "suspicious"]

# --- Stage 1: input ---
[[input]]
stage = "max_length"
max_tokens = 8000

[[input]]
stage = "secret_scan"
needs = ["secrets"]
on_match = "redact"            # or "block"

[[input]]
stage = "sql_context"           # inject schema notes
needs = ["intent"]
when = "intent:sql"

# --- Stage 2: counselors ---
[[counselor]]
stage = "permission"
hooks = ["before_tool_call"]
rules = { read_file = "allow", run_sql = "ask", web_fetch = "deny" }

[[counselor]]
stage = "untrusted_content"
hooks = ["after_tool_result"]
needs = ["injection"]

[[counselor]]
stage = "loop_guard"
hooks = ["after_model_response"]
repeat_limit = 3

# --- Stage 3: turn-end ---
[[turn_end]]
stage = "judge"
model = "qwen3-4b"
prompt = "Did the reply answer every part of the request? Return pass or fail and a reason."
timeout_ms = 3000

[[turn_end]]
stage = "script"
command = "bin/hook-grounding"

[turn_end_policy]
max_retries = 2
streaming = "buffered"
```

The loader validates the profile at startup: unknown stage names, a `needs` that names no classifier, or a `when` label outside that classifier's label set are errors, not runtime surprises.

## Worked example: calculator-guided arithmetic

For "what is 2+2?", all three stages cooperate: a classifier spots arithmetic, an input stage steers the model to a calculator tool, a counselor blocks wrong tools, and the turn-end check verifies the number independently. Modern models get 2+2 right unaided; the example is chosen to be easy to trace, and the same profile pays off on prompts like `48213 × 9177`.

&#91;embedded content: calculator flow · 4 harness checks, 2 loops\]

Highlighted boxes are harness stages; a denied tool or a failed check sends the model back with a reason instead of ending the turn.

**Profile fragment**

```toml
[[classifier]]
name = "math"
kind = "rules"                  # regex first: cheap, deterministic
labels = ["arithmetic", "none"]
extract = "expr"                # pulls "2+2" into labels.math.expr

[[input]]
stage = "tool_hint"
needs = ["math"]
when = "math:arithmetic"
note = "Use the calculator tool for all arithmetic. Do not compute mentally."
force_tool = "calculator"       # optional: provider tool_choice

[[counselor]]
stage = "tool_router"
hooks = ["before_tool_call"]
needs = ["math"]
when = "math:arithmetic"
deny = ["run_sql", "web_search"]
deny_reason = "Use the calculator tool for arithmetic."

[[turn_end]]
stage = "computed_answer"
needs = ["math"]
when = "math:arithmetic"
require_tool = "calculator"
cross_check = "expr"            # harness evaluates labels.math.expr itself
```

**Design points**

- **Hint vs force.** A hidden note nudges; `force_tool` sets the provider's `tool_choice`. Forcing is reliable but blunt; hinting is the better test of whether the counselor and validator catch mistakes.
- **Three numbers must agree.** The validator checks that the calculator was called, that the reply's number matches the tool result, and that both match the harness's own evaluation of the extracted expression. This catches a model that calls `calc(2+3)`.
- **Safe calculator.** A pure expression evaluator (e.g. the `evalexpr` crate), never code execution: deterministic and needs no sandbox.
- **Classifier fallback.** Regex handles "what is 2+2"; word problems ("half of 340 plus tax") need a `judge` classifier behind the rules one.
- **Trace to look for.** One run should show the `math` label, the injected note, any denied tool call with its reason, the calculator call, and the validator's three-way comparison.

## Tech stack

Decision: Rust core with a Ratatui TUI, all in one Cargo workspace, reusing what you already know from the orchestration project but keeping this harness separate and much smaller.

| Layer | Choice | Why |
| --- | --- | --- |
| Runtime | Rust, `tokio` | Familiar; traits map cleanly to stages and providers |
| Workspace | One Cargo workspace: `harness` (core + TUI) and `hooks` (script-hook binaries) | One `cargo build` produces everything; single binary to run |
| Provider streaming | `reqwest` client with SSE stream parsing | Providers stream tokens over SSE; no server needed for a TUI |
| Storage | SQLite via `sqlx` | Zero-ops, easy to inspect |
| Config | TOML via `serde` | Session profiles |
| UI | Ratatui TUI with a chat pane and a trace pane | Pure Rust, no JS toolchain; the trace pane is the learning tool |
| Backends | LM Studio (OpenAI-compatible) first, Anthropic Messages API second | Local is free to iterate on; the hosted API shows real tool-use behavior |

Everything stays in Rust, including script hooks and the calculator tool. If you later want a browser front end without leaving Rust, Leptos or Dioxus compile to WASM and could sit beside the TUI.

## Production harness behaviors

Eight behaviors that real chat harnesses depend on are added as Milestone 6; compaction, caching and interruption give the most insight per hour, so build those first. MCP is deliberately left out: it is a transport for tools, and the tool registry already covers what it would teach.

| Behavior | What the harness does | What it teaches |
| --- | --- | --- |
| Parallel tool calls | Runs several tool calls from one model step concurrently; returns results in request order | Concurrency, per-call counselor verdicts, partial failure (one denied, two succeed) |
| Context compaction | Near a token threshold, clears stale tool results first, then summarizes the oldest turns into one hidden message | What to keep vs drop; why long sessions degrade differently across products |
| Prompt caching | Keeps system prompt and tool schemas as a fixed prefix, appends volatile notes last, marks cache breakpoints (Anthropic) | Why request order matters; cost and latency effects shown as cache hits in the trace |
| Progressive disclosure | Sends only core tool schemas; a `tool_search` tool loads deferred schemas, and skill files load when relevant | Trading a lookup step for a smaller, cheaper context |
| Interruption and steering | User can stop a stream, cancel a running tool, or queue a message mid-turn | What a turn is when it can be cut in half; which state survives a cancel |
| Reasoning management | Sets thinking budgets; chooses whether thinking blocks are kept across tool calls and how they are shown | How reasoning interacts with context assembly and turn-end validation |
| Error recovery | Retries rate-limit and overload errors with backoff, falls back to another model, repairs or rejects malformed tool calls | Where much of a harness's reliability actually lives |
| Conversation branching | Editing a past message or regenerating a reply forks history into a tree | Data modeling: messages get a parent, sessions get an active branch |

**Profile additions**

```toml
[tools]
parallel = true
max_concurrent = 4
deferred = ["run_sql", "web_fetch"]   # loaded via tool_search

[context]
compact_at = 0.8                       # fraction of the context window
clear_tool_results_older_than = 6      # turns
summary_model = "qwen3-4b"

[cache]
enabled = true                         # provider-specific breakpoints

[reasoning]
budget_tokens = 4000
keep_across_tool_calls = true
display = "collapsed"                  # hidden | collapsed | full

[recovery]
max_retries = 3
backoff_ms = [500, 2000, 8000]
fallback_model = "qwen3-14b"
```

## Feature breakdown for Claude Code

Twenty-six features in six milestones; each is one Claude Code session with its own tests, and each milestone ends with something you can run.

| # | Feature | Depends on | Done when |
| --- | --- | --- | --- |
| **M1** | **Plain chat** |  |  |
| F1 | Cargo workspace skeleton (harness and hooks crates), config loading, `TurnContext` and `Verdict` types | — | `cargo test` passes; sample TOML profile parses |
| F2 | `ChatProvider` trait + LM Studio adapter with streaming | F1 | Unit test against a mock server; manual chat via CLI works |
| F3 | SQLite conversation store | F1 | Sessions and messages round-trip; original vs sent text both stored |
| F4 | Context assembler with token budget and trimming | F2, F3 | Test: long history trims oldest first, never drops the system prompt |
| F5 | Ratatui TUI: chat pane with streaming | F2–F4 | Multi-turn chat in the terminal |
| **M2** | **Observability + tools** |  |  |
| F6 | Tracer + trace pane showing exact request per step | F5 | Every model call visible as raw JSON in the UI |
| F7 | Tool registry + tool loop (read file, search, sandbox SQL) | F2, F6 | Model calls a tool, result returns, loop ends on final reply |
| F8 | Anthropic adapter | F2, F7 | Same conversation works on both backends via profile switch |
| **M3** | **The three stages** |  |  |
| F9 | Stage runner + profile validation (ordered stages, `needs` / `when`, verdict handling, timeouts) | F1, F6 | Fake stages prove `Block` stops, `Transform` edits, timeouts `Pass`; bad profiles fail at startup |
| F10 | Classifier registry: `rules` classifiers, lazy per-turn cache on `TurnContext.labels` | F9 | Two stages needing one classifier run it once; labels shown in trace |
| F11 | Input stages: length, secrets, slash commands, context injection | F10 | Each stage has a test; `when` gating skips correctly; transformed text visible in trace |
| F12 | Counselor hooks + permission, loop guard, budget, untrusted-content counselors | F7, F10 | Denied tool returns reason to model; repeated call aborts |
| F13 | Turn-end stage with retry loop + format and promise validators | F10 | Bad JSON triggers one retry; limit reached delivers with a warning |
| F14 | Script-hook protocol for stages and classifiers (JSON stdin/stdout, timeout, error handling) | F10 | A sample Rust hook binary works as a validator and as a classifier |
| **M4** | **Model-based judges** |  |  |
| F15 | Judge helper (small model, structured output) used by `judge` classifiers, drift counselor and completeness validator | F12, F13 | Judges are opt-in per profile; latency shown in trace |
| F16 | Streaming modes (buffered / stream-then-verify / off) + replay a turn under a different profile | F13, F15 | Same input replayed under two profiles, traces compared side by side |
| **M5** | **Worked example: calculator** |  |  |
| F17 | Calculator tool (`evalexpr`) + `math` rules classifier with expression extraction | F7, F10 | `calc("2+2")` returns 4; classifier labels and extracts `2+2`, ignores non-math text |
| F18 | `tool_hint` input stage (note + `force_tool`), `tool_router` counselor, `computed_answer` validator, end-to-end test | F8, F11–F13, F17 | A scripted fake model that calls `run_sql` is denied, a wrong calc triggers one retry, and the final reply passes the three-way check |
| **M6** | **Production harness behaviors** |  |  |
| F19 | Context compaction: clear stale tool results, then summarize oldest turns | F4, F6 | A session pushed past `compact_at` keeps working; trace shows what was cleared and the summary text |
| F20 | Prompt caching: fixed prefix order, Anthropic cache breakpoints, cache stats in trace | F4, F6, F8 | Second turn of a session reports cache hits; injected notes never break the prefix |
| F21 | Interruption and steering: stop stream, cancel tool, queue a mid-turn message | F5, F7 | Cancelled turn leaves a consistent transcript; queued message is answered next |
| F22 | Parallel tool calls: concurrent execution, ordered results, per-call counselors | F7, F12 | Three calls run concurrently; one denied, two succeed, model receives all three results in order |
| F23 | Error recovery: backoff on 429/overload, fallback model, malformed tool-call repair | F2, F8 | Fault-injecting mock provider: retries succeed, exhausted retries fall back, bad JSON args are rejected with a reason |
| F24 | Progressive disclosure: deferred tool schemas via `tool_search`, skill files loaded on demand | F4, F7 | Deferred tool is absent from the first request, found, then called; token savings shown in trace |
| F25 | Reasoning management: thinking budget, keep or strip across tool calls, display modes | F4, F8 | Each `keep_across_tool_calls` setting produces the expected request; UI shows hidden, collapsed and full |
| F26 | Conversation branching: message tree, edit and regenerate, branch switcher in UI | F3, F5 | Editing turn 2 of 5 creates a branch; both branches reload correctly |

**Feeding it to Claude Code**

- Put this doc in the repo as `DESIGN.md` and add a short `CLAUDE.md` with conventions (error handling, test style, no new crates without asking).
- Start each session with: "Implement F*n* from DESIGN.md. Write tests first. Stop when its Done-when passes."
- Review and commit between features so each one is a clean diff you can study.

## Open questions

- [ ] Front end: decided on a Ratatui TUI to keep the project all-Rust.
- [ ] Which small local model will run the judge stages, and is its latency acceptable?
- [ ] Should validator feedback on `Retry` be visible to the user in the chat, or only in the trace?
- [ ] Is this a standalone repo, or should the stage runner be reusable by the orchestration project later?
