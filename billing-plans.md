# Wellin — Тарифные планы: полный анализ

## Резюме

Реально работающий гейтинг сейчас: **только квота токенов**. Все AI-функции доступны на бесплатном плане — разница между Free и Pro это количество AI-запросов в месяц.

---

## Планы (из `ai-access.constants.ts` + `.env`)

| Параметр | Free | Trial | Pro (ACTIVE) |
|---|---|---|---|
| Статус | `FREE` | `TRIAL` | `ACTIVE` |
| Токенов/мес | 100 000 | 300 000 (разово) | 1 000 000 |
| AI actions/мес | ≈100 | ≈300 | ≈1 000 |
| Цена | $0 | $0 | $9.99/мес |
| Срок | бессрочно | 7 дней | ежемесячно |

> **Новый пользователь автоматически получает Trial на 7 дней** (не Free). После окончания триала переходит на Free.
> Примечание: `AI_TRIAL_DAYS=7` нужно прописать в `.env.yaml` (прод) — иначе будет применяться дефолт из кода (14 дней).

---

## Что реально проверяется (`ensureCanExecute`)

Только два места в коде вызывают проверку квоты:

### 1. `agent.service.ts` — AI-агент (`POST /v1/agent/chat`)
```typescript
await this.aiUsageService.ensureCanExecute(userId, {
  feature: AiFeature.ADVANCED_AI_TOOLS,  // ← всегда это
  actionType: "agent.chat",
  model,
});
```

### 2. `ai.service.ts` — OpenAI Responses proxy (`POST /v1/ai/responses`)
```typescript
const feature = dto.feature ?? AiFeature.ADVANCED_AI_TOOLS;  // ← дефолт: ADVANCED_AI_TOOLS
await this.aiUsageService.ensureCanExecute(userId, {
  feature,
  actionType: "responses.create",
  model,
});
```

Клиент передаёт `feature: "MEAL_ANALYSIS"` — это всё тот же `AiFeature.MEAL_ANALYSIS`, но гейтинг идёт через ту же систему квот.

---

## Что НЕ проверяется (мёртвый код)

| Feature | Определён | Проверяется |
|---|---|---|
| `ADVANCED_AI_TOOLS` | ✅ | ✅ (`agent.service`) |
| `MEAL_ANALYSIS` | ✅ | ✅ (`ai.service`, только для proxy endpoint) |
| `RECIPE_GENERATION` | ✅ | ❌ нигде не проверяется |
| `SMART_PRODUCT_MATCH` | ✅ | ❌ нигде не проверяется |

`RECIPE_GENERATION` и `SMART_PRODUCT_MATCH` определены в плане, но ни один `assertFeatureAccess`/`assertQuotaAvailable` для них не вызывается в рабочем коде.

---

## Фичи в планах (из `ensureDefaultPlans`)

```typescript
// Free plan
features: [MEAL_ANALYSIS, SMART_PRODUCT_MATCH, ADVANCED_AI_TOOLS]

// Pro plan  
features: [RECIPE_GENERATION, MEAL_ANALYSIS, SMART_PRODUCT_MATCH, ADVANCED_AI_TOOLS]
```

Отличие: Pro дополнительно имеет `RECIPE_GENERATION` — но эта фича нигде не гейтится в коде, поэтому Free и Pro пользователи имеют доступ к одинаковому набору AI-функций.

---

## Реальная разница Free vs Pro сейчас

| | Free | Pro |
|---|---|---|
| AI-агент (chat) | ✅ (100 req/мес) | ✅ (1000 req/мес) |
| AI-анализ питания | ✅ (из квоты) | ✅ (из квоты) |
| AI-генерация рецептов | ✅ (из квоты) | ✅ (из квоты) |
| Квота токенов | 100K/мес | 1M/мес |
| **Разница** | — | **10× больше запросов** |

---

## Жизненный цикл подписки

```
Регистрация
    ↓
TRIAL (7 дней, 300K токенов, все Pro-фичи)
    ↓ по истечении автоматически
FREE ($0, 100K токенов/мес)
    ↓ при оформлении подписки Paddle
ACTIVE / Pro ($9.99, 1M токенов/мес)
    ↓ при отмене
FREE (до конца оплаченного периода → Free)
```

Переходы управляются в `ai-subscriptions.service.ts` → `normalizeState()`:
- TRIAL → FREE если `trialEndsAt <= now`
- Paddle вебхук (`billing/`) вызывает `setSubscriptionStatus(userId, { subscriptionStatus: "ACTIVE" | "FREE" | ... })`

---

## Что нужно сделать (технический долг)

1. **Активировать гейтинг `RECIPE_GENERATION`** — добавить `assertFeatureAccess` перед вызовом генерации рецептов в агенте, если хотим сделать это Pro-only функцией.
2. **Удалить или активировать `SMART_PRODUCT_MATCH`** — сейчас мёртвый код.
3. **Синхронизировать маркетинг с реальностью** — на лендинге не обещать гейтинг который не работает.

---

## Файлы

| Файл | Роль |
|---|---|
| `foodieai/src/ai-access/ai-access.constants.ts` | Константы: фичи, дефолтные лимиты |
| `foodieai/src/ai-access/ai-subscriptions.service.ts` | Логика планов, квот, нормализации состояния |
| `foodieai/src/ai-access/ai-usage.service.ts` | `ensureCanExecute`, `recordUsage` |
| `foodieai/src/agent/agent.service.ts` | Использует `ADVANCED_AI_TOOLS` для AI-агента |
| `foodieai/src/ai/ai.service.ts` | Использует `dto.feature` для OpenAI proxy |
| `foodieai/src/billing/` | Paddle вебхуки → `setSubscriptionStatus` |
| `foodieai/.env` / `.env.yaml` | Квоты (`AI_FREE_PLAN_*`, `AI_ACTIVE_PLAN_*`, `AI_TRIAL_DAYS=7`) |
