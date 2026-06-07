# Tools Catalog

Все инструменты зарегистрированы в `McpService.buildToolRegistry()`.
Сервис: `foodieai/src/mcp/mcp.service.ts`

---

## Мета-инструменты

### `mcp.capabilities`

| | |
|---|---|
| **Назначение** | Возвращает группировку инструментов по интентам и описание flows |
| **Auth** | Не требуется |
| **Public** | Да |
| **Входные параметры** | — |
| **Выходные данные** | `{ toolsByIntent, flows }` |
| **Кто использует** | Claude (внешний), Claude Desktop, отладка |

### `mcp.help`

| | |
|---|---|
| **Назначение** | Возвращает markdown с примерами использования инструментов |
| **Auth** | Не требуется |
| **Public** | Да |
| **Входные параметры** | `topic?: "recipes" | "products" | "users" | "meal-plans" | "self-care" | "shopping-list" | "all"` |
| **Выходные данные** | `{ topic, helpText, examples }` |
| **Кто использует** | Claude (внешний), онбординг AI |

---

## Продукты

### `product.search`

| | |
|---|---|
| **Назначение** | Поиск продуктов по названию/бренду |
| **Auth** | Не требуется (вернёт публичные + свои если авторизован) |
| **Public** | Да |
| **Входные параметры** | `query?: string` |
| **Выходные данные** | `{ count: number, items: Product[] }` |
| **Используется в режимах** | global, meal_plan_page, meal_plan_slot, recipe_create, recipe_edit, food_image_analysis |
| **DTO** | — |

### `product.createManual`

| | |
|---|---|
| **Назначение** | Создание продукта вручную с питательной ценностью на 100г |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `name, kcal100, protein100, fat100, carbs100, brand?, isPublic?` |
| **Выходные данные** | `{ productId, product }` |
| **Используется в режимах** | (не в стандартных режимах агента, только через MCP напрямую) |
| **DTO** | `CreateProductDto` |
| **Особенность** | Идемпотентность: кэш на 30 секунд по сигнатуре `{name, brand, macros}` |

---

## Пользователь и профиль

### `user.me`

| | |
|---|---|
| **Назначение** | Получить текущего пользователя с профилем и целями |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | — |
| **Выходные данные** | `{ userId, user: UserWithProfile }` |
| **Используется в режимах** | global, meal_plan_page, meal_plan_slot, recipe_*, self_care |
| **DTO** | — |

### `userProfile.upsert`

| | |
|---|---|
| **Назначение** | Создать или обновить профиль пользователя |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `firstName?, lastName?, sex?, birthDate?, heightCm?, weightKg?, activityLevel?, goal?, macroProfile?, calorieDelta?` |
| **Выходные данные** | `{ profileId, profile }` |
| **Используется в режимах** | (не в стандартных режимах) |
| **DTO** | `UpsertUserProfileDto` |

### `userTargets.recalculate`

| | |
|---|---|
| **Назначение** | Пересчитать TDEE и макро-цели пользователя |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | — |
| **Выходные данные** | `{ profileId, profile }` |
| **Используется в режимах** | (не в стандартных режимах) |
| **DTO** | — |

---

## Метрики тела

### `bodyMetrics.dayGet`

| | |
|---|---|
| **Назначение** | Получить дневные метрики тела (вес, замеры) |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `date?: string (ISO)` |
| **Выходные данные** | `UserBodyMetricEntry | null` |
| **Используется в режимах** | global |
| **DTO** | `GetBodyMetricsDayDto` |

### `bodyMetrics.upsertDaily`

| | |
|---|---|
| **Назначение** | Создать или обновить дневные метрики тела |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `date?, weightKg?, neckCm?, bustCm?, waistCm?, hipsCm?, bicepsCm?, thighCm?, calfCm?` |
| **Выходные данные** | `UserBodyMetricEntry` |
| **Используется в режимах** | (не в стандартных режимах) |
| **DTO** | `UpsertBodyMetricsDto` |

### `bodyMetrics.historyGet`

| | |
|---|---|
| **Назначение** | История метрик тела за диапазон дат |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `fromDate?, toDate?, limitDays?` |
| **Выходные данные** | `{ entries: UserBodyMetricEntry[] }` |
| **Используется в режимах** | global |
| **DTO** | `GetBodyMetricsHistoryDto` |

---

## План питания

### `mealPlan.dayGet`

| | |
|---|---|
| **Назначение** | Получить план питания на день с нутриентами по слотам и суммарно |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `date?: string (ISO)` |
| **Выходные данные** | `MealPlanDay` с entries по слотам |
| **Используется в режимах** | meal_plan_page, meal_plan_slot |
| **DTO** | `GetMealPlanDayDto` |

### `mealPlan.historyGet`

| | |
|---|---|
| **Назначение** | История добавленных продуктов за период (уникальные, с поиском по имени) |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `date?, fromDate?, toDate?, query?` |
| **Выходные данные** | `{ items: MealPlanEntry[] }` (desc by addedAt) |
| **Используется в режимах** | meal_plan_page, meal_plan_slot, food_image_analysis |
| **DTO** | `GetMealPlanHistoryDto` |

### `mealPlan.statsGet`

| | |
|---|---|
| **Назначение** | Статистика питания за неделю/месяц/произвольный период |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `period?: "week"|"month"|"custom", date?, fromDate?, toDate?` |
| **Выходные данные** | `{ dailyPoints, totals, averages }` |
| **Используется в режимах** | (не в стандартных режимах агента) |
| **DTO** | `GetMealPlanStatsDto` |

### `mealPlan.addEntry`

| | |
|---|---|
| **Назначение** | Добавить продукт/рецепт/ручной элемент в слот плана питания |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `slot (обязателен), date?, productId?, recipeId?, name?, amount?, unit?, servings?, kcal100?, protein100?, fat100?, carbs100?` |
| **Выходные данные** | `MealPlanDay` (пересчитанный) |
| **Используется в режимах** | meal_plan_page (write), meal_plan_slot (через confirmation) |
| **DTO** | `AddMealPlanEntryDto` |
| **Политика** | WRITE — требует подтверждения |
| **Авто-обогащение** | AgentService добавляет `date` из `context.selectedDate` если не передана |

### `mealPlan.copySlot`

| | |
|---|---|
| **Назначение** | Скопировать все записи из слота одного дня в другой |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `sourceDate, sourceSlot, targetDate, targetSlot?` |
| **Выходные данные** | `MealPlanDay` (пересчитанный целевой день) |
| **Используется в режимах** | meal_plan_page |
| **DTO** | `CopyMealPlanSlotDto` |
| **Политика** | WRITE — требует подтверждения |

### `mealPlan.removeEntry`

| | |
|---|---|
| **Назначение** | Удалить запись из плана питания |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `entryId: string` |
| **Выходные данные** | `MealPlanDay` (пересчитанный) |
| **Используется в режимах** | meal_plan_page |
| **DTO** | `RemoveMealPlanEntryDto` |
| **Политика** | DANGEROUS — требует подтверждения, триггерит refresh |

---

## Рецепты

### `recipe.search`

| | |
|---|---|
| **Назначение** | Поиск рецептов по названию/категории |
| **Auth** | Не требуется (публичные + свои если авторизован) |
| **Public** | Да |
| **Входные параметры** | `query?, category?, limit?` (limit ≤ 50) |
| **Выходные данные** | `Recipe[]` |
| **Используется в режимах** | global, meal_plan_page, meal_plan_slot, recipe_create, recipe_edit |
| **DTO** | `SearchRecipesDto` |

### `recipe.get`

| | |
|---|---|
| **Назначение** | Получить рецепт по ID |
| **Auth** | Не требуется (публичный или свой) |
| **Public** | Да |
| **Входные параметры** | `recipeId: string` |
| **Выходные данные** | `{ recipeId, recipe }` |
| **Используется в режимах** | meal_plan_page, meal_plan_slot, recipe_edit, food_image_analysis |
| **DTO** | `RecipeIdDto` |

### `recipe.create`

| | |
|---|---|
| **Назначение** | Создать рецепт с ингредиентами и шагами за один вызов |
| **Auth** | Требуется |
| **Public** | Нет |
| **Входные параметры** | `title, ingredients[], steps[], category?, description?, servings?, isPublic?` |
| **Выходные данные** | `{ recipeId, recipe }` |
| **Используется в режимах** | recipe_create, recipe_edit, recipe_template |
| **DTO** | `CreateRecipeDto` |
| **Политика** | WRITE — требует подтверждения |

---

## Self-Care рутины

### `selfCare.weekGet`

| | |
|---|---|
| **Назначение** | Получить полную недельную рутину (7 дней, слоты, элементы) |
| **Auth** | Требуется |
| **Входные параметры** | — |
| **Выходные данные** | `WeeklyRoutine` |
| **Используется в режимах** | self_care |

### `selfCare.slotCreate`

| | |
|---|---|
| **Назначение** | Создать слот рутины на день недели |
| **Auth** | Требуется |
| **Входные параметры** | `weekday, name, order?` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | WRITE |

### `selfCare.slotUpdate`

| | |
|---|---|
| **Назначение** | Обновить слот (переименовать, перенести) |
| **Auth** | Требуется |
| **Входные параметры** | `slotId, weekday?, name?, order?` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | WRITE |

### `selfCare.slotRemove`

| | |
|---|---|
| **Назначение** | Удалить слот |
| **Auth** | Требуется |
| **Входные параметры** | `slotId` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | DANGEROUS |

### `selfCare.itemCreate`

| | |
|---|---|
| **Назначение** | Создать элемент внутри слота (сыворотка, маска, процедура) |
| **Auth** | Требуется |
| **Входные параметры** | `slotId, title, description?, note?, order?` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | WRITE |

### `selfCare.itemUpdate`

| | |
|---|---|
| **Назначение** | Обновить элемент |
| **Auth** | Требуется |
| **Входные параметры** | `itemId, title?, description?, note?, order?` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | WRITE |

### `selfCare.itemRemove`

| | |
|---|---|
| **Назначение** | Удалить элемент |
| **Auth** | Требуется |
| **Входные параметры** | `itemId` |
| **Выходные данные** | `WeeklyRoutine` |
| **Политика** | DANGEROUS |

---

## Список покупок

### `shoppingList.get`

| | |
|---|---|
| **Назначение** | Получить текущий список покупок с категориями и элементами |
| **Auth** | Требуется |
| **Входные параметры** | — |
| **Выходные данные** | `ShoppingList` с элементами |
| **Используется в режимах** | global, shopping_list |

### `shoppingList.addCategory`

| | |
|---|---|
| **Назначение** | Добавить категорию покупок (Молочное, Фрукты, ...) |
| **Auth** | Требуется |
| **Входные параметры** | `name: string` |
| **Выходные данные** | `ShoppingCategory` |
| **Политика** | WRITE |

### `shoppingList.addItem`

| | |
|---|---|
| **Назначение** | Добавить элемент (по productId или свободным текстом) |
| **Auth** | Требуется |
| **Входные параметры** | `productId?, customName?, amount?, unit?, note?, categoryId?, categoryName?` |
| **Выходные данные** | `ShoppingList` (обновлённый) |
| **Политика** | WRITE |

### `shoppingList.setItemState`

| | |
|---|---|
| **Назначение** | Отметить элемент выполненным/невыполненным |
| **Auth** | Требуется |
| **Входные параметры** | `itemId, isDone: boolean` |
| **Выходные данные** | `ShoppingList` (обновлённый) |
| **Политика** | WRITE |

### `shoppingList.removeItem`

| | |
|---|---|
| **Назначение** | Удалить элемент из списка |
| **Auth** | Требуется |
| **Входные параметры** | `itemId: string` |
| **Выходные данные** | `ShoppingList` (обновлённый) |
| **Политика** | DANGEROUS |

---

## Сводная таблица по политикам

| Категория | Инструменты | Поведение |
|-----------|-------------|-----------|
| **READ_ONLY** (13) | mcp.capabilities, mcp.help, user.me, product.search, recipe.search, recipe.get, mealPlan.dayGet, mealPlan.historyGet, shoppingList.get, bodyMetrics.dayGet, bodyMetrics.historyGet, selfCare.weekGet | Выполняются немедленно |
| **WRITE** (12) | product.createManual, recipe.create, mealPlan.addEntry, mealPlan.copySlot, shoppingList.addItem, shoppingList.addCategory, shoppingList.setItemState, userProfile.upsert, bodyMetrics.upsertDaily, selfCare.slotCreate, selfCare.slotUpdate, selfCare.itemCreate, selfCare.itemUpdate | Требуют подтверждения пользователя |
| **DANGEROUS** (4) | mealPlan.removeEntry, shoppingList.removeItem, selfCare.slotRemove, selfCare.itemRemove | Требуют подтверждения + триггерят refresh данных |
| **НЕ В АГЕНТЕ** (2) | mealPlan.statsGet, userTargets.recalculate | Доступны только через /mcp JSON-RPC |
