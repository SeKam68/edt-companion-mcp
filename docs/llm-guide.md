# edt-companion-mcp — гайд для LLM-агента

Этот файл — справочник для нейросетевого агента (Claude, Cursor, Cline, ...), который подключён к плагину через MCP. Прочти его один раз перед работой. Установка и подключение описаны в [README](../README.md).

## Что это

OSGi-плагин для 1C:EDT 2025.2 / 2026.1, который поднимает локальный MCP-сервер `http://127.0.0.1:6868/mcp` и отдаёт **46 инструментов** для работы с открытой в EDT конфигурацией 1С: чтение метаданных и BSL-кода, навигация, поиск, валидация и быстрые исправления, редактирование метаданных и BSL-модулей, запуск и отладка yaxunit-тестов, профилирование, проверка запросов (с резолвом метаданных), поиск в платформенной документации.

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

Протокол — JSON-RPC 2.0. Поддержаны методы `initialize`, `tools/list`, `tools/call`. Проверка живости: `GET http://127.0.0.1:6868/health` → `{"status":"ok","tools":46}`.

### Два экземпляра EDT с разными проектами

Плагин работает только над workspace того экземпляра EDT, внутри которого запущен. Чтобы вести два EDT с разными проектами одновременно, каждому нужно выделить свой порт и завести отдельный сервер в `.mcp.json`. Готовая пошаговая инструкция — [docs/multi-instance.md](multi-instance.md).

## Конвенции параметров (важно)

- **FQN метаданного объекта** — Java-формат: `Catalog.Контрагенты`, `Document.РеализацияТоваровУслуг`, `InformationRegister.ЦеныНоменклатуры`, `Enum.СтатусыЗаказов`, `Constant.ВалютаУчёта`, `CommonModule.ОбщегоНазначения`. Имена самих объектов — как в конфигурации (русские/английские, регистрозависимые).
- **FQN формы** — `Catalog.X.Form.ФормаЭлемента`, `Document.Y.Form.ФормаСписка`.
- **FQN макета** — `Catalog.X.Template.Печать` (associated) или `CommonTemplate.УниверсальныйМакет` (top-level).
- **Nested FQN** в `edit_metadata.setObjectProperty` — пары `<Kind>.<Name>` после top: `Document.X.TabularSection.Y`, `Document.X.TabularSection.Y.Attribute.Z`, `Catalog.X.Form.Y`, `Catalog.X.Template.Z`, `AccumulationRegister.X.Dimension.D`, `Enum.X.EnumValue.V`. Поддержаны kind'ы: `TabularSection`, `Attribute`, `Form`, `Template`, `Command`, `Dimension`, `Resource`, `EnumValue`, `AccountingFlag`, `ExtDimensionAccountingFlag`, `AddressingAttribute`, `Column`, `Operation`, `Recalculation`. `forceExport` всегда бьёт по top-FQN.
- **workspacePath модуля BSL** — путь от корня workspace через `/`, начинается со слэша: `/Демо/src/CommonModules/ОбщегоНазначения/Module.bsl`, `/Демо/src/Catalogs/Контрагенты/ObjectModule.bsl`, `/Демо/src/Catalogs/Контрагенты/Forms/ФормаЭлемента/Module.bsl`. **Это не путь в git-репозитории:** корень проекта конфигурации соответствует каталогу `src/cf` репозитория, поэтому сегмента `cf/src` в workspace-пути нет (у расширения — аналогично `src/cfe/<Имя>`). Путь в репозиторной раскладке инструменты примут и сами приведут к workspace-виду (в ответе `pathNormalizedFrom` + `warning`), но полагаться на это не стоит — надёжнее адресоваться парой `objectName` + `moduleType`.
- **`projectName`** — имя проекта в EDT workspace (получи через `list_workspace_projects`), не FQN.
- **`applicationId`** — отображаемое имя 1С Run Configuration (`list_applications`), не technical ID.
- **`dryRun: true`** поддержан большинством мутирующих операций — выполни сначала в dry-run, посмотри payload, потом без флага.

Если не уверен в имени проекта/FQN/пути модуля — **сначала вызови соответствующий list/get**, не угадывай.

## Формат результата и коды отказов

Успех и провал читаются одинаково: полезная нагрузка — JSON в `content[0].text`, разбирай его в обоих случаях.

- **Успех** — объект с полями операции и `"status": "ok"`.
- **Провал** — на уровне результата стоит `"isError": true`, а в `content[0].text` лежит объект:

```json
{"status":"error","code":"object_not_found","message":"addDynamicListTable: форма 'ФормаСписка' не найдена в 'Catalog.Демо'","operation":"addDynamicListTable","projectName":"Демо"}
```

`message` — человеческий текст, его формулировка может меняться: **не разбирай его регэкспом**. Ветвись по `code`. Набор кодов закрытый:

| Код | Что значит | Что делать агенту |
|---|---|---|
| `invalid_argument` | аргумент не передан, пуст или неверен | исправить вызов |
| `object_not_found` | объекта/формы/элемента по адресу нет | уточнить FQN через list/get |
| `project_not_found` | проекта нет в workspace или он закрыт | `list_workspace_projects` |
| `name_taken` | имя занято соседом | взять другое имя |
| `already_exists` | такой объект уже есть, ничего не менялось | считать состояние достигнутым |
| `not_supported_for_extension` | операция не для проекта-расширения | заимствовать объект (`adoptObject`) |
| `not_supported` | вид объекта или операции вне поддержанного набора | другой путь |
| `service_unavailable` | нужный сервис EDT не поднят | среда, не аргументы — сообщить пользователю |
| `needs_rebuild` | модель отстала от диска | `refresh_workspace` / `rebuild_project` |
| `locked_by_support` | объект закрыт правилами поддержки поставщика | делать расширением |
| `invalid_state` | отладка/ИБ/запуск не в нужном состоянии | привести состояние в порядок |
| `timeout` | не уложились в отведённое время | повторить или увеличить бюджет |
| `internal_error` | сбой внутри EDT/платформы | не вина вызова, сообщить пользователю |
| `unspecified` | вид отказа не классифицирован | читать `message` |

`unspecified` означает ровно «не классифицировано», а не «неизвестная ошибка» — такие места размечаются по мере работы. Большинство объектов имеют русские имена и регистр важен.

## Каталог инструментов

### Workspace и среда (3)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `list_workspace_projects` | Какие проекты открыты в EDT, какой тип (Configuration/Extension/External), какая совместимость. Старт любого диалога. | — |
| `list_applications` | 1С Run Configurations типа RuntimeClient (`applicationId` для запуска) **плюс** секция `infobases` — сами информационные базы проектов (базу заводят в проекте, конфигурация лишь способ её запустить; на базу может не быть ни одной конфигурации, и тогда иначе её не видно). По каждой конфигурации: `infobaseBound` (годна ли для запуска), `applicationName`, `lifecycleState`, `infobase` (строка соединения — **куда физически уйдёт запуск**) и `problem` с причиной непригодности. | — |
| `show_edt_version` | Версии EDT/Eclipse/Java, открытые проекты с `kind`, `compatibilityMode`, `baseProject` для расширений. | — |

### Метаданные — чтение (6)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `list_metadata_objects` | Перечислить top-level объекты конфигурации (с фильтрами по типу/regexp). Принимает и Configuration- (cf), и Extension-проект (cfe). | `projectName` (cf/cfe), опц. `objectType` (EClass), `namePattern` (Java regex, case-insensitive), `limit` |
| `list_modules` | BSL-модули объекта или всех объектов; фильтр по виду модуля (`kind`: `commonModule`/`managerModule`/`objectModule`/`recordSetModule`/`valueManagerModule`/`commandModule`). Принимает и Configuration- (cf), и Extension-проект (cfe). | `projectName` (cf/cfe), опц. `objectName`, `kindFilter` |
| `get_object_details` | Принимает и Configuration-, и Extension-проект. Полная структура: реквизиты с типами, ТЧ, измерения/ресурсы, формы, команды, макеты, модули. Скалярные флаги карточки объекта (отдаются с учётом дефолта платформы, даже когда тега нет в `.mdo`): `fullTextSearch`, `useStandardCommands`, `includeInCommandInterface`, `dataLockControlMode`; для Document — `numberType`/`numberLength`/`numberPeriodicity`/`postInPrivilegedMode`/...; для Catalog — `codeType`/`codeLength`/`hierarchical`/`hierarchyType`/...; ссылочные списки `owners`/`basedOn` (массив FQN). Для регистров — `informationRegisterPeriodicity`/`writeMode`/`registerType`/... (от них зависит виртуальная таблица `СрезПоследних`/`Остатки`/...). Для Catalog/ChartOfCharacteristicTypes/ChartOfAccounts/ChartOfCalculationTypes — `predefinedItems` (дерево). Через FQN. | `projectName`, `fqn` |
| `get_config_properties` | Шапка конфигурации: vendor, version, compatibilityMode, scriptVariant + режим запуска (`defaultRunMode`, `useManagedFormInOrdinaryApplication`, `interfaceCompatibilityMode`, `modalityUseMode`, `synchronousPlatformExtensionAndAddInCallUseMode`), `formCounts` (managed/ordinary) и `objectCounts` по всем типам. | `projectName` |
| `get_form_layout` | Принимает и Configuration-, и Extension-проект. Headless-дамп управляемой формы (без открытия редактора): `attributes`, `items` рекурсивно, `formCommands`, `handlers`. По **обычной** форме отдаёт `layoutAvailable:false` + причину и `modulePath` (см. «Обычные формы»), пустых секций не выводит. | `projectName`, `fqn` (родитель), `formName`, опц. `format=tree|json`, `includeHandlers`, `maxDepth` |
| `get_form_screenshot` | **PNG-рендер** формы из WYSIWYG-редактора EDT (визуально «увидеть» раскладку — перекрытия, пустоты, реальные размеры) в дополнение к структурному `get_form_layout`. Открывает `.form` в редакторе, ждёт асинхронную загрузку (до ~минуты) и снимает буфер. Ответ — MCP `image` (PNG base64) + текстовый спутник (`width`/`height`/`renderedForm`). **Требует** активного workbench (не headless) и native-buffered рендера: в `1cedt.ini` после `-vmargs` добавить `-DnativeFormLayoutRender=true` и `-DnativeFormBufferedLayoutRender=true`, иначе буфер пуст. | `projectName`, `fqn` (родитель), `formName` |

### Навигация по BSL-коду (5)

Все 5 видят **несохранённые правки** из открытых редакторов EDT (через BM read-transaction).

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `read_module_source` | Текст BSL-модуля с построчной пагинацией. Большие модули (БСП 1500+ строк) не валятся по token-лимиту: без `limit` возвращается первая порция ≤~48k символов с `truncated:true` + `nextOffset` для дочитывания. В ответе `totalLines`/`totalChars`/`returnedLines`. Адресация: `modulePath` **или** `objectName` (`Type.Name`) + `moduleType` (+ `projectName`) — путь резолвится сам (для CommonModule `moduleType` необязателен). | `modulePath` \| `objectName`+`moduleType`, опц. `offset`/`limit`, `maxChars`, `projectName` |
| `get_module_structure` | Процедуры/функции (имя, kind, export, async, параметры, диапазон строк) + регионы. Сначала структура — потом точечно `read_method_source`. Адресация: `modulePath` **или** `objectName`+`moduleType` (+ `projectName`), как у `read_module_source`. | `modulePath` \| `objectName`+`moduleType`, опц. `projectName` |
| `read_method_source` | Текст одной процедуры/функции — экономит контекст vs `read_module_source`. | `modulePath`, `methodName` |
| `search_in_code` | Workspace-wide или per-project текстовый/regex-поиск по `*.bsl` **и по модулям обычных форм внутри `Form.oform`** (совпадения помечены `source:"oform"`, `openable:false`). До 500 матчей. | `query`, опц. `projectName`, `caseSensitive`, `regex`, `limit` |
| `resolve_symbol` | Резолв символа по позиции (line+column или offset) к декларации. | `modulePath`, `line`+`column` или `offset` |

### Запись BSL (1)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `write_module_source` | Запись BSL-модуля через **shared Eclipse text buffer** — попадает в тот же dirty buffer, который видит открытый Xtext-редактор EDT. То есть: если у пользователя открыт редактор и в нём несохранённые правки, наша запись применяется поверх dirty state (а не перезатирает диск конфликтом). 6 режимов: `replace` (полная замена/создание файла), `append`, `insertBefore`/`insertAfter` (по `line`, 1-based), `replaceLines` (`startLine`+`endLine` inclusive), `searchReplace` (literal find→text + `expectMatches` контроль безопасности — по умолчанию ровно 1 совпадение; **переводы строк `find`/`text` нормализуются к делимитеру модуля** — многострочный LF-фрагмент из read-инструментов корректно матчится к CRLF-модулю и не подмешивает LF). Параметр `save` (default `true`) — commit buffer на диск; `false` оставляет в editor dirty state, пользователь сохранит Ctrl+S. `dryRun` для what-if. В ответе `oldLength`/`newLength`/`startLine`/`endLine`/`dirty`/`saved`/`created`. **Tier: PRO**. | `modulePath`, `mode`, `text`, опц. `line`/`startLine`+`endLine`/`find`+`expectMatches`/`save`/`dryRun` |

### Анализ кода (5)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `find_object_references` | Где используется метаданный объект. Сканирует **все** открытые Configuration и Extension проекты workspace, агрегирует matches с полями `projectName` / `filePath` / `ownerFqn`. Покрывает: BM cross-refs от самого target и его children (StandardAttribute/Attribute/TabularSection — для `Right.objectAttribute`), composite-types (`<types>CatalogRef.X</types>` через UUID-scan `TypeItem`). `kindFilter=code|metadata` — фильтр. | `objectName`, опц. `projectName` (origin для поиска target), `kindFilter`, `limit` |
| `get_method_call_hierarchy` | Callers/callees BSL-метода до depth=5. Записи `hierarchy[]`: `level`, `from`, `caller`/`callee`, `modulePath`, `line`, `qualifier`. **Граф строится не по модели** (`Method.callers`/`callees` объявлены `transient` и при чтении через BM read-транзакцию пусты): границы методов — из AST (несохранённые правки видны), имена — по индексу модулей workspace. `direction=callers` ищет вызовы по всем открытым проектам, `projectName` сужает область. `direction=callees` дополнительно отдаёт `unresolvedCalls` — то, что намеренно не отнесено к BSL-методу (глобальные функции платформы, вызовы через переменную/менеджер объекта). | `methodName`, опц. `projectName`, `modulePath`, `direction=callers|callees`, `depth`, `limit` |
| `get_validation_errors` | Маркеры EDT-валидации (НЕ Eclipse problems). Принимает и Configuration-, и Extension-проекты. Перед чтением ждёт готовности модели маркеров (`waitSeconds`, дефолт 20); в ответе **`markersReady`** — при `false` пустой набор не означает «замечаний нет». | `projectName`, опц. `scope=project|object`, `fqn`, `minSeverity=BLOCKER|CRITICAL|MAJOR|MINOR|TRIVIAL`, `waitSeconds` |
| `get_check_description` | Описание проверки EDT по `checkId` (из `get_validation_errors`) или короткому коду `SU..`: `title`, `severity`, `type`, `shortUid`, параметры и разобранное описание — `rationale`, `wrongExample`, `rightExample`, `sources[]` (плюс плоский `description`). Источник — встроенный `ICheckRepository`, всё офлайн (код наружу не уходит, в отличие от `v8std_explain_diagnostics`). **`descriptionAvailable:false`** — в бандле EDT есть только заголовок (частый случай у проверок метаданных `com.e1c.v8codestyle.md`), идти во внешний источник стандартов. Для BSL compile-маркеров (Syntax/undefined) описания в репозитории нет вовсе. | `checkId`, опц. `projectName` |
| `apply_quick_fix` | Применяет быстрое исправление EDT к **model-маркеру** из `get_validation_errors` (source=model, проверки SU*). `dryRun=true` (по умолчанию) — перечислить применимые варианты (`variants[].index/description/details`), ничего не меняя; `dryRun=false`+`variantIndex` — применить. `markerObjectId` — из ответа `get_validation_errors`. Через штатный `IFixManager` (`com.e1c.g5.v8.dt.check.qfix`). Builder-маркеры (Syntax/undefined, source=builder) не поддержаны — программного fix-API нет. **Tier: PRO**. | `markerObjectId`, опц. `projectName`, `checkId`, `variantIndex`, `dryRun` |

### Валидация запросов (1)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `validate_query` | Проверка текста запроса штатным Xtext-парсером EDT (RU/EN ключевые слова). Без `projectName` — только синтаксис + типовые QL-проверки. С существующим открытым `projectName` — **metadata-aware**: ссылки на таблицы/поля конфигурации (`Справочник.X` и т.п.) резолвятся по BM-индексу проекта (нерезолвленная таблица → ошибка). В ответе флаг `metadataAware`. | `queryText`, опц. `isDcs=true` (грамматика QlDcs), `projectName` |

### Редактирование метаданных (3)

Универсальный диспетчер `edit_metadata` принимает поле `operation`. Все операции в read-write BM-транзакции, поддерживают `dryRun=true`. После коммита плагин синхронно пишет `.mdo` через `forceExport`.

`edit_metadata.operation` (выбор):

- **Создание/удаление объектов:** `createObject` (Catalog/Document/InformationRegister/AccumulationRegister/Enum/Constant/Report/DataProcessor/CommonModule/DocumentJournal/ChartOfCharacteristicTypes/ExchangePlan/Subsystem/Role/DefinedType/EventSubscription/CommonTemplate/XDTOPackage), `removeObject` — синхронно чистит и BM, и `Configuration.mdo`, и **папку объекта на диске** (`src/cf/src/<Type>/<Name>/` со всеми подкаталогами Forms/Templates/Commands). В ответе `folderDeleted`+`folderPath`; `persisted:true` = удаление доведено до диска, то есть снята регистрация в корне (или в родительской подсистеме для вложенной) **и** убрана папка объекта — единый смысл поля со всеми остальными операциями. Принимает и Configuration-, и Extension-проекты (`isExtension:true` в ответе для cfe). Для `Role` пишется skeleton `Rights.rights`, для `XDTOPackage` — skeleton `Package.xdto`, для `CommonModule` — пустой `src/CommonModules/<Имя>/Module.bsl` (иначе EDT считает модуль отсутствующим и писать код некуда; в ответе `moduleFilePath`/`moduleFileCreated`); namespace задаётся параметром `namespace` (пишется и в `XDTOPackage.namespace`, и в `targetNamespace` скелета; если не передан — подставляется `http://<name>` с предупреждением, т.к. пустой namespace ломает реструктуризацию ИБ). `synonym` при отсутствии дефолтится в `name` (как EDT-визард), в ответе `synonymDefaulted`.
- **Каскадное переименование:** `renameObject` (`fqn`, `newName`, опц. `dryRun`) — через РОДНОЙ EDT-рефакторинг (`IMdRefactoringService`): согласованно обновляет `.mdo`, каталог/файл на диске, `Configuration.mdo`, `Subsystem.content`, `Rights.rights`, типизированные ссылки в BSL/формах/`.dcs` и adopted-двойники в подключённых расширениях. `dryRun=true` → превью (`refactoringCount`, `affectedItems`, `hasProblems`) без применения; `dryRun=false` → применяет пакетно через LTK (быстро, без полной «Сортировки объектов метаданных»). Это НЕ то же, что `setObjectProperty propertyPath=name` (тот меняет только тег `<name>`).
  - **CommonModule:** контекст компиляции задаётся параметром `commonModuleType` — `Server` (server + externalConnection + clientOrdinaryApplication), `ServerCall` (serverCall + server), `Client` (оба клиентских), `ClientServer` (server + externalConnection + оба клиентских); плюс опционально `global`, `privileged` (`returnValuesReuse` — через `setObjectProperty`). Без параметра тип **выводится из постфикса имени** (`*ВызовСервера` / `*КлиентСервер` / `*Клиент` / `*Глобальный`, иначе `Server`) и в ответе приходит `warning` + `commonModuleTypeApplied`/`commonModuleTypeSource`. Набор флагов обязателен: модуль со всеми `false` не соответствует ни одному допустимому типу и сразу даёт BLOCKER `SU186` (`com.e1c.v8codestyle.md:common-module-type`) — при этом формулировка диагностики перечисляет флаги так, будто их надо **снять**, хотя нужно наоборот включить (стандарт std469 п. 1.2).
  - **Constant:** передавай `valueType` (тот же синтаксис, что `addObjectAttribute.type`: `Boolean`, `String(150)`, `Number(15,2)`, `Date`, `CatalogRef.X`, `DefinedType.Y`, …). Применяется к `Constant.type` + проставляются дефолты мастера EDT (`useStandardCommands=true`, `dataLockControlMode=Managed`, `minValue`/`maxValue`=Undefined). В ответе `valueTypeApplied`. Без `valueType` — `valueTypeApplied:false` + `warning` (создаётся String неогр.).
  - **`removeObject` preflight входящих ссылок** (по умолчанию `checkReferences=true`): перед удалением сканирует BM cross-ref индекс всех открытых проектов (типы реквизитов, defined types, измерения, `Subsystem.content`, `Right.object`, composite-типы). Если есть **внешние** ссылки (само-ссылки объекта и регистрация в `Configuration.mdo` отфильтрованы) и не задан `force=true` → `status:"blocked"`, объект **не удалён**, в `incomingReferences` список. `force=true` → удаляет и возвращает `danglingReferences`. `checkReferences=false` → пропустить проверку (быстрее при массовом удалении). **BSL-вызовы менеджеров (`Обработки.X`, `Метаданные.X`) preflight НЕ ловит** — проверяй `search_in_code` + Конфигуратор `/CheckConfig`.
- **Свойства:** `setObjectProperty` — `propertyPath` поддерживает простые пути (`comment`, `server`) и EMap (`synonym/ru`). Boolean/Integer/Enum автоматически парсятся; **UUID-свойства** (`ExchangePlan.thisNode`) принимаются строкой-guid; **ссылочные свойства** (`Document.defaultObjectForm`/`defaultListForm`, `Subsystem.parentSubsystem`, …) принимаются строкой-FQN и резолвятся в BM-объект (тип проверяется против EReferenceType). Пустое `value` для ссылки/UUID — сброс в null. `fqn` принимает nested-форму (см. «Конвенции параметров»): `Document.X.TabularSection.Y` для синонима ТЧ, `Catalog.X.Form.Y` для свойств формы и т.п. **`propertyPath="name"` для top-объекта** меняет имя только в BM-модели — каталог и `.mdo`-файл на диске остаются со старым именем, cross-references в `EventSubscription`/Form.form не обновляются. Для полноценного rename нужен `IRefactoringService` (см. известные особенности).
- **Реквизиты/ТЧ:** `addObjectAttribute`, `removeObjectAttribute` (с auto-cleanup полей форм), `addTabularSection`, `removeTabularSection`, `addTabularSectionAttribute`, `removeTabularSectionAttribute`. `cleanedFormItems` в ответе — структурированный массив `[{form, removed:[<itemName>, ...]}]`, имена удалённых FormField'ов нужны чтобы потом grep'ом найти оставшиеся ссылки в BSL. **Тип** (`type`): помимо `XxxRef.Y` / примитивов поддержаны платформенные value-типы по имени — `ValueStorage`/`ХранилищеЗначения`, `UUID`, `BinaryData`/`ДвоичныеДанные` (и др., резолвятся по существующему использованию в конфигурации — нужен донор). **Квалификаторы примитивов** задаются в скобках: `String(N)`, `String(N,fixed)`, `Number(P,S)`, `Number(P,S,nonneg)` и **состав даты** — `Date(Date)` только дата, `Date(Time)` только время, голый `Date` = дата и время. Голый `Date` для реквизита-«срока»/«даты документа» почти всегда неверен — на форме появится ввод времени; пиши `Date(Date)`. В `.mdo` `DateTime` не сериализуется (это EMF-дефолт), поэтому пустой `<dateQualifiers/>` и означает «дата и время». Нераспознанный тип теперь **не создаёт** «битый» typeless-реквизит (атомарность). Новый реквизит получает те же дефолты, что EDT-визард: `minValue`/`maxValue`=UndefinedValue, `dataHistory`/`fullTextSearch`=Use — иначе EDT дописывает их при следующем сохранении и даёт фантомный diff.
- **Регистры:** `addRegisterField`, `removeRegisterField` (`fieldKind=dimensions|resources|attributes`).
- **Enum:** `addEnumValue`, `removeEnumValue`.
- **Predefined items (Catalog/ChartOfCharacteristicTypes/ChartOfAccounts/ChartOfCalculationTypes):** `addPredefinedItem` (`fqn`, `name`, опц. `parentItem` для вложенного, `isFolder`, `code`, `description`, `type`), `removePredefinedItem` (`fqn`, `name` — рекурсивный поиск по всем уровням). `<Type>Predefined` контейнер создаётся лениво на первом add. Code пишется только если EClass хранит его как `EString` (CCT, ChartOfAccounts); для Catalog и ChartOfCalculationTypes тип code — `mcore.Value`, аргумент пока молча игнорируется. Для ChartOfCharacteristicTypes / ChartOfAccounts / ChartOfCalculationTypes передавай `type` (синтаксис как у `addObjectAttribute.type`) — тип предопределённого обязан входить в состав типов самого объекта, иначе обновление ИБ падает на реструктуризации (EDT-валидация это **не** ловит); без `type` в ответе `warning`.
- **Подсистемы:** `addSubsystemContent`, `removeSubsystemContent`. Вложенные подсистемы адресуются дотированным путём `Subsystem.<Родитель>.<Дочерняя>` (или каноническим BM-FQN из `list_metadata_objects.fqn`). **Создание вложенной подсистемы** — `createObject objectType=Subsystem` с параметром `parentFqn` (FQN родителя): ребёнок кладётся в containment родителя + `<parentSubsystem>`, экспортируется ребёнок и родитель (не Configuration.mdo). Без `parentFqn` — подсистема верхнего уровня.
- **Планы обмена (состав):** `addExchangePlanContent` (`fqn`=ExchangePlan, `targetFqn`=объект, опц. `autoRecord=Deny|Allow`), `removeExchangePlanContent`. `createObject objectType=ExchangePlan` теперь сам генерит `thisNode` (предопределённый узел «ЭтотУзел») — без него БСПП-обмены не работают; значение в ответе.
- **Команды объекта:** `createObjectCommand` (опц. `group` — имя стандартной группы как в `.mdo`: `NavigationPanelOrdinary`/`ActionsPanelTools`/`FormCommandBar`/…, резолвится донором среди существующих команд; без `group` группа пустая — наследование; проставляет `commandParameterType` и `representation=Auto`; генерит заготовку `Commands/<Имя>/CommandModule.bsl` с `ОбработкаКоманды`, путь в `commandModulePath`), `removeCommand` (папку модуля на диске **не** удаляет — чистить `Commands/<Имя>/` вручную).
- **Формы:** `addForm` (формат `formPurpose=Custom|Object|Group|List|Choice|RecordSet|Constants|Report`, `autoFillFromParent=true` для авто-заполнения через EDT `IFormFieldGenerator`), `removeForm`, `addFormItem` (`itemType=Field|Group|Table|Button|Decoration`, опц. `itemSubType`, `parentItem`, `dataPath` — создание идёт через EDT `IFormItemManagementService` + `IFormItemTypeManagementService`, т.е. тот же путь, что GUI-визард форм: для Table автоматически появляются `autoCommandBar`/`searchStringAddition`/`viewStatusAddition`/`searchControlAddition`, для FormField/Group/Decoration — наполненные `extInfo`, `extendedTooltip`, `contextMenu`; **явный `itemName` побеждает auto-prefix EDT** — если внутри Table EDT сам бы назвал колонку `<TableName><Attr>`, а ты передала это же имя, повторного префикса не будет; если EDT всё же изменил имя, оно вернётся в payload как `actualItemName`; **авто-дети поля** (`extendedTooltip`/`contextMenu`) переименовываются вслед за полем — двойного префикса нет; **Field внутри Table** по умолчанию получает `editMode=Enter` (как колонка EDT-визарда), перекрывается параметром `editMode`), `removeFormItem`, `setFormItemProperty`, `moveFormItem` (`position=top|bottom|before:X|after:X|<integer>`; **пустой `parentItem` = переставить внутри текущего контейнера**, а не вынести в корень формы — anchor по `before:X`/`after:X` и numeric index резолвятся в parent.items target'а), `addFormCommand`/`removeFormCommand`, `addFormAttribute`/`removeFormAttribute` (`type` поддерживает и form-only платформенные типы по имени — `ФорматированныйДокумент`/`FormattedDocument` и т.п., резолв по существующим формам), `addFormHandler` (Form/extInfo/Item — `target='Item'` через `itemName` для FormField/Table/Button-handlers; auto-заглушка в `Module.bsl`), `removeFormHandler` (симметрично, `itemName` принимает), `addDynamicListTable` (мастер «Добавить динамический список»; Table создаётся тем же `IFormItemManagementService`, что и в `addFormItem`, — со служебными блоками `autoCommandBar`/`searchStringAddition`/`viewStatusAddition`/`searchControlAddition`, форма списка сразу пригодна к открытию). Главный реквизит (`main=true`) получает `savedData=true` только когда форма действительно хранит его данные: для `DynamicList` и `ConstantsSet` флаг не ставится (у формы констант он давал вопрос «Данные были изменены. Сохранить?» при закрытии).
- **Права:** `setRoleRight` (`rightName=Read|Update|Insert|Delete|View|...`, `value=true/false`). `targetFqn` принимает и **`Configuration`/`Configuration.<Имя>`** — права уровня конфигурации (`MainWindowModeNormal`, `DataAdministration`, `ThinClient`, `ExternalReports`, …), в `Rights.rights` они лежат обычным `<object>` с именем `Configuration.<Имя>`, тогда как в BM корень зовётся просто `Configuration`. Глобального реестра прав нет — право должно уже встречаться в какой-нибудь роли конфигурации; при неизвестном имени в ошибке приходит список прав, реально применяемых к объектам этого класса. Роль, только что созданная через `createObject`, пригодна к `setRoleRight` сразу: описание прав подтягивается автоматически (в ответе `rightsImportedFromDisk`), ручной `refresh_workspace` между ними больше не нужен.
- **Подписки на события:** `setEventSubscription` (source/event/handler), `addEventSubscriptionHandler` (генерация процедуры в CommonModule).
- **Определяемые типы:** `setDefinedTypeTypes` (`mode=replace|add|remove`, поддерживает `String(N)`, `Number(P,S)`, `Boolean`, `Date(Date)`, `CatalogRef.X`).
- **Заимствование в расширение:** `adoptObject`/`adoptObjects` (top-объекты), `adoptChild` (реквизит/ТЧ/команда), `adoptModule` (CommonModule/Object/Manager/RecordSet/Form/Command). Через EDT `IModelObjectAdopter`.
- **Макеты:** `addTemplate` (Common или associated), `setTemplateCell`, `mergeTemplateCells`, `drawTemplate` (пакетное заполнение бланка), `setTemplateBorders`, `setTemplateSizes`, `setTemplatePrintSettings`. Для `templateType=BinaryData` содержимое `Template.bin` можно передать сразу — `contentPath` (путь к файлу) или `contentBase64`; иначе создаётся только `.mdo`. **Координаты:** у `setTemplateCell` и `rect` у `mergeTemplateCells` — 1-based (как в редакторе EDT), у `drawTemplate` — 0-based (как в самом `Template.mxlx`). Объединение области из одной ячейки платформа не хранит: `mergeTemplateCells` отвечает отказом, `drawTemplate` считает такие в `mergesSkipped`. `bold`/`italic`/`underline` заводят шрифт в списке шрифтов листа, `fontName`/`fontSize` задают гарнитуру и кегль явно (без них берутся от базового шрифта, у пустого листа Arial 10), `textColor`/`backColor` принимают `#RRGGBB` или `r,g,b`. Оформление ячейки правится **поверх текущего формата**, а не с нуля: вызов только с `bold` не сбросит уже выставленное выравнивание.
  - **Границы** — `setTemplateBorders`: `rect` (1-based, включительно) + `sides` (`all` | `outline` | `inner` | список из `left,top,right,bottom`) + `style` (`None`/`Solid`/`Dotted`/`Double`/`ThinDashed`/`ThickDashed`/`LargeDashed`) + `lineWidth`/`color`. Ставятся прямоугольником, поячеечно вызывать не нужно. Тот же блок доступен пачкой в `drawTemplate`: `borders:[{rect:[r1,c1,r2,c2], sides?, style?, lineWidth?, color?}]` внутри области (координаты там 0-based, как у `merges`).
  - **Размеры** — `setTemplateSizes`: `rowHeights:[{row, height}]` в **пунктах** и `columnWidths:[{column, width}]` в **символах** (дефолтная колонка — 9 символов). В файле лежат свои единицы: четверти пункта по вертикали и восьмые символа по горизонтали, перевод делает сервер. В `drawTemplate` те же ключи внутри области, координаты 0-based.
  - **Печать** — `setTemplatePrintSettings` с объектом `printSettings`: `pageOrientation` (`Portrait`/`Landscape`), `scale` в процентах, `fitToPage`, поля `topMargin`/`leftMargin`/`bottomMargin`/`rightMargin` и `headerSize`/`footerSize` — в **миллиметрах**, плюс `copies`/`blackAndWhite`. Тот же `printSettings` принимает `drawTemplate` ключом верхнего уровня (это свойство листа, не области).
  - Форматы **дедуплицируются**: сотня одинаково оформленных ячеек даёт один узел `<format>`, а вызов без единого применённого свойства не оставляет пустого `<format/>`.
  - **Формат ячейки** — `format` (представление данных: `ЧДЦ=2`, `ДЛФ=D`) и `editFormat` (формат ввода); тем же ключом внутри `cells` у `drawTemplate`. Без него представление числа и даты приходится задавать кодом при заполнении параметра печатной формы.
  - **Размер бумаги** — `paperSize` в `printSettings`: `A4`/`A3`/`A5`/`B4`/`B5`/`Letter`/`Legal`/`Executive`/`Tabloid` либо числовой код принтера (тот же, что в xlsx: A4=9, Letter=1).
  - `columns.size` растёт **по нарисованному**, а не по объявленным ширинам: ячейка, объединение, граница и именованная область поднимают объявленный размер листа так же, как высоту. Прежде лист с ячейками в колонках 0..4 и одной заданной шириной уходил на диск как `<size>1</size>`, и платформа грузила макет усечённым.
  - Параметры `fillType=Template` подставляются по **квадратным** скобкам: `'Договор [Номер] от [Дата]'`. У `fillType=Parameter` имя идёт отдельным полем `parameter`, а `text` к делу не относится.
  - Несуществующий `templateFqn` теперь отвергается («макет не найден, создайте addTemplate»). Раньше любая операция по макету отвечала `ok:true` и заводила на диске каталог с `Template.mxlx`, на который не ссылается ни один `.mdo`. Путь макета собирается по имени каталога типа (`ChartOfCharacteristicTypes` → `ChartsOfCharacteristicTypes`, а не `…Typess`) — до этого макеты планов счетов, ПВХ, планов видов расчёта и бизнес-процессов были недоступны.
- **СКД (`templateType=DataCompositionSchema`):** `createReportSchema` + `addDataSet` (Query/Object/Union) + `addDataSetField` + `addSchemaParameter` + `addCalculatedField` + `addTotalField` + `addDataSetLink` + `addUserField` + `addAutoField` + `addNestedSchema`.
- **Настройки СКД (variant.settings):** `addSettingsVariant`, `addSettingsSelectedField`, `addSettingsFilter`, `addSettingsOrder`, `addSettingsGroup`, `addSettingsTable`, `addSettingsChart`, `addConditionalAppearance`, `addAutoField`, `setOutputParameter`, `setVariantSettings` (bulk-upsert), `repairReportSchema` (диагностика + `autoFix=true` для удаления битых ссылок), `removeSettingsItem`, `removeDataSet`, `removeSchemaParameter`, `removeSchemaItem`.
- **Картинки:** `listPictures` (read-only каталог StdPicture платформы — substring-поиск EN/RU).

### Сборка и обновление ИБ (5)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `rebuild_project` | Полная пересборка проекта (`IProject.build(FULL_BUILD)`), опц. `cleanBuild=true`. После билда ждёт завершения build-job'ов, чтобы счётчик маркеров был стабильным, а не транзиентно завышенным после cleanBuild — но **не дольше `waitSeconds`** (дефолт 180): сборка может упереться в модальный диалог EDT (нет подключённой ИБ, «обновить конфигурацию БД?»), и бесконечный join подвесил бы вызов. В ответе `buildJobsSettled`; при `false` счётчики могут быть неполными. Возвращает счётчики Eclipse problem markers + customChecks. | `projectName`, опц. `cleanBuild`, `extendedChecks`, `waitSeconds` |
| `sync_database` | Headless `Run → Update → Update Database`. Параметры `updateType=INCREMENTAL|FULL`, `skipIfUpdated`, `checkOnly`. **Требует UI Shell** (берётся top-level окно workbench, не временный диалог прогресса). Перед вычислением состояния сам подхватывает внешние правки файлов с диска по проекту конфигурации **и связанным проектам расширений** (`refreshWorkspace=true` по умолчанию) — иначе правка мимо EDT в базу не уедет, см. ниже. В ответе блок `workspaceRefresh` с `changedResources`. Длинную реструктуризацию синхронно не ждёт: по истечении `wait_seconds` (дефолт 45, потолок 120) — `status=Pending` + `jobId`, фоновая работа не отменяется, дальше опрос по `jobId`. Второй update поверх идущего не запускается. | `projectName`, опц. `applicationId`, `updateType`, `skipIfUpdated`, `refreshState`, `refreshWorkspace`, `checkOnly`, `jobId`, `wait_seconds` |
| `job` | Наблюдение за длинной операцией по `jobId`: `action=status` (снимок — состояние, фаза, сколько прошло, сколько без признаков жизни), `wait` (ждать завершения, но не дольше `waitSeconds`, потолок 120), `logs` (хвост лога, `limit`). Работа идёт в фоне и переживает обрыв вызова; завершённая задача хранится 15 мин, потом `jobStatus=unknown` — это «не знаю», а не «не было». Сама ничего не запускает и не отменяет. | `jobId`, опц. `action`, `waitSeconds`, `limit` |
| `refresh_workspace` | Подхватить **внешние** изменения файлов на диске: `IProject.refreshLocal(INFINITE)` + инкрементальный билд с join build-job'ов, чтобы BM-модель EDT переимпортировала изменённые `.mdo`/`.bsl`. Зачем: после `git checkout`/`pull` через shell при открытом EDT воркспейс не замечает правок → read-инструменты отдают stale из кэша BM; этот tool ресинхронит модель с диском. Без правок — безопасный no-op (`changedResources: 0`). Без `projectName` — все открытые проекты. | опц. `projectName`, `build=true` |
| `get_event_log` | Журнал регистрации файловой ИБ (текстовый формат `1Cv8.lgf`+`.lgp`), **без запуска базы**. Диагностика рантайма после `run_yaxunit`/запуска: свежие ошибки/события с фильтром по важности (`Error`/`Warning`/...), периоду (`from`/`to` ISO), пользователю, приложению, событию, подстроке комментария. Каталог резолвится из `projectName` (файловая ИБ → `/1Cv8Log`) или явным `logDir` (серверная ИБ — только через `logDir`). Пагинация `limit`/`offset`, `order=desc\|asc`. **Записи содержат ПДн** — включай PII-фильтр при работе с облачной моделью. | опц. `projectName`\|`logDir`, `infobaseId`, `severity[]`, `from`/`to`, `user[]`/`application[]`/`event[]`, `commentContains`, `limit`/`offset`/`order` |

### yaxunit + отладка (17)

Цикл: `addBreakpoint` → `run_yaxunit mode=debug` → (на BP) `getState` → `getVariables` → `evaluate` → `resume` → `Done` или следующий BP → `get_yaxunit_report`.

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `addBreakpoint` | BSL line BP. Опц. условный (`condition` — BSL-выражение) или N-й проход (`hitCount`). | `modulePath`, `lineNumber`, опц. `condition`, `hitCount` |
| `removeBreakpoint` | Снять BP по `modulePath:line`. | `modulePath`, `lineNumber` |
| `listBreakpoints` | Все BSL line BP в workspace. | опц. `modulePath` |
| `run_yaxunit` | Запустить yaxunit поверх существующей RuntimeClient launch-config. **Preflight**: база определяется до запуска — из привязки конфигурации либо как приложение проекта по умолчанию (`«Запустить на: <Использовать приложение по умолчанию>»` поддержано, в ответе `usesDefaultApplication`). Если баз несколько и умолчание не задано — отказ с их перечнем; тогда задай базу параметром **`infobase`** (имя или GUID). Выбранная база прописывается в одноразовую копию конфигурации и синхронизируется `autoSync`, поэтому `infobase` в ответе — это реально та база, где шли тесты. Перед launch проверяется актуальность конфигурации БД: если не `UPDATED`, возвращается отказ с советом вызвать `sync_database` — иначе 1С показала бы модальный диалог «Обновить конфигурацию базы данных?» и вызов завис бы. Ответ: **Done** (тесты прошли, markdown-отчёт) / **Pending** (попал на BP — открывай debugger tools) / **started** (timeout — продолжай polling через `getState`). По умолчанию перед launch'ем подхватывает внешние правки файлов с диска (проект приложения + его расширения) и делает headless `IApplicationManager.update` (то же что `sync_database`), чтобы EDT не показывал модалку «Обновить конфигурацию базы данных?» после правок метаданных/модулей. В ответе появляется блок `autoSync` с результатом, внутри — `workspaceRefresh.changedResources`. При `autoSync=false` за актуальность модели отвечает вызывающий: правка мимо EDT без `refresh_workspace`/`sync_database` означает прогон тестов по прежнему коду. **Всегда возвращает `jobId`**: прогон живёт дольше вызова, и если ожидание не уложилось в `wait_seconds` (или клиент оборвал соединение), отчёт дожидается фоновым потоком — итог читается через `job` (`action=wait`/`status`), повторный запуск не нужен. Остановка на точке останова тоже отражается в задании (`phase=suspended: <кадр>`). | `applicationId`, опц. `mode=run|debug`, `tests=["fqn1",...]`, `wait_seconds`, `autoSync=true` (default), `autoSyncTimeout=60` |
| `get_yaxunit_report` | Прочитать `junit.xml` от прошлого запуска (без повторного `run_yaxunit`). Используется в связке Pending → resume → get_yaxunit_report. | — |
| `getState` | Снимок debug-сессий: targets/threads/frames. Для каждого приостановленного — расположение (модуль, метод, line). | — |
| `getVariables` | Локалки кадра стека. | `threadId`, `frameIndex` |
| `evaluate` | Произвольное BSL-выражение через `IWatchExpressionDelegate`. Поддерживает function calls (`Константы.X.Получить()`), арифметику, переменные. Fallback — локальный lookup `Var.Prop.Sub`. | `threadId`, `frameIndex`, `expression` |
| `resume`, `suspend`, `stepOver`, `stepInto`, `stepReturn`, `terminate` | Управление debug — стандартный Eclipse Debug API. | опц. `threadId` |
| `get_profiling_results` | Результаты профилирования 1С через штатный `IProfilingService`: тайминги + счётчики вызовов (`frequency`) по строкам модулей. `action=results` (по умолчанию) — топ-N строк по `sortBy` (`duration`/`frequency`/`percentage`); `start`/`stop` — переключить профилирование на активном debug-target (нужна отладка с поддержкой профилирования); `clear` — очистить. **Tier: PRO**. | опц. `action`, `sortBy`, `limit` |

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
     projectName="Демо", extensionName="Демо.ext",
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

## Обычные (неуправляемые) формы

Конфигурации с `defaultRunMode = OrdinaryApplication` (или просто со старыми
формами внутри) EDT импортирует нормально, но **обычную форму в модель не
поднимает**: и раскладка, и модуль лежат в бинарном `Form.oform` в каталоге
формы. Ни `.form`, ни `Module.bsl` там нет, API чтения контейнера EDT не даёт.
На реальной конфигурации это может быть половина кода — 358 обычных форм из
763 и ~4300 процедур вне BSL-индекса.

Полный цикл работы — как найти модуль, как выглядит правка через конфигуратор и что
проверять перед рефакторингом — в [ordinary-forms.md](ordinary-forms.md).

Что из этого следует практически:

- **Модуль читается** — `read_module_source`, `read_method_source`,
  `get_module_structure` принимают путь `.../Forms/<Имя>/Module.bsl`, хотя
  такого файла на диске нет: текст извлекается из контейнера. В ответе
  `source:"oform"`, `readOnly:true` и `container` с путём к `Form.oform`.
- **Модуль можно править.** `write_module_source` по тому же пути перезаписывает текст
  внутри контейнера (все шесть режимов, `dryRun` есть). EDT обычную форму не моделирует, а
  везёт файл как есть и диффит с ИБ — проверено: правка проходит валидацию и обновление базы.
  Состав процедур менять **можно**: служебных процедур в формах около трети, и они ни с чем не
  связаны. Но обработчики привязаны по имени — в раскладке формы либо строкой в коде
  (`ПодключитьОбработчикОжидания`), — и за переименованием привязка не переезжает. Если правка
  задела такое имя, ответ содержит `warning`, `handlerBindingsAffected` и `actionRequired`:
  перепривязать обработчик в конфигураторе. Строгий режим — `handlerChanges:"refuse"`.
  После правки — `sync_database`. Раскладка формы остаётся только для конфигуратора.
- **`search_in_code` досматривает контейнеры** отдельным проходом. Совпадения
  из них помечены `source:"oform"`, `container` и `openable:false` — открыть
  такую позицию в редакторе EDT нечем. Блок `coverage` в ответе появляется
  только если обычные формы в проекте есть.
- **`get_form_layout` не строит пустую раскладку**: отдаёт `layoutAvailable:false`
  с причиной, путь к контейнеру и `modulePath`, которым читается модуль. Секций
  `attributes`/`items` в ответе нет вовсе — их пустота не должна читаться как
  «в форме ничего нет».
- **`get_form_screenshot` отказывает сразу** — рендерить нечего.
- **`find_object_references` и `get_method_call_hierarchy` неполны** по таким
  формам: раскладка вне BM, модули вне BSL-индекса. Оба добавляют блок
  `coverage`, если обычные формы в проекте есть. Перед `removeObject` или
  переименованием на такой конфигурации проверяй ещё и `search_in_code`.
- **Создать обычную форму нельзя.** `edit_metadata addForm` принимает только
  `formType="Managed"`.
- **`get_config_properties`** отдаёт `defaultRunMode`,
  `useManagedFormInOrdinaryApplication`, `interfaceCompatibilityMode`,
  `modalityUseMode`, `synchronousPlatformExtensionAndAddInCallUseMode` и
  `formCounts` — этого хватает, чтобы с первого вызова понять режим
  конфигурации. На 8.2-совместимости модальность разрешена, а асинхронные
  шаблоны БСП не соберутся.
- **Интерфейсы 8.2** (`interfaceCompatibilityMode = Version8_2`) лежат в
  `unknown/Interfaces` без `.mdo` — EDT их не моделирует, `list_metadata_objects`
  их не покажет.

Настройка в Preferences — «Обычные (неуправляемые) формы»: `auto` (по умолчанию;
применимость определяется наличием `Form.oform` в проекте, на управляемых
конфигурациях вывод не меняется), `on`, `off`. Аварийное выключение — значение
`off` или переменная окружения `EDT_COMPANION_ORDINARY_FORMS=off`; отказы с
внятной причиной при этом остаются, пропадает только чтение контейнеров.

## Что плагин НЕ делает

- **Не выполняет произвольный BSL в продуктовом 1С** — только evaluate на приостановленном кадре под отладкой.
- **Не открывает 1С/EDT** — управляет уже открытым workspace.
- **Не валидирует semantic linking запросов к метаданным** — `validate_query` это только синтаксис + типовые QL-проверки.
- **Не редактирует BSL-код метода** — для этого пользователь редактирует файл сам / другой инструмент. Плагин может прочитать (`read_method_source`), найти (`search_in_code`), найти ссылки (`find_object_references` / `get_method_call_hierarchy`), но не редактирует тело процедуры.
- **Не управляет VCS** — git/svn вне scope.
- **Не показывает Form Designer как изображение** — `get_form_layout` отдаёт текстовое/JSON-дерево (для LLM это полезнее PNG).
- **Не покрывает покомпонентный adoption form-item'ов** — формы заимствуются целиком (`adoptObject fqn=...Form.X`), элементы внутри extension-формы добавляются через `addFormItem`.
- **Не редактирует раскладку обычных (неуправляемых) форм и не создаёт такие формы.** Модуль читается и правится, раскладка — только в конфигураторе (см. «Обычные формы» выше).

## Известные особенности

- **Конфигурационная вьюшка кешируется в EDT.** После `sync_database` повторный вызов может вернуть stale `ApplicationUpdateState` — если важно, добавь короткую паузу или явный `checkOnly=true` повтор.
- **Правки файлов мимо EDT не видны модели — и раньше молча не доезжали до базы.** Решение «обновлять / уже `UPDATED`» EDT принимает по BM-модели, а она не знает о правке, сделанной файловым инструментом агента или `git checkout`/`pull`, пока по проекту не выполнен `refreshLocal`. Симптом был обманчивый: `sync_database` отвечал `Done`/`UPDATED` за секунду, а тесты шли по прежней версии кода. Теперь `sync_database` (и `autoSync` внутри `run_yaxunit`) сам делает refresh перед вычислением состояния — по проекту конфигурации **и связанным проектам расширений**, потому что приложения привязаны к конфигурации, а правка обычно в расширении. Смотри `workspaceRefresh.changedResources` в ответе: `0` — модель и так совпадала с диском, больше нуля — правки подхвачены именно этим вызовом. Отключать (`refreshWorkspace=false`) стоит лишь когда все правки шли через `write_module_source`/`edit_metadata`. Обрати внимание: `refreshState` — про состояние **ИБ**, не про диск; одного его недостаточно.
- **Длинное обновление ИБ — это `Pending`, а не ошибка.** Реструктуризация идёт минутами, дольше жизни MCP-вызова. `sync_database` ждёт `wait_seconds` (дефолт 45) и отдаёт `status:"Pending"` + `jobId`; обновление продолжается. Дальше — `sync_database {jobId}` (отвечает только по реестру задач, ничего не блокирует) до `job.status = done|failed`, либо `checkOnly=true` (ещё и состояние ИБ + `updateInProgress`). Во время идущего обновления `checkOnly` не вызывает `check()` — иначе он вставал в очередь за update и вызов отваливался по таймауту. Запускать второй update поверх идущего не нужно и невозможно — вернётся ссылка на текущий job.
- **Маркеры EDT появляются не сразу после сборки.** Они переприкрепляются к объектам асинхронно (derived-data), уже после возврата из `rebuild_project`. Поэтому `get_validation_errors` перед чтением ждёт готовности (`waitSeconds`, дефолт 20) и возвращает **`markersReady`**. Если `markersReady:false` — пустой или короткий список ничего не доказывает, повтори запрос. Прежде это выглядело как «фильтр `minSeverity` теряет маркеры»; на деле фильтр ни при чём — оба источника (model + builder) читаются одним путём, просто модель ещё не была заполнена.
- **`removeObject` принимает и вложенный FQN** (`Document.X.Template.Y`, `Catalog.X.Form.Y`, `Document.X.TabularSection.Y.Attribute.Z` — все виды из списка в описании `fqn`): объект снимается с containment-фичи владельца, экспортируется **владелец** (в ответе `nested`/`ownerFqn`/`ownerPersisted`), у `Template`/`Form`/`Command` удаляется ещё и папка на диске, у макета и формы детачится внешний blob (`blobFqn`). Раньше резолвился только top-объект, и удаление макета по FQN, который сам же вернул `addTemplate`, отвечало «объект не найден».
- **`removeObject` синхронно удаляет папку объекта на диске** (с 2026-05-21). Раньше требовался ручной `rm -rf src/cf/src/<Type>/<Name>/` после `removeObject` — теперь не нужен. Папка удаляется только если её имя совпадает с коротким именем объекта (защита от сноса flat-контейнеров). `persisted:true` в ответе подтверждает что и `.mdo` через `forceExport`, и папка ушли. Для verify — `folderDeleted:true` + `folderPath` в ответе.
- **`find_object_references` сканирует все проекты workspace** (cf + cfe). Покрывает: BM cross-refs от target и его children (StandardAttribute/Attribute/TabularSection — `Right.objectAttribute` ссылки находятся через child URIs), composite-types (`<types>CatalogRef.X</types>` — UUID-scan `TypeItem.compositeId`). Каждая ссылка несёт `projectName`+`filePath`. Что **пока не покрыто**: `Subsystem.content`, `CommandInterface.visibilityFragments.command`, BSL-литералы внутри Extension-проектов (`search_in_code` workspace-wide для этого случая остаётся обязательным).
- **Контейнер обработчика формы выбирается по событию, `target` передавать не нужно.** События объекта — `AfterWrite`, `AfterWriteAtServer`, `BeforeWrite`, `BeforeWriteAtServer`, `OnReadAtServer`, `OnWriteAtServer`, `OnLoadVariantAtServer`, `OnUpdateUserSettingSetAtServer`, `BeforeLoadUserSettingsAtServer` — ложатся в `Form.extInfo.handlers`; всё остальное (`OnCreateAtServer`, `OnOpen`, `BeforeClose`, `NotificationProcessing`, `FillCheckProcessingAtServer`, `ChoiceProcessing`, …) — в `Form.handlers`. Классификация снята разбором 1035 боевых форм по фактическому месту привязки. Прежде дефолт был «уровень формы», и `AfterWrite` уходил не в тот контейнер: ответ `ok`, а обработчик **молча не срабатывает** — ни `get_validation_errors`, ни `sync_database` этого не видят. В ответе `targetInferredFromEvent`; у формы без `extInfo` (`formPurpose=Custom`) привязка падает на уровень формы с `extInfoFallback` и предупреждением. **Директиву** контейнер не определяет: `AfterWrite`/`BeforeWrite` лежат в `extInfo`, но они клиентские (84 процедуры в боевых модулях — все `&НаКлиенте`), решает имя события.
- **Extension-проекты в write-операциях.** Все операции `edit_metadata` принимают и Configuration-, и Extension-проект в `projectName` (`isExtension:true` в ответе для cfe). Примитивные типы (`String/Number/Boolean/Date`) в реквизитах расширения резолвятся через базовый проект. Заимствование — `adoptObject`/`adoptChild`/`adoptModule` (через `extensionName`); формы заимствуются целиком (`adoptObject fqn=...Form.X`).
- **`setObjectProperty propertyPath="name"` для top-объекта НЕ доводится до диска** (меняет только `<name>` в BM/`.mdo`). Для полноценного каскадного переименования используй **`renameObject`** (реализовано через родной EDT-рефакторинг — обновляет файлы/каталог/`Configuration.mdo`/`Rights`/ссылки в BSL/формах). `setObjectProperty(name)` оставляй только для случаев, когда нужно поменять ровно тег без каскада.
- **Write-операции `edit_metadata` могут таймаутить на стороне MCP-клиента**, но фактически записаться в BM. Повтор «вслепую» создаст дубль — после таймаута сначала read-проверка (`get_form_layout`, `get_object_details`).
- **`show_edt_version` отдаёт две версии, и это разные вещи.** `projectRuntimeVersion` — версия платформы **проекта** (`DT-INF/PROJECT.PMF` → `Runtime-Version`), `compatibilityMode` — режим совместимости **конфигурации** из `Configuration.mdo`. Они расходятся штатно: проект 8.3.24 может держать конфигурацию на 8.2.13, и допустимый API определяется вторым значением. Прежде поле называлось `compatibilityMode`, а содержало первое — на такой конфигурации выбор API получался неверным.
- **`evaluate` требует приостановленный кадр.** Без BP — нельзя.
- **`HTTP 200 + JSON-RPC error -32603`** — реальная ошибка инструмента (например NoClassDefFoundError если bundle не пере-stage'нулся после изменения Require-Bundle). Не игнорируй status=200 — всегда смотри `error` в теле.
- **Сравни `get_validation_errors` с `rebuild_project.errors`** — это **разные** marker store: первое — EDT validation markers, второе — стандартные Eclipse problem markers (типы, разрешение ссылок). Оба бывают полезны.

## Минимальный health-check

```
curl http://127.0.0.1:6868/health
→ {"status":"ok","tools":46}
```

Если 404 / connection refused → EDT не запущен или bundle не активирован (при первой установке — разовый `-clean` рестарт).

