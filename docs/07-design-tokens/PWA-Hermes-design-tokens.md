# PWA‑Hermes — Design Tokens (UI, монохром + семантика)

> Цель: фиксировать дизайн токены так, чтобы фронт не “дрейфовал” и чтобы shadcn/ui можно было использовать предсказуемо.

## 1) База (light-only, без градиентов/залива)
- **Только светлая тема**.
- Основа: монохромные шкалы (затемнения/контраст), акцент только один и **семантический** (CTA / статус).
- **Запрещено** на UI: градиенты, цветные fill’ы, тёмные темы.

## 2) Typography
Используем согласованный набор:
- Display / Titles: `Unbounded` (или системный fallback)
- Body: `Inter` (или `system-ui`)
- Mono (кейсы/коды/тех-текст): `JetBrains Mono`

Семантика:
- `font-title`: для заголовков
- `font-body`: для текста
- `font-mono`: для технических вставок/ID

## 3) Space scale
Базовая шкала (пример):
- `space-1`: 4px
- `space-2`: 8px
- `space-3`: 12px
- `space-4`: 16px
- `space-5`: 20px
- `space-6`: 24px
- `space-7`: 32px
- `space-8`: 40px

Правило: компоненты используют токены `space-*` а не “магические” числа.

## 4) Radius
- `radius-sm`: 8px
- `radius-md`: 12px
- `radius-lg`: 16px

## 5) Border
- `border-base`: 1px solid `color-border`
- `border-radius`: как в radius-* для компонента

## 6) Color system (strict shadcn palette)

> Мы фиксируем основу на **shadcn/ui** монохромной palette: `zinc / neutral / stone`.
> Внутри приложения цвета используются только как **семантические токены**.

### 6.1 Neutral шкала
Цель: без своих hex-значений — только маппинг на shadcn нейтралы.

Используем:
- `bg`: `zinc` (или `neutral`) — самые светлые уровни
- `surface`: `stone` (или `zinc`) — чуть отличающиеся светлые поверхности
- `text`: `zinc` — тёмные уровни
- `muted`: `zinc` — средние уровни
- `border`: `zinc`/`neutral` — светлые границы
- `ring`: `zinc` — для фокуса

Как семантические токены маппятся на shadcn:
- `color-bg` → `zinc-50`
- `color-surface` → `zinc-100` или `stone-50`
- `color-text` → `zinc-900`
- `color-muted` → `zinc-600`
- `color-border` → `zinc-200` (или `neutral-200`)
- `color-ring` → `zinc-300`

> Если у тебя уже есть стандарт по конкретным уровням zinc/neutral/stone — скажи, я подгоню mapping под него.

### 6.2 Accent (один)
Акцент **строго** из монохромных уровней shadcn (CTA/ключевые состояния):
- `color-accent` → `zinc-900` (или `neutral-900`)
- `color-accent-text` → `zinc-50` / `white` (если нужно)

### 6.3 Status colors (минимум)
Только функциональные статусы, без “радуги”.

В MVP можно:
- `status-processing`: выделение фокусом/бордером из нейтралов
- `status-ready`: иконка/текст без цветных заливок
- `status-error`: danger-иконка из нейтралов + текст “ошибка” (или красный только если реально нельзя иначе)

> В документе не используем заранее зафиксированные hex — всё через shadcn уровни (zinc/neutral/stone).


## 7) Shadows
Избегаем “тяжёлых” теней и цветных тюнингом.
- `shadow-sm`: лёгкая контрастная
- `shadow-md`: умеренная

## 8) Component tokens (минимальные требования)

### 8.1 Buttons
- Primary CTA:
  - bg: `color-accent`
  - text: `color-accent-text`
  - hover: чуть светлее/темнее нейтраль
- Secondary:
  - bg: `color-surface`
  - border: `color-border`

### 8.2 Cards (чатовые карточки)
- border: `color-border`
- radius: `radius-md`
- padding: `space-4/5`
- background: `color-surface`

### 8.3 Chat bubbles
- `bubble-user`: surface нейтраль + border
- `bubble-assistant`: surface светлее или border слабее

## 9) Токены для состояния обработки job’a
- `status-idle`: muted
- `status-processing`: акцент/анимация без цветной заливки
- `status-ready`: success символ (минимально)
- `status-error`: danger текст/иконка

## 10) Движение (анимации)
- микровзаимодействия только там, где смысл: фокус, нажатие, появление ответа.
- длительности: 150–250ms

## 11) Привязка к shadcn/ui
- Компоненты выбирают palette через кастомизацию `tailwind.config`.
- В документе фиксируются значения семантических токенов — любые “новые цвета” требуют согласования.
