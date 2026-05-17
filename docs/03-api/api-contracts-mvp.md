# PWA‑Hermes — API‑контракты MVP

## 1) Общие соглашения

Базовый префикс:

```text
/api
```

Форматы:

- JSON для обычных запросов и ответов;
- `multipart/form-data` для загрузки аудио;
- даты в ISO 8601;
- идентификаторы строковые: `chatId`, `messageId`, `jobId`, `projectId`.

Ошибки возвращаются единообразно:

```json
{
  "error": {
    "code": "chat_limit_reached",
    "message": "Можно создать не больше 5 активных чатов."
  }
}
```

## 2) Типы

### 2.1 Chat

```ts
type Chat = {
  chatId: string
  title: string | null
  projectId: string | null
  createdAt: string
  updatedAt: string
  latestJobStatus?: JobStatus
}
```

### 2.2 Message

```ts
type Message = {
  messageId: string
  chatId: string
  role: 'user' | 'assistant'
  inputType: 'text' | 'voice' | 'system'
  text: string | null
  transcriptText?: string | null
  assistantText?: string | null
  voiceOriginalUrl?: string | null
  voiceReplyUrl?: string | null
  jobId?: string | null
  createdAt: string
}
```

Правило отображения:

- для пользовательского текста UI показывает `text`;
- для пользовательского голоса UI показывает `transcriptText`, когда STT готов;
- для ассистента UI показывает `assistantText` и play по `voiceReplyUrl`.

### 2.3 Job

```ts
type JobStatus =
  | 'ingested'
  | 'transcribing'
  | 'ready'
  | 'answering'
  | 'synthesizing'
  | 'answered'
  | 'error'

type Job = {
  jobId: string
  chatId: string
  userMessageId: string
  assistantMessageId?: string | null
  status: JobStatus
  progress?: number | null
  errorCode?: string | null
  errorMessage?: string | null
  createdAt: string
  updatedAt: string
  result?: {
    transcriptText?: string | null
    assistantText?: string | null
    voiceReplyUrl?: string | null
  }
}
```

## 3) Чаты

### 3.1 Получить список чатов

```http
GET /api/chats
```

Ответ:

```json
{
  "chats": [
    {
      "chatId": "chat_123",
      "title": "Чат без названия",
      "projectId": null,
      "createdAt": "2026-05-17T10:00:00Z",
      "updatedAt": "2026-05-17T10:01:00Z",
      "latestJobStatus": "answered"
    }
  ],
  "limit": 5
}
```

### 3.2 Создать чат

```http
POST /api/chats
Content-Type: application/json
```

Запрос:

```json
{
  "title": "Новый чат",
  "projectId": null
}
```

Ответ:

```json
{
  "chat": {
    "chatId": "chat_124",
    "title": "Новый чат",
    "projectId": null,
    "createdAt": "2026-05-17T10:05:00Z",
    "updatedAt": "2026-05-17T10:05:00Z"
  }
}
```

Если лимит 5 чатов исчерпан:

```json
{
  "error": {
    "code": "chat_limit_reached",
    "message": "Можно создать не больше 5 активных чатов."
  }
}
```

### 3.3 Получить сообщения чата

```http
GET /api/chats/{chatId}/messages
```

Ответ:

```json
{
  "chatId": "chat_123",
  "messages": []
}
```

## 4) Сообщения

### 4.1 Отправить текст

```http
POST /api/chats/{chatId}/messages/text
Content-Type: application/json
```

Запрос:

```json
{
  "text": "Составь план на сегодня.",
  "replyMode": "text_and_voice"
}
```

Ответ:

```json
{
  "messageId": "msg_user_123",
  "jobId": "job_123",
  "jobStatus": "ingested"
}
```

### 4.2 Отправить голос

```http
POST /api/chats/{chatId}/messages/voice
Content-Type: multipart/form-data
```

Поля формы:

- `audio`: файл, например `audio/webm`, `audio/mp4`, `audio/mpeg`;
- `replyMode`: `text_and_voice` или `text_only`;
- `clientCreatedAt`: ISO 8601, опционально;
- `durationMs`: длительность записи, опционально.

Ответ:

```json
{
  "messageId": "msg_user_124",
  "jobId": "job_124",
  "jobStatus": "ingested"
}
```

Ограничения MVP:

- максимальная длительность аудио фиксируется серверной настройкой;
- UI должен показать ошибку, если браузер не дал доступ к микрофону;
- исходный аудиофайл хранится на сервере и удаляется по TTL.

## 5) Polling job

### 5.1 Получить состояние job

```http
GET /api/jobs/{jobId}
```

Пример в процессе:

```json
{
  "job": {
    "jobId": "job_124",
    "chatId": "chat_123",
    "userMessageId": "msg_user_124",
    "assistantMessageId": null,
    "status": "transcribing",
    "progress": null,
    "createdAt": "2026-05-17T10:10:00Z",
    "updatedAt": "2026-05-17T10:10:05Z"
  }
}
```

Пример готового результата:

```json
{
  "job": {
    "jobId": "job_124",
    "chatId": "chat_123",
    "userMessageId": "msg_user_124",
    "assistantMessageId": "msg_assistant_124",
    "status": "answered",
    "progress": 100,
    "createdAt": "2026-05-17T10:10:00Z",
    "updatedAt": "2026-05-17T10:10:30Z",
    "result": {
      "transcriptText": "Составь план на сегодня.",
      "assistantText": "Вот аккуратный план на сегодня...",
      "voiceReplyUrl": "/api/media/voice-replies/vr_124"
    }
  }
}
```

### 5.2 Polling‑правила UI

- После отправки сообщения UI poll’ит job каждые 2–3 секунды.
- Polling останавливается при `answered` или `error`.
- При `answered` UI обновляет список сообщений чата.
- При `error` UI показывает понятный текст и действие **Повторить** там, где это безопасно.

## 6) Медиа

### 6.1 Получить TTS‑ответ

```http
GET /api/media/voice-replies/{voiceReplyId}
```

Ответ: аудиофайл с корректным `Content-Type`.

### 6.2 Получить исходное аудио пользователя

```http
GET /api/media/user-voice/{voiceId}
```

MVP‑решение: endpoint может быть закрыт от UI, если не нужен playback исходника. Для отладки можно включить только в dev‑режиме.

## 7) Минимальные контуры проектов, планов и cron

### 7.1 Проекты

```http
GET /api/projects
POST /api/projects
```

Минимальный `Project`:

```ts
type Project = {
  projectId: string
  title: string
  createdAt: string
  updatedAt: string
}
```

### 7.2 Планы

```http
GET /api/plans
POST /api/plans
```

MVP: план может быть простой карточкой с названием, описанием и опциональной привязкой к `projectId`.

### 7.3 Cron

```http
GET /api/cron-tasks
POST /api/cron-tasks/{cronTaskId}/run
POST /api/cron-tasks/{cronTaskId}/pause
POST /api/cron-tasks/{cronTaskId}/resume
```

MVP: достаточно отобразить список и статусы, даже если фактическая интеграция с системным cron будет подключена позже.

## 8) Серверный pipeline

### 8.1 Для текста

```text
text input → create user message → create job → Hermes core → assistant text → optional TTS → answered
```

### 8.2 Для голоса

```text
audio upload → store original → create user message → create job → STT → Hermes core → assistant text → TTS → answered
```

## 9) Retention jobs

Серверный планировщик должен иметь задачу очистки:

- удалить исходные голосовые файлы старше TTL;
- опционально удалить TTS‑файлы старше TTL;
- не удалять текстовые transcript/assistantText без явного удаления чата.
