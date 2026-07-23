# edt-companion-mcp

HTTP MCP-сервер внутри 1C:EDT. OSGi-плагин поднимает локальный сервер на
`http://127.0.0.1:6868/mcp` (JSON-RPC 2.0 + MCP) и отдаёт **39 инструментов**,
через которые AI-агент (Claude Code, Cursor, Cline, любой MCP-клиент) видит то
же, что видит сам EDT: типизированную метамодель, BSL-код с несохранёнными
правками открытых редакторов, структуру форм, СКД, XDTO, а также Eclipse Debug
API для запуска и отладки yaxunit-тестов.

Плагин работает **только над тем workspace, который сейчас открыт в EDT** —
отдельного процесса EDT/1С он не запускает.

## Возможности

- **Чтение** метаданных и BSL: объекты конфигурации, формы (с extInfo и
  обработчиками), СКД, XDTO, предопределённые элементы, права ролей,
  подсистемы, defined-типы. Через BM-транзакции — видит несохранённые правки
  открытых редакторов, а не только диск.
- **Поиск и навигация**: текстовый поиск по BSL, резолв символов,
  cross-reference index (`find_object_references`), иерархия вызовов методов.
- **Редактирование метаданных** — единый инструмент `edit_metadata`: создание и
  удаление объектов, реквизиты, табличные части, формы (через штатный
  `IFormGenerator`), элементы форм, обработчики событий с авто-генерацией
  BSL-заглушек, права, подписки, defined-типы, XDTO-схемы, макеты, СКД.
  Принимает и конфигурации, и проекты-расширения.
- **Заимствование в расширение**: `adoptObject` / `adoptChild` / `adoptModule`
  через штатный `IModelObjectAdopter`.
- **Валидация и сборка**: маркеры EDT-валидации, `rebuild_project`, проверка
  запросов, headless-обновление ИБ (`sync_database`) без модального диалога.
- **yaxunit + отладка**: запуск тестов, чтение отчёта, точки останова с
  условием/hit-count, `getState` / `getVariables` / `evaluate`, пошаговое
  выполнение — через стандартный Eclipse Debug API.

Полный каталог инструментов, конвенции параметров и типовые сценарии — в
[docs/llm-guide.md](docs/llm-guide.md).

## Требования

- 1C:EDT 2025.2 (проверялось на 2025.2.5 / EDT core 26.0.1).
- Java 17 (идёт в составе EDT).
- Открытый в EDT workspace с проектом конфигурации или расширения.

## Установка

### Вариант A — update-site (рекомендуется)

Устанавливайте и обновляйте плагин штатным механизмом Eclipse:

1. В 1C:EDT: **Help → Install New Software… → Add… → Location:**

   ```
   https://sekam68.github.io/edt-companion-mcp/
   ```

2. Отметьте категорию **edt-companion-mcp**, нажмите **Next**, примите
   лицензию, дождитесь установки и перезапустите 1C:EDT.
3. Обновления в дальнейшем — через **Help → Check for Updates**.

### Вариант B — drop-in jar

Плагин также поставляется готовым jar-файлом в разделе
[Releases](../../releases).

1. Скачайте `io.github.sekam68.edt.companion.mcp-<версия>.jar` из последнего
   релиза.
2. Скопируйте jar в каталог `dropins` вашей установки 1C:EDT. Типичный путь:

   ```
   <установка 1C:EDT>/components/1c-edt-<версия>-x86_64/dropins/
   ```

   (например `C:/Program Files/1C/1CE/components/1c-edt-2025.2.5+2-x86_64/dropins/`).
3. Перезапустите 1C:EDT.

### Проверка

```
curl http://127.0.0.1:6868/health
→ {"status":"ok","tools":39}
```

Если `/health` не отвечает — EDT не запущен, либо bundle не активировался
(при первой установке помогает разовый запуск EDT с ключом `-clean`).

### Порт

Порт по умолчанию — `127.0.0.1:6868`. Проще всего сменить интерактивно:
**Window → Preferences → edt-companion-mcp → TCP-порт HTTP-сервера** — значение
применяется сразу, без перезапуска EDT. Для CI/автозапуска порт можно задать
VM-аргументом `-Dedt.yaxunit.mcp.port=<порт>` (в `1cedt.ini` после `-vmargs`)
или переменной окружения `EDT_YAXUNIT_MCP_PORT` — они имеют приоритет над
настройкой из окна. Как поднять два EDT с разными проектами одновременно —
см. [docs/multi-instance.md](docs/multi-instance.md).

Опционально можно включить индикатор в статусной строке EDT
(**Preferences → edt-companion-mcp → «Показывать версию и порт в статусной
строке EDT»**): в трее появляется `⚙ <порт>`, а версия и URL сервера — во
всплывающей подсказке. Полезно, когда открыто несколько экземпляров EDT на
разных портах.

## Подключение AI-агента

Пропишите сервер в `.mcp.json` проекта (или в конфигурации вашего
MCP-клиента):

```jsonc
{
  "mcpServers": {
    "edt-companion-mcp": {
      "type": "http",
      "url": "http://127.0.0.1:6868/mcp"
    }
  }
}
```

Протокол — JSON-RPC 2.0 через `POST /mcp`; поддержаны методы `initialize`,
`tools/list`, `tools/call`. Список инструментов и их параметры агент получает
через `tools/list`; человекочитаемый справочник — в
[docs/llm-guide.md](docs/llm-guide.md).

## Каталог инструментов (39)

| Группа | Инструменты |
|---|---|
| Workspace и среда | `list_workspace_projects`, `list_applications`, `show_edt_version` |
| Чтение BSL | `read_module_source`, `read_method_source`, `get_module_structure`, `search_in_code`, `resolve_symbol` |
| Запись BSL | `write_module_source` |
| Метаданные (чтение) | `list_metadata_objects`, `list_modules`, `get_object_details`, `get_form_layout`, `get_config_properties` |
| Анализ | `find_object_references`, `get_method_call_hierarchy`, `get_validation_errors` |
| Редактирование метаданных | `edit_metadata` (единый диспетчер операций) |
| XDTO | `read_xdto_package`, `edit_xdto_package` |
| Сборка и ИБ | `rebuild_project`, `sync_database` |
| Запросы | `validate_query` |
| Документация платформы | `get_object_help`, `get_platform_docs` |
| yaxunit | `run_yaxunit`, `get_yaxunit_report` |
| Отладка | `addBreakpoint`, `removeBreakpoint`, `listBreakpoints`, `getState`, `getVariables`, `evaluate`, `resume`, `suspend`, `stepOver`, `stepInto`, `stepReturn`, `terminate` |

## Когда не подходит

- **Headless / CI / удалённая разработка без открытого EDT** — сервер живёт
  внутри процесса EDT; без UI ряд операций (в т.ч. `sync_database`) недоступен.
- **Редактирование тела существующего BSL-метода** — плагин читает модули, ищет
  по коду, находит ссылки и дописывает заглушки обработчиков, но тело
  процедуры правит сам агент своими file-tools.
