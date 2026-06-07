# Frontend архитектура — smart-food-plan/web-mui

## Обзор

React 19 + MUI v7 + Vite 8. Активная версия приложения.  
Деплой: Vercel + GitHub Pages.  
Dev-сервер: порт **5174**.

---

## Структура `src/`

```
app/            Роутер, guards, провайдеры
pages/          18 страниц (листовые компоненты)
features/       Feature-слой: API-клиенты + хуки
widgets/        Составные компоненты (диалоги, формы, карточки)
shared/
  api/          http.ts — fetch-обёртка с Bearer auth
  i18n/         messages.ts — строки EN/RU
  ui/           Атомарные компоненты (AppButton, AppInput, ...)
  config/       env.ts, aiAgent.ts, openai.ts, appPreferences.ts
  lib/          Утилиты (units.ts, useReveal.ts)
  theme/        macroColors.ts
theme/          appTheme.ts, designSystem.ts
assets/
```

---

## Роутинг (`app/router.tsx`)

### Публичные маршруты
| Путь | Компонент |
|------|-----------|
| `/` | LandingPage |
| `/welcome` | LandingPage |
| `/pricing` | PublicPricingPage |
| `/billing/checkout` | PublicCheckoutPage |
| `/billing/checkout/*` | PublicCheckoutPage |
| `/terms-and-conditions` | PublicLegalPage (terms) |
| `/privacy` | PublicLegalPage (privacy) |
| `/refund` | PublicLegalPage (refund) |

### Auth-only маршруты (PublicOnlyRoute)
| Путь | Компонент |
|------|-----------|
| `/login` | LoginPage |
| `/register` | RegisterPage |

### Защищённые маршруты (ProtectedRoute + DashboardLayout)
| Путь | Компонент |
|------|-----------|
| `/meal-plan` | MealPlanDashboardPage |
| `/recipes` | RecipesPage |
| `/recipes/new` | RecipeEditorPage |
| `/recipes/:recipeId` | RecipeDetailsPage |
| `/recipes/:recipeId/edit` | RecipeEditorPage |
| `/products` | ProductsPage |
| `/products/new` | ProductEditorPage |
| `/products/:productId` | ProductDetailsPage |
| `/products/:productId/edit` | ProductEditorPage |
| `/shopping` | ShoppingPage |
| `/self-care` | SelfCarePage |
| `/stats` | NutritionStatsPage |
| `/ai-agent` | → redirect к `/meal-plan` |
| `/settings` | SettingsPage |

**Примечание**: `/ai-agent` — deprecated route, сейчас редиректит на meal-plan.

---

## Страницы (`pages/`)

| Страница | Файл | Статус |
|----------|------|--------|
| LandingPage | LandingPage.tsx | Активна — маркетинговый лендинг |
| LoginPage | LoginPage.tsx | Активна |
| RegisterPage | RegisterPage.tsx | Активна |
| MealPlanDashboardPage | MealPlanDashboardPage.tsx | Активна — главная страница |
| RecipesPage | RecipesPage.tsx | Активна |
| RecipeEditorPage | RecipeEditorPage.tsx | Активна |
| RecipeDetailsPage | RecipeDetailsPage.tsx | Активна |
| ProductsPage | ProductsPage.tsx | Активна |
| ProductEditorPage | ProductEditorPage.tsx | Активна |
| ProductDetailsPage | ProductDetailsPage.tsx | Активна |
| ShoppingPage | ShoppingPage.tsx | Активна |
| SelfCarePage | SelfCarePage.tsx | Активна |
| NutritionStatsPage | NutritionStatsPage.tsx | Активна |
| SettingsPage | SettingsPage.tsx | Активна |
| PublicPricingPage | public-billing/ | Активна |
| PublicCheckoutPage | public-billing/ | Активна |
| PublicLegalPage | public-billing/ | Активна |
| AiAgentPage | AiAgentPage.tsx | Существует но не в роутере напрямую |
| DashboardPlaceholderPage | DashboardPlaceholderPage.tsx | Вероятно устаревшая |

---

## Features (`features/`)

| Feature | API клиент | Хуки |
|---------|-----------|------|
| `ai/` | aiUsageApi.ts, backendAgentApi.ts, mcpApi.ts, mealPlanAnalysisApi.ts, mealPlanAssistantApi.ts, openaiAgentApi.ts, recipeAssistantApi.ts | — |
| `ai/model/` | — | Промпты: agentSystemPrompt.ts, mealPlanAnalysisPrompt.ts, mealPlanAssistantPrompt.ts, mealPlanPageAssistantPrompt.ts, recipeAssistantPrompt.ts, selfCareAssistantPrompt.ts, shoppingAssistantPrompt.ts |
| `auth/` | authApi.ts | useAuthPage.ts |
| `billing/` | billingApi.ts | — |
| `body-metrics/` | bodyMetricsApi.ts | — |
| `meal-plan/` | mealPlanApi.ts | useMealPlanDashboard.ts |
| `products/` | productsApi.ts | — |
| `recipes/` | recipesApi.ts | — |
| `self-care/` | selfCareApi.ts | useSelfCareRoutine.ts |
| `settings/` | settingsApi.ts | — |
| `shopping/` | shoppingApi.ts | — |

### Важное замечание по AI feature
В `features/ai/` есть **несколько** API-файлов для вызова AI:
- `backendAgentApi.ts` — основной (вызов `/v1/agent/chat`)
- `openaiAgentApi.ts` — прямой вызов OpenAI (со своего ключа)
- `mealPlanAssistantApi.ts` — через `/v1/ai/responses`
- `mealPlanAnalysisApi.ts` — через `/v1/ai/responses`
- `recipeAssistantApi.ts` — через `/v1/ai/responses`
- `mcpApi.ts` — прямой MCP вызов

Это является **техническим долгом** — см. [cleanup-plan.md](./cleanup-plan.md).

---

## Widgets (`widgets/`)

### AI widgets
| Виджет | Файл | Назначение |
|--------|------|------------|
| AgentWorkspace | AgentWorkspace.tsx | Основной workspace агента |
| AiAssistantPanel | AiAssistantPanel.tsx | Панель AI-ассистента |
| AiAgentConversation | AiAgentConversation.tsx | Отображение истории чата |
| AiAgentComposer | AiAgentComposer.tsx | Поле ввода сообщения |
| AgentConfirmationCard | AgentConfirmationCard.tsx | Карточка подтверждения действия |
| AiAgentToolsCard | AiAgentToolsCard.tsx | Отображение tool calls |
| GlobalAiAgentDialog | GlobalAiAgentDialog.tsx | Глобальный диалог агента |
| ContextAgentDialog | ContextAgentDialog.tsx | Контекстный диалог (со страницы) |
| PageAssistantDialogShell | PageAssistantDialogShell.tsx | Общая оболочка для контекстных диалогов |
| AiUsageSummary | AiUsageSummary.tsx | Статистика использования AI |

### Meal Plan widgets
| Виджет | Назначение |
|--------|------------|
| MealPlanDayNavigator | Навигация по дням |
| MealPlanSectionsCard | Слоты питания (4 секции) |
| MealPlanSummaryCard | Сводка дня |
| MealPlanMacroBalanceCard | Баланс КБЖУ |
| MealPlanMetricCard | Метрика (калории/белок/жир/углеводы) |
| MealPlanBodyMetricsCard | Замеры тела |
| MealPlanAssistantDialog | AI-ассистент для meal plan страницы |
| MealPlanAnalysisDialog | Анализ дня через AI |
| MealPlanItemDialog | Добавление/редактирование позиции |

### Dashboard widgets
| Виджет | Назначение |
|--------|------------|
| DashboardLayout | Основной лейаут (sidebar + topbar) |
| DashboardSidebar | Боковое меню |
| DashboardTopbar | Верхняя панель |
| DashboardQuickActions | Быстрые действия |
| DashboardQuickActions | — |
| AiAgentAvatarIcon | Иконка AI-кнопки |

---

## Shared UI компоненты (`shared/ui/`)

| Компонент | Назначение |
|-----------|------------|
| AppButton | Основная кнопка |
| AppInput | Поле ввода |
| AppSelect | Выпадающий список |
| AppLoadingButton | Кнопка с loading state |
| AppContentCard | Карточка контента |
| AppSurface | Поверхность |
| AppStatCard | Карточка статистики |
| AppSectionTitle | Заголовок секции |
| AppEyebrow | Надзаголовок (eyebrow) |
| AppGradientText | Текст с градиентом |
| AppFeedbackToast | Toast-уведомление |
| AppErrorState | Состояние ошибки |
| ConfirmActionDialog | Диалог подтверждения |
| FilterChipRow | Строка фильтров (чипы) |
| FloatingActionMenu | Плавающее меню действий |
| NutritionInlineSummary | Краткое описание нутриентов |
| PageActionButton | Кнопка действия на странице |
| PageTitle | Заголовок страницы |
| PublicSection | Секция лендинга |
| SettingsRow | Строка настроек |
| WellinLogoMark | Логотип |

---

## API Client (`shared/api/http.ts`)

```typescript
apiRequest<T>(path, options) → Promise<T>
```

- Использует `fetch`
- Авторизация: `Bearer` token из `localStorage`
- Ошибки: класс `ApiError` (status + payload)
- Dev-proxy: Vite проксирует `/api` → backend (обход CORS)

---

## Конфигурация (`shared/config/`)

| Файл | Роль |
|------|------|
| `env.ts` | `VITE_API_BASE_URL`, `VITE_API_PROXY_TARGET` |
| `aiAgent.ts` | Настройки AI-агента (тип агента: backend vs openai) |
| `openai.ts` | Настройки прямого OpenAI (для user-owned ключа) |
| `appPreferences.ts` | Пользовательские настройки приложения |

---

## i18n (`shared/i18n/`)

- Поддержка EN + RU
- `messages.ts` — все строки интерфейса
- `languages.ts` — маппинг языков для AI-промптов

---

## Состояние

Нет глобального стора. Данные живут в:
- Хуках компонентов (useState, useEffect)
- Контекстах (LanguageProvider, ThemeModeProvider)
- `localStorage` (JWT token, user preferences)

**Кэша нет** (TanStack Query не используется). Каждый переход к странице делает новый fetch.

---

## Запуск

```bash
cd smart-food-plan/web-mui
npm install
cp .env.example .env   # VITE_API_BASE_URL
npm run dev            # dev-сервер на порту 5174
npm run build          # TypeScript check + Vite build → dist/
npm run preview        # preview собранного dist/
npm run lint           # ESLint
npm run deploy         # build + GitHub Pages
```

### `.env.example`
```
VITE_API_BASE_URL=https://foodieai-59215576464.me-west1.run.app
VITE_API_PROXY_TARGET=https://foodieai-59215576464.me-west1.run.app
```
