# Политика доступа к сервису «Налоговая отчётность» (ReportsTax)

| Параметр | Значение |
|---|---|
| Сервис | Налоговая отчётность (ReportsTax) |
| КОДСЕРВИСА (code) | `payroll-reports-tax-admin` |
| Компонент / КЭ | CoreUI |
| Эпик | Q3.2026_Tax_Общие компоненты |
| Security Level | K-3 |
| Источник ролей | Ролевая модель ReportsTax (матрица функционал/роль) |

## 1. Назначение

Описание политики доступа пользователя к плитке сервиса «Налоговая отчётность» в каркасе Пульс по трём осям: роли, окружение, организация. Политика регулирует видимость плитки (flow) сервиса.

## 2. Роли доступа к плитке

В политику включены роли, работающие с сервисом через UI (по матрице функционал/роль ролевой модели ReportsTax):

| Группа | Назначение | Обоснование включения |
|---|---|---|
| `BR_PAYROLL_ACCOUNTANT` | бухгалтер | Формирование РНУ НДФЛ через UI календаря |
| `BR_METHODOLOGIST` | бизнес-администратор | Просмотр НОБ Пульс/внешних, результатов отправки, статусов, формирование РНУ — через UI |
| `PAYROLL_SUPPORT` | 2-я линия поддержки | Просмотр НОБ, результатов, статусов, BI Viewer — через UI сервиса |
| `PAYROLL_ADMIN` | технический администратор | Доступ к функционалу сервиса для сопровождения |

**Не включается:** ТУЗ сервиса (ReportsAnalytic, ReportsSalary) — технологическая учётная запись для интеграционных сценариев («РНУ НДФЛ для расчётного листа», «Справка о доходах ФЛ для сервиса Справки»), доступ по ТУЗ/эндпоинту, не через плитку.

**Реалмы ролей** (по письму Пахомовой М.Г. от 30.06.2026):

| Группа | Реалмы (приложения/сервисы, где фигурирует роль) |
|---|---|
| `BR_PAYROLL_ACCOUNTANT` | Люди и данные, Launchpad, Переводы, Кадровые перемещения, Оплата труда, Aiops навыков, Configurator |
| `BR_METHODOLOGIST` | app-organization-calc-comp, Configurator |
| `PAYROLL_SUPPORT` | Accountant Work Place, Configurator |
| `PAYROLL_ADMIN` | app-organization-calc-comp, **Отчётность по налогам**, Configurator |

Для проверки членства в политике используется реалм сервиса — **«Отчётность по налогам»**. Все 4 роли проверяются в этом реалме (один блок `is-in-dyn-group`). Роли BR_PAYROLL_ACCOUNTANT, BR_METHODOLOGIST, PAYROLL_SUPPORT добавляются в реалм «Отчётность по налогам» (сейчас он есть у PAYROLL_ADMIN).

## 3. Организации (компании)

Список компаний с доступом к плитке — `<Target>`. Сейчас приведён полный набор по аналогии (подтвердить по требованиям сервиса; если только ПАО Сбербанк — оставить один блок PAOSberbank).

## 4. Ограничение окружения

Приведён блок ограничения по VPN (`deny-flow-...-vpn`, reason `NOT_VPN`). Подтвердить необходимость: если ограничений нет — удалить deny-блок; если нужен Citrix/EMM — добавить блок по образцу (reason из таблицы «Ограничение доступа в Пульс»).

## 5. Политика (XACML)

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

## 6. Перед отладкой

1. **Компании** — подтвердить итоговый список по требованиям сервиса (сейчас полный набор).
2. **Ограничение окружения** — подтвердить (VPN / Citrix / EMM / без ограничений).
3. **Добавление ролей в реалм** — три роли (BR_PAYROLL_ACCOUNTANT, BR_METHODOLOGIST, PAYROLL_SUPPORT) добавить в реалм «Отчётность по налогам».
4. **Отладка** — обкатать политику на стенде DEV через ConfigManager.
