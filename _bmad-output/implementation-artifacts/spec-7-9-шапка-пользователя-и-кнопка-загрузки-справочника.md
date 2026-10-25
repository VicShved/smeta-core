---
title: 'Story 7.9 — Email и роль в шапке, кнопка загрузки справочника'
type: 'feature'
created: '2026-10-24'
status: 'done'
route: 'dispatch'
context:
  - '_bmad-output/implementation-artifacts/spec-3-5-региональная-база-2022-цены-по-кварталам-и-зонам.md'
---

## Intent

**Problem:** в шапке не видно, кто вошёл и с какой ролью; на странице
справочника цен нет кнопки загрузки, особенно когда данные уже есть.

**Approach:** auth-users отдаёт `email` в ответе логина/регистрации; UI хранит
его в сессии и показывает email + локализованную роль. На `/prices` добавлена
кнопка «Загрузить справочник» для всех авторизованных, ведущая на `/catalog` с
текущими фильтрами (регион/год/квартал/зона).

## Tasks & Acceptance

- [x] `TokenResponse.email` в auth-users и UI.
- [x] Email в session/localStorage; очистка при выходе и 401.
- [x] Шапка `App.vue`: email + «Администратор»/«Сметчик».
- [x] Кнопка `prices__upload` на `/prices` видна всем авторизованным.
- [x] Query-параметры предзаполняют форму импорта.
- [x] UI импорта справочника: поле зоны с datalist существующих зон.
- [x] UI справочника: фильтр зоны, колонка зоны, пустое состояние.
- [x] Панель статуса эмбеддингов на `/prices`: дата/время, `x из y`, кнопки «Рассчитать эмбеддинги» и «Пересчитать все».
- [x] UI сметы: выбор зоны в периоде и отправка в match-rates.

**Acceptance Criteria:**
- Given пользователь, when открывает `/prices`, then видит email/роль и кнопку загрузки при любом `total`.
- Given estimator, when открывает `/catalog`, then импорт доступен.
- Given клик по кнопке, when открывается `/catalog`, then регион/период/зона предзаполнены.
- Given выход или 401, then email и роль очищены.

## Verification

- auth-users: `pytest -q` → 19 passed.
- ui: `npm test` → 152 passed; `npm run build` → OK.
- estimate/api-gateway регрессия — зелёная.

## Review Triage Log

- Ожидает ревью.
