# AI-агент — текущая реализация

## Обзор

AI-агент — единый чат-бот, встроенный во все разделы приложения.  
Основан на **LangGraph** с двумя путями вызова LLM:
- **openai** (по умолчанию): OpenAI Responses API (`AGENT_MODEL`, по умолчанию `gpt-4o-mini`)
- **openai_compat**: Chat Completions API — для LM Studio, Ollama и любых OpenAI-совместимых провайдеров

---

## Расположение кода

### Backend
```
foodieai/src/agent/
  agent.controller.ts              — POST /v1/agent/chat, POST /v1/agent/chat/stream
  agent.service.ts                 — Основная логика: processWithConversation, runOpenAiLoop, runChatCompletionsLoop
  agent.module.ts
  dto/
    agent-chat.dto.ts              — Входные данные
    agent-chat-response.dto.ts     — Ответ
  i18n/
    agent-messages.ts              — Все строки EN/RU/HE (централизованный i18n)
  langgraph/
    langgraph-agent.service.ts     — LangGraph граф: router + 5 специализированных узлов
    langgraph-state.ts             — Тип состояния графа
    langgraph-checkpointer.ts      — PostgreSQL чекпоинтер (персистентная история)
  prompts/
    base.prompt.ts                 — Базовый prompt
    meal-plan-page.prompt.ts
    meal-plan-slot.prompt.ts
    recipe.prompt.ts
    router.prompt.ts               — Промпт для LLM-классификатора роутера
    self-care.prompt.ts
    shopping.prompt.ts
  services/
    agent-chat-completions.service.ts  — Chat Completions API (LM Studio/Ollama)
    agent-confirmation.service.ts      — Управление confirmations
    agent-conversation.service.ts      — In-memory история (legacy, для прямого AgentService.chat())
    agent-openai.service.ts            — OpenAI Responses API
    agent-prompt.service.ts            — Выбор промпта по mode
    agent-tool-adapter.service.ts      — Адаптация MCP tools → OpenAI формат
    agent-tool-policy.service.ts       — Политика доступа к инструментам
  types/
    agent-context.types.ts         — TypeScript типы (StoredAgentConversation, SseAgentEvent, ...)
```

### Frontend
```
smart-food-plan/web-mui/src/features/ai/
  api/
    backendAgentApi.ts         — Основной API (/v1/agent/chat + /v1/agent/chat/stream)
    aiUsageApi.ts              — Upload image + usage stats

smart-food-plan/web-mui/src/widgets/ai/
  AgentWorkspace.tsx
  AiAssistantPanel.tsx
  AiAgentConversation.tsx
  AiAgentComposer.tsx
  AgentConfirmationCard.tsx    — Карточка подтверждения (entity-aware кнопки)
  GlobalAiAgentDialog.tsx
  ContextAgentDialog.tsx
  PageAssistantDialogShell.tsx
  AiUsageSummary.tsx
```

---

## Как работает агент (LangGraph flow)

```
Frontend
  ↓ POST /v1/agent/chat/stream (SSE) или /v1/agent/chat
LangGraphAgentService
  ├── Создаёт/восстанавливает thread из PostgreSQL (по conversationId)
  ├── Запускает граф: START → RouterNode
  └── RouterNode → специализированный узел (meal_plan / recipe / shopping / self_care / general)
        ↓
   AgentService.processWithConversation(conversation, userId, headers, dto, streamEmit?)
        ├── aiUsageService.ensureCanExecute (проверка квоты)
        ├── toolAdapter.adaptTools (MCP tools → OpenAI формат)
        ├── LLM Provider выбор:
        │    ├── openai → runOpenAiLoop (Responses API, stateful через previous_response_id)
        │    └── openai_compat → runChatCompletionsLoop (Chat Completions, accumulate messages)
        ├── Для каждого tool call:
        │    ├── toolPolicy.isAllowed(mode, toolName) ?
        │    ├── toolPolicy.requiresConfirmation(toolName) ?
        │    │    → true: создаём pendingConfirmation, прерываем (SSE: done с requiresConfirmation)
        │    └── mcpService.executeTool(toolName, args, {userId, headers, requestId})
        ├── SSE события: tool_call → tool_result → text_delta → done
        └── recordUsage(userId, ...) → обновляем токены
```

---

## Граф LangGraph

```
START → RouterNode → recipe / meal_plan / shopping / self_care / agent → END
```

| Узел | AgentMode | Особенности |
|------|-----------|-------------|
| `router` | — | Детерминированный маппинг mode → route; для global — LLM-классификатор |
| `recipe` | recipe_create/edit/template | Structured Outputs включены автоматически |
| `meal_plan` | meal_plan_page | Принудительно ставит mode = meal_plan_page + selectedDate |
| `shopping` | shopping_list | — |
| `self_care` | self_care | — |
| `agent` | global + другие | Общий узел |

---

## SSE-стриминг

`POST /v1/agent/chat/stream` — Server-Sent Events.

Формат событий:
```json
{ "type": "agent_start" }
{ "type": "tool_call", "tool": "mealPlan.dayGet", "action": "load", "entity": "day" }
{ "type": "tool_result", "tool": "mealPlan.dayGet", "status": "success" }
{ "type": "text_delta", "delta": "Я нашла варианты..." }
{ "type": "done", "conversationId": "...", "answer": "...", "requiresConfirmation": false, ... }
{ "type": "error", "message": "...", "code": "..." }
```

Frontend читает через `fetch` + `ReadableStream` (не `EventSource`, нужен POST с body).

---

## Multi-LLM поддержка

Управляется переменными окружения:

| Переменная | По умолчанию | Описание |
|-----------|-------------|----------|
| `AGENT_MODEL` | `gpt-4o-mini` | Имя модели для LLM-вызовов |
| `LLM_PROVIDER` | `openai` | `openai` = Responses API; `openai_compat` = Chat Completions |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | Базовый URL API |

**LM Studio (локально)**:
```
OPENAI_BASE_URL=http://localhost:1234/v1
LLM_PROVIDER=openai_compat
AGENT_MODEL=<название модели в LM Studio>
```

---

## Управление историей

| Режим | Хранилище | Персистентность |
|-------|-----------|----------------|
| LangGraph (основной) | PostgreSQL через `@langchain/langgraph-checkpoint-postgres` | ✅ Постоянная (переживает рестарт сервера) |
| Прямой `AgentService.chat()` (тесты) | In-memory Map | ❌ Теряется при рестарте |

Максимум 12 сообщений в истории на стороне AgentService.

---

## Система конфирмаций

Все write-инструменты требуют подтверждения пользователя.

**Read-only** (без конфирмации):  
`mcp.capabilities`, `mcp.help`, `user.me`, `product.search`, `recipe.search`, `recipe.get`, `mealPlan.dayGet`, `mealPlan.historyGet`, `shoppingList.get`, `bodyMetrics.dayGet`, `bodyMetrics.historyGet`, `selfCare.weekGet`

**Write** (требует конфирмации):  
`recipe.create`, `mealPlan.addEntry`, `mealPlan.copySlot`, `shoppingList.addItem`, `shoppingList.addCategory`, `shoppingList.setItemState`, `userProfile.upsert`, `bodyMetrics.upsertDaily`, `selfCare.slotCreate/Update/ItemCreate/ItemUpdate`

**Dangerous** (всегда требует конфирмации):  
`mealPlan.removeEntry`, `shoppingList.removeItem`, `selfCare.slotRemove`, `selfCare.itemRemove`

### Кнопка подтверждения (entity-aware)
`AgentConfirmationCard` показывает label кнопки в зависимости от инструмента и варианта:
- `mealPlan.*` → "Add to Plan" / "Remove from Plan"
- `shoppingList.*` → "Add to List" / "Remove from List"
- `recipe.*` → "Create Recipe" / "Update Recipe"
- `selfCare.*` / `routine.*` → "Add Routine" / "Update Routine"
- Danger → "Confirm"

---

## LangSmith трассировка

Автоматически включается через переменные окружения:
```
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=<ключ>
LANGSMITH_PROJECT=wellin-agent
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
```

При старте сервера в логах появится: `LangSmith tracing enabled — project: "wellin-agent"`.

---

## Режимы агента (AgentMode)

| Режим | Кто запускает | Инструменты |
|-------|--------------|-------------|
| `global` | GlobalAiAgentDialog | Read-only набор |
| `meal_plan_page` | MealPlanAssistantDialog | CRUD для meal plan + shoppingList частично |
| `meal_plan_slot` | MealPlanItemDialog | Предложение + addEntry через confirmation |
| `shopping_list` | ShoppingAssistantDialog | CRUD для shopping |
| `recipe_create` | RecipeAssistantDialog (новый) | recipe.create + поиск |
| `recipe_edit` | RecipeAssistantDialog (редактирование) | recipe.create + поиск |
| `recipe_template` | Шаблон рецепта | recipe.create + поиск |
| `self_care` | SelfCareAssistantDialog | selfCare CRUD |
| `food_image_analysis` | Анализ фото | Read-only + historyGet |

---

## Что работает хорошо

- Единая точка входа для всех инструментов через `McpService`
- Чёткое разделение режимов и политик доступа
- Система конфирмаций для защиты от случайных write-операций
- Персистентная история через LangGraph + PostgreSQL
- SSE-стриминг — пользователь видит прогресс в реальном времени
- Multi-LLM: переключение между OpenAI и любым совместимым провайдером через env
- Многоязычность (EN/RU/HE) в централизованном `agent-messages.ts`
- LangSmith трассировка для отладки
