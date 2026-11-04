# Herald — Roadmap

Herald is a fictional multi-tenant notification platform. Organizations submit notifications through an API, and Herald accepts them, processes them asynchronously, delivers them through a fake provider, retries failures and reports outcomes.

The goal is to practice architecture, not to build a polished SaaS. Add complexity only when a requirement or a measured problem justifies it.

## What "practicing architecture" means here

Most tasks below follow the same loop:

1. **Question.** What problem or risk am I addressing?
2. **Options.** What are at least two ways to address it?
3. **Decision.** Which option do I pick, and what am I giving up? (the trade-off)
4. **Evidence.** What test, measurement or experiment shows the decision holds?
5. **Record.** Write it down (brief, ADR or experiment note) and commit it.

A coder asks "how do I build X?" An architect asks "why X instead of Y, and what does X cost us?" Every task produces a written artifact so you can't skip the "why".

**Done check for every task.** Answer these out loud without notes: What problem did I address? What options did I consider? Why this one? What could break? What evidence supports it?

**Working rule.** You design first, and AI reviews afterward. Tangential ideas go in `later.md`.

**Stack.** TypeScript, NestJS, PostgreSQL and Docker locally. A queue and AWS come in only when a task justifies them.

---

## Phase A — Understand the problem (no code)

### Task 1: Actors and user journey
- **Concept.** Actors and actions. Architecturally significant vs. insignificant behavior.
- **Do.** List the actors (tenant admin, tenant's backend system, recipient, Herald operator…). Walk through the core journey: a tenant's system submits notifications via the API and later checks their outcome. Include the failure paths: the same request sent twice, a transient provider failure, a permanent failure, a 10k burst.
- **Output.** `docs/user-journey.md`

### Task 2: Functional requirements and scope
- **Concept.** Functional vs. non-functional requirements. Scope is itself a decision.
- **Do.** Turn the journey into "the system must…" statements. Split them into **V1**:
  - create a tenant
  - submit a notification
  - store it with a status
  - deliver it through a fake adapter
  - query its status and history

  and **later** (scheduling, real channels, quotas/plans, UI…), with a reason for each deferral.
- **Output.** A requirements section (goes into the brief in Task 4)

### Task 3: Architecture characteristics
- **Concept.** Quality attributes, measurable scenarios, choosing the top 3, and the cost of supporting everything.
- **Do.** Answer the questions below, pick 5+ characteristics, and turn each into a measurable scenario. Example: "under 1k req/s, 99% of accepted notifications enter processing within 5s". Choose your **top 3**, and write a **"not optimizing now"** list.
- **Questions to answer:**
  - Can a notification be lost after `202 Accepted`? When do we persist?
  - Which is worse, a duplicate send or a missed one? What makes two requests "the same"?
  - What states can a notification be in? (accepted → queued → processing → delivered / failed / retrying / dead)
  - What does tenant isolation mean: separate DBs, separate schemas, or a `tenant_id` column?
  - Can one tenant's burst slow everyone else down?
  - What is the expected throughput (average and peak)? Which latency matters: the API response or the delivery?
  - Which failures are transient and which are permanent? How many retries, and what happens after the last one?
  - If the worker crashes mid-delivery, what happens to the message?
- **Output.** A characteristics section (goes into the brief)

### Task 4: Architecture brief
- **Concept.** Constraints (fixed facts) vs. assumptions (guesses you may need to revisit). Trade-offs vs. technology preferences.
- **Do.** Combine Tasks 1–3, then add:
  - constraints: solo developer, limited budget, fake provider
  - assumptions, with numbers (e.g. 50 tenants, 1M msgs/day, 10k peak burst)
  - three real trade-offs
- **Output.** `docs/architecture-brief.md`

## Phase B — Shape the system (design, a little code)

### Task 5: Components and module map
- **Concept.** Coupling, cohesion, boundaries, dependency direction, architecture quanta.
- **Do.** Walk through "tenant submits 10k messages" and name the components (Intake, Tenants, Notifications, Delivery, Retry, Reporting…). Define each one's responsibility, the data it owns, and which modules it may depend on. Ask whether any component needs *different* characteristics from the others. If one does, that suggests a separate quantum.
- **Output.** `docs/components.md` with a boxes-and-arrows sketch. This is the first drawing.

### Task 6: Choose an architecture style
- **Concept.** Layered, hexagonal, modular monolith and microservices, with their costs. ADRs. Technical vs. architectural decisions.
- **Do.** Compare at least 3 styles against your top-3 characteristics, and pick one. The expected starting point is a modular monolith plus an async worker, but you have to justify it. State what measured problem would make you change it.
- **Output.** `docs/adr/0001-architecture-style.md`

### Task 7: Data model and notification lifecycle
- **Concept.** Data ownership, explicit state transitions, transaction boundaries.
- **Do.** Draw the notification state machine and list the allowed transitions. Sketch the tables, including where `tenant_id` lives and who owns each table. Decide what must happen atomically.
- **Output.** `docs/data-model.md`

### Task 8: C4 diagrams
- **Concept.** Communicating architecture at different zoom levels.
- **Do.** Draw a context diagram (Herald and its actors and external systems) and a container diagram (API, worker, DB…). A component diagram for one container is optional.
- **Output.** `docs/diagrams/`

## Phase C — Build and prove

Each task here adds a capability, includes a test or experiment that proves it, and gets an ADR whenever the decision has real trade-offs.

### Task 9: Walking skeleton
- **Concept.** Ports and adapters. Fitness functions (tests that enforce architecture).
- **Do.** Create a tenant, submit a notification, persist it, run the fake delivery, and query the status. Keep it synchronous for now. Put the delivery provider behind an interface.
- **Proof.** An architecture test showing modules respect your dependency rules, e.g. the API doesn't import delivery internals.

### Task 10: Go asynchronous
- **Concept.** Commands vs. events, sync vs. async, eventual consistency, event contracts.
- **Do.** Return `202 Accepted` and process in a worker. First decision (ADR): use Postgres as the queue (`SKIP LOCKED`) or a real broker. Document the message/event contract.
- **Proof.** Submissions stay fast while delivery is slow.

### Task 11: Failure handling
- **Concept.** Timeouts, retries with backoff, dead-letter, partial failure.
- **Do.** Make the fake provider fail randomly or on demand. Add a retry policy and a dead state.
- **Proof.** Killing the worker mid-batch loses nothing. `docs/experiments/` documents what happened.

### Task 12: Idempotency and concurrency
- **Concept.** At-least-once vs. at-most-once delivery, idempotency keys, isolation levels, the outbox pattern.
- **Do.** Add idempotency keys. Make sure two workers can't process the same message.
- **Proof.** The same key sent twice produces one delivery, verified by a concurrency test.

### Task 13: Security and tenant isolation
- **Concept.** Authentication and authorization, least privilege, rate limiting, noisy neighbors, threat modeling.
- **Do.** Add API-key auth per tenant, tenant scoping on every query, and per-tenant rate limits.
- **Proof.** A test shows tenant A can never read tenant B's data. Also produce a short threat model.

### Task 14: Observability
- **Concept.** SLIs, SLOs, structured logs, metrics, tracing.
- **Do.** Turn the Task 3 scenarios into SLIs/SLOs and instrument them.
- **Proof.** Run a failure experiment and show you can see it on the metrics.

### Task 15: Performance and scaling
- **Concept.** Throughput vs. latency, bottlenecks, scalability vs. elasticity.
- **Do.** Load test the 10k burst, find the bottleneck and fix one thing.
- **Proof.** Before and after numbers.

### Task 16: Deployment and cost
- **Concept.** Infrastructure as an architectural decision, cost as a characteristic.
- **Do.** Deploy to AWS with the smallest setup that meets your targets. Produce a deployment diagram and a monthly cost estimate.

### Task 17: Final review
- **Concept.** Evolutionary architecture, defending trade-offs.
- **Do.** Revisit the brief: which assumptions were wrong, and what would you change? Give a 15-minute design defense.
- **Output.** `docs/final-review.md`

## Optional extensions (only after Task 12)

- Scheduled sending
- Real channels (email/SMS) behind the existing adapter
- Quotas and plans (a good concurrency exercise)

## Repo layout

```
docs/
  user-journey.md
  architecture-brief.md
  components.md
  data-model.md
  adr/
  diagrams/
  experiments/
  final-review.md
later.md
src/
```
