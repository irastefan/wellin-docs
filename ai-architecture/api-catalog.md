# API Catalog — AI-related Endpoints

## Backend (foodieai)

### Agent API

#### `POST /v1/agent/chat`

Основной endpoint AI-агента.

| | |
|---|---|
| **Auth** | JWT Bearer (обязателен) |
| **Quota** | Проверяет и записывает AI-использование |
| **Controller** | `AgentController.chat()` |
| **Service** | `AgentService.chat()` |

**Request body**:
```typescript
{
  message: string;
  conversationId?: string;          // ID существующего диалога
  confirmationId?: string;          // UUID для подтверждения pending action
  language: "en" | "ru" | "he";
  context: {
    mode: AgentMode;                // см. список режимов
    page?: string;
    selectedDate?: string;          // "YYYY-MM-DD"
    selectedSlot?: "BREAKFAST" | "LUNCH" | "DINNER" | "SNACK";
    sectionTitle?: string;
    existingItems?: string[];
    entityId?: string;
    recipeId?: string;
    recipeTitle?: string;
    source?: string;
  };
  images?: Array<{ name: string; imageUrl: string }>;
}
```

**Response**:
```typescript
{
  conversationId: string;
  answer: string;
  messages: Array<{
    id: string;
    role: "assistant" | "tool";
    text: string;
    toolName?: string;
    toolAction?: string;
    toolEntity?: string | null;
  }>;
  toolCalls: Array<{
    tool: string;
    status: "success" | "error" | "skipped" | "requires_confirmation";
    action?: string;
    entity?: string | null;
    error?: string;
  }>;
  requiresConfirmation: boolean;
  confirmation: null | {
    id: string;
    title: string;
    summary: string;
    variant: "info" | "create" | "add" | "update" | "delete" | "danger";
    actions: Array<{ tool: string; arguments: Record<string, unknown>; label?: string }>;
  };
  data: {
    proposal?: Record<string, unknown> | null;  // meal_plan_slot
    items?: Record<string, unknown>[];           // meal_plan_slot (множество)
    recipeDraft?: Record<string, unknown> | null; // recipe_*
    [key: string]: unknown;
  };
  shouldRefreshData: boolean;
}
```

**Сценарии**:
1. Новое сообщение → OpenAI loop → ответ
2. Новое сообщение с `confirmationId` → выполнить pending actions → ответ
3. Режим meal_plan_slot → JSON ответ → auto confirmation для mealPlan.addEntry
4. Режим recipe_* → JSON ответ → data.recipeDraft

---

### AI Proxy API

#### `POST /v1/ai/responses`

Прокси к OpenAI Responses API. Используется в прямом frontend-агенте.

| | |
|---|---|
| **Auth** | JWT Bearer (обязателен) |
| **Quota** | Проверяет feature доступ, записывает AiUsageLog |
| **Controller** | `AiController` |
| **Service** | `AiService` |

**Request body**: OpenAI Responses API payload (прозрачная проксификация)

**Response**: OpenAI Responses API response

Отличие от прямого вызова OpenAI — через этот endpoint:
- Проверяется AI-квота пользователя (`AiUsageService.ensureCanExecute`)
- Записывается использование токенов

#### `POST /v1/ai/upload-image`

Загрузка изображений для AI-анализа в Google Cloud Storage.

| | |
|---|---|
| **Auth** | JWT Bearer (обязателен) |
| **Response** | `{ imageUrl: string }` (GCS URL) |

---

### MCP API

#### `GET /mcp`

Ping / статус MCP-сервера.

**Response**: `{ name: "FoodieAI MCP", status: "ok" }`

#### `POST /mcp`

Основной MCP JSON-RPC endpoint. Используется внешними AI (Claude Desktop, Claude API).

| | |
|---|---|
| **Auth** | Опционально (Bearer, если tool требует auth) |
| **Protocol** | JSON-RPC 2.0 |

**Поддерживаемые методы**:

| Метод | Описание |
|-------|----------|
| `initialize` | Handshake, возвращает capabilities |
| `tools/list` | Список всех зарегистрированных инструментов |
| `tools/call` | Вызов конкретного инструмента |
| `resources/list` | Список ресурсов (пустой) |
| `prompts/list` | Список промптов (пустой) |

**Пример запроса**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "recipe.search",
    "arguments": { "query": "omelette", "limit": 5 }
  }
}
```

---

### AI Access API

#### `GET /v1/ai/usage`

Получить текущее использование AI для пользователя.

| | |
|---|---|
| **Auth** | JWT Bearer |

---

## Frontend API Clients

Находятся в `smart-food-plan/web-mui/src/features/ai/api/`.

### `backendAgentApi.ts`

**Функция**: `runBackendAgentTurn(input)`

Вызывает `POST /v1/agent/chat`.

Дополнительно:
- `uploadAgentImages()` → `POST /v1/ai/upload-image` перед запросом
- `backendAgentMessagesToAgentMessages()` → нормализация ответа
- `backendToolCallsToProgressEvents()` → события для UI progress

Используется в:
- `AiAgentPage` (global mode)
- `MealPlanAssistantDialog`
- `MealPlanItemDialog`
- `RecipeAssistantDialog`
- `ShoppingAssistantDialog`
- `SelfCareAssistantDialog`

---

### `openaiAgentApi.ts`

**Функция**: `runAgentTurn(input)`

Прямой агентный цикл через `POST /v1/ai/responses` (прокси OpenAI).
Инструменты получает через `callMcpTool()` из `mcpApi.ts`.

Особенности:
- До 4 итераций цикла
- Защита от дублирующихся tool calls
- Строит промпт через `buildAgentSystemPrompt()`

Используется в некоторых виджетах (legacy path).

---

### `mcpApi.ts`

**Функция**: `callMcpTool(name, args)`

Вызывает `POST /mcp` (JSON-RPC) для выполнения инструмента.

Используется в `openaiAgentApi.ts` (прямой OpenAI flow).

---

### `aiUsageApi.ts`

- `postAiResponse()` — POST /v1/ai/responses (с quota check)
- `uploadAiImage()` — POST /v1/ai/upload-image
- Константы: `AiFeature.ADVANCED_AI_TOOLS`, etc.

---

### `mealPlanAnalysisApi.ts`

Разовый AI-вызов для анализа рациона (не агент, нет tool loop).

---

### `mealPlanAssistantApi.ts` / `recipeAssistantApi.ts`

Контекстные AI-помощники со своими промптами (frontend side).

---

## Сводная таблица endpoint'ов

| Endpoint | Метод | Auth | Назначение |
|----------|-------|------|------------|
| `/v1/agent/chat` | POST | JWT | AI-агент (основной) |
| `/v1/ai/responses` | POST | JWT | OpenAI прокси с quota |
| `/v1/ai/upload-image` | POST | JWT | Загрузка изображений |
| `/v1/ai/usage` | GET | JWT | Статус AI-квоты |
| `/mcp` | GET | — | MCP ping |
| `/mcp` | POST | Опц. | MCP JSON-RPC (инструменты) |
| `/health` | GET | — | Health check |
