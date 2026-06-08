# Обзор проекта Wellin / FoodieAI

## Что такое Wellin

Wellin — приложение для питания и здоровья с AI-функциями.  
Прежнее рабочее название: **SmartFood / FoodieAI**.

Два репозитория в монорепо `/Wellin`:

| Репо | Путь | Роль |
|------|------|------|
| `foodieai` | `./foodieai/` | Backend (NestJS + Prisma + PostgreSQL) |
| `smart-food-plan` | `./smart-food-plan/` | Frontend монорепо (активная версия: `web-mui/`) |

---

## Основные возможности

### Для пользователя
- Дневной план питания — 4 слота (завтрак, обед, ужин, перекус)
- Учёт КБЖУ по целевым показателям (TDEE-формулы)
- База продуктов (GLOBAL / USER, с nutrition per 100g)
- Рецепты с ингредиентами и шагами
- Список покупок с категориями
- Self-care рутины (beauty / wellness) по дням недели
- Трекинг замеров тела
- Статистика нутриентов (день/неделя/месяц)
- **AI-агент** — единый чат-бот, встроенный во все разделы приложения

### Для бизнеса
- Email OTP аутентификация
- Paddle подписки (FREE → TRIAL → Pro)
- AI квоты per-user, per-feature
- Google Cloud Run деплой
- MCP JSON-RPC сервер для Claude

---

## Архитектура системы

```
[Пользователь]
       ↓
[Frontend: smart-food-plan/web-mui] (React 19 + MUI v7)
       ↓ HTTPS REST API (JWT Bearer)
[Backend: foodieai] (NestJS 10 + Prisma + PostgreSQL)
       ├── [OpenAI Responses API] (модели, AI-агент)
       ├── [Paddle API] (подписки, платежи)
       ├── [Google Cloud Storage] (изображения)
       └── [Resend → Cloud Run auth-email] (OTP email)
```

---

## Стек

### Backend (`foodieai/`)
- **Runtime**: Node.js 20, TypeScript 5.4
- **Framework**: NestJS 10
- **ORM**: Prisma 5.22 + PostgreSQL
- **Auth**: Email OTP → JWT Bearer
- **AI**: OpenAI Responses API или Chat Completions API; модель задаётся через `AGENT_MODEL` (по умолчанию `gpt-4o-mini`)
- **Billing**: Paddle
- **Storage**: Google Cloud Storage (изображения)
- **Docs**: Swagger на `/docs`
- **Deploy**: Google Cloud Run

### Frontend (`smart-food-plan/web-mui/`)
- **Framework**: React 19, TypeScript 5.9
- **UI**: Material-UI (MUI) v7 + Emotion
- **Сборка**: Vite 8
- **Роутинг**: React Router DOM v7
- **Состояние**: React Context + хуки (нет Redux/Zustand/TanStack Query)
- **i18n**: EN + RU
- **Deploy**: Vercel + GitHub Pages

---

## Деплой

| Сервис | URL |
|--------|-----|
| Backend (Cloud Run) | `https://foodieai-59215576464.me-west1.run.app` |
| Frontend (Vercel) | `https://smart-food-plan-web.vercel.app` |
| Frontend (GitHub Pages) | `https://irastefan.github.io` |
| Домен | `https://wellin.io` |

---

## Статус проекта (на 2026-06-08)

- Backend: активен, задеплоен, работает
- Frontend (web-mui): активен, задеплоен
- Frontend (web): старый прототип, НЕ активен
- Mobile (React Native): заготовка, НЕ активна
- AI-агент: LangGraph + SSE стриминг + multi-LLM поддержка
- MCP сервер: реализован, работает
- Billing: реализован через Paddle (past_due UI готов)
- SEO: robots.txt, sitemap.xml, OG/Twitter теги
- JWT refresh: автоматический (проактивный + реактивный)
- Тесты: частично (ts-node-based в `foodieai/test/`)

---

## Монорепо `smart-food-plan/`

```
web/        Старый прототип (React 18, НЕ активен)
web-mui/    АКТИВНАЯ версия (MUI, основная разработка)
mobile/     React Native (заготовка, НЕ активна)
shared/     Общие утилиты (i18n, theme, lib)
```

---

## Ссылки по проекту

- Подробная архитектура backend → [backend-architecture.md](./backend-architecture.md)
- Подробная архитектура frontend → [frontend-architecture.md](./frontend-architecture.md)
- База данных → [database-model.md](./database-model.md)
- AI-агент → [agent-current-implementation.md](./agent-current-implementation.md)
- Billing → [billing-and-subscriptions.md](./billing-and-subscriptions.md)
- Дизайн-система → [design-system.md](./design-system.md)
- Карта API → [api-map.md](./api-map.md)
- Технический долг → [cleanup-plan.md](./cleanup-plan.md)
- Готовность к запуску → [launch-readiness.md](./launch-readiness.md)
- Будущая миграция на LangGraph → [future-langgraph-migration.md](./future-langgraph-migration.md) _(LangGraph уже реализован, файл исторический)_
