# План очистки и технический долг

## Приоритеты

- 🔴 Критично — исправить до запуска
- 🟡 Важно — исправить в ближайшее время
- 🟢 Желательно — при возможности

---

## 1. Frontend: дублирование AI путей вызова

### Проблема
В `features/ai/api/` существуют 6 файлов для AI-вызовов:
- `backendAgentApi.ts` — НОВЫЙ путь (рекомендуемый)
- `openaiAgentApi.ts` — СТАРЫЙ путь (прямой OpenAI, через user key)
- `mealPlanAssistantApi.ts` — через `/v1/ai/responses`
- `mealPlanAnalysisApi.ts` — через `/v1/ai/responses`
- `recipeAssistantApi.ts` — через `/v1/ai/responses`
- `mcpApi.ts` — прямой MCP вызов

### Использование
| Диалог | Текущий API |
|--------|------------|
| `GlobalAiAgentDialog` | `backendAgentApi.ts` ✅ |
| `ContextAgentDialog` | `backendAgentApi.ts` ✅ |
| `MealPlanAssistantDialog` | `mealPlanAssistantApi.ts` ❓ Нужно проверить |
| `MealPlanAnalysisDialog` | `mealPlanAnalysisApi.ts` ❓ Нужно проверить |
| `RecipeAssistantDialog` | `recipeAssistantApi.ts` ❓ Нужно проверить |
| `SelfCareAssistantDialog` | ? ❓ Нужно проверить |
| `ShoppingAssistantDialog` | ? ❓ Нужно проверить |

### Рекомендация (🟡 Важно)
Перевести все диалоги на `backendAgentApi.ts` с соответствующим `mode`.  
После проверки — удалить старые API файлы и промпты на frontend.

---

## 2. Frontend: дублирование промптов

### Проблема
12 файлов промптов в двух местах:

**Backend** (`foodieai/src/agent/prompts/`):
- `base.prompt.ts`
- `meal-plan-page.prompt.ts`
- `meal-plan-slot.prompt.ts`
- `recipe.prompt.ts`
- `self-care.prompt.ts`
- `shopping.prompt.ts`

**Frontend** (`smart-food-plan/web-mui/src/features/ai/model/`):
- `agentSystemPrompt.ts` — для openaiAgentApi
- `mealPlanAnalysisPrompt.ts`
- `mealPlanAssistantPrompt.ts`
- `mealPlanPageAssistantPrompt.ts`
- `recipeAssistantPrompt.ts`
- `selfCareAssistantPrompt.ts`
- `shoppingAssistantPrompt.ts`

### Рекомендация (🟡 Важно)
Промпты должны жить ТОЛЬКО на backend.  
Frontend передаёт только `context` и `mode`.  
После перехода всех диалогов на backend agent — удалить frontend промпты.

---

## 3. Устаревшие страницы и компоненты

### Кандидаты на проверку и удаление

| Файл | Статус | Действие |
|------|--------|----------|
| `pages/AiAgentPage.tsx` | Route `/ai-agent` редиректит на `/meal-plan` | Проверить, скорее всего можно удалить |
| `pages/DashboardPlaceholderPage.tsx` | Нигде не используется в роутере | Удалить |
| `features/auth/components/PasswordField.tsx` | OTP не использует пароли | Проверить использование, удалить если unused |
| `features/auth/components/SocialRow.tsx` | Social auth не реализован | Проверить, скорее всего удалить |
| `smart-food-plan/web/` | Старый прототип | НЕ активен, можно архивировать |
| `smart-food-plan/mobile/` | React Native заготовка | НЕ активна, не трогать пока |

### Действие (🟢 Желательно)
Провести grep-аудит использования каждого компонента перед удалением.

---

## 4. Backend: техдолг

### `(this.prisma as any)` каст

В `billing.service.ts` многократно используется `(this.prisma as any)`:
```typescript
await (this.prisma as any).user.update(...)
await (this.prisma as any).user.findUnique(...)
```

**Причина**: Вероятно, несоответствие типов Prisma Client после миграции.  
**Рекомендация** (🟡): Убрать `as any` — обновить типы или добавить поля в Prisma select.

### Захардкоженная модель AI

В `agent.service.ts`:
```typescript
const model = "gpt-5-mini";
```

**Рекомендация** (🟢): Вынести в env-переменную `OPENAI_AGENT_MODEL`.

### In-memory история агента

`AgentConversationService` хранит историю в `Map` — теряется при рестарте.  
В Cloud Run с несколькими инстансами — разные инстансы имеют разные истории.

**Рекомендация** (🟡): Перенести историю в Redis или добавить sticky sessions на Cloud Run.  
Или принять как ограничение (история короткая, потеря некритична).

### Отсутствие retry для OpenAI API

**Рекомендация** (🟢): Добавить exponential backoff для транзиентных ошибок OpenAI.

---

## 5. Отсутствие `index.ts` в shared/ui

Сейчас каждый компонент импортируется по полному пути:
```typescript
import { AppButton } from "../shared/ui/AppButton";
import { AppInput } from "../shared/ui/AppInput";
```

**Рекомендация** (🟢): Создать `shared/ui/index.ts` для barrel exports.

---

## 6. Конфликт дизайн-систем

В корне проекта: `Wellin Design System/redesign/` — новый CSS+HTML дизайн.  
В web-mui: `theme/designSystem.ts` — текущая реализация.  
Расхождение в цветах и стилях.

**Рекомендация** (🟢): Провести аудит расхождений и синхронизировать CSS-переменные.

---

## 7. Отсутствие TanStack Query (кэширование данных)

Сейчас: каждый переход страницы делает новый fetch. Нет кэша, нет optimistic updates.  
При добавлении позиции в meal plan — перезагружаем всю страницу.

**Рекомендация** (🟢): Рассмотреть внедрение TanStack Query для основных сущностей.  
Это существенно улучшит UX и снизит нагрузку на backend.

---

## 8. Тесты

**Текущее состояние**:
- Backend тесты: ts-node based, в `test/` папке
- Есть тесты: agent-foundation, ai.service, mcp-*, meal-plan-stats, tdee, user-me-contract
- Frontend тесты: **отсутствуют**

**Рекомендация** (🟡):
- Добавить frontend smoke tests (хотя бы для основных flows)
- Добавить integration test для Paddle webhook flow
- Добавить E2E тест регистрации + первого meal plan

---

## 9. CORS список

В `main.ts` CORS список захардкожен. При добавлении нового домена нужен деплой.

**Рекомендация** (🟢): Перенести CORS_ORIGINS в env переменную (она уже есть, но с fallback-списком в коде).

---

## Краткий список для быстрой очистки

### Можно удалить без риска (после проверки grep)
- [ ] `pages/DashboardPlaceholderPage.tsx`
- [ ] `features/auth/components/SocialRow.tsx` (если не используется)
- [ ] `features/auth/components/PasswordField.tsx` (если не используется)

### Требует рефакторинга
- [ ] Диалоги перевести на `backendAgentApi.ts`
- [ ] Убрать `(this.prisma as any)` в billing.service.ts

### Требует архивирования
- [ ] `smart-food-plan/web/` — старый прототип

### Требует документирования
- [ ] Что используется в `AiAgentPage.tsx`
- [ ] Статус `mobile/` workspace
