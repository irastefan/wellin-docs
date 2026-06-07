# System Overview — Wellin AI Architecture

## Общее описание

Wellin — приложение для питания и здоровья с AI-функциями. Система состоит из трёх уровней:

- **Frontend** (`smart-food-plan/web-mui`) — React 19 + MUI, SPA
- **Backend** (`foodieai`) — NestJS + Prisma + PostgreSQL
- **AI-слой** — OpenAI Responses API + собственный агент + MCP-сервер

---

## Диаграмма потоков системы

```
┌─────────────────────────────────────────────────────────────────┐
│                        Пользователь                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              Frontend: smart-food-plan/web-mui                   │
│                   (React 19 + MUI v7 + Vite)                    │
│                                                                  │
│  ┌──────────────────┐  ┌────────────────────────────────────┐   │
│  │  Page UI         │  │        AI Layer (frontend)         │   │
│  │  MealPlanPage    │  │  openaiAgentApi.ts (direct OpenAI) │   │
│  │  RecipesPage     │  │  backendAgentApi.ts  (→ /v1/agent) │   │
│  │  ShoppingPage    │  │  mcpApi.ts           (→ /mcp)      │   │
│  │  SelfCarePage    │  │  mealPlanAssistantApi.ts           │   │
│  │  AiAgentPage     │  │  recipeAssistantApi.ts             │   │
│  └──────────────────┘  └────────────────────────────────────┘   │
└──────────┬───────────────────────┬─────────────────────────────┘
           │ REST API (JWT Bearer) │
           ▼                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Backend: foodieai (NestJS)                      │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Agent Module                          │    │
│  │                                                          │    │
│  │  AgentController  POST /v1/agent/chat                   │    │
│  │       │                                                  │    │
│  │  AgentService (orchestrator)                            │    │
│  │    ├── AgentPromptService    (prompt building)          │    │
│  │    ├── AgentToolAdapterService (tools → OpenAI format)  │    │
│  │    ├── AgentToolPolicyService  (access control)         │    │
│  │    ├── AgentConversationService (in-memory state)       │    │
│  │    ├── AgentConfirmationService (confirmation flow)     │    │
│  │    ├── AgentOpenAiService   (HTTP → OpenAI API)         │    │
│  │    └── McpService           (tool execution)            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     MCP Module                           │    │
│  │  McpController    POST /mcp  (JSON-RPC)                  │    │
│  │  McpService       (tool registry + execution)            │    │
│  │    ├── 31 зарегистрированных инструмента                 │    │
│  │    └── Делегирует в domain services                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │  Meal Plans  │  │   Recipes    │  │  Shopping List     │    │
│  │  Products    │  │  Self-Care   │  │  Users/Profile     │    │
│  └──────────────┘  └──────────────┘  └────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  AI Access Module                                        │    │
│  │  AiUsageService   (quota check + recording)              │    │
│  │  AiSubscriptionService (plans: free/pro/trial)          │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────┬──────────────────────┬──────────────────────────────┘
            │                      │
            ▼                      ▼
┌─────────────────┐    ┌──────────────────────────────────────────┐
│   PostgreSQL    │    │           Внешние сервисы                 │
│   (Prisma ORM)  │    │  OpenAI Responses API (gpt-5-mini)       │
│   22+ моделей   │    │  Paddle (подписки и платежи)             │
└─────────────────┘    │  Resend (OTP email)                      │
                       │  Google Cloud Storage (AI images)         │
                       └──────────────────────────────────────────┘
```

---

## Два пути вызова AI на frontend

### Путь 1: Backend Agent (основной, рекомендуемый)

```
User → AiAgentPage / MealPlanAssistantDialog / ...
     → backendAgentApi.runBackendAgentTurn()
     → POST /v1/agent/chat  (с JWT)
     → AgentService.chat()
     → AgentOpenAiService → OpenAI Responses API
     → McpService.executeTool() (при необходимости)
     → ответ с toolCalls + answer + confirmation
```

Используется в:
- `AiAgentPage` (global mode)
- `MealPlanAssistantDialog` (meal_plan_page mode)
- `MealPlanItemDialog` (meal_plan_slot / food_image_analysis mode)
- `RecipeAssistantDialog` (recipe_create/edit/template mode)
- `ShoppingAssistantDialog` (shopping_list mode)
- `SelfCareAssistantDialog` (self_care mode)

### Путь 2: Direct OpenAI Agent (legacy/deprecated)

```
User → openaiAgentApi.runAgentTurn()
     → aiUsageApi.postAiResponse()  (через /v1/ai/responses прокси)
     → OpenAI Responses API
     → mcpApi.callMcpTool() → POST /mcp (JSON-RPC)
     → ответ только с сообщениями
```

Используется в некоторых компонентах (более старая реализация).

### Путь 3: Specialized AI APIs (контекстные помощники)

```
mealPlanAnalysisApi → AI для анализа рациона (не агент, разовый вызов)
mealPlanAssistantApi → контекстный помощник для страницы плана
recipeAssistantApi → помощник в редакторе рецептов
```

---

## Квоты и подписки

```
Запрос агента
    ↓
AiUsageService.ensureCanExecute()
    ↓
UserAiSubscription проверяется в БД
    ├── FREE план: 100 000 токенов / мес, 100 AI-действий
    ├── PRO план: 1 000 000 токенов / мес, 1000 AI-действий
    └── TRIAL: 300 000 токенов, 14 дней
    ↓
После выполнения → AiUsageService.recordUsage() → AiUsageLog
```

---

## Деплой

| Компонент | Платформа | URL |
|-----------|-----------|-----|
| Backend | Google Cloud Run | `https://foodieai-59215576464.me-west1.run.app` |
| Frontend | GitHub Pages (+ gh-pages) | prod |
| DB | PostgreSQL (Cloud SQL или managed) | — |
| Auth email | Cloud Run (auth-email-resend) | — |
