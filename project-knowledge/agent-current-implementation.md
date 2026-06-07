# AI-агент — текущая реализация

## Обзор

AI-агент — единый чат-бот, встроенный во все разделы приложения.  
Основан на **OpenAI Responses API** (`gpt-5-mini`) с tool calling через **MCP tools**.

---

## Расположение кода

### Backend
```
foodieai/src/agent/
  agent.controller.ts          — POST /v1/agent/chat
  agent.service.ts             — Основная логика
  agent.module.ts
  dto/
    agent-chat.dto.ts          — Входные данные
    agent-chat-response.dto.ts — Ответ
  prompts/
    base.prompt.ts             — Базовый prompt + buildBasePrompt()
    meal-plan-page.prompt.ts   — Промпт для страницы meal plan
    meal-plan-slot.prompt.ts   — Промпт для слота meal plan
    recipe.prompt.ts           — Промпт для рецептов
    self-care.prompt.ts        — Промпт для self-care
    shopping.prompt.ts         — Промпт для shopping
  services/
    agent-conversation.service.ts  — In-memory история
    agent-openai.service.ts        — Вызов OpenAI API
    agent-prompt.service.ts        — Выбор промпта по mode
    agent-tool-adapter.service.ts  — Адаптация MCP tools
    agent-tool-policy.service.ts   — Политика доступа к инструментам
    agent-confirmation.service.ts  — Управление confirmations
  types/
    agent-context.types.ts     — TypeScript типы
```

### Frontend
```
smart-food-plan/web-mui/src/features/ai/
  api/
    backendAgentApi.ts         — Основной API (POST /v1/agent/chat)
    openaiAgentApi.ts          — Прямой OpenAI (legacy/user key)
    mealPlanAssistantApi.ts    — Через /v1/ai/responses
    mealPlanAnalysisApi.ts     — Через /v1/ai/responses
    recipeAssistantApi.ts      — Через /v1/ai/responses
    mcpApi.ts                  — Прямой MCP вызов
    aiUsageApi.ts              — Upload image + usage stats
  model/
    agentSystemPrompt.ts       — Промпт для openaiAgentApi (legacy)
    mealPlanAnalysisPrompt.ts  — Промпт для mealPlanAnalysisApi
    mealPlanAssistantPrompt.ts — Промпт для mealPlanAssistantApi
    mealPlanPageAssistantPrompt.ts — Промпт для pageAssistant
    recipeAssistantPrompt.ts   — Промпт для recipeAssistantApi
    selfCareAssistantPrompt.ts — Промпт для selfCare
    shoppingAssistantPrompt.ts — Промпт для shopping

smart-food-plan/web-mui/src/widgets/ai/
  AgentWorkspace.tsx
  AiAssistantPanel.tsx
  AiAgentConversation.tsx
  AiAgentComposer.tsx
  AgentConfirmationCard.tsx
  AiAgentToolsCard.tsx
  GlobalAiAgentDialog.tsx
  ContextAgentDialog.tsx
  PageAssistantDialogShell.tsx
  AiUsageSummary.tsx
```

---

## Как работает агент (backend flow)

```
Frontend
  ↓ POST /v1/agent/chat
AgentController
  ↓ getUserId(headers) → userId
AgentService.chat()
  ├── Проверка quotas (aiUsageService.ensureCanExecute)
  ├── getOrCreate(conversationId, userId) → conversation (in-memory)
  ├── buildPrompt(context, language) → system prompt
  ├── adaptTools(mcpTools, mode) → OpenAI tools (sanitized names)
  ├── buildInitialInput(conversation, dto, prompt) → messages array
  └── runOpenAiLoop(model, input, tools)
       ├── [Step 1-4]: openAi.createResponse(...)
       ├── Если нет function_calls → возврат ответа
       ├── Для каждого tool call:
       │    ├── toolPolicy.isAllowed(mode, toolName)?
       │    ├── toolPolicy.requiresConfirmation(toolName)?
       │    │    → true: создаём pendingConfirmation, прерываем
       │    └── executeTool(toolName, args, context)
       │         → mcpService.executeTool(toolName, args, {userId, headers, requestId})
       └── Передаём output обратно в OpenAI
  ↓ applyStructuredResponse(dto, response)
  │   (для meal_plan_slot/food_image_analysis: парсим JSON proposal)
  │   (для recipe modes: парсим JSON recipeDraft)
  └── recordUsage(userId, ...) → обновляем токены
  ↓ Возвращаем AgentChatResponse
```

---

## Управление историей (AgentConversationService)

**Хранилище**: `Map<conversationId, StoredAgentConversation>` — **in-memory** (НЕ в БД).

### Ограничения истории
- Максимум 12 сообщений в истории
- В OpenAI передаются последние **8** сообщений
- История живёт только в рамках серверного процесса (при рестарте Cloud Run теряется)
- Нет персистентности между сессиями

### Структура разговора
```typescript
{
  id: string;           // UUID
  userId: string;
  messages: Array<{
    role: "user" | "assistant" | "tool";
    text: string;
    toolName?: string;
  }>;
  pendingConfirmation: PendingConfirmation | null;
  createdAt: Date;
  updatedAt: Date;
}
```

---

## Система конфирмаций

Все write-инструменты требуют подтверждения пользователя.

### Типы инструментов по уровню опасности

**Read-only** (без конфирмации):  
`mcp.capabilities`, `mcp.help`, `user.me`, `product.search`, `recipe.search`, `recipe.get`, `mealPlan.dayGet`, `mealPlan.historyGet`, `shoppingList.get`, `bodyMetrics.dayGet`, `bodyMetrics.historyGet`, `selfCare.weekGet`

**Write** (требует конфирмации):  
`product.createManual`, `recipe.create`, `mealPlan.addEntry`, `mealPlan.copySlot`, `shoppingList.addItem`, `shoppingList.addCategory`, `shoppingList.setItemState`, `userProfile.upsert`, `bodyMetrics.upsertDaily`, `selfCare.slotCreate/Update/ItemCreate/ItemUpdate`

**Dangerous** (всегда требует конфирмации):  
`mealPlan.removeEntry`, `shoppingList.removeItem`, `selfCare.slotRemove`, `selfCare.itemRemove`

### Флоу конфирмации
1. Агент предлагает действие → frontend получает `requiresConfirmation: true` + `confirmation` объект
2. Пользователь нажимает "Подтвердить" → frontend отправляет тот же запрос с `confirmationId`
3. Backend выполняет `executeConfirmedActions` → выполняет инструменты

---

## Промпты (backend)

### Базовый промпт (`base.prompt.ts`)
```
"You are SmartFood / Wellin AI inside a nutrition planning app."
"You help with meal plans, recipes, shopping lists, food analysis, products, self-care, and user progress."
"Write the final answer in {language}."
"Do not expose internal IDs."
"Do not expose internal tool names."
"Use tools only when needed..."
"Keep answers concise and practical."
```

### Промпт для meal_plan_slot
- Указывает текущий слот и дату
- Перечисляет существующие позиции слота
- Требует JSON ответ со структурой `proposal/items`
- Запрещает вызывать `mealPlan.addEntry` (backend сам создаёт confirmation)

### Промпт для meal_plan_page (в frontend)
- Передаёт snapshot текущего дня (sections, items, totals)
- Передаёт относительные даты (yesterday, tomorrow)
- Инструкции по работе со слотами и historyGet

### Остальные промпты (recipe, self-care, shopping, global)
Более простые — ориентируют агента на контекст страницы.

---

## Режимы агента и инструменты (AgentMode)

| Режим | Кто запускает | Инструменты |
|-------|--------------|-------------|
| `global` | GlobalAiAgentDialog | Read-only набор |
| `meal_plan_page` | MealPlanAssistantDialog | CRUD для meal plan |
| `meal_plan_slot` | MealPlanItemDialog | Предложение + addEntry через confirmation |
| `shopping_list` | ShoppingAssistantDialog | CRUD для shopping |
| `recipe_create` | RecipeAssistantDialog (новый рецепт) | recipe.create + поиск |
| `recipe_edit` | RecipeAssistantDialog (редактирование) | recipe.create + поиск |
| `recipe_template` | Шаблон рецепта | recipe.create + поиск |
| `self_care` | SelfCareAssistantDialog | selfCare CRUD |
| `food_image_analysis` | Анализ фото | Read-only + historyGet |

---

## Параллельная старая реализация (технический долг)

На frontend существуют **ДВА** пути вызова AI:

### Путь 1: Backend агент (основной, рекомендуемый)
```
backendAgentApi.ts → POST /v1/agent/chat
```
Использует: `GlobalAiAgentDialog`, `ContextAgentDialog`, `AgentWorkspace`

### Путь 2: Direct OpenAI (устаревший, через user key)
```
openaiAgentApi.ts → прямой вызов OpenAI API (с ключом пользователя)
mealPlanAssistantApi.ts → POST /v1/ai/responses (proxy)
mealPlanAnalysisApi.ts → POST /v1/ai/responses (proxy)
recipeAssistantApi.ts → POST /v1/ai/responses (proxy)
mcpApi.ts → POST /mcp (прямо в MCP)
```
Использует: `MealPlanAssistantDialog`, `MealPlanAnalysisDialog`, `RecipeAssistantDialog`, `SelfCareAssistantDialog`, `ShoppingAssistantDialog`

### Промпты на frontend (технический долг)
В `features/ai/model/` — 7 файлов промптов для **frontend** вызовов.  
На **backend** — ещё 5 файлов промптов.  
Итого: **12 промптов** в двух местах, разные для одних и тех же режимов.

---

## Проблемы текущего агента

1. **История не персистентна** — при рестарте сервера теряется. В Cloud Run возможны несколько инстансов.

2. **Два пути вызова AI** — frontend поддерживает и старый (direct OpenAI), и новый (backend agent). Это создаёт путаницу и дублирование промптов.

3. **Модель gpt-5-mini** — захардкожена в `agent.service.ts`. Нет возможности переключить без деплоя.

4. **Промпты в двух местах** — backend `agent/prompts/` + frontend `features/ai/model/`. Промпты расходятся.

5. **Нет ретрай логики** — если OpenAI API недоступен, просто падаем с ошибкой.

6. **Максимум 4 шага** — ограничение цикла. Сложные задачи могут не завершиться.

7. **Structured response хрупкий** — парсинг JSON из ответа AI может дать сбой. Fallback есть, но обработка несовершенна.

---

## Что работает хорошо

- Единая точка входа для всех инструментов через `McpService`
- Чёткое разделение режимов и политик доступа
- Система конфирмаций для защиты от случайных write-операций
- Автоматическая нормализация имён инструментов для OpenAI
- Аргументы обогащаются контекстом (selectedDate автоматически прокидывается)
- Многоязычность (EN/RU/HE) на уровне промптов
- Квоты AI по пользователям и функциям
