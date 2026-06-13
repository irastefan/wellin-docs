# Wellin — Task Board

> Этот файл обновляется автоматически при каждой рабочей сессии.
> Последнее обновление: **2026-06-11** (сессия 7)

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

## Баги (активные)

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | Meal Plan: количество для единицы "Штука" не меняется корректно | `gramsPerUnit` теперь сохраняется в MealPlanEntry и передаётся при update |
| ✅ | Agent settings: кастомный промпт не добавляется к основному | `userInstructions` добавлен в DTO и аппендится к системному промпту |
| ✅ | Admin: очистка неиспользуемых изображений удаляет изображения рецептов | Фикс: DB хранит полные URL, теперь извлекаем GCS-ключ перед сравнением |

---

## Diet Planner (AI-генерация плана питания)

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | Страница DietPlannerPage | Форма → OrbitaLoader → карточка подтверждения → redirect на выбранную дату |
| ✅ | Расширенная форма | Раздел цели (4 варианта), предпочтения по слотам (завтрак/обед/ужин), время готовки |
| ✅ | Лоадер — OrbitaLoader logo | Карточка из HTML-дизайна: W-лого из OrbitaLoader + шаги + прогресс-бар + нота |
| ✅ | SSE стриминг в Diet Planner | `runStreamingBackendTurn` + `onTextDelta` → текст агента в реальном времени |
| ✅ | ±50 ккал допуск в промпте | `diet-planner.prompt.ts` — «within ±50 kcal of target» |
| ✅ | Редирект на выбранную дату | `navigate(/meal-plan?date=${selectedDate})` после применения плана |
| ✅ | MealPlanDashboard читает `?date=` | `useSearchParams` → `setSelectedDate` на маунте |
| ⬜ 🟡 | Роут diet-planner | Убедиться что `/diet-planner` добавлен в router и доступен из навигации |

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
| ✅ | recipeId rule в base prompt | Агент везде (глобальный режим + meal_plan_page) использует recipeId+servings, а не manual entry |
| ✅ | Serving unit в calculateFromProduct | Backend корректно считает калории для порция/serving/portion |
| ✅ | Confirmation в RecipeAssistantDialog | Карточка подтверждения показывается при запросе «добавить рецепт в план» со страницы рецепта |
| ⬜ 🟢 | Phase 6: Long-term Memory | UserMemory + pgvector, помнить предпочтения между сессиями |
| ⬜ 🟢 | Phase 7: CriticAgent | Финальная проверка ответов агента |
| ✅ | LangSmith интеграция | Автоматическая трассировка через `LANGSMITH_TRACING=true`; startup-лог; `start:dev` загружает `.env` через `NODE_OPTIONS` |

---

## Фронтенд — очистка и унификация

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | AI API 6→1 файл | Удалены `openaiAgentApi.ts`, `mealPlanAssistantApi.ts`, `recipeAssistantApi.ts`, `mcpApi.ts`, `AiAgentToolsCard.tsx` |
| ✅ | Удаление frontend prompts | Папка `features/ai/model/` удалена — 6 файлов, дублировали backend |
| ⬜ 🟡 | AiAgentPage | Страница "в разработке". Нужен полноценный чат-интерфейс с историей |
| ⬜ 🟢 | TanStack Query | Кэширование данных и optimistic updates |
| ✅ | Entity-aware кнопки подтверждения | "Add to Plan", "Create Recipe", "Add to List", "Add Routine" — вместо универсального "Add" |
| ⬜ 🟢 | Удалить неиспользуемые компоненты | `DashboardPlaceholderPage`, `SocialRow`, `PasswordField` |
| ⬜ 🟢 | Архивировать `smart-food-plan/web/` | Старый прототип, не нужен |
| ⬜ 🟡 | Agent settings: убрать отображение tool data и выбор моделей | Скрыть из UI настроек агента блок с tool data и селектор моделей |
| ✅ | Desktop sidebar: подменю настроек | Десктоп: навигация по `/settings?section=X` работает корректно; мобильный Drawer: подменю всегда раскрыто |
| ✅ | Settings: навигация по URL-параметру | `SettingsPage` читает `?section=` через `useSearchParams`; tab и URL синхронизированы |
| ✅ | Meal Plan slot AI: паттерн AgentConfirmationCard | Кнопка агента в слоте использует `runBackendAgentTurn` + `AgentConfirmationCard` — без дублирования кода |
| ⬜ 🟡 | Settings: переключатель метрической/имперской системы | Выбор между кг/фунты, см/футы в настройках профиля |
| ⬜ 🟢 | Landing page: добавить раздел с тарифными планами | Секция с ценами и планами на лендинге |
| ⬜ 🟢 | Landing page: добавить переключатель языка | Смена языка прямо на лендинге |
| ✅ | Frontend тесты (Vitest) | 81 тест: units, http, aiUsageApi, mealPlanApi, i18n |

---

## Критические задачи перед запуском 🔴

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | UI для PAST_DUE подписки | Баннер в DashboardLayout → кнопка Update payment (Paddle portal) |
| ✅ | UI для исчерпания AI-квоты | `AiQuotaExceededError` → сообщение + кнопка "Выбрать план" |
| ✅ | JWT auto-refresh | `POST /v1/auth/refresh`; проактивный refresh при <7 днях до expiry; 401 → redirect `/login` |
| ✅ | SEO: meta tags, OG, robots.txt | `index.html` OG/Twitter; `robots.txt`; `sitemap.xml`; `useMeta` хук на landing/pricing/legal страницах |
| ⬜ 🔴 | Paddle live flow тест | Тестировался только sandbox. Нужен прогон с реальной картой |
| ⬜ 🔴 | Ограничить действия для не-про подписок | Закрыть/лочить определённые фичи для пользователей без про-подписки |
| ✅ | Mobile responsive check | Dashboard sidebar скрыт на xs/md; мобильный Drawer + Dock; страницы имеют pb для dock; чарты со скроллом |
| ✅ | Billing UI при отмене подписки | `SubscriptionStatusBanner`: cancel scheduled (info), PAST_DUE (warning), EXPIRED/CANCELED (error) |
| ✅ | RTL проверка (иврит) | theme+Emotion RTL корректны; исправлено 3 проблемы: textAlign "right"→"end" (MealPlanSectionsCard, PricingContent), left→insetInlineStart (LandingNav underline) |

---

## Технический долг

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | Модель в env | `AGENT_MODEL` env var; `LLM_PROVIDER=openai_compat` для Chat Completions (LM Studio/Ollama) |
| ⬜ 🟡 | Retry/backoff для OpenAI | Нет обработки transient ошибок (429, 503) |
| ✅ | Admin page: управление квотами действий по планам | GET/PATCH `/v1/admin/ai-plans`, GET `/v1/admin/ai-stats`; карточка с дашбордом статистики и редактированием квот |
| ✅ | [INVESTIGATE] Добавление по фото точнее, чем по тексту | Причина: фото даёт больше контекста (размер порции, состав, способ приготовления) — это ожидаемое поведение, не баг |
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
| ✅ | Технический долг / очистка | `docs/project-knowledge/cleanup-plan.md` |
| ✅ | Готовность к запуску | `docs/project-knowledge/launch-readiness.md` |
| ✅ | LangGraph — простым языком | `docs/ai-architecture/langgraph-explained.md` |
| ✅ | LangSmith + Multi-LLM | `docs/ai-architecture/langgraph-explained.md` (секции внизу) |
| ✅ | Роадмап LangGraph | `docs/langgraph-migration-roadmap.md` |
| ✅ | Diet Planner — логика работы | `docs/project-knowledge/diet-planner.md` |
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
