# PWA‑Hermes — DDD Domain Model

## 0) Назначение документа

Этот документ описывает первичную доменную модель PWA‑Hermes перед проектированием Postgres/Supabase схемы.

Правило проекта:

> Не начинаем frontend, пока не сделана модель БД. Не делаем модель БД, пока не понятна доменная модель.

Источник контекста:

- `docs/01-spec/requirements-mvp.md`
- `docs/01-spec/ddd-context-map.md`
- `docs/00-setup/architecture-constraints.md`

---

## 1) Ubiquitous Language / Glossary

### 1.1 Identity & Access

#### AuthUser

**Определение:** внешний пользователь Supabase Auth (`auth.users`).

**Owner context:** Identity & Access.

**Lifecycle:** создаётся/управляется Supabase Auth; домен только ссылается на `auth.users.id`.

**DB mapping candidate:** `auth.users` external table; foreign key from `profiles.user_id`.

**Важно:** не хранить доменную логику внутри `auth.users`.

---

#### UserProfile

**Определение:** доменный профиль пользователя PWA‑Hermes, связывающий Supabase Auth с пользовательскими данными приложения.

**Owner context:** Identity & Access.

**Lifecycle:** создаётся при первом входе/инициализации пользователя; активен, пока существует пользователь.

**DB mapping candidate:** `profiles`.

**Основные поля-кандидаты:**

- `id`
- `user_id` → `auth.users.id`
- `display_name`
- `created_at`
- `updated_at`

**Связи:** владеет chats, projects, plans, cron tasks, jobs/media через user scope.

---

#### AccountScope

**Определение:** граница владения данными. В MVP может совпадать с `UserProfile`; позже может стать workspace/multi-user account.

**Owner context:** Identity & Access.

**Lifecycle:** в MVP можно не материализовать отдельной таблицей, если достаточно `profile_id`/`user_id` на всех owned tables.

**DB mapping candidate:**

- MVP option A: no table, use `profile_id`/`user_id` columns.
- Later option B: `account_scopes`.

**Решение для MVP:** начать с `profiles` как ownership root; `AccountScope` оставить как доменное понятие для будущего расширения.

---

### 1.2 Chats & Messages

#### Chat

**Определение:** пользовательская беседа с Hermes, содержащая упорядоченную историю сообщений.

**Owner context:** Chats & Messages.

**Lifecycle:** создаётся пользователем; активен до архивирования/удаления; ограничение MVP — до 5 активных чатов на пользователя.

**DB mapping candidate:** `chats`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `project_id` nullable
- `title`
- `status`: `active | archived | deleted`
- `created_at`
- `updated_at`
- `last_message_at`

**Инварианты:**

- один chat принадлежит одному user/profile;
- пользователь не может иметь больше 5 active chats;
- messages не существуют вне chat.

---

#### ConversationThread

**Определение:** логическое представление упорядоченной истории `Message` внутри `Chat`.

**Owner context:** Chats & Messages.

**Lifecycle:** не обязательно отдельная entity; может быть view/read model над `messages`.

**DB mapping candidate:** no table in MVP; possible view `chat_threads` / query over `messages`.

**Важно:** это не UI list, а доменное представление истории для assistant context и чтения пользователем.

---

#### Message

**Определение:** атомарная запись в истории чата: пользовательский ввод, ответ ассистента или системное сообщение.

**Owner context:** Chats & Messages.

**Lifecycle:** создаётся при отправке текста/голоса, ответе ассистента или системном событии; хранится в истории, пока не удалён chat/сообщение.

**DB mapping candidate:** `messages`.

**Основные поля-кандидаты:**

- `id`
- `chat_id`
- `profile_id` / `user_id` denormalized for RLS/indexing
- `role`: `user | assistant | system`
- `input_type`: `text | voice | system`
- `text` nullable
- `transcript_text` nullable
- `assistant_text` nullable or use `text` for assistant body — final decision in DB design
- `voice_asset_id` nullable
- `assistant_run_id` nullable
- `processing_job_id` nullable
- `created_at`

**Инварианты:**

- message всегда принадлежит одному chat;
- normal user не создаёт `assistant` message напрямую;
- user text message имеет `role=user`, `input_type=text`, `text not null`;
- user voice message имеет `role=user`, `input_type=voice`, связь с `VoiceAsset` или pending ingest state;
- assistant message создаётся как результат `AssistantRun`.

---

#### MessageAttachmentRef

**Определение:** ссылка message на media/domain asset, не сам binary.

**Owner context:** Chats & Messages.

**Lifecycle:** создаётся вместе с message или после обработки; может ссылаться на voice original, TTS asset, будущие файлы.

**DB mapping candidate:**

- MVP option A: direct nullable columns in `messages`: `voice_asset_id`, `voice_reply_asset_id`.
- Later option B: `message_attachments`.

**Решение для MVP:** начать с явных FK на `voice_assets`, если хватит двух типов audio; отдельную `message_attachments` не вводить без необходимости.

---

### 1.3 Voice Processing

#### VoiceAsset

**Определение:** metadata аудиофайла, связанного с сообщением или TTS‑ответом.

**Owner context:** Voice Processing.

**Lifecycle:** создаётся при audio upload или TTS synthesis; binary хранится во внешнем storage; metadata хранится в Postgres; user originals удаляются по retention.

**DB mapping candidate:** `voice_assets`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `chat_id` nullable but useful
- `message_id` nullable until message created / or set after transaction
- `kind`: `user_original | assistant_tts`
- `storage_bucket`
- `storage_path`
- `mime_type`
- `duration_ms`
- `size_bytes`
- `checksum` nullable
- `status`: `available | expired | deleted | failed`
- `expires_at` nullable for TTS depending policy; not null for user originals
- `deleted_at` nullable
- `created_at`

**Инварианты:**

- `user_original` has `expires_at`;
- deletion of voice asset binary does not delete transcript/message;
- asset belongs to exactly one profile/user scope.

---

#### VoiceProcessingResult

**Определение:** результат STT/TTS обработки аудио.

**Owner context:** Voice Processing.

**Lifecycle:** появляется после STT/TTS; в MVP может быть частью `processing_jobs.result` and/or columns on `messages`/`voice_assets`.

**DB mapping candidate:**

- STT transcript → `messages.transcript_text`
- TTS output → `voice_assets(kind=assistant_tts)` and assistant message reference
- processing metadata → `processing_jobs.result_json`

**Важно:** результат должен быть durable in Postgres, не только в Redis.

---

#### TranscriptionJob

**Определение:** специализированная стадия job для STT.

**Owner context:** Voice Processing.

**Lifecycle:** начинается после audio ingest, завершается transcript или error.

**DB mapping candidate:** no separate table in MVP; represented by `processing_jobs.status = transcribing` + result/error fields.

---

#### TtsSynthesisJob

**Определение:** специализированная стадия job для генерации голосового ответа.

**Owner context:** Voice Processing.

**Lifecycle:** начинается после assistant text, завершается `VoiceAsset(kind=assistant_tts)` или error.

**DB mapping candidate:** no separate table in MVP; represented by `processing_jobs.status = synthesizing` + `voice_assets`.

---

### 1.4 Assistant Interaction

#### AssistantRun

**Определение:** серверный запуск Hermes core над нормализованным входом и контекстом чата/проекта.

**Owner context:** Assistant Interaction.

**Lifecycle:** создаётся после готовности user input; завершается assistant result или error.

**DB mapping candidate:** `assistant_runs`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `chat_id`
- `input_message_id`
- `output_message_id` nullable until materialized
- `processing_job_id`
- `status`: `pending | running | completed | failed`
- `input_text`
- `assistant_text` nullable
- `model/provider metadata` nullable
- `error_code` nullable
- `error_message` nullable
- `started_at`
- `completed_at`
- `created_at`

**Инварианты:**

- assistant run belongs to one input message;
- completed assistant run should produce assistant text;
- assistant message should reference the run that produced it.

---

#### AssistantRequest

**Определение:** нормализованная команда/вход для Hermes core.

**Owner context:** Assistant Interaction.

**Lifecycle:** derived object, not necessarily persisted separately.

**DB mapping candidate:** no table in MVP; can be `assistant_runs.input_text` + optional JSON context.

---

#### AssistantResult

**Определение:** результат Hermes core: текст ответа and optional metadata.

**Owner context:** Assistant Interaction.

**Lifecycle:** created when assistant run completes; materialized into assistant message.

**DB mapping candidate:** `assistant_runs.assistant_text`, `messages` assistant row.

---

### 1.5 Processing Jobs & Queue Coordination

#### ProcessingJob

**Определение:** durable server-side lifecycle записи обработки входа пользователя: от ingest до answered/error.

**Owner context:** Processing Jobs & Queue Coordination.

**Lifecycle:** создаётся при submit text/voice; проходит state machine; завершённые jobs остаются для истории/диагностики.

**DB mapping candidate:** `processing_jobs`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `chat_id`
- `input_message_id`
- `assistant_message_id` nullable
- `assistant_run_id` nullable
- `status`: `ingested | transcribing | ready | answering | synthesizing | answered | error`
- `reply_mode`: `text_only | text_and_voice`
- `progress` nullable
- `result_json` nullable
- `error_code` nullable
- `error_message` nullable
- `attempt_count`
- `created_at`
- `updated_at`
- `completed_at` nullable

**Инварианты:**

- job state is durable in Postgres;
- Redis queue is not source of truth;
- `answered` requires assistant output;
- normal user cannot update status directly.

---

#### JobAttempt

**Определение:** отдельная попытка worker processing для retry/debug.

**Owner context:** Processing Jobs & Queue Coordination.

**Lifecycle:** создаётся worker’ом при попытке выполнить stage; завершается success/error/timeout.

**DB mapping candidate:** `job_attempts` later; can be deferred if `processing_jobs.attempt_count` enough for MVP.

**Решение для MVP:** deferred unless debugging/retry needs force table.

---

#### QueueMessageRef

**Определение:** временная ссылка на Redis queue item/lock.

**Owner context:** Processing Jobs & Queue Coordination.

**Lifecycle:** ephemeral; created in Redis, may be referenced in logs.

**DB mapping candidate:** no durable table in MVP; optionally `processing_jobs.queue_ref` for debug.

---

### 1.6 Projects

#### Project

**Определение:** пользовательская рабочая область/папка для группировки чатов, планов и cron tasks.

**Owner context:** Projects.

**Lifecycle:** создаётся пользователем; может быть active/archived.

**DB mapping candidate:** `projects`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `title`
- `description` nullable
- `status`: `active | archived | deleted`
- `created_at`
- `updated_at`

**Инварианты:**

- project belongs to one profile/user scope;
- linked chats/plans/cron tasks must belong to same profile/scope.

---

#### ProjectLink

**Определение:** связь project с другим доменным объектом.

**Owner context:** Projects.

**Lifecycle:** created when chat/plan/cron is assigned to project.

**DB mapping candidate:**

- MVP option A: direct nullable `project_id` in `chats`, `plans`, `cron_tasks`.
- Later option B: `project_links` for many-to-many/poly references.

**Решение для MVP:** direct `project_id` on owned tables.

---

### 1.7 Plans & Cron

#### Plan

**Определение:** пользовательский план/намерение/рабочая карточка, которая может быть связана с проектом и cron task.

**Owner context:** Plans & Cron.

**Lifecycle:** создаётся пользователем; может быть active/completed/archived.

**DB mapping candidate:** `plans`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `project_id` nullable
- `title`
- `body` nullable
- `status`: `active | completed | archived | deleted`
- `created_at`
- `updated_at`

**MVP:** минимальный контур, не блокирует chat/voice core.

---

#### CronTask

**Определение:** задача по расписанию, которую можно активировать, поставить на паузу, запустить вручную.

**Owner context:** Plans & Cron.

**Lifecycle:** создаётся пользователем/системой; запускается по schedule; порождает `CronRun`.

**DB mapping candidate:** `cron_tasks`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id`
- `project_id` nullable
- `plan_id` nullable
- `title`
- `schedule_expression`
- `timezone`
- `status`: `active | paused | disabled | error`
- `payload_json` nullable
- `last_run_at` nullable
- `next_run_at` nullable
- `created_at`
- `updated_at`

**Инварианты:**

- paused/disabled cron task не исполняется;
- schedule must be parseable by chosen scheduler;
- task and linked project/plan belong to same profile/scope.

---

#### CronRun

**Определение:** один факт запуска cron task.

**Owner context:** Plans & Cron.

**Lifecycle:** создаётся при dispatch; завершается success/error.

**DB mapping candidate:** `cron_runs`.

**Основные поля-кандидаты:**

- `id`
- `cron_task_id`
- `profile_id` / `user_id`
- `status`: `queued | running | succeeded | failed`
- `processing_job_id` nullable
- `result_json` nullable
- `error_code` nullable
- `error_message` nullable
- `started_at`
- `completed_at` nullable
- `created_at`

---

### 1.8 Media Retention

#### RetentionPolicy

**Определение:** правило удаления временных/binary assets.

**Owner context:** Media Retention.

**Lifecycle:** в MVP может быть config/policy constant; later table if user-configurable.

**DB mapping candidate:** no table in MVP; `voice_assets.expires_at` materializes policy result.

**MVP policy:** user original voice audio expires around 30 days after upload.

---

#### RetentionRun

**Определение:** один запуск cleanup job.

**Owner context:** Media Retention.

**Lifecycle:** created by cleanup worker; records deleted/failed counts.

**DB mapping candidate:** `retention_runs` later; can be `audit_log` event in MVP.

**Решение для MVP:** start with audit/domain event unless operational visibility requires table.

---

### 1.9 Audit & Operations

#### DomainEvent

**Определение:** append-only запись значимого доменного события.

**Owner context:** Audit & Operations.

**Lifecycle:** создаётся service/DB function/worker when domain action happens.

**DB mapping candidate:** `domain_events`.

**Основные поля-кандидаты:**

- `id`
- `profile_id` / `user_id` nullable for system events
- `event_type`
- `aggregate_type`
- `aggregate_id`
- `payload_json`
- `created_at`

**MVP decision:** полезно иметь с первой миграции, но можно сделать простым и не строить на нём всю систему.

---

#### AuditLogEntry

**Определение:** operational/audit запись для debugging/security.

**Owner context:** Audit & Operations.

**DB mapping candidate:** `audit_log` later or merge into `domain_events` for MVP.

**Решение для MVP:** использовать `domain_events` как минимальный append-only журнал; `audit_log` добавить позже при необходимости.

---

## 2) Value Objects

### JobStatus

**Values:**

- `ingested`
- `transcribing`
- `ready`
- `answering`
- `synthesizing`
- `answered`
- `error`

**DB mapping candidate:** Postgres enum `job_status`.

**Allowed high-level flow:**

```text
ingested → transcribing → ready → answering → synthesizing → answered
                   ↘ answering → answered        ↘ error
any processing state → error
```

Для text input можно пропустить `transcribing`:

```text
ingested → ready → answering → synthesizing → answered
```

Для `replyMode=text_only` можно пропустить `synthesizing`:

```text
ingested → ready → answering → answered
```

---

### MessageRole

**Values:** `user | assistant | system`

**DB mapping candidate:** Postgres enum `message_role`.

**Rules:**

- `user`: создаётся пользователем через text/voice submit.
- `assistant`: создаётся service role as result of `AssistantRun`.
- `system`: создаётся system/service for operational context, not normal UI submit.

---

### MessageInputType

**Values:** `text | voice | system`

**DB mapping candidate:** Postgres enum `message_input_type`.

**Rules:**

- `text` requires text body.
- `voice` requires voice asset/pending voice ingest and later transcript.
- `system` is service-created.

---

### ReplyMode

**Values:** `text_only | text_and_voice`

**DB mapping candidate:** Postgres enum `reply_mode`.

**MVP default:** `text_and_voice`.

---

### MediaKind

**Values:** `user_original | assistant_tts`

**DB mapping candidate:** Postgres enum `voice_asset_kind`.

**Rules:**

- `user_original`: uploaded recording, retention required.
- `assistant_tts`: generated audio answer, retention policy can be same or shorter/longer, final DB policy later.

---

### VoiceAssetStatus

**Values:** `available | expired | deleted | failed`

**DB mapping candidate:** Postgres enum `voice_asset_status`.

---

### RetentionTTL

**Definition:** duration after which expiring binary asset should be deleted.

**MVP value:** around 30 days for user original voice files.

**DB mapping candidate:** materialized as `voice_assets.expires_at`, not a dedicated type.

---

### ScheduleExpression

**Definition:** cron-like expression + timezone for scheduled tasks.

**DB mapping candidate:** `cron_tasks.schedule_expression` text + `cron_tasks.timezone` text.

**Rule:** parser/validator must reject invalid expressions before active scheduling.

---

### EntityStatus

**Values vary by aggregate:**

- Chat/Project/Plan: `active | archived | deleted`
- Plan additionally: `completed`
- CronTask: `active | paused | disabled | error`
- CronRun: `queued | running | succeeded | failed`

**DB mapping candidate:** separate enums/checks, not one universal enum if semantics differ.

---

## 3) Aggregates and ownership

### 3.1 UserProfile / AccountScope ownership boundary

**Aggregate root:** `UserProfile` for MVP ownership purposes.

**Child/owned concepts:** chats, projects, plans, cron tasks, jobs, media metadata.

**Allowed commands:**

- initialize profile;
- update display profile fields;
- deactivate/delete profile later.

**Transaction boundary:** small; most domain actions happen in their own aggregates but must carry `profile_id`/`user_id`.

**DB implication:** every user-owned table should include ownership column for RLS and indexes.

---

### 3.2 ChatAggregate

**Aggregate root:** `Chat`.

**Child entities:** `Message` metadata within the chat; direct references to `VoiceAsset` and `ProcessingJob`.

**Allowed commands:**

- `CreateChat(profileId, title?, projectId?)`
- `ArchiveChat(chatId)`
- `SubmitTextMessage(chatId, text, replyMode)`
- `AttachVoiceMessage(chatId, voiceAssetId, replyMode)`
- `AppendAssistantMessage(chatId, assistantRunId, assistantText, voiceReplyAssetId?)` service-only

**Invariants inside aggregate:**

- chat belongs to profile;
- active chat count limit checked at create;
- messages are ordered by created_at/id;
- assistant messages are service-created, not user-created.

**Transaction boundaries:**

- Create chat: one transaction with active count check.
- Submit text: create user message + processing job in one transaction.
- Attach voice: create voice asset + user message + processing job should be atomic from domain perspective, though binary upload may happen before final DB transaction.
- Append assistant result: assistant_run completion + assistant message + job status update should be transactionally consistent.

---

### 3.3 ProcessingJobAggregate

**Aggregate root:** `ProcessingJob`.

**Child entities:** possibly `JobAttempt` later.

**Allowed commands:**

- `CreateProcessingJob(inputMessageId, replyMode)`
- `MarkTranscribing(jobId)`
- `MarkReady(jobId, normalizedInput)`
- `MarkAnswering(jobId, assistantRunId)`
- `MarkSynthesizing(jobId)`
- `MarkAnswered(jobId, assistantMessageId, result)`
- `MarkFailed(jobId, error)`

**Invariants:**

- status transitions must be valid;
- `answered` requires assistant output;
- user cannot update job status;
- durable state lives in Postgres.

**Transaction boundaries:**

- Each stage transition is transactionally persisted.
- Worker may use Redis locks, but final state update is in Postgres.

---

### 3.4 VoiceAssetAggregate

**Aggregate root:** `VoiceAsset`.

**Child entities:** none in MVP; STT/TTS stage belongs to jobs.

**Allowed commands:**

- `RegisterUserOriginalAudio(profileId, chatId, storagePath, metadata)`
- `RegisterAssistantTtsAudio(profileId, chatId, messageId, storagePath, metadata)`
- `MarkAssetExpired(assetId)`
- `MarkAssetDeleted(assetId)`
- `MarkAssetFailed(assetId, error)`

**Invariants:**

- user original has retention deadline;
- asset ownership matches chat/message ownership;
- binary deletion does not remove transcript/message.

**Transaction boundaries:**

- Register metadata after upload succeeds or as pending before upload finalization — final DB design decides.
- Retention deletion: mark metadata after storage delete attempt, with audit event.

---

### 3.5 AssistantRunAggregate

**Aggregate root:** `AssistantRun`.

**Allowed commands:**

- `CreateAssistantRun(inputMessageId, normalizedInput)`
- `MarkAssistantRunRunning(runId)`
- `CompleteAssistantRun(runId, assistantText)`
- `FailAssistantRun(runId, error)`

**Invariants:**

- one run belongs to one input message and chat;
- completed run has assistant text;
- assistant message references completed run.

**Transaction boundaries:**

- Completion should coordinate with creating assistant message and updating processing job.

---

### 3.6 ProjectAggregate

**Aggregate root:** `Project`.

**Allowed commands:**

- `CreateProject(profileId, title)`
- `UpdateProject(projectId, fields)`
- `ArchiveProject(projectId)`
- `LinkChatToProject(chatId, projectId)`
- `LinkPlanToProject(planId, projectId)`

**Invariants:**

- all linked objects belong to same profile/scope;
- archived/deleted projects should not accept new links unless explicitly allowed.

---

### 3.7 PlanAggregate

**Aggregate root:** `Plan`.

**Allowed commands:**

- `CreatePlan(profileId, title, body?, projectId?)`
- `UpdatePlan(planId, fields)`
- `CompletePlan(planId)`
- `ArchivePlan(planId)`

**Invariants:**

- plan belongs to profile;
- linked project belongs to same profile.

---

### 3.8 CronTaskAggregate

**Aggregate root:** `CronTask`.

**Child entities:** `CronRun`.

**Allowed commands:**

- `CreateCronTask(profileId, schedule, payload)`
- `PauseCronTask(cronTaskId)`
- `ResumeCronTask(cronTaskId)`
- `DisableCronTask(cronTaskId)`
- `RecordCronRun(cronTaskId)`
- `MarkCronRunSucceeded(cronRunId)`
- `MarkCronRunFailed(cronRunId, error)`

**Invariants:**

- paused/disabled tasks are not dispatched;
- schedule expression is valid;
- cron run belongs to cron task’s profile;
- cron execution result must be durable in Postgres.

---

## 4) DB mapping candidates summary

| Domain term | Table / DB object candidate | Notes |
| --- | --- | --- |
| AuthUser | `auth.users` | Supabase external identity |
| UserProfile | `profiles` | Domain profile, RLS ownership root |
| AccountScope | no table in MVP / later `account_scopes` | Use profile ownership first |
| Chat | `chats` | Active chat limit applies |
| ConversationThread | view/query over `messages` | No table in MVP |
| Message | `messages` | User/assistant/system messages |
| MessageAttachmentRef | direct FKs or later `message_attachments` | Prefer direct voice asset refs in MVP |
| VoiceAsset | `voice_assets` | Metadata only; binary in storage |
| VoiceProcessingResult | columns/result JSON | Transcript + TTS asset references |
| AssistantRun | `assistant_runs` | Hermes core execution record |
| ProcessingJob | `processing_jobs` | Durable job lifecycle |
| JobAttempt | later `job_attempts` | Defer unless needed |
| Project | `projects` | Minimal MVP structure |
| ProjectLink | direct `project_id` columns | Defer polymorphic links |
| Plan | `plans` | Minimal MVP structure |
| CronTask | `cron_tasks` | Schedule/status |
| CronRun | `cron_runs` | Execution history |
| RetentionPolicy | config / materialized `expires_at` | No table in MVP |
| RetentionRun | domain event / later table | Defer table |
| DomainEvent | `domain_events` | Minimal append-only log |
| AuditLogEntry | no table in MVP / later `audit_log` | Use domain_events first |

---

## 5) Commands by scenario

### Create chat

```text
CreateChat(profileId, title?, projectId?)
```

Effects:

- validate active chat count < 5;
- insert `chats`;
- optionally emit `ChatCreated`.

Likely implementation:

- SQL function/RPC to enforce limit transactionally.

---

### Submit text message

```text
SubmitTextMessage(chatId, text, replyMode=text_and_voice)
```

Effects:

- insert user message;
- insert processing job with `ingested` or `ready` depending pipeline decision;
- enqueue assistant run via Redis/service;
- emit `UserTextMessageSubmitted`.

Likely implementation:

- service route or SQL RPC + worker enqueue.

---

### Submit voice message

```text
SubmitVoiceMessage(chatId, audioFile, replyMode=text_and_voice)
```

Effects:

- upload binary;
- create `voice_assets(kind=user_original, expires_at)`;
- insert user voice message;
- insert processing job with `ingested`;
- enqueue STT;
- emit `UserVoiceMessageUploaded`.

Likely implementation:

- service route, not pure PostgREST, because file upload + queue.

---

### Complete assistant answer

```text
CompleteAssistantRun(processingJobId, assistantText, voiceReplyAssetId?)
```

Effects:

- update `assistant_runs`;
- create assistant message;
- link TTS asset if present;
- update processing job to `answered`;
- emit `AssistantRunCompleted` and maybe `TtsSynthesisCompleted`.

Likely implementation:

- service/worker transaction.

---

### Retention cleanup

```text
ExpireVoiceAssets(now)
```

Effects:

- find expired voice assets;
- delete storage objects;
- mark metadata expired/deleted;
- emit retention/audit event.

Likely implementation:

- scheduled worker/cron service, not frontend.

---

## 6) Decisions made for MVP

1. `AccountScope` is a domain concept but not a separate MVP table unless DB design shows immediate need.
2. Assistant output should materialize as an `assistant` message, not only live inside job result.
3. TTS output should use `VoiceAsset(kind=assistant_tts)` in the same `voice_assets` table as user originals.
4. `domain_events` is a useful first migration candidate, but can remain minimal.
5. `message_attachments` and `job_attempts` are deferred unless schema design proves they are needed.
6. Redis is never canonical; all durable state is in Postgres.
7. Frontend cannot own domain transitions; it sends commands and renders DB/service state.

---

## 7) Open questions before invariants / DB schema

1. Should active chat limit count archived chats or only `status=active`? Current assumption: only active.
2. Should TTS assets have the same 30-day retention as user originals, or a different policy?
3. Should `processing_jobs` begin text flow at `ingested` or directly at `ready`? Current assumption: create `ingested`, then worker/service transitions to `ready`.
4. Should `assistant_runs.input_text` duplicate message text/transcript for audit reproducibility? Current assumption: yes, store normalized input snapshot.
5. Should `messages.text` hold assistant text too, or separate `assistant_text` column? Current assumption for DB design: prefer one `body_text`/`text` column plus role; keep transcript in separate `transcript_text` for voice input.

---

## 8) Следующий документ

Следующий шаг: создать `docs/01-spec/ddd-invariants.md`.

Он должен перевести правила из этой модели в enforcement matrix:

- DB constraint
- index/unique constraint
- trigger/function/RPC
- RLS policy
- service/worker rule
- frontend display-only rule
