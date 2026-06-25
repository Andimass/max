# Доработка параметра system_client_id при отправке НОБ Пульс в Мастер Систему

## Метаданные

| Параметр | Значение |
|---|---|
| Тип / приоритет | Task / Minor |
| Компонент | ReportsTax (app-payroll-reports-tax) |
| Security Level | K-3 |
| Эпик | Q2.2026_Tax_Общие компоненты |
| Команда | Регуляторная отчётность (PY) |
| КЭ | ReportsTax(8449473) |
| Связанные задачи | HRPY-23866 (SA. do_id) |

---

## 1. Бизнес-контекст и причина доработки

При отправке НОБ Пульс в Мастер Систему НДФЛ в параметре `system_client_id` в роли идентификатора клиента передаётся СНИЛС. По требованию бизнеса для сверки на стороне Мастер Системы необходимо перейти на идентификатор 3 уровня — `do_id` (идентификатор договорных отношений).

Дополнительная причина: на текущий момент отправка НОБ Пульс частично нерабочая — сообщения уходят в DLQ (`MSNDFL.OPERATIONS_PULS-DLQ.V1`). Причина в том, что `system_client_id` передаётся числом (number). Ранее числовые значения проходили валидацию на стороне SEDR, но после обновления схем со стороны Мастер Системы строгая валидация перестала пропускать число в этом поле.

Перевод поля в строку решает обе задачи одновременно: `do_id` представляет собой UUID (строку), что выполняет требование бизнеса и устраняет причину попадания сообщений в DLQ.

Частота операции: проверка статусов в таблице `employee_py_nob` каждые 5 минут, отправляются строки со статусом `send_status = R`.

---

## 2. Как реализовано сейчас (AS-IS)

Алгоритм формирования сообщения отправки (раздел 2.2.1):

1. JOIN с таблицей `epk_person_relation.snils` для получения СНИЛС.
2. Фильтрация по статусу.
3. Группировка по `epk_id`.
4. В разрезе `epk_id` — группировка по `operation_id`.
5. Раскладка по объектам `rate / base / tax`.

В теле сообщения, внутри `operations[] -> epk_operations[]`, поле `system_client_id` заполняется значением СНИЛС.

В JSON-схеме запроса поле `system_client_id` имеет тип number (minimum 0, maximum — 19 девяток). В JSON-схеме ответа поле `system_client_id` уже имеет тип string (maxLength 100).

---

## 3. Как должно быть (TO-BE)

### 3.1. Источник значения

В параметр `system_client_id` подставляется `do_id`. Значение берётся из таблицы `employee_py_nob`, поле `do_id` — оно уже присутствует в отправляемой строке (заполняется из `resultSet.do_id` на этапе получения данных прогонов). Дополнительный JOIN не требуется.

Шаг 1 алгоритма формирования сообщения (JOIN с `epk_person_relation.snils` для получения СНИЛС) удаляется. Шаги 2–5 (фильтрация по статусу, группировка по `epk_id`, группировка по `operation_id`, раскладка по `rate / base / tax`) остаются без изменений.

### 3.2. Группировка

Группировка операций остаётся по `epk_id`. Структура массива `epk_operations` по `do_id` не разворачивается. Мастер Система рассчитывает НОБ в разрезе `epk_id` и не использует `system_client_id` для расчёта, поэтому значение этого поля для неё непринципиально. Изменяется только содержимое поля, структура сообщения сохраняется.

### 3.3. Тип поля и JSON-схемы

Основное изменение вносится в схему запроса: тип поля `system_client_id` изменяется с number на string, `maxLength` устанавливается равным 255 (по аналогии с соседними строковыми полями схемы запроса — `external_id`, `kpp`, `oktmo`, `rq_uuid`; запас закладывается из-за строгой валидации SEDR, UUID занимает 36 символов).

В схеме ответа поле `system_client_id` уже имеет тип string (maxLength 100), доработка не требуется. Необходимо убедиться, что длины 100 символов достаточно (UUID = 36 символов, запас имеется), и что обработка ответа корректно читает строковое значение.

### 3.4. Маппинг поля

| Тэг | Схема | Тип AS-IS | Тип TO-BE | Обязательность | Источник TO-BE | Комментарий |
|---|---|---|---|---|---|---|
| system_client_id | Запрос (2.2.1) | Число (number, max 19 девяток) | Строка (string, maxLength 255) | 1 | employee_py_nob.do_id | Внутренний ИД клиента. Было — СНИЛС из epk_person_relation.snils, стало — do_id (UUID) |
| system_client_id | Ответ (2.2.2) | Строка (string, maxLength 100) | Строка (string, maxLength 100) | 1 | — | Уже string, изменения не требуются. Проверить достаточность длины |

Остальные поля сообщения (`rq_uuid`, `rq_date_time`, `system_code`, `base_type`, `year`, `epk_id`, `external_id`, `operation_date_time`, `kpp`, `oktmo`, `base_tax[]{rate, base, tax}`) остаются без изменений.

---

## 4. JSON-схемы и примеры (проверены в SEDR)

Схемы и примеры проверяются аналитиком через валидатор APIStudio SEDR до передачи в разработку: `https://apistudio-iamosh.sigma.sbrf.ru/#/tools/json` (сверху выбрать профиль SEDR). В разработку передаётся уже провалидированная схема запроса.

### 4.1. Схема запроса (SRC.MSNDFL.OPERATIONS_PULS.V1) — изменённая

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "template",
  "type": "object",
  "properties": {
    "rq_uuid": { "type": "string", "maxLength": 255 },
    "rq_date_time": { "type": "string", "maxLength": 255 },
    "system_code": { "type": "string", "maxLength": 255 },
    "base_type": { "type": "string", "maxLength": 255 },
    "year": { "type": "number", "minimum": 1990, "maximum": 2100 },
    "operations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "epk_id": { "type": "number", "minimum": 0, "maximum": 9999999999999999999 },
          "epk_id_string": { "type": "string", "maxLength": 255 },
          "epk_operations": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "system_client_id": { "type": "string", "maxLength": 255 },
                "external_id": { "type": "string", "maxLength": 255 },
                "operation_date_time": { "type": "string", "maxLength": 255 },
                "kpp": { "type": "string", "maxLength": 255 },
                "oktmo": { "type": "string", "maxLength": 255 },
                "base_tax": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "rate": { "type": "number", "minimum": 0, "maximum": 50 },
                      "base": { "type": "number", "minimum": 0, "maximum": 9999999999999999999 },
                      "tax": { "type": "number", "minimum": 0, "maximum": 9999999999999999999 }
                    },
                    "additionalProperties": false
                  },
                  "maxItems": 1000,
                  "uniqueItems": true,
                  "additionalItems": false
                }
              },
              "additionalProperties": false
            },
            "maxItems": 1000,
            "uniqueItems": true,
            "additionalItems": false
          }
        },
        "additionalProperties": false
      },
      "maxItems": 1000,
      "uniqueItems": true,
      "additionalItems": false
    }
  },
  "additionalProperties": false
}
```

### 4.2. Пример запроса (system_client_id = do_id)

```json
{
  "rq_uuid": "9abbc670-9b12-4ce1-91cc-05983e249be4",
  "rq_date_time": "2025-12-01T16:59:11.089",
  "system_code": "HR",
  "base_type": "GENERAL",
  "year": 2025,
  "operations": [
    {
      "epk_id": 1815551541587704,
      "epk_operations": [
        {
          "system_client_id": "da8aeb88-eb5e-46a7-9c72-938852e798ac",
          "external_id": "o1",
          "operation_date_time": "2025-12-01T16:59:11.089",
          "kpp": "123456789",
          "oktmo": "12345678",
          "base_tax": [
            { "rate": 13, "base": 1000.89, "tax": 130 },
            { "rate": 15, "base": 5557.89, "tax": 834 }
          ]
        }
      ]
    }
  ]
}
```

### 4.3. Схема ответа (DST.MSNDFL.OPERATIONS_TO_PULS.V1) — без изменений

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "rq_uuid": { "type": "string", "maxLength": 255 },
    "status_code": { "type": "integer", "minimum": 0, "maximum": 100 },
    "error_message": { "type": ["string", "null"], "maxLength": 1024 },
    "sums": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "epk_id": { "type": "integer", "minimum": 1, "maximum": 100000000000000000000 },
          "epk_operations": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "system_client_id": { "type": "string", "maxLength": 100 },
                "external_id": { "type": "string", "maxLength": 100 },
                "status_code": { "type": "integer", "minimum": 0, "maximum": 599 },
                "error_message": { "type": ["string", "null"], "maxLength": 1024 },
                "kpp": { "type": "string", "maxLength": 20 },
                "oktmo": { "type": "string", "maxLength": 20 },
                "note": { "type": ["string", "null"], "maxLength": 2048 },
                "base_tax": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "rate": { "type": "number", "minimum": 0, "maximum": 100 },
                      "base": { "type": "number", "minimum": 0, "maximum": 100000000000000000000 },
                      "tax": { "type": "number", "minimum": 0, "maximum": 100000000000000000000 }
                    },
                    "required": ["rate", "base", "tax"],
                    "additionalProperties": false
                  },
                  "maxItems": 1000,
                  "uniqueItems": true,
                  "additionalItems": false
                }
              },
              "required": ["system_client_id", "external_id", "status_code"],
              "additionalProperties": false
            },
            "maxItems": 1000,
            "uniqueItems": true,
            "additionalItems": false
          }
        },
        "required": ["epk_id", "epk_operations"],
        "additionalProperties": false
      },
      "maxItems": 1000,
      "uniqueItems": true,
      "additionalItems": false
    }
  },
  "required": ["rq_uuid", "status_code", "sums"],
  "additionalProperties": false
}
```

### 4.4. Пример ответа (system_client_id = do_id)

```json
{
  "rq_uuid": "9abbc670-9b12-4ce1-91cc-05983e249be4",
  "status_code": 0,
  "sums": [
    {
      "epk_id": 1815551541587704,
      "epk_operations": [
        {
          "system_client_id": "da8aeb88-eb5e-46a7-9c72-938852e798ac",
          "external_id": "o1",
          "status_code": 0,
          "kpp": "123456789",
          "oktmo": "12345678",
          "base_tax": [
            { "rate": 13, "base": 1000.89, "tax": 130 },
            { "rate": 15, "base": 5557.89, "tax": 834 }
          ]
        }
      ]
    }
  ]
}
```

---

## 5. Задача на разработку

### 5.1. Что сделать

1. **Источник значения system_client_id.**
   В алгоритме формирования сообщения (топик `SRC.MSNDFL.OPERATIONS_PULS.V1`) брать `system_client_id` из `employee_py_nob.do_id` (значение уже есть в отправляемой строке).
   Удалить шаг JOIN с `epk_person_relation.snils`, который сейчас используется только для получения СНИЛС.

2. **Тип поля и JSON-схема запроса.**
   В схеме сообщения отправки сменить `system_client_id` с number на string, `maxLength` = 255 (см. раздел 4.1).

3. **Схема ответа — доработка НЕ требуется.**
   В схеме ответного сообщения (топик `DST.MSNDFL.OPERATIONS_TO_PULS.V1`) поле `system_client_id` уже имеет тип string (maxLength 100). Менять тип не нужно. Необходимо лишь убедиться, что обработка ответа корректно читает строковое значение и что длины 100 символов достаточно (UUID = 36, запас есть).

4. **Группировка — не трогать.**
   Группировка операций остаётся по `epk_id`; структура `epk_operations` не меняется. Меняется только значение поля.

Провалидированные через SEDR схемы и примеры приведены в разделе 4 — разработка берёт готовую схему запроса оттуда.

### 5.2. Критерии приёмки

- В сообщении отправки `system_client_id` содержит do_id (значение из `employee_py_nob.do_id`), тип — строка.
- Схема запроса реализована по провалидированному образцу из раздела 4 (`system_client_id` = string, maxLength 255).
- Схема ответа не изменялась (поле уже string); обработка строкового значения в ответе корректна.
- Сообщения с do_id в `system_client_id` не уходят в DLQ.
- Группировка по `epk_id` сохранена, структура `epk_operations` не изменилась.

### 5.3. Тест-кейсы (минимум)

- Отправка записи из `employee_py_nob` со статусом R -> в сообщении `system_client_id` = do_id (string, напр. `da8aeb88-eb5e-46a7-9c72-938852e798ac`), статус переходит R->P, сообщение проходит SEDR (не DLQ).
- Несколько do_id на одном epk_id -> группировка по epk_id сохраняется, в `epk_operations` корректные значения.
- Получение ответа из `DST.MSNDFL.OPERATIONS_TO_PULS.V1` со строковым `system_client_id` -> корректная обработка, статус S/E по `status_code`.

---

## 6. Порядок внедрения

1. Аналитик: прогон схем и примеров через валидатор SEDR, размещение провалидированных схем и примеров в спецификации (раздел 4).
2. Разработка на стороне ReportsTax: источник значения `system_client_id` (`employee_py_nob.do_id`), тип поля string в схеме запроса (по готовой схеме из раздела 4).
3. После выполнения доработки — уведомить Бочкарева для внесения соответствующего изменения на его стороне.

Порядок действий важен: смена схемы до готовности доработки отправки приведёт к остановке отправки. Сначала выполняется доработка, затем направляется сигнал Бочкареву.

---

## 7. Данные для тестирования

Mock-контур SberMock (ReportsTax):

| Параметр | Значение |
|---|---|
| Кластер Kafka | pvlsq-mock00081…086.sigma.sbrf.ru:9092 |
| Префикс топиков | reportstaxmock |
| Входящий топик | reportstaxmock.signedrun |
| Ответный топик | reportstaxmock.RPL.signedrun |

Тело входящего события: `runId` (int), `runData[]{ calcHeaderId (string), doIds[] }`, `package`, `size`.

Сценарий проверки: отправить тестовое событие в топик `reportstaxmock.signedrun`, дождаться появления записи в таблице `employee_py_nob` со статусом `send_status = R`, проверить, что в исходящем сообщении `system_client_id` содержит `do_id` в строковом формате, и убедиться, что сообщение проходит валидацию SEDR и не уходит в DLQ.

---

## 8. Влияние доработки

- Устраняется текущее падение отправки НОБ Пульс в DLQ.
- Параметр `system_client_id` заполняется значением `do_id` (string) из `employee_py_nob.do_id`.
- Обновляется JSON-схема запроса (2.2.1). Схема ответа (2.2.2) изменений не требует — поле уже string.
- Группировка по `epk_id` сохраняется, структура `epk_operations` не изменяется.
- Затрагивается сторона Бочкарева (по сигналу после завершения разработки).
