<!-- bmad:context -->
<!-- Проверено 2026-10-17, коммитов в репозитории пока нет. Управляется bmad-project-context; правки внутри блока перезаписываются при обновлении. Всё, что нужно сохранить, держите вне маркеров. -->

## smeta-core

Каркас BMAD v6.12.4 под OpenCode для проекта «smeta / core». Исходного кода и манифестов ещё нет — стек не выбран, команд сборки/тестов/линта нет. Планирование и артефакты — `_bmad-output/`, знания проекта — `docs/` (пока отсутствует).

## Политика

- Не редактировать вручную `_bmad/`, `.agents/`, `.opencode/` — это installer-managed вывод BMAD, перезаписывается при установке/обновлении (все три в `.gitignore`).
- `_bmad/config.toml` и `_bmad/config.user.toml` read-only и регенерируются. Устойчивые оверрайды — только в `_bmad/custom/config.toml` (команда) и `_bmad/custom/config.user.toml` (личное).
- Стек ещё не выбран: не выдумывать и не запускать «очевидные» команды сборки/тестов.

## Где что лежит

- BMAD-воркфлоу вызываются слэш-командами `.opencode/commands/bmad-*.md` (алиасы на `.agents/skills/bmad-*/SKILL.md` через `@skills/<id>`); точка входа — `/bmad` (алиас `/bmad-help`).
- Конфиг, командные оверрайды и скрипты BMAD: `_bmad/config.toml`, `_bmad/custom/`, `_bmad/scripts/`.
- Коммитится только `_bmad-output/` (`planning-artifacts`, `implementation-artifacts`, `test-artifacts`); дерево BMAD (`.agents/`, `.opencode/`, `_bmad/`) — локальное.
- `AGENTS.md` управляется скиллом `bmad-project-context`.

## Соглашения, отличные от дефолтов

- Отвечать по-русски и писать документы по-русски (`communication_language` / `document_output_language` = Russian).
- Скрипты BMAD запускать через `uv run` (PEP 723, Python ≥ 3.11), а не `python`: `uv run _bmad/scripts/resolve_config.py --project-root .`.
- Установка/обновление BMAD: `npx bmad-method install` (Node ≥ 20.12).

## Известные подводные камни

- После установки/обновления BMAD обязательно переоткрыть проект в OpenCode: команды и скиллы сканируются при создании инстанса, иначе `/bmad` и `/bmad-*` не появятся (в логе `message=init count=1` вместо ~39).
- Текст внутри маркеров `bmad:context` теряется при `bmad-project-context refresh` — свои инструкции держите вне блока.
<!-- /bmad:context -->
