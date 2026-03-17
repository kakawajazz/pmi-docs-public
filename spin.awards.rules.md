# Настройка вознаграждений за коды – spin.awards.rules

- [1) Как сервис принимает решение](#1-как-сервис-принимает-решение)
- [2) Формат конфига `spin.awards.rules`](#2-формат-конфига-spinawardsrules)
- [3) Правило (Rule)](#3-правило-rule)
  - [3.1 `minCodes` (обязательное)](#31-mincodes-обязательное)
  - [3.2 `spin` (опциональное)](#32-spin-опциональное)
  - [3.3 `guarantee` (условно обязательное)](#33-guarantee-условно-обязательное)
- [4) Валидации (когда сервис бросит `SpinConfigError`)](#4-валидации-когда-сервис-бросит-spinconfigerror)
- [5) Примеры конфигурации](#5-примеры-конфигурации)
  - [5.1 LM: первые 3 спина особые, дальше default](#51-lm-первые-3-спина-особые-дальше-default)
  - [5.2 MRL: первые 6 спинов по 1 коду, дальше default](#52-mrl-первые-6-спинов-по-1-коду-дальше-default)
- [6) Примеры расчёта доступности](#6-примеры-расчёта-доступности)
- [7) Нюансы и рекомендации](#7-нюансы-и-рекомендации)
  - [7.1 Ключи overrides — это номера спинов и должны быть строками](#71-ключи-overrides--это-номера-спинов-и-должны-быть-строками)
  - [7.2 Номер следующего спина берётся из таблицы `Spin`](#72-номер-следующего-спина-берётся-из-таблицы-spin)
  - [7.3 Конфиг привязан к бренду пользователя и активностям этого бренда](#73-конфиг-привязан-к-бренду-пользователя-и-активностям-этого-бренда)
  - [7.4 Поведение, если бренда нет в конфиге](#74-поведение-если-бренда-нет-в-конфиге)
  - [7.5 Если `spin` не указан](#75-если-spin-не-указан)
- [Дефолтное значение конфига](#дефолтное-значение-конфига)

Конфиг `spin.awards.rules` управляет правилами доступности и типа спина на основе кодов пользователя. Он используется сервисом `SpinEligibilityService` **до выполнения спина**: сервис определяет, доступен ли спин сейчас, какой он будет, и какие коды нужно списать.

---

## 1) Как сервис принимает решение

Сервис анализирует:

- `Spin` пользователя — чтобы определить **номер следующего спина**
- `Activity` пользователя — чтобы определить, хватает ли кодов и какие именно коды будут списаны

### Какие активности считаются кодами

Код — это `Activity` с:

- `type = ActivityType.CODE`
- `brand__code = brand_code` текущего бренда пользователя

### Активные коды

Активными считаются коды с:

- `is_used = False`

### Порядок списания

Списание идёт по FIFO:

- списываются **самые старые активные** коды
- порядок: `created_at ASC`

### Номер следующего спина (N)

Сервис выбирает правило по **номеру следующего спина**.

Номер считается так:

- берётся количество уже существующих `Spin` пользователя по текущему бренду
- `N = count(spins) + 1`

Далее правило выбирается так:

- если существует `overrides[str(N)]` — берётся оно
- иначе берётся `default`

### Доступность спина

- `active_count` = количество активных кодов (`is_used=False`)
- спин доступен, если: `active_count >= minCodes`

Если спин доступен, сервис возвращает `activity_ids_to_consume` — список `Activity.id`, которые должны быть списаны при выполнении спина:

- берутся первые активные коды в количестве `minCodes`
- порядок — `created_at ASC`

### Guaranteed-спин

Если у выбранного правила:

- `spin = "guaranteed"`

то сервис дополнительно пытается найти `Balance` по:

- `brand = user.brand`
- `code = guarantee`

Если подходящий баланс не найден или в нём закончились доступные выдачи (`number - claims_number <= 0`), сервис возвращает `guarantee_id = 0`.

---

## 2) Формат конфига `spin.awards.rules`

`spin.awards.rules` — словарь, ключом которого является **код бренда** (например, `MRL`, `LM`, `PRL_NPL`).

Пример верхнего уровня:

```json
{
  "MRL": { "...": "..." },
  "LM":  { "...": "..." }
}
```

Каждый бренд содержит:

- `default` — правило по умолчанию
- `overrides` — специальные правила для **конкретных номеров спина** (`"1"`, `"2"`, `"3"`, ...)

---

## 3) Правило (Rule)

Правило — объект с полями:

### 3.1 `minCodes` (обязательное)

**Тип:** integer  
**Ограничение:** `>= 1`

Сколько активных кодов нужно, чтобы спин стал доступен, и сколько кодов будет списано при его выполнении.

### 3.2 `spin` (опциональное)

**Тип:** string  
**Допустимые значения:** значения `SpinType`:

- `"full"`
- `"not_full"`
- `"guaranteed"`

Если `spin` не задан, сервис возвращает `spin = None`, и дальше тип спина выбирается остальной логикой приложения.

### 3.3 `guarantee` (условно обязательное)

**Тип:** string  
**Смысл:** `Balance.code` (например, `"welcome.500"`)

Используется **только** когда `spin = "guaranteed"`.

---

## 4) Валидации (когда сервис бросит `SpinConfigError`)

Сервис валидирует правило при вычислении:

1. `minCodes < 1`  
   → ошибка: `minCodes must be >= 1`

2. `spin == "guaranteed"` и `guarantee` отсутствует  
   → ошибка: `guarantee is required for guaranteed spin`

3. `spin != "guaranteed"` и `guarantee` присутствует  
   → ошибка: `guarantee must be empty for non-guaranteed spin`

4. У пользователя нет бренда  
   → ошибка: `User has no brand`

5. Для бренда нет конфига  
   → ошибка: `No config for brand=<brand_code>`

6. Для guaranteed-спина не передан `balance_code` или у пользователя нет `brand_id`  
   → ошибка на этапе получения `guarantee_id`

---

## 5) Примеры конфигурации

### 5.1 LM: первые 3 спина особые, дальше default

```json
{
  "LM": {
    "default": {
      "minCodes": 2
    },
    "overrides": {
      "1": {
        "spin": "guaranteed",
        "minCodes": 1,
        "guarantee": "welcome.500"
      },
      "2": {
        "spin": "guaranteed",
        "minCodes": 2,
        "guarantee": "welcome.1000"
      },
      "3": {
        "spin": "guaranteed",
        "minCodes": 2,
        "guarantee": "welcome.1000"
      }
    }
  }
}
```

Интерпретация:

- 1-й спин: 1 код, guaranteed, `welcome.500`
- 2-й спин: 2 кода, guaranteed, `welcome.1000`
- 3-й спин: 2 кода, guaranteed, `welcome.1000`
- 4-й и все последующие: `default` — 2 кода, тип спина не задан в конфиге

### 5.2 MRL: первые 6 спинов по 1 коду, дальше default

```json
{
  "MRL": {
    "default": {
      "minCodes": 2
    },
    "overrides": {
      "1": {
        "spin": "guaranteed",
        "minCodes": 1,
        "guarantee": "welcome.500"
      },
      "2": {
        "minCodes": 1
      },
      "3": {
        "spin": "guaranteed",
        "minCodes": 1,
        "guarantee": "welcome.1000"
      },
      "4": {
        "minCodes": 1
      },
      "5": {
        "spin": "guaranteed",
        "minCodes": 1,
        "guarantee": "welcome.1000"
      },
      "6": {
        "minCodes": 1
      }
    }
  }
}
```

Интерпретация:

- 1-й спин: 1 код, guaranteed, `welcome.500`
- 2-й спин: 1 код, тип спина не задан
- 3-й спин: 1 код, guaranteed, `welcome.1000`
- 4-й спин: 1 код, тип спина не задан
- 5-й спин: 1 код, guaranteed, `welcome.1000`
- 6-й спин: 1 код, тип спина не задан
- 7-й и все последующие: `default` — 2 кода

---

## 6) Примеры расчёта доступности

### Пример 1: LM, второй спин

Предположим:

- у пользователя уже есть 1 `Spin` по бренду `LM`
- значит следующий номер спина: `N = 2`
- по бренду найдено правило `overrides["2"]`
- у этого правила `minCodes = 2`

Если активных кодов сейчас:

- `active_count = 1` → спин **недоступен**
- `active_count = 2` → спин **доступен**, будут списаны 2 самых старых активных кода

### Пример 2: MRL, седьмой спин

Предположим:

- у пользователя уже есть 6 `Spin` по бренду `MRL`
- значит следующий номер спина: `N = 7`
- в `overrides` правила `"7"` нет
- значит берётся `default`
- у `default` `minCodes = 2`

Если активных кодов сейчас:

- `active_count = 1` → спин **недоступен**
- `active_count = 2` → спин **доступен**

---

## 7) Нюансы и рекомендации

### 7.1 Ключи overrides — это номера спинов и должны быть строками

Сервис ищет правило так:

```python
overrides.get(str(N), default)
```

Поэтому ключи в `overrides` — это строки:

- `"1"`
- `"2"`
- `"3"`

Их смысл — **номер спина**, а не номер кода и не количество уже списанных кодов.

### 7.2 Номер следующего спина берётся из таблицы `Spin`

Сервис считает номер спина так:

- `Spin.objects.filter(user=user, brand__code=brand_code).count() + 1`

### 7.3 Конфиг привязан к бренду пользователя и активностям этого бренда

Сервис:

- получает `brand_code` из `user.brand`
- выбирает конфиг по `brand_code`
- ищет активности по `Activity.brand__code = brand_code`
- считает номер спина по `Spin.brand__code = brand_code`

Поэтому важно, чтобы:

- у пользователя был корректный `brand`
- у `Activity` и `Spin` был корректный бренд
- код бренда совпадал с ключом в конфиге

### 7.4 Поведение, если бренда нет в конфиге

Если для бренда отсутствует запись в `spin.awards.rules`, сервис:

- логирует `No config for brand=<brand_code>`
- кидает `SpinConfigError`

### 7.5 Если `spin` не указан

Если в правиле нет поля `spin`, сервис:

- возвращает `rule.spin = None`
- не требует `guarantee`
- не вычисляет `guarantee_id`

Это означает, что конфиг определяет только требуемое число кодов, а конкретный тип спина выбирается дальше в другом месте приложения.

---

## Дефолтное значение конфига

<details>

<summary>Показать</summary>

<pre>
{
    "MRL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            }
        }
    },
    "MRL_NPL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            }
        }
    },
    "LM": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            }
        }
    },
    "LM_NPL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            }
        }
    },
    "PRL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1500"
            }
        }
    },
    "PRL_NPL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "2": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1500"
            }
        }
    }
}
</pre>
</details>
