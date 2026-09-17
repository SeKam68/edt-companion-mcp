# Каталог инструментов — документация и XDTO

Часть [гайда для LLM-агента](../llm-guide.md). Конвенции параметров, коды отказов, типовые сценарии, известные особенности и обычные формы — там; здесь только инструменты этой группы.

## Документация (2)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `get_object_help` | Справка метаданного объекта: `synonym` (ru/en EMap), `comment`, и **HTML-страницы из Help.pages** (тот же mechanism, что `MdHelpContentFileEditor` в EDT UI). | `projectName`, `fqn`, опц. `includeContent=false` |
| `get_platform_docs` | Substring-поиск по дереву платформенной справки 1С (Синтакс-помощник) — типы, методы, свойства, глобальные функции. Источник — `satree.xml` версионного EDT-бандла `com._1c.g5.v8.dt.platform.doc_v8_X_Y`, выбранного по CompatibilityMode проекта (результат соответствует именно той платформе). Возвращает title, путь предков, `isCatalog`, `childCount`. | `query`, опц. `projectName`, `lang=ru\|en`, `limit` |

## XDTO (2)

| Tool | Зачем | Ключевые параметры |
|---|---|---|
| `read_xdto_package` | JSON-дамп схемы XDTO-пакета (`Package.xdto`): `nsUri`, `objectTypes`, top-level `properties`, `valueTypes`, `dependencies`. Type-references рендерятся компактно: `xs:string` для XSD, голое имя для ссылок внутри пакета, `{ns}:Name` для cross-package. | `fqn` (например `XDTOPackage.ApdexExport`), опц. `projectName` |
| `edit_xdto_package` | Редактор схемы. `operation`: `setNamespace`, `addObjectType`/`removeObjectType`, `addObjectProperty`/`removeObjectProperty`, `addTopProperty`/`removeTopProperty`. Тип property — в той же компактной форме, что у reader'а. forceExport бьёт по двум FQN: `<fqn>` (для .mdo) и `<fqn>.Package` (для Package.xdto blob). Поддерживает `dryRun=true`. | `fqn`, `operation`, контекстные `typeName`/`propertyName`/`type`/`lowerBound`/`upperBound`/`form`/`nillable`/`namespace`/... |

