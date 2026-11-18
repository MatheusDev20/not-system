# Herald — Functional Requirements and Scope

Derived from [user-journey.md](user-journey.md). Each requirement describes observable behavior (trigger → response), not how it is built. This section will be merged into `architecture-brief.md` in Task 4.

Phrasing (EARS style):

- `The system must <behavior>.`
- `When <trigger>, the system must <response>.`
- `If <unwanted condition>, then the system must <response>.`

## V1

| ID | The system must… | Source (journey) | Notes |
|----|------------------|------------------|-------|
| FR-01 | The system must accept a batch submitted as JSON, containing an idempotency key, one channel, and a list of messages (recipient + payload). | §1 | |
| FR-02 | When a valid batch is submitted, the system must respond with accepted and the batch ID without waiting for delivery. | §1 | |
| FR-03 | If a message fails channel validation after acceptance, then the system must mark only that message as failed, with a reason, and continue processing the rest of the batch. | §1, messages that break channel rules | |
| FR-04 | If the request is missing required fields, uses an unsupported channel, or exceeds the row limit, then the system must return 400 and store nothing. | §1, invalid request | |
| FR-05 | The system must record a status for every accepted message, from acceptance until it ends as delivered or failed. | §2, every message has a status | |
| FR-06 | When the tenant requests a batch by its ID, the system must return the batch progress (pending, delivered, failed) and each message's status and delivery attempts, including the failure reason. | §2 | Roadmap V1: "query its status and history". |
| FR-07 | The system must allow an operator to create a tenant and must issue it credentials. | Roadmap V1 scope | The journey has no onboarding step. |
| FR-08 | The system must associate every batch with the tenant whose credentials submitted it, and must return a batch only to that tenant. | §1, §2 | |
| FR-09 | When a tenant submits a batch with an idempotency key it has already used, the system must return the original batch ID and create no new messages. | §1, same batch twice | Keys are scoped per tenant. How long a key is remembered is an open question. |
| FR-10 | The system must deliver each valid message through the adapter for its batch's channel and record the outcome: delivered, transient failure or permanent failure. | Definitions, channel adapter | |
| FR-11 | The system must provide a `fake` channel that accepts any recipient and can be made to fail on demand, transiently or permanently. | V1 scope | |
| FR-12 | If a delivery attempt fails transiently, then the system must retry it according to the retry policy. | §1, can't be delivered | Attempts and backoff are set in Task 3. |
| FR-13 | If a delivery attempt fails permanently, then the system must mark the message as failed with the reason and must not retry it. | §1, can't be delivered | |
| FR-14 | If a message has used all attempts allowed by the retry policy, then the system must mark it as failed with the last failure reason. | §1, maximum retries | Also covers a provider outage that outlasts the retry policy. |
| FR-15 | If a message stays in processing longer than the stuck timeout, then the system must detect and recover it so that it still ends as delivered or failed. | §2, lost without a status | Timeout set in Task 3. How to recover is decided later. |

## Later

| ID | The system must… | Source (journey) | Reason for deferral |
|----|------------------|------------------|---------------------|
| FR-L01 | The system must accept a batch submitted as a CSV file. | Later, CSV entry point | Needs file storage and asynchronous parsing. JSON covers V1. |
| FR-L02 | The system must deliver messages through WhatsApp using pre-approved templates with variables. | Later, WhatsApp channel | Needs a real provider and template approval. The fake channel is enough to build and test the core. |
| FR-L03 | The system must deliver messages through email. | Later, second real channel | Only useful after the first real channel. Adding it later checks that only the adapter changes, not the core. |
| FR-L04 | The system must accept a batch whose messages use different channels. | Later, mixed-channel batches | V1 batches have exactly one channel, which keeps validation and progress simple. |
| FR-L05 | The system must send a batch at a time the tenant schedules. | Roadmap, optional extensions | Not part of the core journey. The roadmap only allows it after Task 12. |
| FR-L06 | The system must enforce message quotas per tenant plan. | Roadmap, optional extensions | No plans or billing in V1. The roadmap only allows it after Task 12. |
| FR-L07 | The system must let a company sign up and create its own tenant. | FR-07 | Operator-created tenants are enough while there are few tenants. |
| FR-L08 | The system must provide a UI for tenants to submit and track batches. | Roadmap, Task 2 | Tenants are backend systems, and the API covers the whole journey. |

## Open questions

Questions that block a requirement from being written precisely.

- Which checks run before accept (FR-04) and which after (FR-03)? E.g. request shape before, per-message channel rules after.
- How does the tenant see messages that failed validation: as normal failed messages with a reason, or as a separate list? (FR-03, FR-06)
- Retry policy numbers: how many attempts, what backoff, and how long counts as an "extended" outage? (FR-12, FR-14; Task 3)
- How long is an idempotency key remembered? (FR-09)
- What counts as "stuck in processing", and after how long? (FR-15; Task 3)

## Parked for Task 3 (non-functional)

Quality attributes that came up while writing functional requirements.

- Durability: an accepted message must never be lost, even if the server or worker crashes. (Relates to FR-02 and FR-05.)
- Security / tenant isolation: one tenant must never read or affect another tenant's data. (Relates to FR-07 and FR-08.)
- Delivery guarantee: recovering a stuck message (FR-15) or retrying after a timeout (FR-12) can send the same message twice. Decide which is worse, a duplicate send or a missed one. (Relates to FR-12 and FR-15.)
