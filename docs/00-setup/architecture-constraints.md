# PWA‑Hermes — архитектурные ограничения и стек

## Технологический стек

Зафиксированный стек проекта:

- Supabase
- PostgREST
- Redis
- Vite + React + TypeScript
- Next.js App Router
- shadcn/ui
- Tailwind CSS
- Postgres как основная база данных

## Архитектурный принцип

UI может быть большим, но frontend должен оставаться тонким и «тупым»: минимум бизнес-логики на клиенте, максимум явной модели в домене и базе данных.

**База данных — всему голова.**

Postgres/Supabase — центр архитектуры, источник истины и место, от которого проектируется приложение.

## Порядок работы

1. Не начинать реализацию, пока не определён DDD: bounded contexts, aggregates/entities/value objects, domain events, invariants.
2. Не начинать frontend, пока не сделана модель базы данных.
3. Сначала проектируем домен.
4. Затем проектируем Postgres/Supabase schema, RLS, связи, статусы, job-модель и retention.
5. Только после этого делаем API и frontend.

## Следствие для MVP

Перед любым UI-каркасом нужно подготовить:

- DDD-карту домена PWA‑Hermes;
- ERD / модель таблиц Postgres;
- Supabase/PostgREST контракт;
- правила RLS и ownership;
- модель job processing через Postgres + Redis;
- retention‑политику для голосовых файлов.
