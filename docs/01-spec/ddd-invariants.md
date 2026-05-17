# PWA‑Hermes — DDD Invariants

## 0) Назначение документа

Этот документ переводит доменные правила PWA‑Hermes в матрицу enforcement: что должна защищать БД, что — RLS, что — RPC/service/worker, а что остаётся только отображением во frontend.

Источник контекста:

- `docs/01-spec/requirements-mvp.md`
- `docs/01-spec/ddd-context-map.md`
- `docs/01-spec/ddd-domain-model.md`
- `docs/00-setup/architecture-constraints.md`

Главное правило:

> Frontend не владеет доменными переходами. Источник истины — Postgres/Supabase. Redis — временная координация, не canonical state.

---

## 1) Enforcement levels

### DB constraint

Жёсткое правило на уровне таблицы:

- `not null`
- `check`
- `foreign key`
- enum type
- exclusion/unique constraint where possible

Использовать для правил, которые не требуют сложной бизнес‑процедуры.

### Unique/index

Структурная защита и быстрые проверки:

- уникальность связи;
- быстрый lookup по owner/status;
- частичные индексы для active/deleted state.

### Trigger / function / RPC

Использовать там, где нужен transaction-safe бизнес‑переход:

- active chat limit;
- create message + create job atomically;
- controlled job transitions;
- service-only materialization of assistant message.

### RLS policy

Защищает ownership и запрещает прямые операции пользователя:

- пользователь видит только свои rows;
- пользователь может insert/update только разрешённые rows;
- service role/worker делает privileged transitions.

### Service / worker rule

Для side effects и внешних систем:

- file upload/storage;
- Redis queues;
- STT/TTS;
- Hermes core;
- retention cleanup.

### Frontend display-only rule

Frontend может помогать UX, но не является enforcement boundary.

Пример: UI скрывает кнопку «Новый чат» при 5 активных чатах, но БД/RPC всё равно обязаны запретить шестой чат.

---

## 2) Core invariants matrix

### I-001: Пользователь видит только свои данные

**Rule:** пользователь может читать только свои chats/messages/jobs/media/projects/plans/cron rows.

**Owner context:** Identity & Access.

**Enforcement:**

- RLS policy: required.
- DB schema: all user-owned tables include `profile_id` or `user_id`.
- Index: owner column indexed.

**Tables:**

- `profiles`
- `chats`
- `messages`
- `voice_assets`
- `processing_jobs`
- `assistant_runs`
- `projects`
- `plans`
- `cron_tasks`
- `cron_runs`
- `domain_events`

**Notes:** Supabase Auth identity maps through `profiles.user_id = auth.uid()`.

---

### I-002: Доменная модель не растворяется в `auth.users`

**Rule:** `auth.users` — внешний identity provider, доменный профиль хранится в `profiles`.

**Owner context:** Identity & Access.

**Enforcement:**

- DB FK: `profiles.user_id → auth.users.id`.
- Unique constraint: one `profiles` row per `auth.users.id`.
- Service rule: create profile on first login if missing.

**Tables:** `profiles`.

---

### I-003: Не больше 5 активных чатов на пользователя

**Rule:** пользователь может иметь максимум 5 chats со статусом `active`.

**Owner context:** Chats & Messages.

**Enforcement:**

- RPC/function: `create_chat(...)` checks count in transaction.
- Trigger/function: optional DB-level guard on insert/update to active.
- RLS: user can only create chat for self.
- Frontend: hide/disable create button at 5, display explanation.

**Tables:** `chats`.

**DB design note:** partial unique cannot directly express “max 5”; use function/trigger or controlled RPC.

---

### I-004: Chat belongs to exactly one user/profile

**Rule:** каждый chat имеет owner scope.

**Owner context:** Chats & Messages.

**Enforcement:**

- DB constraint: `chats.profile_id not null`.
- FK: `chats.profile_id → profiles.id`.
- RLS: select/insert/update only own chats.

**Tables:** `chats`.

---

### I-005: Message always belongs to one chat

**Rule:** message не существует вне chat.

**Owner context:** Chats & Messages.

**Enforcement:**

- DB constraint: `messages.chat_id not null`.
- FK: `messages.chat_id → chats.id`.
- Optional denormalized owner: `messages.profile_id not null` for RLS/index.
- Trigger/function: ensure `messages.profile_id = chats.profile_id` if denormalized.

**Tables:** `messages`, `chats`.

---

### I-006: User cannot directly create assistant messages

**Rule:** normal user может создать только user text/voice message. `assistant` message создаётся service role/worker after `AssistantRun`.

**Owner context:** Chats & Messages / Assistant Interaction.

**Enforcement:**

- RLS insert policy: user insert allowed only where `role = 'user'` and allowed `input_type`.
- Service role: allowed to insert `assistant`/`system` messages.
- DB check: assistant message requires `assistant_run_id` not null or service-created pathway.

**Tables:** `messages`, `assistant_runs`.

---

### I-007: Text user message has text body

**Rule:** `role=user` + `input_type=text` requires non-empty text.

**Owner context:** Chats & Messages.

**Enforcement:**

- DB check: if `role='user' and input_type='text'`, then `body_text is not null and length(trim(body_text)) > 0`.
- API/service validation: reject empty input before DB.
- Frontend: disable send on empty input.

**Tables:** `messages`.

---

### I-008: Voice user message references voice original asset

**Rule:** `role=user` + `input_type=voice` must be linked to `VoiceAsset(kind=user_original)` once ingest transaction completes.

**Owner context:** Chats & Messages / Voice Processing.

**Enforcement:**

- DB FK: `messages.voice_asset_id → voice_assets.id`.
- DB check/RPC: voice message pathway must set voice asset.
- Trigger/function: verify `voice_assets.kind = 'user_original'` and same owner/chat.
- Service route: upload audio first or within transaction choreography.

**Tables:** `messages`, `voice_assets`.

**Open design point:** if supporting pending upload state, DB must model that explicitly instead of allowing invalid nulls silently.

---

### I-009: Assistant message references completed assistant run

**Rule:** assistant message must be produced by `AssistantRun` and linked to it.

**Owner context:** Assistant Interaction.

**Enforcement:**

- DB FK: `messages.assistant_run_id → assistant_runs.id`.
- DB check: if `role='assistant'`, then `assistant_run_id is not null`.
- Service transaction: complete run + create message + update job.
- RLS: user cannot insert assistant rows.

**Tables:** `messages`, `assistant_runs`.

---

### I-010: AssistantRun belongs to one input message

**Rule:** assistant run is tied to one user input message and one chat.

**Owner context:** Assistant Interaction.

**Enforcement:**

- DB constraints: `assistant_runs.input_message_id not null`, `chat_id not null`.
- FKs to `messages` and `chats`.
- Trigger/function: input message must belong to same chat/profile.
- Service validation: run only for user/system input, not arbitrary assistant message.

**Tables:** `assistant_runs`, `messages`, `chats`.

---

### I-011: Completed AssistantRun has assistant text

**Rule:** `assistant_runs.status = completed` requires non-empty `assistant_text`.

**Owner context:** Assistant Interaction.

**Enforcement:**

- DB check: completed implies text exists.
- Service/worker validation.

**Tables:** `assistant_runs`.

---

### I-012: ProcessingJob is durable in Postgres

**Rule:** any user submission that needs processing creates a `processing_jobs` row; Redis cannot be the only job state.

**Owner context:** Processing Jobs & Queue Coordination.

**Enforcement:**

- Service/RPC: create message + job before enqueue.
- Worker: load/update job from Postgres.
- Redis rule: queue item references `processing_jobs.id`.

**Tables:** `processing_jobs`.

---

### I-013: User cannot mutate job status directly

**Rule:** only service/worker/RPC can transition job status.

**Owner context:** Processing Jobs & Queue Coordination.

**Enforcement:**

- RLS update policy: normal users cannot update status/progress/result/error fields.
- Service role: allowed.
- Optional RPC: controlled transition function.

**Tables:** `processing_jobs`.

---

### I-014: ProcessingJob status transitions are valid

**Rule:** jobs follow allowed lifecycle.

**Owner context:** Processing Jobs & Queue Coordination.

**Allowed flows:**

```text
voice: ingested → transcribing → ready → answering → synthesizing → answered
text + TTS: ingested → ready → answering → synthesizing → answered
text only: ingested → ready → answering → answered
any active state → error
```

**Enforcement:**

- Service/worker rule required.
- Optional DB transition function: update status only via `transition_processing_job(...)`.
- Audit/domain event for transitions.

**Tables:** `processing_jobs`, optional `domain_events`.

**DB design note:** Simple enum is not enough; transition validity needs function/trigger if enforced in DB.

---

### I-015: Answered job requires assistant output

**Rule:** `processing_jobs.status = answered` requires assistant result/message reference.

**Owner context:** Processing Jobs & Queue Coordination / Assistant Interaction.

**Enforcement:**

- DB check: answered implies `assistant_message_id is not null` or `assistant_run_id completed`.
- Service transaction: create assistant message before final status.

**Tables:** `processing_jobs`, `messages`, `assistant_runs`.

---

### I-016: Error job preserves error metadata

**Rule:** `processing_jobs.status = error` requires error code/message sufficient for UI and debugging.

**Owner context:** Processing Jobs & Queue Coordination.

**Enforcement:**

- DB check: error implies `error_code` or `error_message` not null.
- Service/worker rule.

**Tables:** `processing_jobs`.

---

### I-017: Voice original has retention deadline

**Rule:** `voice_assets.kind = user_original` requires `expires_at`.

**Owner context:** Voice Processing / Media Retention.

**Enforcement:**

- DB check: if kind user_original then `expires_at is not null`.
- Service upload rule: compute expires_at at registration.

**Tables:** `voice_assets`.

---

### I-018: VoiceAsset belongs to same owner as chat/message

**Rule:** voice asset linked to chat/message must have same `profile_id` and compatible `chat_id`.

**Owner context:** Voice Processing / Chats & Messages.

**Enforcement:**

- DB FK where possible.
- Trigger/function: validate cross-table profile/chat ownership.
- RLS: user can read own assets only.

**Tables:** `voice_assets`, `messages`, `chats`.

---

### I-019: Deleting voice binary does not delete transcript/history

**Rule:** retention cleanup may remove binary audio but must preserve transcript and message history.

**Owner context:** Media Retention / Chats & Messages.

**Enforcement:**

- Service/worker rule: delete storage object, then mark `voice_assets.status=deleted/expired`.
- DB schema: transcript lives in `messages.transcript_text` or equivalent, not only in audio asset.
- DB FK delete behavior: do not cascade-delete messages when voice asset metadata changes.

**Tables:** `voice_assets`, `messages`.

---

### I-020: TTS asset is separate from assistant text

**Rule:** assistant text remains available even if TTS asset is missing/deleted/failed.

**Owner context:** Voice Processing / Assistant Interaction.

**Enforcement:**

- DB schema: assistant text in `messages.body_text` / `assistant_runs.assistant_text`; TTS is asset reference only.
- Service rule: TTS failure may mark job error or partial depending product decision; must not lose assistant text.

**Tables:** `messages`, `assistant_runs`, `voice_assets`.

---

### I-021: Project links stay within same owner scope

**Rule:** chat/plan/cron task linked to project must belong to same profile/user.

**Owner context:** Projects.

**Enforcement:**

- Trigger/function or RPC on link/update.
- RLS: user only sees own projects and own linked entities.
- Optional FK composite pattern if schema uses `(id, profile_id)` uniqueness.

**Tables:** `projects`, `chats`, `plans`, `cron_tasks`.

---

### I-022: Archived/deleted project does not accept new links

**Rule:** new links to archived/deleted project are disallowed unless explicit restore/reactivation.

**Owner context:** Projects.

**Enforcement:**

- Service/RPC validation.
- Optional trigger/function.

**Tables:** `projects`, linked tables.

---

### I-023: CronTask schedule must be valid before activation

**Rule:** active cron task requires valid schedule expression and timezone.

**Owner context:** Plans & Cron.

**Enforcement:**

- Service validation with chosen parser.
- DB check for non-empty fields when active.
- Worker rejects invalid schedule and marks error.

**Tables:** `cron_tasks`.

---

### I-024: Paused/disabled CronTask is not dispatched

**Rule:** cron dispatcher must skip tasks with `paused` or `disabled` status.

**Owner context:** Plans & Cron.

**Enforcement:**

- Worker query filters only `status='active'` and due `next_run_at`.
- DB index on `(status, next_run_at)`.
- UI can display but not enforce.

**Tables:** `cron_tasks`.

---

### I-025: CronRun belongs to CronTask owner

**Rule:** cron run profile/scope must match its cron task.

**Owner context:** Plans & Cron.

**Enforcement:**

- DB FK to `cron_tasks`.
- Trigger/function to copy/validate `profile_id`.
- RLS: user sees own runs only.

**Tables:** `cron_runs`, `cron_tasks`.

---

### I-026: Domain events are append-only

**Rule:** domain events should not be updated/deleted by normal users.

**Owner context:** Audit & Operations.

**Enforcement:**

- RLS: users may select own events if exposed, no insert/update/delete.
- Service role insert only.
- DB permissions: no update/delete path except admin maintenance.

**Tables:** `domain_events`.

---

## 3) Frontend display-only rules

Frontend may:

- hide “Новый чат” at 5 active chats;
- disable empty text send button;
- show job status labels;
- show missing/deleted audio playback state;
- show cron paused/active status;
- poll job state.

Frontend must not be trusted to:

- enforce chat limit;
- decide job transitions;
- create assistant messages;
- bypass RLS;
- calculate authoritative retention deletion;
- dispatch cron tasks;
- write durable assistant/STT/TTS results directly.

---

## 4) Service/worker-only operations

These operations require service role, server route, worker, or controlled RPC:

- `create_chat` if enforcing 5-chat limit transactionally;
- `submit_text_message` if creating message + job atomically;
- `submit_voice_message` because file upload + voice asset + job;
- STT transcription;
- Hermes core assistant run;
- TTS synthesis;
- job status transitions;
- assistant message creation;
- retention cleanup;
- cron dispatch;
- domain event append.

---

## 5) DB-first schema implications

Initial schema should prefer:

- explicit owner columns on all owned tables;
- enums for stable statuses/types;
- FK relationships for all aggregate references;
- check constraints for simple compatibility rules;
- RPC/functions for transactional business commands;
- RLS policies before exposing PostgREST resources;
- Redis references by `processing_job_id`, never standalone canonical jobs.

---

## 6) Open questions for Postgres ERD

1. Use `profile_id` everywhere, `user_id` everywhere, or both? Current preference: `profile_id` canonical + maybe `user_id` only where RLS simplicity requires.
2. Use one `messages.body_text` for both user text and assistant text, with `transcript_text` for voice, or separate `assistant_text` column? Current preference: one `body_text` plus role; `transcript_text` for voice.
3. Should `processing_jobs` be created by SQL RPC for both text and voice, or by service route after inserts? Current preference: controlled RPC/service transaction, not direct frontend insert.
4. How strongly enforce job transition state machine in DB v1: service-only or DB function required? Current preference: service-only plus no direct user update; add DB function if transitions become messy.
5. Should `domain_events` be RLS-readable to user or internal only? Current preference: internal/service-only for MVP.

---

## 7) Следующий документ

Следующий шаг: `docs/01-spec/ddd-events.md`.

После events можно переходить к DB documents:

- `docs/05-data/postgres-erd.md`
- `docs/05-data/supabase-schema.md`
- `docs/05-data/rls-policy-model.md`
- `docs/05-data/redis-job-queue-model.md`
