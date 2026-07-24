# edt-companion-mcp — обзор

**HTTP MCP-сервер внутри 1C:EDT.** OSGi-bundle, JDK-builtin `HttpServer` на `127.0.0.1:6868`, JSON-RPC 2.0 + MCP-protocol. 43 инструмента: метаданные, BSL-навигация, рефакторинг-search, валидация, редактирование форм/реквизитов/прав/СКД/XDTO, headless обновление ИБ, запуск и **отладка** yaxunit-тестов, офлайн-описание проверок EDT, чтение журнала регистрации, PNG-скриншот форм. Опциональный PII-фильтр результатов.

## Для кого

1С-разработчик внутри 1C:EDT, который хочет, чтобы AI-агент (Claude Code, Cursor, любой MCP-клиент) видел **всё**, что видит сам EDT — типизированную метамодель, derived data, debug API, форму как структуру FormItem-дерева, не XML.

## Что умеет коротко

- **Чтение** метаданных и BSL: объекты конфигурации, формы с extInfo и handler'ами, СКД, XDTO, predefined items, права ролей, подсистемы, defined types. См. [llm-guide.md](llm-guide.md).
- **Поиск** через BM cross-reference index — `find_object_references` теперь workspace-wide, покрывает composite types (`<types>CatalogRef.X</types>` через UUID-scan), child URIs (`Right.objectAttribute` через `Catalog.X.StandardAttribute.Y`), ссылки в роле через path-fallback.
- **Запись**: единый диспетчер `edit_metadata` с 70+ операциями. Создание/удаление любых top-объектов, реквизиты, ТЧ, формы (через EDT `IFormGenerator` — тот же путь, что мастер «Добавить форму»), элементы форм, обработчики событий **с авто-генерацией BSL-заглушек** в нужной директиве, права, подписки, defined types, XDTO-схемы, предопределённые элементы каталогов и планов. `removeObject` синхронно чистит и `.mdo`, и папку объекта на диске.
- **Заимствование в расширение** через EDT `IModelObjectAdopter`: `adoptObject`, `adoptChild`, `adoptModule`.
- **Headless обновление ИБ** (`sync_database`) — без модального диалога «Обновить конфигурацию?», тот же `IApplicationManager.update` что EDT, плюс resolve `ACTIVE_SHELL` через `Display.syncExec`.
- **YAxUnit-цикл**: `run_yaxunit` + `get_yaxunit_report` как отдельные tools — фоновый процесс не прерывается HTTP-таймаутом MCP-клиента, агент стартует тест в `wait_seconds=0` и потом отдельно ждёт отчёт. Авто-`sync_database` перед launch'ем подавляет модалку. Нормализация имени расширения (EDT-проект → `Configuration.name`).
- **Отладка** через стандартный Eclipse Debug API: BP с условием/hitCount, `getState`/`getVariables`/`evaluate`, `stepOver`/`stepInto`/`stepReturn`/`resume`/`suspend`/`terminate`. То же ядро, что использует UI EDT — yaxunit-target виден.
- **Диагностика и визуализация**: `get_check_description` — офлайн-описание проверки EDT-валидации по `checkId` (стандарты 1С, код наружу не уходит); `get_event_log` — чтение журнала регистрации файловой ИБ (ошибки рантайма после `run_yaxunit`, без запуска базы); `get_form_screenshot` — PNG-рендер формы из WYSIWYG-редактора; `refresh_workspace` — подхват внешних изменений диска (после `git checkout` через shell).
- **Защита ПДн**: опциональный output-фильтр — перед отправкой агенту содержимое прогоняется через набор правил, персональные данные заменяются на маску/коррелируемый псевдоним (включается флагом в настройках плагина).

## Что характеризует подход

- **Видит несохранённые правки EDT** через `IBmTransaction` read/readonly — открытый редактор с грязным буфером отдаёт актуальный AST/BM-граф, а не диск.
- **Не пишет XML руками.** `edit_metadata` идёт через `MdClassFactory` + `IBmTransaction.attachTopObject` + `IModelObjectFactory.fillDefaultReferences`, формы — через `IFormGenerator` + `IFormFieldGenerator`, права — `RightsFactory` + donor-pattern. `IBmModelManager.forceExport` синхронно пишет `.mdo`.
- **Поддерживает Extension-проекты**: `withProject`/`withProjectWritable` API в `MetadataAccess` принимает и `IConfigurationProject`, и `IExtensionProject`. Все операции `edit_metadata` работают над обоими типами проектов; примитивные типы в реквизитах расширения резолвятся через базовый проект.
- **Workspace-wide find_object_references**: сканирует все открытые cf+cfe-проекты, dedupe по `projectName::sourceUri::feature`, каждая ссылка несёт `projectName`+`filePath`.
- **Замечания от пользователей пишутся в `feedback/`** прямо в репо и закрываются точечными правками — см. историю в `git log feedback/`.

## Полный AI-цикл

```
get_validation_errors        // что плохо
  → edit_metadata             // правка метаданных
  → write Module.bsl          // (правит сам агент через свои file-tools)
  → rebuild_project           // компиляция + проверка
  → sync_database             // обновить ИБ headless
  → run_yaxunit               // запуск тестов в фоне
  → get_yaxunit_report        // ждём отчёт без HTTP-таймаута
  → addBreakpoint + ...       // отладка по найденным проблемам
  → терминальный коммит
```

Каждый шаг не блокирует agent-loop модальными диалогами EDT/1С-Платформы.

## Установка

См. [README.md](../README.md) → разделы «Установка» и «Подключение AI-агента». Подключается в `.mcp.json` проекта как `http://127.0.0.1:6868/mcp`. Health: `curl http://127.0.0.1:6868/health` → `{"status":"ok","tools":43}`.

## Когда не подходит

- **Headless / CI / удалённая разработка без открытого EDT** — сервер живёт внутри Eclipse-процесса, без UI нет `ACTIVE_SHELL` для `IApplicationManager.update`.
- **Отладка curl→HTTP-сервиса основной конфы** — платформенный лимит 1С (rphost видит модули только расширений), не наша проблема.
- **Активное редактирование BSL-кода методов** — мы умеем читать модули, искать в коде, находить ссылки, добавлять заглушки обработчиков, дописывать процедуры из EventSubscription/Form handlers. **Тело уже существующего метода** — пишет сам агент через свои file-tools или другой MCP (Edit/Write на `.bsl`).
