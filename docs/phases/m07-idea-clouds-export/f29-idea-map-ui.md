# F29 — Idea map UI

**Milestone:** M7 Idea clouds and export · **Status:** ⬜ Not started · **Depends on:** [F28](f28-claim-graph-clustering.md) · **Unblocks:** —

[← Back to plan](../PLAN.md)

## Goal

Show the claim graph from F28 as a screen you can explore. Desktop gets an interactive graph. Phones get a ranked list of clusters, contradictions and gaps, because graphs don't work at 380 pixels. Any gap can become a new question with one tap.

## Scope

**In scope**

- The Idea map screen in the Project area.
- Desktop: an interactive Cytoscape.js graph of claims, entities and notes, with typed edges and clusters.
- Phone: a ranked list in three sections: clusters, contradictions and gaps.
- Contradiction view: claim pairs side by side with their sources.
- Gap view, with a one-tap "turn this into a new question" action.
- Tapping a node or cluster opens the claim, its passages and sources.
- An "Ask about this cluster" button that links into chat once F33 exists.

**Out of scope**

- Graph data, clustering and naming: [F28](f28-claim-graph-clustering.md).
- Scoped chat behind "Ask about this cluster": [F33](../m08-project-chat/f33-scoped-chat.md).
- The plan checkpoint the new question goes through: [F24](../m06-async-research/f24-planner-checkpoint.md).

## Design notes

- Screen: "Idea map — The claim graph (desktop-first; a simplified cluster list on phones), with 'Ask about this cluster'." (Design: Web interface › Screens)
- On desktop the map is an interactive graph (Cytoscape.js). On a phone it collapses to a ranked list of clusters, contradictions and gaps. (Design: Idea clouds)
- Cluster size shows how much evidence exists; color shows average confidence. (Design: Idea clouds)
- Contradiction view: pairs of claims with a contradicts edge, side by side with their sources. "Often the most valuable screen." (Design: Idea clouds)
- Gap view: sub-questions with few or no supporting claims, and clusters with only one source. One tap turns a gap into a new question. (Design: Idea clouds)
- Your notes appear as nodes, so you see where your own thinking agrees or disagrees with the sources. (Design: Idea clouds)
- The UI is server-rendered HTML with htmx. There is no build step or JS framework. The only client-side library is the graph view. (Design: Web interface; Tech stack)
- Phone-first: one column, large tap targets, works on slow connections. (Design: Web interface)
- Viewers can read; editors start runs. Starting a question from a gap needs editor rights. (Design: Teams and permissions › Roles)
- Personal sources that aren't shared never appear to other members. (Design: Teams and permissions › Sharing and privacy)

## Implementation sketch

Suggestions, not decisions:

1. Add a JSON endpoint, e.g. `GET /projects/{id}/graph.json`, in Cytoscape's elements format. Filter by the viewer's permissions on the server.
2. Vendor a pinned Cytoscape.js build into the hub's static assets. Load it only on the desktop graph page.
3. Pick the layout by viewport: desktop shows the graph, narrow screens show the list. Server rendering the list keeps phones script-light.
4. Rank list items. Clusters by size × confidence, contradictions by the reliability of both sides, gaps by sub-question order. Tune the ranking later.
5. Add a "New question from gap" form, prefilled from the gap's sub-question or cluster name. It posts to the existing new-question flow.
6. Style nodes by kind (claim, entity, note) and edges by type. Contradicts edges should stand out.

## Acceptance criteria

- [ ] Tap a gap to start a new question
- [ ] On a desktop browser, the graph renders a test project's clusters with a size and confidence color for each.
- [ ] On a 380 px wide viewport, the page shows the ranked list and loads no Cytoscape script.
- [ ] The contradiction view shows both claims with their sources side by side.
- [ ] Notes appear as distinct nodes on the graph.
- [ ] A viewer-role user sees the map but not the "start question" action.

## Open questions

- How many nodes before the desktop graph needs filtering or aggregation? The design says Cytoscape "handles a few thousand nodes."
- Should "Ask about this cluster" be hidden until F33 ships, or link to unscoped chat?
