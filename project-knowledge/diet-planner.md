# Diet Planner — Логика работы

## Что это

Diet Planner — страница `/diet-planner`, на которой пользователь заполняет анкету и AI-агент автоматически генерирует полный дневной план питания (завтрак, обед, ужин, перекус) и сразу добавляет блюда в выбранную дату.

---

## Пользовательский сценарий

```
[/diet-planner]
  ↓ Выбор даты (DateNavigator)
  ↓ Заполнение анкеты:
     - Цель (похудение / поддержание / набор / энергия)
     - Предпочтения по слотам (завтрак: сладкий/солёный, обед: горячий/лёгкий, ужин: лёгкий/сбалансированный/сытный)
     - Тип питания (omnivore / vegetarian / vegan / pescatarian / keto / paleo / mediterranean)
     - Диетические ограничения (глютен, лактоза, орехи, соя, морепродукты, яйца, свинина)
     - Время готовки (до 15 мин / до 30 мин / без ограничений / без готовки)
     - Дополнительные заметки (свободный текст + голосовой ввод)
  ↓ Нажатие «Generate plan»
  ↓ Лоадер: карточка с W-логотипом, анимированными шагами, прогрессом
  ↓ Агент выполняет инструменты (SSE в реальном времени)
  ↓ Карточка подтверждения: превью блюд по слотам с КБЖУ
  ↓ «Add to plan» → план добавлен → redirect на /meal-plan?date=выбранная_дата
```

---

## Архитектура

### Frontend

| Файл | Роль |
|------|------|
| `pages/DietPlannerPage.tsx` | Вся логика: форма → лоадер → подтверждение → навигация |
| `widgets/common/OrbitaLoader.tsx` | Экспортирует `spinKf`, `breatheKf` — используются в анимации W-логотипа |
| `theme/designSystem.ts` | `wellinColors`, `wellinGradients` — цвета W-логотипа и прогресс-бара |
| `features/ai/api/backendAgentApi.ts` | `runStreamingBackendTurn()` — SSE-стриминг к бэкенду |
| `shared/i18n/messages.ts` | Ключи `dietPlanner.*` — все строки в EN/RU/HE |
| `pages/MealPlanDashboardPage.tsx` | Читает `?date=` из URL при маунте → `setSelectedDate()` |

### Backend

| Файл | Роль |
|------|------|
| `agent/prompts/diet-planner.prompt.ts` | Системный промпт для режима `diet_planner` |
| `agent/agent.service.ts` | `processWithConversation()` — основной цикл агента |
| `agent/agent.controller.ts` | `POST /v1/agent/chat/stream` — SSE endpoint |
| `meal-plans/meal-plans.service.ts` | `mealPlan.addEntry`, `mealPlan.dayGet` — инструменты агента |

---

## Форма — что собирается и зачем

### 1. Цель (`goal`)
```
weightLoss → "Goal: weight loss (moderate caloric deficit ~15–20% below TDEE)"
maintain   → "Goal: maintain current weight (match TDEE)"
gain       → "Goal: muscle gain (moderate caloric surplus +200–300 kcal above TDEE)"
energy     → "Goal: energy and wellness (balanced macros, focus on whole foods)"
```
Влияет на распределение КБЖУ и размер порций.

### 2. Предпочтения по слотам (`breakfastStyle`, `lunchStyle`, `dinnerStyle`)
```
breakfast sweet  → "breakfast should be sweet (oatmeal, yogurt, fruits, granola...)"
breakfast savory → "breakfast should be savory (eggs, cheese, toast, avocado...)"
lunch hot        → "lunch should be hot and filling (soup, stew, grain bowl...)"
lunch light      → "lunch should be light and fresh (salads, wraps, cold plates...)"
dinner light     → "dinner should be light and low-calorie (...under ~400 kcal)"
dinner balanced  → "dinner should be balanced (moderate portions...)"
dinner hearty    → "dinner should be hearty and filling (pasta, meat dishes...)"
```
Позволяет персонализировать характер каждого приёма пищи.

### 3. Тип питания и ограничения
Передаются дословно в промпт. Ограничения — жёсткое правило агента (абсолютный запрет игнорировать).

### 4. Время готовки (`cookingTime`)
```
quick    → "max 15 minutes per meal — simple, minimal cooking only"
moderate → "up to 30 minutes per meal"
relaxed  → "no limit — complex, elaborate recipes are welcome"
noCook   → "only ready-to-eat, zero-prep meals (fruits, yogurt, salads, sandwiches...)"
```

### 5. Заметки
Свободный текст передаётся как `Additional preferences: ...`. Поддерживает голосовой ввод.

---

## Построение сообщения агенту (`buildMessage`)

`buildMessage()` формирует одно многострочное сообщение, объединяя все заполненные поля:

```
Generate a complete meal plan for 2026-06-08.
Goal: weight loss (moderate caloric deficit ~15–20% below TDEE)
Diet type: vegetarian.
Dietary restrictions and allergies: glutenFree, nutFree.
Meal style preferences: breakfast should be sweet; dinner should be light and low-calorie.
Prep time constraint: max 15 minutes per meal — simple, minimal cooking only
Additional preferences: no onions
Create breakfast (2–3 items), lunch (3–4 items), dinner (2–3 items), and snack (1–2 items)...
```

Пустые поля (`goal === null`, `any` для предпочтений) не добавляются.

---

## Системный промпт (`diet-planner.prompt.ts`)

Агент работает в режиме `mode: "diet_planner"`. Промпт задаёт:

1. **Роль** — «certified professional dietitian and nutritionist»
2. **Шаги выполнения**:
   - Step 1: `mealPlan.dayGet` → проверить существующие записи, пропустить слоты с ≥2 блюдами
   - Step 2: `mealPlan.addEntry` для всех новых блюд **одним параллельным batch** (не по одному)
3. **Слоты**: BREAKFAST, LUNCH, DINNER, SNACK — по 2–4 блюда в каждом
4. **Абсолютные правила**:
   - Соблюдать все ограничения и аллергии
   - Включать точные КБЖУ на 100г для блюд без `productId`
   - Использовать unit='г'/'мл'/'шт' + gramsPerUnit
5. **Калорийность**: суммарные калории дня — в пределах **±50 ккал** от цели профиля
6. **Финальный ответ**: 2–3 предложения с резюме и персональным советом

---

## SSE-стриминг во время генерации

```
Frontend                              Backend
   │                                     │
   │ POST /v1/agent/chat/stream ────────→│
   │                                     │ agent_start
   │ ←── data: {"type":"agent_start"} ───│
   │                                     │ tool_call (mealPlan.dayGet)
   │ ←── data: {"type":"tool_call",...} ─│
   │                                     │ tool_result
   │ ←── data: {"type":"tool_result",...}│
   │                                     │ tool_call (mealPlan.addEntry x8)
   │ ←── data: {"type":"tool_call",...} ─│
   │                                     │ text_delta (резюме агента)
   │ ←── data: {"type":"text_delta",...} │
   │                                     │ done
   │ ←── data: {"type":"done",...} ──────│
```

**Frontend-сторона** (`runStreamingBackendTurn`):
- Парсит SSE-события через `ReadableStream`
- `onTextDelta` → накапливает `agentText` (показывается в note-боксе лоадера и в confirm-карточке)
- При событии `done` → `reader.cancel()` немедленно (без ожидания закрытия потока)

**Лоадер-карточка** во время генерации:
- W-логотип с вращающимся кольцом (`spinKf` из OrbitaLoader)
- Шаги анимируются таймером: первый шаг через 500ms, каждый следующий через 2200ms (≈15с на 7 шагов)
- Прогресс-бар и % синхронизированы с активным шагом
- Note-бокс: per-step шаблонная фраза (i18n) → при `isDone` заменяется на `agentText` агента
- Кнопка Cancel: останавливает таймер, возвращает форму

---

## Карточка подтверждения

После генерации `requiresConfirmation: true` → агент вернул `confirmation` с блюдами.

`DietPlanPreviewCard` показывает:
- Блюда, сгруппированные по слотам (BREAKFAST / LUNCH / DINNER / SNACK)
- Для каждого слота: суммарные КБЖУ + список блюд с граммовкой
- Кнопка «Add to plan» → `POST /v1/agent/chat` с `confirmationId` → агент выполняет `mealPlan.addEntry`
- Кнопка «Cancel» → возврат к форме

---

## Redirect после применения плана

```typescript
navigate(`/meal-plan?date=${selectedDate}`);
```

`MealPlanDashboardPage` при маунте:
```typescript
const [searchParams] = useSearchParams();
useEffect(() => {
  const dateParam = searchParams.get("date");
  if (dateParam && /^\d{4}-\d{2}-\d{2}$/.test(dateParam)) setSelectedDate(dateParam);
}, []);
```
→ Dashboard открывается сразу на дате, для которой был составлен план.

---

## Связанные файлы

```
smart-food-plan/web-mui/src/
  pages/
    DietPlannerPage.tsx          ← вся логика страницы
    MealPlanDashboardPage.tsx    ← читает ?date= из URL
  widgets/common/
    OrbitaLoader.tsx             ← экспортирует spinKf, breatheKf
  theme/
    designSystem.ts              ← wellinColors, wellinGradients
  features/ai/api/
    backendAgentApi.ts           ← runStreamingBackendTurn()
  shared/i18n/
    messages.ts                  ← ключи dietPlanner.*

foodieai/src/
  agent/prompts/
    diet-planner.prompt.ts       ← системный промпт
  agent/
    agent.service.ts             ← цикл агента, mode routing
    agent.controller.ts          ← POST /v1/agent/chat/stream
```

---

## Ограничения и известные особенности

- **Существующие слоты не перезаписываются**: если в выбранной дате уже есть ≥2 блюда в слоте, агент его пропускает
- **±50 ккал**: это рекомендация, не жёсткий лимит — агент может чуть выйти за пределы на практике
- **Без TanStack Query**: данные не кэшируются, каждый переход на `/meal-plan` делает новый запрос
- **Confirmation обязателен**: агент всегда возвращает `requiresConfirmation: true` для diet_planner — план не добавляется без явного подтверждения пользователем
