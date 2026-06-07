# Billing и подписки

## Обзор

Монетизация через **Paddle** (Merchant of Record).  
Два уровня доступа: **Free** и **Pro** (с Trial).

---

## Планы подписки

| Параметр | Free | Pro (Active) | Trial |
|----------|------|--------------|-------|
| Статус | FREE | ACTIVE | TRIAL |
| Токены AI/месяц | 100 000 | 1 000 000 | 300 000 |
| AI actions/месяц | 100 | 1 000 | — |
| Цена | 0 | $9.99/мес | — |
| Срок trial | — | — | 14 дней |
| `isPro` | false | true | true |

---

## Пользовательский сценарий (user journey)

### 1. Регистрация
- Пользователь вводит email
- Backend отправляет OTP через Resend (Cloud Run сервис)
- Пользователь вводит OTP → получает JWT

**После регистрации**:
- Автоматически создаётся Paddle customer (`provisionCustomerForRegistration`)
- Пользователь получает статус `FREE`
- Создаётся `UserAiSubscription` с бесплатным планом

### 2. Использование бесплатной версии
- Доступны все UI-функции приложения
- AI ограничен квотами Free плана (100 000 токенов, 100 actions)
- При превышении квоты — ошибка от `aiUsageService.ensureCanExecute`

### 3. Переход на платную подписку
- Пользователь нажимает кнопку "Upgrade" / переходит на `/pricing`
- Frontend вызывает `POST /v1/billing/checkout-session`
- Backend создаёт Paddle transaction с `PADDLE_PRICE_ID`
- Frontend редиректит на Paddle-hosted checkout страницу

### 4. Trial период
- Если Paddle настроен с trial — пользователь начинает с TRIAL статуса
- Paddle отправляет `subscription.trialing` webhook
- Backend переводит пользователя в статус `TRIAL`, `isPro = true`
- Доступны квоты trial: 300 000 токенов

### 5. Активация Pro
- После успешной оплаты Paddle отправляет webhook `subscription.activated` или `transaction.completed`
- Backend переводит в статус `ACTIVE`, `isPro = true`
- Пользователь получает квоты Pro: 1 000 000 токенов

### 6. Управление подпиской
- `POST /v1/billing/portal-session` → Paddle Customer Portal
- Пользователь может обновить платёжные данные, посмотреть историю

### 7. Отмена
- Пользователь отменяет в Paddle Customer Portal
- Paddle отправляет `subscription.canceled` webhook
- Backend переводит в статус `CANCELED`, `isPro = false`
- Пользователь теряет Pro-доступ

### 8. Продление
- Paddle автоматически списывает каждый месяц
- При успешном списании — `subscription.activated` или `transaction.completed`
- При провале — `subscription.past_due`

---

## Paddle интеграция (backend)

### `BillingService`

**Ключевые методы:**

| Метод | Описание |
|-------|----------|
| `ensurePaddleCustomer(userId, email)` | Создаёт Paddle customer если не существует |
| `provisionCustomerForRegistration(userId, email)` | Тихое создание customer при регистрации |
| `createCheckoutSession(userId, email)` | Создаёт Paddle transaction → URL для оплаты |
| `createBillingPortalSession(userId)` | Создаёт ссылку на Paddle Portal |
| `verifyWebhookSignature(rawBody, signature)` | HMAC-SHA256 проверка подписи |
| `handleWebhook(event)` | Роутинг и обработка Paddle событий |

### Webhook маппинг статусов

```
Paddle "trialing"      → SubscriptionStatus.TRIAL    (isPro = true)
Paddle "active"        → SubscriptionStatus.ACTIVE   (isPro = true)
Paddle "past_due"      → SubscriptionStatus.PAST_DUE (isPro = false)
Paddle "canceled"      → SubscriptionStatus.CANCELED (isPro = false)
Paddle "paused"        → SubscriptionStatus.CANCELED (isPro = false)
```

### Обрабатываемые события
```
transaction.completed    → syncFromTransaction
transaction.paid         → syncFromTransaction
subscription.created     → syncFromSubscription
subscription.trialing    → syncFromSubscription
subscription.activated   → syncFromSubscription
subscription.updated     → syncFromSubscription
subscription.past_due    → syncFromSubscription
subscription.paused      → syncFromSubscription
subscription.resumed     → syncFromSubscription
subscription.canceled    → syncFromSubscription
```

---

## AI-подписки (`ai-access/`)

**Отдельная** система квот для AI, независимая от Paddle.

### Константы
```typescript
DEFAULT_FREE_PLAN_MONTHLY_TOKEN_LIMIT    = 100_000
DEFAULT_FREE_PLAN_MONTHLY_AI_ACTIONS     = 100
DEFAULT_ACTIVE_PLAN_MONTHLY_TOKEN_LIMIT  = 1_000_000
DEFAULT_ACTIVE_PLAN_MONTHLY_AI_ACTIONS   = 1_000
DEFAULT_TRIAL_TOKEN_LIMIT                = 300_000
DEFAULT_TRIAL_DAYS                       = 14
DEFAULT_FREE_PLAN_CODE                   = "free"
DEFAULT_PAID_PLAN_CODE                   = "pro"
```

### Модели AI (в БД)
- `AiPlan` — описание плана (токены, цена, фичи)
- `AiPlanFeature` — какие фичи включены в план
- `UserAiSubscription` — текущая подписка пользователя (токены использованы/лимит)
- `AiUsageLog` — лог каждого AI-запроса

### AI Features
```
RECIPE_GENERATION    — генерация рецептов
MEAL_ANALYSIS        — анализ питания
SMART_PRODUCT_MATCH  — умный поиск продуктов
ADVANCED_AI_TOOLS    — агент (основной AI-чат, используется везде)
```

---

## Переменные окружения для Paddle

```
PADDLE_ENV=sandbox|live
PADDLE_API_KEY_SANDBOX=...
PADDLE_API_KEY_LIVE=...
PADDLE_WEBHOOK_SECRET=...
PADDLE_PRICE_ID=...               — ID тарифного плана в Paddle
PADDLE_CHECKOUT_URL=...           — URL возврата после оплаты
PADDLE_SUBSCRIPTION_PLAN_NAME=monthly_pro
PADDLE_WEBHOOK_TOLERANCE_SECONDS=300
PADDLE_API_BASE_URL=...           — Опционально, переопределяет базовый URL
```

---

## Frontend страницы billing

| Страница | Файл | Описание |
|----------|------|----------|
| `/pricing` | PublicPricingPage.tsx | Публичная страница тарифов |
| `/billing/checkout` | PublicCheckoutPage.tsx | Страница после/во время оплаты |
| `/terms-and-conditions` | PublicLegalPage.tsx (terms) | Пользовательское соглашение |
| `/privacy` | PublicLegalPage.tsx (privacy) | Политика конфиденциальности |
| `/refund` | PublicLegalPage.tsx (refund) | Политика возврата |

---

## Анализ готовности (статус)

| Шаг | Реализован | Протестирован | Проблемы |
|-----|-----------|---------------|----------|
| Регистрация → Paddle customer | ✅ | Частично | Тихий fallback при ошибке |
| OTP Email | ✅ | Да | — |
| Free plan | ✅ | Да | — |
| Checkout session | ✅ | Нет (sandbox?) | — |
| Paddle webhooks | ✅ | Нет (требует live/ngrok) | — |
| Trial активация | ✅ | Нет | — |
| Pro активация | ✅ | Нет | — |
| AI квоты | ✅ | Да (unit tests) | — |
| Billing portal | ✅ | Нет | — |
| Renewal | ✅ | Нет | Нет нотификаций пользователю |
| Cancellation | ✅ | Нет | Нет graceful downgrade UI |
| Past due | ✅ | Нет | Нет UI предупреждения |

### Критические пробелы

1. **Нет UI для состояния past_due** — пользователь не знает что подписка просрочена
2. **Нет UI downgrade-а** — после отмены нет чёткого сообщения "вы переведены на Free"
3. **Admin bypass** — admin пользователи имеют Pro доступ без подписки (это хорошо, но не задокументировано в UI)
4. **Paddle sandbox → live** — не известно протестирован ли полный флоу в production окружении
