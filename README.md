# Signal

Signal is an AI-powered personal football media inbox. It will combine football
articles from BBC Sport and Sky Sports into a personalised, unread-first feed.

The project is being built to answer one question: can structured AI
classification and simple, transparent ranking make these feeds more useful
than reading them separately?

## Initial scope

- Football content from two fixed RSS feeds
- RSS-provided metadata and text only
- Conservative URL canonicalisation and exact deduplication
- Structured extraction of teams, competitions, people and topics
- Deterministic ranking against an explicit interest profile
- Read state and worth-seeing feedback

The initial interests are Chelsea, the Premier League, transfers and injuries.
Each match contributes equally to relevance; publication time breaks ties.

## Project status

The project is currently at **Phase 0: feasibility spike**. This phase will
verify the feeds, collect and profile a representative dataset, test structured
classification, and measure latency and cost before an application is built.

No working product or setup instructions are available yet.

## Technical direction

Phase 0 uses small host-run Python command-line tools and JSONL files. The first
product version will use server-rendered FastAPI with Dockerized PostgreSQL.
Queues, workers and production infrastructure are deliberately deferred until
the product demonstrates value.

See [the project plan](docs/project-plan.md) for the complete scope, acceptance
criteria, phased implementation plan and target-state architecture.
