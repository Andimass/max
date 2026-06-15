# Блок валидации РНУ — модель данных (таблицы для Confluence)

Вставлять через «+» → Разметка/Markup → Markdown (конвертирует и заголовки, и таблицы за один проход). Соответствует `validation_block_db_full.sql`.

## Перечисления (ENUM)

| Тип | Значения |
| --- | --- |
| validation_check_type | PRESENCE, FORMAT, LOGIC, CONTROL_NUMBER |
| validation_severity | ERROR, WARNING |
| validation_check_status | ACTIVE, INACTIVE |
| validation_applicability | QUARTERLY, DAILY, BOTH |
| validation_result_kind | DETECTED, NOT_DETECTED |
| validation_correction_status | NEW, SENT, IN_PROGRESS, FIXED, REFORMED, CLOSED |
| validation_overall_status | OK, WARNING, ERROR |
| validation_send_readiness | NOT_READY, READY_TO_SEND, SENT |
| validation_send_status | SUCCESS, ERROR |
| validation_send_target | VALIDATION, UOD |

## 1. rnu_setting.validation_check — реестр проверок

| Поле | Тип | Обяз. | По умолч. | Описание |
| --- | --- | --- | --- | --- |
| id | bigserial | X |  | Первичный ключ |
| code | varchar(10) | X |  | Код проверки (при отправке уходит как reasonId) |
| name | varchar(255) | X |  | Наименование |
| check_type | validation_check_type | X |  | Тип: наличие/формат/логика/контрольное число |
| severity | validation_severity | X |  | Важность: ошибка/предупреждение |
| error_code | varchar(4) |  |  | Код ошибки для UI/уведомления (10/12/25) |
| error_text_tpl | text |  |  | Шаблон текста отклонения |
| rule | text | X |  | Правило/выражение проверки |
| target_entity | varchar(20) |  |  | PERSON_DATA / INCOME / DEDUCTION |
| applicability | validation_applicability | X | BOTH | Квартальная / ежедневная (ЗОД) / обе |
| recommended_action | text |  |  | Рекомендация (источник для регистрации в Матрице Валидатора) |
| start_date | date | X |  | Действует с |
| end_date | date | X | 9999-12-31 | Действует по |
| status | validation_check_status | X | ACTIVE | Активна / неактивна |
| created_at | timestamptz | X | now() | Создано |
| updated_at | timestamptz | X | now() | Обновлено (триггер) |
| updated_by | varchar(50) |  |  | Кто изменил |

Ограничения: UNIQUE (code, start_date); CHECK (end_date ≥ start_date).

## 2. rnu_report.validation_result — лог результатов проверок

| Поле | Тип | Обяз. | По умолч. | Описание |
| --- | --- | --- | --- | --- |
| id | uuid | X | gen_random_uuid() | Первичный ключ |
| check_code | varchar(10) | X |  | Ссылка на validation_check.code (по дате); reasonId |
| job_id | uuid | X |  | Задание формирования |
| job_period_id | uuid |  |  | Период отчёта |
| person_id | uuid |  |  | ФЛ |
| do_id | uuid |  |  | Договорные отношения |
| employee_number | varchar(10) |  |  | Табельный номер |
| operation_id | varchar(30) |  |  | Ссылка на запись РНУ |
| income_id | uuid |  |  | Строка дохода (rnu_income) |
| service_id | varchar(30) |  |  | Мастер-сервис (serviceId, опц.) |
| result | validation_result_kind | X |  | Выявлено / не выявлено (→ verificationStatus) |
| severity | validation_severity |  |  | Важность на момент прогона |
| error_code | varchar(4) |  |  | Код ошибки |
| error_text | text |  |  | Текст отклонения (→ message) |
| recommended_action | text |  |  | Рекомендация (для UI; в Валидацию не шлётся) |
| checked_at | timestamptz | X | now() | Дата/время проверки (→ verificationDate/time) |
| deviation_id | uuid |  |  | Стабильный ключ отклонения |
| sent_to_validation | boolean | X | false | Признак отправки |
| sent_at | timestamptz |  |  | Когда отправлено |
| validator_request_no | varchar(50) |  |  | Номер заявки task_id (из ответа Валидатора) |
| correction_status | validation_correction_status | X | NEW | Жизненный цикл отклонения |
| created_at | timestamptz | X | now() | Создано |
| updated_at | timestamptz | X | now() | Обновлено (триггер) |

## 3. rnu_report.validation_summary — итог по сотруднику (результирующая)

| Поле | Тип | Обяз. | По умолч. | Описание |
| --- | --- | --- | --- | --- |
| id | uuid | X | gen_random_uuid() | Первичный ключ |
| job_id | uuid | X |  | Задание формирования |
| job_period_id | uuid | X |  | Период отчёта |
| person_id | uuid | X |  | ФЛ |
| do_id | uuid |  |  | Договорные отношения (опц.) |
| employee_number | varchar(10) |  |  | Табельный номер |
| validation_status | validation_overall_status | X |  | Вердикт: OK / WARNING / ERROR |
| error_count | int | X | 0 | Кол-во ошибок (CHECK ≥ 0) |
| warning_count | int | X | 0 | Кол-во предупреждений (CHECK ≥ 0) |
| send_readiness | validation_send_readiness | X | NOT_READY | «Готов к отправке» |
| last_checked_at | timestamptz | X | now() | Время последней проверки |
| created_at | timestamptz | X | now() | Создано |
| updated_at | timestamptz | X | now() | Обновлено (триггер) |

Ограничения: UNIQUE (job_period_id, person_id) — грейн по ФЛ (вариант: + do_id). Питает UI-бейдж, гейт XML по ФЛ и гейт платёжек УОД.

## 4. rnu_report.validation_sending_log — лог отправки (Валидация + УОД)

| Поле | Тип | Обяз. | По умолч. | Описание |
| --- | --- | --- | --- | --- |
| id | uuid | X | gen_random_uuid() | Первичный ключ |
| target | validation_send_target | X |  | Потребитель: VALIDATION / UOD |
| message_id | uuid |  |  | messageId запроса — идемпотентность |
| batch_id | uuid |  |  | Идентификатор пачки |
| deviation_id | uuid |  |  | Если лог по одному отклонению |
| task_id | varchar(50) |  |  | Номер заявки из ответа Валидатора |
| operation_type | varchar(20) | X | CREATE_DEVIATION | Тип операции |
| operation_status | validation_send_status | X |  | SUCCESS / ERROR (исход вызова) |
| payload | jsonb |  |  | Отправленное тело (аудит/реплей) |
| response_code | varchar(20) |  |  | Код ответа |
| response_message | text |  |  | Текст ответа |
| error_code | varchar(20) |  |  | Код ошибки отправки |
| error_text | text |  |  | Текст ошибки отправки |
| created_at | timestamptz | X | now() | Создано |
