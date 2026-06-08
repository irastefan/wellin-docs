# План очистки и технический долг

> Обновлено: 2026-06-08

## Приоритеты

- 🔴 Критично — исправить до запуска
- 🟡 Важно — исправить в ближайшее время
- 🟢 Желательно — при возможности
- ✅ Сделано

---

## 1. Frontend: дублирование AI путей вызова

### ✅ ВЫПОЛНЕНО
Все старые API-файлы удалены. В `features/ai/` остались только:
- `backendAgentApi.ts` — единственный API, вызывает `/v1/agent/chat` и `/v1/agent/chat/stream`
- `aiUsageApi.ts` — загрузка изображений и квоты

Папка `features/ai/model/` (7 frontend-промптов) **удалена**.

---

## 2. Frontend: унификация AI диалогов на backendAgentApi

### ✅ ВЫПОЛНЕНО
Все диалоги переведены на `backendAgentApi.ts` через LangGraph:

| Диалог | API | Статус |
|--------|-----|--------|
| `GlobalAiAgentDialog` | `backendAgentApi.ts` | ✅ |
| `ContextAgentDialog` | `backendAgentApi.ts` | ✅ |
| `MealPlanAssistantDialog` | `backendAgentApi.ts` (SSE) | ✅ |
| `MealPlanAnalysisDialog` | `backendAgentApi.ts` (SSE) | ✅ |
| `RecipeAssistantDialog` | `backendAgentApi.ts` (SSE) | ✅ |
| `SelfCareAssistantDialog` | `backendAgentApi.ts` (SSE) | ✅ |
| `ShoppingAssistantDialog` | `backendAgentApi.ts` (SSE) | ✅ |

---

## 3. Backend: захардкоженная модель AI

### ✅ ВЫПОЛНЕНО
Модель теперь берётся из `AGENT_MODEL` env var (по умолчанию `gpt-4o-mini`).  
`LLM_PROVIDER=openai_compat` переключает на Chat Completions API (LM Studio / Ollama).

---

## 4. Backend: In-memory история агента

### ✅ ВЫПОЛНЕНО
LangGraph + `@langchain/langgraph-checkpoint-postgres` — история диалогов хранится в PostgreSQL.  
Переживает рестарт сервера и работает с несколькими инстансами Cloud Run.

`AgentConversationService` (in-memory Map) сохранён как fallback для тестов и прямого `AgentService.chat()`.

---

## 5. Устаревшие страницы и компоненты

### Кандидаты на удаление (🟢 Желательно)

| Файл | Статус | Действие |
|------|--------|----------|
| `pages/AiAgentPage.tsx` | Route `/ai-agent` редиректит на `/meal-plan` | Проверить использование, вероятно удалить |
| `pages/DashboardPlaceholderPage.tsx` | Нигде не используется | Удалить |
| `features/auth/components/PasswordField.tsx` | OTP не использует пароли | Проверить, удалить если unused |
| `features/auth/components/SocialRow.tsx` | Social auth не реализован | Проверить, вероятно удалить |
| `smart-food-plan/web/` | Старый прототип | НЕ активен, можно архивировать |
| `smart-food-plan/mobile/` | React Native заготовка | НЕ активна, не трогать пока |

---

## 6. Backend: `(this.prisma as any)` каст

В `billing.service.ts` используется `(this.prisma as any)`.

**Причина**: Вероятно несоответствие типов Prisma Client.  
**Рекомендация** (🟡): Убрать `as any`, обновить типы или добавить поля в Prisma select.

---

## 7. Отсутствие retry для OpenAI API

**Рекомендация** (🟡): Добавить exponential backoff для транзиентных ошибок (429, 503).

---

## 8. Frontend: нет глобального кэша

**Текущее состояние**: Нет TanStack Query. Каждый переход на страницу — новый fetch.  
**Рекомендация** (🟢): Добавить TanStack Query для кэширования и optimistic updates.

---

## 9. CORS_ORIGINS частично хардкодом

**Рекомендация** (🟢): Все разрешённые origins должны браться только из `CORS_ORIGINS` env.

---

## 10. `shared/ui` barrel exports

**Текущее состояние**: Используются полные пути к каждому компоненту.  
**Рекомендация** (🟢): Добавить `index.ts` barrel exports для `shared/ui/`.
