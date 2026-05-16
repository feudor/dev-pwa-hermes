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

## 6) Color system

### 6.1 Neutral шкала (используем shadcn-like)
Поставим целевые нейтралы в семантические токены:
- `color-bg`: `#ffffff`
- `color-surface`: `#f8fafc` (очень светлая поверхность)
- `color-text`: `#0b0f19`
- `color-muted`: `#5b6577`
- `color-border`: `#e5e7eb`
- `color-ring`: `#d1d5db`

### 6.2 Accent (один)
Акцент задаём одним семантическим цветом:
- `color-accent`: `#111827` (под пример “CTA zinc/neutral”)
- `color-accent-text`: `#ffffff`

### 6.3 Status colors (минимум)
Без радужности, только функциональные статусы:
- `color-success` / `color-danger` / `color-warning` — только если UI реально требует.

> На MVP можно ограничиться текстовыми бейджами + иконками без насыщенной раскраски.

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
