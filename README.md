# dev-pwa-hermes (PWA‑Hermes)

Рабочий фронт/интеграционный слой для **PWA‑Hermes** — MVP мультимодальный (текст + голос).

PWA‑Hermes предназначен для общения с Hermes **не через Telegram**:
- кнопка **«Записать аудио послание»**
- аудио → **STT** → Hermes core → **TTS**
- до **5 параллельных чатов**
- обязательные пользовательские контуры: **cron**, **планировщик**, **проекты**

Документация проекта (спецификации/контракты):
- `docs/00-setup/architecture-constraints.md` — стек и архитектурные ограничения
- `docs/01-spec/requirements-mvp.md` — канонические требования MVP
- `docs/01-spec/ddd-context-map.md` — DDD context map
- `docs/01-spec/ddd-domain-model.md` — DDD domain model / glossary / aggregates
- `docs/03-api/api-contracts-mvp.md` — API‑контракты PWA ↔ сервер
- `docs/02-ui/ui-flow-and-screens-mvp.md` — UI flow и экраны
- `docs/07-design-tokens/PWA-Hermes-design-tokens.md` — дизайн‑токены

## Ключевые решения (MVP)
- **MVP без стриминга**: пользователь видит результат после завершения обработки (polling).
- Ответ по умолчанию: **текст + голос (TTS)**.

## Быстрый старт (как делать проект на старте)
1. Сначала фиксируем **DDD**: context map → domain model → invariants → events.
2. Затем проектируем **модель БД**: Postgres/Supabase schema, RLS, Redis/job queue.
3. После этого выравниваем **PostgREST/API** под модель БД.
4. Только после readiness gate начинаем backend/frontend реализацию.

## Трекер артефактов
- Архитектурные ограничения: `docs/00-setup/architecture-constraints.md`
- Канонические требования: `docs/01-spec/requirements-mvp.md`
- DDD context map: `docs/01-spec/ddd-context-map.md`
- DDD domain model: `docs/01-spec/ddd-domain-model.md`
- API‑контракты: `docs/03-api/api-contracts-mvp.md`
- UI flow: `docs/02-ui/ui-flow-and-screens-mvp.md`

## Архитектурные ограничения
- Зафиксированный стек: Supabase, PostgREST, Redis, Vite + React + TypeScript, Next.js App Router, shadcn/ui, Tailwind CSS.
- Frontend должен быть тонким: UI может быть большим, но бизнес-логика и источник истины — в домене и Postgres/Supabase.
- Не начинаем реализацию, пока не определили DDD.
- Не начинаем frontend, пока не сделали модель БД.

Подробно: `docs/00-setup/architecture-constraints.md`.

