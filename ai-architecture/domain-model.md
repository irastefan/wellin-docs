# Domain Model — Wellin

Описание бизнес-модели на основе Prisma schema (`foodieai/prisma/schema.prisma`) и domain services.

---

## Диаграмма связей

```
User
 ├── UserProfile          (1:1) — параметры тела, TDEE-цели, макро-профиль
 ├── UserAiSubscription   (1:1) — AI подписка (free/trial/pro)
 ├── AiUsageLog[]         (1:N) — логи использования AI
 ├── UserBodyMetricEntry[] (1:N) — ежедневные метрики (вес, замеры)
 ├── Product[]            (1:N) — пользовательские продукты
 ├── Recipe[]             (1:N) — рецепты пользователя
 ├── MealPlanDay[]        (1:N) — дни плана питания
 ├── ShoppingList         (1:1) — список покупок
 ├── ShoppingCategory[]   (1:N) — категории покупок
 └── SelfCareRoutineSlot[] (1:N) — слоты рутин ухода

MealPlanDay
 └── MealPlanEntry[]      (1:N) — записи (по слотам BREAKFAST/LUNCH/DINNER/SNACK)
      ├── Product?         → ссылка на продукт
      └── Recipe?          → ссылка на рецепт

Recipe
 ├── RecipeIngredient[]   (1:N) — ингредиенты
 │    └── Product?         → ссылка на продукт
 └── RecipeStep[]         (1:N) — шаги приготовления

ShoppingList
 └── ShoppingListItem[]   (1:N) — элементы
      ├── Product?
      └── ShoppingCategory?

SelfCareRoutineSlot
 └── SelfCareRoutineItem[] (1:N) — процедуры (сыворотка, маска, ...)

UserAiSubscription
 ├── AiPlan?              → план (free/pro/trial)
 └── AiUsageLog[]

AiPlan
 └── AiPlanFeature[]      (1:N) — список включённых AI-фичей
```

---

## Сущности

### User

Основная сущность пользователя.

| Поле | Тип | Описание |
|------|-----|---------|
| `id` | cuid | Primary key |
| `email` | String (unique) | Email адрес |
| `passwordHash` | String? | Не используется (OTP auth) |
| `role` | UserRole | USER / ADMIN |
| `paddleCustomerId` | String? | ID в Paddle |
| `paddleSubscriptionId` | String? | ID подписки Paddle |
| `subscriptionStatus` | SubscriptionStatus | FREE/TRIAL/ACTIVE/EXPIRED/PAST_DUE/CANCELED |
| `isPro` | Boolean | Быстрая проверка Pro-статуса |
| `subscriptionPlan` | String? | Код плана |

---

### UserProfile

Физические параметры и цели пользователя (для расчёта TDEE).

| Поле | Тип | Описание |
|------|-----|---------|
| `sex` | Sex? | FEMALE / MALE |
| `birthDate` | DateTime? | Дата рождения |
| `heightCm` | Int? | Рост (см) |
| `weightKg` | Float? | Вес (кг) |
| `activityLevel` | ActivityLevel? | SEDENTARY/LIGHT/MODERATE/VERY_ACTIVE |
| `goal` | GoalType? | MAINTAIN/LOSE/GAIN |
| `targetFormula` | TargetFormula | Формула TDEE (Mifflin/Harris-Benedict/Owen) |
| `macroProfile` | MacroProfile | BALANCED/HIGH_PROTEIN/LOW_CARB/HIGH_CARB |
| `targetCalories` | Int? | Рассчитанная цель по калориям |
| `targetProteinG` | Int? | Цель по белку (г) |
| `targetFatG` | Int? | Цель по жирам (г) |
| `targetCarbsG` | Int? | Цель по углеводам (г) |
| `calorieDelta` | Int? | Дельта от TDEE (для похудения/набора) |

---

### Product

База продуктов питания.

| Поле | Тип | Описание |
|------|-----|---------|
| `scope` | ProductScope | GLOBAL (общий) / USER (личный) |
| `status` | ProductStatus | VERIFIED / PENDING_REVIEW |
| `source` | ProductSource | INTERNAL / FATSECRET / AI_ESTIMATED |
| `ownerUserId` | String | Владелец (для USER-продуктов) |
| `name` / `brand` | String | Название и бренд |
| `normalizedName` | String | Нормализованное имя для поиска |
| `kcal100` | Int | Калории на 100г |
| `protein100` / `fat100` / `carbs100` | Float | Макронутриенты на 100г |

**Важно**: Продукты бывают публичными (GLOBAL) и личными (USER). AI-created продукты помечаются `AI_ESTIMATED`.

---

### Recipe

Рецепт с ингредиентами и шагами.

| Поле | Тип | Описание |
|------|-----|---------|
| `title` | String | Название |
| `category` | String? | Категория (main, breakfast, ...) |
| `description` | String? | Описание |
| `servings` | Int? | Количество порций |
| `visibility` | RecipeVisibility | PRIVATE / PUBLIC |
| `nutritionTotal` | Json? | Суммарная КБЖУ |
| `nutritionPerServing` | Json? | КБЖУ на порцию |
| `ingredients` | RecipeIngredient[] | Ингредиенты |
| `steps` | RecipeStep[] | Шаги |

---

### RecipeIngredient

Ингредиент рецепта, может быть привязан к продукту или быть ручным.

| Поле | Тип | Описание |
|------|-----|---------|
| `productId` | String? | Ссылка на Product |
| `isManual` | Boolean | true = ручной (без productId) |
| `name` | String | Название |
| `amount` | Float? | Количество |
| `unit` | String? | Единица (g, ml, pcs, ...) |
| `kcal100` / `protein100` / `fat100` / `carbs100` | Float? | Питательность на 100г |
| `nutritionTotal` | Json? | Рассчитанная КБЖУ |

---

### MealPlanDay

День плана питания. Один на (пользователь + дата).

| Поле | Тип | Описание |
|------|-----|---------|
| `date` | DateTime | Дата |
| `nutritionBySlot` | Json? | КБЖУ по слотам |
| `nutritionTotal` | Json? | Суммарная КБЖУ |
| `entries` | MealPlanEntry[] | Записи |

---

### MealPlanEntry

Запись в плане питания.

| Поле | Тип | Описание |
|------|-----|---------|
| `slot` | MealSlot | BREAKFAST / LUNCH / DINNER / SNACK |
| `entryType` | MealEntryType | PRODUCT / RECIPE |
| `order` | Int | Порядок в слоте |
| `productId` | String? | Ссылка на продукт |
| `recipeId` | String? | Ссылка на рецепт |
| `customName` | String? | Ручное название |
| `isManual` | Boolean | Ручная запись (без productId/recipeId) |
| `amount` | Float? | Количество |
| `unit` | String? | Единица |
| `servings` | Float? | Количество порций (для рецептов) |
| `nutritionPer100` | Json? | КБЖУ на 100г |
| `nutritionTotal` | Json? | Суммарная КБЖУ для записи |

---

### ShoppingList / ShoppingListItem / ShoppingCategory

Список покупок пользователя (один на пользователя).

ShoppingListItem:
| Поле | Описание |
|------|---------|
| `productId?` | Привязка к продукту |
| `customName?` | Свободный текст |
| `amount? / unit?` | Количество |
| `note?` | Заметка |
| `isDone` | Отмечен выполненным |
| `categoryId?` | Категория |

---

### SelfCareRoutineSlot / SelfCareRoutineItem

Еженедельные рутины ухода за собой.

SelfCareRoutineSlot:
| Поле | Описание |
|------|---------|
| `weekday` | MONDAY..SUNDAY |
| `name` | Название (Утро, Вечер, ...) |
| `order` | Порядок в дне |

SelfCareRoutineItem:
| Поле | Описание |
|------|---------|
| `title` | Название процедуры |
| `description?` | Описание |
| `note?` | Заметка / частота |
| `order` | Порядок в слоте |

---

### AI модели

#### UserAiSubscription

Подписка пользователя на AI-возможности.

| Поле | Описание |
|------|---------|
| `subscriptionStatus` | FREE/TRIAL/ACTIVE/EXPIRED/... |
| `planId?` | Ссылка на AiPlan |
| `tokensUsed` | Использовано токенов в текущем периоде |
| `tokensLimit` | Лимит токенов |
| `trialStartedAt / trialEndsAt` | Trial период |
| `trialTokenLimit` | Лимит токенов в trial |

#### AiPlan

Шаблон плана (free, pro, ...).

| Поле | Описание |
|------|---------|
| `code` | "free", "pro" (unique) |
| `monthlyTokenLimit` | Лимит токенов в месяц |
| `monthlyAiActions` | Лимит AI-действий в месяц |
| `features` | AiPlanFeature[] — доступные фичи |

Дефолтные значения:
- Free: 100 000 токенов / 100 действий
- Pro: 1 000 000 токенов / 1000 действий
- Trial: 300 000 токенов / 14 дней

#### AiUsageLog

Лог каждого AI-вызова.

| Поле | Описание |
|------|---------|
| `actionType` | "agent.chat", etc. |
| `feature` | AiFeature enum |
| `promptTokens / completionTokens / totalTokens` | Использование |
| `model` | Модель (gpt-5-mini) |

---

### UserBodyMetricEntry

Ежедневные замеры тела.

| Поле | Описание |
|------|---------|
| `date` | Дата замера |
| `weightKg?` | Вес |
| `measurements` | Json (neck, bust, waist, hips, biceps, forearm, thigh, calf — все в см) |

---

### AuthEmailCode

Временный OTP-код для входа.

| Поле | Описание |
|------|---------|
| `email` | Email |
| `purpose` | REGISTER / LOGIN |
| `codeHash` | Хэш кода |
| `attempts` | Количество попыток |
| `expiresAt` | TTL |

---

### IdempotencyKey

Для безопасных повторных запросов (идемпотентность).

---

## Перечисления (Enums)

| Enum | Значения |
|------|----------|
| `ProductScope` | GLOBAL, USER |
| `ProductStatus` | VERIFIED, PENDING_REVIEW |
| `ProductSource` | INTERNAL, FATSECRET, AI_ESTIMATED |
| `RecipeVisibility` | PRIVATE, PUBLIC |
| `Sex` | FEMALE, MALE |
| `ActivityLevel` | SEDENTARY, LIGHT, MODERATE, VERY_ACTIVE |
| `GoalType` | MAINTAIN, LOSE, GAIN |
| `TargetFormula` | MIFFLIN_ST_JEOR, HARRIS_BENEDICT_ORIGINAL, HARRIS_BENEDICT_REVISED, OWEN |
| `MacroProfile` | BALANCED, HIGH_PROTEIN, LOW_CARB, HIGH_CARB |
| `SubscriptionStatus` | FREE, TRIAL, ACTIVE, EXPIRED, PAST_DUE, CANCELED |
| `AiFeature` | RECIPE_GENERATION, MEAL_ANALYSIS, SMART_PRODUCT_MATCH, ADVANCED_AI_TOOLS |
| `MealSlot` | BREAKFAST, LUNCH, DINNER, SNACK |
| `MealEntryType` | PRODUCT, RECIPE |
| `RoutineWeekday` | MONDAY..SUNDAY |
| `UserRole` | USER, ADMIN |
