# Брендовый вопрос – question feature

## Содержание

- [1) Общая схема работы](#1-общая-схема-работы)
  - [1.1 Фронтенд (Next.js)](#11-фронтенд-nextjs)
  - [1.2 Бэкенд (Django + Wagtail)](#12-бэкенд-django--wagtail)
- [2) Получение вопроса — GET /api/v2/question/](#2-получение-вопроса--get-apiv2question)
  - [2.1 Query-параметры](#21-query-параметры)
  - [2.2 Алгоритм выборки](#22-алгоритм-выборки)
  - [2.3 Формат ответа](#23-формат-ответа)
- [3) Отправка ответа — POST /api/v2/question-answer/](#3-отправка-ответа--post-apiv2question-answer)
  - [3.1 Тело запроса](#31-тело-запроса)
  - [3.2 Валидация](#32-валидация)
  - [3.3 Формат ответа](#33-формат-ответа)
- [4) Модель QuestionAnswer](#4-модель-questionanswer)
- [5) Модель QuestionModel (Wagtail)](#5-модель-questionmodel-wagtail)
  - [5.1 Поля](#51-поля)
  - [5.2 Валидация уникальности](#52-валидация-уникальности)
- [6) Маршрутизация на фронтенде](#6-маршрутизация-на-фронтенде)
- [7) Нюансы и рекомендации](#7-нюансы-и-рекомендации)

---

Фича «брендовый вопрос» позволяет показывать пользователю вопросы с вариантами ответов в зависимости от его уровня и типа активности. Ответы сохраняются в базе и отображаются в Django admin.

---

## 1) Общая схема работы

### 1.1 Фронтенд (Next.js)

1. Страница `/question`, `/question/code` или `/question/content` монтируется через один файл `app/(app)/question/[[...slug]]/page.tsx`.
2. Slug из URL передаётся как `activity_type` в query-параметр запроса к бэкенду.
3. После ответа пользователя хук `useQuestionForm` отправляет POST-запрос с данными ответа, затем — вне зависимости от результата — переходит на `GET_PRIZE`.

### 1.2 Бэкенд (Django + Wagtail)

1. `QuestionViewSet.listing_view` выбирает вопрос согласно алгоритму выборки.
2. `QuestionAnswerView.post` принимает ответ и сохраняет запись `QuestionAnswer`.
3. Записи доступны в Django admin (readonly).

---

## 2) Получение вопроса — `GET /api/v2/question/`

### 2.1 Query-параметры

| Параметр | Тип | Описание |
|---|---|---|
| `activity_type` | string | Необязательный. Допустимые значения: `code`, `content`, `challenge`. Если не передан — выборка без фильтра по типу. |

### 2.2 Алгоритм выборки

Выборка построена как **каскад расширяющихся выборок**: начинаем с самой узкой (точно под пользователя) и, если она пуста, последовательно ослабляем критерии — сначала уровень, потом тип активности, в самом конце разрешаем показывать уже отвеченные вопросы.

Порядок ослабления осей: **`user_level` → `activity_type` → `answered`**.

Базовый набор на всех шагах — живые вопросы бренда пользователя (`QuestionsIndex` → `child_of`). Если на этом базовом наборе вопросов нет — возвращается `{}` со статусом `200`.

Шаги каскада:

1. **Свой уровень + свой `activity_type` + без ответа.** Если выборка непуста — берём первый по `step_number ASC`.
2. **Уровень и ниже + свой `activity_type` + без ответа.** Если непуста — берём первый по `step_number ASC`.
3. **Уровень и ниже + любой `activity_type` + без ответа.** Если непуста — берём первый по `step_number ASC`.
4. **Уровень и ниже + любой `activity_type` + с ответами тоже.** Если непуста — берём **случайный** вопрос.
5. **Fallback: любой вопрос бренда.** Берём случайный.

> **Стратегия выбора внутри шага.** Пока в каскаде есть шаг, где нашлись неотвеченные вопросы (шаги 1–3), вопрос берётся детерминированно — первый по `step_number ASC`. Как только неотвеченных не осталось и мы вынуждены показывать уже отвеченные (шаги 4–5) — вопрос выбирается случайно, чтобы пользователь не упирался в один и тот же.

> **Порядок уровней** задан перечислением в `UserLevel` (по возрастанию: `silver` → `gold` → `platinum`). «Уровень и ниже» для `gold` означает `{silver, gold}`.

> **Пропуск шагов.** Если у пользователя не определён уровень — шаги 1–4 пропускаются, выбор сразу из шага 5. Если `activity_type` не передан или невалиден — пропускаются шаги 1 и 2, каскад начинается с шага 3.

### 2.3 Формат ответа

```json
{
  "id": 171,
  "title": "Название вопроса",
  "brand": "LM",
  "brand_title": "IQOS",
  "content": [
    "<p data-block-key=\"f1z7y\">Текст вопроса</p>"
  ],
  "items": "<fieldset>...</fieldset>",
  "show_header": true,
  "background_model": {},
  "index": {},
  "json": {
    "activity_type": "code",
    "id": 171,
    "content": ["Текст вопроса"],
    "items": [
      { "id": "answer_171_0", "value": "yes" },
      { "id": "answer_171_1", "value": "no" }
    ]
  }
}
```

Поле `json` предназначено содержит машиночитаемое представление вопроса и вариантов ответа.

---

## 3) Отправка ответа — `POST /api/v2/question-answer/`

### 3.1 Тело запроса

| Поле | Тип | Обязательное | Описание |
|---|---|---|---|
| `question_id` | integer | ✅ | Wagtail page id вопроса |
| `answer_id` | string | ⚠️ optional* | ID варианта ответа, например `answer_171_0` |
| `answer_text` | string | ⚠️ optional* | Текстовый ответ (для textarea) |
| `question_text` | string | — | Текст вопроса для отображения в admin |

\* Хотя бы одно из полей `answer_id` или `answer_text` должно быть заполнено.

`user_id`, `brand_id`, `game_id` извлекаются из токена на стороне бэкенда — передавать их в теле не нужно.

### 3.2 Валидация

- Если отсутствует `user_id` или `question_id` → `400 Missing fields`
- Если оба `answer_id` и `answer_text` пусты → `400 Missing fields`
- Если пользователь не найден → `404 user not found`
- Если ошибка при сохранении → `500 failed to save answer`

### 3.3 Формат ответа

```json
// Успех
{ "status": "success", "detail": null, "message": "Answer submitted successfully" }

// Ошибка
{ "status": "error", "detail": "...", "message": "..." }
```

> После успешной записи ответа фронтенд уходит на `GET_PRIZE`.

---

## 4) Модель `QuestionAnswer`

| Поле | Тип | Описание |
|---|---|---|
| `user` | FK → `goodluck.User` | Пользователь |
| `brand` | FK → `goodluck.Brand` | Бренд пользователя на момент ответа |
| `game_id` | CharField(255) | ID игры из токена |
| `question_id` | IntegerField | Wagtail page id вопроса (без FK) |
| `question_text` | TextField(1000), nullable | Текст вопроса для отображения в admin |
| `answer_id` | CharField(255), nullable | ID выбранного варианта, например `answer_171_0` |
| `answer_text` | TextField, nullable | Текстовый ответ |
| `created_at` | DateTimeField | Время создания (auto) |

> Связь с таблицами Wagtail (`QuestionModel`) намеренно не задана через FK — вопросы могут удаляться или переводиться, а история ответов должна сохраняться.

---

## 5) Модель `QuestionModel` (Wagtail)

### 5.1 Поля

| Поле | Тип | Описание |
|---|---|---|
| `brand` | FK → Brand | Устанавливается автоматически из родительского `QuestionsIndex` при сохранении |
| `brand_title` | CharField(255), nullable | Отображаемое название бренда |
| `content` | StreamField | Тело вопроса. Блоки: `rich_text`, `video`, `image` |
| `user_level` | CharField, choices=UserLevel | Уровень пользователя для которого предназначен вопрос |
| `activity_type` | CharField, choices=ActivityType, nullable | Тип активности: `code`, `content`, `challenge` |
| `step_number` | PositiveIntegerField, nullable | Порядковый номер вопроса внутри комбинации `user_level + activity_type` |
| `items` | StreamField (AnswerOptionBlock) | Варианты ответа |
| `show_header` | BooleanField | Показывать ли шапку |
| `background` | FK → Image, nullable | Фоновое изображение |

### 5.2 Валидация уникальности

Метод `clean()` запрещает создание двух вопросов с одинаковой комбинацией:

- `locale`
- `user_level`
- `activity_type`
- `step_number`

> `unique_together` не используется — он не работает корректно с `NULL`-значениями. Защита реализована только через `clean()`.

Пример допустимых комбинаций:

| user_level | activity_type | step_number | Допустимо? |
|---|---|---|---|
| silver | code | 1 | ✅ |
| silver | content | 1 | ✅ другой activity_type |
| gold | code | 1 | ✅ другой user_level |
| silver | code | 1 | ❌ дубликат |

---

## 6) Маршрутизация на фронтенде

Один файл покрывает три URL через optional catch-all сегмент:

```
app/
  (app)/
    question/
      [[...slug]]/
        page.tsx
```

| URL | `slug` | Query к бэкенду |
|---|---|---|
| `/question` | `undefined` | `/api/v2/question/` |
| `/question/code` | `['code']` | `/api/v2/question/?activity_type=code` |
| `/question/content` | `['content']` | `/api/v2/question/?activity_type=content` |

> В Next.js 15 `params` асинхронный — необходимо `await params` перед обращением к `slug`.

```tsx
type Props = { params: Promise<{ slug?: string[] }> }

export default async function QuestionPage({ params }: Props) {
    const { slug } = await params
    const section = slug?.[0] // 'code' | 'content' | undefined
    return <QuestionWidget section={section} />
}
```

---

## 7) Нюансы и рекомендации

### 7.1 Порядок вопросов определяется `step_number`

Поле `step_number` задаётся вручную в Wagtail admin.

### 7.2 История ответов не привязана к уровню

`QuestionAnswer` хранит только `question_id`. Фильтр «без ответа» в шагах 1–3 каскада работает по всей истории пользователя — каждый вопрос пользователь видит не более одного раза, пока в каскаде есть шаг с неотвеченными вопросами.

### 7.3 Смена уровня пользователя

При смене `user_level` (например Silver → Gold) пользователь начинает получать вопросы нового уровня с `step_number=1`. История ответов на вопросы предыдущего уровня не влияет на новую выборку, так как вопросы разных уровней имеют разные `id`.

### 7.4 Все вопросы отвечены — случайный из отвеченных

Если во всех шагах с фильтром «без ответа» (1–3) выборка пуста — каскад переходит к шагу 4, где отвеченные уже не исключаются. На этом этапе вопрос выбирается случайно, а не по `step_number`, чтобы пользователь не получал один и тот же первый вопрос раз за разом.

### 7.5 Фронтенд уходит со страницы не всегда

Ошибка POST-запроса с ответом блокирует переход на `GET_PRIZE`

### 7.6 `answer_id` — строка, не число

Формат: `answer_{question_id}_{index}`, например `answer_171_0`. Генерируется на бэкенде в `AnswerOptionItemSerializer` и передаётся фронтенду как `value` radio-инпута.
