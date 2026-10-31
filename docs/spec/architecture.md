# Архитектура

Модульный монолит в монорепозитории.

```text
Браузер → HTTPS → Reverse proxy (Caddy или Nginx)
                      ├─ web (Next.js: интерфейс + API)
                      ├─ worker (pg-boss: импорт, письма, уведомления, webhooks)
                      ├─ postgres
                      └─ minio (файлы)
Опционально: локальная модель ИИ; SMTP/IMAP заказчика.
```

## Стек
| Слой | Выбор |
|---|---|
| Пакеты | pnpm workspaces, Turborepo |
| Web | Next.js App Router, React, TypeScript strict |
| UI | Tailwind CSS, shadcn/ui, Lucide, шрифт Inter с кириллицей (локально) |
| Формы | React Hook Form + Zod |
| Таблицы | TanStack Table |
| БД | PostgreSQL, Prisma, обязательные миграции |
| Очередь | pg-boss |
| Вход | Auth.js (Credentials) или Better Auth — решение записать в `docs/adr/0001-auth-library.md` в первой задаче |
| Пароли | argon2id |
| Файлы | MinIO |
| ИИ | Vercel AI SDK + собственный адаптер провайдеров |
| Тесты | Vitest, Playwright, PostgreSQL в контейнере |
| Логи | pino, OpenTelemetry; ошибки — GlitchTip или Sentry self‑hosted |
| Доставка | Docker multi‑stage, Docker Compose |

## Правила слоёв
- Зависимости направлены внутрь: интерфейс/API → сервисы приложения → домен → интерфейсы репозиториев; инфраструктура реализует интерфейсы.
- Prisma только в `packages/db` и реализациях репозиториев в `packages/dal`.
- Права проверяются в DAL. Server Actions и Route Handlers валидируют вход (Zod) и повторяют проверку прав.
- Клиенту — только DTO.
- Домен не зависит от Next.js, Prisma, ИИ‑провайдера.
- Между модулями — вызовы сервисов или события, не прямой импорт чужих репозиториев.

## Модули
identity, companies, contacts, deals (pipelines), activities/tasks, notifications, reporting, imports (с дубликатами и слиянием), privacy, audit; в Medium: leads, automation, integrations, ai.

## Структура репозитория
```text
crm/
├── apps/
│   ├── web/
│   │   └── src/
│   │       ├── app/
│   │       │   ├── (public)/login, reset-password, invite/[token]
│   │       │   ├── (app)/
│   │       │   │   ├── layout.tsx
│   │       │   │   ├── dashboard/page.tsx
│   │       │   │   ├── companies/{page.tsx,[id]/page.tsx}
│   │       │   │   ├── contacts/{page.tsx,[id]/page.tsx}
│   │       │   │   ├── deals/{page.tsx,board/page.tsx,[id]/page.tsx}
│   │       │   │   ├── tasks/page.tsx
│   │       │   │   ├── imports/page.tsx
│   │       │   │   ├── reports/sales-by-manager/page.tsx
│   │       │   │   ├── assistant/page.tsx
│   │       │   │   └── settings/{users,pipeline,privacy,ai}/
│   │       │   └── api/{auth,v1,ai/chat,health}/route.ts
│   │       ├── components/{shell,data-table,command-menu,entity}/
│   │       ├── features/{companies,contacts,deals,tasks,imports,reports,assistant}/
│   │       ├── server/{actions,api}/
│   │       └── styles/globals.css
│   ├── worker/src/{jobs/{imports,notifications,webhooks,automations},registry.ts,index.ts}
│   └── docs/{operations,api,adr,privacy}/
├── packages/
│   ├── db/{prisma/{schema.prisma,migrations,seed.ts},src/{client.ts,transaction.ts}}
│   ├── domain/src/{companies,contacts,deals,tasks,activities,privacy}/
│   ├── dal/src/{context,policies,repositories,queries,dto}/
│   ├── auth/src/{config.ts,session.ts,password.ts,lockout.ts}
│   ├── ui/src/{components,patterns,tokens}/
│   ├── ai/src/{providers,agent,tools,approvals,privacy,evals}/
│   ├── jobs/src/{client.ts,events.ts,schemas.ts}
│   ├── email/src/{smtp.ts,imap.ts,templates}/
│   ├── observability/src/{logger.ts,tracing.ts,mask.ts}
│   ├── config/src/{env.ts,features.ts}
│   └── shared/src/{money.ts,phone-ru.ts,dates.ts,pagination.ts,result.ts}
├── tests/{e2e,integration,fixtures}/
├── infra/
│   ├── compose/{docker-compose.yml,docker-compose.dev.yml,Caddyfile}
│   ├── docker/{Dockerfile.web,Dockerfile.worker}
│   └── scripts/{install.sh,update.sh,backup.sh,restore.sh,healthcheck.sh,acceptance-check.sh}
├── .github/workflows/{ci.yml,security.yml}
├── AGENTS.md  CONTRIBUTING.md  SECURITY.md  .env.example
├── turbo.json  pnpm-workspace.yaml  tsconfig.base.json
```

## Правила файлов
- Один файл — одна ответственность; `utils.ts`, `helpers.ts`, `common.ts` запрещены.
- Публичный API пакета — `index.ts`; импорт вглубь чужого пакета блокирует ESLint.
- `page.tsx`/`layout.tsx` только композиция, без бизнес‑правил.
- Время в UTC, деньги `numeric`, валюта — код ISO.
- Набор функций определяется конфигурацией сборки заказа (`packages/config/src/features.ts`): Medium‑функции подключаются флагами, а не лицензией.
