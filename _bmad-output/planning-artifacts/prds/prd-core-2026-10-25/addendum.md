# Addendum: smeta1 — технические решения и обоснования

Дополнение к PRD. Не является контрактом для UX/эпиков; фиксирует технические решения, которые не должны раздувать основной документ. Актуальные инварианты — `docs/architecture-spine.md`.

## 1. Топология и границы

- Поли-репозиторная топология: `api-gateway`, `auth-users`, `company-org`, `estimate`, `pdf-extract`, `spec-service`, `llm-proxy`, `llm-gateway`, `rate-catalog`, `ui`, `deploy`.
- Наружу из бэкенда публикуется только `api-gateway`; UI — отдельный фронтенд-контейнер. Postgres и MinIO — только localhost.
- Один Postgres, отдельная база на сервис (AD-014); прямые чтения чужих БД запрещены.
- Очередь конвейера — таблица `spec_job` в Postgres (`FOR UPDATE SKIP LOCKED`), отдельный брокер не вводится.

## 2. LLM и приватность

- Локальная модель: `Llama-3.2-3B-Instruct-Q2_K.gguf`, CPU-only, `LLM_N_CTX=16384`, KV-cache `q4_0`.
- Рабочий облачный маршрут — `llm-proxy`: alias → локальный/облачный backend, fallback, аудит.
- Каталог провайдеров/моделей — в `spec-service`; ключи не хранятся в БД, только `api_key_env`.
- Облако: `LLM_ALLOW_CLOUD`/`LLM_ALLOW_CLOUD_FALLBACK` + `X-Allow-Cloud`; payload без `structure_only` — осознанное исключение для chat-трафика.

## 3. Справочник и сопоставление

- Справочник глобальный/региональный, без `company_id`. Материал уникален по `(region_code, code)`.
- База 2022 — `material_base_price`; текущие цены — `material_price` по `(material_id, year, quarter, zone_code)`.
- Эмбеддинги — `intfloat/multilingual-e5-small` (384, CPU), HNSW cosine; импорт не считает векторы, их считает фоновый воркер или backfill.
- `POST /v1/rate-catalog/matches` — ближайшая цена за точный регион/год/квартал/зону; `confidence` — cosine, 4 знака; порог `needs_review` — 0.70.
- Текущий алгоритм возвращает top-1 без abstention; если кандидат неверен, пользователь помечает `reject` или исправляет ссылкой. Атрибуты/rerank/abstention — вне MVP (отложены).
- `estimate` сохраняет период в смете и обновляет его каждым успешным сопоставлением; `confirmed`/`corrected` не перезаписываются.

## 4. Сметы и расчёт

- Версия сметы — полный снапшот позиций; ровно одна активная.
- Расчёт в `estimate`: Decimal, `ROUND_HALF_UP`, снапшоты `rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`; LLM не участвует.
- Рассчитанная версия заморожена; новые цены — только в новой версии.
- API diff сравнивает снапшоты без пересчёта; UI diff — 1.1.

## 4a. Обратная связь и ручная привязка (Stories 8.2–8.4)

- `estimate_item`: `reviewed_at`, `status=rejected`, `correction_link`; `match_feedback` — append-only журнал решений `confirm`/`correct`/`reject` со снимком запроса, предложенной/исправленной расценкой, периодом и комментарием.
- Найденная при `match-rates` расценка сохраняется как `confirmed` с `reviewed_at=null`; повторное сопоставление обновляет такие позиции и пропускает ручные решения (`reviewed_at` заполнен).
- Review API принимает `rate_link` (URL из `/prices`), `rate_code`, `material_id` или `unit`; цена материала проверяется за период/зону. `estimate_item.material_id` — ссылка на `material.id`, цена периода остаётся в `material_price`.
- `rate-catalog` отдаёт `GET /v1/rate-catalog/rates/lookup` по коду и периоду; `GET /prices` поддерживает `q` — поиск по коду ИЛИ наименованию.
- JSONL-экспорт: `python -m app.tools.export_match_feedback`; положительные/отрицательные пары, без автоматического обучения.
- UI: чекбокс «Верно» (`confirm`/`reject`), «Исправить» с модальным справочником за период сметы, sticky-колонки; copy-link в `/prices` сохранён.

## 5. Доступ и эксплуатация

- JWT HS256 (7 дней), проверка только на `api-gateway`; identity — заголовки `X-User-Id`, `X-Company-Id`, `X-Role`.
- Публичная регистрация — `estimator`; `admin` — bootstrap.
- Решение 2026-10-25: импорт справочника и настройки LLM доступны всем авторизованным; серверной проверки `admin` нет.
- Браузерный e2e (Playwright) — блокирующий job; integration `pdf-extract` использует отдельную БД `pdf_extract_test`.

## 6. Рассмотренные альтернативы

- **Только облачная LLM** — отклонено: приватность и стоимость; локальный CPU-only остаётся основным.
- **LLM для маппинга** — отклонено: дороже, нестабильнее и хуже объяснимо; выбраны эмбеддинги + cosine.
- **Отдельный брокер очереди** — отклонено для MVP: `spec_job` в Postgres достаточен.
- **Справочник на компанию и версии справочника** — отклонено: реальные данные региональные, а неизменность смет обеспечивают снапшоты.
- **Полный UI валидации/diff в MVP** — отложено до 1.1; API уже готов.

## 7. Ссылки

- `docs/brief.md` — продуктовый обзор.
- `docs/architecture-spine.md` — инварианты AD-001…AD-016.
- `docs/epics-stories.md` — состав эпиков и историй.
- `_bmad-output/implementation-artifacts/sprint-status.yaml` — фактический статус.
