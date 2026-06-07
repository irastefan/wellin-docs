# Backend архитектура — foodieai

## Обзор

NestJS 10 + Prisma 5.22 + PostgreSQL. Деплой на Google Cloud Run.  
Swagger документация: `/docs` (в production: `https://foodieai-59215576464.me-west1.run.app/docs`)

---

## Модульная структура `src/`

| Модуль | Путь | Роль |
|--------|------|------|
| `AgentModule` | `agent/` | AI-агент чат (OpenAI Responses API + tool orchestration) |
| `AiModule` | `ai/` | Прокси OpenAI Responses API + загрузка изображений в GCS |
| `AiAccessModule` | `ai-access/` | Квоты и подписки AI (per-user, per-plan) |
| `AuthModule` | `auth/` | Email OTP аутентификация + JWT |
| `BillingModule` | `billing/` | Paddle (подписки, вебхуки) |
| `MealPlansModule` | `meal-plans/` | Дневной план питания (4 слота) |
| `McpModule` | `mcp/` | MCP JSON-RPC сервер (30+ инструментов) |
| `ProductsModule` | `products/` | База продуктов питания |
| `RecipesModule` | `recipes/` | Рецепты с ингредиентами |
| `SelfCareRoutinesModule` | `self-care-routines/` | Self-care рутины по дням недели |
| `ShoppingListModule` | `shopping-list/` | Список покупок |
| `UsersModule` | `users/` | Профиль, метрики тела, TDEE |
| `PrismaModule` | `common/prisma/` | Общий Prisma Client |
| `HealthController` | `health/` | GET /health |

---

## Модуль агента (`agent/`)

Самый сложный модуль. Реализует AI-агент с tool use.

### Сервисы

| Сервис | Роль |
|--------|------|
| `AgentService` | Основная логика чата: запуск OpenAI-цикла, tool execution, structured response |
| `AgentOpenAiService` | Вызов OpenAI Responses API |
| `AgentPromptService` | Выбор prompt по mode/context |
| `AgentConversationService` | In-memory хранение истории (до 12 сообщений, MAP) |
| `AgentConfirmationService` | Создание и проверка pending confirmations |
| `AgentToolAdapterService` | Адаптация MCP-инструментов для OpenAI (sanitize имён) |
| `AgentToolPolicyService` | Разрешения: какие инструменты доступны в каком режиме |

### Режимы агента (AgentMode)

| Режим | Описание | Разрешённые инструменты |
|-------|----------|------------------------|
| `global` | Глобальный чат (весь приложение) | Read-only: user.me, product.search, recipe.search, mealPlan.dayGet/historyGet, shoppingList.get, bodyMetrics |
| `meal_plan_page` | Страница дневного плана | CRUD для meal plan: addEntry, removeEntry, copySlot + поиск |
| `meal_plan_slot` | Добавление в конкретный слот | Предложение + mealPlan.addEntry (через confirmation) |
| `shopping_list` | Список покупок | CRUD для shopping list |
| `recipe_create` | Создание рецепта | product.search, recipe.create |
| `recipe_edit` | Редактирование рецепта | product.search, recipe.search, recipe.create |
| `recipe_template` | Шаблон рецепта | product.search, recipe.search, recipe.create |
| `self_care` | Self-care рутины | Полный CRUD для selfCare |
| `food_image_analysis` | Анализ фото еды | user.me, product.search, recipe.search, mealPlan.historyGet |

### OpenAI цикл (runOpenAiLoop)
- Модель: `gpt-5-mini`
- Максимум шагов: 4
- Reasoning: `effort: low`
- История контекста: последние 8 сообщений
- Конфирмации для write-инструментов (кроме dangerous)
- Dangerous tools (remove/delete) всегда требуют конфирмации

---

## Модуль MCP (`mcp/`)

Реализует два интерфейса:
1. **JSON-RPC сервер** (для Claude через MCP): `POST /mcp`
2. **Инструменты агента** (внутренние): `AgentService` вызывает `McpService.executeTool()`

### Инструменты MCP (30 штук)

**Meta**: `mcp.capabilities`, `mcp.help`  
**Products**: `product.search`, `product.createManual`  
**Recipes**: `recipe.search`, `recipe.get`, `recipe.create`  
**Users**: `user.me`, `userProfile.upsert`, `userTargets.recalculate`  
**Body Metrics**: `bodyMetrics.dayGet`, `bodyMetrics.upsertDaily`, `bodyMetrics.historyGet`  
**Meal Plans**: `mealPlan.dayGet`, `mealPlan.historyGet`, `mealPlan.statsGet`, `mealPlan.addEntry`, `mealPlan.removeEntry`, `mealPlan.copySlot`  
**Self Care**: `selfCare.weekGet`, `selfCare.slotCreate`, `selfCare.slotUpdate`, `selfCare.slotRemove`, `selfCare.itemCreate`, `selfCare.itemUpdate`, `selfCare.itemRemove`  
**Shopping**: `shoppingList.get`, `shoppingList.addCategory`, `shoppingList.addItem`, `shoppingList.setItemState`, `shoppingList.removeItem`

### Архитектурная особенность
`McpService` — общая точка входа для всей бизнес-логики приложения. Агент НЕ вызывает сервисы напрямую — только через `McpService.executeTool()`. Это даёт единую валидацию и единый контроль авторизации.

---

## Модуль AI Access (`ai-access/`)

Управляет AI-квотами и подписками.

### Константы (defaults)
| Параметр | Free | Pro/Active | Trial |
|----------|------|------------|-------|
| Токены/месяц | 100 000 | 1 000 000 | 300 000 |
| AI actions/месяц | 100 | 1 000 | — |
| Цена | 0 | $9.99 | — |
| Trial дней | — | — | 14 |

### Фичи AI
- `RECIPE_GENERATION` — генерация рецептов
- `MEAL_ANALYSIS` — анализ питания  
- `SMART_PRODUCT_MATCH` — умный поиск продуктов
- `ADVANCED_AI_TOOLS` — агент (основной AI-чат)

---

## Модуль Billing (`billing/`)

### Paddle интеграция
- `POST /v1/billing/checkout-session` — создаёт checkout session (Paddle transactions)
- `POST /v1/billing/portal-session` — управление подпиской в Paddle portal
- `POST /v1/billing/webhooks/paddle` — вебхуки от Paddle

### Обрабатываемые события вебхуков
- `transaction.completed`, `transaction.paid` → syncFromTransaction
- `subscription.created/trialing/activated/updated/past_due/paused/resumed/canceled` → syncFromSubscription

### Маппинг статусов Paddle → Internal
| Paddle | Internal |
|--------|----------|
| trialing | TRIAL |
| active | ACTIVE |
| past_due | PAST_DUE |
| canceled/paused | CANCELED |
| остальные | FREE |

### isPro = true
Когда subscriptionStatus = ACTIVE или TRIAL

---

## Модуль Auth (`auth/`)

Email OTP аутентификация:
1. `POST /v1/auth/login/request-code` — отправляет OTP на email (через Resend Cloud Run)
2. `POST /v1/auth/login` — принимает OTP → возвращает JWT Bearer token

JWT хранится в localStorage на frontend.

---

## Общие компоненты (`common/`)

| Компонент | Файл | Роль |
|-----------|------|------|
| `PrismaService` | `common/prisma/` | Singleton Prisma Client |
| `GlobalHttpExceptionFilter` | `common/errors/` | Единый обработчик ошибок |
| `HttpLoggingInterceptor` | `common/logging/` | Логирование всех HTTP запросов |

---

## CORS

Разрешённые origins (из `main.ts`):
- `http://localhost:3000`
- `http://localhost:5173`
- `https://irastefan.github.io`
- `https://smart-food-plan-web.vercel.app`
- `https://wellin.io`
- `https://www.wellin.io`

---

## Запуск

```bash
cd foodieai
npm install
cp .env.example .env   # заполнить переменные
npm run start:dev      # dev с hot-reload
npm run build          # сборка в dist/
npm run start:prod     # production
```

### База данных
```bash
npm run prisma:migrate:dev    # применить миграции (dev)
npm run prisma:migrate:prod   # применить в production
npm run prisma:generate       # сгенерировать Prisma Client
```

### Тесты
```bash
npm test   # ts-node запускает файлы из папки test/
```

### Docker/Cloud Run
`CMD`: `prisma migrate deploy && node dist/main.js`

---

## Переменные окружения (`.env.example`)

```
DATABASE_URL
PORT=3000
CORS_ORIGINS
JWT_SECRET, AUTH_CODE_SALT
AUTH_EMAIL_*           # OTP email конфиг
OPENAI_API_KEY, OPENAI_BASE_URL, OPENAI_DEFAULT_TOOLS
AI_*_PLAN_*            # Квоты AI-планов
GCS_UPLOAD_*           # Google Cloud Storage
PADDLE_*               # Paddle billing
MCP_API_KEY
BODY_LIMIT             # Размер тела запроса (по умолчанию 20mb)
```
