# Каталог промптов — Wellin AI Agent

> **Важно:** Этот файл нужно обновлять при каждом изменении промптов.
> Промпты находятся в `foodieai/src/agent/prompts/` и `foodieai/src/agent/i18n/`.
> Последнее обновление: **2026-06-08**

---

## Как строится финальный промпт

Каждый запрос к агенту получает **системный промпт из двух частей**:

```
[Base Prompt]  +  [Mode Prompt]
```

Склейка происходит в `AgentPromptService.buildPrompt()`:
```typescript
[buildBasePrompt(language), buildModePrompt(context, language)].join("\n\n")
```

Режим (`mode`) приходит с фронтенда в поле `context.mode`. Он определяет, на какой странице находится пользователь и какие инструменты ему доступны.

---

## 1. Base Prompt — базовые правила

**Файл:** `prompts/base.prompt.ts`  
**Применяется:** всегда, в любом режиме, первым блоком

### Содержание и объяснение каждого правила

**"You are SmartFood / Wellin AI inside a nutrition planning app."**
→ Задаёт личность агента. Он называет себя Wellin AI, знает что находится внутри приложения питания.

**"You help with meal plans, recipes, shopping lists, food analysis, products, self-care, and user progress."**
→ Перечисляет область знаний — агент не выходит за эти темы и не отвечает на нерелевантные вопросы.

**"Write the final answer in {language}."**
→ Язык ответа определяется настройками пользователя (EN / RU / HE). Передаётся как текстовое название языка через `getLanguageName()` из `agent-messages.ts`.

**"Do not expose internal IDs."**
→ Агент не должен показывать UUID из базы данных. Пользователь видит имена, а не `3f8a-...`.

**"Do not expose internal tool names."**
→ Пользователь не должен видеть `mealPlan.addEntry` или `shoppingList.addItem` в ответах. Только человекочитаемый текст.

**"Use tools only when needed to read real app data or perform app actions."**
→ Агент не вызывает инструменты «на всякий случай». Если данные уже известны из контекста — использует их.

**"Do not claim an app action happened unless a tool actually performed it."**
→ Ключевое правило честности. Агент не может написать «добавил в план» если `mealPlan.addEntry` не был вызван. Без этого правила LLM склонен к галлюцинациям о выполненных действиях.

**"When a user requests multiple items or actions in one message, call all required tools in a single parallel batch."**
→ «Добавь яйца, овсянку и йогурт на завтрак» → три вызова `mealPlan.addEntry` одновременно, а не по очереди с паузами. Критично для скорости.

**"When food is described in pieces or units (e.g. '1 egg', '2 protein bars'), use unit='шт' and always include gramsPerUnit."**
→ Решает проблему подсчёта калорий для штучных продуктов. Без `gramsPerUnit` калории для «1 яйца» считались бы неправильно (база данных хранит на 100г). `gramsPerUnit=55` для яйца = 55г × ккал/100г = правильный результат.  
Дефолты: яйцо=55, протеиновый батончик=55, куриная грудка=200, банан=120, яблоко=180, тост=30.

**"Correct obvious speech-to-text mistakes, keyboard typos, and approximate brand spellings."**
→ Пользователь говорит голосом — «форель» может прийти как «трел» или «торель». Агент исправляет и работает с правильным названием.

**"Ask one short clarification if the request is uncertain."**
→ Если запрос неоднозначен — задать один вопрос, а не несколько. Избегает перегруженного диалога.

**"Keep answers concise and practical."**
→ Агент не пишет эссе. Конкретные короткие ответы.

---

## 2. Router Prompt — маршрутизатор

**Файл:** `prompts/router.prompt.ts`  
**Применяется:** только в `global` режиме для LLM-классификации сообщения  
**Это отдельный, одиночный вызов LLM — не часть основного промпта**

### Задача

Когда пользователь пишет из «глобального» чата (не с конкретной страницы), нужно определить — о чём запрос. Роутер классифицирует сообщение в одну из категорий и отправляет в нужный узел графа.

### Категории

| Категория | Когда |
|-----------|-------|
| `meal_plan` | Добавить, убрать, спланировать, скопировать еду |
| `recipe` | Создать, найти, изменить рецепт |
| `shopping` | Управление списком покупок |
| `self_care` | Рутины, замеры тела, wellness |
| `general` | Приветствие, вопросы о питании, всё остальное |

### Важная особенность

Промпт требует ответ ровно **одним словом** (`Reply with ONLY the category name`). Это позволяет детерминированно парсить ответ без JSON-парсинга. Если ответ не из списка — дефолт `general`.

---

## 3. Global Mode Prompt

**Файл:** `services/agent-prompt.service.ts` (inline строка)  
**Применяется:** когда `mode = "global"` и роутер уже определил `route = "general"`

```
You are in global mode. Use read-only app tools for context and propose changes carefully.
```

Это минималистичный промпт для «общего» разговора с агентом. Агент может смотреть данные (read-only инструменты), но осторожно предлагает изменения. Нет специализированных инструкций.

---

## 4. Meal Plan Page Prompt

**Файл:** `prompts/meal-plan-page.prompt.ts`  
**Применяется:** `mode = "meal_plan_page"` — пользователь на странице плана питания  
**Доступные инструменты:** `mealPlan.dayGet`, `mealPlan.historyGet`, `mealPlan.addEntry`, `mealPlan.removeEntry`, `mealPlan.copySlot`, `recipe.search`, `recipe.get`, `recipe.create`, `product.search`, `shoppingList.get`, `shoppingList.addCategory`, `shoppingList.addItem`

### Динамические данные

В промпт инжектируются:
- `context.selectedDate` — выбранная дата в интерфейсе (может отличаться от сегодня)
- `relativeDates` — объект с yesterday/tomorrow относительно выбранной даты

### Ключевые правила и почему они нужны

**"The selected meal plan date is {date}. Interpret today/yesterday/tomorrow relative to the selected date."**
→ Пользователь может смотреть план на понедельник, а сегодня пятница. Без этого правила «добавь на завтра» добавит на субботу вместо вторника.

**"Prefer loading current day data with mealPlan.dayGet instead of trusting a frontend snapshot."**
→ Фронтенд передаёт снапшот данных в контексте, но он может устареть. Агент всегда перечитывает через API для точности.

**"You also have access to shopping list tools."**
→ С недавнего времени (добавлено в этой сессии) — пользователь может попросить добавить продукт в шопинг-лист прямо с дашборда плана питания.

**"You also have access to recipe.create."**
→ Пользователь может попросить «сохрани это как рецепт» прямо из плана — агент создаёт рецепт с полными нутриентами.

**"When calling mealPlan.removeEntry, always pass itemName."**
→ В карточке подтверждения пользователь должен видеть что именно будет удалено по имени, а не по ID.

**"When using mealPlan.addEntry without productId or recipeId, it is a manual item and you must include name, amount, unit, kcal100..."**
→ Если продукт не найден в базе — агент добавляет «ручную запись» с оценочными данными нутриентов. Обязательные поля перечислены явно, иначе LLM их пропускает.

**"For multi-day planning requests, call mealPlan.addEntry for each day/slot combination... Include all days in a single parallel batch."**
→ «Запланируй завтраки на неделю» → 7 параллельных вызовов, не последовательных. Без этого это занимало бы 7 × время_ответа_LLM.

**"For composed dishes, salads, bowls, keep useful components separate."**
→ «Добавь цезарь с курицей» → два отдельных элемента (салат + курица), а не одна запись «цезарь с курицей». Это важно для точного подсчёта КБЖУ.

**"Use mealPlan.historyGet for fuzzy matching of repeated, branded, or typo-prone foods."**
→ Если пользователь говорит «добавь тот же творог что вчера» — агент ищет в истории вместо того чтобы создавать новый продукт.

---

## 5. Meal Plan Slot Prompt

**Файл:** `prompts/meal-plan-slot.prompt.ts`  
**Применяется:** `mode = "meal_plan_slot"` и `mode = "food_image_analysis"`  
**Ключевое отличие от Meal Plan Page:** агент **не вызывает инструменты изменения** — только готовит предложение

### Контекст

Пользователь открыл диалог добавления еды в конкретный слот (например, завтрак 12 июня). Передаются:
- `context.selectedSlot` — BREAKFAST / LUNCH / DINNER / SNACK
- `context.selectedDate` — дата
- `context.existingItems` — что уже есть в этом слоте

### Почему НЕ вызывать mealPlan.addEntry

В этом режиме агент работает как **"предложения"** — он формирует proposal/items, отправляет их фронтенду, фронтенд показывает карточку подтверждения. Только после подтверждения бэкенд добавляет запись. Если агент вызовет `addEntry` сам — действие выполнится без подтверждения.

### Форма ответа — строгий JSON

```json
{
  "message": "Вот предложение для завтрака",
  "needsConfirmation": true,
  "proposal": {
    "name": "Овсянка на молоке",
    "amount": 250,
    "unit": "g",
    "kcal100": 68,
    "protein100": 3.2,
    "fat100": 1.4,
    "carbs100": 12
  },
  "items": []
}
```

**Правила:**
- `proposal` — используется если предложение ровно одно
- `items` — массив для двух и более отдельных компонентов
- Если `items.length >= 2` → `proposal = null` (они взаимоисключают друг друга)
- `proposal = null, items = []` → нужно уточнение у пользователя

### Режим food_image_analysis

Используется тот же промпт что и `meal_plan_slot`, но контекст приходит с фотографией еды. Агент анализирует что на фото, оценивает нутриенты и создаёт proposal. Логика одинаковая.

---

## 6. Recipe Prompt

**Файл:** `prompts/recipe.prompt.ts`  
**Применяется:** `mode = "recipe_create"`, `"recipe_edit"`, `"recipe_template"`  
**Доступные инструменты:** `recipe.search`, `recipe.get`, `product.search`, `mealPlan.dayGet`, `mealPlan.addEntry`, `shoppingList.get`, `shoppingList.addCategory`, `shoppingList.addItem`

### Три режима — один промпт с условиями

| Режим | Контекст | Поведение |
|-------|---------|-----------|
| `recipe_create` | Новый рецепт с нуля | `intent = "create"` |
| `recipe_edit` | Существующий рецепт (`recipeTitle` из context) | `intent = "update"` |
| `recipe_template` | Шаблон (без конкретного рецепта) | `intent = "create"` |

### Ключевые правила

**"Do not call recipe.create — prepare a draft for the UI editor instead."**
→ Агент не сохраняет рецепт напрямую в базу. Он возвращает черновик, который отображается в редакторе рецептов. Пользователь нажимает «Сохранить»/«Создать» сам. Это даёт возможность проверить и отредактировать перед сохранением.

**"Set intent='update' when preparing an updated draft."**
→ Это поле читает фронтенд — если `intent = "update"` и передан `currentRecipe`, то вызывается `updateRecipe()` вместо `createRecipe()`. Именно из-за отсутствия этого поля в ответе раньше агент всегда создавал новый рецепт вместо обновления (баг исправлен в этой сессии через фронтенд).

**Форма ответа:**
```json
{
  "message": "Вот обновлённый рецепт лосося",
  "intent": "update",
  "draft": {
    "title": "Лосось с овощами",
    "description": "Лёгкое блюдо на ужин",
    "category": "dinner",
    "servings": 2,
    "ingredients": [
      { "name": "Филе лосося", "amount": 100, "unit": "g", "kcal100": 206, "protein100": 20, "fat100": 13, "carbs100": 0 }
    ],
    "steps": ["Посолить лосось.", "Обжарить 10 минут."]
  }
}
```

**"Use one of these categories: breakfast, lunch, dinner, snack, dessert, salad, soup, main, side, drink."**
→ Фиксированный список категорий совпадает с enum на бэкенде. Если агент придумает другую — сохранение упадёт с ошибкой валидации.

---

## 7. Shopping Prompt

**Файл:** `prompts/shopping.prompt.ts`  
**Применяется:** `mode = "shopping_list"`  
**Доступные инструменты:** `shoppingList.get`, `shoppingList.addCategory`, `shoppingList.addItem`, `shoppingList.setItemState`, `shoppingList.removeItem`, `product.search`, `mealPlan.dayGet`

### Задача

Агент-помощник для раздела списка покупок. Умеет:
- Просмотреть текущий список (`shoppingList.get`)
- Добавить категорию и позиции
- Снять/поставить отметку «куплено»
- Сгенерировать список из плана питания (читает `mealPlan.dayGet` за нужные дни, добавляет через `shoppingList.addItem`)

**"When adding multiple items at once, call shoppingList.addItem for each item in a single parallel batch."**
→ Аналогично мил-плану — параллельные вызовы для скорости.

**"Do not claim items were added, removed, or changed unless tools performed the action."**
→ Стандартное правило честности, продублировано в специфическом промпте для надёжности.

---

## 8. Self-Care Prompt

**Файл:** `prompts/self-care.prompt.ts`  
**Применяется:** `mode = "self_care"`  
**Доступные инструменты:** `selfCare.weekGet`, `selfCare.slotCreate`, `selfCare.slotUpdate`, `selfCare.slotRemove`, `selfCare.itemCreate`, `selfCare.itemUpdate`, `selfCare.itemRemove`

### Задача

Помощник для управления wellness-рутинами. Пользователь может попросить добавить/изменить/удалить рутины (утренняя зарядка, приём воды, уход за кожей и т.д.).

**"context.entityId — Do not expose it."**
→ Если пользователь открыл конкретный слот рутины, его ID передаётся в контексте, но агент не показывает его в ответе.

**"Keep suggestions realistic, small, and easy to maintain."**
→ Агент не предлагает «делайте спорт 3 часа в день». Маленькие реалистичные шаги.

---

## 9. Diet Planner Prompt

**Файл:** `prompts/diet-planner.prompt.ts`  
**Применяется:** `mode = "diet_planner"` — пользователь запросил генерацию полного дневного плана  
**Доступные инструменты:** `mealPlan.dayGet`, `mealPlan.addEntry`, `product.search`

### Задача

Агент выступает в роли сертифицированного диетолога. Получает в сообщении предпочтения пользователя (диета, ограничения, цель, стиль питания) и строит полный дневной план на 4 слота (BREAKFAST / LUNCH / DINNER / SNACK).

### Алгоритм выполнения

1. Вызывает `mealPlan.dayGet` → пропускает слоты, где уже ≥ 2 элемента
2. Вызывает `mealPlan.addEntry` для всех новых элементов **одним параллельным батчем**
3. Возвращает краткое резюме с нутритивными акцентами и одним персонализированным советом

### Ключевые правила

- **Строгое соблюдение ограничений** — аллергии и запреты никогда не нарушаются
- **Точность нутриентов** — для каждого элемента без `productId` указываются `kcal100/protein100/fat100/carbs100`
- **Калорийная цель** — суммарный день должен попасть в ±50 ккал от целевого TDEE профиля
- **Практичность** — только доступные, реальные продукты; варьировать текстуры и вкусы по слотам
- **Параллельный батч** — все `mealPlan.addEntry` в одном запросе, не последовательно

### Отличие от meal_plan_page

В `diet_planner` агент **сам вызывает** `mealPlan.addEntry` (не proposal). Пользователь уже дал согласие через форму DietPlannerPage до отправки запроса.

---

## 10. i18n агента — строки подтверждения

**Файл:** `i18n/agent-messages.ts`  
**Применяется:** в `AgentConfirmationService` — для формирования заголовков карточек подтверждения и системных ответов

Это не промпт для LLM, а набор локализованных строк, которые бэкенд формирует программно.

### Что хранится

| Ключ | Назначение |
|------|-----------|
| `promptLanguageName` | Текст для промпта: «Write in Russian» |
| `confirmationPending` | Сообщение пока действие ждёт подтверждения |
| `allSuccess` / `allError` / `partialSuccess` | Ответ после выполнения подтверждённого действия |
| `addTitle` / `removeTitle` / `createTitle` | Заголовок карточки подтверждения |
| `dest.shoppingList` / `dest.selfCare` | Куда добавляется: «в список покупок» / «в рутину» |

### Три языка

| Ключ | EN | RU | HE |
|------|----|----|-----|
| `confirmationPending` | "I prepared the action. Confirm it and I will run it." | "Я подготовила действие. Подтверди, чтобы я его выполнила." | "הכנתי פעולה. אשר כדי שאבצע אותה." |
| `allSuccess` | "Done, the action was completed." | "Готово, действие выполнено." | "בוצע, הפעולה הושלמה." |

---

## Как промпты связаны с tool policy

Каждый режим имеет свой набор разрешённых инструментов в `AgentToolPolicyService`. Промпт и policy должны быть **согласованы** — если промпт упоминает инструмент, он должен быть в списке разрешённых для этого режима, иначе LLM попытается вызвать инструмент, а бэкенд его заблокирует.

```
Промпт упоминает → tool policy разрешает → MCP выполняет
```

### Таблица соответствия

| Режим | Файл промпта | Инструменты добавлены в policy |
|-------|-------------|-------------------------------|
| `meal_plan_page` | `meal-plan-page.prompt.ts` | shoppingList.* добавлены 2026-06-08 |
| `recipe_*` | `recipe.prompt.ts` | shoppingList.* добавлены 2026-06-08 |
| `shopping_list` | `shopping.prompt.ts` | mealPlan.dayGet для генерации по плану |
| `meal_plan_slot` | `meal-plan-slot.prompt.ts` | нет write-инструментов (только proposal) |
| `diet_planner` | `diet-planner.prompt.ts` | mealPlan.dayGet + mealPlan.addEntry (не proposal) |

---

## Что нужно обновить при изменении промптов

1. Обновить этот файл (`docs/ai-architecture/prompts-catalog.md`)
2. Если добавлен новый инструмент в промпт — добавить его в `AgentToolPolicyService` для нужного режима
3. Если изменилась форма JSON-ответа — обновить парсинг на фронтенде (`backendAgentApi.ts`)
4. Если добавлен новый режим — создать файл промпта + добавить case в `AgentPromptService` + добавить `AgentMode` тип + добавить в tool policy
