# F28 — Claim graph edges and clustering

**Milestone:** M7 Idea clouds and export · **Status:** ⬜ Not started · **Depends on:** [F23](../m05-source-quality/f23-claim-confidence.md), [F24](../m06-async-research/f24-planner-checkpoint.md) · **Unblocks:** [F29](f29-idea-map-ui.md), [F33](../m08-project-chat/f33-scoped-chat.md), [F34](../m08-project-chat/f34-outputs.md), [F40](../m09-review/f40-verdict-card.md)

[← Back to plan](../PLAN.md)

## Goal

Turn the claim ledger into a graph. Claims, entities and notes become nodes, typed edges connect them, and claims are grouped into named clusters. This is the data layer behind the idea map, the contradiction and gap views, and the chat tools that read them. It ships no UI of its own beyond what is needed to test it.

## Scope

**In scope**

- Edge types: supports, contradicts, cites, same-entity, same-topic.
- Entity extraction for people, organizations, methods and datasets, linked to claims.
- Notes as graph nodes.
- Clustering claims by embedding (HDBSCAN), with a cluster name from a small model.
- Per-cluster stats: size (amount of evidence) and average confidence.
- Contradiction pairs query: claims joined by a contradicts edge, with their sources.
- Gap query: plan sub-questions with few or no supporting claims, and clusters with only one source.
- Recomputing clusters when the ledger changes.

**Out of scope**

- The map UI and the phone list view: [F29](f29-idea-map-ui.md).
- "Ask about this cluster" and the chat tools `list_contradictions`, `list_gaps`, `get_cluster`: [F33](../m08-project-chat/f33-scoped-chat.md).
- Contradicts edges added by accepted review verdicts: [F40](../m09-review/f40-verdict-card.md).
- Turning a gap into a new question: [F29](f29-idea-map-ui.md).

## Design notes

- The idea map is a view over the claim ledger, not a separate store. Claims and entities are nodes, and the edges between them are where the insight is. (Design: Idea clouds)
- **Nodes:** claims, entities (people, organizations, methods, datasets), and your notes. (Design: Idea clouds)
- **Edges:** supports, contradicts, cites, same-entity, and "same topic" from embedding similarity. (Design: Idea clouds)
- **Clusters:** claims are embedded and clustered with HDBSCAN. A small model names each cluster. Size shows how much evidence exists; color shows average confidence. (Design: Idea clouds)
- **Contradiction view:** pairs of claims with a contradicts edge, side by side with their sources. "This is often the most valuable screen." (Design: Idea clouds)
- **Gap view:** sub-questions from the plan with few or no supporting claims, and clusters with only one source. (Design: Idea clouds)
- Cluster naming uses the Summarizer role ("condenses sources, names clusters"; small model). Embeddings use the local Embeddings role. (Design: Model routing › Roles)
- Independence comes from F23. "Only one source" in the gap view should count independent origins, not raw sources. (Design: Source quality › Claim confidence)
- Embeddings are local, so private notes never leave the machine. Local-only notes must never be sent to a cloud model for naming. (Design: Knowledge store › Retrieval; Model routing › Rules)
- Vector storage is `sqlite-vec`. (Design: Tech stack)

## Implementation sketch

Suggestions, not decisions:

1. Add tables: `entities`, `claim_entities`, `graph_edges (src_kind, src_id, dst_kind, dst_id, edge_type, weight, created_by_run, created_at)`, `clusters (project_id, label, size, avg_confidence, computed_at)`, `cluster_members`.
2. Derive supports and contradicts edges from the ledger's supporting and contradicting passages (F13, F23). Derive cites from citation metadata where available.
3. Same-entity: extract entities during claim extraction (Extractor role) or in a follow-up pass. Normalize names with a simple key first.
4. Same-topic: k-nearest neighbors over claim embeddings in `sqlite-vec`, above a tunable similarity threshold.
5. Clustering: check Rust HDBSCAN crates (e.g. `hdbscan`, `petal-clustering`) for maturity when the feature starts. Running it as a job in the Analyst worker is an alternative.
6. Name clusters with the Summarizer role. Cache names by membership hash so stable clusters aren't renamed every run.
7. Recompute after each run's ledger merge, as a background job, and after ledger edits.
8. Expose query functions: `contradiction_pairs(project)`, `gaps(project)`, `cluster(project, id)`. F29 and F33 build on these.

## Acceptance criteria

- [ ] Clusters named; contradiction pairs listed
- [ ] A test project produces all five edge types, each traceable to the claims or passages that caused it.
- [ ] Each cluster has a name, a size and an average confidence.
- [ ] The gap query returns a planted sub-question with no supporting claims and a cluster backed by a single origin.
- [ ] Rejecting or editing a claim updates its edges and cluster membership on the next recompute.
- [ ] Trace shows that local-only notes never reach a cloud model during naming.

## Open questions

- What is the similarity threshold for same-topic edges, and is it per project or global?
- How should entity resolution here relate to product entity resolution in [F44](../m10-profiles-media/f44-products-profile.md)? It's probably best kept generic now and extended by profiles later.
- Gaps depend on plan sub-questions, which come from [F24](../m06-async-research/f24-planner-checkpoint.md). F24 isn't a listed dependency of F28. Quick-depth runs without a plan will only show single-source cluster gaps.
- Do notes join clusters, or appear only as nodes linked by edges?
