# PWA-Hermes DDD + DB-First Implementation Plan

> **For Hermes:** Use hermes-subagent-driven-development skill to implement this hermes-plan task-by-task.

**Goal:** Сначала определить DDD-модель PWA-Hermes, затем спроектировать Postgres/Supabase модель БД, и только после этого переходить к API/frontend.

**Architecture:** Проект строится DB-first: Postgres/Supabase — источник истины, frontend тонкий. DDD описывает bounded contexts, aggregates, entities, value objects, domain events и invariants; затем это маппится в Supabase schema, RLS, PostgREST contracts и Redis/job pipeline.

**Tech Stack:** Supabase, Postgres, PostgREST, Redis, Next.js App Router, Vite + React + TypeScript, shadcn/ui, Tailwind CSS.

---

## Current Context

Уже зафиксировано в проекте:

- `README.md` — общий MVP и ссылки на документы.
- `docs/00-setup/architecture-constraints.md` — стек и ограничения: не начинать реализацию без DDD, не начинать frontend без модели БД.
- `docs/01-spec/requirements-mvp.md` — MVP-требования.
- `docs/03-api/api-contracts-mvp.md` — предварительные API-контракты.
- `docs/02-ui/ui-flow-and-screens-mvp.md` — UI flow.
- `docs/07-design-tokens/PWA-Hermes-design-tokens.md` — UI design tokens.

Ключевые решения:

- До 5 параллельных чатов.
- Главная кнопка: «Записать аудио послание».
- MVP: текст + голос.
- Голосовой pipeline: audio upload → STT → Hermes core → assistant text → TTS.
- Без стриминга, polling по job.
- Голосовые оригиналы удаляются примерно через месяц.
- Frontend тонкий, БД — всему голова.

---

## Proposed Approach

Работа идёт в четыре фазы:

1. **DDD discovery** — описать домен, bounded contexts, aggregates, invariants.
2. **DB model** — спроектировать Supabase/Postgres schema как источник истины.
3. **PostgREST/API alignment** — сверить предварительные API-контракты с моделью БД.
4. **Implementation readiness** — подготовить миграции, RLS, seed/dev data и только потом открыть путь к backend/frontend.

В этом плане **не начинаем frontend** и **не пишем runtime-код приложения**. Разрешены только документы, SQL-модель, тестовые SQL-проверки и подготовительные схемы.

---

## Deliverables

### DDD documents

- Create: `docs/01-spec/ddd-context-map.md`
- Create: `docs/01-spec/ddd-domain-model.md`
- Create: `docs/01-spec/ddd-invariants.md`
- Create: `docs/01-spec/ddd-events.md`

### Database design

- Create: `docs/05-data/postgres-erd.md`
- Create: `docs/05-data/supabase-schema.md`
- Create: `docs/05-data/rls-policy-model.md`
- Create: `docs/05-data/redis-job-queue-model.md`

### SQL artifacts, when ready

- Create: `supabase/migrations/0001_initial_domain.sql`
- Create: `supabase/seed.sql`
- Create: `supabase/tests/domain_invariants.sql` or `docs/05-data/sql-validation-checklist.md` if SQL test harness is not chosen yet.

### API alignment

- Modify: `docs/03-api/api-contracts-mvp.md`
- Create: `docs/03-api/postgrest-resource-map.md`

---

## Phase 1 — DDD Discovery

### Task 1: Create DDD context map

**Objective:** Зафиксировать bounded contexts и отношения между ними.

**Files:**

- Create: `docs/01-spec/ddd-context-map.md`

**Content to include:**

- Context: `Identity & Access`
- Context: `Chats & Messages`
- Context: `Voice Processing`
- Context: `Assistant Interaction`
- Context: `Projects`
- Context: `Plans & Cron`
- Context: `Media Retention`
- Context: `Audit & Operations`

**Key decisions:**

- `Chats & Messages` — core domain для MVP.
- `Voice Processing` — supporting domain, управляет STT/TTS lifecycle.
- `Plans & Cron` — отдельный context, не смешивать с chat messages.
- `Media Retention` — отдельная policy/service boundary.
- Identity может опираться на Supabase Auth, но доменная модель не должна растворяться в auth.users.

**Verification:**

- Проверить, что каждый MVP-сценарий из `docs/01-spec/requirements-mvp.md` попадает хотя бы в один bounded context.
- Проверить, что voice pipeline не размазан по frontend.

---

### Task 2: Define domain model glossary

**Objective:** Описать доменные термины на русском языке с точным смыслом.

**Files:**

- Create: `docs/01-spec/ddd-domain-model.md`

**Entities / aggregates to define:**

- `UserProfile`
- `Workspace` or `AccountScope` if needed
- `Chat`
- `Message`
- `VoiceAsset`
- `AssistantRun`
- `ProcessingJob`
- `Project`
- `Plan`
- `CronTask`
- `RetentionPolicy`

**Value objects to define:**

- `JobStatus`
- `MessageRole`
- `MessageInputType`
- `ReplyMode`
- `MediaKind`
- `RetentionTTL`
- `ScheduleExpression`

**Verification:**

- Each term has: definition, owner context, lifecycle, DB mapping candidate.
- No entity exists only because UI needs it.

---

### Task 3: Define aggregates and ownership

**Objective:** Понять, какие объекты изменяются вместе и где границы транзакций.

**Files:**

- Modify: `docs/01-spec/ddd-domain-model.md`

**Aggregate candidates:**

- `ChatAggregate`: Chat + ordered Messages metadata.
- `ProcessingJobAggregate`: Job + status transitions + error/result references.
- `VoiceAssetAggregate`: media asset + retention metadata.
- `ProjectAggregate`: project metadata and links.
- `CronTaskAggregate`: schedule + state + execution references.

**Rules:**

- `Message` не хранит весь binary/audio; только ссылки на `VoiceAsset`.
- `ProcessingJob` не должен быть просто UI status; он отражает серверный lifecycle.
- `Chat` не должен знать детали STT/TTS providers.

**Verification:**

- Для каждого aggregate указать root, child entities, allowed commands.
- Указать транзакционные границы.

---

### Task 4: Define invariants

**Objective:** Зафиксировать правила, которые БД должна защищать constraints/RLS/trigger’ами где возможно.

**Files:**

- Create: `docs/01-spec/ddd-invariants.md`

**Invariants:**

- У пользователя не больше 5 активных чатов.
- Сообщение всегда принадлежит одному chat.
- Chat принадлежит одному user/account scope.
- `assistant` message создаётся только как результат `AssistantRun`/`ProcessingJob`.
- Voice message должен иметь `VoiceAsset` или явную ошибку ingest.
- `ProcessingJob.status` меняется только по разрешённому state machine.
- `answered` job обязан иметь assistant result.
- Исходный voice asset должен иметь retention deadline.
- Удаление voice original не удаляет transcript.
- CronTask не исполняется, если paused/disabled.

**Verification:**

- Для каждого invariant указать enforcement level: DB constraint, RLS, trigger/function, application service, background worker.

---

### Task 5: Define domain events

**Objective:** Описать события, на которых будут строиться jobs, Redis и последующая интеграция.

**Files:**

- Create: `docs/01-spec/ddd-events.md`

**Events:**

- `ChatCreated`
- `UserTextMessageSubmitted`
- `UserVoiceMessageUploaded`
- `VoiceTranscriptionStarted`
- `VoiceTranscriptionCompleted`
- `AssistantRunRequested`
- `AssistantRunCompleted`
- `TtsSynthesisRequested`
- `TtsSynthesisCompleted`
- `ProcessingJobFailed`
- `VoiceAssetRetentionExpired`
- `CronTaskTriggered`

**Verification:**

- For each event define payload, producer, consumer, persistence requirement.

---

## Phase 2 — Database Model

### Task 6: Draft Postgres ERD

**Objective:** Перевести DDD model в первичную ERD.

**Files:**

- Create: `docs/05-data/postgres-erd.md`

**Tables candidates:**

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
- `audit_log`

**Verification:**

- Каждая таблица привязана к bounded context.
- У каждой таблицы есть primary key, owner/user scope, timestamps.
- Нет frontend-only таблиц.

---

### Task 7: Define enum/status model

**Objective:** Спроектировать enum/check constraints для статусов и типов.

**Files:**

- Modify: `docs/05-data/postgres-erd.md`
- Modify: `docs/05-data/supabase-schema.md`

**Enums / checks:**

- `message_role`: `user`, `assistant`, `system`
- `message_input_type`: `text`, `voice`, `system`
- `reply_mode`: `text_only`, `text_and_voice`
- `job_status`: `ingested`, `transcribing`, `ready`, `answering`, `synthesizing`, `answered`, `error`
- `voice_asset_kind`: `user_original`, `assistant_tts`
- `cron_task_status`: `active`, `paused`, `disabled`, `error`

**Verification:**

- Status list matches `docs/03-api/api-contracts-mvp.md` or API doc is updated.

---

### Task 8: Design Supabase schema document

**Objective:** Подробно описать таблицы, поля, индексы, constraints.

**Files:**

- Create: `docs/05-data/supabase-schema.md`

**For each table include:**

- Purpose
- Columns
- Types
- Nullability
- Defaults
- Foreign keys
- Indexes
- Constraints
- RLS ownership basis

**Verification:**

- Можно написать SQL migration без дополнительных вопросов.

---

### Task 9: Design RLS policy model

**Objective:** Зафиксировать правила доступа до написания SQL.

**Files:**

- Create: `docs/05-data/rls-policy-model.md`

**Policies:**

- User can select own chats/messages/jobs/media metadata.
- User can insert messages only into own chats.
- User cannot directly insert assistant messages.
- User cannot arbitrarily update job statuses.
- Service role/worker can transition jobs and create assistant messages.
- Media access requires ownership and non-expired asset.

**Verification:**

- For each table define SELECT/INSERT/UPDATE/DELETE policy.
- Mark service-role-only operations.

---

### Task 10: Design Redis/job queue model

**Objective:** Определить, что живёт в Postgres, а что временно в Redis.

**Files:**

- Create: `docs/05-data/redis-job-queue-model.md`

**Rules:**

- Postgres is source of truth for job state.
- Redis is for queueing, locks, retries, and worker coordination.
- Redis data can be rebuilt from Postgres if lost.

**Queues:**

- `voice:stt`
- `assistant:run`
- `voice:tts`
- `retention:cleanup`
- `cron:dispatch`

**Verification:**

- Every Redis key has owner, TTL, rebuild strategy.

---

## Phase 3 — SQL Migration Draft

### Task 11: Create initial migration skeleton

**Objective:** Подготовить SQL-файл с типами и таблицами, не подключая frontend.

**Files:**

- Create: `supabase/migrations/0001_initial_domain.sql`

**Include:**

- extensions if needed: `pgcrypto`, maybe `uuid-ossp` only if chosen;
- enum types;
- tables;
- foreign keys;
- check constraints;
- indexes;
- updated_at trigger helper.

**Verification command:**

Run later when Supabase local setup is ready:

```bash
supabase db reset
```

Expected: migration applies cleanly.

---

### Task 12: Add database invariant checks

**Objective:** Защитить критичные правила на уровне БД.

**Files:**

- Modify: `supabase/migrations/0001_initial_domain.sql`

**Candidate DB checks:**

- `messages.role/input_type` compatible combinations.
- Voice messages require voice asset reference after ingest stage, or model explicit pending state.
- `processing_jobs.status` valid enum.
- `voice_assets.expires_at` not null for `user_original`.
- Unique active chat limit via trigger/function if simple constraint is insufficient.

**Verification:**

- SQL comments explain why each invariant is DB-level or not.

---

### Task 13: Add RLS SQL

**Objective:** Включить RLS и политики Supabase.

**Files:**

- Modify: `supabase/migrations/0001_initial_domain.sql`

**Include:**

- `alter table ... enable row level security;`
- `create policy ...` for each user-facing table;
- service role assumptions documented in comments.

**Verification:**

- Policies map back to `docs/05-data/rls-policy-model.md`.

---

### Task 14: Add seed/dev data

**Objective:** Подготовить минимальные dev fixtures для проверки PostgREST/API.

**Files:**

- Create: `supabase/seed.sql`

**Seed data:**

- one profile placeholder;
- one project;
- one chat;
- one text user message;
- one completed assistant result;
- one voice job example with fake media URLs.

**Verification:**

- Seed should not require production secrets.
- Seed data demonstrates text and voice paths.

---

## Phase 4 — API Alignment

### Task 15: Map tables to PostgREST resources

**Objective:** Связать Supabase tables/views/RPC с текущими API-контрактами.

**Files:**

- Create: `docs/03-api/postgrest-resource-map.md`
- Modify: `docs/03-api/api-contracts-mvp.md`

**Decide:**

- Which endpoints can be direct PostgREST.
- Which actions require RPC/functions.
- Which actions require Next.js route handler/server action because they include file upload, Redis, STT/TTS, or Hermes core.

**Likely split:**

- Direct PostgREST: list chats, list messages, list projects/plans/cron tasks.
- RPC/service: create chat with 5-chat invariant.
- Next.js/backend route: upload voice, submit text to assistant pipeline, media streaming.

**Verification:**

- No API endpoint contradicts RLS or source-of-truth model.

---

### Task 16: Update API contracts after DB design

**Objective:** Сделать API-контракты производными от DDD + DB, а не UI-first.

**Files:**

- Modify: `docs/03-api/api-contracts-mvp.md`

**Update:**

- IDs and field names to match schema.
- Status enum to match DB.
- Ownership model.
- Error codes.
- Media URL model.
- Job polling response.

**Verification:**

- Every API field maps to a table column, view, computed field, or service-produced response.

---

## Phase 5 — Readiness Gate Before Frontend

### Task 17: Create frontend readiness checklist

**Objective:** Явно зафиксировать, когда можно начинать frontend.

**Files:**

- Create: `docs/00-setup/frontend-readiness-gate.md`

**Checklist:**

- DDD context map approved.
- Domain model approved.
- Invariants mapped to DB/app/service levels.
- ERD approved.
- Supabase schema approved.
- RLS model approved.
- Redis/job queue model approved.
- API/PostgREST map approved.
- Initial migration applies locally.
- Seed data works.

**Rule:**

Frontend starts only after this checklist is complete or explicitly waived by the user.

---

### Task 18: Create DB validation checklist

**Objective:** Дать простые проверки без запуска frontend.

**Files:**

- Create: `docs/05-data/sql-validation-checklist.md`

**Checks:**

- Can create profile/project/chat/message.
- Cannot exceed 5 active chats.
- Cannot insert assistant message as normal user.
- Cannot read another user's chat.
- Can create voice asset with expiration.
- Retention query finds expired assets.
- Job state transitions are valid.

**Verification:**

- Checklist can be executed manually in Supabase SQL editor or converted to SQL tests later.

---

## Risks and Tradeoffs

### Risk: Next.js + Vite overlap

Both are in the stack. Need define responsibilities:

- Next.js App Router can serve app + route handlers.
- Vite may be used for isolated frontend package/storybook-style development or a separate PWA shell.

Open question: keep both in one monorepo, or choose one as primary runtime and one as tooling?

### Risk: PostgREST vs custom endpoints

PostgREST is good for DB resources, but audio upload/STT/TTS/Hermes core require service endpoints/workers.

Decision: direct DB resources via PostgREST where safe; side-effect-heavy operations through service routes/RPC.

### Risk: RLS complexity

RLS can become hard to debug. Document policies before SQL and keep service-role operations explicit.

### Risk: Redis source-of-truth drift

Redis must not become canonical. All durable state belongs in Postgres.

---

## Open Questions

1. Primary app runtime: Next.js App Router as main app, with Vite only for component/dev package, or separate Vite PWA frontend?
2. Supabase Auth: use directly, or wrap with separate `profiles/account_scopes` model from day one?
3. Storage: Supabase Storage for audio assets, or filesystem/object storage behind service endpoint?
4. Voice providers: STT/TTS provider choice now or later?
5. Should `domain_events` be append-only table in MVP or deferred until worker pipeline?
6. How strict should DB-level job state transition enforcement be in first migration?

---

## Immediate Next Step

Start with **Task 1: Create DDD context map**.

Do not create frontend files. Do not initialize UI. Do not implement app runtime until DDD and DB readiness gates are complete.
