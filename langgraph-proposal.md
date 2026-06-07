# LangGraph Architecture Proposal — Wellin

## Цель

Заменить монолитный `AgentService` на граф специализированных агентов на базе LangGraph.
Результат: масштабируемая, расширяемая, наблюдаемая агентная платформа, способная выполнять сложные многошаговые задачи.

---

## Принципы новой архитектуры

1. **Специализация**: каждый агент знает свою область и делает её хорошо
2. **Оркестрация**: Router Agent принимает решение о делегировании
3. **Память**: три уровня — кратковременная, долгосрочная, семантическая
4. **Подтверждение**: явный human-in-the-loop checkpoint перед write операциями
5. **Наблюдаемость**: каждый шаг трассируется через LangSmith / OpenTelemetry
6. **Streaming**: SSE из графа на frontend в реальном времени

---

## Предлагаемые агенты

### 1. Router Agent (Planner)

**Назначение**: Входная точка. Понимает намерение пользователя и маршрутизирует к нужному агенту.

**Входные данные**:
- Сообщение пользователя
- Контекст (текущая страница, режим, дата, слот)
- Краткая история диалога
- Профиль пользователя (кэш из Long-term memory)

**Решения**:
- Какой специализированный агент активировать
- Нужен ли Critic Agent после выполнения
- Требуется ли запрос нескольких агентов параллельно (multi-agent fan-out)

**Выходные данные**: `{ agent: AgentType, plan: string, context: enrichedContext }`

**Примеры маршрутизации**:
```
"что добавить на обед?" → MealPlanAgent
"создай рецепт борща" → RecipeAgent
"не хватает белка сегодня" → NutritionAgent + MealPlanAgent
"купи продукты для рецептов на неделю" → ShoppingAgent + MealPlanAgent
"посмотри мой план на неделю" → NutritionAgent (read-only)
```

---

### 2. Recipe Agent

**Назначение**: Всё, что связано с рецептами.

**Умеет**:
- Создавать рецепт по описанию (черновик → подтверждение → сохранение)
- Редактировать существующий рецепт
- Искать рецепты по критериям (КБЖУ, категория, ингредиенты)
- Предлагать рецепт для слота плана питания
- Рассчитывать КБЖУ на порцию

**Инструменты**: `recipe.create`, `recipe.search`, `recipe.get`, `product.search`

**Специальные паттерны**:
- Structured output для recipe draft (JSON Schema enforced)
- Proposal flow: черновик → показать пользователю → confirm → save
- Умеет работать с фото блюда (анализ → создание рецепта)

**Human checkpoint**: перед `recipe.create`

---

### 3. Nutrition Agent

**Назначение**: Анализ и рекомендации по питанию.

**Умеет**:
- Рассчитывать КБЖУ для продуктов/блюд
- Анализировать баланс рациона за день/неделю
- Проверять достижение целей (калории, белок, жиры, углеводы)
- Предлагать корректировки для достижения целей
- Объяснять нутриционные концепции

**Инструменты**: `user.me`, `mealPlan.dayGet`, `mealPlan.statsGet`, `mealPlan.historyGet`, `product.search`

**Ключевые данные**: `UserProfile.targetCalories/Protein/Fat/Carbs` + `UserProfile.goal`

**Выходные данные**: Всегда текстовые рекомендации, никогда не пишет данные напрямую

---

### 4. Meal Plan Agent

**Назначение**: Управление планом питания.

**Умеет**:
- Получать и показывать текущий план
- Добавлять записи в слоты (с proposal flow)
- Удалять записи (с подтверждением)
- Копировать слоты между днями
- Планировать питание на несколько дней (с учётом целей Nutrition Agent)
- Заполнять недельный план по профилю пользователя

**Инструменты**: все `mealPlan.*`, `product.search`, `recipe.search`, `recipe.get`, `user.me`

**Human checkpoint**: перед `mealPlan.addEntry`, `mealPlan.copySlot`, `mealPlan.removeEntry`

**Взаимодействие**: Получает рекомендации от Nutrition Agent, может вызывать Recipe Agent

---

### 5. Shopping Agent

**Назначение**: Управление списком покупок.

**Умеет**:
- Добавлять товары в список
- Генерировать список из плана питания на период
- Объединять дубликаты
- Группировать по категориям
- Оптимизировать список (убирать лишнее, добавлять недостающее)
- Отмечать купленное

**Инструменты**: все `shoppingList.*`, `product.search`, `mealPlan.historyGet`, `recipe.get`

**Human checkpoint**: перед bulk-добавлением (5+ позиций)

---

### 6. User Profile Agent

**Назначение**: Работа с профилем и предпочтениями пользователя.

**Умеет**:
- Читать и объяснять текущие цели
- Обновлять профиль (вес, активность, цель)
- Пересчитывать TDEE и цели
- Фиксировать метрики тела
- Запоминать долгосрочные предпочтения (хранить в Long-term memory)

**Инструменты**: `user.me`, `userProfile.upsert`, `userTargets.recalculate`, `bodyMetrics.*`

**Долгосрочная память**: Записывает предпочтения в UserMemoryStore

---

### 7. Self-Care Agent

**Назначение**: Управление рутинами ухода за собой.

**Умеет**:
- Создавать недельные рутины
- Добавлять/редактировать/удалять слоты и процедуры
- Предлагать рутины по типу кожи / цели
- Добавлять заметки к процедурам

**Инструменты**: все `selfCare.*`

**Human checkpoint**: перед `selfCare.slotRemove`, `selfCare.itemRemove`

---

### 8. Critic Agent

**Назначение**: Проверка результатов других агентов перед финальным ответом.

**Умеет**:
- Проверять корректность предложенного плана (покрывает ли цели?)
- Проверять рецепт на соответствие ограничениям пользователя (аллергии, диета)
- Выявлять противоречия ("пользователь хочет похудеть, но ты предлагаешь 500г пасты на ужин")
- Давать краткое заключение: ✅ одобрено / ⚠️ есть замечания / ❌ не рекомендую

**Входные данные**: Результат работы другого агента + профиль пользователя + цели
**Выходные данные**: `{ approved: boolean, warnings: string[], suggestions: string[] }`

**Активируется**: По решению Router Agent для критичных операций

---

## Архитектура графа

```
User Message + Context
        ↓
┌────────────────────────────────────────────────────────────┐
│                      LangGraph State                        │
│  messages, context, userId, memory, pendingCheckpoints     │
└────────────────────────────────────────────────────────────┘
        ↓
┌──────────────────┐
│   Router Node    │ ← Определяет намерение + маршрут
└────────┬─────────┘
         │
   ┌─────┴──────────────────────────────────────┐
   │                                            │
   ▼                                            ▼
┌───────────────┐                    ┌───────────────────┐
│ Single Agent  │                    │  Parallel Agents  │
│ (Recipe,      │                    │  (fan-out для     │
│ MealPlan,     │                    │  комплексных      │
│ Shopping,     │                    │  задач)           │
│ SelfCare,     │                    └────────┬──────────┘
│ UserProfile,  │                             │ merge
│ Nutrition)    │                             ▼
└───────┬───────┘                    ┌────────────────┐
        │                            │  Aggregator    │
        │                            └────────┬───────┘
        └──────────────┬─────────────────────┘
                       ↓
              ┌────────────────┐
              │   Critic Node  │ (опционально)
              └────────┬───────┘
                       ↓
              ┌────────────────────┐
              │  Human Checkpoint  │ (если pending actions)
              │  (interrupt graph) │
              └────────┬───────────┘
                       │ confirm / reject
                       ↓
              ┌────────────────┐
              │ Execute Actions│ → MCP Tools
              └────────┬───────┘
                       ↓
              ┌────────────────┐
              │  Final Answer  │ → SSE stream → Frontend
              └────────────────┘
```

---

## Архитектура памяти

### Short-term Memory (кратковременная)

**Что хранит**: Текущий диалог.
**Где**: LangGraph thread state (PostgreSQL-backed через LangGraph Persistence)
**Размер**: Последние N сообщений + summary при превышении лимита
**TTL**: Сессия или 24 часа

**Структура**:
```typescript
{
  threadId: string;           // = conversationId
  messages: Message[];        // полная история
  summary?: string;           // AI-сжатие при >20 сообщениях
  pendingCheckpoints: Checkpoint[];
  activeAgents: string[];
}
```

**Ключевое отличие от текущей реализации**: Хранится в PostgreSQL, переживает рестарты.

---

### Long-term Memory (долгосрочная)

**Что хранит**: Устойчивые предпочтения и факты о пользователе.
**Где**: Отдельная таблица `UserMemory` в PostgreSQL или Redis
**Когда обновляется**: UserProfile Agent фиксирует новую информацию

**Структура**:
```typescript
{
  userId: string;
  key: string;           // "dietary_restrictions", "favorite_breakfast", "usual_snack"
  value: string;
  confidence: number;    // 0-1, снижается если пользователь меняет поведение
  lastUpdated: Date;
  source: "explicit" | "inferred";  // сказал явно или выведено из поведения
}
```

**Примеры записей**:
```
{ key: "dietary_restrictions", value: "не ест молочное", source: "explicit" }
{ key: "preferred_breakfast", value: "омлет или протеиновый коктейль", source: "inferred" }
{ key: "cooking_skill", value: "intermediate", source: "inferred" }
{ key: "meal_prep_style", value: "batch cooking на воскресенье", source: "explicit" }
```

**Загружается**: При каждом запросе Router Agent загружает релевантные факты в контекст.

---

### Semantic Memory (семантическая)

**Что хранит**: Векторные embeddings прошлых действий для семантического поиска.
**Где**: pgvector (PostgreSQL extension) или Pinecone
**Когда используется**: Recipe Agent ищет "похожие рецепты из прошлого", Meal Plan Agent — "что я обычно ем по понедельникам"

**Структура**:
```typescript
{
  userId: string;
  embedding: vector(1536);  // OpenAI text-embedding-3-small
  content: string;          // текстовое представление
  type: "recipe" | "meal_entry" | "conversation_summary" | "user_preference";
  metadata: Record<string, unknown>;
  createdAt: Date;
}
```

**Use cases**:
- "Что я обычно ем на завтрак?" → nearest-neighbor по embeddings meal entries
- "Есть ли рецепт похожий на то что я делала летом?" → semantic search по recipes

---

## Реализация Human-in-the-Loop

### Checkpoint механизм через LangGraph Interrupt

```typescript
// В LangGraph node
const checkpoint: Checkpoint = {
  id: uuid(),
  type: "write_action",
  actions: pendingActions,
  confirmationVariant: "add",
  title: "Добавить в план питания?",
  summary: actions.map(a => a.label).join(", ")
};

// Граф прерывается здесь и ждёт
await interrupt(checkpoint);

// После подтверждения пользователем → граф возобновляется
// с результатом подтверждения в state
```

**Отличие от текущей реализации**:
- Текущая: один `pendingConfirmation` per conversation (теряется при новом запросе)
- LangGraph: multi-checkpoint, каждый checkpoint — состояние графа, граф можно возобновить в любой момент

---

## Streaming через SSE

```
Frontend → POST /v1/agent/chat (создать thread + запустить граф)
         ← EventStream (SSE)
              event: agent_start   { agent: "RecipeAgent" }
              event: tool_call     { tool: "product.search", entity: "брокколи" }
              event: tool_result   { tool: "product.search", count: 3 }
              event: thinking      { text: "Анализирую рецепт..." }
              event: checkpoint    { id: "...", title: "Создать рецепт?" }
              event: text_delta    { delta: "Я подготовила " }
              event: text_delta    { delta: "рецепт омлета..." }
              event: done
```

---

## Технологический стек

| Компонент | Технология |
|-----------|-----------|
| Graph framework | LangGraph.js (`@langchain/langgraph`) |
| State persistence | LangGraph Checkpointer + PostgreSQL |
| LLM calls | LangChain OpenAI + Anthropic adapters |
| Tools | Существующий McpService (без изменений) |
| Streaming | SSE через NestJS @Sse() |
| Tracing | LangSmith |
| Vector store | pgvector (PostgreSQL) |
| Memory | Custom `UserMemoryStore` (PostgreSQL) |

---

## Интеграция с существующим кодом

Ключевой принцип: **McpService остаётся без изменений**.

Все 31 инструмент доступны LangGraph агентам через тот же `McpService.executeTool()` интерфейс. Это минимизирует риски при миграции.

```typescript
// LangGraph tool wrapper для McpService
const mealPlanAddEntry = tool(
  async (args, config) => {
    const userId = config.configurable.userId;
    return mcpService.executeTool("mealPlan.addEntry", args, { userId, ... });
  },
  { name: "mealPlan.addEntry", schema: mealPlanAddEntrySchema }
);
```

---

## Контракт с Frontend

Frontend API (`/v1/agent/chat`) сохраняет обратную совместимость.
Внутренняя реализация меняется с `AgentService` на `LangGraphAgentService`.

Новые возможности для frontend:
- SSE endpoint для streaming
- Несколько checkpoint'ов в одном ответе
- События прогресса агента (какой агент работает)

```typescript
// Новый response shape (расширяет текущий)
{
  ...currentShape,
  stream?: EventSource,          // SSE stream URL
  activeAgent?: string,          // текущий агент
  checkpoints?: Checkpoint[],    // несколько pending actions
  agentPlan?: string,            // что планирует сделать Router
}
```
