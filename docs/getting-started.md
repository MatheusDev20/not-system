# RelayForge — Where to Begin

Start the way *Fundamentals of Software Architecture* does. Before you pick a style or a technology, write down what the system has to be good at, and give the reasons. The book's "Second Law" says *why* matters more than *how*.

## Step 0: Write the kata

RelayForge is fictional, so you are also the stakeholder. Write a one-page **architecture kata** (the book's format) with four parts:

- **Users**: who calls the API, how many tenants, how big they are
- **Requirements**: what the system does (functional)
- **Context**: constraints such as team size, budget, deadline and hosting
- **Business context**: what failure costs (a lost notification vs. a duplicate one)

Put **numbers** on every assumption, even made-up ones, such as "50 tenants, peak 10k msgs/burst, 1M/day". Characteristics you can't measure can't be validated later.

## Step 1: Ask the questions that reveal architecture characteristics (Ch. 4–5)

### Data and correctness
- Can a notification ever be lost after we return `202 Accepted`? (durability → *when* do we persist?)
- Which is worse, a duplicate send or a missed one? (at-least-once vs. at-most-once → idempotency keys)
- What exactly makes two requests "the same"? Who supplies the idempotency key, and how long is it remembered?
- What states can a notification be in? Draw the lifecycle (accepted → queued → processing → delivered / failed / retrying / dead).

### Multi-tenancy
- What does "isolate the customer's data" mean: separate DBs, separate schemas, or a `tenant_id` column? What happens if it's violated?
- Can one tenant's 10k burst slow everyone else down? (the noisy-neighbor problem → fairness and rate limits)

### Scale and performance
- What is the expected throughput, both average and peak? What latency matters: API response time, or time to delivery?
- Is the load bursty or steady? (Bursty points toward elasticity, not just scalability.)

### Failure
- Which failures are *transient* and which are *permanent*? How many retries, with what backoff, and what happens after the last one?
- If the worker crashes mid-delivery, what happens to that message?

### Observability
- What does "expose delivery outcomes" mean: polling an endpoint, webhooks, or both? How fresh must the data be?

## Step 2: Pick the top 3 characteristics

The book warns against trying to support everything. Initial guess: **reliability/durability, data integrity (idempotency), and tenant isolation**, with elasticity close behind. Write down which ones you are explicitly *not* optimizing yet. That list is part of the architecture.

## Step 3: Find components before choosing a style (Ch. 8)

Use the actor/actions or workflow approach. Walk through "tenant submits 10k messages" and name the responsibilities, for example:

- Intake/API
- Tenant & Auth
- Notification Store
- Dispatch/Queue
- Delivery Worker
- Retry Policy
- Outcome Tracking

Then ask whether these components need *different* characteristics. If intake needs low latency and delivery needs elasticity, that's a hint toward separate **architecture quanta** (Ch. 7). If they don't differ, one quantum is enough.

## Step 4: Choose a style and record it as an ADR (Ch. 19–20)

Given "only add complexity when justified", a strong first ADR is: **a modular monolith with an async worker, backed by one database used as the queue** (e.g. Postgres with `SELECT ... FOR UPDATE SKIP LOCKED`).

Then write down what *measured* problem would justify moving to a real broker or splitting services. That trigger condition is exactly the kind of reasoning the book wants you to practice.

## Step 5: Turn characteristics into fitness functions (Ch. 6)

- A test proves tenant A can never read tenant B's notifications.
- Submitting the same idempotency key twice results in one delivery.
- Killing the worker mid-batch loses nothing.
- An architecture test proves the API module doesn't import worker internals.

## Suggested repo layout

```
docs/
  kata.md                 # step 0
  characteristics.md      # steps 1–2, with the top 3 and the "not now" list
  components.md           # step 3
  adr/0001-*.md           # step 4 onward
  diagrams/
```

## First milestone: walking skeleton

submit → persist → worker picks up → simulated delivery → outcome queryable

Once that works, add retries, then idempotency, then tenant isolation, with an ADR each time a decision has real trade-offs.
