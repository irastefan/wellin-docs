# Карта API

## Backend base URL

Production: `https://foodieai-59215576464.me-west1.run.app`  
Swagger: `{base_url}/docs`

---

## Аутентификация

Все защищённые маршруты требуют: `Authorization: Bearer <jwt_token>`

---

## Auth (`/v1/auth`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| POST | `/v1/auth/login/request-code` | Запросить OTP на email | Нет |
| POST | `/v1/auth/login` | Войти с OTP → JWT + user data | Нет |
| POST | `/v1/auth/register` | Зарегистрироваться | Нет |
| POST | `/v1/auth/refresh` | Переиздать JWT (авто-refresh) | Да |

---

## Users (`/v1/me`, `/v1/profile`, `/v1/body-metrics`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/me` | Текущий пользователь + профиль | Да |
| PUT | `/v1/profile` | Сохранить/обновить профиль | Да |
| GET | `/v1/body-metrics` | История замеров тела | Да |
| GET | `/v1/body-metrics/day` | Замеры за конкретный день | Да |
| PUT | `/v1/body-metrics` | Сохранить/обновить замеры | Да |

---

## Meal Plans (`/v1/meal-plans`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/meal-plans/day` | Дневной план питания | Да |
| POST | `/v1/meal-plans/day` | Создать/обновить дневной план | Да |
| GET | `/v1/meal-plans/stats` | Статистика нутриентов | Да |
| POST | `/v1/meal-plans/entries` | Добавить позицию в план | Да |
| DELETE | `/v1/meal-plans/entries/:id` | Удалить позицию | Да |
| POST | `/v1/meal-plans/copy-slot` | Скопировать слот питания | Да |

---

## Recipes (`/v1/recipes`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/recipes` | Поиск рецептов | Опционально |
| POST | `/v1/recipes` | Создать рецепт | Да |
| GET | `/v1/recipes/:id` | Получить рецепт | Опционально |
| PUT | `/v1/recipes/:id` | Обновить рецепт | Да |
| DELETE | `/v1/recipes/:id` | Удалить рецепт | Да |

---

## Products (`/v1/products`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/products` | Поиск продуктов | Опционально |
| POST | `/v1/products` | Создать продукт | Да |
| GET | `/v1/products/:id` | Получить продукт | Опционально |
| PUT | `/v1/products/:id` | Обновить продукт | Да |
| DELETE | `/v1/products/:id` | Удалить продукт | Да |

---

## Shopping List (`/v1/shopping-list`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/shopping-list` | Получить список покупок | Да |
| POST | `/v1/shopping-list/items` | Добавить позицию | Да |
| PUT | `/v1/shopping-list/items/:id` | Обновить позицию | Да |
| DELETE | `/v1/shopping-list/items/:id` | Удалить позицию | Да |
| POST | `/v1/shopping-list/categories` | Добавить категорию | Да |
| DELETE | `/v1/shopping-list/categories/:id` | Удалить категорию | Да |

---

## Self-Care Routines (`/v1/self-care-routines`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| GET | `/v1/self-care-routines` | Получить недельную рутину | Да |
| POST | `/v1/self-care-routines/slots` | Создать слот | Да |
| PUT | `/v1/self-care-routines/slots/:id` | Обновить слот | Да |
| DELETE | `/v1/self-care-routines/slots/:id` | Удалить слот | Да |
| POST | `/v1/self-care-routines/slots/:id/items` | Добавить элемент в слот | Да |
| PUT | `/v1/self-care-routines/items/:id` | Обновить элемент | Да |
| DELETE | `/v1/self-care-routines/items/:id` | Удалить элемент | Да |

---

## AI (`/v1/ai`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| POST | `/v1/ai/responses` | Прокси OpenAI Responses API | Да (AI quota) |
| POST | `/v1/ai/uploads/image` | Загрузить изображение в GCS | Да |

---

## AI Agent (`/v1/agent`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| POST | `/v1/agent/chat` | Чат с AI-агентом (tool use) | Да (AI quota) |
| POST | `/v1/agent/chat/stream` | Стриминг через SSE (LangGraph) | Да (AI quota) |

### Параметры `/v1/agent/chat`
```json
{
  "message": "string",
  "conversationId": "string?",
  "confirmationId": "string?",
  "language": "en|ru|he",
  "context": {
    "mode": "global|meal_plan_page|meal_plan_slot|shopping_list|recipe_create|recipe_edit|recipe_template|self_care|food_image_analysis",
    "page": "string?",
    "selectedDate": "YYYY-MM-DD?",
    "selectedSlot": "BREAKFAST|LUNCH|DINNER|SNACK?",
    "sectionTitle": "string?",
    "existingItems": "string[]?",
    "entityId": "string?",
    "recipeId": "string?",
    "recipeTitle": "string?",
    "source": "string?"
  },
  "images": [{ "name": "string", "imageUrl": "string" }]
}
```

---

## MCP (`/mcp`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| POST | `/mcp` | MCP JSON-RPC endpoint | MCP_API_KEY |

### Поддерживаемые MCP методы
- `tools/list` — список инструментов
- `tools/call` — вызов инструмента
- `initialize` — инициализация

---

## Billing (`/v1/billing`)

| Метод | Маршрут | Описание | Auth |
|-------|---------|----------|------|
| POST | `/v1/billing/checkout-session` | Создать Paddle checkout сессию | Да |
| POST | `/v1/billing/portal-session` | Открыть Paddle billing portal | Да |
| POST | `/v1/billing/webhooks/paddle` | Paddle вебхук (raw body) | Нет (HMAC подпись) |

---

## Служебные

| Метод | Маршрут | Описание |
|-------|---------|----------|
| GET | `/health` | Health check |
| GET | `/docs` | Swagger UI |

---

## Frontend API вызовы

### Dev-proxy (Vite)
В dev-режиме Vite проксирует `/api/*` → `VITE_API_PROXY_TARGET` (backend).  
Frontend вызывает `apiRequest("/v1/...")` и получает `{API_BASE_URL}/v1/...`.

### Ключевые frontend API файлы
| Файл | Маршруты |
|------|---------|
| `authApi.ts` | `/v1/auth/*` |
| `mealPlanApi.ts` | `/v1/meal-plans/*` |
| `recipesApi.ts` | `/v1/recipes/*` |
| `productsApi.ts` | `/v1/products/*` |
| `shoppingApi.ts` | `/v1/shopping-list/*` |
| `selfCareApi.ts` | `/v1/self-care-routines/*` |
| `settingsApi.ts` | `/v1/me`, `/v1/profile` |
| `bodyMetricsApi.ts` | `/v1/body-metrics/*` |
| `authApi.ts` | `/v1/auth/refresh` |
| `backendAgentApi.ts` | `/v1/agent/chat`, `/v1/agent/chat/stream` |
| `aiUsageApi.ts` | `/v1/ai/usage`, `/v1/ai/uploads/image` |
| `billingApi.ts` | `/v1/billing/*` |
