# Wellin — Task Board

> Этот файл обновляется автоматически при каждой рабочей сессии.
> Последнее обновление: **2026-06-13** (сессия 8 — pre-launch audit)

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
| ✅ | MCP controller: console.warn/error вместо NestJS Logger | Добавлен `Logger` в `mcp.controller.ts`, заменены все console.* |
| ✅ | SettingsPage: названия вкладок захардкодены на английском | Использует теперь `t("settings.sections.*.title")` — все языки корректны |
| ⬜ 🟡 | recipes.service: silent fail при неизвестном unit ингредиента | `continue` молча пропускает ингредиент → неправильная калорийность рецепта; нужен `logger.warn` |
| ⬜ 🟡 | auth.service: OTP код попадает в лог при `AUTH_EMAIL_ALLOW_LOG_FALLBACK=true` | Dev-фича, но опасна если включить в проде случайно; добавить WARNING в `.env.example` |
| ⬜ 🟡 | Trial period: DEFAULT_TRIAL_DAYS=14 в коде, но docs и `.env` упоминают 7 | Проверить реальное значение в prod `.env.yaml`; синхронизировать billing-plans.md |

---

## Критические задачи перед запуском 🔴

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | UI для PAST_DUE подписки | Баннер в DashboardLayout → кнопка Update payment (Paddle portal) |
| ✅ | UI для исчерпания AI-квоты | `AiQuotaExceededError` → сообщение + кнопка "Выбрать план" |
| ✅ | JWT auto-refresh | `POST /v1/auth/refresh`; проактивный refresh при <7 днях до expiry; 401 → redirect `/login` |
| ✅ | SEO: meta tags, OG, robots.txt | `index.html` OG/Twitter; `robots.txt`; `sitemap.xml`; `useMeta` хук на landing/pricing/legal страницах |
| ⬜ 🔴 | Paddle live flow тест | Тестировался только sandbox. Нужен прогон с реальной картой |
| ⬜ 🔴 | Ограничить действия для не-про подписок | Закрыть/лочить определённые фичи для пользователей без про-подписки. Сейчас `RECIPE_GENERATION` определён в плане, но нигде не гейтится |
| ✅ | Mobile responsive check | Dashboard sidebar скрыт на xs/md; мобильный Drawer + Dock; страницы имеют pb для dock; чарты со скроллом |
| ✅ | Billing UI при отмене подписки | `SubscriptionStatusBanner`: cancel scheduled (info), PAST_DUE (warning), EXPIRED/CANCELED (error) |
| ✅ | RTL проверка (иврит) | theme+Emotion RTL корректны; исправлено 3 проблемы |

---

## Тестирование перед запуском

### Manual QA чеклист (golden paths)

| # | Сценарий | Как проверить | Статус |
|---|----------|---------------|--------|
| 1 | Регистрация нового пользователя через Email OTP | Ввести email → получить код → войти → видеть дашборд | ⬜ |
| 2 | Добавление еды в план питания вручную | Нажать + в слоте → ввести продукт → сохранить → видеть КБЖУ | ⬜ |
| 3 | AI-агент: добавить еду через чат | Написать «добавь овсянку на завтрак» → confirmation card → подтвердить → запись появилась | ⬜ |
| 4 | Diet Planner: сгенерировать дневной план | `/diet-planner` → заполнить форму → OrbitaLoader → план появился в /meal-plan | ⬜ |
| 5 | Создание рецепта через AI | Recipes → AI → описать рецепт → черновик в редакторе → сохранить | ⬜ |
| 6 | Список покупок: добавить из плана питания | Shopping → AI → «добавь всё из плана на понедельник» → items появились | ⬜ |
| 7 | Смена языка EN→RU→HE | Settings → General → поменять язык → все надписи переключились | ⬜ |
| 8 | Dark/Light mode переключение | Кнопка луны → тема переключилась без артефактов | ⬜ |
| 9 | Paddle checkout flow (sandbox) | Pricing → Start Trial → Paddle checkout → webhook → статус обновился | ⬜ |
| 10 | JWT refresh: проверить что работает | Выдать токен с коротким TTL → дождаться авто-refresh → не выбросило на /login | ⬜ |
| 11 | Mobile: основные страницы на 390px | iPhone 14 viewport: meal-plan, recipes, shopping, settings | ⬜ |
| 12 | RTL: ивритская локаль | Переключить на HE → layout зеркален, текст справа | ⬜ |
| 13 | Admin panel: управление квотами | `/admin` → AI Stats → изменить квоту → проверить через API | ⬜ |

### E2E тесты (Playwright) — нет, нужно добавить

| Приоритет | Тест-кейс |
|-----------|-----------|
| 🔴 HIGH | Auth flow: register → login → logout → protected route redirect |
| 🔴 HIGH | Meal plan: add entry → delete entry → TDEE отображается правильно |
| 🟡 MED | Recipe: create → search → edit → delete |
| 🟡 MED | Shopping list: add items → mark done → delete |
| 🟢 LOW | Settings: save profile → recalculate targets |

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
| ✅ | Роут diet-planner в router и навигации | `/diet-planner` добавлен в router.tsx и в navigation.ts |

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
| ⬜ 🟡 | AiAgentPage | Страница "в разработке". Нужен полноценный чат-интерфейс с историей. Сейчас `/ai-agent` редиректит на `/meal-plan` |
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
| ⬜ 🟡 | SettingsPage: inline label strings → i18n | «Interface language», «Theme», «Dark is default» и ещё ~12 строк внутри компонента захардкодены; вкладки уже исправлены |

---

## Технический долг

| Статус | Задача | Детали |
|--------|--------|--------|
| ✅ | Модель в env | `AGENT_MODEL` env var; `LLM_PROVIDER=openai_compat` для Chat Completions (LM Studio/Ollama) |
| ⬜ 🟡 | Retry/backoff для OpenAI | Нет обработки transient ошибок (429, 503) — на высоком трафике возможны сбои |
| ✅ | Admin page: управление квотами действий по планам | GET/PATCH `/v1/admin/ai-plans`, GET `/v1/admin/ai-stats`; карточка с дашбордом статистики и редактированием квот |
| ✅ | [INVESTIGATE] Добавление по фото точнее, чем по тексту | Причина: фото даёт больше контекста — это ожидаемое поведение, не баг |
| ⬜ 🟡 | Prisma type casts | `(this.prisma as any)` в `billing.service.ts`, `ai-subscriptions.service.ts`, `meal-plans.service.ts`, `recipes.service.ts` — снижает типобезопасность |
| ⬜ 🟢 | `shared/ui` barrel exports | Сейчас полные пути к каждому компоненту |
| ⬜ 🟡 | Rate limiting для OTP endpoint | Нет ограничения частоты запросов на `/v1/auth/login/request-code` — уязвимость к abuse/spam |
| ⬜ 🟡 | Активировать гейтинг RECIPE_GENERATION | Feature определена в плане, но нигде не проверяется — Free и Pro пользователи имеют одинаковый доступ к генерации рецептов |
| ⬜ 🟢 | OpenAI API key UX | Ключ хранится в localStorage пользователя (не app key). Неочевидно для новых пользователей. Стоит сделать backend proxy |
| ✅ | MCP controller: Logger | Заменены `console.warn/error` на NestJS `Logger` |

---

## Документация

| Статус | Задача | Файл |
|--------|--------|------|
| ✅ | Обзор проекта | `docs/project-knowledge/project-overview.md` |
| ✅ | Архитектура backend | `docs/project-knowledge/backend-architecture.md` |
| ✅ | Архитектура frontend | `docs/project-knowledge/frontend-architecture.md` |
| ✅ | База данных | `docs/project-knowledge/database-model.md` |
| ✅ | API-карта | `docs/project-knowledge/api-map.md` (+ фото рецепта) |
| ✅ | Агент (реализация) | `docs/project-knowledge/agent-current-implementation.md` |
| ✅ | Billing | `docs/project-knowledge/billing-and-subscriptions.md` |
| ✅ | Технический долг / очистка | `docs/project-knowledge/cleanup-plan.md` |
| ✅ | Готовность к запуску | `docs/project-knowledge/launch-readiness.md` |
| ✅ | LangGraph — простым языком | `docs/ai-architecture/langgraph-explained.md` |
| ✅ | Роадмап LangGraph | `docs/langgraph-migration-roadmap.md` |
| ✅ | Diet Planner — логика работы | `docs/project-knowledge/diet-planner.md` |
| ✅ | future-langgraph-migration.md | Помечен как АРХИВ — LangGraph уже реализован |
| ✅ | current-limitations.md | Обновлён: раздел in-memory диалогов помечен как РЕШЕНО |
| ✅ | prompts-catalog.md | Добавлен diet_planner промпт (раздел 9) |
| ✅ | frontend-architecture.md | Добавлены diet-planner и admin маршруты |
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
| E2E тесты (Playwright) | Критичный путь: auth, meal-plan, recipes, shopping |
| Sentry / error tracking | Структурированный мониторинг ошибок в production |
| AiAgentPage (полноценная) | Отдельный чат-интерфейс с историей диалогов, поиском по чатам |
| Long-term Memory (pgvector) | Агент помнит предпочтения пользователя между сессиями |

---

## Как пользоваться этим файлом

- **Начинаешь сессию** → открой этот файл, посмотри на 🔴 задачи
- **Обнаружила баг** → скажи мне, я добавлю в соответствующий раздел
- **Задача выполнена** → статус меняется на ✅ автоматически в конце сессии
- **Новая идея** → добавляю в "Бэклог"
