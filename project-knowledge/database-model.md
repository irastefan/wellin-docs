# Модель базы данных

## Обзор

Prisma 5.22 + PostgreSQL. 22 модели, 15 enum-типов.  
Файл схемы: `foodieai/prisma/schema.prisma`

---

## Enums

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
| `RoutineWeekday` | MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY |
| `AuthCodePurpose` | REGISTER, LOGIN |
| `UserRole` | USER, ADMIN |

---

## Модели

### Пользователи

#### `User`
```
id                 cuid (PK)
email              String (unique)
passwordHash       String? (устаревший, сейчас OTP)
externalId         String? (unique)
role               UserRole (USER/ADMIN)
paddleCustomerId   String? (unique)
paddleSubscriptionId String? (unique)
subscriptionStatus SubscriptionStatus (default: FREE)
isPro              Boolean (default: false)
subscriptionPlan   String?
createdAt          DateTime
```
**Связи**: profile (1:1), aiSubscription (1:1), aiUsageLogs (1:N), bodyMetricEntries (1:N), mealPlanDays (1:N), shoppingLists (1:N), shoppingCategories (1:N), products (1:N), recipes (1:N), selfCareSlots (1:N)

#### `AuthEmailCode`
```
id        cuid (PK)
email     String
purpose   AuthCodePurpose (REGISTER/LOGIN)
codeHash  String (хэш OTP)
attempts  Int (default: 0)
expiresAt DateTime
createdAt DateTime

@@unique([email, purpose])
@@index([expiresAt])
```

#### `UserProfile`
```
id            cuid (PK, unique per user)
userId        String (FK → User)
firstName     String?
lastName      String?
sex           Sex?
birthDate     DateTime?
heightCm      Int?
weightKg      Float?
activityLevel ActivityLevel?
goal          GoalType? (default: MAINTAIN)
targetFormula TargetFormula (default: MIFFLIN_ST_JEOR)
macroProfile  MacroProfile (default: BALANCED)
targetCalories Int? (рассчитанные целевые калории)
targetProteinG Int?
targetFatG     Int?
targetCarbsG   Int?
calorieDelta  Int? (корректировка от цели)
createdAt/updatedAt DateTime
```

#### `UserBodyMetricEntry`
```
id        cuid (PK)
userId    String (FK → User)
date      DateTime
weightKg  Float?
measurements Json? (замеры: шея, грудь, талия, бёдра и т.д.)

@@unique([userId, date])
@@index([userId, date])
```

---

### AI

#### `AiPlan`
```
id                cuid (PK)
name              String
code              String (unique, например "free" или "pro")
monthlyTokenLimit Int
monthlyAiActions  Int?
priceCents        Int
currency          String (default: "USD")
isActive          Boolean
```
**Связи**: features (1:N), subscriptions (1:N)

#### `AiPlanFeature`
```
id      cuid (PK)
planId  String (FK → AiPlan)
feature AiFeature

@@unique([planId, feature])
@@index([feature])
```

#### `UserAiSubscription`
```
id                 cuid (PK, unique per user)
userId             String (FK → User)
subscriptionStatus SubscriptionStatus (default: FREE)
planId             String? (FK → AiPlan, nullable)
currentPeriodStart DateTime?
currentPeriodEnd   DateTime?
tokensUsed         Int (default: 0)
tokensLimit        Int (default: 0)
trialStartedAt     DateTime?
trialEndsAt        DateTime?
trialTokenLimit    Int?
```

#### `AiUsageLog`
```
id               cuid (PK)
userId           String (FK → User)
subscriptionId   String? (FK → UserAiSubscription)
actionType       String ("responses.create", "agent.chat", ...)
feature          AiFeature?
promptTokens     Int?
completionTokens Int?
totalTokens      Int
model            String

@@index([userId, createdAt])
@@index([subscriptionId, createdAt])
@@index([feature, createdAt])
```

---

### Продукты

#### `Product`
```
id             cuid (PK)
name           String
brand          String?
normalizedName String (для поиска)
scope          ProductScope (GLOBAL/USER)
status         ProductStatus (VERIFIED/PENDING_REVIEW)
ownerUserId    String (FK → User)
source         ProductSource (INTERNAL/FATSECRET/AI_ESTIMATED)
kcal100        Int
protein100     Float
fat100         Float
carbs100       Float

@@index([normalizedName])
```
**Связи**: recipeIngredients (1:N), mealPlanEntries (1:N), shoppingListItems (1:N)

---

### Рецепты

#### `Recipe`
```
id          cuid (PK)
ownerUserId String (FK → User)
title       String
category    String?
description String?
servings    Int?
visibility  RecipeVisibility (PRIVATE)
nutritionTotal      Json? (суммарное КБЖУ)
nutritionPerServing Json? (КБЖУ на порцию)
```
**Связи**: ingredients (1:N), steps (1:N), mealPlanEntries (1:N)

#### `RecipeIngredient`
```
id           cuid (PK)
recipeId     String (FK → Recipe)
order        Int?
originalText String? (оригинальный текст от AI)
name         String
amount       Float?
unit         String?
productId    String? (FK → Product, опциональная связь)
isManual     Boolean (default: false)
kcal100/protein100/fat100/carbs100 Float? (для ручных ингредиентов)
assumptions  Json? (допущения AI)
nutritionTotal/nutritionPerServing Json?
```

#### `RecipeStep`
```
id       cuid (PK)
recipeId String (FK → Recipe)
order    Int
text     String
```

---

### Plan питания

#### `MealPlanDay`
```
id             cuid (PK)
ownerUserId    String (FK → User)
date           DateTime
nutritionBySlot Json? (КБЖУ по слотам)
nutritionTotal  Json? (суммарное КБЖУ за день)

@@unique([ownerUserId, date])
```
**Связи**: entries (1:N)

#### `MealPlanEntry`
```
id         cuid (PK)
dayId      String (FK → MealPlanDay)
slot       MealSlot (BREAKFAST/LUNCH/DINNER/SNACK)
entryType  MealEntryType (PRODUCT/RECIPE)
order      Int
customName String? (для ручных записей)
productId  String? (FK → Product)
recipeId   String? (FK → Recipe)
isManual   Boolean (default: false)
amount     Float?
unit       String?
servings   Float?
nutritionPer100 Json?
nutritionTotal  Json?

@@index([dayId, slot])
@@unique([dayId, slot, order])
```

---

### Список покупок

#### `ShoppingList`
```
id          cuid (PK)
ownerUserId String (FK → User)
title       String (default: "My shopping list")

@@unique([ownerUserId])  — один список на пользователя
```
**Связи**: items (1:N)

#### `ShoppingCategory`
```
id             cuid (PK)
ownerUserId    String (FK → User)
name           String
normalizedName String

@@unique([ownerUserId, normalizedName])
```

#### `ShoppingListItem`
```
id         cuid (PK)
listId     String (FK → ShoppingList)
productId  String? (FK → Product)
customName String?
amount     Float?
unit       String?
note       String?
isDone     Boolean (default: false)
categoryId String? (FK → ShoppingCategory)

@@index([listId, isDone])
```

---

### Self-care рутины

#### `SelfCareRoutineSlot`
```
id             cuid (PK)
ownerUserId    String (FK → User)
weekday        RoutineWeekday (MONDAY..SUNDAY)
name           String (например: "Morning", "Evening")
normalizedName String
order          Int

@@unique([ownerUserId, weekday, normalizedName])
@@unique([ownerUserId, weekday, order])
@@index([ownerUserId, weekday, order])
```
**Связи**: items (1:N)

#### `SelfCareRoutineItem`
```
id          cuid (PK)
slotId      String (FK → SelfCareRoutineSlot)
title       String (например: "Vitamin C serum")
description String?
note        String?
order       Int

@@unique([slotId, order])
@@index([slotId, order])
```

---

### Утилиты

#### `IdempotencyKey`
```
id        cuid (PK)
key       String
operation String
entityId  String?
result    Json?

@@unique([operation, key, entityId])
```
Используется для безопасных повторных запросов.

---

## Схема связей (краткая)

```
User
 ├── UserProfile (1:1)
 ├── UserAiSubscription (1:1) → AiPlan → AiPlanFeature
 ├── AiUsageLog (1:N)
 ├── UserBodyMetricEntry (1:N)
 ├── Product (1:N) → RecipeIngredient, MealPlanEntry, ShoppingListItem
 ├── Recipe (1:N) → RecipeIngredient, RecipeStep, MealPlanEntry
 ├── MealPlanDay (1:N) → MealPlanEntry
 ├── ShoppingList (1:1) → ShoppingListItem
 ├── ShoppingCategory (1:N) → ShoppingListItem
 └── SelfCareRoutineSlot (1:N) → SelfCareRoutineItem
```

---

## Миграции (хронология)

| Дата | Миграция |
|------|----------|
| 2025-12-25 | init — базовые модели |
| 2025-12-27 | add_user_profile |
| 2025-12-28 | add_recipes |
| 2025-12-30 | — |
| 2025-12-31 | — (x2) |
| 2026-02-20 | add_meal_plans, add_shopping_list, user_auth_ownership |
| 2026-03-19 | add_target_formula, add_manual_meal_plan_and_recipe_ingredient_fields |
| 2026-03-26 | add_user_body_metrics_daily |
| 2026-04-03 | add_self_care_routines |
| 2026-04-07 | add_macro_profile_to_user_profile |
| 2026-04-10 | add_ai_subscriptions_and_usage (x2) |
| 2026-04-26 | email_otp_auth |
| 2026-05-17 | add_paddle_subscription_fields |
| 2026-05-30 | add_user_role |

**⚠️ Важно**: НЕ использовать `prisma migrate reset` в production. Только `prisma migrate deploy`.
