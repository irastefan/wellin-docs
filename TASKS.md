# Wellin — Task Board

> Этот файл обновляется автоматически при каждой рабочей сессии.
> Последнее обновление: **2026-06-08**

---

## Как читать этот файл

| Символ | Значение |
|--------|----------|
| ✅ | Сделано |
| 🔄 | В процессе |
| 🔴 | Критично (нужно до запуска) |
| 🟡 | Важно (сделать вскоре после запуска) |
| 🟢 | Желательно (nice-to-have) |
| ⬜ | Не начато |

---

## Архитектура AI-агента (LangGraph)

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | Phase 0-1: LangGraph + PostgresSaver | Граф запущен, история диалогов сохраняется в PostgreSQL |
| ✅ | Phase 2: SSE стриминг | `POST /v1/agent/chat/stream`, события text_delta/tool_call/tool_result |
| ✅ | Phase 3: RouterNode | Детерминированный роутер + LLM-классификатор для global mode |
| ✅ | Phase 4: RecipeAgentNode | Structured Outputs, надёжный парсинг рецептов |
| ✅ | Phase 5: MealPlanAgentNode + ShoppingAgentNode | Полный доступ к write-инструментам |
| ✅ | SelfCareAgentNode | Маршрут self_care → отдельный узел |
| ✅ | Piece units (gramsPerUnit) | "1 яйцо" корректно считает калории |
| ✅ | Централизованный i18n | Все строки EN/RU/HE в одном файле `agent-messages.ts` |
| ✅ | recipe.create в meal_plan_page | Агент теперь может сохранять рецепты из контекста плана питания |
| ⬜ 🟢 | Phase 6: Long-term Memory | UserMemory + pgvector, помнить предпочтения между сессиями |
| ⬜ 🟢 | Phase 7: CriticAgent | Финальная проверка ответов агента |
| ⬜ 🟡 | LangSmith интеграция | Трассировка и мониторинг агента. Инструкция: `docs/ai-architecture/langgraph-explained.md` |

---

## Фронтенд — очистка и унификация

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | AI API 6→1 файл | Удалены `openaiAgentApi.ts`, `mealPlanAssistantApi.ts`, `recipeAssistantApi.ts`, `mcpApi.ts`, `AiAgentToolsCard.tsx` |
| ✅ | Удаление frontend prompts | Папка `features/ai/model/` удалена — 6 файлов, дублировали backend |
| ⬜ 🟡 | AiAgentPage | Страница "в разработке". Нужен полноценный чат-интерфейс с историей |
| ⬜ 🟢 | TanStack Query | Кэширование данных и optimistic updates |
| ⬜ 🟢 | Удалить неиспользуемые компоненты | `DashboardPlaceholderPage`, `SocialRow`, `PasswordField` |
| ⬜ 🟢 | Архивировать `smart-food-plan/web/` | Старый прототип, не нужен |

---

## Критические задачи перед запуском 🔴

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | UI для PAST_DUE подписки | Баннер в DashboardLayout → кнопка Update payment (Paddle portal) |
| ✅ | UI для исчерпания AI-квоты | `AiQuotaExceededError` → сообщение + кнопка "Выбрать план" |
| ✅ | JWT auto-refresh | `POST /v1/auth/refresh`; проактивный refresh при <7 днях до expiry; 401 → redirect `/login` |
| ✅ | SEO: meta tags, OG, robots.txt | `index.html` OG/Twitter; `robots.txt`; `sitemap.xml`; `useMeta` хук на landing/pricing/legal страницах |
| ⬜ 🔴 | Paddle live flow тест | Тестировался только sandbox. Нужен прогон с реальной картой |
| ⬜ 🟡 | Mobile responsive check | Проверить на реальных устройствах |
| ⬜ 🟡 | Billing UI при отмене подписки | Сообщение при graceful downgrade |
| ⬜ 🟡 | RTL проверка (иврит) | Задекларирован, не проверен на всех страницах |

---

## Технический долг

| Статус | Задача | Детали |
|--------|--------|--------|
| ⬜ 🟡 | Модель в env | `gpt-5-mini` хардкодом в `agent.service.ts:44` → перенести в `OPENAI_AGENT_MODEL` |
| ⬜ 🟡 | Retry/backoff для OpenAI | Нет обработки transient ошибок (429, 503) |
| ⬜ 🟡 | Prisma type casts | `(this.prisma as any)` в `billing.service.ts` |
| ⬜ 🟢 | `shared/ui` barrel exports | Сейчас полные пути к каждому компоненту |
| ⬜ 🟢 | CORS_ORIGINS в env | Сейчас частично хардкодом |

---

## Документация

| Статус | Задача | Файл |
|--------|--------|------|
| ✅ | Обзор проекта | `docs/project-knowledge/project-overview.md` |
| ✅ | Архитектура backend | `docs/project-knowledge/backend-architecture.md` |
| ✅ | Архитектура frontend | `docs/project-knowledge/frontend-architecture.md` |
| ✅ | База данных | `docs/project-knowledge/database-model.md` |
| ✅ | API-карта | `docs/project-knowledge/api-map.md` |
| ✅ | Агент (реализация) | `docs/project-knowledge/agent-current-implementation.md` |
| ✅ | Billing | `docs/project-knowledge/billing-and-subscriptions.md` |
| ✅ | LangGraph — простым языком | `docs/ai-architecture/langgraph-explained.md` |
| ✅ | LangSmith — как подключить | `docs/ai-architecture/langgraph-explained.md` (секция внизу) |
| ✅ | Роадмап LangGraph | `docs/langgraph-migration-roadmap.md` |
| ✅ | Этот файл (таск-борд) | `docs/TASKS.md` |

---

## Бэклог — идеи на будущее

| Идея | Зачем |
|------|-------|
| Анализ питания в NutritionStats | Кнопка AI → агент даёт рекомендации по неделе питания |
| Напоминания / нотификации | Push или email при пропуске рутины или плана питания |
| Импорт рецептов по URL | Вставил ссылку — агент парсит и создаёт рецепт |
| Barcode scanner для продуктов | Мобильное приложение — сканировать штрихкод |
| Seasonal meal suggestions | Агент предлагает блюда по сезону и погоде |

---

## Как пользоваться этим файлом

- **Начинаешь сессию** → открой этот файл, посмотри на 🔴 задачи
- **Обнаружила баг** → скажи мне, я добавлю в соответствующий раздел
- **Задача выполнена** → статус меняется на ✅ автоматически в конце сессии
- **Новая идея** → добавляю в "Бэклог"
