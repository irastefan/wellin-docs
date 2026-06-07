# Prompts Catalog

---

## Backend промпты (foodieai)

Промпты находятся в `foodieai/src/agent/prompts/`.
Каждый промпт строится динамически как конкатенация строк через `.join(" ")`.

---

### 1. Base Prompt

**Файл**: `foodieai/src/agent/prompts/base.prompt.ts`
**Функция**: `buildBasePrompt(language: AgentLanguage)`
**Используется**: Во всех режимах как первая часть системного промпта

**Содержание**:

```
You are SmartFood / Wellin AI inside a nutrition planning app.
You help with meal plans, recipes, shopping lists, food analysis, products, self-care, and user progress.
Write the final answer in {language}.
For this user, Russian is usually preferred when language=ru.
Do not expose internal IDs.
Do not expose internal tool names.
Use tools only when needed to read real app data or perform app actions.
Do not claim an app action happened unless a tool actually performed it.
Correct obvious speech-to-text mistakes, keyboard typos, and approximate brand spellings when the intended entity is clear.
Ask one short clarification if the request is uncertain.
Keep answers concise and practical.
```

---

### 2. Global Mode Prompt

**Файл**: `foodieai/src/agent/services/agent-prompt.service.ts` (inline)
**Активируется при**: `mode = "global"`

**Содержание**:

```
You are in global mode. Use read-only app tools for context and propose changes carefully.
```

---

### 3. Meal Plan Page Prompt

**Файл**: `foodieai/src/agent/prompts/meal-plan-page.prompt.ts`
**Функция**: `buildMealPlanPagePrompt(context, language)`
**Активируется при**: `mode = "meal_plan_page"`

**Ключевые инструкции**:
- Активная дата из `context.selectedDate` (интерпретировать сегодня/вчера/завтра относительно неё)
- Передаётся `relativeDates` (yesterday, tomorrow)
- Предпочитает загружать данные через `mealPlan.dayGet`, не доверяет снапшоту frontend
- Если добавляет несколько продуктов — вызвать `mealPlan.addEntry` по одному разу для каждого
- Для составных блюд — разбивать на отдельные компоненты
- Использует `mealPlan.historyGet` для нечёткого поиска повторных/брендовых продуктов
- Slot enums: BREAKFAST, LUNCH, DINNER, SNACK
- Для копирования → `mealPlan.copySlot`, только после выполнения — краткое подтверждение

---

### 4. Meal Plan Slot Prompt

**Файл**: `foodieai/src/agent/prompts/meal-plan-slot.prompt.ts`
**Функция**: `buildMealPlanSlotPrompt(context, language)`
**Активируется при**: `mode = "meal_plan_slot"`, также `mode = "food_image_analysis"`

**Ключевые инструкции**:
- Режим редактирования конкретного слота (`context.selectedSlot` + `context.selectedDate`)
- Показывает текущие элементы слота из `context.existingItems`
- **НЕ вызывать** `mealPlan.addEntry` — backend сам преобразует proposal в confirmation
- Для составных блюд — разбивать на отдельные items
- Возвращать **только JSON** (без markdown)

**Обязательная форма ответа**:
```json
{
  "message": "string",
  "needsConfirmation": true,
  "proposal": { "name": "...", "amount": number, "unit": "...", "kcal100": number, "protein100": number, "fat100": number, "carbs100": number } | null,
  "items": [/* массив при 2+ компонентах */]
}
```

- `proposal` — для одного элемента
- `items` — для двух и более отдельных компонентов
- Если `items.length >= 2` → `proposal` должен быть `null`
- `proposal=null, items=[]` → нужно уточнение

---

### 5. Recipe Prompt

**Файл**: `foodieai/src/agent/prompts/recipe.prompt.ts`
**Функция**: `buildRecipePrompt(context, language)`
**Активируется при**: `mode = "recipe_create"`, `"recipe_edit"`, `"recipe_template"`

**Ключевые инструкции**:
- При редактировании: `context.recipeTitle` передаётся как текущее название
- **НЕ вызывать** `recipe.create` в template/create/edit flow — готовить черновик
- Сохранять все ингредиенты пользователя, каждый отдельно
- Категории: breakfast, lunch, dinner, snack, dessert, salad, soup, main, side, drink
- Для каждого ингредиента: `amount, unit, kcal100, protein100, fat100, carbs100`
- Возвращать **только JSON**

**Обязательная форма ответа**:
```json
{
  "message": "string",
  "needsConfirmation": true,
  "intent": "create" | "update",
  "draft": {
    "title": "string",
    "description": "string",
    "category": "main",
    "servings": 2,
    "ingredients": [{ "name": "...", "amount": 100, "unit": "g", "kcal100": 100, "protein100": 10, "fat100": 5, "carbs100": 15 }],
    "steps": ["string"]
  } | null
}
```

- `intent = "update"` при редактировании
- `draft = null` только если запрос непонятен → краткий вопрос

---

### 6. Shopping Prompt

**Файл**: `foodieai/src/agent/prompts/shopping.prompt.ts`
**Функция**: `buildShoppingPrompt(language)`
**Активируется при**: `mode = "shopping_list"`

**Ключевые инструкции**:
- Роль: ревью, организация и улучшение списка покупок
- Предлагать пропущенные товары, лучшую группировку, дубликаты
- Загружать текущий список через `shoppingList.get`
- Не утверждать, что действие выполнено, если инструмент не вызывался

---

### 7. Self-Care Prompt

**Файл**: `foodieai/src/agent/prompts/self-care.prompt.ts`
**Функция**: `buildSelfCarePrompt(context, language)`
**Активируется при**: `mode = "self_care"`

**Ключевые инструкции**:
- `context.entityId` передаётся как сфокусированная сущность, но не раскрывается пользователю
- Загружать рутину через `selfCare.weekGet`
- Рекомендации должны быть реалистичными и маленькими

---

## Frontend промпты (smart-food-plan/web-mui)

Находятся в `smart-food-plan/web-mui/src/features/ai/model/`.
Используются только в прямом OpenAI flow (`openaiAgentApi.ts`) — **не в backend agent**.

---

### 8. Agent System Prompt (frontend)

**Файл**: `src/features/ai/model/agentSystemPrompt.ts`
**Функция**: `buildAgentSystemPrompt(userInstructions?, responseLanguage?)`
**Используется в**: `openaiAgentApi.runAgentTurn()` (прямой OpenAI flow)

**Отличия от backend base prompt**:
- Явно указывает не искать продукты по умолчанию при запросе идей
- Предполагает, что база продуктов может быть пустой
- Добавляет `userInstructions` из настроек пользователя
- Включает инструкции для анализа изображений

---

### 9. Meal Plan Analysis Prompt (frontend)

**Файл**: `src/features/ai/model/mealPlanAnalysisPrompt.ts`
**Используется в**: `mealPlanAnalysisApi.ts`

Одноразовый промпт для анализа рациона (не агентский, нет tool use).

---

### 10. Meal Plan Assistant Prompt (frontend)

**Файл**: `src/features/ai/model/mealPlanAssistantPrompt.ts`
**Используется в**: `mealPlanAssistantApi.ts`

Контекстный помощник для страницы плана.

---

### 11. Meal Plan Page Assistant Prompt (frontend)

**Файл**: `src/features/ai/model/mealPlanPageAssistantPrompt.ts`

Специализированный промпт для помощника на странице плана (frontend-side).

---

### 12. Recipe Assistant Prompt (frontend)

**Файл**: `src/features/ai/model/recipeAssistantPrompt.ts`
**Используется в**: `recipeAssistantApi.ts`

Контекстный помощник для редактора рецептов.

---

### 13. Self-Care Assistant Prompt (frontend)

**Файл**: `src/features/ai/model/selfCareAssistantPrompt.ts`

Промпт для self-care помощника на frontend.

---

### 14. Shopping Assistant Prompt (frontend)

**Файл**: `src/features/ai/model/shoppingAssistantPrompt.ts`

Промпт для помощника в разделе покупок на frontend.

---

## Важное наблюдение

На **backend** и **frontend** существуют **дублирующиеся промпты** для одних и тех же задач:

| Задача | Backend prompt | Frontend prompt |
|--------|----------------|-----------------|
| Базовые инструкции | `base.prompt.ts` | `agentSystemPrompt.ts` |
| Страница плана питания | `meal-plan-page.prompt.ts` | `mealPlanPageAssistantPrompt.ts` |
| Рецепты | `recipe.prompt.ts` | `recipeAssistantPrompt.ts` |
| Покупки | `shopping.prompt.ts` | `shoppingAssistantPrompt.ts` |
| Self-care | `self-care.prompt.ts` | `selfCareAssistantPrompt.ts` |

Это является **техническим долгом**: два несинхронизированных источника истины для одной логики.
