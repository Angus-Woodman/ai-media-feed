# Signal: AI-Powered Personal Sports Media Inbox

**Decision:** Build Signal as an AI-powered personal sports media inbox.

It ingests articles and other content from user-chosen sources, removes
duplicates, tags and scores each item against the user’s interests, and presents
an unread-first feed that gets better from explicit feedback.

## Build order

**Prove the product before proving the infrastructure.**

The first goal is five real feeds → ingestion → exact deduplication → basic AI
classification → a feed that is noticeably more useful than reading the source
feeds. Start with the simplest architecture that can test that loop. Add queues,
tracing, Terraform and production hardening only after using the ranked feed
reveals genuine value.

## Product concept

### One-line pitch

**A personalised, unread-first sports briefing that turns noisy feeds into a
small queue of genuinely relevant stories.**

### Target user

A sports fan who follows several teams, competitions, players and topics but
does not want to check many sites or repeatedly see the same story.

### Core job to be done

> “Show me the new sports content I’m likely to care about, explain why it
> matches my interests, and hide repeats and things I’ve already consumed.”

### Opinionated scope

Start with **football and Formula 1**, or two sports personally followed.
Support only:

- User-added RSS feeds
- A “save URL” endpoint or a simple browser bookmarklet
- Article metadata and short extracted text where permitted
- Links back to the original publisher rather than republishing full articles

Do **not** start with every sport, social media, video ingestion, mobile apps,
automatic scraping of arbitrary sites, or generative chat.

## The actual AI feature

The AI is a **content-triage pipeline**, not a chatbot wrapper.

For each item:

1. Extract structured entities: sport, competition, team, athlete and topic.
2. Classify it against the user’s interest profile.
3. Produce a relevance score and short, inspectable reason.
4. Detect near-duplicate coverage of the same underlying story.
5. Create a compact summary using only the supplied text.
6. Learn from “more like this”, “less like this”, read, saved and dismissed
   feedback.

### Structured output

```json
{
  "entities": {
    "sport": "football",
    "competition": ["Premier League"],
    "teams": ["Arsenal"],
    "people": []
  },
  "topics": ["transfer"],
  "relevance_score": 0.86,
  "reason": "Matches Arsenal and transfer-news interests",
  "summary": "...",
  "confidence": 0.91
}
```

Use schema validation, bounded retries, prompt versions, model metadata,
timeouts and a deterministic fallback classifier.

## MVP user journey

The first version is deliberately **single-user**. Five publisher RSS feeds are
configured globally; there is no account system or source-management UI yet.

1. Configure a simple interest profile: teams, competitions, athletes and
   topics.
2. Poll five real feeds and ingest their latest items.
3. Normalise URLs and remove exact duplicates.
4. Run a basic structured AI classification as part of the initial ingestion
   flow.
5. Rank the unread feed by relevance and show a short reason for each
   recommendation.
6. Open, mark read, save, dismiss, or choose “more like this” / “less like
   this”.
7. Review which surfaced stories were actually worth seeing.

## MVP acceptance criteria

- Five real feeds reliably produce new items.
- The same canonical URL is never stored twice.
- Every ingested item receives a schema-valid basic classification or a visible
  fallback state.
- The first usable UI is AI-ranked—not merely chronological.
- Feed items show why they were recommended.
- Read items disappear from the default feed but remain accessible.
- Feedback can be recorded with one action.
- Over a one-week personal trial, record whether each surfaced story was **worth
  seeing**.
- The ranked feed either achieves **≥70% worth-seeing precision** or reduces the
  number of items scanned by **≥50%** compared with the combined source feeds.

Async workers, multi-user authentication, story clustering, automated deployment
and full observability are **not MVP acceptance criteria**. They become
justified engineering work after the core loop shows value.

## Technical design

### Progressive stack

#### Phase 0–1: minimum product-test stack

- **App:** FastAPI + Pydantic, with server-rendered HTML or the thinnest
  practical Nuxt UI
- **Storage:** PostgreSQL if setup is already comfortable; otherwise SQLite for
  the spike, followed by an explicit migration decision
- **Ingestion:** an in-process command or scheduled task
- **AI:** one provider adapter with structured outputs
- **Operations:** local Docker Compose only if it makes setup faster

#### After the feed proves useful

- Nuxt 3 + TypeScript for a richer product UI
- PostgreSQL + SQLAlchemy + Alembic as the production data layer
- Celery + Redis when processing needs isolation, retries or concurrency
- OpenTelemetry, structured logs and error reporting
- Docker + Terraform on GCP
- GitHub Actions with tests, migrations and staging deployment

Choose familiar tools for the product surface. Each infrastructure component
must solve an observed problem or create deliberate portfolio evidence; it
should not block testing the product hypothesis.

### Phase 1 pipeline

Start with the shortest pipeline that can test the central product hypothesis:

RSS feeds
  → fetch + parse
  → canonicalise URL
  → exact dedup
  → structured AI classification
  → relevance ranking
  → persist
  → unread feed

### Target Pipeline

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

### Core tables

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

### Important modelling decisions

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

### Grow the dataset in stages

Start with **30–50 representative items**. Use them to discover whether the
taxonomy, relevance scale and duplicate definition are actually useful before
investing in a large labelling exercise.

Include:

- Clearly relevant and irrelevant content
- Ambiguous relevance
- Duplicate and near-duplicate stories
- Multi-team or multi-sport articles
- Thin, clickbait and malformed items

Initially label only what the first product loop needs: entities, topics and
whether the item was worth seeing. Add relevance grades and duplicate-cluster
IDs when those features are implemented.

After the taxonomy stabilises, expand to **150–250 items** and create a frozen
test split that is never used to tune prompts.

### Metrics

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

Track quality, latency and estimated cost together. A cheaper model that is
slightly less accurate may be the right production choice.

### Eval workflow

- Version prompts in the database or repository.
- Run every candidate against the frozen dataset.
- Compare against a deterministic keyword baseline.
- Review a confusion matrix and worst failures.
- Block deployment if key metrics regress beyond a defined threshold.

## Reliability and observability

Trace one item across scheduler → API/queue → worker → model → Postgres.
Include:

- Correlation and job IDs in every log
- Queue depth, job duration, retry count and failure rate
- Model latency, schema-validation failures, token use and estimated cost
- Feed/API p50 and p95 latency
- Dead-letter or terminal-failure view
- Health and readiness endpoints
- Alerts for stalled queues and ingestion failure spikes

### Failure modes to design for

- RSS source unavailable or malformed
- Duplicate delivery
- Extraction timeout
- Model timeout, rate limit or invalid JSON
- Worker crash after calling the model but before persistence
- Database migration failure
- Prompt change silently reducing relevance quality

## Six-phase plan

### Phase 0 — Feasibility spike (1–2 days)

- [ ] Choose the initial sport/domain and five real RSS feeds
- [ ] Collect 30 sample items
- [ ] Prototype parsing, canonical URLs and one structured LLM call
- [ ] Confirm that useful metadata/text can be obtained without arbitrary
      scraping
- [ ] Write down the minimum taxonomy and a simple personal interest profile

**Kill criterion:** if reliable, permitted source content is unavailable, switch
to user-submitted links and publisher-provided RSS metadata only.

### Phase 1 — Test the central hypothesis (3–5 days)

- [ ] Build the thinnest end-to-end application
- [ ] Ingest five feeds into simple persistent storage
- [ ] Implement canonical URLs and exact deduplication
- [ ] Run basic structured AI classification during ingestion
- [ ] Rank items against the configured interest profile
- [ ] Render an unread feed with recommendation reasons
- [ ] Add read, worth-seeing and not-worth-seeing actions

**Milestone:** use an AI-ranked feed personally. Do not proceed because the
pipeline works; proceed only if the feed shows signs of saving attention.

### Phase 2 — Validate usefulness and improve the model

- [ ] Use the feed for one week
- [ ] Label 30–50 real items as worth seeing / not worth seeing
- [ ] Measure worth-seeing precision and reduction in items scanned
- [ ] Refine the taxonomy based on actual classification failures
- [ ] Compare one LLM approach with a deterministic keyword baseline
- [ ] Add read, save, dismiss, “more like this” and “less like this” feedback
- [ ] Improve the ranking query and recommendation reasons

**Decision gate:** target ≥70% worth-seeing precision or ≥50% fewer items
scanned. If neither improves over the baseline, revise the product or ranking
approach before adding infrastructure.

### Phase 3 — Strengthen the product loop

- [ ] Add near-duplicate story clustering
- [ ] Add summaries only where they improve the experience
- [ ] Introduce a proper Postgres schema and migrations if not already used
- [ ] Separate immutable source content from per-user state
- [ ] Add prompt/model metadata and a lightweight eval runner
- [ ] Add the save-URL endpoint only if it is useful in practice

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

- Unit tests for canonicalisation, ranking and state transitions
- Contract tests for RSS parsers and model adapters
- Integration tests with real Postgres and Redis
- End-to-end test for source → worker → feed → feedback
- Property-based tests for URL canonicalisation and idempotency
- Failure-injection tests for timeouts, invalid model output and worker restarts
- Migration test against a copy of the previous schema

## Keep the scope under control

### Must have for product validation

- Five real RSS feeds
- Exact URL deduplication
- Basic structured classification and relevance ranking
- A simple interest profile
- Unread/read state and recommendation reasons
- Worth-seeing / not-worth-seeing feedback
- A measured product-value result

### Add after validation for engineering depth

- Near-duplicate story clustering
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

> I built an AI content-triage system that turns noisy sports feeds into a
> personalised unread queue. The interesting work was not the summary prompt: it
> was designing a measurable classification and deduplication pipeline,
> idempotent asynchronous processing, production data models, failure handling,
> end-to-end tracing and repeatable cloud deployment.

Evidence to show:

1. Live product and 3-minute demo
2. Architecture diagram and sequence diagram
3. Postgres schema with indexing decisions and `EXPLAIN ANALYZE`
4. Prompt/model evaluation report and labelled dataset methodology
5. Trace of one item through the complete system
6. CI/CD and Terraform structure
7. Short postmortem of a real failure or poor model result

## First three actions

- [ ] Decide the initial sports/domain and select five permitted RSS sources.
- [ ] Create a 30-item feasibility dataset and test extraction, canonicalisation
      and structured classification.
- [ ] Write the MVP README with the acceptance criteria above before
      implementing the UI.

**Recommended working rule:** do not add a feature unless it strengthens either
the user value or one of the five learning goals. If it does neither, cut it.
