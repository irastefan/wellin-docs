# Как работает подписка в Wellin

Полный гайд: от регистрации нового пользователя до всех возможных сценариев.

---

## Планы и лимиты

| | Free | Trial | Pro |
|---|---|---|---|
| **Статус** | `FREE` | `TRIAL` | `ACTIVE` |
| **Токены / месяц** | 100 000 | 300 000 (разово за весь триал) | 1 000 000 |
| **AI actions / месяц** | 100 | ~300 | 1 000 |
| **Цена** | $0 | $0 (первые 14 дней) | $9.99/мес |
| **Срок действия** | бессрочно | 14 дней | ежемесячно |
| **Генерация рецептов** | ✗ | ✓ | ✓ |
| **Анализ питания** | ✓ | ✓ | ✓ |
| **Умный поиск продуктов** | ✓ | ✓ | ✓ |
| **Расширенный AI агент** | ✗ | ✓ | ✓ |

**1 AI action = 1 000 токенов.** Когда агент выполняет запрос, он тратит токены пропорционально длине сообщений и ответа.

---

## Откуда берётся TokenLimit для триала

Токены триала — **одноразовый пул**, не возобновляемый ежемесячно. Если пользователь за 14 дней потратит все 300 000 токенов — AI заблокируется раньше, чем истечёт время. Если не потратит — всё равно после 14 дней план переключится на Free.

У Pro: лимит **обновляется каждый месяц** — токены сбрасываются в 0 при каждом новом расчётном периоде.

---

## Путь нового пользователя

### 1. Регистрация

```
Пользователь вводит email
  → Backend отправляет OTP через Resend (Cloud Run)
  → Пользователь вводит код → получает JWT
  → Paddle customer создаётся тихо (ошибка не блокирует регистрацию)
  → User создан, User.subscriptionStatus = FREE
```

В момент регистрации `UserAiSubscription` ещё **не создаётся**.

### 2. Первый вход в AI-функции (триал начинается)

Когда пользователь впервые открывает дашборд или запрашивает AI, frontend вызывает `GET /v1/me/ai-usage`. В этот момент backend:

```
getNormalizedState() → findSubscription() → null
  → createInitialSubscription()
     subscriptionStatus = TRIAL
     planId = plan_pro  ← триал даёт PRO-фичи
     tokensLimit = 300 000
     trialStartedAt = now
     trialEndsAt = now + 14 дней
```

**Вывод:** Trial начинается не при регистрации, а при первом обращении к AI. Если пользователь зарегистрировался и ничего не сделал — триал не «тикает».

### 3. Использование в период триала

- Все 4 AI-фичи доступны
- Агент доступен полностью
- Каждый запрос тратит токены, пока `tokensRemaining > 0`
- Квота отображается в интерфейсе

---

## Сценарий A: Триал заканчивается по времени (токены не потрачены)

```
14 дней прошло →
  normalizeState() проверяет: trialEndsAt <= now
  → subscriptionStatus = FREE
  → planId = plan_free
  → токены обнуляются при следующем resetPeriod
```

**Что видит пользователь:**
- Рецепты AI больше не генерируются (RECIPE_GENERATION недоступен)
- AI агент в ограниченном режиме (ADVANCED_AI_TOOLS недоступен)
- Остаются: анализ питания, умный поиск продуктов
- Лимит снижается до 100 000 токенов / 100 actions в месяц

---

## Сценарий B: Токены закончились раньше срока триала

```
tokensUsed >= tokensLimit (300 000) →
  assertQuotaAvailable() выбрасывает:
  HTTP 403 { code: "AI_QUOTA_EXCEEDED" }
```

**Что видит пользователь:**
- Статус в API всё ещё `TRIAL`, `trialEndsAt` ещё не наступил
- AI заблокирован: любой запрос возвращает ошибку квоты
- Интерфейс показывает 0 оставшихся токенов

Технически это «заморозка без downgrade» — статус TRIAL сохраняется до `trialEndsAt`, но действия невозможны.

---

## Сценарий C: Пользователь оформляет подписку Pro

### 3.1 Создание checkout

```
Frontend: POST /v1/billing/checkout-session
  → BillingService.createCheckoutSession()
  → Paddle создаёт transaction с PADDLE_PRICE_ID и custom_data.userId
  → Backend возвращает URL → редирект на Paddle-hosted checkout
```

### 3.2 Оплата прошла — Paddle вебхук

```
Paddle отправляет subscription.activated (или transaction.completed)
  → BillingService.syncFromSubscription()
     User.subscriptionStatus = ACTIVE
     User.isPro = true
     User.paddleSubscriptionId = "sub_xxx"
     User.subscriptionCancelAt = null
```

**Важно:** `UserAiSubscription.subscriptionStatus` при этом **не меняется**. Он остаётся `TRIAL` (или `FREE` если уже прошёл). Это техническая особенность текущей реализации — два трека работают независимо.

На практике: если пользователь покупает Pro в период триала, `UserAiSubscription` остаётся TRIAL до его окончания (14 дней от первого входа), потом не переключится в ACTIVE сам по себе — это Gap, который потребует дополнительной синхронизации.

### 3.3 После оплаты

- `GET /v1/me/ai-usage` продолжает возвращать `subscriptionStatus: "TRIAL"` (из UserAiSubscription)
- `User.isPro = true` — backend может использовать это для прямых проверок
- Токены сбрасываются при начале нового периода (ежемесячно)

---

## Сценарий D: Оплата не прошла (PAST_DUE)

```
Paddle: оплата провалилась → subscription.past_due →
  syncFromSubscription():
    User.subscriptionStatus = PAST_DUE
    User.isPro = false
```

**Что видит пользователь (frontend):**
```
aiUsage.subscriptionStatus === "PAST_DUE"
  → SubscriptionStatusBanner: жёлтый баннер с кнопкой "Обновить способ оплаты"
  → Кнопка открывает Paddle billing portal
```

---

## Сценарий E: Отмена с запланированной датой (cancel scheduled)

Когда пользователь нажимает "Отменить подписку" в Paddle Portal, Paddle **не отменяет сразу** — он планирует отмену на конец оплаченного периода.

```
Paddle отправляет subscription.updated со scheduled_change:
  { action: "cancel", effective_at: "2026-07-08T00:00:00Z" }

syncFromSubscription():
  User.subscriptionStatus = ACTIVE  ← подписка ещё активна!
  User.subscriptionCancelAt = 2026-07-08T00:00:00Z
```

**Что видит пользователь (frontend):**
```
aiUsage.subscriptionStatus === "ACTIVE" + aiUsage.subscriptionCancelAt !== null
  → SubscriptionStatusBanner: синий info-баннер
  → "Подписка будет отменена 8 июля 2026 г."
  → Кнопка "Сохранить подписку" → Paddle portal
```

### Финальная отмена (дата наступила)

```
Paddle отправляет subscription.canceled →
  syncFromSubscription():
    User.subscriptionStatus = CANCELED
    User.isPro = false
    User.subscriptionCancelAt = null (или оставляется)
```

---

## Сценарий F: Pro → возобновление после истечения

Если пользователь отменил, но потом хочет снова:
- Переходит на `/pricing` → новый checkout session
- Проходит оплату → Paddle создаёт новую подписку
- Webhook → `User.subscriptionStatus = ACTIVE`, `User.isPro = true`

---

## Архитектура: два независимых трека

```
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│   User (Paddle billing track)   │   │  UserAiSubscription (AI quota)  │
├─────────────────────────────────┤   ├─────────────────────────────────┤
│ subscriptionStatus              │   │ subscriptionStatus              │
│   FREE / ACTIVE / PAST_DUE /    │   │   FREE / TRIAL                  │
│   CANCELED / TRIAL              │   │                                 │
│ isPro: boolean                  │   │ tokensUsed / tokensLimit        │
│ subscriptionCancelAt: DateTime? │   │ trialStartedAt / trialEndsAt   │
│ paddleSubscriptionId: string?   │   │ currentPeriodStart / End       │
└─────────────────────────────────┘   └─────────────────────────────────┘
        ↑                                         ↑
  Обновляется через                   Обновляется через
  Paddle webhooks                     normalizeState() (ленивая логика)
  (syncFromSubscription)              при каждом /v1/me/ai-usage
```

**Что использует frontend:**
- `GET /v1/me/ai-usage` → возвращает данные из `UserAiSubscription` + `subscriptionCancelAt` из `User`
- `subscriptionStatus` в ответе = `UserAiSubscription.subscriptionStatus`
- `subscriptionCancelAt` в ответе = `User.subscriptionCancelAt`

---

## Переходы статусов (UserAiSubscription)

```
[новый пользователь]
        ↓ первый /v1/me/ai-usage
     TRIAL (14 дней, 300k токенов)
        ↓
  ┌─────┴──────────────────┐
  │ время вышло            │ токены 0 (раньше срока)
  ↓                        ↓
 FREE               ← TRIAL (заморожен, AI недоступен)
  ↓ каждый месяц           ↓ после trialEndsAt
период сбрасывается       FREE
100k токенов/мес
```

---

## Переходы статусов (User.subscriptionStatus через Paddle)

```
FREE
  ↓ checkout + оплата
ACTIVE
  ↓ оплата провалилась
PAST_DUE
  ↓ оплата обновлена
ACTIVE
  ↓ отмена оформлена (конец периода)
ACTIVE (subscriptionCancelAt установлен)
  ↓ дата отмены наступила
CANCELED
  ↓ новый checkout
ACTIVE
```

---

## Что происходит на стороне API при каждом запросе

При каждом вызове `POST /v1/agent/chat` или другого AI-эндпоинта:

1. `assertQuotaAvailable(userId, feature)` → вызывает `getUsageSummary()`
2. `getUsageSummary()` → `getNormalizedState()` → **lazy check**:
   - Если `UserAiSubscription` не существует → создаётся TRIAL
   - Если TRIAL и `trialEndsAt` прошёл → переключается на FREE
   - Если `currentPeriodEnd` прошёл → сбрасывает `tokensUsed = 0`
3. Проверяет `availableFeatures` (feature запрошена есть в плане?)
4. Проверяет `tokensRemaining > 0`
5. Если проверки прошли → выполняет запрос → логирует использованные токены

---

## Где хранится информация о подписке (DB модели)

| Модель | Поле | Описание |
|--------|------|----------|
| `User` | `subscriptionStatus` | Paddle billing статус |
| `User` | `isPro` | Быстрый флаг Pro доступа |
| `User` | `subscriptionCancelAt` | Дата запланированной отмены |
| `User` | `paddleCustomerId` | ID в Paddle системе |
| `User` | `paddleSubscriptionId` | ID подписки в Paddle |
| `UserAiSubscription` | `subscriptionStatus` | AI quota статус |
| `UserAiSubscription` | `tokensUsed` / `tokensLimit` | Использование токенов |
| `UserAiSubscription` | `trialStartedAt` / `trialEndsAt` | Граница триала |
| `UserAiSubscription` | `currentPeriodStart` / `currentPeriodEnd` | Текущий расчётный период |
| `AiUsageLog` | — | Лог каждого AI запроса (модель, токены, фича) |
| `AiPlan` | — | Определение плана (free / pro) |

---

## Admin пользователи

Пользователи с `User.role = ADMIN` получают:
- Специальный `"admin"` план в ответе AI usage
- Безлимитные токены (`tokensRemaining = MAX_SAFE_INTEGER`)
- Все фичи Pro
- Без списания токенов

Проверяется через `isAdminAiUsage(usage)` на frontend.

---

## Известные технические особенности

1. **Два трека не синхронизированы автоматически.** Paddle webhook обновляет `User.subscriptionStatus`, но не `UserAiSubscription.subscriptionStatus`. Они живут независимо. Будущая задача — добавить синхронизацию при `subscription.activated`.

2. **Триал создаётся лениво.** Если пользователь зарегистрировался, но не открыл приложение — триал не начинается. Таймер триала запускается при первом `GET /v1/me/ai-usage`.

3. **Нормализация состояния — ленивая.** Проверка и переключение FREE/TRIAL происходит не по cron, а при каждом обращении к `getUsageSummary()`. Это означает, что статус меняется в момент следующего запроса, а не ровно в полночь.

4. **SubscriptionStatusBanner** использует `UserAiSubscription.subscriptionStatus` (TRIAL/FREE), но `subscriptionCancelAt` из `User`. Баннер cancellation (`ACTIVE + subscriptionCancelAt`) сработает корректно только когда Paddle вернёт `subscription.updated` с `scheduled_change`.
