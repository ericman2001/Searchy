# Target Environment (template)

Copy this file to `ENVIRONMENT.md` in the same folder and fill it in for your deployment. `ENVIRONMENT.md` is gitignored, because it describes your own network: hosts, addresses, accounts and running services.

The design docs stay generic. `ENVIRONMENT.md` records the specifics, so features like F3 (installer), F8 (model routing), F9 (embeddings) and F19 (SearXNG) can be tested against the real setup. Re-check it before relying on it, since services and models change.

[← Back to plan](PLAN.md)

## Hub host: `<hostname>` (`<hub-ip>`)

| Item | Value |
| --- | --- |
| OS | e.g. Fedora Server, x86_64 or arm64 |
| Hardware | Cores, RAM, free disk, GPU (if any) |
| Account | The dedicated `searchy` user; how you reach it (`ssh searchy@<hub-ip>`); lingering on or off; sudo access |
| Podman | Version, rootless, Quadlet available, cgroups v2, network backend; subuid/subgid set |
| `podman.socket` (user) | Enabled or disabled. The launcher needs it (F3, F6) |
| SELinux | Enforcing, permissive or absent. If enforcing, volumes and per-job mounts need labels (F3, F6) |
| Firewall | Whether the hub port is reachable from the LAN |
| Tailscale / TLS | Whether HTTPS is available (D3) |

**Ports already in use.** List them, so the installer's `--port` (F3) picks a free one. 8080 is often taken.

## Services Searchy can use

| Service | Address | Notes |
| --- | --- | --- |
| SearXNG | `http://<host>:<port>` (from containers on the hub: `host.containers.internal:<port>`) | Check that the JSON API is enabled (`/search?q=test&format=json`). Use as F19's external instance |
| Local inference (primary) | `http://<host>:<port>/v1` | LM Studio, Ollama or llama.cpp |
| Local inference (fallback) | `http://<host>:<port>/v1` | For when the primary is off or asleep |

### Models

List each model with the context length it is *configured* to load with, not its advertised maximum. On a machine with limited memory these differ a lot.

| Model | Kind | Configured context | Note |
| --- | --- | --- | --- |
| `<model-id>` | LLM / VLM / MoE / embeddings | | |

## Suggested starting role assignment (F8)

A starting point to tune. Keep the number of distinct local models small: servers like LM Studio load models on demand, swapping costs time, and every loaded model plus its KV cache shares the same memory. Two mixture-of-experts models from different families can do most of the work, with a dense model only where quality matters most.

```toml
[models]
fast   = { provider = "openai_compat", base_url = "http://<primary>/v1", model = "<fast-moe-model>" }
large  = { provider = "openai_compat", base_url = "http://<primary>/v1", model = "<dense-model>" }
critic = { provider = "openai_compat", base_url = "http://<primary>/v1", model = "<model-from-another-family>" }
embed  = { provider = "openai_compat", base_url = "http://<primary>/v1", model = "<embedding-model>" }
backup = { provider = "openai_compat", base_url = "http://<fallback>/v1", model = "<small-model>" }

[roles]
director   = ["large", "backup"]      # or a cloud model, if a key is set
writer     = ["large", "backup"]
reviewer   = ["critic", "large"]      # a different family from the writer: no self-grading
extractor  = ["fast", "backup"]
summarizer = ["fast", "backup"]
classifier = ["fast", "backup"]
chat       = ["fast", "backup"]
verifier   = { first = "fast", escalate = "large", when = "uncertain" }
embeddings = "embed"                  # no fallback; see below
```

## Lessons that apply to any local-model setup

- **Availability.** A workstation serving models may be asleep or off. Fallback chains (F8, H12) keep runs alive, slowly. The trace and report record which model actually ran.
- **Embeddings must not silently fall back.** Vectors from different embedding models, or even different quantizations of the same one, aren't comparable. If the embedding endpoint is down, queue embedding work instead of switching models. Changing the embedding model means re-embedding everything (F9).
- **Context windows are per deployment, not per model.** The model registry (F8) should store the context length each endpoint actually serves, and the context assembler (H4) and compaction (H13) should budget against that, never against the advertised maximum. Long sources go through chunked retrieval (F9) rather than whole-document prompts.
- **Ollama's default context.** Ollama uses its own default context length unless `num_ctx` (or `OLLAMA_CONTEXT_LENGTH`) is set, and that default can be far below the model's maximum. Context the router thinks it has, but doesn't, gets silently truncated.
- **Memory pressure.** Several chat models loaded at once, each with a large KV cache, may not fit in memory, and the server will evict models to make room. If runs stall on model loading, use fewer distinct models or lower the configured context lengths.
- **Model loading.** The first call to an unloaded model waits for it to load, which can take a while for 27B+ models. Router timeouts (H12) need to allow for that.
