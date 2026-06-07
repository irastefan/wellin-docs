# Будущая миграция на LangGraph

> **Важно**: Это документ планирования. Никаких изменений кода на этом этапе.  
> Реализация только после полной стабилизации текущего агента.

---

## Текущая архитектура агента

```
AgentService
  ├── runOpenAiLoop() — цикл до 4 шагов
  │    ├── openAi.createResponse() — OpenAI Responses API
  │    ├── toolPolicy.isAllowed() — проверка доступа
  │    ├── toolPolicy.requiresConfirmation() — нужна ли конфирмация
  │    └── mcpService.executeTool() — выполнение инструмента
  ├── applyStructuredResponse() — парсинг JSON из ответа
  └── AgentConversationService — in-memory история (Map)
```

### Технологии
- **LLM**: OpenAI Responses API (`gpt-5-mini`)
- **Tools**: MCP Service (30 инструментов)
- **Memory**: In-memory Map (теряется при рестарте)
- **Orchestration**: Ручной цикл (max 4 шага)

---

## Слабые места текущего решения

### 1. Отсутствие персистентной памяти
Текущее: `Map<conversationId, StoredAgentConversation>` — in-memory.  
При рестарте Cloud Run вся история теряется.  
При горизонтальном масштабировании — разные инстансы не видят историю друг друга.

**LangGraph решение**: LangGraph имеет встроенный `Checkpointer` (Redis, PostgreSQL). История автоматически сохраняется и доступна из любого инстанса.

### 2. Жёсткий цикл (max 4 шага)
Текущее: `for (let step = 0; step < 4; step++)` — простой цикл.  
Для сложных сценариев (создание рецепта + добавление в план + обновление shopping list) может не хватить шагов.

**LangGraph решение**: граф с явными состояниями. Каждый инструмент — отдельный узел. Нет искусственного лимита шагов.

### 3. Монолитная логика агента
Текущее: `agent.service.ts` — 630 строк, всё в одном месте.  
Трудно тестировать отдельные части, трудно расширять.

**LangGraph решение**: каждый шаг — отдельная функция/узел. Легко добавить новые ветви без изменения существующей логики.

### 4. Нет параллельного выполнения инструментов
Текущее: инструменты выполняются последовательно.  
Для запросов типа "добавь в план И в shopping list" — два последовательных tool call.

**LangGraph решение**: параллельные ребра в графе. Несколько инструментов запускаются одновременно.

### 5. Хрупкий парсинг JSON из ответа
Текущее: `applyStructuredResponse()` — regex + try/catch.  
Ненадёжно при нестандартных ответах модели.

**LangGraph решение**: structured output через Pydantic/Zod схемы, проверенные на этапе компиляции графа.

---

## Какие backend сервисы могут стать tools

Все 30 текущих MCP инструментов остаются tools, но организованы иначе:

### Группы инструментов (возможные nodes в графе)

| Группа | Tools | LangGraph node |
|--------|-------|----------------|
| Данные пользователя | `user.me`, `userProfile.upsert`, `bodyMetrics.*` | `user_data_node` |
| Продукты | `product.search`, `product.createManual` | `products_node` |
| Рецепты | `recipe.search`, `recipe.get`, `recipe.create` | `recipes_node` |
| Plan питания | `mealPlan.*` | `meal_plan_node` |
| Покупки | `shoppingList.*` | `shopping_node` |
| Self-care | `selfCare.*` | `self_care_node` |
| Подтверждение | Новый | `confirmation_node` |

---

## Предполагаемая архитектура LangGraph

```
                    [START]
                       ↓
              [Intent Classifier]
              (понять тип запроса)
                       ↓
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   [Meal Plan]    [Recipes]    [Shopping]  ...
   subgraph       subgraph     subgraph
        ↓              ↓              ↓
 [Tool Executor]  [Tool Executor]  ...
 (параллельно)
        ↓
 [Confirmation Gate]
 (нужна ли конфирмация?)
        ↓
 [Response Generator]
        ↓
      [END]
```

### Состояние агента (AgentState)
```typescript
type AgentState = {
  userId: string;
  conversationId: string;
  messages: BaseMessage[];
  mode: AgentMode;
  context: AgentContext;
  pendingConfirmation: PendingConfirmation | null;
  toolCalls: AgentToolCallResult[];
  data: Record<string, unknown>;
  shouldRefreshData: boolean;
}
```

---

## Сценарии, подходящие для LangGraph

### Подходят отлично
1. **Сложный meal planning**: "Составь план питания на неделю, учти мои цели и добавь список покупок"
   - Требует: `user.me` → несколько `mealPlan.addEntry` → `shoppingList.addItem`
   - LangGraph: параллельное добавление, checkpoint после каждого дня

2. **Recipe research**: "Найди рецепты с курицей, выбери подходящий и добавь в план на ужин"
   - Требует: `recipe.search` → анализ → `mealPlan.addEntry`
   - LangGraph: явная ветвь "выбор рецепта"

3. **Self-care planning**: "Создай полную утреннюю рутину на неделю"
   - Требует: множество `selfCare.slotCreate` + `selfCare.itemCreate`
   - LangGraph: loop node для каждого дня недели

### Не требуют LangGraph (остаются как есть)
- Простой чат: "Сколько калорий в 100г куриной грудки?"
- Одно действие: "Добавь 200г риса в обед"
- Поиск: "Найди рецепты с тунцом"

---

## Этапы безопасной миграции

### Этап 0: Подготовка (сейчас)
- [ ] Стабилизировать текущего агента (унифицировать диалоги)
- [ ] Написать тесты для существующего agent flow
- [ ] Задокументировать все edge cases
- [ ] Убедиться что все MCP инструменты работают корректно

### Этап 1: LangGraph параллельно (feature flag)
- [ ] Установить `@langchain/langgraph` как dev dependency
- [ ] Реализовать простейший граф для одного режима (например `meal_plan_page`)
- [ ] Добавить feature flag `USE_LANGGRAPH=true/false`
- [ ] Тестировать оба пути параллельно

### Этап 2: Постепенная миграция
- [ ] Перенести `global` mode на LangGraph
- [ ] Добавить персистентную память (Redis checkpointer)
- [ ] Перенести `meal_plan_page` mode
- [ ] Постепенно перенести остальные режимы

### Этап 3: Финальный
- [ ] Удалить старый `agent.service.ts` цикл
- [ ] Все режимы работают через LangGraph
- [ ] Мониторинг и алерты настроены

---

## Технические требования для LangGraph

### Новые зависимости
```
@langchain/core
@langchain/langgraph
@langchain/openai (или прямой OpenAI SDK)
```

### Инфраструктура
- **Redis** — для LangGraph Checkpointer (персистентная память)
  - Или PostgreSQL Checkpointer (есть уже PG)
- **Трассировка** — LangSmith для мониторинга графов

### Минимальное изменение backend
```typescript
// Новый AgentGraphService рядом со старым AgentService
// Старый AgentService остаётся как fallback
// AgentController выбирает который использовать
```

---

## Риски миграции

| Риск | Вероятность | Митигация |
|------|-------------|----------|
| LangGraph overhead (latency) | Средняя | Профилировать перед деплоем |
| Breaking changes в LangGraph API | Низкая | Закрепить версию |
| Сложность отладки графов | Средняя | LangSmith трассировка |
| Redis как новая зависимость | Низкая | PostgreSQL checkpointer как альтернатива |
| Prompt несовместимость | Высокая | Тщательное тестирование промптов |

---

## Решение НЕ мигрировать (если текущего хватает)

Если приложение стабильно работает с текущим агентом, LangGraph НЕ обязателен.  
LangGraph оправдан только если:
- Нужны сложные multi-step сценарии (5+ шагов)
- Нужна персистентная память между сессиями
- Нужно параллельное выполнение инструментов
- Нужна визуализация и мониторинг графов

**Рекомендация**: Сначала запустить продукт. LangGraph — следующий шаг после Product-Market Fit.
