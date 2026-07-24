# edt-companion-mcp

HTTP MCP-сервер внутри 1C:EDT. OSGi-плагин поднимает локальный сервер на
`http://127.0.0.1:6868/mcp` (JSON-RPC 2.0 + MCP) и отдаёт **45 инструментов**,
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

### Через update-site (рекомендуется)

Стандартный механизм Eclipse. В 1C:EDT: **Help → Install New Software…**, в поле
*Work with* укажите адрес репозитория обновлений:

```
https://sekam68.github.io/edt-companion-mcp/
```

Отметьте фичу edt-companion-mcp, пройдите мастер и перезапустите EDT.
Обновления ставятся тем же путём (**Help → Check for Updates**).

### Через drop-in jar (альтернатива)

1. Скачайте `io.github.sekam68.edt.companion.mcp-<версия>.jar` из раздела
   [Releases](../../releases).
2. Скопируйте jar в каталог `dropins` установки 1C:EDT. Типичный путь:

   ```
   <установка 1C:EDT>/components/1c-edt-<версия>-x86_64/dropins/
   ```

   (например `C:/Program Files/1C/1CE/components/1c-edt-2025.2.5+2-x86_64/dropins/`).
3. Перезапустите 1C:EDT (при первой установке помогает разовый запуск с `-clean`).

### Проверка

```
curl http://127.0.0.1:6868/health
→ {"status":"ok","tools":45}
```

Если `/health` не отвечает — EDT не запущен либо bundle не активировался.

Порт по умолчанию — `127.0.0.1:6868`. Меняется прямо в **Window → Preferences →
edt-companion-mcp** (поле «TCP-порт», применяется сразу, без перезапуска EDT).
Для headless/CI порт можно задать VM-аргументом `-Dedt.yaxunit.mcp.port=<порт>`
(в `1cedt.ini` после `-vmargs`) или переменной окружения `EDT_YAXUNIT_MCP_PORT` —
они имеют приоритет над значением на странице настроек.

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

## Защита персональных данных (PII-редактор)

Опциональный output-фильтр: перед отправкой результата любого инструмента
агенту (в т.ч. в облачную модель) содержимое прогоняется через набор правил,
и обнаруженные ПДн заменяются на маску `[redacted]` или коррелируемый псевдоним
`Физлицо#<hmac>` (одинаковый вход → один токен, без раскрытия и без хранения
таблицы соответствий). По умолчанию **выключен** — включается осознанно на
базах с реальными ПДн.

Включается одним флагом в **Window → Preferences → edt-companion-mcp** —
**«Фильтровать персональные данные (ПДн) в ответах инструментов»** (применяется
сразу, перезапуск EDT не нужен). Остальное работает на разумных умолчаниях:
встроенный набор правил под 152-ФЗ (email, телефон, СНИЛС, ИНН) и случайный ключ
псевдонимайзера на запуск.

Флаг можно перекрыть переменной окружения `EDT_COMPANION_PII`
(`on`/`1`/`true`/`yes`) — приоритетнее галочки, удобно для CI/headless. Для
продвинутых сценариев доступны (только через env, в UI не выведены):
`EDT_COMPANION_PII_SALT` — постоянная соль для стабильных псевдонимов между
сессиями; `EDT_COMPANION_PII_RULES` — путь к своему JSON-набору правил.

Текущая версия применяет правила по **содержимому значения** (regex): email,
телефон `+7…`, СНИЛС, ИНН, паспортные/длинные числовые последовательности.
Формат правила: `{ "enabled", "scope": "VALUE", "countable", "representation", "regex" }`
(`countable:false` — плоская маска, `countable:true` — псевдоним).

## Каталог инструментов (45)

| Группа | Инструменты |
|---|---|
| Workspace и среда | `list_workspace_projects`, `list_applications`, `show_edt_version` |
| Чтение BSL | `read_module_source`, `read_method_source`, `get_module_structure`, `search_in_code`, `resolve_symbol` |
| Запись BSL | `write_module_source` |
| Метаданные (чтение) | `list_metadata_objects`, `list_modules`, `get_object_details`, `get_form_layout`, `get_form_screenshot`, `get_config_properties` |
| Анализ | `find_object_references`, `get_method_call_hierarchy`, `get_validation_errors`, `get_check_description`, `apply_quick_fix` |
| Редактирование метаданных | `edit_metadata` (единый диспетчер операций) |
| XDTO | `read_xdto_package`, `edit_xdto_package` |
| Сборка и ИБ | `rebuild_project`, `sync_database`, `get_event_log`, `refresh_workspace` |
| Запросы | `validate_query` |
| Документация платформы | `get_object_help`, `get_platform_docs` |
| yaxunit | `run_yaxunit`, `get_yaxunit_report` |
| Отладка | `addBreakpoint`, `removeBreakpoint`, `listBreakpoints`, `getState`, `getVariables`, `evaluate`, `resume`, `suspend`, `stepOver`, `stepInto`, `stepReturn`, `terminate`, `get_profiling_results` |

> **`get_form_screenshot`** требует native-buffered рендера форм: добавьте в
> `1cedt.ini` после строки `-vmargs` две строки
> `-DnativeFormLayoutRender=true` и `-DnativeFormBufferedLayoutRender=true`
> и перезапустите EDT. Без них буфер изображения пуст (форма рисуется в
> нативное окно). Остальные инструменты в этих аргументах не нуждаются.

## Когда не подходит

- **Headless / CI / удалённая разработка без открытого EDT** — сервер живёт
  внутри процесса EDT; без UI ряд операций (в т.ч. `sync_database`) недоступен.
- **Редактирование тела существующего BSL-метода** — плагин читает модули, ищет
  по коду, находит ссылки и дописывает заглушки обработчиков, но тело
  процедуры правит сам агент своими file-tools.

