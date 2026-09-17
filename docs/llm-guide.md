# edt-companion-mcp — гайд для LLM-агента

Этот файл — справочник для нейросетевого агента (Claude, Cursor, Cline, ...), который подключён к плагину через MCP. Прочти его один раз перед работой: здесь то, что действует при любом вызове, — конвенции параметров, коды отказов, типовые сценарии, ограничения и особенности. Описания самих инструментов вынесены в каталог [llm-guide/](llm-guide/) и читаются по надобности — какой файл под какую задачу, показывает таблица в разделе «Каталог инструментов». Установка и подключение описаны в [README](../README.md).

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

Протокол — JSON-RPC 2.0. Поддержаны методы `initialize`, `tools/list`, `tools/call`. Проверка живости: `GET http://127.0.0.1:6868/health` → `{"status":"ok","tools":46,"workspace":"<путь>","workspaceName":"<каталог>","projects":["…"]}`. Поля `workspace` / `projects` отвечают на вопрос «тот ли это EDT»: при двух запущенных экземплярах ответы на разных портах иначе неотличимы, и вызовы уходят в чужой workspace с `project_not_found`.

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

- **`objectName` и `fqn` взаимозаменяемы.** FQN объекта в одних инструментах называется `objectName`, в других `fqn` — исторически. Сервер принимает оба имени: пришедшее значение подставляется под то, которое читает инструмент, и в ответе появляется `warnings` с правильным именем. Исключение — `edit_metadata`, где `fqn` и `objectName` значат разное; там имена не подменяются.
- **Имя инструмента принимается в любом стиле.** В каталоге три стиля (`snake_case` у большинства, `camelCase` у группы отладчика, одиночные слова вроде `evaluate`/`job`). Переименовывать не стали, но вызов терпим: `get_state` найдёт `getState`, `add_breakpoint` — `addBreakpoint`. В `tools/list` имена остаются в исходном виде.
- **Опечатка в имени параметра отвергается, а не игнорируется.** Ключ не из схемы, похожий на схемный (общий camelCase-токен или расстояние правки ≤ 2), даёт отказ `invalid_argument` с названием близкого имени: `неизвестный аргумент 'parentName' — возможно, имелся в виду 'parentItem' (operation=addFormItem)`. Иначе значение терялось, а вызов падал где-то в глубине EDT — по `NullPointerException: The path can not be resolved: /Список/Лот` причину не восстановить. Отличие только в регистре (`formname` → `formName`) исправляется само, с предупреждением. Ключ, ни на что не похожий (служебные поля клиента), схему не ломает — про него только `warnings`. Внутри `operations[]` элементы проверяются так же, отказ называет индекс.

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

`unspecified` означает ровно «не классифицировано», а не «неизвестная ошибка» — такие места размечаются по мере работы. Пропущенный или пустой обязательный аргумент всегда даёт `invalid_argument`, каким бы словом об этом ни было сказано в `message` («обязателен», «не задан», «не передан»).

### Предупреждения

Успешный ответ может нести массив `warnings` — операция сделана, но у неё есть последствие, о котором стоит сказать пользователю. Молча игнорировать их не надо.

### Конфигурация на поддержке

Если конфигурация на поддержке поставщика, `edit_metadata` перед работой спрашивает EDT о правилах поддержки целевого объекта:

- поставщик **запретил** правку → отказ с кодом `locked_by_support`; выход — заимствовать объект в расширение (`operation=adoptObject`) либо снять объект с поддержки вручную в EDT;
- править **можно**, но объект поставляемый → операция выполняется, а в ответ добавляется `warnings`: правка всплывёт конфликтом слияния при следующем обновлении поставщика. Для необязательных изменений надёжнее расширение.

Конфигурация не на поддержке — проверка молчит и ничего не добавляет.

### Пакет операций одной транзакцией

`edit_metadata` принимает массив `operations` вместо одиночного `operation`. Элементы — объекты с полем `operation` и своими аргументами; `projectName` и `dryRun` берутся с верхнего уровня.

```json
{"name":"edit_metadata","arguments":{"projectName":"Демо","dryRun":true,"operations":[
  {"operation":"addForm","fqn":"InformationRegister.Демо","formName":"ФормаСписка","formPurpose":"List"},
  {"operation":"addDynamicListTable","fqn":"InformationRegister.Демо","formName":"ФормаСписка","attributeName":"Список","mainTable":"InformationRegister.Демо"},
  {"operation":"addFormItem","fqn":"InformationRegister.Демо","formName":"ФормаСписка","itemType":"Field","itemSubType":"InputField","itemName":"СписокКод","parentItem":"Список","dataPath":"Список.Код"}
]}}
```

Что это даёт против семи отдельных вызовов:

- **всё или ничего** — отказ любой операции откатывает весь пакет; в ответе `appliedNothing: true` и `operationIndex` сбойной. Полуфабрикатов, которые потом доводят руками, не остаётся;
- каждая операция **видит результат предыдущей** (форма, созданная первой, доступна второй);
- на диск объекты уходят **один раз в конце**, а не после каждого шага;
- ответ — массив `effects` по операциям: индекс, имя, `payload` с тем, что получилось. При `dryRun: true` это и есть превью пакета.

Ограничения: один проект на пакет (переключать `projectName` внутри нельзя), вложенные пакеты не поддержаны, `dryRun` внутри элемента игнорируется, `createObject(objectType=Role)` в пакете запрещён (описанию прав роли нужна отдельная транзакция — роль создавай отдельным вызовом). Большинство объектов имеют русские имена и регистр важен.

## Каталог инструментов

46 инструментов разложены по четырём файлам — читай тот, что нужен задаче, а не
весь каталог: половину его объёма занимает `edit_metadata`, который при чтении
кода не нужен. Имена и схемы параметров ты и так получаешь из `tools/list`; в
файлах — когда что применять, что приходит в ответе и чего ждать не стоит.

| Файл | Группы | Инструменты |
|---|---|---|
| [tools-code.md](llm-guide/tools-code.md) | Workspace и среда (3), Метаданные — чтение (6), Навигация по BSL-коду (5), Запись BSL (1), Анализ кода (5), Валидация запросов (1) | `list_workspace_projects`, `list_applications`, `show_edt_version`, `list_metadata_objects`, `list_modules`, `get_object_details`, `get_config_properties`, `get_form_layout`, `get_form_screenshot`, `read_module_source`, `get_module_structure`, `read_method_source`, `search_in_code`, `resolve_symbol`, `write_module_source`, `find_object_references`, `get_method_call_hierarchy`, `get_validation_errors`, `get_check_description`, `apply_quick_fix`, `validate_query` |
| [tools-metadata-edit.md](llm-guide/tools-metadata-edit.md) | Редактирование метаданных (1) | `edit_metadata` — диспетчер операций: объекты, каскадное переименование, свойства, реквизиты и ТЧ, регистры, enum, предопределённые, подсистемы, планы обмена, команды, формы, справка объекта, права, подписки, определяемые типы, заимствование в расширение, макеты, СКД, картинки |
| [tools-run.md](llm-guide/tools-run.md) | Сборка и обновление ИБ (5), yaxunit + отладка (15) | `rebuild_project`, `sync_database`, `job`, `refresh_workspace`, `get_event_log`, `run_yaxunit`, `get_yaxunit_report`, `addBreakpoint`, `removeBreakpoint`, `listBreakpoints`, `getState`, `getVariables`, `evaluate`, `resume`, `stepOver`, `stepInto`, `stepReturn`, `suspend`, `terminate`, `get_profiling_results` |
| [tools-reference.md](llm-guide/tools-reference.md) | Документация (2), XDTO (2) | `get_object_help`, `get_platform_docs`, `read_xdto_package`, `edit_xdto_package` |

Сквозное — конвенции параметров, коды отказов и формат результата, обычные
формы, известные особенности — остаётся в этом файле и действует для всех
четырёх.

## Типовые сценарии

Сценарии показывают порядок вызовов и заодно подсказывают, какой файл каталога
открывать: параметры инструментов, названных в шагах, — там.

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
- **`find_object_references` сканирует все проекты workspace** (cf + cfe). Покрывает: BM cross-refs от target и его children (StandardAttribute/Attribute/TabularSection — `Right.objectAttribute` ссылки находятся через child URIs), composite-types (`<types>CatalogRef.X</types>` — UUID-scan `TypeItem.compositeId`), `Subsystem.content`, и **обращения из BSL** отдельным текстовым сканом (BM cross-ref индекс их не отдаёт: обращения к менеджерам не доходят до индекса в headless-пути). Каждая ссылка несёт `projectName`+`filePath`. Что **пока не покрыто**: `CommandInterface.visibilityFragments.command`, голое имя объекта в строковом литерале BSL (`РольДоступна("X")`, `ПолучитьФункциональнуюОпцию("X")`) и имена, собранные из частей в рантайме — для них остаётся `search_in_code`.
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
→ {"status":"ok","tools":46,"workspace":"D:\\1C\\workspaces\\Демо",
   "workspaceName":"Демо","projects":["Демо","Демо.Расширение"]}
```

Если 404 / connection refused → EDT не запущен или bundle не активирован (при первой установке — разовый `-clean` рестарт).

Сверь `projects` с проектом, с которым собираешься работать: при нескольких экземплярах EDT нужный порт определяется этим полем, отдельный `list_workspace_projects` для выбора порта больше не нужен.

Если по своему порту тишина или отвечает чужой workspace — **не считай сервер недоступным сразу**. Раскладка запущенных экземпляров лежит в файлах реестра:

```
cat ~/.edt-companion-mcp/instances/*.json
```

Каждый файл — один живой сервер: `port`, `url`, `workspace`, `projects`, `pid`. Найди запись, чей `projects` содержит нужный проект, и работай по её `url`. Записи мёртвых процессов подчищаются при следующем старте, поэтому найденный порт всё равно подтверди `/health`. Пустой каталог (или его отсутствие) значит, что не запущен ни один экземпляр — вот это и есть «MCP недоступен».

