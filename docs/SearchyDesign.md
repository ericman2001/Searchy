# Searchy Design

Nov 2, 2026

## Overview

Searchy is a self-hosted, browser-first research workbench: you start a question from your phone, it researches asynchronously, and every claim it returns links to a scored, archived source.

Existing deep research tools (OpenAI and Gemini Deep Research, Perplexity, NotebookLM, open-source STORM and GPT Researcher) mostly produce one-shot reports. This project targets what they lack: persistent projects that accumulate knowledge across runs, visible source scoring, claim-level verification, idea maps, and your own notes and data as first-class sources.

It reuses two existing designs in this project:

- **Chat harness design** ([appendix](appendix/HarnessDesign.md)). The three-stage pipeline (input stages, mid-turn counselors, turn-end validation with retry) and the separate classifier layer become the research brain. The Rust core and TOML profiles carry over. Searchy builds this pipeline in its own workspace (features H1–H14 in [the plan](phases/PLAN.md)), and the harness's Ratatui TUI is replaced by web pages.
- **Podman hub.** The web hub, SQLite job queue, sibling worker containers with tmpfs scratch, Quadlet deployment and Tailscale access become the body that runs research while you're away.

The server side is Rust wherever Rust makes sense, reusing the chat harness crates. The client only ever sees a web interface, so it works from a phone with nothing installed. Project chat (below) is where the chat harness plugs in most directly.

## Goals and non-goals

**Goals**

- Start, approve, monitor and read research from any browser, including a phone on cellular.
- Research keeps running when the browser closes; results wait in the project.
- Every claim in a report links to a specific passage in an archived source.
- Sources carry a visible quality score with the reasons behind it.
- Notes, PDFs and datasets you upload are searchable sources alongside the web.
- Local corpora answer first; the live web is fetched politely and only when needed.
- Deployment is one command on a Linux host, with no Rust toolchain needed to run it.

**Non-goals**

- Not a general chat app or a coding agent.
- Not multi-tenant SaaS. One small team or household per instance.
- No MCP support, consistent with the chat harness decision; tools cover it.
- No native mobile app; the web UI is the mobile app.

## Architecture

Three layers: an always-on hub you reach from any browser, short-lived worker containers that do the research, and volumes that hold everything durable.

&#91;embedded content: system architecture · hub, workers, storage\]

Only the Fetcher touches the network; the Reader and Analyst work offline on what it archived and on your own files, so a malicious page can't exfiltrate your notes. Workers that need a model reach it through the hub's model gateway over an internal network that has no route to the internet (see Model routing).

## Web interface

The UI is server-rendered HTML from the Rust hub with htmx for interactivity, designed phone-first: one column, large tap targets, and pages that work on a slow connection. There is no build step and no JavaScript framework; the only client-side library is the graph view.

**Screens**

| Screen | What you do there |
| --- | --- |
| Inbox | See what needs you: plans awaiting approval, finished runs and reviews, failed jobs, watch updates |
| Projects | List and create projects; each shows its open questions and last activity |
| Project | Questions, runs, the claim ledger, sources, notes, outputs, reviews and chat threads for one project |
| New question | Type or dictate a question, pick a depth (quick, standard, thorough) and confirm the domain profile, set a budget |
| Plan review | Approve, edit or reject the proposed sub-questions and source list before fan-out |
| Run | Live progress over SSE: current step, sources fetched, claims found, cost so far |
| Report | The answer with inline citations; tap a citation to see the exact passage and source ratings; select text to Ask or Review |
| Ask | Chat with the project's findings, scoped to the project, a report, a claim or a cluster; make charts, tables and summaries |
| Review | Verdict card for a passage, claim, source or report: findings, evidence, proposed confidence change, accept or dismiss |
| Outputs | Saved charts, tables and summaries, with stale markers and a regenerate button |
| Sources | Every source with its tier, reliability breakdown, last-reviewed date, archive copy and override controls |
| Idea map | The claim graph (desktop-first; a simplified cluster list on phones), with "Ask about this cluster" |
| Library | Upload notes, PDFs and CSVs; replace a document by re-uploading it; share documents to projects; see ingestion status and data checks |
| Settings | Model endpoints, API keys, domain trust list, budgets, local corpora status |

**Async behavior**

- All state lives in SQLite on the host, so closing the tab or losing signal changes nothing.
- Progress streams over Server-Sent Events; on reconnect the page re-renders from the database.
- Web Push (via the browser's push API, no app store) notifies you when a plan needs approval or a run finishes. ntfy is a simpler fallback.
- The hub is installable as a PWA so it opens like an app from the home screen.

**Remote access and auth**

- Default: reach it over Tailscale or the local network; nothing is exposed to the internet.
- Optional: Cloudflare Tunnel or a reverse proxy for a public URL.
- HTTPS is optional. Over plain HTTP (for example `http://<lan-ip>:8080`), the hub uses password login and ntfy notifications. Browsers only allow passkeys, Web Push and PWA install in a secure context, so those turn on automatically when the hub is served over HTTPS, for example with `tailscale serve`.
- Built-in login is required even on a private network, because the hub can start containers: passwords always, plus passkeys (WebAuthn) when served over HTTPS.
- Sessions are long-lived on trusted devices so the phone stays signed in.

## Teams and permissions

Searchy is built for small teams from the start, because sharing, attribution and privacy are much harder to retrofit than to design in.

**Roles**

| Level | Role | Can |
| --- | --- | --- |
| Instance | Admin | Invite users, set API keys and model endpoints, set budgets, manage corpora and profiles |
| Instance | Member | Create projects, upload personal notes and datasets |
| Project | Owner | Share the project, set its default profile and budget, delete it |
| Project | Editor | Start runs and reviews, chat, edit the claim ledger, save outputs |
| Project | Viewer | Read reports, sources, outputs and chat threads; comment |

**Sharing and privacy**

- Projects are private to their creator until shared.
- Uploaded documents and datasets belong to a person and are private by default. Adding one to a project is an explicit share, and project members see only what was shared.
- A personal source that isn't shared never appears in another member's citations.

**Attribution and coordination**

- Every run, review, ledger edit and accepted verdict records who did it, shown in a project activity feed.
- Comments on claims, passages and reports, with notifications to whoever is mentioned.
- The job queue schedules fairly across users, so one person's thorough run can't starve everyone else.

**Budgets and keys**

- Cloud API keys are set once by the admin; spending is tracked per user and per project.
- Budget caps apply at three levels: instance, user and project. A run that would exceed any of them asks first.

**Accounts**

- Invite links from the admin; each person sets a password and, when the hub is served over HTTPS, registers a passkey.
- SQLite in WAL mode handles a small team's concurrent use comfortably; Postgres is not needed at this scale.

## Research pipeline

A research run is the chat harness's turn pipeline stretched over minutes or hours, with one human checkpoint after planning.

1. **Scope.** Input stages classify the question (factual, comparative, literature review, data analysis), check it against the project's existing claim ledger, and ask a clarifying question if it is ambiguous.
2. **Plan.** The planner proposes sub-questions, which corpora and source types to use, and a budget. The run pauses in `awaiting_approval` and notifies you. The planner also receives the running summary of the project's chat threads, so the run builds on what you've discussed.
3. **Gather.** Approved sub-questions fan out to workers. Each searches local corpora first, then the web through the fetch proxy.
4. **Read and extract.** Reader workers turn documents into passages and extract candidate claims, each tied to a passage span.
5. **Score.** Sources are scored (see the next section) and claims are merged with matching claims already in the ledger.
6. **Synthesize.** The writer drafts the report from the claims, citing passage ids, never raw URLs.
7. **Verify.** Every cited claim is checked against its passage. Only the failing sentences go back to step 6 with the reasons, up to the retry limit, then ship with visible flags.

If a counselor needs your input mid-run, the question becomes an Inbox item. The run step that asked pauses until you answer, and other branches keep going. Each research step otherwise starts from a fresh context.

**How the chat harness stages map**

| Harness stage | Research use |
| --- | --- |
| Classifiers | Question type, source type, language, paywall detection, claim type |
| Input stages | Scope check, ledger lookup, clarifying question, plan checkpoint |
| Counselors | Source-quality gate, prompt-injection guard on fetched text, budget, loop guard, domain politeness |
| Turn-end validation | Claim-to-passage verification, citation completeness, contradiction disclosure |

**Depth presets**

Depth is a named bundle of knobs that trade speed and cost for quality, independent of the domain profile: a health question can run at any depth. Every knob can be overridden per question in plan review, and the budget counselor enforces the caps. Values below are starting defaults to tune.

| Knob | Quick | Standard | Thorough |
| --- | --- | --- | --- |
| Plan checkpoint | Off | On | On |
| Clarifying questions | Only if ambiguous | Only if ambiguous | Always confirm scope |
| Sub-questions | 1–3 | 3–6 | 5–12 |
| Sources searched | Local corpora | Adds the web | Adds full-text academic sources and video transcripts |
| Stopping rule | First well-supported answer | Fixed sources per sub-question | Saturation: stop when two rounds change no confidence label |
| Citation chasing | Off | One hop | Until a primary source |
| Lateral reading | Off | Unknown domains only | Every source |
| Reasoning budget | Low everywhere | Medium for planner and judge | High for planner, judge and reviewer |
| Verification | Load-bearing claims only | Every citation | Every citation, plus a second verifier model; disagreements flagged |
| Retry limit | 1 | 2 | 3 |
| Synthesis | One draft | One draft and a self-critique pass | Two drafts, critiqued and merged |
| Adversarial review | Off | Top load-bearing claims | All load-bearing claims, debate mode |
| Budget cap | Small | Medium | Large; asks before exceeding |

Reasoning budget matters most for planning, judging and review. Extraction and summarizing run on small fast models at every depth. Most of the quality gain comes from verification, citation chasing and the stopping rule, not from longer thinking.

## Domain profiles

What counts as good evidence depends on the domain, so quality rules live in swappable TOML profiles, not in code. A profile can extend another, the system picks one automatically when a question starts, and you can override it at any level.

**What a profile defines**

| Field | Science example | Products example |
| --- | --- | --- |
| Evidence hierarchy | Umbrella review > meta-analysis > RCT > cohort > case report > mechanism > opinion | Lab test with published method > long-term owner reports > hands-on review > spec sheet > listicle |
| What "primary" means | The study's own data | Direct testing or months of ownership |
| Claim schema | Population, exposure, outcome, design, effect size | Product, criterion, test method, firmware or revision |
| Claim-kind weights | Measurement > mechanism > opinion | Measurement and observation > spec > opinion |
| Incentive weight | Flag only | Heavy discount on opinions |
| Evidence decay | Slow (years) | Fast (months) |
| Preferred corpora | PubMed baseline, PubMed Central, OpenAlex, arXiv | Test-site archives, video transcripts, owner forums |
| Reviewer checklist | Confounding, reverse causation, dose, extrapolation | Sponsorship, single unit, test conditions, firmware |
| Output template | Summary-of-findings table | Weighted comparison matrix |

Claim kinds are shared across profiles: measurement, observation, spec, opinion and mechanism. The kind decides how much incentive bias matters, so a sponsored reviewer's footage still counts as an observation while their "best of the year" verdict is discounted.

**Built-in profiles**

| Profile | For | Primary sources | Distinctive rules |
| --- | --- | --- | --- |
| Science | Any empirical question | Peer-reviewed studies and datasets | Evidence pyramid, structured claims, replication counts |
| ↳ Health overlay | Medicine, nutrition, fitness | Trials, cohort data, clinical guidelines | PICO claims, GRADE-style certainty, trial-registry checks, mechanism tier, health disclaimer |
| ↳ Climate overlay | Climate and environment | Observational datasets, assessment reports | IPCC assessments as anchors, model vs. observation claims, region and scenario in the claim schema |
| ↳ Software overlay | Software engineering research | Empirical studies, benchmarks, reproducible artifacts | Reproducibility and benchmark-setup checks |
| Products | Buying and comparing things | Tests with published methods, ownership reports | Incentive detection, product entity resolution, priority weighting, fast decay |
| News and events | What happened and what's happening | Official documents, on-the-scene reporting | Very fast decay, wire-syndication independence, corrections tracking, outlet diversity |
| Legal and policy | Laws, regulations, rulings | Statutes, regulations, court opinions | Jurisdiction and effective date on every claim, superseded-law checks, not-legal-advice notice |
| History and humanities | Past events, texts, ideas | Archival documents, contemporaneous accounts | No decay, dating and authorship checks, historiography |
| Technical docs | How to do X with tool Y | Official docs, source code, changelogs, issue trackers | Version on every claim; claims can be tested by running code in an Analyst worker |
| Finance and companies | Companies, markets, filings | Regulatory filings, earnings reports | Volatile figures fetched live, heavy incentive weighting, not-financial-advice notice |
| General | Anything else | Judged per source | The defaults described in this doc |

Overlays declare `extends = "science"` and override only what differs, so health and climate share the science base's evidence pyramid and replication rules.

**Auto-selection**

- At the Scope stage, a classifier proposes a profile with a confidence and a one-line reason, such as "health overlay: effect of a food on outcomes in people."
- Plan review shows the choice as a dropdown; changing it re-plans the run.
- Low confidence means the system asks instead of guessing.
- Mixed questions get a profile per sub-question. "Is coffee healthy, and which espresso machine should I buy?" splits into a health branch and a products branch.
- Overrides work at three levels: project default, question and sub-question. The profile used is recorded on every run so reports are reproducible.
- Custom profiles are TOML files in a profiles folder, validated at startup like the chat harness's stage profiles.

## Model routing

Every model call belongs to a role, and each role is assigned a model, so a large cloud model can direct the research while small, cheap models do the high-volume work.

**Roles**

| Role | Does | Suggested default |
| --- | --- | --- |
| Director | Scopes questions, writes plans, decides when to stop | Large cloud model |
| Classifier | Profile selection, source type, claim kind, answerability | Small model, with rules first where possible |
| Extractor | Turns passages into structured claims | Small or mid model; the highest-volume role |
| Summarizer | Condenses sources, names clusters | Small model |
| Writer | Drafts reports and outputs | Large model |
| Verifier | Turn-end check that each passage supports its claim | Small model first, escalating unclear cases |
| Reviewer | Adversarial review and debate | Large model, ideally from a different family than the writer |
| Chat | Project chat | Mid or large model |
| Embeddings | Vectors for retrieval and clustering | Local model |

**Assignment layers**, each overriding the one before:

1. Instance defaults, set by the admin.
2. Domain profile: the health overlay might raise the verifier to a mid-size model.
3. Depth preset: quick uses small models everywhere except the director; thorough adds a second verifier from another model family.
4. Project.
5. Question, in plan review.

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

**Rules**

- **Model gateway.** Every model call goes through a gateway in the hub. Workers sit on an internal Podman network (`--internal`, no internet route) where the gateway is the only thing they can reach. The gateway reaches local models on the LAN or localhost and the cloud APIs, holds the API keys, and records per-role cost, so keys never enter a worker.
- **Local-only data stays local.** A role that would send a local-only document or dataset to a cloud model is rerouted to a local model, or the step is skipped and flagged. The gateway enforces this in one place.
- **Escalation, not blind trust.** Small models handle format checks and clear support or no-support cases well but miss subtle mismatches, such as a different population or hedged wording. The verifier escalates anything it's unsure of.
- **Fallback chains.** Rate limits or outages move to the next model in a role's list, and the trace and report record which model actually ran.
- **No self-grading.** The reviewer avoids the writer's model when another is configured.
- **Per-role costs** feed the budget counselor, so you can see where money goes and which role to downsize.

## Source quality and verification

Every claim carries three separate ratings: how far it is from the original, how reliable its source is, and how well it is corroborated. The UI shows all three as badges instead of blending them into one number.

**1. Provenance distance**

| Tier | Meaning | Examples |
| --- | --- | --- |
| Primary | Original data, text or firsthand account | A study's own results, a statute, a dataset, a quote in the outlet that did the interview |
| Secondary | Reports on or analyzes primaries | Review articles, news coverage of a study, analyst reports |
| Tertiary | Summarizes secondaries | Wikipedia, textbooks, explainers |
| Derived | Machine-generated text | Saved chat answers, suspected AI content-farm pages |

- The tier belongs to the claim-passage pair, not the source. A news article is primary for its own interview quote and secondary for a study it describes.
- Citation chasing: when a passage attributes a claim to another work, a counselor queues that work, and reports cite the deepest tier found.
- Derived passages are searchable and visible but never count toward corroboration.

**2. Source reliability**

| Signal | Where it comes from | Notes |
| --- | --- | --- |
| Source type | Classifier on URL, metadata and content | Peer-reviewed, preprint, government, standards body, news, reference, blog, forum, your notes |
| Retraction or correction | Crossref metadata (includes Retraction Watch data) | A retraction caps the score regardless of other signals |
| Evidence level | Judge extraction for studies | Design (meta-analysis, RCT, observational, case report), sample size, preregistration, data availability |
| Citation impact | OpenAlex | Normalized by field and age so new papers aren't punished |
| Domain trust | Your editable list, seeded from Wikipedia's perennial sources list and DOAJ membership | Per-project overrides |
| Lateral reputation | Searches for what other sources say about this one (the SIFT method) | Cached per domain and author |
| Specificity | Judge on the passage | Named sources, verifiable details and linked data score up; "experts say" scores down |
| Content-farm and AI signals | No author, templated structure, very high publish rate, no outbound citations | Weak down-weight plus the derived tier; detectors are unreliable, so never an automatic exclusion |
| Recency | Publication date vs. question type | Matters for current-state questions, not history |
| Conflict of interest | Judge on funding and affiliation statements | Flag only; never auto-penalize |

Signals are stored per source, per project, so overrides and judgments in one project don't leak into another. They combine into a reliability score using weights from the domain profile (General defaults until a profile sets them). Caps, such as a retraction, apply after weighting.

**3. Claim confidence**

- Independence comes first. Sources collapse into one origin when they share a press release, wire syndication, near-duplicate text (MinHash), the same author or organization, or a citation loop (citogenesis: Wikipedia citing an article that took the fact from Wikipedia).
- The earliest dated appearance of a claim is treated as its likely origin; later copies add nothing.
- Support and contradiction, weighted by reliability and tier, map to calibrated labels in the style of IPCC and GRADE: high, moderate, low, contested, unverified.
- Starting rule, tunable in the profile TOML: "high" needs at least two independent origins, one of them primary, and no unresolved contradiction from a source of similar reliability.

**Your own data**

- Uploaded notes are their own tier with a reliability you set per document or collection.
- Datasets record where they came from, and ingestion runs schema, null and range checks; failures show on the dataset and on any chart built from it.

**Verification (turn-end stage)**

- Every sentence in a report that makes a factual claim must cite at least one passage id.
- A judge model checks each citation: does the passage support, partially support, or not support the sentence?
- Unsupported sentences are rewritten or removed on retry, and only those sentences are sent back, not the whole draft. After the limit, they ship marked as unverified.
- Numbers get the calculator-example treatment from the chat harness: the harness re-reads the number from the passage itself and compares.

## Adversarial review

Select a passage, claim, source, report or chart and ask for a review: a separate reviewer agent tries to break it and returns a verdict card saying how well it holds up, like a code-review agent run against a codebase.

**What can be reviewed**

- A highlighted passage in a report, a source or your own notes.
- A single claim from the ledger.
- A whole source ("is this paper still sound?").
- A report or output, which fans out to its load-bearing claims: the ones its conclusion depends on.

**The reviewer's checklist** (a TOML profile, like the review skills in coding tools)

1. **Status check:** retractions, corrections, expressions of concern and errata (Crossref); a preprint whose published version changed the results; laws, standards or guidelines that were later superseded.
2. **Trace to primary:** does the original actually say this? Looks for quote drift, misread numbers, stripped context and a population or timeframe that quietly changed.
3. **Counter-evidence search:** deliberately searches for refutations, failed replications, published comments and letters, and post-publication critique sites such as PubPeer.
4. **Independence audit:** are the supporting sources really independent, or one origin repeated?
5. **Method critique** (studies): design, sample size, confounders, multiple comparisons, and generalizing beyond the study population.
6. **Reasoning and framing:** correlation presented as causation, relative risk without absolute risk, cherry-picked timeframes, claims broader than the evidence.
7. **Currency:** is it still true, or has newer evidence moved past it?

**How it runs**

- The reviewer is a fresh agent that never saw the writing, so the work isn't grading itself. A different model from the writer is the recommended default.
- The `thorough` depth uses a debate: a prosecutor builds the strongest case against the claim, a defender answers, and a judge writes the verdict from both, citing passages.
- It's a background job with its own budget. Fetches go through the same polite proxy. A single claim runs without a plan checkpoint; a whole report shows its list of claims to review first.

**The verdict card**

| Verdict | Meaning |
| --- | --- |
| Holds up | Survived every check; nothing material found |
| Holds with caveats | True as far as it goes, but narrower than stated or missing context |
| Weakened | Significant counter-evidence or method problems; confidence should drop |
| Contradicted | Better evidence says otherwise |
| Retracted or superseded | The source itself was withdrawn or replaced |

Under the verdict: a short summary, each finding with its severity and cited passages, and the proposed change to the claim's confidence. Nothing changes until you accept it; accepting updates the ledger, adds contradicts edges to the idea map, and marks dependent reports and outputs stale.

**Keeping reviews current**

- Every claim and source shows a "last reviewed" date.
- Optional auto-review of the top load-bearing claims at the end of each research run.
- Watches can re-run status checks (retractions, corrections) on a project's key sources and notify only on change.

## Worked examples

Two projects run through the same pipeline with different profiles, and the contrast is the test: if one profile system handles both well, it will handle most questions.

### Coffee and health (science + health overlay)

1. **Scope.** The classifier picks the health overlay. The system asks which outcomes (mortality, heart, cancer, sleep, pregnancy, anxiety), which population (healthy adults, pregnant people, people with hypertension) and what exposure means (cups per day, caffeinated or decaf, filtered or unfiltered). Without this, "coffee is healthy" and "coffee is harmful" are both true somewhere.
2. **Plan.** One sub-question per outcome, working down the evidence pyramid: umbrella reviews and meta-analyses first, then guideline bodies such as EFSA's caffeine safety opinion, and individual studies only where reviews disagree or are out of date.
3. **Gather and weed.** The local PubMed baseline and PubMed Central subset answer most of it. Blogs, supplement sellers and "coffee cures X" pages are fetched but classified as promotional or tertiary and never count as support. Mouse and cell studies go to the mechanism tier.
4. **Extract.** Claims carry population, exposure, outcome, design and effect size, so a finding in older Japanese adults never merges into "coffee lowers mortality."
5. **Review.** The health checklist looks for confounding (coffee drinkers once smoked more), reverse causation (people who get sick often quit coffee), dose-response shape, relative vs. absolute risk, animal-to-human extrapolation, and whether Mendelian randomization studies agree with observational ones. ClinicalTrials.gov catches trials whose reported outcomes differ from their registered ones.
6. **Stop.** Thorough depth keeps searching until new sources stop moving any outcome's certainty.
7. **Report.** A summary-of-findings table (outcome, direction, number of studies, certainty), a populations-to-be-careful-with section, a "claims circulating online" table comparing popular claims with the evidence, and the health disclaimer.

### Robot vacuum comparison (products)

1. **Scope.** The classifier picks products. The system asks about floor types, rugs and thresholds, pets, budget, mopping or self-emptying, privacy (cameras, cloud accounts) and region, since model names differ by country. Your answers become a priority weighting such as "pet hair 3×, mopping 0×."
2. **Plan.** Shortlist candidate models, then one sub-question per criterion: suction on carpet, edge cleaning, navigation, obstacle avoidance, noise, app and privacy, long-term reliability, running costs.
3. **Gather.** Testers who publish their methods first, then long-term owner reports in forums, then hands-on video reviews. Video is ingested as transcripts with timestamp citations. Manufacturer pages count only as specs.
4. **Extract.** Claims carry product, criterion, test method and firmware, and are typed: measurement, observation, spec or opinion. Entity resolution merges regional names and revisions of the same model.
5. **Weigh incentives.** Affiliate links, sponsorship disclosures, free review units and listicle networks copying each other are detected. Opinions from these sources are discounted heavily; their observations ("it got stuck on this rug at 4:12") still count.
6. **Compare fairly.** Scores are compared only within one tester's method; across testers, the system checks whether rankings agree. Prices and availability are fetched live at report time, never stored as claims.
7. **Report.** A weighted models × criteria matrix with evidence behind each cell, a "who it's for" summary per model, and a watch that flags new models, firmware changes and reliability reports.

### What the contrast teaches

|  | Coffee | Vacuums |
| --- | --- | --- |
| "Primary" means | Study data | Direct testing or months of ownership |
| Claim shape | Population, exposure, outcome, design | Product, criterion, test method, firmware |
| Main source of bias | Confounding, weak study designs | Money: affiliates, sponsorship |
| Evidence shelf life | Years | Months |
| Key media | Papers | Video, forums, test sites |
| Answer shape | Per-outcome evidence table | Weighted comparison matrix |

Everything in the right column that the base design lacked became part of the profile system: claim kinds, incentive weighting, video transcripts, entity resolution, priority weighting, volatile live facts and per-profile decay.

## Knowledge store

Each project owns a durable store of sources, passages and claims; runs add to it instead of starting over, so the tenth question in a project is faster and better grounded than the first.

**Organization**

- **Project** → **questions** → **runs**. A question can be rerun or put on a watch.
- **Claim ledger:** the project's running "what we know" list. Each claim has its supporting and contradicting passages, a confidence, and the runs that touched it. You can pin, edit or reject claims, and the next run respects that.
- **Sources** are shared across projects (one archive copy), but trust overrides and notes are per project.

**Bringing in your own material**

| Input | How it's ingested |
| --- | --- |
| Markdown notes | Uploaded in the Library, singly or several at once; re-uploading replaces and re-indexes; links and tags kept as metadata |
| PDFs | Text extraction in a no-network worker; scanned PDFs get OCR |
| Web pages you save | Bookmarklet or share-sheet target posts the URL to the hub |
| CSV, Parquet, SQLite | Registered as datasets; the Analyst worker queries them with DuckDB, never pastes them into prompts |
| BibTeX or Zotero library | Imported as sources with their metadata |

Your material is its own trust tier, labeled "your notes" in citations so you can see when an answer leans on you rather than the outside world.

**Retrieval**

- Hybrid search: BM25 keyword search (`tantivy`) plus vector search, merged with reciprocal rank fusion.
- Passages are chunked by document structure (headings, paragraphs), not fixed token windows.
- Embeddings come from a local model through LM Studio or `fastembed-rs`, so private notes never leave the machine.
- Every fetched page is archived as WARC, so citations keep working after the original changes or disappears.

## Idea clouds

The idea map is a view over the claim ledger, not a separate store: claims and entities are nodes, and the edges between them are where the insight is.

- **Nodes:** claims, entities (people, organizations, methods, datasets), and your notes.
- **Edges:** supports, contradicts, cites, same-entity, and "same topic" from embedding similarity.
- **Clusters:** claims are embedded and clustered (HDBSCAN), and a small model names each cluster. Cluster size shows how much evidence exists, and color shows average confidence.
- **Contradiction view:** pairs of claims with a contradicts edge, side by side with their sources. This is often the most valuable screen.
- **Gap view:** sub-questions from the plan with few or no supporting claims, and clusters with only one source. One tap turns a gap into a new question.
- **Your notes on the map:** notes appear as nodes, so you see where your own thinking agrees or disagrees with the sources.

On desktop the map is an interactive graph (Cytoscape.js). On a phone it collapses to a ranked list of clusters, contradictions and gaps, since graphs don't work at 380 pixels.

## Project chat

Once a project has findings, you can talk to them: ask questions, challenge claims and generate outputs, all answered from the project's own store with no new research. When the store can't answer, the chat says so and offers a research run instead of quietly searching.

**Two modes, one project**

|  | Ask (project chat) | Research run |
| --- | --- | --- |
| Answers from | Claim ledger, passages, archived sources, your notes and datasets | Local corpora and the web, then the ledger |
| Network | None | Fetcher only |
| Time | Seconds, one model loop | Minutes to hours, many workers |
| Plan checkpoint | No | Yes |
| Changes the ledger | Only when you pin, edit or reject a claim | Adds and merges claims |

**How it works.** Project chat is the chat harness itself, pointed at one project. It runs in the hub with project-scoped tools:

- `search_claims`, `get_passage`, `get_source`: the same hybrid retrieval, limited to the project.
- `list_contradictions`, `list_gaps`, `get_cluster`: the idea-map views as tools.
- `query_dataset`: DuckDB over registered datasets, run in an Analyst worker.
- `make_chart`, `make_table`, `save_output`: produce outputs (below).
- `start_research`: proposes a run; it never starts one without your approval.

The chat harness stages carry over. A classifier labels each question as answerable, partly answerable, or out of scope for the project; answers cite passage ids; and the turn-end verifier checks them exactly as it checks reports.

**Scoping.** Start a chat from the project, a report, a single claim or an idea-map cluster, and retrieval is limited to that subset.

**When the project can't answer.** The reply says what is missing and offers two actions: research it as a new related question in this project, or extend the original question's run. Both go through the normal plan checkpoint, new claims merge into the same ledger, and the chat picks up the new findings when the run finishes.

**Outputs without new research**

| Output | Built from | Format |
| --- | --- | --- |
| Charts | Numeric claims or dataset queries | Vega-Lite spec, rendered in the browser |
| Tables and comparisons | Claims grouped by entity or question | HTML table, CSV export |
| Summaries and briefings | The ledger, scoped to a question, cluster or audience | Markdown with citations |
| Timelines | Dated claims | Vega-Lite or HTML list |
| Contradiction digests | Contradicts edges | Side-by-side Markdown |

- Every output is saved with the claim and query ids it was built from, so it can be regenerated, and it is marked stale when those claims change.
- Numbers in charts come from passages or queries, never from the model's prose; the harness reads the values itself.
- Chat answers can be saved to the project, labeled as derived. Derived text is searchable but never counts as an independent source, so the tool can't corroborate itself.
- Chat threads are kept per project and are searchable later.
- Chat is part of ongoing research. Long threads are compacted, and their important points are kept as a running summary that is injected into the planner and research agents' context.

## Being a good netizen

The harness answers from local copies first and treats every live fetch as a cost to someone else's server.

**Local corpora (optional, installed from Settings)**

| Corpus | Form | Use |
| --- | --- | --- |
| Wikipedia | Kiwix ZIM file served by `kiwix-serve` in its own container | General background, entities, definitions |
| Wikidata | Dump loaded into a local index | Structured facts and entity linking |
| arXiv metadata | Bulk metadata dump | Finding preprints before fetching any PDF |
| OpenAlex | Snapshot, or the API with a `mailto` for the polite pool | Scholarly metadata and citation counts |
| Stack Exchange | Data dumps | Technical questions |
| Project Gutenberg | Mirror subset | Historical and literary text |

The default Wikipedia download is Kiwix's no-pictures English build: full article text without images. ZIM files contain articles only, so talk pages are never included. Corpus sizes and update schedules should be checked at install time rather than hard-coded.

**Fetch proxy (the only worker with network)**

The Fetcher is a queued per-job worker, started by the launcher like the other worker roles.


- Obeys robots.txt and honors `Crawl-delay`.
- Per-domain rate limits and concurrency caps, with exponential backoff on 429 and 503.
- An honest User-Agent naming the project with a contact URL.
- A shared HTTP cache with conditional requests (ETag, If-Modified-Since), so a page is fetched once across all projects.
- Prefers APIs and bulk dumps over scraping where a site offers them.
- Never bypasses paywalls or logins; a paywalled source is recorded as metadata only.
- Search goes through your SearXNG instance, which spreads queries across engines.

**Archiving:** every fetched page is stored as WARC, which also means reruns and verification never refetch.

## Deployment

The whole system ships as one container image containing one static Rust binary, and installing it is one command; everything after that happens in the browser.

**One image, several roles.** The same image runs as `hub`, `fetcher`, `reader` or `analyst` depending on its first argument, matching the sibling-worker pattern from the Podman chat. Kiwix runs as its own upstream image. Images are published to a container registry for amd64 and arm64, so a Raspberry Pi 5 or a small ARM server works.

**Two supported install paths**

| Path | Best for | How |
| --- | --- | --- |
| Podman Quadlets (recommended) | Fedora, Bazzite, any systemd host | Installer writes `.container`, `.volume` and `.network` units to `~/.config/containers/systemd/`, then starts them |
| Compose file | Docker users, quick trials | `docker compose up -d` or `podman compose up -d` with the same file |

The installer is a short script that checks for Podman or Docker, writes the units or compose file, and prints the URL. A minimal Quadlet for the hub looks like this:

```ini
[Container]
Image=ghcr.io/<owner>/searchy:latest
Exec=hub
Volume=searchy-data:/data
Volume=searchy-corpora:/corpora
Volume=searchy-launcher:/run/searchy-launcher
PublishPort=127.0.0.1:8080:8080
Label=io.containers.autoupdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

**Isolation, in two layers.** The hub never holds the container socket, and the account that does hold it owns nothing else.

1. **Dedicated user.** The installer creates a `searchy` Linux user (the one step that needs sudo), enables lingering so its services run without a login, and installs the Quadlets in that user's systemd folder. A compromise reaches only that account's containers and volumes, never your home folder.
2. **Launcher sidecar.** A small container is the only thing that mounts the rootless Podman socket. It listens on a Unix socket shared with the hub and accepts a fixed menu of requests: start a worker of a given role for a job, stop it, report status, stream logs. The launcher itself chooses the image (pinned by digest), mounts (only that job's folders), network and resource limits, rejects anything else, and logs every request. Workers join only the internal model network, which reaches the hub's model gateway and nothing else; the Fetcher additionally joins the proxy network.

A compromised hub can therefore only start approved workers on approved paths, and a compromised worker is still inside a no-network, read-only container with tmpfs scratch.

**Worker runner backends.** Launching workers sits behind a `Runner` trait so one deployment shape doesn't lock out others:

- `podman`: sibling containers started by the launcher sidecar, which holds the rootless Podman socket (default).
- `docker`: the same launcher, holding the Docker socket instead.
- `local`: workers as child processes in the hub's container, no extra isolation. For trials and machines without a socket; the UI warns that it is less isolated.

**Storage**

- `searchy-data`: SQLite, indexes, WARC archive. The volume to back up.
- `searchy-corpora`: Wikipedia and other dumps, kept separate so it can live on a big, slow disk and be skipped in backups.
- Uploaded documents live in `searchy-data` with everything else; there are no host folder mounts.

**First run, in the browser**

1. Create the admin account with a password, and a passkey if the hub is served over HTTPS.
2. Pick model backends. Cloud APIs (Anthropic, OpenAI) work out of the box with an API key; local OpenAI-compatible endpoints (LM Studio, Ollama, llama.cpp) are optional. A document or dataset can be marked local-only, so its contents never go to a cloud model. Each role (director, writer, verifier and the rest) gets a sensible default model you can change later.
3. Optionally choose local corpora; downloads run as background jobs with progress in the Inbox.
4. Point it at a SearXNG instance, or use the bundled SearXNG container. Notes and documents are uploaded later in the Library.

**Upgrades and operations**

- `podman auto-update` (or Watchtower for Docker) pulls new images; database migrations run at hub startup.
- `hub` exposes `/healthz` for container health checks.
- A `backup` role writes a consistent SQLite snapshot plus the archive to a target folder.
- Logs go to stdout, so `journalctl` or `docker logs` just work.

## Tech stack

Everything server-side is Rust in one Cargo workspace; the browser gets plain HTML plus a few small libraries.

| Layer | Choice | Why |
| --- | --- | --- |
| Web server | `axum` + `tokio` | Async, pairs with the harness core |
| Templates | `askama` (compile-time) | Type-checked HTML, nothing to ship at runtime |
| Interactivity | htmx + SSE | Phone-friendly, no frontend build |
| Chat streaming | SSE token stream into an htmx-swapped message list | Same transport as run progress |
| Graph view | Cytoscape.js | Mature, handles a few thousand nodes |
| Charts | Vega-Lite specs rendered with vega-embed | Models write the JSON reliably; specs are saved and portable |
| Auth | `webauthn-rs`, `argon2` | Passkeys plus password fallback |
| Database | SQLite via `sqlx` | One file, easy backup, migrations built in |
| Keyword search | `tantivy` | BM25 in Rust |
| Vector search | `sqlite-vec` (or LanceDB if it outgrows it) | Stays in the same database file |
| Embeddings | LM Studio endpoint or `fastembed-rs` | Local, private |
| Data analysis | `duckdb` crate | Your CSVs and Parquet, queried in place |
| HTTP fetching | `reqwest` + `texting_robots` + `governor` | Robots.txt parsing and rate limiting |
| Archiving | `warc` crate | Standard web-archive format |
| HTML to text | `readability` port + `scraper` | Main-content extraction |
| PDF text | `pdf-extract` or Poppler in the reader image | Poppler is more robust for odd PDFs |
| Containers | Podman / Docker API via `bollard` | One client for both sockets |
| Harness core | Crates from the chat harness | Stages, classifiers, profiles, tracer, project chat loop |

Crate maturity changes quickly; check each one when its feature starts rather than locking choices now.

## Feature breakdown

Forty-seven features in ten milestones, ordered so the tool is deployable and usable from a phone after M1, and useful for real research after M3. M8 (project chat) can start right after M3; its cluster and dataset features wait on M6 and M7. M9 (review) needs M5 and the web fetcher, and M10 (profiles) builds on M9. Each row is sized for one Claude Code session.

| # | Feature | Depends on | Done when |
| --- | --- | --- | --- |
| **M1** | **Deployable shell** |  |  |
| F1 | Cargo workspace; one binary with `hub` / worker roles; harness core crates imported | — | `searchy hub` serves a page; `searchy fetcher` runs and exits |
| F2 | Container image (static musl build), multi-arch CI publish | F1 | Image runs on amd64 and arm64 |
| F3 | Quadlet units, compose file, installer script that creates the dedicated searchy user with lingering | F2 | Fresh Fedora VM: one command to a working login page |
| F4 | Accounts and auth: invite links, passwords, passkeys when served over HTTPS, long-lived sessions, admin and member roles | F1 | Phone stays signed in across days |
| F5 | Projects, questions, runs in SQLite with migrations; project roles (owner, editor, viewer); attribution on every action; htmx pages | F1 | Create a project from a phone; a second user sees it only after being added |
| F6 | Job queue + `Runner` trait; launcher sidecar with a fixed request menu and pinned images; podman, docker and local backends; restart recovery | F5 | Dummy job runs in a sibling container; the hub has no socket; an off-menu launcher request is rejected and logged; hub restart re-queues the job |
| F7 | SSE progress, Inbox, Web Push (HTTPS) and ntfy notifications, PWA manifest | F6 | Phone gets a push when a dummy job finishes |
| **M2** | **Local knowledge** |  |  |
| F8 | Model registry and role routing: endpoints (Anthropic, OpenAI, OpenAI-compatible), role assignments, verifier escalation, fallback chains, local-only enforcement, per-role cost | F5 | Each role test-calls its model; a local-only note never reaches a cloud model in the trace |
| F9 | Passage store: structure-aware chunking, `tantivy` index, `sqlite-vec` embeddings, hybrid retrieval | F8 | Query over a test corpus returns ranked passages with ids |
| F10 | Kiwix container + corpus manager (download as background job, status in Inbox) | F6, F9 | Wikipedia ZIM searchable through the hub |
| F11 | Notes ingestion: uploaded Markdown notes and PDFs, private by default and shareable to projects; PDF extraction in reader worker | F9 | Re-upload an edited note; it is re-indexed within a minute |
| F12 | Quick depth end to end: question → local retrieval → cited answer | F9–F11 | Answer cites Wikipedia and your notes by passage |
| **M3** | **Ledger and verification** |  |  |
| F13 | Claim extraction tied to passage spans; claim ledger with merge of matching claims | F12 | Two runs on related questions share ledger claims |
| F14 | Verification stage: judge per citation, retry, unverified flags | F13 | Planted unsupported sentence is caught and removed or flagged |
| F15 | Number cross-check (calculator-style re-read from passage) | F14 | Altered figure in a draft triggers a retry |
| F16 | Report page: tap a citation to see passage, source and score | F14 | Works one-handed on a phone |
| **M4** | **The web, politely** |  |  |
| F17 | Fetch proxy worker: robots.txt, per-domain limits, backoff, cache, User-Agent | F6 | Load test against a local server never exceeds the configured rate |
| F18 | WARC archiving + archive viewer | F17 | Citation opens the archived copy after the origin is offline |
| F19 | SearXNG integration (external or bundled) | F17 | Standard depth pulls web sources |
| F20 | Prompt-injection counselor on fetched text | F17 | Test page with planted instructions doesn't change the run |
| **M5** | **Source quality** |  |  |
| F21 | Source-type classifier; domain trust list with per-project overrides | F17 | Sources show type and trust in the Sources screen |
| F22 | Crossref retraction check; OpenAlex citation and metadata lookup (with `mailto`) | F21 | Known retracted paper is capped and flagged |
| F23 | Claim confidence: independence detection and corroboration counts | F13, F21 | Two articles from one press release count once |
| **M6** | **Async research** |  |  |
| F24 | Planner + plan checkpoint (`awaiting_approval`, edit plan from phone) | F7, F12 | Approve a plan from a phone push notification |
| F25 | Fan-out of sub-questions to parallel workers with budget counselor | F24 | Thorough run stays within its budget and reports cost |
| F26 | Datasets: register CSV/Parquet, Analyst worker with DuckDB | F6, F8 | Question about your CSV answered with a cited query |
| F27 | Watches: scheduled reruns showing only new claims | F24 | Weekly watch notifies only on change |
| **M7** | **Idea clouds and export** |  |  |
| F28 | Claim graph edges (supports, contradicts, cites, same-entity), clustering and cluster naming | F23 | Clusters named; contradiction pairs listed |
| F29 | Idea map UI: Cytoscape on desktop, ranked cluster/contradiction/gap list on phone | F28 | Tap a gap to start a new question |
| F30 | Export: Markdown report, BibTeX, Zotero import/export; backup role | F16 | Report and bibliography open cleanly in Zotero |
| **M8** | **Project chat and outputs** |  |  |
| F31 | Project chat loop: chat harness in the hub with project-scoped retrieval tools, cited answers, turn-end verifier, saved threads | F14 | Question about a finished project is answered with passage citations and no network calls in the trace |
| F32 | Answerability classifier + research handoff (new related question or extend the run) through the plan checkpoint | F24, F31 | Out-of-scope question offers a run; when it finishes, the same thread answers it |
| F33 | Scoped chat (report, claim, cluster) + idea-map tools (contradictions, gaps, clusters) | F28, F31 | "Ask about this cluster" retrieves only that cluster's claims |
| F34 | Outputs: Vega-Lite charts, tables, summaries, timelines, with values read by the harness and saved with source ids | F26, F31 | A chart from a dataset query and one from numeric claims both match their sources |
| F35 | Output staleness + regenerate; save chat answers as derived notes excluded from corroboration | F23, F34 | Editing a claim marks its chart stale; a derived note never raises a claim's confidence |
| **M9** | **Source depth and adversarial review** |  |  |
| F36 | Provenance tiers per claim-passage, citation-chasing counselor, derived tier with content-farm signals | F13, F17, F21 | A news passage citing a study queues the study, and the report cites it at primary tier |
| F37 | Lateral reputation, evidence-level and specificity judges; calibrated confidence labels; dataset ingestion checks | F23, F36 | Claims show high, moderate, low or contested badges with their reasons |
| F38 | Review job: select a passage, claim, source or report; reviewer checklist profile; fresh reviewer agent | F14, F17, F22 | A planted citation to a retracted paper returns "Retracted or superseded" |
| F39 | Debate mode (prosecutor, defender, judge) at thorough depth | F38 | Both sides cite passages and the verdict cites both |
| F40 | Verdict card + accept flow (ledger, contradicts edges, staleness); last-reviewed badges; auto-review of load-bearing claims; status-check watches | F27, F28, F35, F38 | Accepting a "Weakened" verdict lowers the claim's confidence and marks its report stale |
| **M10** | **Domain profiles and media** |  |  |
| F41 | Profile system: TOML schema, extends overlays, startup validation, profile recorded on every run | F9, F24 | A custom overlay that changes one field loads and shows in plan review |
| F42 | Auto-selection: classifier with confidence and reason, dropdown override, per-sub-question profiles | F41 | "Is coffee healthy, and which espresso machine should I buy?" splits into health and products branches |
| F43 | Science base + health overlay: structured claims, evidence pyramid, mechanism tier, trial-registry check, summary-of-findings output, PubMed baseline and PMC corpora | F37, F41 | Coffee example produces a per-outcome certainty table |
| F44 | Products profile: claim kinds, incentive counselor, product entity resolution, priority weighting, live volatile facts, comparison matrix | F37, F41 | Vacuum matrix discounts a sponsored "best of" verdict but keeps its footage as an observation |
| F45 | Video transcript ingestion with timestamp citations | F11, F17 | A citation opens the video at the cited time |
| F46 | Saturation stopping for thorough runs; "claims circulating online" view | F25, F43 | A run stops once two search rounds change no certainty label |
| F47 | Remaining built-ins: news, legal, history, technical docs (code-tested claims), finance | F41 | Each ships with a sample question that passes its own reviewer checklist |

## Open questions

- [ ] **Name check.** Confirm "Searchy" is free on GitHub, crates.io and container registries, and consider whether it reads too close to SearXNG, which Searchy uses.

**Decided**

- [x] Container access: a dedicated searchy user plus a launcher sidecar; the hub never holds the socket.
- [x] Front end: web UI only; the chat harness is a separate, unbuilt project and doesn't constrain this one.
- [x] Pipeline: Searchy implements the full harness pipeline itself, in Rust, adapted from the harness design; it doesn't wait on a separate harness project.
- [x] Models: cloud APIs enabled out of the box; local models optional; notes and datasets can be marked local-only; models assigned per role.
- [x] Users: small teams from the start (see Teams and permissions).
- [x] Wikipedia: no-pictures English build (article text only).
- [x] License: AGPL, so the source stays open, including for hosted forks.
- [x] Name: Searchy.
- [x] Name check: "Searchy" stays for now.
- [x] HTTPS: optional. Plain HTTP gets password login and ntfy; passkeys, Web Push and PWA install turn on over HTTPS (for example `tailscale serve`).
- [x] Model access from workers: a model gateway in the hub, reached over an internal network with no internet route. The gateway holds the keys and enforces local-only routing.
- [x] Fetcher: a queued per-job worker started by the launcher.
- [x] Personal notes: uploaded documents in the Library, indexed into the database store; no folder mounts.
- [x] Reliability: signals stored per source, per project; signal weights are a domain-profile setting.
- [x] Turn records: no separate session record; each turn belongs to a run step or a chat thread.
- [x] Mid-run questions: become Inbox items; the asking step pauses, other branches continue.
- [x] Retries: turn-end retries target only the failing sentences.
- [x] Chat and research: chat threads are compacted into a running summary that is injected into the planner and research agents.
