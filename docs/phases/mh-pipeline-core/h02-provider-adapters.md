# H2 — Provider adapters and streaming

**Milestone:** MH Pipeline core · **Status:** ⬜ Not started · **From harness:** F2, F8 · **Depends on:** [H1](h01-pipeline-core-types.md) · **Unblocks:** [H4](h04-context-assembler.md), [H5](h05-tool-loop.md), [H10](h10-judges.md), [H12](h12-error-recovery-reasoning.md), [F8](../m02-local-knowledge/f08-model-routing.md)

[← Back to plan](../PLAN.md)

## Goal

Put every model backend behind one `ChatProvider` trait. Each adapter must stream tokens and parse tool calls into a common format. Every model call Searchy makes goes through this trait: director, extractor, verifier, chat and the rest. F8's role routing then becomes a question of which provider and model to hand a call to.

## Scope

**In scope**

- The `ChatProvider` trait, with normalized message, tool-schema and tool-call types.
- Three adapters:
  - Anthropic Messages API.
  - OpenAI.
  - OpenAI-compatible: LM Studio, Ollama and llama.cpp, through a configurable `base_url`.
- Streaming over SSE into a common event stream: text deltas, tool-call deltas, usage and stop reason.
- Token usage reporting on every call, so budgets and per-role costs can be computed.
- An embeddings method, or a sibling `EmbeddingProvider` trait, for the `embed` role.
- A mock provider for tests that replays scripted responses.

**Out of scope**

- Role assignment, the instance/profile/depth/project/question layers and local-only enforcement → [F8](../m02-local-knowledge/f08-model-routing.md).
- Retries, backoff, fallback models and malformed tool-call repair → [H12](h12-error-recovery-reasoning.md).
- Cache breakpoints and request ordering → [H4](h04-context-assembler.md).
- The tool loop itself → [H5](h05-tool-loop.md).

## Design notes

- Provider adapter: one `ChatProvider` trait over different backends. It normalizes to a common message format, streams events and parses tool calls. (Harness: Core services)
- The harness supports a local OpenAI-compatible server and the Anthropic API behind one interface. (Harness: Goals)
- Searchy's backends:
  - Cloud APIs (Anthropic and OpenAI) work out of the box with an API key.
  - Local OpenAI-compatible endpoints (LM Studio, Ollama, llama.cpp) are optional.

  (Design: Deployment › First run)
- Model config shape. (Design: Model routing)

```toml
[models]
large = { provider = "anthropic", model = "<large-model-id>" }
small = { provider = "openai_compat", base_url = "http://lmstudio:1234/v1", model = "<small-model-id>" }
embed = { provider = "local", model = "<embedding-model-id>" }
```

- Embeddings come from a local model, through LM Studio or `fastembed-rs`. (Design: Knowledge store › Retrieval)
- Per-role costs feed the budget counselor. (Design: Model routing › Rules)
- Provider streaming uses a `reqwest` client with SSE parsing. (Harness: Tech stack)
- Only the Fetcher touches the open web. Model calls from workers go through the hub's **model gateway** (D4): workers sit on an internal Podman network with no internet route, and the gateway is the only service they can reach. The gateway holds the keys and runs the real adapters. (Design: Architecture; Model routing › Rules)

## Implementation sketch

Suggestions, not decisions:

1. Put `trait ChatProvider { async fn stream(&self, req: ChatRequest) -> Result<EventStream>; fn id(&self) -> ProviderId; }` in `searchy-pipeline`. Put the adapters in a `searchy-providers` crate, so that the pipeline core stays network-free.
2. `ChatRequest` carries `system`, `messages`, `tools`, `tool_choice`, `max_tokens`, an optional `reasoning`, which H12 fills in, and opaque `cache_hints`, which H4 fills in.
3. `StreamEvent` has these variants: `TextDelta`, `ToolCallStart { id, name }`, `ToolCallArgsDelta`, `ToolCallEnd`, `Usage { input, output, cache_read, cache_write }` and `Stop { reason }`. Add a helper that collects a stream into a full `ModelResponse`.
4. Keep pricing as a static table per model id, for cost estimates. Admins can override it later in F8.
5. Test each adapter against recorded SSE fixtures served by a local mock HTTP server.
6. Add a `GatewayProvider` that implements `ChatProvider` by forwarding the normalized `ChatRequest` to the hub's gateway endpoint and relaying its `StreamEvent`s (e.g. as SSE). Workers use only this provider. They hold a per-job token, never an API key. The gateway endpoint itself, with routing and auth, is [F8](../m02-local-knowledge/f08-model-routing.md).
7. Add a manual test: a `searchy dev chat` subcommand streams one reply to stdout.

## Acceptance criteria

- [ ] Each adapter passes a unit test against a mock SSE server. A manual one-shot chat works against at least one real cloud API and one local endpoint.
- [ ] The same tool-call fixture parses to identical normalized `ToolCall` values from the Anthropic, OpenAI and OpenAI-compatible adapters.
- [ ] Every call reports input and output token usage. Where the backend reports cache tokens, those come through too.
- [ ] The mock provider can script a sequence of replies, including tool calls, for use by H5–H11 tests.
- [ ] Embeddings return vectors of the configured dimension from a local endpoint.
- [ ] `GatewayProvider` against a mock gateway yields the same normalized events as calling the underlying adapter directly.

## Open questions

- Resolved (D4): workers reach models through the hub's model gateway over an internal network, using `GatewayProvider`.
- `fastembed-rs` versus an LM Studio embedding endpoint. Check crate maturity when this starts.
