# edt-companion-mcp — гайд для LLM-агента

Этот файл — справочник для нейросетевого агента (Claude, Cursor, Cline, ...), который подключён к плагину через MCP. Прочти его один раз перед работой. Установка и подключение описаны в [README](../README.md).

## Что это

OSGi-плагин для 1C:EDT 2025.2, который поднимает локальный MCP-сервер `http://127.0.0.1:6868/mcp` и отдаёт **39 инструментов** для работы с открытой в EDT конфигурацией 1С: чтение метаданных и BSL-кода, навигация, поиск, валидация, редактирование метаданных и BSL-модулей, запуск и отладка yaxunit-тестов, проверка запросов, поиск в платформенной документации.

Инструменты работают **только над тем workspace, который сейчас открыт в EDT** — отдельного процесса 1С/EDT плагин не поднимает. Если EDT закрыт — `/health` недоступен.



## Подключение

```jsonc
// .mcp.json
{
  "mcpServers": {
    "edt-companion-mcp": {
      "type": "http",
      "url": "http://127.0.0.1:6868/mcp"
    }
  }
}
```

Протокол — JSON-RPC 2.0. Поддержаны методы `initialize`, `tools/list`, `tools/call`. Проверка живости: `GET http://127.0.0.1:6868/health` → `{"status":"ok","tools":39}`.

### Два экземпляра EDT с разными проектами

Плагин работает только над workspace того экземпляра EDT, внутри которого запущен. Чтобы вести два EDT с разными проектами одновременно, каждому нужно выделить свой порт и завести отдельный сервер в `.mcp.json`. Готовая пошаговая инструкция — [docs/multi-instance.md](multi-instance.md).

## Конвенции параметров (важно)

- **FQN метаданного объекта** — Java-формат: `Catalog.Контрагенты`, `Document.РеализацияТоваровУслуг`, `InformationRegister.ЦеныНоменклатуры`, `Enum.СтатусыЗаказов`, `Constant.ВалютаУчёта`, `CommonModule.ОбщегоНазначения`. Имена самих объектов — как в конфигурации (русские/английские, регистрозависимые).
- **FQN формы** — `Catalog.X.Form.ФормаЭлемента`, `Document.Y.Form.ФормаСписка`.
- **FQN макета** — `Catalog.X.Template.Печать` (associated) или `CommonTemplate.УниверсальныйМакет` (top-level).
- **Nested FQN** в `edit_metadata.setObjectProperty` — пары `<Kind>.<Name>` после top: `Document.X.TabularSection.Y`, `Document.X.TabularSection.Y.Attribute.Z`, `Catalog.X.Form.Y`, `Catalog.X.Template.Z`, `AccumulationRegister.X.Dimension.D`, `Enum.X.EnumValue.V`. Поддержаны kind'ы: `TabularSection`, `Attribute`, `Form`, `Template`, `Command`, `Dimension`, `Resource`, `EnumValue`, `AccountingFlag`, `ExtDimensionAccountingFlag`, `AddressingAttribute`, `Column`, `Operation`, `Recalculation`. `forceExport` всегда бьёт по top-FQN.
- **workspacePath модуля BSL** — путь от корня workspace через `/`, начинается со слэша: `/WMS/src/CommonModules/ОбщегоНазначения/Module.bsl`, `/WMS/src/Catalogs/Контрагенты/ObjectModule.bsl`, `/WMS/src/Catalogs/Контрагенты/Forms/ФормаЭлемента/Module.bsl`.
- **`projectName`** — имя проекта в EDT workspace (получи через `list_workspace_projects`), не FQN.
- **`applicationId`** — отображаемое имя 1С Run Configuration (`list_applications`), не technical ID.
- **`dryRun: true`** поддержан большинством мутирующих операций — выполни сначала в dry-run, посмотри payload, потом без флага.

Если не уверен в имени проекта/FQN/пути модуля — **сначала вызови соответствующий list/get**, не угадывай. Большинство объектов имеют русские имена и регистр важен.

## Каталог инструментов

### Workspace и среда (3)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `list_workspace_projects` | Какие проекты открыты в EDT, какой тип (Configuration/Extension/External), какая совместимость. Старт любого диалога. | — |
| `list_applications` | 1С Run Configurations типа RuntimeClient (`applicationId` для запуска). | — |
| `show_edt_version` | Версии EDT/Eclipse/Java, открытые проекты с `kind`, `compatibilityMode`, `baseProject` для расширений. | — |

### Метаданные — чтение (5)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `list_metadata_objects` | Перечислить top-level объекты конфигурации (с фильтрами по типу/regexp). Принимает и Configuration- (cf), и Extension-проект (cfe). | `projectName` (cf/cfe), опц. `objectType` (EClass), `namePattern` (Java regex, case-insensitive), `limit` |
| `list_modules` | BSL-модули объекта или всех объектов; фильтр по виду модуля. Принимает и Configuration- (cf), и Extension-проект (cfe). | `projectName` (cf/cfe), опц. `objectName`, `kindFilter` |
| `get_object_details` | Полная структура: реквизиты с типами, ТЧ, измерения/ресурсы, формы, команды, макеты, модули. Скалярные флаги карточки объекта (отдаются с учётом дефолта платформы, даже когда тега нет в `.mdo`): `fullTextSearch`, `useStandardCommands`, `includeInCommandInterface`, `dataLockControlMode`; для Document — `numberType`/`numberLength`/`numberPeriodicity`/`postInPrivilegedMode`/...; для Catalog — `codeType`/`codeLength`/`hierarchical`/`hierarchyType`/...; ссылочные списки `owners`/`basedOn` (массив FQN). Для регистров — `informationRegisterPeriodicity`/`writeMode`/`registerType`/... (от них зависит виртуальная таблица `СрезПоследних`/`Остатки`/...). Для Catalog/ChartOfCharacteristicTypes/ChartOfAccounts/ChartOfCalculationTypes — `predefinedItems` (дерево). Через FQN. | `projectName`, `fqn` |
| `get_config_properties` | Шапка конфигурации: vendor, version, compatibilityMode, scriptVariant + `objectCounts` по всем типам. | `projectName` |
| `get_form_layout` | Headless-дамп управляемой формы (без открытия редактора): `attributes`, `items` рекурсивно, `formCommands`, `handlers`. | `projectName`, `fqn` (родитель), `formName`, опц. `format=tree|json`, `includeHandlers`, `maxDepth` |

### Навигация по BSL-коду (5)

Все 5 видят **несохранённые правки** из открытых редакторов EDT (через BM read-transaction).

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `read_module_source` | Текст BSL-модуля с построчной пагинацией. Большие модули (БСП 1500+ строк) не валятся по token-лимиту: без `limit` возвращается первая порция ≤~48k символов с `truncated:true` + `nextOffset` для дочитывания. В ответе `totalLines`/`totalChars`/`returnedLines`. | `modulePath`, опц. `offset`/`limit` (строки), `maxChars` |
| `get_module_structure` | Процедуры/функции (имя, kind, export, async, параметры, диапазон строк) + регионы. Сначала структура — потом точечно `read_method_source`. | `modulePath` |
| `read_method_source` | Текст одной процедуры/функции — экономит контекст vs `read_module_source`. | `modulePath`, `methodName` |
| `search_in_code` | Workspace-wide или per-project текстовый/regex-поиск по `*.bsl`. До 500 матчей. | `query`, опц. `projectName`, `caseSensitive`, `regex`, `limit` |
| `resolve_symbol` | Резолв символа по позиции (line+column или offset) к декларации. | `modulePath`, `line`+`column` или `offset` |

### Запись BSL (1)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `write_module_source` | Запись BSL-модуля через **shared Eclipse text buffer** — попадает в тот же dirty buffer, который видит открытый Xtext-редактор EDT. То есть: если у пользователя открыт редактор и в нём несохранённые правки, наша запись применяется поверх dirty state (а не перезатирает диск конфликтом). 6 режимов: `replace` (полная замена/создание файла), `append`, `insertBefore`/`insertAfter` (по `line`, 1-based), `replaceLines` (`startLine`+`endLine` inclusive), `searchReplace` (literal find→text + `expectMatches` контроль безопасности — по умолчанию ровно 1 совпадение; **переводы строк `find`/`text` нормализуются к делимитеру модуля** — многострочный LF-фрагмент из read-инструментов корректно матчится к CRLF-модулю и не подмешивает LF). Параметр `save` (default `true`) — commit buffer на диск; `false` оставляет в editor dirty state, пользователь сохранит Ctrl+S. `dryRun` для what-if. В ответе `oldLength`/`newLength`/`startLine`/`endLine`/`dirty`/`saved`/`created`. **Tier: PRO**. | `modulePath`, `mode`, `text`, опц. `line`/`startLine`+`endLine`/`find`+`expectMatches`/`save`/`dryRun` |

### Анализ кода (3)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `find_object_references` | Где используется метаданный объект. Сканирует **все** открытые Configuration и Extension проекты workspace, агрегирует matches с полями `projectName` / `filePath` / `ownerFqn`. Покрывает: BM cross-refs от самого target и его children (StandardAttribute/Attribute/TabularSection — для `Right.objectAttribute`), composite-types (`<types>CatalogRef.X</types>` через UUID-scan `TypeItem`). `kindFilter=code|metadata` — фильтр. | `objectName`, опц. `projectName` (origin для поиска target), `kindFilter`, `limit` |
| `get_method_call_hierarchy` | Callers/callees BSL-метода до depth=5. | `projectName`, `methodName`, опц. `modulePath`, `direction=callers|callees` |
| `get_validation_errors` | Маркеры EDT-валидации (НЕ Eclipse problems). Принимает и Configuration-, и Extension-проекты. | `projectName`, опц. `scope=project|object`, `fqn`, `minSeverity=BLOCKER|CRITICAL|MAJOR|MINOR|TRIVIAL` |

### Валидация запросов (1)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `validate_query` | Проверка текста запроса штатным Xtext-парсером EDT (RU/EN ключевые слова). Только синтаксис + типовые QL-проверки, без semantic linking к метаданным. | `queryText`, опц. `isDcs=true` (грамматика QlDcs) |

### Редактирование метаданных (3)

Универсальный диспетчер `edit_metadata` принимает поле `operation`. Все операции в read-write BM-транзакции, поддерживают `dryRun=true`. После коммита плагин синхронно пишет `.mdo` через `forceExport`.

`edit_metadata.operation` (выбор):

- **Создание/удаление объектов:** `createObject` (Catalog/Document/InformationRegister/AccumulationRegister/Enum/Constant/Report/DataProcessor/CommonModule/DocumentJournal/ChartOfCharacteristicTypes/ExchangePlan/Subsystem/Role/DefinedType/EventSubscription/CommonTemplate/XDTOPackage), `removeObject` — синхронно чистит и BM, и `Configuration.mdo`, и **папку объекта на диске** (`src/cf/src/<Type>/<Name>/` со всеми подкаталогами Forms/Templates/Commands). В ответе `folderDeleted`+`folderPath`; `persisted:true` только если файлы реально удалены. Принимает и Configuration-, и Extension-проекты (`isExtension:true` в ответе для cfe). Для `Role` пишется skeleton `Rights.rights`, для `XDTOPackage` — skeleton `Package.xdto`, для `CommonModule` — пустой `src/CommonModules/<Имя>/Module.bsl` (иначе EDT считает модуль отсутствующим и писать код некуда; в ответе `moduleFilePath`/`moduleFileCreated`); namespace задаётся параметром `namespace` (пишется и в `XDTOPackage.namespace`, и в `targetNamespace` скелета; если не передан — подставляется `http://<name>` с предупреждением, т.к. пустой namespace ломает реструктуризацию ИБ). `synonym` при отсутствии дефолтится в `name` (как EDT-визард), в ответе `synonymDefaulted`.
  - **Constant:** передавай `valueType` (тот же синтаксис, что `addObjectAttribute.type`: `Boolean`, `String(150)`, `Number(15,2)`, `Date`, `CatalogRef.X`, `DefinedType.Y`, …). Применяется к `Constant.type` + проставляются дефолты мастера EDT (`useStandardCommands=true`, `dataLockControlMode=Managed`, `minValue`/`maxValue`=Undefined). В ответе `valueTypeApplied`. Без `valueType` — `valueTypeApplied:false` + `warning` (создаётся String неогр.).
  - **`removeObject` preflight входящих ссылок** (по умолчанию `checkReferences=true`): перед удалением сканирует BM cross-ref индекс всех открытых проектов (типы реквизитов, defined types, измерения, `Subsystem.content`, `Right.object`, composite-типы). Если есть **внешние** ссылки (само-ссылки объекта и регистрация в `Configuration.mdo` отфильтрованы) и не задан `force=true` → `status:"blocked"`, объект **не удалён**, в `incomingReferences` список. `force=true` → удаляет и возвращает `danglingReferences`. `checkReferences=false` → пропустить проверку (быстрее при массовом удалении). **BSL-вызовы менеджеров (`Обработки.X`, `Метаданные.X`) preflight НЕ ловит** — проверяй `search_in_code` + Конфигуратор `/CheckConfig`.
- **Свойства:** `setObjectProperty` — `propertyPath` поддерживает простые пути (`comment`, `server`) и EMap (`synonym/ru`). Boolean/Integer/Enum автоматически парсятся; **UUID-свойства** (`ExchangePlan.thisNode`) принимаются строкой-guid; **ссылочные свойства** (`Document.defaultObjectForm`/`defaultListForm`, `Subsystem.parentSubsystem`, …) принимаются строкой-FQN и резолвятся в BM-объект (тип проверяется против EReferenceType). Пустое `value` для ссылки/UUID — сброс в null. `fqn` принимает nested-форму (см. «Конвенции параметров»): `Document.X.TabularSection.Y` для синонима ТЧ, `Catalog.X.Form.Y` для свойств формы и т.п. **`propertyPath="name"` для top-объекта** меняет имя только в BM-модели — каталог и `.mdo`-файл на диске остаются со старым именем, cross-references в `EventSubscription`/Form.form не обновляются. Для полноценного rename нужен `IRefactoringService` (см. известные особенности).
- **Реквизиты/ТЧ:** `addObjectAttribute`, `removeObjectAttribute` (с auto-cleanup полей форм), `addTabularSection`, `removeTabularSection`, `addTabularSectionAttribute`, `removeTabularSectionAttribute`. `cleanedFormItems` в ответе — структурированный массив `[{form, removed:[<itemName>, ...]}]`, имена удалённых FormField'ов нужны чтобы потом grep'ом найти оставшиеся ссылки в BSL. **Тип** (`type`): помимо `XxxRef.Y` / примитивов поддержаны платформенные value-типы по имени — `ValueStorage`/`ХранилищеЗначения`, `UUID`, `BinaryData`/`ДвоичныеДанные` (и др., резолвятся по существующему использованию в конфигурации — нужен донор). Нераспознанный тип теперь **не создаёт** «битый» typeless-реквизит (атомарность). Новый реквизит получает те же дефолты, что EDT-визард: `minValue`/`maxValue`=UndefinedValue, `dataHistory`/`fullTextSearch`=Use — иначе EDT дописывает их при следующем сохранении и даёт фантомный diff.
- **Регистры:** `addRegisterField`, `removeRegisterField` (`fieldKind=dimensions|resources|attributes`).
- **Enum:** `addEnumValue`, `removeEnumValue`.
- **Predefined items (Catalog/ChartOfCharacteristicTypes/ChartOfAccounts/ChartOfCalculationTypes):** `addPredefinedItem` (`fqn`, `name`, опц. `parentItem` для вложенного, `isFolder`, `code`, `description`, `type`), `removePredefinedItem` (`fqn`, `name` — рекурсивный поиск по всем уровням). `<Type>Predefined` контейнер создаётся лениво на первом add. Code пишется только если EClass хранит его как `EString` (CCT, ChartOfAccounts); для Catalog и ChartOfCalculationTypes тип code — `mcore.Value`, аргумент пока молча игнорируется. Для ChartOfCharacteristicTypes / ChartOfAccounts / ChartOfCalculationTypes передавай `type` (синтаксис как у `addObjectAttribute.type`) — тип предопределённого обязан входить в состав типов самого объекта, иначе обновление ИБ падает на реструктуризации (EDT-валидация это **не** ловит); без `type` в ответе `warning`.
- **Подсистемы:** `addSubsystemContent`, `removeSubsystemContent`. Вложенные подсистемы адресуются дотированным путём `Subsystem.<Родитель>.<Дочерняя>` (или каноническим BM-FQN из `list_metadata_objects.fqn`). **Создание вложенной подсистемы** — `createObject objectType=Subsystem` с параметром `parentFqn` (FQN родителя): ребёнок кладётся в containment родителя + `<parentSubsystem>`, экспортируется ребёнок и родитель (не Configuration.mdo). Без `parentFqn` — подсистема верхнего уровня.
- **Планы обмена (состав):** `addExchangePlanContent` (`fqn`=ExchangePlan, `targetFqn`=объект, опц. `autoRecord=Deny|Allow`), `removeExchangePlanContent`. `createObject objectType=ExchangePlan` теперь сам генерит `thisNode` (предопределённый узел «ЭтотУзел») — без него БСПП-обмены не работают; значение в ответе.
- **Команды объекта:** `createObjectCommand` (опц. `group` — имя стандартной группы как в `.mdo`: `NavigationPanelOrdinary`/`ActionsPanelTools`/`FormCommandBar`/…, резолвится донором среди существующих команд; без `group` группа пустая — наследование; проставляет `commandParameterType` и `representation=Auto`; генерит заготовку `Commands/<Имя>/CommandModule.bsl` с `ОбработкаКоманды`, путь в `commandModulePath`), `removeCommand` (папку модуля на диске **не** удаляет — чистить `Commands/<Имя>/` вручную).
- **Формы:** `addForm` (формат `formPurpose=Custom|Object|Group|List|Choice|RecordSet|Constants|Report`, `autoFillFromParent=true` для авто-заполнения через EDT `IFormFieldGenerator`), `removeForm`, `addFormItem` (`itemType=Field|Group|Table|Button|Decoration`, опц. `itemSubType`, `parentItem`, `dataPath` — создание идёт через EDT `IFormItemManagementService` + `IFormItemTypeManagementService`, т.е. тот же путь, что GUI-визард форм: для Table автоматически появляются `autoCommandBar`/`searchStringAddition`/`viewStatusAddition`/`searchControlAddition`, для FormField/Group/Decoration — наполненные `extInfo`, `extendedTooltip`, `contextMenu`; **явный `itemName` побеждает auto-prefix EDT** — если внутри Table EDT сам бы назвал колонку `<TableName><Attr>`, а ты передала это же имя, повторного префикса не будет; если EDT всё же изменил имя, оно вернётся в payload как `actualItemName`; **авто-дети поля** (`extendedTooltip`/`contextMenu`) переименовываются вслед за полем — двойного префикса нет; **Field внутри Table** по умолчанию получает `editMode=Enter` (как колонка EDT-визарда), перекрывается параметром `editMode`), `removeFormItem`, `setFormItemProperty`, `moveFormItem` (`position=top|bottom|before:X|after:X|<integer>`; **пустой `parentItem` = переставить внутри текущего контейнера**, а не вынести в корень формы — anchor по `before:X`/`after:X` и numeric index резолвятся в parent.items target'а), `addFormCommand`/`removeFormCommand`, `addFormAttribute`/`removeFormAttribute` (`type` поддерживает и form-only платформенные типы по имени — `ФорматированныйДокумент`/`FormattedDocument` и т.п., резолв по существующим формам), `addFormHandler` (Form/extInfo/Item — `target='Item'` через `itemName` для FormField/Table/Button-handlers; auto-заглушка в `Module.bsl`), `removeFormHandler` (симметрично, `itemName` принимает), `addDynamicListTable` (мастер «Добавить динамический список»).
- **Права:** `setRoleRight` (`rightName=Read|Update|Insert|Delete|View|...`, `value=true/false`).
- **Подписки на события:** `setEventSubscription` (source/event/handler), `addEventSubscriptionHandler` (генерация процедуры в CommonModule).
- **Определяемые типы:** `setDefinedTypeTypes` (`mode=replace|add|remove`, поддерживает `String(N)`, `Number(P,S)`, `Boolean`, `Date(DateOnly)`, `CatalogRef.X`).
- **Заимствование в расширение:** `adoptObject`/`adoptObjects` (top-объекты), `adoptChild` (реквизит/ТЧ/команда), `adoptModule` (CommonModule/Object/Manager/RecordSet/Form/Command). Через EDT `IModelObjectAdopter`.
- **Макеты:** `addTemplate` (Common или associated), `setTemplateCell`, `mergeTemplateCells`, `drawTemplate` (пакетное заполнение бланка).
- **СКД (`templateType=DataCompositionSchema`):** `createReportSchema` + `addDataSet` (Query/Object/Union) + `addDataSetField` + `addSchemaParameter` + `addCalculatedField` + `addTotalField` + `addDataSetLink` + `addUserField` + `addAutoField` + `addNestedSchema`.
- **Настройки СКД (variant.settings):** `addSettingsVariant`, `addSettingsSelectedField`, `addSettingsFilter`, `addSettingsOrder`, `addSettingsGroup`, `addSettingsTable`, `addSettingsChart`, `addConditionalAppearance`, `addAutoField`, `setOutputParameter`, `setVariantSettings` (bulk-upsert), `repairReportSchema` (диагностика + `autoFix=true` для удаления битых ссылок), `removeSettingsItem`, `removeDataSet`, `removeSchemaParameter`, `removeSchemaItem`.
- **Картинки:** `listPictures` (read-only каталог StdPicture платформы — substring-поиск EN/RU).

### Сборка и обновление ИБ (2)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `rebuild_project` | Полная пересборка проекта (`IProject.build(FULL_BUILD)`), опц. `cleanBuild=true`. После билда ждёт завершения build-job'ов (join MANUAL/AUTO_BUILD), чтобы счётчик маркеров был стабильным, а не транзиентно завышенным после cleanBuild. Возвращает счётчики Eclipse problem markers + customChecks. | `projectName`, опц. `cleanBuild`, `extendedChecks` |
| `sync_database` | Headless `Run → Update → Update Database`. Параметры `updateType=INCREMENTAL|FULL`, `skipIfUpdated`, `checkOnly`. **Требует UI Shell** (используется первый available). | `projectName`, опц. `applicationId`, `updateType`, `skipIfUpdated`, `checkOnly`, `wait_seconds` |

### yaxunit + отладка (16)

Цикл: `addBreakpoint` → `run_yaxunit mode=debug` → (на BP) `getState` → `getVariables` → `evaluate` → `resume` → `Done` или следующий BP → `get_yaxunit_report`.

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `addBreakpoint` | BSL line BP. Опц. условный (`condition` — BSL-выражение) или N-й проход (`hitCount`). | `modulePath`, `lineNumber`, опц. `condition`, `hitCount` |
| `removeBreakpoint` | Снять BP по `modulePath:line`. | `modulePath`, `lineNumber` |
| `listBreakpoints` | Все BSL line BP в workspace. | опц. `modulePath` |
| `run_yaxunit` | Запустить yaxunit поверх существующей RuntimeClient launch-config. Ответ: **Done** (тесты прошли, markdown-отчёт) / **Pending** (попал на BP — открывай debugger tools) / **started** (timeout — продолжай polling через `getState`). По умолчанию перед launch'ем делает headless `IApplicationManager.update` (то же что `sync_database`), чтобы EDT не показывал модалку «Обновить конфигурацию базы данных?» после правок метаданных/модулей. В ответе появляется блок `autoSync` с результатом. | `applicationId`, опц. `mode=run|debug`, `tests=["fqn1",...]`, `wait_seconds`, `autoSync=true` (default), `autoSyncTimeout=300` |
| `get_yaxunit_report` | Прочитать `junit.xml` от прошлого запуска (без повторного `run_yaxunit`). Используется в связке Pending → resume → get_yaxunit_report. | — |
| `getState` | Снимок debug-сессий: targets/threads/frames. Для каждого приостановленного — расположение (модуль, метод, line). | — |
| `getVariables` | Локалки кадра стека. | `threadId`, `frameIndex` |
| `evaluate` | Произвольное BSL-выражение через `IWatchExpressionDelegate`. Поддерживает function calls (`Константы.X.Получить()`), арифметику, переменные. Fallback — локальный lookup `Var.Prop.Sub`. | `threadId`, `frameIndex`, `expression` |
| `resume`, `suspend`, `stepOver`, `stepInto`, `stepReturn`, `terminate` | Управление debug — стандартный Eclipse Debug API. | опц. `threadId` |

### Документация (2)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `get_object_help` | Справка метаданного объекта: `synonym` (ru/en EMap), `comment`, и **HTML-страницы из Help.pages** (тот же mechanism, что `MdHelpContentFileEditor` в EDT UI). | `projectName`, `fqn`, опц. `includeContent=false` |
| `get_platform_docs` | Substring-поиск по дереву платформенной справки 1С (Синтакс-помощник) — типы, методы, свойства, глобальные функции. Источник — `satree.xml` версионного EDT-бандла `com._1c.g5.v8.dt.platform.doc_v8_X_Y`, выбранного по CompatibilityMode проекта (результат соответствует именно той платформе). Возвращает title, путь предков, `isCatalog`, `childCount`. | `query`, опц. `projectName`, `lang=ru\|en`, `limit` |

### XDTO (2)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `read_xdto_package` | JSON-дамп схемы XDTO-пакета (`Package.xdto`): `nsUri`, `objectTypes`, top-level `properties`, `valueTypes`, `dependencies`. Type-references рендерятся компактно: `xs:string` для XSD, голое имя для ссылок внутри пакета, `{ns}:Name` для cross-package. | `fqn` (например `XDTOPackage.ApdexExport`), опц. `projectName` |
| `edit_xdto_package` | Редактор схемы. `operation`: `setNamespace`, `addObjectType`/`removeObjectType`, `addObjectProperty`/`removeObjectProperty`, `addTopProperty`/`removeTopProperty`. Тип property — в той же компактной форме, что у reader'а. forceExport бьёт по двум FQN: `<fqn>` (для .mdo) и `<fqn>.Package` (для Package.xdto blob). Поддерживает `dryRun=true`. | `fqn`, `operation`, контекстные `typeName`/`propertyName`/`type`/`lowerBound`/`upperBound`/`form`/`nillable`/`namespace`/... |

## Типовые сценарии

### 1) Изучение незнакомой конфигурации

```
1. list_workspace_projects                      → выбери projectName
2. get_config_properties (projectName)          → vendor, compatibilityMode, objectCounts
3. list_metadata_objects (objectType="Catalog") → перечень справочников
4. get_object_details (fqn="Catalog.X")         → детали интересного объекта
5. list_modules (objectName="Catalog.X")        → его модули
6. read_module_source / get_module_structure    → почитать BSL
```

### 2) Чтение конкретного метода

```
1. get_module_structure (modulePath=...)        → найди диапазон строк
2. read_method_source  (modulePath, methodName) → точно метод
```

Используй `read_method_source`, а не `read_module_source` — контекст-эффективнее.

### 3) Найти всех, кто вызывает метод

```
1. get_method_call_hierarchy (methodName, modulePath, direction="callers", depth=3)
```

Для metadata-ссылок (где используется справочник) — `find_object_references`. Для текстового поиска по коду — `search_in_code`.

### 4) Запуск yaxunit с отладкой

```
1. addBreakpoint (modulePath, lineNumber)
2. run_yaxunit (applicationId, mode="debug", tests=["..."])
   → status=Pending — попал на BP
3. getState → возьми threadId/frameIndex приостановленного потока
4. getVariables (threadId, frameIndex=0)
5. evaluate (threadId, frameIndex=0, expression="Объект.Сумма + 100")
6. resume
7. (опц.) повторение или Done
8. get_yaxunit_report → markdown + summary
9. removeBreakpoint
```

При Done сразу из `run_yaxunit` отчёт уже в ответе — `get_yaxunit_report` нужен только после Pending → resume.

### 5) Добавить реквизит и поле формы

```
1. edit_metadata operation="addObjectAttribute"
     fqn="Catalog.Контрагенты", name="ИНН", type="String(12)"
2. edit_metadata operation="addFormItem"
     fqn="Catalog.Контрагенты", formName="ФормаЭлемента",
     itemType="Field", itemSubType="InputField",
     parentItem="ГруппаОсновное", dataPath="Объект.ИНН"
3. rebuild_project (projectName)
4. get_validation_errors (scope="object", fqn="Catalog.Контрагенты")
```

Перед мутацией прогони шаги 1-2 с `dryRun=true` — посмотри payload.

### 6) Создание отчёта на СКД

```
1. edit_metadata createObject (objectType="Report", name="ОтчётПродажи")
2. edit_metadata createReportSchema (parentFqn="Report.ОтчётПродажи", name="ОсновнаяСхема")
3. edit_metadata addDataSet (templateFqn="...", dataSetKind="Query", queryText="...")
4. edit_metadata addDataSetField (dataPath="Контрагент", ...)
5. edit_metadata addSettingsSelectedField (variantName="Основной", dataPath="Контрагент")
6. edit_metadata addSettingsGroup (groupFields=["Контрагент"])
```

После любого изменения схемы — `repairReportSchema` для диагностики битых ссылок.

### 7) Заимствование в расширение

```
1. edit_metadata adoptObject
     projectName="WMS", extensionName="WMS.ext",
     fqn="Catalog.Контрагенты"                  → top-объект
2. edit_metadata adoptChild
     parentFqn="Catalog.Контрагенты", childKind="Attribute", name="ИНН"
3. edit_metadata adoptModule
     targetFqn="Catalog.Контрагенты", moduleKind="ObjectModule"
```

Сначала parent, потом дети — иначе adopter'у некуда attach'ить.

### 8) Предопределённые элементы ПВХ / справочника / плана счетов

```
1. get_object_details (fqn="ChartOfCharacteristicTypes.ВидыДоступа")
   → читаем существующее дерево predefinedItems
2. edit_metadata addPredefinedItem
     fqn="ChartOfCharacteristicTypes.ВидыДоступа",
     name="MCP_TestFolder", isFolder=true, description="MCP folder"
3. edit_metadata addPredefinedItem
     fqn="ChartOfCharacteristicTypes.ВидыДоступа",
     parentItem="MCP_TestFolder", name="MCP_NestedItem", code="MCP02"
4. edit_metadata removePredefinedItem (для отката — рекурсивно найдёт по name)
```

Для `Catalog` и `ChartOfCalculationTypes` параметр `code` пока не пишется (тип `mcore.Value`). Полнотекстовое имя в режиме «Предприятие» задаётся через `description`, не `name`.

### 9) Проверка запроса перед коммитом

```
validate_query queryText="ВЫБРАТЬ ... ИЗ Справочник.X" isDcs=false
```

Для запросов в схеме СКД — `isDcs=true`. RU и EN ключевые слова поддержаны.

## Что плагин НЕ делает

- **Не выполняет произвольный BSL в продуктовом 1С** — только evaluate на приостановленном кадре под отладкой.
- **Не открывает 1С/EDT** — управляет уже открытым workspace.
- **Не валидирует semantic linking запросов к метаданным** — `validate_query` это только синтаксис + типовые QL-проверки.
- **Не редактирует BSL-код метода** — для этого пользователь редактирует файл сам / другой инструмент. Плагин может прочитать (`read_method_source`), найти (`search_in_code`), найти ссылки (`find_object_references` / `get_method_call_hierarchy`), но не редактирует тело процедуры.
- **Не управляет VCS** — git/svn вне scope.
- **Не показывает Form Designer как изображение** — `get_form_layout` отдаёт текстовое/JSON-дерево (для LLM это полезнее PNG).
- **Не покрывает покомпонентный adoption form-item'ов** — формы заимствуются целиком (`adoptObject fqn=...Form.X`), элементы внутри extension-формы добавляются через `addFormItem`.

## Известные особенности

- **Конфигурационная вьюшка кешируется в EDT.** После `sync_database` повторный вызов может вернуть stale `ApplicationUpdateState` — если важно, добавь короткую паузу или явный `checkOnly=true` повтор.
- **`removeObject` синхронно удаляет папку объекта на диске** (с 2026-05-21). Раньше требовался ручной `rm -rf src/cf/src/<Type>/<Name>/` после `removeObject` — теперь не нужен. Папка удаляется только если её имя совпадает с коротким именем объекта (защита от сноса flat-контейнеров). `persisted:true` в ответе подтверждает что и `.mdo` через `forceExport`, и папка ушли. Для verify — `folderDeleted:true` + `folderPath` в ответе.
- **`find_object_references` сканирует все проекты workspace** (cf + cfe). Покрывает: BM cross-refs от target и его children (StandardAttribute/Attribute/TabularSection — `Right.objectAttribute` ссылки находятся через child URIs), composite-types (`<types>CatalogRef.X</types>` — UUID-scan `TypeItem.compositeId`). Каждая ссылка несёт `projectName`+`filePath`. Что **пока не покрыто**: `Subsystem.content`, `CommandInterface.visibilityFragments.command`, BSL-литералы внутри Extension-проектов (`search_in_code` workspace-wide для этого случая остаётся обязательным).
- **Extension-проекты в write-операциях.** Все операции `edit_metadata` принимают и Configuration-, и Extension-проект в `projectName` (`isExtension:true` в ответе для cfe). Примитивные типы (`String/Number/Boolean/Date`) в реквизитах расширения резолвятся через базовый проект. Заимствование — `adoptObject`/`adoptChild`/`adoptModule` (через `extensionName`); формы заимствуются целиком (`adoptObject fqn=...Form.X`).
- **`setObjectProperty propertyPath="name"` для top-объекта НЕ доводится до диска.** BM-модель и `<name>` внутри `.mdo` обновляются, но имя файла, каталог объекта, cross-references в `EventSubscription.source` / `Form.form` (`<types>DocumentObject.X</types>`) и BSL-литералы (`Документ.X`, `Документы.X`, `ДокументСсылка.X`, ...) остаются со старым именем. `list_metadata_objects` может отдать новый name + старый fqn (рассогласованная запись). Полноценный rename требует `IRefactoringService.initiateRename` с custom `IRefactoringOperation`, который пока не реализован. Workaround: после rename вручную `git mv` каталог+`.mdo` + `search_in_code`+regex по BSL-литералам.
- **Write-операции `edit_metadata` могут таймаутить на стороне MCP-клиента**, но фактически записаться в BM. Повтор «вслепую» создаст дубль — после таймаута сначала read-проверка (`get_form_layout`, `get_object_details`).
- **`evaluate` требует приостановленный кадр.** Без BP — нельзя.
- **`HTTP 200 + JSON-RPC error -32603`** — реальная ошибка инструмента (например NoClassDefFoundError если bundle не пере-stage'нулся после изменения Require-Bundle). Не игнорируй status=200 — всегда смотри `error` в теле.
- **Сравни `get_validation_errors` с `rebuild_project.errors`** — это **разные** marker store: первое — EDT validation markers, второе — стандартные Eclipse problem markers (типы, разрешение ссылок). Оба бывают полезны.

## Минимальный health-check

```
curl http://127.0.0.1:6868/health
→ {"status":"ok","tools":39}
```

Если 404 / connection refused → EDT не запущен или bundle не активирован (при первой установке — разовый `-clean` рестарт).
