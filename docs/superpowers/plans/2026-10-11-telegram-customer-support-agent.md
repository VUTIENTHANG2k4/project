# Telegram E-commerce Customer Support Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a locally runnable Telegram support-agent demo that looks up a customer's seeded orders, creates eligible return requests and complaint tickets, summarizes closed conversations, and reports operational metrics to an authorized admin.

**Architecture:** A Python modular monolith uses an async Telegram polling adapter, PostgreSQL persistence, and an Ollama client for validated Vietnamese intent/slot extraction and response drafting. Business operations, customer ownership, return eligibility, confirmation, idempotency, and admin access are enforced in backend code; persistent conversation state allows a flow to resume after restart.

**Tech Stack:** Python 3.12; aiogram 3.x; Pydantic 2; SQLAlchemy 2 async + Alembic; PostgreSQL 16; Ollama HTTP API; pytest + pytest-asyncio; Docker Compose.

**Spec:** `docs/superpowers/specs/2026-10-11-telegram-customer-support-agent-design.md`

## Global Constraints

- Use one Telegram bot channel and fictional shop data only.
- Prioritize fast, stable local responses on Intel i7-13650HX, 16 GB RAM, RTX 4050 Laptop GPU with 6 GB VRAM.
- Use a small quantized Ollama model; compare two or three candidates on an identical fixed workload before selecting the demo default.
- The model may classify, extract slots, and draft language; it must not access the database or determine authoritative business state.
- Require `/demo_login <code>` before customer order access; the command is demo-only and not production authentication.
- Permit order access only for the linked customer; admin reports require a separately configured Telegram ID.
- A return request requires a delivered order, delivery within the prior seven days, and no existing `pending_review` request; accept the exact seven-day boundary and use UTC timestamps.
- Commit a return request only after a valid, unexpired, unused inline callback token bound to the initiating Telegram user.
- Persist Telegram update processing state and active conversation workflow state in PostgreSQL.
- `/done` is required to close and summarize a conversation; no inactivity timeout is implemented.
- `/report` includes open tickets older than 24 hours and closed summaries labelled `dissatisfied` or `unclear`.
- Store credentials in environment variables. Use only seeded fictional data and tell demo users not to submit real personal/payment information.
- Do not build Messenger integration, real refunds/shipping, a web dashboard, RAG/vector search, or multi-agent orchestration in the first demo.

## Review Focus

- Malformed, missing, or unsupported model JSON/intent must not call a business tool; pin in Task 4 and Task 7.
- A Telegram update replay or callback replay must not duplicate tickets or returns; pin in Task 3 and Task 5.
- Cross-customer order IDs, demo-login switching, and forged callbacks must not expose or change another customer's data; pin in Task 5 and Task 6.
- Return cutoff boundaries, non-delivered orders, and existing open requests must receive deterministic decisions without writes; pin in Task 5.
- Ollama/PostgreSQL outages and prompt-injection attempts must fail closed for writes and permissions while returning a useful message; pin in Task 4, Task 6, and Task 7.

---

### Task 1: Repository and local runtime foundation

**Files:**
- Create: `.gitignore`
- Create: `pyproject.toml`
- Create: `.env.example`
- Create: `compose.yaml`
- Create: `README.md`
- Create: `src/support_agent/__init__.py`
- Create: `src/support_agent/config.py`
- Create: `tests/conftest.py`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces `Settings` in `src/support_agent/config.py`, loaded by `get_settings() -> Settings` from environment variables.
- `Settings` fields: `telegram_bot_token`, `telegram_admin_id`, `database_url`, `ollama_base_url`, `ollama_model`, and `demo_login_codes`.
- Defines pytest asyncio mode and reusable PostgreSQL test fixture configuration.

- [ ] **Step 1: Initialize Git and write failing configuration tests**

Create a Git repository because this workspace currently has no `.git` directory. Add tests for required Telegram/database configuration, optional Ollama defaults, and rejection of a missing admin ID or invalid demo-code mapping.

- [ ] **Step 2: Run the focused tests and confirm the expected failures**

Run: `pytest tests/test_config.py -q`
Expected: FAIL because `Settings` and `get_settings()` are not implemented.

- [ ] **Step 3: Implement configuration and project setup**

Implement `Settings` with Pydantic Settings, add the declared runtime/test dependencies, exclude `.env`, caches, model files, and local data from Git, and provide a PostgreSQL 16 Compose service with health check and persistent local volume. `.env.example` contains names and safe placeholders only.

- [ ] **Step 4: Run focused tests and verify runtime setup**

Run: `pytest tests/test_config.py -q`
Expected: PASS. Run: `docker compose config --quiet`
Expected: exit code 0.

- [ ] **Step 5: Commit the foundation**

Commit message: `chore: initialize local support agent project`.

### Task 2: Persistence schema, migrations, and fictional seed data

**Files:**
- Create: `src/support_agent/db.py`
- Create: `src/support_agent/models.py`
- Create: `alembic.ini`
- Create: `migrations/env.py`
- Create: `migrations/versions/0001_initial_schema.py`
- Create: `src/support_agent/seed.py`
- Create: `tests/db/conftest.py`
- Test: `tests/db/test_schema_and_seed.py`

**Interfaces:**
- `create_async_session_factory(settings: Settings) -> async_sessionmaker[AsyncSession]` in `db.py`.
- ORM entities: `Customer`, `DemoAccessCode`, `Conversation`, `Message`, `ProcessedUpdate`, `ConversationState`, `Order`, `OrderItem`, `ReturnRequest`, `SupportTicket`, `ConfirmationToken`, `ConversationSummary`.
- `seed_demo_data(session: AsyncSession) -> None` creates at least three customers, distinct login codes, owned orders covering return eligibility cases, and fictional FAQ policy/config.

- [ ] **Step 1: Write failing schema and seed tests**

Test all required tables/relationships; unique Telegram IDs and update IDs; JSONB conversation slots; the partial unique index allowing only one `pending_review` return per order; and seed fixtures for delivered within-window, exactly seven-day boundary, expired, non-delivered, and existing-pending cases.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/db/test_schema_and_seed.py -q`
Expected: FAIL because ORM models/migrations/seed function do not exist.

- [ ] **Step 3: Implement SQLAlchemy models, Alembic migration, and seed**

Use timezone-aware UTC timestamps. Give `ProcessedUpdate` a unique Telegram update ID plus processing status. Define `ConversationState` as one row per active conversation with flow, step, collected JSONB slots, pending action, and timestamps. Store only hashed one-time demo codes and confirmation tokens.

- [ ] **Step 4: Run focused PostgreSQL integration tests**

Run: `pytest tests/db/test_schema_and_seed.py -q`
Expected: PASS against the Compose PostgreSQL service, including enforcement of the partial unique index.

- [ ] **Step 5: Commit persistence foundation**

Commit message: `feat: add postgres schema and demo seed data`.

### Task 3: Repository operations, idempotency, and persisted workflow state

**Files:**
- Create: `src/support_agent/repositories/updates.py`
- Create: `src/support_agent/repositories/conversations.py`
- Create: `src/support_agent/repositories/identities.py`
- Create: `src/support_agent/repositories/orders.py`
- Create: `src/support_agent/repositories/tickets.py`
- Create: `tests/db/test_update_idempotency.py`
- Test: `tests/db/test_conversation_state.py`
- Test: `tests/db/test_identity_and_orders.py`

**Interfaces:**
- `claim_update(session, update_id: int) -> bool`: atomically claims an unseen update; returns false for a duplicate.
- `complete_update(session, update_id: int) -> None` and `fail_update(session, update_id: int, error_code: str) -> None` record terminal processing state without storing raw secrets.
- `get_or_create_conversation(session, telegram_user_id: int, demo_customer_id: UUID) -> Conversation`.
- `set_conversation_state(session, conversation_id: UUID, flow: str, step: str, slots: dict[str, object], pending_action: str | None) -> ConversationState`.
- `clear_conversation_state(session, conversation_id: UUID) -> None`.
- `link_demo_identity(session, telegram_user_id: int, code: str) -> Customer | None`; consume a valid code once and persist the Telegram-to-customer link. A valid new code replaces the current demo link for that Telegram user; all later operations use only the new customer identity.
- `get_owned_order(session, customer_id: UUID, order_number: str) -> Order | None`; do not distinguish unknown from another customer's order at the public interface.

- [ ] **Step 1: Write failing repository tests**

Cover duplicate update claims, retryable failed update status, state round-trip across a new session, one-time demo code consumption, identity relinking behavior, and same/other-customer order lookup.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/db/test_update_idempotency.py tests/db/test_conversation_state.py tests/db/test_identity_and_orders.py -q`
Expected: FAIL because repository functions are not implemented.

- [ ] **Step 3: Implement repositories and transaction boundaries**

Keep update claims short-lived and database-backed; a retry of an already completed update cannot repeat effects. Define explicit retry behavior for a failed claim. Do not hold a transaction open during model inference or Telegram network calls.

- [ ] **Step 4: Run repository tests**

Run: `pytest tests/db/test_update_idempotency.py tests/db/test_conversation_state.py tests/db/test_identity_and_orders.py -q`
Expected: PASS, including concurrent duplicate claims and persisted state after reopening the session.

- [ ] **Step 5: Commit repository layer**

Commit message: `feat: persist bot identity and workflow state`.

### Task 4: Local model client and validated Vietnamese intent schema

**Files:**
- Create: `src/support_agent/agent/schemas.py`
- Create: `src/support_agent/agent/model_client.py`
- Create: `src/support_agent/agent/prompts.py`
- Create: `src/support_agent/agent/intent_parser.py`
- Test: `tests/agent/test_schemas.py`
- Test: `tests/agent/test_intent_parser.py`
- Test: `tests/agent/test_model_client.py`

**Interfaces:**
- `Intent` enum: `faq`, `order_lookup`, `return_request`, `complaint`, `confirm`, `cancel`, `other`.
- `IntentEnvelope`: Pydantic model with `intent: Intent`, nullable `order_number`, `reason`, `complaint_description`, `faq_topic`, and `reply_draft`; forbid additional properties and constrain string lengths.
- `SatisfactionLabel` enum: `satisfied`, `neutral`, `dissatisfied`, `unclear`. `SummaryEnvelope` contains `summary: str` and `satisfaction: SatisfactionLabel`; forbid additional properties and constrain summary length.
- `OllamaClient.generate_intent(messages: Sequence[ChatMessage]) -> IntentEnvelope` and `OllamaClient.summarize(messages: Sequence[ChatMessage]) -> SummaryEnvelope` request structured JSON output and validate it with Pydantic.
- `OllamaUnavailable` and `InvalidModelOutput` are explicit exceptions handled by orchestration.

- [ ] **Step 1: Write failing schema/parser/client tests**

Test valid Vietnamese intent examples, all seven intents, null optional fields, extra-key rejection, malformed JSON, invalid enum values, oversized fields, timeout, and Ollama unavailable. Test `SummaryEnvelope` with all four satisfaction labels, malformed or empty summaries, and invalid labels. Include adversarial text that requests a refund or instruction override and verify it cannot name or invoke a tool.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/agent/test_schemas.py tests/agent/test_intent_parser.py tests/agent/test_model_client.py -q`
Expected: FAIL because schema and client are absent.

- [ ] **Step 3: Implement schema and Ollama adapter**

Use Ollama structured output with a JSON Schema generated from the Pydantic model. Keep model name configurable. The parser returns only validated intent/slots; never execute a tool from a model-provided tool name.

- [ ] **Step 4: Run offline tests and perform the model feasibility spike**

Run the focused pytest command with mocked HTTP responses; expected PASS. Then manually run a small Vietnamese prompt set against two or three candidate local models on the target machine and record schema validity, intent/slot accuracy, and latency in `docs/evaluation/model-feasibility.md` before selecting the demo default.

- [ ] **Step 5: Commit model boundary**

Commit message: `feat: add validated local model client`.

### Task 5: Customer tools, return policy, ticket creation, and one-time confirmation

**Files:**
- Create: `src/support_agent/services/orders.py`
- Create: `src/support_agent/services/returns.py`
- Create: `src/support_agent/services/tickets.py`
- Create: `src/support_agent/services/confirmations.py`
- Test: `tests/services/test_order_service.py`
- Test: `tests/services/test_return_service.py`
- Test: `tests/services/test_ticket_service.py`
- Test: `tests/services/test_confirmation_service.py`

**Interfaces:**
- `get_order_status(session, customer_id: UUID, order_number: str) -> OrderStatusResult | None`.
- `evaluate_return_eligibility(order: Order, now: datetime, has_open_request: bool) -> ReturnEligibility`.
- `prepare_return_confirmation(session, conversation_id: UUID, telegram_user_id: int, order_id: UUID, reason: str, now: datetime) -> ConfirmationView`; create a cryptographically random, single-use token with a 10-minute expiry.
- `confirm_return(session, token: str, telegram_user_id: int, now: datetime) -> ReturnRequestResult` consumes token and creates the request atomically; verify the Telegram user and currently linked customer still match the token record.
- `create_support_ticket(session, customer_id: UUID, description: str, order_number: str | None) -> TicketResult`.
- Eligibility is exactly: owned order, status `delivered`, non-null `delivered_at`, `now <= delivered_at + 7 days`, and no `pending_review` request.

- [ ] **Step 1: Write failing service tests**

Test each eligibility condition independently, UTC timestamps, exactly seven days accepted and one microsecond beyond rejected, cross-customer denial, generic not-found behavior, duplicate open request, ticket ID creation, malformed/expired/used token, token bound to another Telegram user, token bound to a customer identity that has since changed, and concurrent confirmation callbacks.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/services -q`
Expected: FAIL because service functions are not implemented.

- [ ] **Step 3: Implement deterministic business services**

Use database queries and explicit UTC `now` parameters; model output is never the source of eligibility. Store only a hash of the callback token, bind it to the action/customer/conversation/payload, and consume it in the same transaction as return creation. Keep return requests in `pending_review`; never issue a refund or label.

- [ ] **Step 4: Run service tests**

Run: `pytest tests/services -q`
Expected: PASS, with no writes for any ineligible, unconfirmed, expired, replayed, or wrong-user case.

- [ ] **Step 5: Commit business tools**

Commit message: `feat: add order return and ticket services`.

### Task 6: Agent orchestration and shop FAQ behavior

**Files:**
- Create: `src/support_agent/agent/orchestrator.py`
- Create: `src/support_agent/agent/faq.py`
- Create: `src/support_agent/agent/tool_router.py`
- Create: `tests/agent/test_tool_router.py`
- Test: `tests/agent/test_orchestrator.py`
- Test: `tests/agent/test_faq.py`

**Interfaces:**
- `handle_customer_message(session, conversation_id: UUID, customer_id: UUID, text: str) -> AgentReply`.
- `AgentReply` contains `text: str`, optional `inline_confirmation: ConfirmationView`, and optional `action_status`.
- Tool routing maps only validated `Intent` values to fixed Python service calls; ownership and business validation occur in the called service.
- FAQ policy is loaded from a small checked-in fictional shop config and returns an explicit unknown answer when no configured entry matches.

- [ ] **Step 1: Write failing orchestration tests**

Test FAQ answer/unknown policy, order lookup, missing order number follow-up, return flow collection and confirmation view, complaint detail collection, cancel flow, other/irrelevant input, malformed model output, and model outage. Verify prompt-injection phrasing cannot create a return, ticket with fabricated data, or admin action.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/agent/test_tool_router.py tests/agent/test_orchestrator.py tests/agent/test_faq.py -q`
Expected: FAIL because router/orchestrator are absent.

- [ ] **Step 3: Implement intent routing and persisted dialogue transitions**

For incomplete flows, save collected slots and step in `conversation_states`; fetch current state before classifying new text. Free-text `confirm` never commits a return; only the callback handler can do so. Return a concise Vietnamese fallback when Ollama is unavailable, and allow deterministic order lookup only when both validated intent and order number are available.

- [ ] **Step 4: Run orchestration tests**

Run the focused pytest command above.
Expected: PASS; no business write occurs on unknown intent, invalid output, injection attempt, or model outage.

- [ ] **Step 5: Commit orchestration**

Commit message: `feat: route customer intents to validated tools`.

### Task 7: Telegram adapter, login, commands, and callback security

**Files:**
- Create: `src/support_agent/bot/__init__.py`
- Create: `src/support_agent/bot/handlers.py`
- Create: `src/support_agent/bot/keyboards.py`
- Create: `src/support_agent/bot/middleware.py`
- Create: `src/support_agent/main.py`
- Test: `tests/bot/test_handlers.py`
- Test: `tests/bot/test_callbacks.py`
- Test: `tests/bot/test_admin_authorization.py`

**Interfaces:**
- `build_dispatcher(session_factory, settings, ollama_client) -> Dispatcher`.
- `run_polling(settings: Settings) -> None` constructs the bot, dispatcher, and session factory and starts Telegram long polling.
- Commands: `/start`, `/demo_login <code>`, `/done`, `/cancel`, `/report`.
- Inline confirmation callback contains an opaque random token only; server-side token record binds the action and initiating Telegram user.

- [ ] **Step 1: Write failing Telegram handler/callback tests**

Test login success/reuse/invalid code, order request requiring login, valid and invalid order ownership, `/done` acknowledgement before model summary, `/cancel` clears persisted state, valid callback, expired/reused/foreign-user callback, admin report allowlist, and non-admin report denial. Test replayed Telegram update results in one side effect.

- [ ] **Step 2: Run focused bot tests and confirm expected failures**

Run: `pytest tests/bot -q`
Expected: FAIL because handlers and dispatcher are absent.

- [ ] **Step 3: Implement aiogram handlers and startup**

Keep handlers thin: parse Telegram update, claim idempotency key, load identity/conversation state, call orchestrator/service, persist outcome, then send response. Send “Đang tổng kết…” before `/done` invokes summary generation. On `/report`, authorize before querying report data. Escape/limit model-derived Telegram text and never place sensitive payloads in callback data.

- [ ] **Step 4: Run bot tests**

Run: `pytest tests/bot -q`
Expected: PASS; unauthorized handlers do not query or return report contents, and callback replay is side-effect free.

- [ ] **Step 5: Commit Telegram integration**

Commit message: `feat: add telegram customer and admin flows`.

### Task 8: Conversation summary and admin report

**Files:**
- Create: `src/support_agent/services/summaries.py`
- Create: `src/support_agent/services/reports.py`
- Test: `tests/services/test_summaries.py`
- Test: `tests/services/test_reports.py`

**Interfaces:**
- `summarize_conversation(session, conversation_id: UUID, model_client: OllamaClient) -> ConversationSummary`.
- `build_admin_report(session, now: datetime) -> AdminReport`.
- Summary labels are exactly `satisfied`, `neutral`, `dissatisfied`, `unclear`.
- Attention report includes open tickets created more than 24 hours before `now` and closed summaries labelled `dissatisfied` or `unclear`.

- [ ] **Step 1: Write failing summary/report tests**

Test summary label validation for all four labels, empty conversation, model failure without falsely closing as summarized, `/done` persistence, report 24-hour boundary, status counts, inclusion of dissatisfied/unclear only, and exclusion of open conversations without summaries.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/services/test_summaries.py tests/services/test_reports.py -q`
Expected: FAIL because services are absent.

- [ ] **Step 3: Implement summary and report services**

Summarize only messages belonging to the conversation and bound the input size. Keep `/report` data generation separate from authorization; the Telegram handler must authorize first. Call `model_client.summarize(...) -> SummaryEnvelope`. If summary generation fails, preserve the conversation as open and allow `/done` retry without duplicating a summary.

- [ ] **Step 4: Run focused tests**

Run the focused pytest command above.
Expected: PASS, including the exact 24-hour cutoff and retry behavior.

- [ ] **Step 5: Commit summary and reporting**

Commit message: `feat: summarize conversations and report support status`.

### Task 9: Fixed evaluation corpus and model comparison

**Files:**
- Create: `evaluation/scenarios.jsonl`
- Create: `evaluation/run_model_eval.py`
- Create: `evaluation/metrics.py`
- Create: `docs/evaluation/README.md`
- Create: `docs/evaluation/model-feasibility.md`
- Test: `tests/evaluation/test_metrics.py`
- Test: `tests/evaluation/test_scenario_validation.py`

**Interfaces:**
- Each scenario has stable `id`, `category`, `messages`, expected `intent`, expected extracted slots, expected action/decision, and optional expected satisfaction label.
- `score_predictions(scenarios, predictions) -> EvaluationReport` returns intent accuracy/macro-F1, exact-match order extraction, slot accuracy, action-selection accuracy, return-eligibility accuracy, unauthorized-write count, completion rate, mean latency, and p95 latency.
- The same seed data, prompts, model settings, and scenarios are used for every candidate model.

- [ ] **Step 1: Write failing metric and corpus validation tests**

Test macro-F1 for imbalanced labels, exact-match slot scoring, p95 calculation, missing/duplicate scenario IDs, required scenario categories, and enforcement of a corpus size from 50 to 100.

- [ ] **Step 2: Run focused tests and confirm expected failures**

Run: `pytest tests/evaluation -q`
Expected: FAIL because the evaluator and corpus schema are absent.

- [ ] **Step 3: Implement labelled scenarios and evaluation runner**

Include FAQ, intents, order extraction, missing slots, return eligibility and exact cutoff, confirmation, complaint, cross-customer denial, injection, unsupported requests, model outage, and summary. Run two or three quantized Ollama candidates on the same target hardware. Save raw predictions and summarized metrics without real personal data.

- [ ] **Step 4: Run evaluation and record the selection rationale**

Run: `python -m evaluation.run_model_eval --scenarios evaluation/scenarios.jsonl`
Expected: a reproducible report with per-model intent/slot/action/security metrics and mean/p95 latency for ordinary turns and `/done`; document the speed/quality trade-off and chosen default model.

- [ ] **Step 5: Commit evaluation artifacts**

Commit message: `feat: add repeatable local model evaluation`.

### Task 10: End-to-end demo documentation and final verification

**Files:**
- Modify: `README.md`
- Create: `docs/demo-script.md`
- Test: `tests/integration/test_demo_acceptance.py`

**Interfaces:**
- README documents prerequisites, `.env` setup, database migration/seed, Ollama model setup, bot startup, evaluation command, and cleanup of local demo containers.
- Demo script lists the exact Telegram steps for each acceptance scenario and expected result.

- [ ] **Step 1: Write failing integration acceptance checks**

Add integration scenarios for login, own/foreign order, FAQ, return allowed/denied/confirmed, ticket creation, `/done`, admin/non-admin `/report`, restart-resumed state, replayed update, and model/database failure. Use a real PostgreSQL test service and mocked Telegram/Ollama boundaries where appropriate.

- [ ] **Step 2: Run the integration checks and confirm expected failures**

Run: `pytest tests/integration/test_demo_acceptance.py -q`
Expected: FAIL until the application wiring is complete.

- [ ] **Step 3: Complete setup and demo documentation**

Document only the verified commands and explicitly state `/demo_login` is demo-only and `/done` is required for summaries.

- [ ] **Step 4: Run the full verification suite and local smoke demo**

Run: `pytest -q`
Expected: PASS. Then follow `docs/demo-script.md` against local PostgreSQL, Ollama, and Telegram; confirm every acceptance scenario and capture latency results in the evaluation report.

- [ ] **Step 5: Commit the complete demo**

Commit message: `docs: document local telegram support agent demo`.

