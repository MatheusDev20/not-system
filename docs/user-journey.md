# Herald — User Journey

Herald is a multi-tenant notification platform. A company submits a batch of messages for a channel, Herald delivers them asynchronously, and the company tracks the outcome of the batch and of each message.

This journey describes Herald's **core**, which is channel-agnostic. Channel and input-format details (WhatsApp templates, phone formats, CSV parsing) live in adapters and are listed under [Later](#later--product-details).

## Definitions

- **Tenant:** a company using Herald.
- **Batch:** one submission from a tenant. It has a unique ID, a client-supplied idempotency key, exactly one channel, and progress (how many messages are pending, delivered or failed).
- **Message (notification):** one per recipient in a batch. It has a recipient, a payload, its own status, and its delivery attempts.
  - The core treats `recipient` as an opaque string and `payload` as opaque data. Only the channel adapter knows what they mean (e.g. a phone number and template variables for WhatsApp).
- **Channel adapter:** validates a message's recipient and payload for its channel, and sends it. A send results in *delivered*, *transient failure* or *permanent failure*.

**V1 scope:** a single `fake` channel that accepts any recipient and can fail on demand (transient or permanent), and a JSON entry point with a row limit.

## 1. The company submits a batch

The company's system sends a batch: an idempotency key, the channel, and a list of messages (recipient + payload). Herald answers immediately with **accepted** and the **batch ID**.

- **What if the request is invalid (missing fields, unsupported channel, too many rows)?** *Before accept*
  - Herald synchronously returns `400 Bad Request` and stores nothing.
- **What if some messages don't meet the channel's rules (e.g. a recipient the channel can't use)?** *After accept*
  - Per-message validation is asynchronous. Valid messages are processed; an invalid message fails on its own and doesn't invalidate the batch.
- **What if a message can't be delivered?** *After accept*
  - Transient failures (e.g. the provider is down) are retried according to a retry policy.
  - Permanent failures (e.g. the recipient doesn't exist) are not retried; the message goes straight to **failed**.
- **What if the maximum number of retries is reached?**
  - The message is marked **failed**.
- **What if the provider is unavailable for an extended period?**
  - Retries absorb short outages. If the outage outlasts the retry policy, the affected messages end as **failed**.
- **What if the company sends the same batch twice?**
  - Herald deduplicates by the client-supplied idempotency key: a resend with a known key returns the original batch ID and creates no new messages.

## 2. The company tracks its batches

Using the batch ID, the company can see:

- **Batch progress**, e.g. "8,200 of 10,000 delivered, 37 failed".
- **Each message's status**, including why it failed.

- **What if a message gets lost without a status?**
  - Herald promises that every accepted message always has a status. A message stuck in processing for too long is detected and recovered. (*How* is decided later.)

## Open questions

- Which checks run before accept and which after? (e.g. request shape before, per-message channel rules after)
- How does the company see messages that failed validation: as normal failed messages with a reason, or as a separate list?
- Retry policy numbers: how many attempts, what backoff, and how long counts as "extended" (Task 3).
- How long is an idempotency key remembered?
- What counts as "stuck in processing", and after how long?

## Later — product details

- **WhatsApp channel**, as the first real adapter. Business-initiated WhatsApp messages use pre-approved templates with variables, so its payload is probably a template + variables.
- **A second real channel (e.g. email).** Adding it is the trigger for checking the channel abstraction: only the adapter should change, not the core.
- **CSV entry point** for large batches: a second adapter that turns a file into the same batch. It likely needs file storage and asynchronous parsing.
- **Mixed-channel batches.**
