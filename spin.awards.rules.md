# Настройка вознаграждений за коды – spin.awards.rules

- [Как сервис принимает решение](#1-как-сервис-принимает-решение)
- [2) Формат конфига `spin.awards.rules`](#2-формат-конфига-spinawardsrules)
- [3) Правило (Rule)](#3-правило-rule)
    - [3.1 `minCodes` (обязательное)](#31-mincodes-обязательное)
    - [3.2 `spin` (опциональное)](#32-spin-опциональное)
    - [3.3 `guarantee` (условно обязательное)](#33-guarantee-условно-обязательное)
- [4) Валидации (когда сервис бросит `SpinConfigError`)](#4-валидации-когда-сервис-бросит-spinconfigerror)
- [5) Пример конфигурации](#5-пример-конфигурации)
- [6) Пример расчёта доступности](#6-пример-расчёта-доступности)
- [7) Нюансы и рекомендации](#7-нюансы-и-рекомендации)
    - [7.1 Ключи overrides должны быть строками](#71-ключи-overrides-должны-быть-строками)
    - [7.2 Конфиг привязан к `Activity.brand__code`](#72-конфиг-привязан-к-activitybrand__code)
    - [7.3 Поведение, если бренда нет в конфиге](#73-поведение-если-бренда-нет-в-конфиге)
- [Дефолтное значение конфига](#дефолтное-значение-конфига)

Конфиг `spin.awards.rules` управляет правилами доступности и типа спина на основе введённых пользователем кодов. Используется сервисом `SpinEligibilityService` **до выполнения спина**: сервис определяет, доступен ли спин сейчас, какой он будет, и какие коды нужно списать.

---

## 1) Как сервис принимает решение

Сервис анализирует `Activity` пользователя:

- **код** — это `Activity` с типом `code`
- **активные коды**: `is_used = False`
- **использованные коды**: `is_used = True`
- **порядок списания**: FIFO — списываются **самые старые активные** коды (по `created_at ASC`)


### Номер правила (N)

Сервис выбирает правило по номеру:

- `used_count` = количество использованных кодов у пользователя (`is_used=True`)
- `N = used_count + 1` - номер override из конфига

Далее правило выбирается сервисом так:

- если существует `overrides[str(N)]` — берётся оно
- иначе берётся `default`

### Доступность спина

- `active_count` = количество активных кодов (`is_used=False`)
- спин доступен, если: `active_count >= minCodes`

Если доступен, сервис возвращает `activity_ids_to_consume` — список `Activity.id`, которые должны быть списаны при выполнении спина:

- берутся первые активные коды в количестве `minCodes` в порядке `created_at ASC`

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
- `overrides` — правила-исключения по номеру `N` (строковый ключ `"1"`, `"2"`, ...)

---

## 3) Правило (Rule)

Правило — объект с полями:

### 3.1 `minCodes` (обязательное)

**Тип:** integer  
**Ограничение:** `>= 1`

Сколько активных кодов нужно, чтобы спин стал доступен.

### 3.2 `spin` (опциональное)

**Тип:** string  
**Допустимые значения:** значения `SpinType`:

- `"full"`
- `"not_full"`
- `"guaranteed"`

Если `spin` не задан — тип спина будет выбран автоматически - full или not_full.

### 3.3 `guarantee` (условно обязательное)

**Тип:** string  
**Смысл:** `Balance.code` (например, `"welcome.500"`)

Используется **только** когда `spin="guaranteed"`.

---

## 4) Валидации (когда сервис бросит `SpinConfigError`)

Сервис валидирует правило при вычислении:

1) `minCodes < 1`  
→ ошибка: `minCodes must be >= 1`

2) `spin == "guaranteed"` и `guarantee` отсутствует  
→ ошибка: `guarantee is required for guaranteed spin`

3) `spin != "guaranteed"` и `guarantee` присутствует  
→ ошибка: `guarantee must be empty for non-guaranteed spin`

---

## 5) Пример конфигурации

```json
{
  "LM": {
    "default": { "minCodes": 2 },
    "overrides": {
      "1": {
        "minCodes": 1,
        "spin": "guaranteed",
        "guarantee": "welcome.500"
      },
      "2": {
        "minCodes": 1,
        "spin": "full"
      },
      "3": {
        "minCodes": 1,
        "spin": "guaranteed",
        "guarantee": "welcome.1000"
      },
      "4": {
        "minCodes": 1
      },
      "5": {
        "minCodes": 1,
        "spin": "guaranteed",
        "guarantee": "welcome.1000"
      },
      "6": {
        "minCodes": 1,
        "spin": "not_full"
      }
    }
  }
}
```

Интерпретация:

- По умолчанию требуется 2 активных кода.
- Для `N=1` — спин гарантированный, нужен 1 код, выдаётся `Balance.code="welcome.500"`.
- Для `N=2` — нужен 1 код, тип `"full"`, приз - рандомный, пул включает ценные призы.
- Для `N=3` — гарантированный `"welcome.1000"`.
- Для `N=4` — тип спина не указан, движок выберет тип рандомно и самостоятельно.
- Для `N=5` — гарантированный `"welcome.1000"`.
- Для `N=6` — нужен 1 код, тип `"not_full"`, приз - тикет.

Далее сконфигурированные N заканчиваются, значит, для всех последующих кодов (7, 8, 9...) будет использоваться default – нужны 2 кода, выбор типа спина автоматически.

---

## 6) Пример расчёта доступности

Предположим:

- у пользователя `used_count = 2` (2 кода уже списаны ранее)
- значит `N = 3`
- по бренду найдено правило `overrides["3"]`, там `minCodes=2`

Если активных кодов сейчас:

- `active_count = 1` → спин **недоступен**
- `active_count = 2` → спин **доступен**, будут списаны 2 самых старых активных кода

---

## 7) Нюансы и рекомендации

### 7.1 Ключи overrides должны быть строками

Сервис ищет `overrides.get(str(N))`, поэтому ключи — `"1"`, `"2"`, ...

### 7.2 Конфиг привязан к `Activity.brand__code`

Сервис фильтрует активности по `brand__code=brand_code`, поэтому важно:

- чтобы у `Activity.brand` был корректный `code`
- чтобы коды создавались в активности с правильным `brand`

### 7.3 Поведение, если бренда нет в конфиге

Если для бренда отсутствует запись в `spin.awards.rules`, сервис:

- логирует `No config for brand=<brand_code>`
- кидает `SpinConfigError`

---

### Дефолтное значение конфига

<details>

<summary>
Показать
</summary>

```json
{
    "MRL": {
        "default": { "minCodes": 2 },
        "overrides": {
            "1": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.500"
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "5": {
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
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "5": {
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
                "minCodes": 1,
                "spin": "full"
            },
            "3": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "4": {
                "minCodes": 1
            },
            "5": {
                "minCodes": 1,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "6": {
                "minCodes": 1,
                "spin": "not_full"
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
                "minCodes": 1
            },
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "5": {
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
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "5": {
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
            "3": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1000"
            },
            "5": {
                "minCodes": 2,
                "spin": "guaranteed",
                "guarantee": "welcome.1500"
            }
        }
    }
}
```
</details>
