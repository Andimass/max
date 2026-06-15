# Блок валидации РНУ — модель данных (таблицы для Confluence)

Вставлять через «+» → Разметка/Markup → Markdown (конвертирует заголовки, текст и таблицы за один проход). Соответствует `validation_block_db_full.sql`.

## Перечисления (ENUM)

| Тип | Значения |
| --- | --- |
| validation_check_type | PRESENCE, FORMAT, LOGIC, CONTROL_NUMBER |
| validation_severity | ERROR, WARNING |
| validation_check_status | ACTIVE, INACTIVE |
| validation_applicability | QUARTERLY, DAILY, BOTH |
| validation_recipient | SUPPORT (техподдержка), PRO_USER (профпользователь), INDIVIDUAL (физлицо) |
| validation_result_kind | DETECTED, NOT_DETECTED |
| validation_correction_status | NEW, SENT, IN_PROGRESS, FIXED, REFORMED, CLOSED |
| validation_overall_status | OK, WARNING, ERROR |
| validation_send_readiness | NOT_READY, READY_TO_SEND, SENT |
| validation_send_status | SUCCESS, ERROR |
| validation_send_target | VALIDATION, UOD |

## 1. rnu_setting.validation_check — реестр проверок

**Назначение.** Справочник всех проверок РНУ: определение проверки (тип, правило, важность), применимость (квартал/ЗОД), маршрутизация результата (Валидация / ДКЗ / Управление задолженностью), ответственный за исправление и алгоритм обработки ошибки сервисом Валидации. Настроечная таблица, ведётся методологом, расширяется новыми записями. Детальные рекомендации по получателям вынесены в `validation_check_action`.

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
| recommended_action | text |  |  | Общая рекомендация (детализация по получателям — в validation_check_action) |
| responsible_role | validation_recipient |  |  | Ответственный: техподдержка/профпользователь/физлицо |
| validation_algorithm | text |  |  | Алгоритм обработки ошибки сервисом Валидации |
| send_to_validation | boolean | X | true | Маршрутизация: отправлять ли ошибки в сервис Валидация |
| send_to_dkz | boolean | X | false | Маршрутизация: отправлять ли в ДКЗ |
| send_to_debt_management | boolean | X | false | Маршрутизация: отправлять ли в Управление задолженностью |
| start_date | date | X |  | Действует с |
| end_date | date | X | 9999-12-31 | Действует по |
| status | validation_check_status | X | ACTIVE | Активна / неактивна |
| created_at | timestamptz | X | now() | Создано |
| updated_at | timestamptz | X | now() | Обновлено (триггер) |
| updated_by | varchar(50) |  |  | Кто изменил |

Ограничения: UNIQUE (code, start_date); CHECK (end_date ≥ start_date).

## 2. rnu_setting.validation_check_action — рекомендации по получателям

**Назначение.** Рекомендованные действия по исправлению ошибки в разрезе получателя (техподдержка / профпользователь / физлицо). Нормализованная таблица: одна строка на пару «проверка × получатель», поэтому новые роли добавляются записями, а не колонками. Привязана к проверке через `check_code`; версионируется датами действия.

| Поле | Тип | Обяз. | По умолч. | Описание |
| --- | --- | --- | --- | --- |
| id | bigserial | X |  | Первичный ключ |
| check_code | varchar(10) | X |  | Ссылка на validation_check.code (по дате действия) |
| recipient | validation_recipient | X |  | Получатель: техподдержка/профпользователь/физлицо |
| recommended_action | text | X |  | Рекомендованное действие для этого получателя |
| start_date | date | X |  | Действует с |
| end_date | date | X | 9999-12-31 | Действует по |
| created_at | timestamptz | X | now() | Создано |
| updated_at | timestamptz | X | now() | Обновлено (триггер) |

Ограничения: UNIQUE (check_code, recipient, start_date); CHECK (end_date ≥ start_date).

## 3. rnu_report.validation_result — лог результатов проверок

**Назначение.** Лог результатов прогона по каждой паре «проверка × запись РНУ». Источник отклонений для отправки в Валидацию и основа повторной сверки после исправления. Транзакционная таблица. Поля выровнены под контракт отправки (`check_code`→reasonId, `error_text`→message, `result`→verificationStatus, `validator_request_no`=task_id).

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

## 4. rnu_report.validation_summary — итог по сотруднику (результирующая)

**Назначение.** Итог валидации по сотруднику за период (ОК/не ОК) — роллап из `validation_result`. Питает UI-бейдж статуса, гейт формирования XML по ФЛ и гейт платёжек в сервисе УОД (платёжки формируются только при общем ОК). Транзакционная таблица.

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

Ограничения: UNIQUE (job_period_id, person_id) — грейн по ФЛ (вариант: + do_id).

## 5. rnu_report.validation_sending_log — лог отправки (Валидация + УОД)

**Назначение.** Аудит исходящих уведомлений — и в сервис Валидация (массив отклонений), и в сервис УОД (гейт платёжек); потребитель различается полем `target`. Хранит `message_id` (идемпотентность запроса), `task_id` (заявку из ответа), отправленное тело и исход вызова. Транзакционная, аналог `rnu_xml_sending_log`.

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
