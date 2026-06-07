# Current Agent Architecture

## Точка входа

```
POST /v1/agent/chat
Authorization: Bearer <JWT>
Content-Type: application/json
```

Тело запроса (`AgentChatDto`):

```typescript
{
  message: string;               // текст пользователя
  conversationId?: string;       // ID существующего диалога (опционально)
  confirmationId?: string;       // ID подтверждения (если пользователь подтверждает действие)
  language: "en" | "ru" | "he"; // язык ответа
  context: {
    mode: AgentMode;             // контекстный режим (см. ниже)
    page?: string;
    selectedDate?: string;       // выбранная дата (для meal plan)
    selectedSlot?: "BREAKFAST" | "LUNCH" | "DINNER" | "SNACK";
    sectionTitle?: string;
    existingItems?: string[];    // текущие элементы в слоте
    entityId?: string;
    recipeId?: string;
    recipeTitle?: string;
    source?: string;
  };
  images?: Array<{ name: string; imageUrl: string }>;
}
```

---

## Режимы агента (AgentMode)

| Режим | Описание | Prompt |
|-------|----------|--------|
| `global` | Глобальный чат | `buildBasePrompt` + read-only инструкция |
| `meal_plan_page` | Страница плана питания | `buildMealPlanPagePrompt` |
| `meal_plan_slot` | Редактирование слота | `buildMealPlanSlotPrompt` → JSON-ответ |
| `shopping_list` | Список покупок | `buildShoppingPrompt` |
| `recipe_create` | Создание рецепта | `buildRecipePrompt(isEdit=false)` → JSON-ответ |
| `recipe_edit` | Редактирование рецепта | `buildRecipePrompt(isEdit=true)` → JSON-ответ |
| `recipe_template` | Шаблон рецепта | `buildRecipePrompt` → JSON-ответ |
| `self_care` | Уход и рутины | `buildSelfCarePrompt` |
| `food_image_analysis` | Анализ фото еды | `buildMealPlanSlotPrompt` (alias) → JSON-ответ |

---

## Жизненный цикл запроса

### Основной flow (новое сообщение):

```
1. AgentController.chat()
   └── Извлечь userId из JWT

2. AgentService.chat()
   ├── getOrCreate(conversationId, userId)  → StoredAgentConversation
   ├── AiUsageService.ensureCanExecute()    → проверка квоты
   ├── AgentToolAdapterService.adaptTools() → фильтр инструментов по mode
   ├── buildInitialInput()                  → системный prompt + история (last 8) + сообщение
   ├── AgentConversationService.appendMessages([user_message])
   └── runOpenAiLoop(model, input, tools)

3. runOpenAiLoop (до 4 итераций):
   ├── AgentOpenAiService.createResponse()   → OpenAI Responses API
   ├── Разбор ответа:
   │    ├── Если нет function_call → выход из цикла (финальный ответ)
   │    └── Если есть function_call → обработка каждого:
   │         ├── AgentToolPolicyService.isAllowed(mode, tool) → пропустить если запрещён
   │         ├── AgentToolPolicyService.requiresConfirmation(tool)
   │         │    ├── true → добавить в pendingActions, НЕ выполнять
   │         │    └── false → McpService.executeTool(name, args, context)
   │         └── Если есть pendingActions → создать PendingConfirmation и вернуть
   └── Если outputs.length > 0 → следующая итерация с outputs как input

4. applyStructuredResponse()
   ├── Если meal_plan_slot / food_image_analysis:
   │    → распарсить JSON-ответ { message, proposal, items }
   │    → создать PendingConfirmation для mealPlan.addEntry
   └── Если recipe_create / recipe_edit / recipe_template:
        → распарсить JSON-ответ { message, intent, draft }
        → вернуть draft в data

5. AiUsageService.recordUsage()  → записать токены в AiUsageLog

6. Вернуть AgentChatResponse:
   {
     conversationId,
     answer,        // финальный текст
     messages,      // tool messages + assistant message
     toolCalls,     // что выполнялось
     requiresConfirmation,
     confirmation,  // если нужно подтверждение
     data,          // структурированные данные (draft, proposal)
     shouldRefreshData
   }
```

### Flow подтверждения (confirmationId передан):

```
1. AgentService.chat() обнаруживает confirmationId
2. Уходит в executeConfirmedActions():
   ├── ensureConfirmation(pendingConfirmation, confirmationId) → проверить UUID
   ├── Для каждого action в confirmation.actions:
   │    └── McpService.executeTool(action.tool, action.arguments, ctx)
   ├── conversations.setPendingConfirmation(null)
   └── вернуть результат (без новой квоты)
```

---

## Хранение состояния диалога

```typescript
// AgentConversationService — in-memory Map
Map<conversationId, StoredAgentConversation>

StoredAgentConversation {
  id: string;
  userId: string;
  messages: AgentConversationMessage[];   // max 12, в OpenAI передаём last 8
  pendingConfirmation: PendingConfirmation | null;
  createdAt: Date;
  updatedAt: Date;
}
```

**Важно**: хранится только в памяти процесса. При рестарте сервера все диалоги теряются. Нет TTL-очистки.

---

## Цепочка вызовов MCP-инструментов

```
AgentService.executeTool(toolName, args, requestContext)
    ↓
McpService.executeTool(name, rawArgs, context)
    ├── Найти инструмент в toolRegistry
    ├── Проверить аутентификацию (tool.auth === "required")
    ├── validateInput() → coerceValue() (приведение типов + валидация JSON Schema)
    ├── validateDtoIfPresent() → class-validator DTO
    ├── normalizeArgs() → нормализация единиц (г→g, мл→ml, etc.)
    └── tool.handler(args, context)
              ↓
        Domain Service (MealPlansService, RecipesService, etc.)
              ↓
        Prisma → PostgreSQL
```

---

## Политика инструментов (AgentToolPolicyService)

### Три категории инструментов:

| Категория | Поведение | Примеры |
|-----------|-----------|---------|
| **READ_ONLY** | Выполняется сразу, без подтверждения | `user.me`, `mealPlan.dayGet`, `recipe.search` |
| **WRITE** | Требует подтверждения пользователя | `mealPlan.addEntry`, `recipe.create`, `shoppingList.addItem` |
| **DANGEROUS** | Требует подтверждения, триггерит refresh данных | `mealPlan.removeEntry`, `shoppingList.removeItem`, `selfCare.slotRemove` |

### Доступные инструменты по режиму:

| Режим | Доступные группы |
|-------|-----------------|
| `global` | Только read-only: me, search, dayGet, historyGet |
| `meal_plan_page` | Read + write для meal plan (add, copy, remove) |
| `meal_plan_slot` | Read + add entry (без remove) |
| `shopping_list` | Все shopping инструменты |
| `recipe_create/edit` | product.search + recipe.create |
| `self_care` | Все selfCare инструменты |
| `food_image_analysis` | Только read (поиск продуктов/рецептов/истории) |

---

## Построение системного промпта

```
buildPrompt(context, language)
    = buildBasePrompt(language)          // базовые инструкции (8 правил)
    + "\n\n"
    + buildModePrompt(context, language) // контекстный промпт для режима
```

Каждый режим добавляет специализированные инструкции. В режимах `meal_plan_slot`, `recipe_*`, `food_image_analysis` агент обязан возвращать **JSON**, а не свободный текст.

---

## Структурированные JSON-ответы

### meal_plan_slot:

```json
{
  "message": "string",
  "needsConfirmation": true,
  "proposal": { "name": "...", "amount": 100, "unit": "g", "kcal100": 0, "protein100": 0, "fat100": 0, "carbs100": 0 },
  "items": [/* array — для нескольких компонентов */]
}
```

Backend автоматически создаёт из `proposal`/`items` → `PendingConfirmation` для `mealPlan.addEntry`.

### recipe_*:

```json
{
  "message": "string",
  "intent": "create" | "update",
  "draft": {
    "title": "string",
    "description": "string",
    "category": "main",
    "servings": 2,
    "ingredients": [{ "name": "...", "amount": 100, "unit": "g", "kcal100": 0, ... }],
    "steps": ["string"]
  }
}
```

Draft передаётся на frontend в `response.data.recipeDraft` — пользователь может отредактировать перед сохранением.

---

## Обработка ошибок инструментов

- **Ошибки HTTP** (NestJS exceptions) → извлекается message из `getResponse()`
- **Прочие ошибки** → `error.message` или "Tool execution failed"
- Ошибка инструмента **не прерывает** весь запрос, возвращается `toolCall.status = "error"`

---

## Модель OpenAI

Используется: **`gpt-5-mini`** (захардкожено в `agent.service.ts:44`)

Параметры запроса:
```typescript
{
  model: "gpt-5-mini",
  input: [...messages],
  previous_response_id: string | undefined,  // для продолжения
  tools: [...openAiTools],
  reasoning: { effort: "low" }
}
```

Использует **OpenAI Responses API** (не Chat Completions), что позволяет передавать `previous_response_id` для stateful продолжения в рамках одного цикла.
