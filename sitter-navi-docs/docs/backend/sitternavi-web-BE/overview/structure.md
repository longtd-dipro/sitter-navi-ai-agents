# sitternavi-web-BE — Structure

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi (xem Memory Update Gate).

NestJS 11 · TypeORM 0.3.26 · PostgreSQL · Redis/BullMQ · WebSockets · `@dataui/crud-typeorm` · AWS S3/SES · Firebase FCM · nestjs-i18n. Roles enum `RoleType` = `admin` / `parent` / `caregiver`.

---

## 1. Directory layout

```
sitternavi-web-BE/
├── src/
│   ├── main.ts                     ← bootstrap: global prefix 'api', URI versioning v1, CORS, ValidationPipe(whitelist), Swagger (dev/staging)
│   ├── app.module.ts               ← root module
│   ├── app.controller.ts
│   ├── swagger.ts                  ← multi-audience Swagger setup
│   ├── typeorm-cli.config.ts       ← DataSource cho migration CLI
│   ├── bootstrap/
│   │   └── dataui-swagger-fix.ts   ← patch Swagger của @dataui/crud (import ĐẦU TIÊN trong main.ts)
│   ├── configs/                    ← config namespaces (@configs)
│   ├── decorators/                 ← @Auth, @CurrentUserId, @AuthUser, @ApiKey, digit-param, common + validators/
│   ├── guards/                     ← auth.guard, roles.guard, refresh-auth.guard, api-key.guard
│   ├── interceptors/
│   ├── pipes/
│   ├── entities/                   ← base entities + shared entities (base.entity, base-timestamp.entity, snowflake-instance, uploaded-file) + index.ts barrel
│   ├── migrations/                 ← TypeORM migrations (auto/)
│   ├── i18n/                       ← nestjs-i18n locale files
│   ├── workers/                    ← BullMQ standalone workers (all.worker, email/, push/, base/worker-bootstrap)
│   ├── common/
│   │   ├── constants/              ← queue, role (ROLES_KEY), swagger (ApiAudience), index
│   │   ├── dtos/                   ← attachment, create-many, ids, import-result
│   │   ├── enums/                  ← role-type.enum (RoleType), index
│   │   ├── events/                 ← domain-event.base + user/ event classes
│   │   ├── filters/                ← ws-validation-exception.filter, QueryErrorFilter.ts
│   │   ├── helpers/                ← business-hours.helper
│   │   ├── interfaces/             ← IAwsConfig, IFile
│   │   ├── repositories/           ← base.repository
│   │   ├── services/               ← base-business-service (Template Method create/update)
│   │   └── utils/                  ← snowflake, sanitize-event-payload, encryption-hashing, typeorm.transformer, module-classifier
│   └── modules/                    ← 37 feature modules (mỗi module 1 folder — xem §2)
├── libs/
│   ├── common/                     ← @foundation — pure utils (array/async/date/delay/encryption/error/file/guard/hash/id/number/object/pagination/result/retry/string/transform/validation)
│   └── infrastructure/             ← @infrastructure
│       ├── cache/                  ← RedisCacheService, cacheGet/Set/Del helpers, CacheTtlSeconds, in-memory adapter
│       ├── lock/                   ← @DistributedLock, LockManager, redis + postgres adapters
│       ├── logging/                ← LoggingService, correlation-id middleware, exception filter, sensitive-field-redactor, typeorm-logger
│       ├── messaging/              ← @messaging — BaseOutboxService, IOutboxRecord, IMessageDispatcher, IMessageRenderer
│       ├── redis/                  ← RedisIoAdapter (WS), redis.config, key prefix
│       ├── http-client/           ← HttpClientService
│       ├── interceptors/           ← RequestLoggingInterceptor
│       └── utils/                  ← token-bucket, local-cache, retry
├── _base/                          ← Plop templates (plop-templates/) + setup_environment
├── plopfile.js                     ← Plop generator 'crud'
├── docs/                           ← canonical skill docs (docs/skills/*.skill.md) + onboarding + architecture
└── .claude/skills/                 ← skill summaries (13 skills) — authoritative patterns
```

### Module folder (mỗi feature trong `src/modules/<feature>/`)

```
<feature>/
├── <feature>.module.ts
├── <feature>.entity.ts             (hoặc entities/ nếu nhiều entity)
├── <feature>.service.ts            ← extends TypeOrmCrudService<T>
├── <feature>.controller.admin.ts   ← @Controller('admin/<resource>'), @Auth(RoleType.ADMIN)
├── <feature>.controller.client.ts  ← @Controller('client/<resource>'), @Auth() hoặc role parent/caregiver
└── dtos/
    ├── request/    (index.ts barrel)
    └── response/   (index.ts barrel)
```

Modules phức tạp (auth, conversation, permission) có thêm `controller/`, `entities/`, `<feature>.gateway.ts` (WebSocket — chỉ `conversation`), `*.scheduler.ts`, `*-renderer.service.ts`, `*-dispatcher.*`, `*.registry.ts` (messaging modules: email, push-notification).

---

## 2. Path aliases (tsconfig `paths`)

LUÔN dùng alias — KHÔNG relative `../../../`.

| Alias | Real path |
|---|---|
| `@entities` | `src/entities` |
| `@modules` | `src/modules` |
| `@decorators` (+ `/*`) | `src/decorators` |
| `@enums` (+ `/*`) | `src/common/enums` |
| `@events` (+ `/*`) | `src/common/events` |
| `@constants` (+ `/*`) | `src/common/constants` |
| `@utils` (+ `/*`) | `src/common/utils` |
| `@interfaces` | `src/common/interfaces` |
| `@types` | `src/common/types` |
| `@services` | `src/modules/shared/services` |
| `@infrastructure` (+ `/*`) | `libs/infrastructure` |
| `@messaging` (+ `/*`) | `libs/infrastructure/messaging` |
| `@logging` (+ `/*`) | `libs/infrastructure/logging` |
| `@foundation` (+ `/*`) | `libs/common` |
| `@configs` (+ `/*`) | `src/configs` |
| `@*` | `src/*` (catch-all) |

Barrel exports cần cập nhật khi thêm mới: `src/entities/index.ts`, `libs/infrastructure/index.ts`, và thêm alias mới vào `tsconfig.json`.

---

## 3. Naming conventions

| Đối tượng | Convention | Ví dụ |
|---|---|---|
| File | kebab-case | `sitter-profile.controller.admin.ts`, `email-renderer.service.ts` |
| Class | PascalCase | `SitterProfileService`, `MailScheduler` |
| Biến / method | camelCase | `firstDevice`, `getStuckPending` |
| Hằng số | UPPER_SNAKE_CASE | `MAX_RETRIES`, `EMAIL_DISPATCHER`, `ROLES_KEY` |
| Route path | kebab-case | `admin/sitter-profiles`, `client/bookings` |
| DB table/column | snake_case (TypeORM auto-map) | `company_code`, `created_at` |
| Migration file | PascalCase + timestamp prefix | `1699999999999-AddXxx.ts` |

---

## 4. Base classes / mixins

| Base | File | Cung cấp | Dùng khi |
|---|---|---|---|
| `WithIdAndTimestamp` | `@entities/base.entity` | `id` (`bigint` PK, Snowflake gán qua `@BeforeInsert`), `createdAt`/`updatedAt` (`timestamptz`, default `CURRENT_TIMESTAMP(6)`), `deletedAt` (soft-delete). Extends TypeORM `BaseEntity`. | **Mặc định** cho mọi entity có PK riêng. |
| `WithTimestamp` | `@entities/base-timestamp.entity` | Chỉ `createdAt`/`updatedAt`/`deletedAt` (KHÔNG có `id`). | Profile entity mà PK = FK (vd `parent-profile` với PK = `user_id` qua `@PrimaryColumn`). |
| `BaseOutboxService<T>` | `@messaging` (`base-outbox.service.ts`) | CRUD + recovery cho transactional outbox: `createPending`, `getPendingBatch`, `getStuckPending(minutes)`, `markSent`, `markFailed(id, error, permanent?, patch?)`, `cleanup(days)`. | Outbox service của messaging module (email/push). `extends` — KHÔNG override. |
| `IOutboxRecord` | `@messaging` (`outbox-record.interface.ts`) | Contract: `id, status(OutboxStatus), tryCount, maxRetries, lastError, lastAttemptAt, nextRetryAt, createdAt`. `OutboxStatus` = `pending/sending/sent/failed/permanently_failed`. | Outbox entity `implements` interface này. |
| `BaseBusinessService<T>` | `src/common/services/base-business-service.ts` | Template Method cho `create`/`update` (validateCreate → mapToEntity → save → emit event). | Complex business module không dùng thẳng CRUD auto. |
| `TypeOrmCrudService<T>` | `@dataui/crud-typeorm` | Auto CRUD service base. | Đa số service — `extends TypeOrmCrudService<Entity>` (xem patterns.md). |

---

## 5. Scaffold module mới (Plop)

Generator `crud` (khai báo trong `plopfile.js`, templates trong `_base/plop-templates/`).

- **1 module (interactive):** `npm run generate` (= `plop`) rồi `npx plop crud`. Prompt: (1) `name` — tên module; (2) `controllers` checkbox — **Admin** (default on), **Client** (default on), **External (x-api-key)** (default off).
- **Nhiều module (batch):** dùng `node-plop` API — KHÔNG pipe stdin (checkbox prompt bị treo). Xem skill `plop-generator`.
- Actions tự sinh: entity, service, các controller đã chọn, module, DTOs (request/response), và AST actions `ast-register-module` + `ast-register-entity` (tự thêm vào app.module + entities/index.ts).

### Manual steps SAU khi generate (bắt buộc)

1. Xóa placeholder `name: string` trong entity, fill columns thực tế theo ERD (explicit `@Column({ type })`) — xem skill `entity`.
2. Sub-entity (BookingChild, PayoutItem, MessageAttachment…) → tạo file riêng, thêm vào `TypeOrmModule.forFeature([...])` của module **và** `src/entities/index.ts`.
3. Profile entity (PK = user_id) → đổi `WithIdAndTimestamp` → `WithTimestamp` + `@PrimaryColumn`.
4. Tạo migration: `npx typeorm migration:generate -d src/typeorm-cli.config.ts src/migrations/AddXxx` (KHÔNG `synchronize: true` trên prod).
5. Điền DTO validation (required vs optional) + `@Crud()` config (routes.only/exclude) cho controller.
