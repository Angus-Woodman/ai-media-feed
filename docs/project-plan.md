# Signal: AI-Powered Personal Football Media Inbox

**Decision:** Build Signal as an AI-powered personal football media inbox.

It ingests articles from two fixed football RSS feeds, removes exact duplicates,
extracts structured football entities and topics, ranks each item against an
explicit interest profile, and presents an unread-first feed.

## Build order

**Prove the product before proving the infrastructure.**

The first goal is two real feeds → ingestion → exact deduplication → structured
AI classification → deterministic ranking → a feed that is noticeably more
useful than reading the source feeds. Start with the simplest architecture that
can test that loop. Add sources, sports, richer feedback, queues, tracing,
Terraform and production hardening only when evidence justifies them.

## Product concept

### One-line pitch

**A personalised, unread-first football briefing that turns two noisy feeds into
a small queue of genuinely relevant stories.**

### Target user

A football fan who follows particular teams, competitions and topics but does
not want to check multiple sites or repeatedly see the same story.

### Core job to be done

> “Show me the new football content I’m likely to care about, explain why it
> matches my interests, and hide repeats and things I’ve already consumed.”

### Opinionated scope

Start with **football only** and two fixed sources:

- one BBC Sport football RSS feed
- one Sky Sports football RSS feed
- RSS-provided metadata and text only
- links back to the original publisher rather than republishing articles

The exact feed URLs, accessibility, terms, freshness and metadata quality must
be verified during Phase 0. Do not initially fetch article pages.

Do **not** start with user-managed sources, additional publishers, additional
sports, saved URLs, social media, video ingestion, mobile apps, arbitrary web
scraping or generative chat.

### Initial interest profile

The initial profile is deliberately small and typed:

- **Scope:** Football
- **Team:** Chelsea
- **Competition:** Premier League
- **Topics:** Transfers and injuries

Football defines the content domain and does not contribute to relevance
scoring. The candidate dataset may reveal taxonomy gaps, but it must not change
the user’s interests merely to fit the available articles.

## The actual AI feature

The Phase 0/1 AI feature is **structured content classification**, not a chatbot
wrapper. For each item, the model extracts teams, competitions, people and
topics from the supplied RSS metadata. It does not decide relevance.

### Structured output

```json
{
  "teams": ["Chelsea"],
  "competitions": ["Premier League"],
  "people": [],
  "topics": ["transfers"]
}
```

Use strict schema validation, explicit timeouts and at most one bounded retry.
Empty arrays are valid when the supplied metadata does not support a field.
Preserve the raw response, validated output, model identifier, token use,
latency and estimated cost so the result can be inspected.

Do not add model-generated relevance scores, recommendation reasons, summaries,
confidence values, near-duplicate detection or learned feedback in Phase 0/1.

### Baseline relevance ranking

Classification and ranking are separate concerns. Deterministic application code
gives one point for each extracted match with:

- Chelsea
- Premier League
- transfers
- injuries

All matches have equal weight. Sort by score descending and then publication
time descending; items without a publication time come last within their score.
Build the recommendation reason from the matched interests. This policy is a
deliberately simple baseline to evaluate and change later.

## MVP user journey

The first version is deliberately **single-user**. The BBC Sport and Sky Sports
football feeds and the initial interest profile are fixed in configuration.
There is no account system or source-management UI.

1. Poll the two configured feeds and ingest their latest items.
2. Preserve the source metadata and canonicalise each URL conservatively.
3. Normalise URLs and remove exact duplicates.
4. Extract structured teams, competitions, people and topics.
5. Rank the unread feed using the deterministic baseline and show its matched
   interests.
6. Open or mark an item read, and mark it worth seeing or not worth seeing.
7. Review whether the ranked feed saved attention compared with the two source
   feeds.

## MVP acceptance criteria

- Both configured feeds reliably produce recent items.
- The same canonical URL is never stored twice.
- Every ingested item receives a schema-valid basic classification or a visible
  fallback state.
- Relevance scores and recommendation reasons are derived deterministically from
  the configured interest profile.
- The first usable UI is relevance-ranked, not merely chronological.
- Feed items show which interests matched.
- Read items disappear from the default feed but remain accessible.
- Worth-seeing feedback can be recorded with one action.
- Over a one-week personal trial, record whether each surfaced story was **worth
  seeing**.
- The ranked feed either achieves **≥70% worth-seeing precision** or reduces the
  number of items scanned by **≥50%** compared with the combined source feeds.

User-managed sources, additional sports, summaries, semantic duplicate
detection, learned feedback, async workers, multi-user authentication, automated
deployment and full observability are **not MVP acceptance criteria**. They
become candidates only after the core loop shows value.

## Current-phase technical design

Only the architecture in this section is approved for Phase 0 and Phase 1.
Target-state sections later in this document are future direction, not current
requirements.

### Phase 0 stack

- **Runtime:** mise-pinned Python with `uv` and a project-local virtual
  environment
- **Interface:** small command-line programs; no web server or UI
- **Fetching and parsing:** `httpx` and `feedparser`
- **Validation:** Pydantic models and strict provider Structured Outputs
- **Storage:** immutable JSONL inputs and derived JSONL outputs
- **AI:** OpenAI Responses API through a small provider boundary
- **Model:** `gpt-6-luna`, verified against official OpenAI documentation on
  2026-09-30
- **Budget:** no more than $5 total Phase 0 API spend
- **Operations:** host-run only; no database, scheduler or application container

OpenAI documents `gpt-6-luna` as its efficient model for focused, high-volume
tasks. It supports the Responses API and Structured Outputs. At verification,
standard short-context pricing was $0.10 per million input tokens and $0.50 per
million output tokens. Record actual usage and revisit the choice if extraction
quality is inadequate or provider capabilities and pricing change.

### Phase 1 stack

- **App and UI:** host-run FastAPI + Pydantic with server-rendered HTML
- **Storage:** Dockerized PostgreSQL with SQLAlchemy and Alembic
- **Ingestion:** an explicit in-process command; add simple scheduling only if
  needed for the personal trial
- **AI:** the Phase 0 provider boundary and validated structured output
- **Operations:** Docker Compose for PostgreSQL only

PostgreSQL is justified by persistent feed state, uniqueness constraints and
transactions. It does not justify Redis, Celery, workers, queues, job tables or
distributed-system patterns.

### Conservative URL canonicalisation

Prefer a missed duplicate over incorrectly merging distinct articles.

- lowercase the scheme and host
- remove URL fragments
- remove default ports
- normalise empty paths and safe path syntax
- remove only well-known generic tracking parameters
- preserve unknown query parameters
- do not rewrite publisher-specific paths or identifiers without evidence that a
  rule is necessary and safe
- retain the original URL alongside the canonical URL

### Phase 1 pipeline

Start with the shortest pipeline that can test the central product hypothesis:

```text
Two fixed RSS feeds
  → fetch + parse
  → canonicalise URL
  → exact dedup
  → structured AI classification
  → deterministic relevance ranking
  → persist
  → unread feed
```

## Target-state architecture

Everything below this heading describes possible later phases. It must not be
implemented merely because it appears in the plan. Each component must solve an
observed problem or deliberately demonstrate an agreed engineering capability.

### Target pipeline

```text
Scheduler / URL webhook
  → ingestion job
  → fetch + parse
  → canonicalise URL + content hash
  → exact dedup
  → embedding / similarity candidate search
  → LLM structured classification + summary
  → relevance ranking
  → persist result
  → feed becomes available
```

### Possible target tables

- `users`
- `interest_profiles`
- `interest_rules`
- `sources`
- `source_poll_runs`
- `content_items`
- `content_entities`
- `story_clusters`
- `cluster_members`
- `ai_runs`
- `prompt_versions`
- `ingestion_jobs`
- `user_content_state`
- `feedback_events`

### Target-state modelling principles

- Separate immutable source content from per-user state.
- Store raw model output, validated output, model, prompt version, latency and
  token/cost data.
- Model job state explicitly:
  `queued → running → succeeded | retryable_failed | terminal_failed`.
- Use uniqueness constraints for source item IDs, canonical URLs and idempotency
  keys.
- Use transactions when a job changes state and persists its result.
- Start similarity search simply; introduce `pgvector` only after exact dedup
  works.

## Evaluation plan

### Phase 0 dataset

Fetch a candidate pool larger than the evaluation sample from both configured
feeds, targeting at least 50 recent items when the feeds make that possible.
Preserve every candidate as immutable JSONL before selecting approximately 30
representative real items.

Select naturally occurring examples of:

- Clearly relevant and irrelevant content
- Ambiguous relevance
- Obvious duplicate URL variants or duplicate stories, when present
- Multi-team and multi-topic articles
- Thin, metadata-poor or malformed items

Do not manufacture examples merely to balance the dataset. Ensure both sources
are represented. The user labels every selected item worth seeing or not worth
seeing before reviewing model output, with a short note for ambiguous cases.

### Phase 0 feasibility evidence

Phase 0 is a feasibility study, not a statistically meaningful model evaluation.
Record and review:

- feed availability, freshness and candidate volume
- RSS field coverage and whether most items are understandable without page
  extraction
- variation across teams, competitions, people, topics and likely relevance
- conservative canonicalisation examples and any uncertain cases
- schema-valid classification rate after at most one retry
- substantive extraction errors across representative and difficult items
- deterministic ranking output compared with the independent worth-seeing labels
- per-item and total latency, token use and estimated cost

Failures should first trigger a revision to the sources, taxonomy, prompt, model
or ingestion strategy. They are not automatically reasons to abandon the
product.

### Later evaluation dataset

After the taxonomy stabilises, expand to **150–250 items** and create a frozen
test split that is never used to tune prompts.

### Later evaluation targets

| Capability | Metric | Initial target |
| --- | --- | --- |
| **Product value** | Surfaced stories marked “worth seeing” | **≥ 70%** |
| **Attention saved** | Reduction in items scanned vs source feeds | **≥ 50%** |
| Entity extraction | Macro F1 | ≥ 0.85 |
| Relevant/not relevant | F1 | ≥ 0.85 |
| Ranked feed | nDCG@10 | ≥ 0.80 |
| Duplicate clustering | Pairwise F1 | ≥ 0.85 |
| Structured output | Valid response rate | ≥ 99% including retries |
| Reliability | Successful jobs | ≥ 99% excluding bad sources |
| Performance | p95 processing latency | < 30 seconds |

These are later-phase targets, not Phase 0 exit criteria. Track quality, latency
and estimated cost together. A cheaper model that is slightly less accurate may
be the right production choice.

### Later eval workflow

- Version prompts in the database or repository.
- Run every candidate against the frozen dataset.
- Compare against a deterministic keyword baseline.
- Review a confusion matrix and worst failures.
- Block deployment if key metrics regress beyond a defined threshold.

## Target-state reliability and observability

This section applies only after product validation and when asynchronous or
deployed processing has been justified.

Trace one item across scheduler → API/queue → worker → model → Postgres.
Include:

- Correlation and job IDs in every log
- Queue depth, job duration, retry count and failure rate
- Model latency, schema-validation failures, token use and estimated cost
- Feed/API p50 and p95 latency
- Dead-letter or terminal-failure view
- Health and readiness endpoints
- Alerts for stalled queues and ingestion failure spikes

### Target-state failure modes

- RSS source unavailable or malformed
- Duplicate delivery
- Extraction timeout
- Model timeout, rate limit or invalid JSON
- Worker crash after calling the model but before persistence
- Database migration failure
- Prompt change silently reducing relevance quality

## Phased plan

### Phase 0 — Feasibility spike (1–2 days)

1. [ ] **Verify the two feeds** — automated investigation plus user review of
       publisher terms. Record final URLs, redirects, status, format, freshness
       and permitted use. Success: both feeds are repeatably accessible and
       suitable for this metadata-only test.
2. [ ] **Collect the candidate pool** — automated. Preserve all available
       entries from both feeds as immutable JSONL, targeting at least 50 recent
       candidates when available. Success: both sources contribute reproducible
       records.
3. [ ] **Profile source quality** — automated report plus user review. Measure
       field coverage, text length, dates, URL quality, malformed entries,
       source balance and content diversity. Success: most items can be
       understood from RSS metadata.
4. [ ] **Define and test canonicalisation** — automated. Implement the
       conservative policy using examples from the pool. Success: obvious URL
       variants collapse without merging distinct known URLs.
5. [ ] **Select approximately 30 items** — automated shortlist plus user
       approval. Include representative and naturally occurring difficult cases.
       Success: both sources and the observed content range are represented
       without manufactured examples.
6. [ ] **Define the taxonomy and structured schema** — engineering proposal plus
       user approval. Specify normalisation for teams, competitions, people and
       topics. Success: the sample can be represented without adding fields that
       Phase 0/1 do not need.
7. [ ] **Create independent labels** — user task. Label each selected item worth
       seeing or not worth seeing before reviewing model output. Success: every
       sample item has an independent product-value label.
8. [ ] **Run structured classification** — automated. Classify the full sample,
       allow at most one retry and preserve raw and validated outputs, latency,
       usage and cost. Success: feasibility and failure modes can be assessed
       across ordinary and difficult items within the $5 limit.
9. [ ] **Apply deterministic ranking** — automated output plus user review.
       Score typed interest matches equally and use recency as the tie-breaker.
       Success: scores and reasons are reproducible and accurately reflect
       matched interests.
10. [ ] **Produce the feasibility report** — automated evidence plus a joint
        decision. Report all Phase 0 success criteria and recommend proceeding,
        revising or stopping.

**Phase 0 decision gate:** proceed to Phase 1 only when both feeds are usable,
most representative items contain enough permitted metadata, structured
classification is sensible across difficult examples, the source mix makes
personal filtering meaningful, conservative canonicalisation is safe, and
latency and cost fit expected feed volumes. Otherwise revise the relevant
source, taxonomy, model or ingestion assumption and repeat only the affected
task.

### Phase 1 — Test the central hypothesis (3–5 days)

- [ ] Build the thinnest end-to-end application
- [ ] Run FastAPI on the host and PostgreSQL in Docker
- [ ] Ingest the two fixed feeds into persistent storage
- [ ] Implement canonical URLs and exact deduplication
- [ ] Run validated structured classification during ingestion
- [ ] Rank items with the deterministic baseline
- [ ] Render a server-side unread feed with matched-interest reasons
- [ ] Add read, worth-seeing and not-worth-seeing actions

**Milestone:** use an AI-ranked feed personally. Do not proceed because the
pipeline works; proceed only if the feed shows signs of saving attention.

### Phase 2 — Validate usefulness and improve the model

- [ ] Use the feed for one week
- [ ] Continue labelling surfaced items as worth seeing / not worth seeing
- [ ] Measure worth-seeing precision and reduction in items scanned
- [ ] Refine the taxonomy based on actual classification failures
- [ ] Compare one LLM approach with a deterministic keyword baseline
- [ ] Improve the ranking query and recommendation reasons
- [ ] Add save, dismiss, “more like this” and “less like this” only if the
      initial feedback loop demonstrates a need

**Decision gate:** target ≥70% worth-seeing precision or ≥50% fewer items
scanned. If neither improves over the baseline, revise the product or ranking
approach before adding infrastructure.

### Phase 3 — Strengthen the product loop

- [ ] Add near-duplicate story clustering
- [ ] Add summaries only where they improve the experience
- [ ] Separate immutable source content from per-user state
- [ ] Add prompt/model metadata and a lightweight eval runner
- [ ] Add the save-URL endpoint only if it is useful in practice
- [ ] Consider additional publishers or sports only after the football loop is
      useful

**Milestone:** the feed is useful enough to use repeatedly, and failures are
understood with real examples.

### Phase 4 — Production workflow engineering

- [ ] Introduce Celery + Redis because ingestion now needs isolation, retries or
      concurrency
- [ ] Add explicit job states, idempotency and bounded retries
- [ ] Add scheduled polling and a failure/replay view
- [ ] Write integration tests for duplicate delivery, timeouts and worker
      crashes
- [ ] Add structured logs, correlation IDs and basic traces

**Milestone:** the proven product loop is asynchronous, observable and safe to
retry.

### Phase 5 — Evaluation, deployment and reliability

- [ ] Expand the labelled set to 150–250 items after the taxonomy stabilises
- [ ] Create a frozen test split and compare prompt/model versions
- [ ] Add OpenTelemetry traces, dashboards and error reporting
- [ ] Deploy staging with Docker, Terraform and GCP
- [ ] Add GitHub Actions tests, migrations and deployment
- [ ] Add backups, secrets handling, health checks and a runbook

**Milestone:** measured AI quality and a deployed system showing deliberate
production-engineering decisions.

### Phase 6 — Portfolio polish and external testing

- [ ] Tighten UX and accessibility
- [ ] Invite 3–5 testers and compare their worth-seeing rates
- [ ] Record a 3-minute demo
- [ ] Publish an architecture diagram, eval report and trade-off notes
- [ ] Write a case study: hypothesis → first loop → evidence → failures →
      hardening
- [ ] Add screenshots and a one-command local setup

**Milestone:** an interviewer can understand both the product evidence and the
engineering depth in five minutes.

## Testing strategy

Testing grows with the phase rather than anticipating later infrastructure.

### Phase 0 tests

- Unit tests for conservative URL canonicalisation and deterministic ranking
- Contract tests for RSS parsing against preserved source fixtures
- Schema-validation tests for model output, including empty fields and failures
- A small provider integration check before classifying the full sample

### Phase 1 tests

- Unit tests for canonicalisation, ranking and state transitions
- PostgreSQL integration tests for uniqueness, persistence and state changes
- End-to-end test for feed ingestion → classification → feed → feedback
- Failure tests for source errors, model timeouts and invalid model output

### Later-phase tests

- Property-based tests for canonicalisation and idempotency when the rules grow
- Queue and worker integration tests only after async processing exists
- Failure-injection tests for retries and worker restarts only after Phase 4
- Migration tests against a copy of the previous schema before deployment

## Keep the scope under control

### Must have for product validation

- Two fixed football RSS feeds from BBC Sport and Sky Sports
- Exact URL deduplication
- Structured classification of teams, competitions, people and topics
- Deterministic ranking against the fixed typed interest profile
- Unread/read state and matched-interest reasons
- Worth-seeing / not-worth-seeing feedback
- A measured product-value result

### Add after validation for engineering depth

- Near-duplicate story clustering
- Summaries where evidence shows they improve the product
- User-managed sources and additional sports
- Async jobs, retries and explicit failure states
- A larger evaluation suite
- Traces, logs and failure dashboards
- Staging deployment, CI/CD and Terraform

### Nice to have

- Daily email digest
- Browser extension
- Multiple users
- More sophisticated learning-to-rank
- Mobile layout polish

### Explicitly later

- Social media integrations
- Native mobile app
- Full-article republishing
- General-purpose chat
- Supporting every content type and sport
- Building a custom foundation model

## Portfolio narrative

> I built an AI content-triage system that turns two noisy football feeds into a
> personalised unread queue. I first proved that structured classification and
> deterministic ranking saved attention, then added engineering depth only where
> observed product and operational needs justified it.

Evidence to show:

1. Live product and 3-minute demo
2. Architecture diagram and sequence diagram
3. Postgres schema with indexing decisions and `EXPLAIN ANALYZE`
4. Prompt/model evaluation report and labelled dataset methodology
5. Trace of one item through the complete system
6. CI/CD and Terraform structure
7. Short postmortem of a real failure or poor model result

## First three actions

- [ ] Verify the BBC Sport and Sky Sports football feeds and their permitted
      metadata use.
- [ ] Collect and profile a larger candidate pool from both feeds.
- [ ] Select approximately 30 representative items and complete the Phase 0
      feasibility tasks before beginning the application.

**Recommended working rule:** do not add a feature unless it strengthens either
the user value or the current phase’s learning goal. If it does neither, cut it.
