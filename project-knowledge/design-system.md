# Дизайн-система Wellin

## Обзор

Дизайн-система реализована в двух файлах:
- `smart-food-plan/web-mui/src/theme/designSystem.ts` — токены и хелперы
- `smart-food-plan/web-mui/src/theme/appTheme.ts` — тема MUI

Дополнительно: папка `Wellin Design System/` в корне — содержит HTML/CSS прообраз (editorial/redesign), пока **не интегрированный** в web-mui.

---

## Цветовая палитра

### Бренд-цвета (Brand)

| Токен | Hex | Использование |
|-------|-----|---------------|
| `primary` | `#12b483` | Основной зелёный (Wellin green) |
| `primaryBright` | `#3ddc97` | Яркий зелёный (success, акцент) |
| `primaryDark` | `#0e8f6a` | Тёмный зелёный |
| `primaryInk` | `#063b2e` | Тёмно-зелёный (текст на фоне) |

### Акценты

| Токен | Hex | Использование |
|-------|-----|---------------|
| `lavender` | `#9a86bf` | Жир (макрос) / вторичный акцент |
| `teal` | `#7493c7` | Белок (макрос) / info |

### Семантические

| Токен | Hex |
|-------|-----|
| `info` | `#7493c7` |
| `success` | `#3ddc97` |
| `warning` | `#bb7c98` |
| `error` | `#ef4444` |

### Макросы (КБЖУ)

| Макрос | Dark | Light |
|--------|------|-------|
| Калории | `#4cae8e` | `#0a9d6f` |
| Белок | `#7493c7` | `#4669a2` |
| Жир | `#9a86bf` | `#725b9a` |
| Углеводы | `#bb7c98` | `#944f6f` |

### Поверхности — Dark mode (Slate)

| Токен | Hex | Роль |
|-------|-----|------|
| `backgroundDark` | `#0d1117` | Основной фон |
| `surfaceDark` | `#11161d` | Поверхности / карточки |
| `textDark` | `#e8eef5` | Основной текст |
| `textMutedDark` | `#8a97a8` | Приглушённый текст |
| `textFaintDark` | `#5d6b7d` | Слабый текст |

### Поверхности — Light mode

| Токен | Hex | Роль |
|-------|-----|------|
| `backgroundLight` | `#eef1f5` | Основной фон |
| `surfaceLight` | `#e7ecf2` | Поверхности |
| `textLight` | `#111a26` | Основной текст |
| `textMutedLight` | `#56657a` | Приглушённый текст |
| `textFaintLight` | `#93a0b2` | Слабый текст |

---

## Градиенты

| Токен | Gradient |
|-------|----------|
| `brand` | `linear-gradient(130deg,#063b2e,#0a7d5c 55%,#12b483)` |
| `ai` | `linear-gradient(140deg,#0e8f6a 0%,#12b483 48%,#46e6ab 100%)` |
| `cal` | `linear-gradient(160deg,#2c7a5f,#0b4733)` |
| `textDark` | `linear-gradient(120deg,#12b483,#3ddc97)` |
| `textLight` | `linear-gradient(120deg,#0e8f6a,#12b483)` |

---

## Радиусы

| Токен | Значение | Использование |
|-------|----------|---------------|
| `sm` | 10px | Мелкие элементы (чипы, кнопки) |
| `md` | 13px | Карточки, inputs |
| `lg` | 18px | Большие карточки |
| `xl` | 22px | Диалоги, панели |
| `2xl` | 40px | Hero-секции |
| `pill` | 999px | Пилюля (кнопки-таблетки) |

---

## Типографика

### Шрифт
**Manrope** — основной шрифт приложения.

### Шкала

| Токен | Размер | Вес | Использование |
|-------|--------|-----|---------------|
| `eyebrow` | 12px | 600, uppercase, ls 0.14em | Надзаголовки секций |
| `pageTitle` | 2.9rem–5rem | 800, ls -0.055em | Заголовки лендинга |
| `sectionTitle` | 2rem–3rem | 800, ls -0.05em | Заголовки секций |
| `subtitle` | 1rem–1.08rem | — | Подзаголовки |
| `body` | — | —, lh 1.6 | Основной текст |

---

## Spacing

| Токен | MUI units | px (при base 8px) |
|-------|-----------|-------------------|
| `xs` | 1 | 8px |
| `sm` | 1.5 | 12px |
| `md` | 2 | 16px |
| `lg` | 3 | 24px |
| `xl` | 4 | 32px |

---

## Theme-aware хелперы

Все хелперы принимают `theme: Theme` и возвращают CSS-значение.

| Хелпер | Роль |
|--------|------|
| `wellinSurface(theme)` | Основная поверхность карточки |
| `wellinSectionSurface(theme)` | Поверхность секции |
| `wellinBorder(theme)` | Цвет границы hairline |
| `wellinAccentSurface(theme)` | Акцентный фон (primary с opacity) |
| `wellinControlSurface(theme)` | Поверхность контрола (input, select) |
| `wellinCardShadow(theme)` | Тень карточки |
| `wellinHeroShadow(theme)` | Тень hero-секции |
| `wellinPrimaryTint(theme, opacity?)` | Окрашенный фон (primary tint) |
| `wellinGradientText(theme)` | Gradient text (dark/light) |
| `wellinHeroPreviewSurface(theme)` | Фон hero-preview |
| `wellinAuthBackground(theme)` | Фон страницы авторизации |
| `wellinAuthOverlay(theme)` | Overlay поверх auth-фона |
| `wellinAuthCardSurface(theme)` | Поверхность auth-карточки |
| `wellinMetric(theme)` | Цвета макросов (protein/fat/carbs/cal) |

---

## Атомарные компоненты (`shared/ui/`)

### Кнопки
- `AppButton` — основная кнопка (extends MUI Button)
- `AppLoadingButton` — кнопка с loading state
- `PageActionButton` — action кнопка на странице (обычно FAB-style)

### Формы
- `AppInput` — текстовое поле (extends MUI TextField)
- `AppSelect` — выпадающий список (extends MUI Select)

### Карточки и поверхности
- `AppContentCard` — карточка контента
- `AppSurface` — универсальная поверхность
- `AppStatCard` — статистическая карточка

### Типографика
- `AppSectionTitle` — заголовок секции
- `AppEyebrow` — надзаголовок
- `AppGradientText` — текст с brand-градиентом

### Служебные
- `AppFeedbackToast` — Toast-уведомление
- `AppErrorState` — состояние ошибки
- `ConfirmActionDialog` — диалог подтверждения
- `FilterChipRow` — строка фильтр-чипов
- `FloatingActionMenu` — плавающее меню
- `NutritionInlineSummary` — inline КБЖУ
- `PageTitle` — заголовок страницы
- `PublicSection` — секция лендинга
- `SettingsRow` — строка настроек
- `WellinLogoMark` — логотип

---

## Текущее состояние дизайн-системы

### Сильные стороны
- Единая цветовая палитра с CSS-переменными (`--wellin-*`)
- Хорошая поддержка dark/light mode через theme-aware хелперы
- Шрифт Manrope задан глобально
- Все радиусы и spacing через токены

### Проблемы и технический долг

1. **Расхождение с папкой `Wellin Design System/`**: В корне проекта есть HTML/CSS redesign с более новыми стилями, который ещё НЕ интегрирован в web-mui.

2. **Дублирование макро-цветов**: `shared/theme/macroColors.ts` и `designSystem.ts` — два места для одних цветов. Нужна единая точка.

3. **Нет Storybook**: Компоненты не задокументированы и не изолированы. Трудно понять что переиспользуется.

4. **Смешение MUI и кастомных стилей**: Часть компонентов использует `sx={{}}`, часть — `styled()`. Нет единого подхода.

5. **`AiAgentPage.tsx` и `DashboardPlaceholderPage.tsx`**: Скорее всего устаревшие, но нуждаются в проверке.

---

## Рекомендации по унификации (без редизайна)

1. Перенести все цвета макросов из `macroColors.ts` → `designSystem.ts`
2. Проверить и внедрить CSS-переменные из `Wellin Design System/` в `appTheme.ts`
3. Провести аудит использования `AppButton` vs прямого `Button` из MUI
4. Создать index-файл для `shared/ui/` для удобного импорта
