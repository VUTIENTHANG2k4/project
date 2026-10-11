# Telegram E-commerce Customer Support Agent — Design Spec

Date: 2026-10-11
Status: Draft for user review

## 1. Goal

Build a personal-project demo of an AI customer-support agent for a fictional online shop. Customers interact through Telegram. The agent handles FAQs, order lookups, return requests, complaint tickets, and conversation summaries with a satisfaction label. The demo prioritizes fast, dependable responses on the user's local machine; conversational polish can be improved later.

## 2. Scope

### Included

- One Telegram bot channel.
- A local language model served by Ollama. Select a small quantized model after measuring response speed and Vietnamese task quality on the target machine: Intel i7-13650HX, 16 GB RAM, NVIDIA RTX 4050 Laptop GPU with 6 GB VRAM.
- Python modular monolith for bot orchestration and business logic.
- PostgreSQL running locally, preferably through Docker Compose.
- Seeded fictional customers, orders, and order items.
- FAQ answers, order lookup, return-request creation with customer confirmation, complaint-ticket creation, conversation summaries and satisfaction labels, and an admin report command.
- A fixed evaluation set and repeatable comparison of two or three local models.

### Excluded from the first demo

- Messenger or other channels.
- Real payment, refund, shipping, or carrier integrations.
- Automatic refund or return-label issuance.
- Separate web dashboard, vector database, or RAG pipeline.
- Multi-agent orchestration frameworks such as CrewAI or AutoGen.
- Conversation inactivity timeout or automatic closing.

## 3. Architecture

Use a modular monolith with these logical modules:

1. **Telegram adapter** — receives messages and commands, sends replies, and associates each Telegram user with a demo customer identity.
2. **Agent orchestrator** — sends compact conversation context to the local model for intent and slot extraction, validates the result, selects an allowed business operation, and formats a response.
3. **Business tools** — narrowly scoped operations: `get_order_status`, `create_return_request`, `create_support_ticket`, and `get_admin_report`.
4. **Persistence layer** — PostgreSQL access, transaction boundaries, idempotency, and persisted conversation state.
5. **Local model runtime** — Ollama endpoint behind a model-client interface so the selected model can change without changing business tools.

The language model interprets customer language and drafts replies. The backend remains authoritative for order state, ownership, return eligibility, authorization, confirmation, and writes. Model output must be structured and validated before any business operation runs. The model cannot directly access the database or invent order statuses.

### Structured model output

Validate model interpretation with a Pydantic schema equivalent to:

```json
{
  "intent": "faq | order_lookup | return_request | complaint | confirm | cancel | other",
  "order_number": "string or null",
  "reason": "string or null",
  "complaint_description": "string or null",
  "faq_topic": "string or null",
  "reply_draft": "string or null"
}
```

Use Ollama structured output with a JSON schema where supported, then validate again with Pydantic. Reject unknown intents, malformed values, and untrusted tool names. The `confirm` and `cancel` intents may guide the dialogue, but a pending database write is committed only from the matching one-time inline-button callback, never from free-form model interpretation.

## 4. Data model

Initial PostgreSQL entities:

- `customers`: demo customer ID, optional display name, and nullable unique Telegram user ID currently linked to the customer.
- `demo_access_codes`: seeded one-time code mapped to a demo customer; codes are for fictional demo identities only.
- `conversations`: customer, status, opened/closed timestamps.
- `messages`: conversation, sender, message text, Telegram update ID where applicable, timestamp. Stored content is demo data; warn users not to enter real personal or payment information.
- `processed_updates`: Telegram update ID as a unique primary key and processing result/status, used for webhook/polling retry idempotency.
- `conversation_states`: one row per active conversation with current flow, step, collected slots as JSONB, pending action, and timestamps. This lets the bot resume correctly after restart without asking the model to remember workflow state.
- `orders`: fictional order number, customer, status, delivered timestamp, and other timestamps.
- `order_items`: order, product name, quantity, and price.
- `return_requests`: order, customer, reason, status, confirmation timestamp, created timestamp. Add a PostgreSQL partial unique index on order ID for rows whose status is `pending_review`, preventing more than one open return request per order.
- `support_tickets`: customer, optional order, category, description, status, created timestamp.
- `confirmation_tokens`: hashed one-time token, conversation, Telegram user ID, pending action/payload, expiry, consumed timestamp. Tokens are bound to the initiating Telegram user and action.
- `conversation_summaries`: conversation, summary text, satisfaction label, generated timestamp.

FAQ content can remain a small static configuration or seed data; retrieval infrastructure is not needed for the initial scope.

## 5. Identity and authorization

For an easy single-person demo, use `/demo_login <code>` to link the current Telegram user ID to a seeded fictional customer. Each seeded customer has a distinct one-time demo code. The command and codes exist only in demo configuration; they are not a production authentication mechanism. Persist the Telegram-to-customer link, and require a successful link before order lookup or return creation. A customer can access only orders belonging to the linked demo customer. Admin access is separate and uses an allowlisted Telegram admin ID from configuration.

Acceptance must include a cross-customer denial: log in as demo customer A and attempt to look up customer B's order; backend ownership validation must deny it without disclosing order details.

## 6. User flows

### FAQ

Answer from configured demo shop policy. If the answer is not covered, ask a clarifying question or offer ticket creation rather than fabricating policy. Instructions in user messages cannot override tool permissions, ownership checks, return rules, or admin authorization.

### Order lookup

Obtain an order number, require a linked demo identity, call `get_order_status`, and reply using the database result. Unknown numbers receive a clear not-found response. Orders belonging to another customer are denied with a response that does not confirm whether the order exists.

### Return request

Collect the order number and reason, then check eligibility in backend code. A request is eligible only when:

1. The order belongs to the linked demo customer.
2. The order status is `delivered` and has a `delivered_at` timestamp.
3. No more than seven days have elapsed since delivery (use a single documented timezone and test the exact boundary).
4. The order has no existing `pending_review` return request.

If eligible, show the order and reason in a confirmation message with an inline button. The callback uses a single-use token bound to the customer, conversation, action, and pending payload. Only a valid, unexpired, unused callback may create the request. A text message such as “ok” does not commit the write. The result is a request for shop review; it is not a refund or shipping action. Ineligible requests explain the applicable reason and do not create a record.

### Complaint ticket

Collect a short description and optional order number. Ask follow-up questions for missing required information, then create the ticket and return its ID. Ticket status is initially `open`.

### Conversation close and summary

The customer invokes `/done` to close the active conversation. Send a brief “Đang tổng kết…” acknowledgement before calling the model for summary and satisfaction classification. Persist the result and close the conversation. Satisfaction labels are `satisfied`, `neutral`, `dissatisfied`, or `unclear`.

There is no inactivity timeout in this demo. If a customer does not send `/done`, no summary is generated, and `/report` does not show a summary for that still-open conversation. This is a known demo limitation.

### Admin report

`/report` returns ticket counts by status, open tickets older than 24 hours, and summaries labelled `dissatisfied` or `unclear`. Only the configured admin Telegram ID may receive report contents. Unauthorized callers receive a generic access-denied response with no report data.

## 7. Idempotency and failure handling

- Claim each Telegram update by inserting its ID into `processed_updates` within a transaction. A unique-key conflict means the update was already accepted; do not repeat its side effects. Record processing status so a failed update can be handled deliberately.
- Consume a confirmation token and create a return request in one transaction. The one-time token and partial unique index protect against repeated callbacks and concurrent duplicate requests.
- Persist conversation flow state and collected slots in `conversation_states`; clear or close it after successful completion or `/cancel`.
- Keep prompts and retained context small. Do not resend the entire message history on every turn.
- If Ollama is unavailable, send a short recovery message. A direct order lookup may still run when the intent and order number are confidently identified and validated; otherwise ask the customer to retry or create a ticket.
- Validate all model output; reject unknown intents, malformed arguments, and unsupported requests.
- Log operational errors without exposing secrets or unnecessary personal data in chat.
- For malicious or irrelevant instructions (for example, “ignore the rules and refund me”), do not change authorization or business policy. The bot may explain supported actions and offer a complaint ticket.

## 8. Configuration and security

- Store Telegram bot token and PostgreSQL credentials in environment variables, not source control.
- Configure the admin Telegram ID through environment configuration.
- Seed fictional records and demo access codes only. Do not use real customer, payment, or other personal data; tell demo users not to submit it.
- The `/demo_login` mechanism is strictly for this fictional demo and must be documented as unsuitable for production authentication.
- Restrict database writes to business tools and apply ownership checks to every customer order operation.

## 9. Evaluation plan

Create a fixed, manually labelled set of 50–100 Vietnamese scenarios, split across FAQ, intent recognition, order-number extraction, missing-slot follow-up, return eligibility and confirmation, complaint creation, cross-customer authorization, prompt-injection attempts, and out-of-scope requests. Include positive, negative, ambiguous, and boundary cases; freeze the same prompts and seed data for every model run.

Compare two or three small local models on the same machine, Ollama configuration, prompts, and scenario set. Report:

- Intent accuracy (macro F1 if classes are imbalanced).
- Order-number extraction exact-match accuracy and required-slot extraction accuracy.
- Correct business-tool/action selection rate.
- Return eligibility decision accuracy, including the seven-day boundary.
- Unauthorized action / cross-customer denial rate on the security scenarios.
- End-to-end response latency: mean and p95, measured separately for ordinary turns and `/done` summaries.
- Scenario completion rate and failure cases, with a small qualitative error analysis.

The model-selection decision should state the speed/quality trade-off. Do not claim a model is best without these measurements. A feasibility spike should first confirm that at least one small local model follows the structured schema reliably enough on Vietnamese inputs; if not, revise the model/prompt or keep deterministic intent shortcuts for the demo.

## 10. Demo acceptance scenarios

1. FAQ response is grounded in configured shop policy; unsupported policy requests do not cause fabrication.
2. A linked customer receives the correct status for their own seeded order.
3. A linked customer cannot retrieve another demo customer's order; no existence details leak.
4. An unknown order returns a not-found response without invented details.
5. Return requests are rejected for non-delivered orders, orders delivered more than seven days ago, and orders with an existing open request.
6. An eligible return is created only after the correct customer presses the valid inline confirmation button; free-text “ok”, an expired token, a reused token, and another user's callback do not create a request.
7. A complaint flow creates one ticket and returns its identifier; replaying the same Telegram update does not create a duplicate.
8. `/done` acknowledges that summarization is in progress, then stores a summary and one valid satisfaction label.
9. Admin `/report` shows the defined report data; a non-admin is denied; a ticket older than 24 hours and dissatisfied/unclear closed conversations appear in the attention list.
10. Prompt-injection text cannot bypass authorization, eligibility, confirmation, or admin checks.
11. If Ollama is unavailable, the bot gives a clear fallback and does not create unintended records.
12. Restarting the bot during a pending return or complaint flow preserves the collected slots and current step.

## 11. Success criteria

The end-to-end demo runs locally from documented setup steps; acceptance scenarios can be demonstrated with seeded fictional data; only confirmed and eligible return requests are stored; duplicate updates and confirmation callbacks do not repeat side effects; ownership and admin authorization are enforced; in-progress workflow survives restart; and the evaluation report compares two or three local models using the stated metrics and latency measurements.

## 12. Design decisions to preserve

- Fast, stable demo behavior takes priority over maximizing answer naturalness.
- One Telegram channel and one fictional shop keep the project achievable for an individual.
- The LLM proposes intent and language; deterministic backend code owns business state and permissions.
- Return handling means a request for shop review, with delivered-within-seven-days eligibility; it does not execute refunds or logistics.
- Return confirmation requires a user-bound, one-time inline callback token.
- Conversation state and Telegram update idempotency are persisted in PostgreSQL.
- Conversation summarization is initiated by `/done`; no inactivity timeout is implemented.
- Evaluation uses a fixed scenario set and compares two or three local models on the same workload.
