# PWA‑Hermes — DDD Context Map

## 0) Назначение документа

Этот документ фиксирует первичную DDD‑карту домена PWA‑Hermes до начала frontend/backend реализации.

Главное архитектурное правило проекта:

> UI может быть большим, но frontend остаётся тонким. Источник истины — Postgres/Supabase. Сначала домен и БД, потом API и frontend.

MVP: мультимодальное общение с Hermes вне Telegram: текст + голос, до 5 параллельных чатов, голосовой pipeline audio → STT → Hermes core → TTS, без стриминга, с polling по job.

---

## 1) Карта bounded contexts

### 1.1 Core domain

#### Chats & Messages

**Роль:** основной домен MVP.

**Почему core:** пользовательская ценность PWA‑Hermes проявляется через управляемые чаты, историю сообщений, текстовый и голосовой вход, ответы ассистента и связь с проектами/планами.

**Основные сущности-кандидаты:**

- `Chat`
- `Message`
- `ConversationThread`
- `MessageAttachmentRef`

**Ответственность:**

- создание и лимитирование чатов;
- хранение истории сообщений;
- различение ролей сообщений: user / assistant / system;
- различение входов: text / voice / system;
- связь сообщений с jobs и результатами обработки;
- предоставление читаемой истории для UI и Hermes core.

**Не отвечает за:**

- STT/TTS детали;
- binary storage аудио;
- реализацию cron;
- auth provider;
- low-level Redis queue.

---

### 1.2 Supporting domains

#### Voice Processing

**Роль:** supporting domain для обработки аудио.

**Основные сущности-кандидаты:**

- `VoiceAsset`
- `TranscriptionJob`
- `TtsSynthesisJob`
- `VoiceProcessingResult`

**Ответственность:**

- приём аудио после загрузки;
- хранение metadata по аудио;
- запуск STT;
- сохранение transcript;
- запуск TTS для ответа ассистента;
- фиксация ссылок на user original и assistant TTS;
- подготовка данных для retention.

**Не отвечает за:**

- смысл ответа Hermes;
- создание assistant message как доменного результата;
- UI‑preview записи;
- auth ownership, кроме проверки через связанные агрегаты.

**Ключевой принцип:** frontend только записывает/загружает аудио и показывает статус. Вся обработка голоса — серверная.

---

#### Assistant Interaction

**Роль:** supporting domain, интеграционная граница с Hermes core.

**Основные сущности-кандидаты:**

- `AssistantRun`
- `AssistantRequest`
- `AssistantResult`
- `ToolInvocationRef` later, не в MVP

**Ответственность:**

- подготовка нормализованного входа для Hermes core;
- запуск assistant run;
- сохранение assistant text;
- создание/привязка assistant message;
- передача текста ответа в TTS при `replyMode = text_and_voice`.

**Не отвечает за:**

- UI‑форматирование;
- raw audio storage;
- cron scheduling как самостоятельную бизнес‑область.

---

#### Projects

**Роль:** supporting domain для организации чатов, планов и будущих рабочих материалов.

**Основные сущности-кандидаты:**

- `Project`
- `ProjectLink`
- `ProjectScope`

**Ответственность:**

- хранение минимальной проектной структуры;
- привязка чатов/планов/cron tasks к проектам;
- выбор текущего проекта в UI.

**MVP‑ограничение:** проектный контур минимальный. Он не должен блокировать голосовой/chat core.

---

#### Plans & Cron

**Роль:** supporting domain для планов и расписаний.

**Основные сущности-кандидаты:**

- `Plan`
- `CronTask`
- `CronRun`
- `ScheduleExpression`

**Ответственность:**

- хранение пользовательских планов;
- хранение cron tasks;
- статусы задач: active / paused / disabled / error;
- запуск по расписанию или вручную;
- связь с проектами и, позже, с чатами/assistant runs.

**Не отвечает за:**

- обычную chat history;
- STT/TTS;
- rendering UI.

**MVP‑ограничение:** допускается минимальный CRUD/контур, но доменная граница должна быть отдельной с самого начала.

---

#### Media Retention

**Роль:** policy/service boundary для удаления голосовых файлов и других временных media.

**Основные сущности-кандидаты:**

- `RetentionPolicy`
- `RetentionRun`
- `ExpiringAssetRef`

**Ответственность:**

- расчёт `expiresAt` для voice assets;
- поиск просроченных оригиналов;
- удаление user original audio примерно через 30 дней;
- сохранение transcript/assistantText даже после удаления аудио;
- аудит удаления.

**Не отвечает за:**

- создание сообщений;
- STT/TTS processing;
- бизнес‑смысл чата.

---

### 1.3 Generic / infrastructure contexts

#### Identity & Access

**Роль:** generic subdomain на базе Supabase Auth + проектной модели `profiles`/`account_scopes`.

**Основные сущности-кандидаты:**

- `AuthUser` external / Supabase Auth
- `Profile`
- `AccountScope` later, если понадобится workspace/multi-user

**Ответственность:**

- связь `auth.users.id` с доменным профилем;
- ownership данных;
- RLS‑основа для таблиц;
- базовая идентификация пользователя.

**Ключевое решение:** доменная модель не растворяется в `auth.users`. Supabase Auth — identity provider, а `profiles`/scope — доменная точка привязки.

---

#### Processing Jobs & Queue Coordination

**Роль:** infrastructure/supporting boundary между Postgres и Redis.

**Основные сущности-кандидаты:**

- `ProcessingJob`
- `JobAttempt`
- `QueueMessageRef`
- Redis queue keys

**Ответственность:**

- durable job state в Postgres;
- временные очереди/locks/retries в Redis;
- state machine: `ingested → transcribing → ready → answering → synthesizing → answered` или `error`;
- polling state для UI.

**Ключевое решение:** Redis не является source of truth. Если Redis потерян, job state восстанавливается из Postgres.

---

#### Audit & Operations

**Роль:** infrastructure/generic context для прозрачности действий и отладки.

**Основные сущности-кандидаты:**

- `DomainEvent`
- `AuditLogEntry`
- `OperationalError`

**Ответственность:**

- append-only события важных действий;
- отладка worker/job pipeline;
- аудит удаления media;
- диагностика ошибок STT/TTS/Hermes/cron.

**MVP‑решение:** можно начать с простой `domain_events` или `audit_log` таблицы, но не превращать её в бизнес‑центр.

---

## 2) Relationships между context’ами

```text
Identity & Access
  └─ owns/scopes → Chats & Messages
  └─ owns/scopes → Projects
  └─ owns/scopes → Plans & Cron

Projects
  └─ groups → Chats & Messages
  └─ groups → Plans & Cron

Chats & Messages
  └─ creates → Processing Jobs & Queue Coordination
  └─ references → Voice Processing
  └─ requests → Assistant Interaction

Voice Processing
  └─ updates → Processing Jobs & Queue Coordination
  └─ produces transcript → Chats & Messages
  └─ produces TTS asset → Chats & Messages
  └─ registers expiring media → Media Retention

Assistant Interaction
  └─ reads context from → Chats & Messages
  └─ creates assistant result → Chats & Messages
  └─ may request TTS → Voice Processing

Plans & Cron
  └─ can trigger → Assistant Interaction
  └─ can create operational entries in → Processing Jobs & Queue Coordination

Media Retention
  └─ deletes media from → Voice Processing storage
  └─ writes event to → Audit & Operations

Audit & Operations
  └─ observes events from all contexts
```

---

## 3) MVP сценарии через context map

### 3.1 Создать чат

1. `Identity & Access` определяет пользователя/scope.
2. `Chats & Messages` проверяет invariant: не больше 5 активных чатов.
3. `Chats & Messages` создаёт `Chat`.
4. `Audit & Operations` может записать `ChatCreated`.

**Frontend role:** отправить команду и отобразить результат.

---

### 3.2 Отправить текст

1. `Identity & Access` проверяет ownership чата через RLS/service.
2. `Chats & Messages` создаёт user `Message(inputType=text)`.
3. `Processing Jobs & Queue Coordination` создаёт durable job `ingested`.
4. `Assistant Interaction` получает нормализованный текст.
5. `Assistant Interaction` создаёт assistant result.
6. `Chats & Messages` сохраняет assistant message.
7. `Voice Processing` создаёт TTS asset, если replyMode = `text_and_voice`.
8. `Processing Jobs & Queue Coordination` переводит job в `answered`.

**Frontend role:** отправить текст, poll по `jobId`, показать историю.

---

### 3.3 Отправить голос

1. `Identity & Access` проверяет ownership чата.
2. `Voice Processing` принимает audio upload и создаёт `VoiceAsset(kind=user_original)` с `expiresAt`.
3. `Chats & Messages` создаёт user `Message(inputType=voice)` со ссылкой на voice asset.
4. `Processing Jobs & Queue Coordination` создаёт durable job `ingested`.
5. `Voice Processing` запускает STT и сохраняет transcript.
6. `Assistant Interaction` получает transcript как нормализованный вход.
7. `Assistant Interaction` создаёт assistant result.
8. `Chats & Messages` сохраняет assistant message.
9. `Voice Processing` создаёт TTS asset для ответа.
10. `Media Retention` получает expiring media metadata.
11. `Processing Jobs & Queue Coordination` переводит job в `answered`.

**Frontend role:** записать аудио, загрузить, poll по `jobId`, показать transcript/text/play.

---

### 3.4 Retention голосовых файлов

1. `Media Retention` выбирает `VoiceAsset` с истёкшим `expiresAt`.
2. Storage/service удаляет binary object.
3. `Voice Processing` metadata помечается как deleted/expired.
4. `Chats & Messages` сохраняет transcript и историю.
5. `Audit & Operations` записывает событие удаления.

**Frontend role:** не управляет retention, только корректно отображает отсутствие старого audio playback.

---

### 3.5 Cron / планировщик

1. `Plans & Cron` хранит `CronTask` и состояние расписания.
2. Cron dispatcher/worker создаёт событие запуска.
3. `Processing Jobs & Queue Coordination` создаёт durable job/run.
4. `Assistant Interaction` может выполнить задачу через Hermes core.
5. Результат связывается с `Project`, `Plan`, `CronRun` или отдельным chat/system message — точный mapping определить в DB model.

**Frontend role:** показать список, статусы, паузу/возобновление/ручной запуск.

---

## 4) Context classification

- **Core domain:** `Chats & Messages`.
- **Supporting domains:** `Voice Processing`, `Assistant Interaction`, `Projects`, `Plans & Cron`, `Media Retention`, `Processing Jobs & Queue Coordination`.
- **Generic/infrastructure:** `Identity & Access`, `Audit & Operations`.

---

## 5) Integration styles

### 5.1 Direct DB / PostgREST

Подходит для:

- чтения chats/messages/projects/plans/cron tasks;
- простых пользовательских inserts, если RLS и invariants безопасны;
- отображения job state.

### 5.2 RPC / database function

Подходит для:

- создания chat с проверкой лимита 5 активных чатов;
- безопасного создания user text message + processing job в одной транзакции;
- контролируемых transitions, если решим делать их DB‑enforced.

### 5.3 Service route / worker

Нужен для:

- audio upload;
- STT/TTS;
- Hermes core interaction;
- Redis queue;
- media streaming/download;
- retention cleanup.

---

## 6) Что обязана защищать БД

Эти правила должны быть спроектированы как DB/RLS/trigger/function invariants, если возможно:

- пользователь видит только свои chats/messages/jobs/media metadata;
- пользователь не может создать больше 5 активных чатов;
- пользователь не может напрямую создать assistant message;
- пользователь не может произвольно менять `processing_jobs.status`;
- voice original всегда имеет `expiresAt`;
- удаление voice original не удаляет transcript;
- Redis state не является единственным источником job status.

---

## 7) Проверка покрытия MVP

| MVP сценарий | Context owner | Supporting contexts |
| --- | --- | --- |
| Создать чат | Chats & Messages | Identity & Access, Audit & Operations |
| Отправить текст | Chats & Messages | Assistant Interaction, Processing Jobs, Voice Processing for TTS |
| Отправить голос | Chats & Messages | Voice Processing, Assistant Interaction, Processing Jobs, Media Retention |
| Polling job | Processing Jobs | Chats & Messages |
| Play TTS ответ | Voice Processing | Chats & Messages, Identity & Access |
| Проекты | Projects | Identity & Access |
| Планы | Plans & Cron | Projects, Assistant Interaction later |
| Cron | Plans & Cron | Processing Jobs, Assistant Interaction, Audit & Operations |
| Retention audio | Media Retention | Voice Processing, Audit & Operations |

Вывод: каждый MVP‑сценарий покрыт bounded context’ом. Voice pipeline остаётся серверным и не размазывается по frontend.

---

## 8) Open questions перед domain model

1. Нужен ли `AccountScope/Workspace` уже в MVP, или достаточно `profiles.user_id`?
2. Assistant result должен всегда создавать отдельное `assistant` message, или может быть result внутри job до материализации?
3. TTS asset хранить как `VoiceAsset(kind=assistant_tts)` вместе с user originals или отдельной таблицей?
4. Нужна ли append-only `domain_events` таблица в первой миграции, или хватит `audit_log`?
5. Cron result должен попадать в chat history, project activity, отдельный cron_run result, или все три через связи?

---

## 9) Следующий документ

Следующий шаг: создать `docs/01-spec/ddd-domain-model.md` с glossary, entities, value objects, aggregates и первичными DB mapping candidates.
