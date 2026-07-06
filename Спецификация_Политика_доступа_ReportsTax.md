# Спецификация: политика доступа к сервису «Налоговая отчётность» (ReportsTax)

## Метаданные

| Параметр | Значение |
|---|---|
| Сервис | Налоговая отчётность (ReportsTax) |
| КОДСЕРВИСА (code) | `payroll-reports-tax-admin` |
| Компонент | CoreUI |
| Эпик | Q3.2026_Tax_Общие компоненты |
| Security Level | K-3 |
| Тип политики | Видимость плитки сервиса (flow) |
| RuleCombiningAlgId | deny-overrides |

---

## 1. Назначение

Необходимо настроить доступ пользователей к плитке сервиса «Налоговая отчётность» в каркасе Пульс. Доступ регулируется политикой авторизации (XACML) по трём осям: роли, окружение, организация. Без внедрённой политики сервис возвращает решение авторизации `NotApplicable` («доступа нет — отсутствует нужная роль»).

Настоящая спецификация описывает требования к политике и к её внедрению. Разработка XACML, размещение в репозитории и вывод по стендам через ConfigManager выполняются на стороне разработки.

---

## 2. Требования к доступу

### 2.1. Роли

Доступ к плитке предоставляется ролям, работающим с сервисом через UI (по матрице функционал/роль ролевой модели ReportsTax):

| Техническое имя | Бизнес-наименование | Реалм | Основание |
|---|---|---|---|
| BR_PAYROLL_ACCOUNTANT | бухгалтер | Отчётность по налогам | Формирование РНУ НДФЛ через UI календаря |
| BR_METHODOLOGIST | бизнес-администратор «Финансы» | Отчётность по налогам | Просмотр НОБ Пульс/внешних, результатов, статусов, формирование РНУ |
| PAYROLL_SUPPORT | 2 линия поддержки расчёта зарплаты | Отчётность по налогам | Просмотр НОБ, результатов, статусов, BI Viewer |
| PAYROLL_ADMIN | технический администратор «Финансы» | Отчётность по налогам | Доступ к функционалу сервиса для сопровождения |

Роль ТУЗ сервиса (ReportsAnalytic, ReportsSalary) в политику видимости плитки не включается — это технологические учётные записи для интеграционных сценариев (РНУ для расчётного листа, справки о доходах), доступ по ТУЗ/эндпоинту, не через плитку.

### 2.2. Реалм

Все четыре роли проверяются в реалме сервиса — **«Отчётность по налогам»**. Проверка членства в политике выполняется в этом реалме (один блок `is-in-dyn-group`).

**Требование:** роли BR_PAYROLL_ACCOUNTANT, BR_METHODOLOGIST, PAYROLL_SUPPORT добавить в реалм «Отчётность по налогам» (на текущий момент реалм присвоен только роли PAYROLL_ADMIN).

### 2.3. Организации (компании)

Плитка доступна сотрудникам тенанта Sberbank в следующих компаниях: PAOSberbank, AOSBT, ANOSCU, AOSCIB, AOSL, OOOSI.

### 2.4. Ограничение окружения

Доступ ограничен использованием VPN — при отсутствии VPN плитка отключается (reason `NOT_VPN`).

---

## 3. Политика (XACML)

```xml
<Policy xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17"
        PolicyId="urn:sberbank:names:hrp-launchpad:xacml:policy:flow:payroll-reports-tax-admin"
        RuleCombiningAlgId="urn:oasis:names:tc:xacml:1.0:rule-combining-algorithm:deny-overrides"
        Version="1.0">
<Description>Сотрудники тенанта Сбербанк в указанных компаниях, состоящие в авторизационных группах сервиса (бухгалтер, бизнес-администратор, поддержка, технический администратор), могут видеть плитку сервиса «Налоговая отчётность».</Description>
<Target>
<AnyOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">PAOSberbank</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">AOSBT</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">ANOSCU</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">AOSCIB</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">AOSL</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">OOOSI</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:company-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
</AnyOf>
<AnyOf>
<AllOf>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">Sberbank</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:tenant-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true"/>
</Match>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:boolean-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#boolean">true</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:sberbank:names:hrp-authn:xacml:subject:is-employed" DataType="http://www.w3.org/2001/XMLSchema#boolean" MustBePresent="false"/>
</Match>
<Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">urn:sberbank:names:hrp-launchpad:xacml:action:get-flows</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="false"/>
</Match>
</AllOf>
</AnyOf>
</Target>
<VariableDefinition VariableId="is-allowed-by-group">
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:or">
<Apply FunctionId="urn:sberbank:names:hrp-authz:xacml:function:is-in-dyn-group">
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:integer-one-and-only">
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:oasis:names:tc:xacml:1.0:subject:subject-id" DataType="http://www.w3.org/2001/XMLSchema#integer" MustBePresent="true"/>
</Apply>
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:string-bag">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">BR_PAYROLL_ACCOUNTANT</AttributeValue>
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">BR_METHODOLOGIST</AttributeValue>
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">PAYROLL_SUPPORT</AttributeValue>
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">PAYROLL_ADMIN</AttributeValue>
</Apply>
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">Отчётность по налогам</AttributeValue>
</Apply>
</Apply>
</VariableDefinition>
<Rule RuleId="permit-flow-payroll-reports-tax-admin" Effect="Permit">
<Description>Разрешить доступ к плитке в случае наличия групп</Description>
<Condition>
<VariableReference VariableId="is-allowed-by-group"/>
</Condition>
<AdviceExpressions>
<AdviceExpression AdviceId="showFlow" AppliesTo="Permit">
<AttributeAssignmentExpression AttributeId="urn:sberbank:names:hrp-launchpad:xacml:data:flow-id">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">payroll-reports-tax-admin</AttributeValue>
</AttributeAssignmentExpression>
</AdviceExpression>
</AdviceExpressions>
</Rule>
<Rule RuleId="deny-flow-payroll-reports-tax-admin-vpn" Effect="Permit">
<Description>Запретить если без использования vpn</Description>
<Condition>
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:and">
<VariableReference VariableId="is-allowed-by-group"/>
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:not">
<Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:boolean-is-in">
<AttributeValue DataType="http://www.w3.org/2001/XMLSchema#boolean">true</AttributeValue>
<AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:requesting-machine" AttributeId="urn:sberbank:names:hrp-authn:xacml:requesting-machine:is-vpn" DataType="http://www.w3.org/2001/XMLSchema#boolean" MustBePresent="false"/>
</Apply>
</Apply>
</Apply>
</Condition>
<ObligationExpressions>
<ObligationExpression ObligationId="disableFlow" FulfillOn="Permit">
<AttributeAssignmentExpression AttributeId="urn:sberbank:names:hrp-launchpad:xacml:data:flows">
<AttributeValue DataType="urn:sberbank:names:xacml:data-type:json"> { "code": "payroll-reports-tax-admin", "decision": "disable", "reason": "NOT_VPN" } </AttributeValue>
</AttributeAssignmentExpression>
</ObligationExpression>
</ObligationExpressions>
</Rule>
</Policy>
```

---

## 4. Требования к внедрению (сторона разработки)

1. **Добавить роли в реалм.** Роли BR_PAYROLL_ACCOUNTANT, BR_METHODOLOGIST, PAYROLL_SUPPORT добавить в реалм «Отчётность по налогам» (PAYROLL_ADMIN уже присвоен).
2. **Разместить политику в репозитории сервиса** по структуре ConfigManager: `HRP-pipeline\<service-name>\platform-config-external\spine-authorization\<config_directory>\` с мета-файлом доставки.
3. **Оформить/связать задачу на изменение ролевой модели** (добавление ролей в реалм — изменение РМ). Без связанной задачи и согласования внедрение на ПРОМ блокируется. Заявка: [SIGMA] «Задача на изменение ролевой модели sigma» либо «Заявочный процесс по Ролевой модели в Пульс».
4. **Разработка и проверка на DEV** — админ-центр авторизации `https://hr-dev.sberbank.ru/admin/authorization`. Требуемые полномочия: AUTH_POLICY_EDIT, REALM_EDITOR.
5. **Вывод по стендам** через ConfigManager: DEV → ИФТ → ПСИ → ПРОМ (с шагами Update ConfigManager на каждом этапе).

---

## 5. Открытые пункты

1. Точное техническое имя реалма «Отчётность по налогам» (у смежного сервиса реалмы именуются технически, напр. `app-organization-calc-comp`) — уточнить перед разработкой политики.
2. Список компаний (раздел 2.3) — подтвердить по требованиям сервиса.
3. Ограничение окружения (раздел 2.4) — подтвердить (VPN / Citrix / EMM / без ограничений).

---

## Приложения / источники

- Ролевая модель ReportsTax (матрица функционал/роль) — источник ролей.
- Реалмы ролей — по данным команды авторизации (Пахомова М.Г.).
- Инструкция по политике доступа (пример XACML), КОДСЕРВИСА подтверждён ответственным за модуль.
- Процесс изменения политик через ConfigManager (pageId 7401736688).
