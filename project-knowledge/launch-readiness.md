# Готовность к публичному запуску

> Обновлено: 2026-06-08

## Система оценки

- ✅ Готово — реализовано и работает
- ⚠️ Требует доработки — есть проблемы или пробелы
- ❌ Критическая проблема — блокирует запуск

---

## 1. Авторизация

| Функция | Статус | Заметки |
|---------|--------|---------|
| Регистрация через Email OTP | ✅ | Работает |
| Вход через Email OTP | ✅ | Работает |
| JWT токен в localStorage | ✅ | TTL 30 дней |
| Защищённые маршруты (ProtectedRoute) | ✅ | Работает |
| Публичные маршруты (PublicOnlyRoute) | ✅ | Работает |
| Выход из системы | ✅ | Работает (удаление токена) |
| JWT автоматический refresh | ✅ | `POST /v1/auth/refresh`; проактивный при < 7 дней до expiry |
| 401 → редирект `/login` | ✅ | `apiRequest` перехватывает 401 |
| Восстановление доступа | ⚠️ | Нет специального флоу "забыл email" |
| Верификация email | ✅ | OTP = подтверждение email |

---

## 2. AI-функции

| Функция | Статус | Заметки |
|---------|--------|---------|
| Глобальный AI-агент | ✅ | LangGraph + SSE стриминг |
| AI в Meal Plan (page mode) | ✅ | LangGraph, SSE |
| AI в слоте Meal Plan (slot mode) | ✅ | Proposal + confirmation flow |
| AI для рецептов | ✅ | LangGraph RecipeAgentNode, Structured Outputs |
| AI для Shopping List | ✅ | LangGraph ShoppingAgentNode |
| AI для Self-Care | ✅ | LangGraph SelfCareAgentNode |
| Анализ фото еды | ✅ | food_image_analysis mode |
| AI квоты (Free план) | ✅ | 100 000 токенов/месяц |
| AI квоты (Pro план) | ✅ | 1 000 000 токенов/месяц |
| UI при исчерпании квот | ✅ | `AiQuotaExceededError` → сообщение + кнопка "Выбрать план" |
| Многоязычность AI (EN/RU/HE) | ✅ | Централизованный `agent-messages.ts` |
| Персистентная история | ✅ | PostgreSQL через LangGraph checkpointer |
| SSE стриминг (real-time tool progress) | ✅ | `POST /v1/agent/chat/stream` |
| Multi-LLM поддержка | ✅ | `AGENT_MODEL` + `LLM_PROVIDER=openai_compat` для LM Studio/Ollama |
| LangSmith трассировка | ✅ | `LANGSMITH_TRACING=true` + `LANGSMITH_API_KEY` |

---

## 3. Основные пользовательские функции

| Функция | Статус | Заметки |
|---------|--------|---------|
| Дневной план питания | ✅ | Полностью работает |
| Добавление продуктов в слоты | ✅ | Работает |
| Добавление рецептов в слоты | ✅ | Работает |
| КБЖУ расчёт и отображение | ✅ | Работает |
| База продуктов | ✅ | CRUD + поиск |
| Создание рецептов | ✅ | CRUD + AI |
| Список покупок | ✅ | CRUD + категории |
| Self-care рутины | ✅ | CRUD по дням недели |
| Статистика нутриентов | ✅ | Графики неделя/месяц |
| Замеры тела | ✅ | История + дневные записи |
| Профиль + TDEE цели | ✅ | 4 формулы расчёта |
| Навигация по дням | ✅ | Вперёд/назад по плану |

---

## 4. UI и адаптивность

| Аспект | Статус | Заметки |
|--------|--------|---------|
| Desktop версия | ✅ | DashboardLayout с sidebar |
| Mobile версия | ⚠️ | Нужно проверить responsive behaviour на реальных устройствах |
| Dark mode | ✅ | Полная поддержка |
| Light mode | ✅ | Полная поддержка |
| RTL (иврит) | ⚠️ | Заявлен, не проверен на всех страницах |
| EN локализация | ✅ | |
| RU локализация | ✅ | |
| Landing page | ✅ | Маркетинговый контент |
| Pricing page | ✅ | |
| Legal pages | ✅ | Terms, Privacy, Refund |
| Лого и брендинг | ✅ | Wellin |
| Confirmation кнопки (entity-aware) | ✅ | "Add to Plan", "Create Recipe", "Add to List", ... |

---

## 5. Billing

| Функция | Статус | Заметки |
|---------|--------|---------|
| Страница тарифов | ✅ | |
| Checkout через Paddle | ✅ | Требует теста в live |
| Paddle webhooks | ✅ | Реализованы, требуют теста |
| Trial активация | ✅ | По webhook |
| Pro активация | ✅ | По webhook |
| Billing portal | ✅ | Paddle Customer Portal |
| UI для past_due | ✅ | Баннер в DashboardLayout → кнопка Update payment |
| UI для отмены | ✅ | Баннер cancellation scheduled + дата + кнопка "Сохранить подписку" |
| Тест в production | ⚠️ | Sandbox OK, live flow не протестирован |

---

## 6. Производительность и надёжность

| Аспект | Статус | Заметки |
|--------|--------|---------|
| Backend health check | ✅ | GET /health |
| Error handling (backend) | ✅ | GlobalHttpExceptionFilter |
| Error handling (frontend) | ⚠️ | AppErrorState есть, но coverage неполный |
| AI история при рестарте | ✅ | LangGraph PostgreSQL — постоянная |
| OpenAI retry логика | ⚠️ | Нет retry/backoff для 429/503 |
| Logging (backend) | ✅ | HttpLoggingInterceptor |
| CORS настройка | ✅ | Список доменов в main.ts |
| Paddle webhook защита | ✅ | HMAC-SHA256 проверка |
| JWT security | ✅ | Bearer token + авто-refresh |
| Body size limit | ✅ | 20mb (configurable) |

---

## 7. SEO и метаданные

| Аспект | Статус | Заметки |
|--------|--------|---------|
| Page titles | ✅ | `useMeta` хук на всех публичных страницах |
| Meta descriptions | ✅ | OG + Twitter Card в `index.html` и через `useMeta` |
| OG tags (Social sharing) | ✅ | `og:title`, `og:description`, `og:image`, `og:url` |
| Sitemap | ✅ | `public/sitemap.xml` (7 URL) |
| robots.txt | ✅ | `public/robots.txt` |
| favicon | ✅ | В public/ |

---

## 8. Тесты

| Тип | Статус | Заметки |
|-----|--------|---------|
| Backend unit тесты | ✅ | ts-node в test/ (mcp, meal-plan, tdee, ai, agent-foundation) |
| Frontend тесты | ✅ | Vitest: 81 тест (units, http, aiUsageApi, mealPlanApi, i18n) |
| E2E тесты | ❌ | Отсутствуют |
| Paddle webhook тесты | ⚠️ | Требуют live окружения |

---

## Итоговая оценка

### Критические проблемы (блокируют запуск)

Нет критических проблем.

### Требуют доработки перед/после запуска
1. ⚠️ **Протестировать Paddle в live** — только sandbox тестирование
2. ⚠️ **Mobile responsive** — нужна проверка на реальных устройствах
3. ⚠️ **UI при отмене подписки** — нет graceful downgrade сообщения
4. ⚠️ **RTL проверка (иврит)** — задекларирован, не проверен полностью
5. ⚠️ **OpenAI retry/backoff** — нет защиты от transient ошибок 429/503

### Готово к запуску
- ✅ Все основные функции приложения
- ✅ Auth (Email OTP + JWT auto-refresh)
- ✅ AI-агент (LangGraph + SSE + multi-LLM)
- ✅ Billing infrastructure (past_due UI готов)
- ✅ SEO (meta, OG, sitemap, robots.txt)
- ✅ Dark/Light mode
- ✅ EN/RU локализация
- ✅ Персистентная история диалогов (PostgreSQL)
