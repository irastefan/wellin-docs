# Загрузка и хранение изображений

## Обзор

Изображения хранятся в Google Cloud Storage (GCS). Бакет настроен с:
- **Uniform Bucket-Level Access** — запрещает ACL на уровне объектов
- **Public Access Prevention** — блокирует публичный доступ через IAM (`allUsers`)

Из-за этих ограничений изображения раздаются **через streaming-прокси на бэкенде**, а не по прямым публичным URL.

---

## Два типа изображений

| Тип | Префикс в GCS | Логика хранения | Как раздаётся |
|-----|--------------|-----------------|---------------|
| Фото рецептов | `recipe-photos/` | `Recipe.photoUrl` в БД (objectKey) | `GET /v1/recipes/:id/photo` |
| Изображения AI-агента | `ai-input-images/` | Не хранятся в БД | Signed URL (живёт 15 мин), передаётся в OpenAI |

---

## Путь объекта в GCS

```
{prefix}/{userId}/{YYYY}/{MM}/{DD}/{uuid}.{ext}
```

Пример: `recipe-photos/user_abc123/2026/06/06/f47ac10b.jpg`

---

## API

### `POST /v1/ai/uploads/recipe-photo`

Загружает фото рецепта в GCS.

- **Auth:** Bearer (обязательно)
- **Content-Type:** `multipart/form-data`, поле `file`
- **Форматы:** JPEG, PNG, WEBP, GIF; максимум `GCS_UPLOAD_MAX_FILE_BYTES` (по умолчанию 10 МБ)

Ответ:
```json
{
  "objectKey": "recipe-photos/user_abc/2026/06/06/uuid.jpg",
  "imageUrl": "https://storage.googleapis.com/...",
  "expiresAt": null,
  "contentType": "image/jpeg",
  "size": 198439
}
```

> **Важно:** `imageUrl` в ответе не работает (Public Access Prevention). Фронтенд использует только `objectKey` — сохраняет его в рецепт и отображает через прокси-endpoint.

---

### `POST /v1/ai/uploads/image`

Загружает изображение для анализа AI-агентом (OpenAI Responses API).

- **Auth:** Bearer (обязательно)
- **Content-Type:** `multipart/form-data`, поле `file`

Возвращает **signed URL** со временем жизни `GCS_UPLOAD_URL_TTL_SECONDS` (по умолчанию 15 минут). URL передаётся напрямую в OpenAI — **не сохранять в БД**.

---

### `GET /v1/recipes/:recipeId/photo`

Стримит фото рецепта из GCS. Авторизация не нужна.

- Читает `Recipe.photoUrl` (objectKey) из БД
- Стримит объект через `bucket.file(objectKey).createReadStream()`
- Заголовок: `Cache-Control: public, max-age=86400` (кэш 24 ч в браузере)
- 404, если у рецепта нет фото

Фронтенд строит URL с cache-busting:
```
/api/v1/recipes/{id}/photo?v={recipe.updatedAt}
```
При обновлении рецепта `updatedAt` меняется → браузер запрашивает свежую версию.

---

### `POST /v1/recipes/photos/cleanup`

Удаляет неиспользуемые изображения из GCS. Обрабатывает оба типа за один вызов.

- **Auth:** Bearer (обязательно)
- **Query params:**
  - `dryRun=true` — показать что будет удалено, без фактического удаления
  - `aiMaxAgeMinutes=15` — порог возраста для AI-изображений (по умолчанию 15)

**Логика по каждому типу:**

| Тип | Критерий удаления |
|-----|--------------------|
| Фото рецептов (`recipe-photos/`) | objectKey не найден ни в одном `Recipe.photoUrl` в БД |
| AI-изображения (`ai-input-images/`) | `timeCreated` в GCS старше `aiMaxAgeMinutes` минут |

Когда появляются «осиротевшие» фото рецептов:
- Пользователь загрузил фото, но не сохранил рецепт
- Пользователь заменил фото (старый objectKey убран из БД, объект в GCS остался)
- Рецепт удалён (каскад не трогает GCS)

Ответ:
```json
{
  "dryRun": false,
  "totalDeletedCount": 5,
  "recipePhotos": {
    "scanned": 12,
    "referencedInDb": 9,
    "orphanedCount": 2,
    "orphanedKeys": ["recipe-photos/user/2026/06/04/old.jpg"],
    "deletedCount": 2
  },
  "aiImages": {
    "scanned": 8,
    "maxAgeMinutes": 15,
    "orphanedCount": 3,
    "orphanedKeys": ["ai-input-images/user/2026/06/06/uuid.jpg"],
    "deletedCount": 3
  }
}
```

---

## Переменные окружения

| Переменная | Обязательна | По умолчанию | Описание |
|---|---|---|---|
| `GCS_UPLOAD_BUCKET` | ✅ | — | Имя GCS-бакета |
| `GOOGLE_CLOUD_PROJECT` | — | из ADC | ID проекта GCP |
| `GCS_UPLOAD_MAX_FILE_BYTES` | — | `10485760` (10 МБ) | Максимальный размер файла |
| `GCS_UPLOAD_URL_TTL_SECONDS` | — | `900` (15 мин) | Время жизни signed URL для AI-изображений |
| `GCS_UPLOAD_PATH_PREFIX` | — | `ai-input-images` | Префикс для AI-загрузок |
| `GCS_UPLOAD_PUBLIC` | — | `false` | Если `true` — возвращать публичный URL (требует IAM `allUsers`) |

---

## Почему не публичные URL

В production-бакете на уровне организации GCP включён **Public Access Prevention**. Это блокирует:
- Per-object ACL (`public: true` при загрузке) — ошибка «uniform bucket-level access is enabled»
- IAM `allUsers objectViewer` — ошибка «public access prevention is enforced»
- v4 Signed URLs с личными ADC-кредами (gcloud auth application-default login) — нельзя подписать без сервис-аккаунта с `iam.serviceAccounts.signBlob`

**Выбранное решение:** backend streaming proxy — работает с любыми credentials, браузер кэширует 24 ч.

---

## Интеграция на фронтенде

### Загрузка (`RecipeForm.tsx`)
1. Пользователь выбирает файл → `URL.createObjectURL(file)` сразу показывает превью
2. `POST /v1/ai/uploads/recipe-photo` → получаем `objectKey`
3. `objectKey` сохраняется в `RecipeFormValues.photoUrl`

### Сохранение (`recipesApi.ts → toPayload`)

| `photoUrl` в форме | Что отправляется бэкенду | Поведение |
|---|---|---|
| `undefined` | поле не отправляется | бэкенд сохраняет текущее фото |
| `null` | `photoUrl: null` | бэкенд очищает фото |
| `"recipe-photos/..."` | `photoUrl: "recipe-photos/..."` | бэкенд сохраняет новый objectKey |

### Отображение (`mapSummary`)
```typescript
photoUrl: recipe.photoUrl
  ? `${getApiBaseUrl()}/v1/recipes/${recipe.id}/photo?v=${recipe.updatedAt}`
  : undefined
```

### Форма редактирования (`RecipeEditorPage`)
- `recipe.photoUrl` (прокси-URL) передаётся как `existingPhotoUrl` — только для отображения, не для отправки
- `RecipeFormValues.photoUrl` начинается как `undefined` — устанавливается только при загрузке нового файла
