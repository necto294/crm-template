# Модель данных

## Standard
`users`, `roles`, `role_permissions`, `sessions`, `invitations`, `companies`, `contacts`, `company_contacts`, `pipelines`, `pipeline_stages`, `deals`, `deal_contacts`, `activities`, `activity_links`, `tasks`, `tags`, `entity_tags`, `notifications`, `attachments`, `saved_views`, `audit_events`, `privacy_records`, `settings`, `import_jobs`, `import_rows`, `merge_events`.

## Medium
`leads`, `lead_forms`, `custom_field_definitions`, `custom_field_values`, `products`, `deal_items`, `automation_rules`, `automation_runs`, `webhook_endpoints`, `webhook_deliveries`, `mail_accounts`, `mail_messages`, `duplicate_candidates`, `ai_conversations`, `ai_messages`, `ai_tool_calls`, `ai_approvals`, `ai_usage`, `document_chunks`.

## Общие поля основных таблиц
`id`, `created_at`, `updated_at`, `created_by`, `updated_by`, `version` (оптимистичная блокировка), `deleted_at` там, где нужно мягкое удаление. `owner_id` у пользовательских записей.

## Поля для поиска дубликатов
- `companies`: `inn`, `inn_normalized` (только цифры), `name_normalized`.
- `contacts`: `email_normalized` (нижний регистр), `phone_normalized` (формат +7, только цифры).
- Эти поля вычисляются при записи и индексируются. Это не уникальные ограничения: дубликат разрешён, если пользователь явно выбрал «создать как новую».
- Слитые записи: `merged_into_id` и `deleted_at`.

## Импорт и слияние
- `import_jobs`: источник, статус, счётчики (создано, обновлено, пропущено, ошибки), автор.
- `import_rows`: номер строки, статус (новая, совпадение, дубликат в файле, ошибка), id совпавшей записи, выбранное действие, текст ошибки без персональных значений.
- `merge_events`: основная и слитая запись, автор, время, список полей, у которых выбрано значение; значения персональных данных не хранятся.

## Типы
- Деньги: `numeric`, рядом код валюты ISO.
- Даты: `timestamptz`, хранение в UTC.
- Email и телефон: нормализованное значение + исходное отображение. Телефон РФ нормализуется к +7.

## Ограничения и индексы
- Уникальность там, где есть бизнес‑смысл (email пользователя, позиция стадии в pipeline).
- Списки: индекс по `updated_at desc`.
- Ответственный + статус для сделок, задач.
- Kanban: `(pipeline_id, stage_id, rank)`.
- Полнотекстовый поиск: GIN по компаниям, контактам, сделкам.
- Дубликаты: индексы по `inn_normalized`, `email_normalized`, `phone_normalized`.
- Отчёт по менеджерам: индекс по `(owner_id, status, closed_at)` для сделок.
- Частичные индексы `WHERE deleted_at IS NULL`.
- pgvector и HNSW — только в Medium и только после измерений.

## Журнал аудита
`audit_events`: идентификатор пользователя, действие, тип и id сущности, список изменённых полей, время. Значения персональных данных не сохраняются (см. `privacy-152fz.md`).

## Миграции
Только `prisma migrate`. Каждая миграция проверяется на чистой БД. `db push` в production запрещён.
