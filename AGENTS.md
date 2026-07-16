# Student Affairs Agent Project Guidance

## Mission

This repository is a production-oriented university student affairs AI Agent system. It is not a tutorial rewrite and must not inherit architecture from `shopkeeper_brain` without an explicit fit assessment.

The target is one independently deployed instance per university, serving students and authorized service handlers across campuses, colleges, and administrative departments.

## Non-Negotiable Principles

- Design for real university deployment, operation, audit, recovery, and future integration.
- Local Docker Compose may reduce replica counts, but must preserve production interfaces, state semantics, authorization, idempotency, and failure behavior.
- `ServiceCase` is the business source of truth. Conversation history, vector data, task queues, and LangGraph checkpoints are projections or execution state.
- LLM output is untrusted input to deterministic business command handlers.
- High-risk decisions such as financial-aid approval, policy conflicts, and exceptions require an authorized human decision.
- Authorization is enforced by backend application services and repositories before data retrieval or tool execution.
- Sensitive student data must be classified, minimized, redacted, and routed only to approved model endpoints.
- External side effects require authorization, idempotency keys, audit records, timeouts, and explicit result handling.
- Do not expose prompts, raw traces, token usage, vector scores, stack traces, or internal node names to students or service handlers.

## Project Boundaries

- `shopkeeper_brain` is a read-only reference for selected document processing, embedding, retrieval, reranking, tracing, and evaluation techniques.
- Do not copy product-oriented state, `item_name` identity, prompts, in-memory task registries, or FastAPI `BackgroundTasks` long-running workflows.
- Neo4j is not part of the initial architecture. Add graph storage only after evaluation demonstrates stable value over relational policy structures and hybrid retrieval.

## Development Environment

- Windows development root: `D:\agent\student-affairs-agent`
- Conda environment: `D:\conda_envs\student-affairs-agent`
- Python: 3.11, CPU-compatible by default
- Local infrastructure: Docker Compose with data under `D:\Programs\DockerData\student-affairs-agent`
- Production target: university-controlled private cloud or data center, with independently scalable stateless services and highly available data services

## Change Discipline

Before implementation:

1. Read the current design specification and relevant ADRs.
2. Confirm the GitHub issue and acceptance criteria.
3. Identify the owning module and interface contract.
4. Add or update tests before changing behavior.

After implementation:

1. Run focused tests and relevant integration tests.
2. Inspect the diff for unrelated changes and secrets.
3. Update architecture, domain, operational, or API documentation when contracts change.
4. Record deferred production risks explicitly; do not hide them behind local-environment assumptions.

Never commit real credentials, student data, policy documents with restricted access, or model traces containing unredacted personal information.
