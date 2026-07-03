# sitternavi-web-BE — Patterns

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi (xem Memory Update Gate).

Các pattern canonical của repo. Skill summaries ở `.claude/skills/`, bản đầy đủ ở `docs/skills/*.skill.md`.

---

## 1. CRUD via `@dataui/crud-typeorm`

**Service** — `extends TypeOrmCrudService<Entity>`, chỉ inject repo:

```ts
@Injectable()
export class CategoryService extends TypeOrmCrudService<Category> {
  constructor(@InjectRepository(Category) repo: Repository<Category>) { super(repo); }
}
```

**Controller** — `@Crud()` + `implements CrudController<Entity>`, expose `get base()`:

```ts
@Auth(RoleType.ADMIN)
@Crud({
  model: { type: Category },
  params: { id: { type: 'string', primary: true, field: 'id' } },
  query: { alwaysPaginate: true, softDelete: true, maxLimit: 100, join: { parent: { eager: false } } },
  routes: { only: ['getOneBase', 'getManyBase'] },
})
@ApiTags(ApiAudience.ADMIN, 'admin/categories')
@Controller('admin/categories')
export class AdminCategoryController implements CrudController<Category> {
  constructor(public service: CategoryService) {}
  get base(): CrudController<Category> { return this; }
}
```

- Auto routes: `getManyBase` (GET /), `getOneBase` (GET /:id), `createOneBase` (POST /), `createManyBase` (POST /bulk), `updateOneBase` (PATCH /:id), `replaceOneBase` (PUT /:id), `deleteOneBase` (DELETE /:id). Bật/tắt bằng `routes.only` / `routes.exclude`.
- Custom logic: override method + `@Override()` + `@ParsedRequest() req: CrudRequest`, gọi `this.service.getMany(req, query)`.
- **How to apply:** scaffold bằng Plop → set `routes.only` theo audience (admin thường read-only hoặc full; client giới hạn) → override khi cần filter/business rule.
- **Ví dụ:** `src/modules/category/category.controller.admin.ts` + `category.service.ts`; override: `src/modules/booking/booking.controller.client.ts`.

---

## 2. Auth & authorization

- **`@Auth(...roles: RoleType[])`** (`src/decorators/auth.decorator.ts`): `applyDecorators(SetMetadata(ROLES_KEY, roles), UseGuards(AuthGuard, RolesGuard), ApiBearerAuth())`. Không truyền role → chỉ auth (bất kỳ user đăng nhập).
- **`@Public()`** → `SetMetadata('isPublic', true)`; **`@AuthRefreshToken()`** → RefreshTokenAuthGuard; **`@AuthOptional()`** → OptionalJwtAuthGuard (không throw nếu thiếu token).
- **Guards** (`src/guards/`): `AuthGuard` extends passport `'jwt'` (bypass nếu `isPublic`); `RolesGuard` đọc `ROLES_KEY`, throw `ForbiddenException('auth.forbidden_resource')` nếu `user.role` không khớp; `RefreshTokenAuthGuard`; `ApiKeyGuard` đọc header `x-api-key`, validate DB-backed (scopes) → fallback legacy env key.
- **Current-user decorators**: `@CurrentUserId()` (`request.user.id` cho HTTP, `client.data.userId` cho WS); `@AuthUser()` → `request.user`.
- **Admin vs client split**: `*.controller.admin.ts` → `@Controller('admin/<resource>')` + `@Auth(RoleType.ADMIN)`; `*.controller.client.ts` → `@Controller('client/<resource>')` + `@Auth()` hoặc `@Auth(RoleType.PARENT)` / `@Auth(RoleType.CAREGIVER)`. `RoleType` = `admin` / `parent` / `caregiver`.
- **How to apply:** đặt `@Auth(RoleType.X)` ở class-level cho toàn controller; role check ở guard, KHÔNG check role trong service.
- **Ví dụ:** `src/decorators/auth.decorator.ts`, `src/guards/roles.guard.ts`, `src/guards/api-key.guard.ts`.

---

## 3. Entity conventions

- **Explicit `type` trong mọi `@Column`** — bắt buộc cho `varchar/text/boolean/jsonb/timestamptz`; thiếu → runtime error `"Data type Object not supported by postgres"`.
- `extends WithIdAndTimestamp` (PK Snowflake `bigint` + timestamps + soft-delete) hoặc `WithTimestamp` (profile PK=FK).
- Enum cho status: `@Column({ type: 'varchar', default: OutboxStatus.PENDING }) status: OutboxStatus`.
- Index unique có điều kiện: `@Index(['email'], { unique: true, where: "role = '...'" })`.
- Relations: `@ManyToOne(() => X, { eager: false })` + `@JoinColumn({ name: 'x_id' })` (tránh N+1). Column snake_case explicit: `@Column({ name: 'company_code' })`.
- Outbox entity `implements IOutboxRecord`.
- Sửa entity → tạo migration (skill `database`).
- **Ví dụ / cite:** skill `entity` (`.claude/skills/entity/SKILL.md`); `src/entities/base.entity.ts`.

---

## 4. DTO validation

- **Required vs Optional (quy tắc vàng):** Required = `@IsString() @IsNotEmpty()`; Optional = `@IsString() @IsOptional()`. **KHÔNG BAO GIỜ** dùng cả `@IsNotEmpty()` + `@IsOptional()` trên cùng field.
- Enum → `@IsEnum(SomeEnum)`. Nested object → `@ValidateNested() @Type(() => ChildDto)`. Array → `@IsArray()`.
- Update DTO = `PartialType(OmitType(CreateDto, ['x']))`.
- Custom validator → `registerDecorator(...)` trong `src/decorators/validators/`.
- **Validation groups**: khi validate theo group, validator không-group bị bỏ qua — thêm `{ always: true }` nếu muốn luôn chạy (xem `docs/validation-groups.md`).
- Global pipe: `ValidationPipe({ whitelist: true })` (strip field lạ) — set trong `main.ts`.
- **How to apply:** DTO request đặt trong `dtos/request/`, có `@ApiProperty`/`@ApiPropertyOptional` cho Swagger.
- **Ví dụ:** `src/modules/children/dtos/request/` (`RequestGetManyChildrenDto`).

---

## 5. Messaging — kiến trúc 5 lớp + outbox

| Lớp | Trách nhiệm | File (module email) |
|---|---|---|
| Template | CRUD template + Redis cache | `email-template.service.ts` |
| Renderer | Pure: template + data → output (không inject DB/Redis) | `email-renderer.service.ts` |
| Outbox | Persist + query + recovery (`extends BaseOutboxService`) | `outbox-email.service.ts` |
| Dispatcher | Interface (token) + concrete SMTP/FCM | `email-dispatcher.interface.ts` + `smtp-email-dispatcher.service.ts` |
| Orchestrator | Kết nối 4 lớp + event handlers | `mail.service.ts` |

- **Orchestrator** chỉ: validate template → `outbox.createPending()` → `dispatcher.dispatch(job)` → `markSent`/`markFailed`. KHÔNG render ở đây (render deferred trong dispatcher/worker).
- **Dispatcher token-based DI:** `export const EMAIL_DISPATCHER = '...'` + provider `{ provide: EMAIL_DISPATCHER, useClass: SmtpEmailDispatcher }`, inject qua `@Inject(EMAIL_DISPATCHER)`. Dispatcher trả `DispatchResult { success, error?, permanent?, persist? }` — KHÔNG throw.
- **Event registry (single source of truth):** `MAIL_EVENT_MAP: Record<eventName, TEMPLATE_CODE>` (`mail-event.registry.ts`). Dynamic registration trong `onModuleInit()` qua `eventEmitter.on(eventName, handler)` — KHÔNG `@OnEvent('**')`. Event chưa map → skip. Handler fire-and-forget (try-catch, sanitize, KHÔNG re-throw).
- **Scheduler recovery** (`mail.scheduler.ts`): `@Cron(EVERY_30_SECONDS)` + `@DistributedLock({ key: 'mail:recovery', ttlMs: 25_000 })` — chỉ recover record stuck (PENDING > 2 phút), dispatch concurrent batch (`Promise.all`, concurrency 5). Cleanup `@Cron('0 3 * * *')` xóa record > 30 ngày. Dispatch chính vẫn do orchestrator lo lúc runtime.
- Shared contracts (`IMessageDispatcher`, `IMessageRenderer`, `BaseOutboxService`, `IOutboxRecord`, `OutboxStatus`) từ `@messaging` — không duplicate giữa email/push.
- **How to apply:** module messaging mới (SMS…) → nhân theo 5 lớp, outbox entity `implements IOutboxRecord` + `extends WithIdAndTimestamp`.
- **Ví dụ:** `src/modules/email/*` (đầy đủ), `src/modules/push-notification/*`.

---

## 6. Cache — cache-aside + fallback

- Dùng wrapper `cacheGet<T>` / `cacheSet` / `cacheDel` từ `@infrastructure/cache` — KHÔNG `cacheManager` trực tiếp (ngoại lệ: `cache-helpers.ts`).
- TTL qua enum `CacheTtlSeconds`: `TEN_SECONDS`, `ONE_MINUTE`, `FIVE_MINUTES` (default & recommend cho template), `ONE_HOUR`, `ONE_DAY`, `ONE_WEEK`.
- Cache key có prefix: `const CACHE_PREFIX = 'email:template:'`.
- Cache-aside: check cache → miss thì query DB → populate best-effort (`.catch(() => {})`).
- Invalidate khi update/delete (override `updateOneBase`/`deleteOneBase`).
- Redis sập → tự fallback in-memory (`InMemoryCacheAdapter` set up bởi `RedisCacheService.onModuleInit()`), không cần code thêm.
- **How to apply:** wrap read path của template/lookup data ít đổi.
- **Ví dụ:** `libs/infrastructure/cache/cache-helpers.ts`, `src/modules/email/email-template.service.ts`.

---

## 7. Distributed lock (cron / concurrency)

- `@DistributedLock({ key, ttlMs?, waitMs?, onFail? })` (`libs/infrastructure/lock/distributed-lock.decorator.ts`). `key` static hoặc `(args) => string`. `onFail` default `'skip'` (bỏ qua nếu không lấy được lock), `'throw'` để ném lỗi.
- Backend: Redis (primary) → PostgreSQL advisory lock (fallback tự động). Cấu hình adapter ở bootstrap (`main.ts` → `configureRedisLockAdapter`).
- Rule: mọi `@Cron` phải có `@DistributedLock` (tránh duplicate khi chạy nhiều instance). Lock TTL < cron interval.
- **Ví dụ:** `src/modules/email/mail.scheduler.ts`.

---

## 8. Error handling & logging

- **Logger:** `private readonly logger = new Logger(ClassName.name)` — KHÔNG `console.log`. Level: `debug` (dev), `log` (milestone), `warn` (recoverable: Redis down, template not found), `error` (cần attention + `error.stack`).
- **Luôn log context đầy đủ**: `to=`, `template=`, `outboxId=` — không log message trống.
- **Không nuốt lỗi**: cấm `catch {}`; tối thiểu `logger.warn`.
- **Event handler = fire-and-forget**: luôn try-catch, KHÔNG re-throw (re-throw → crash event loop).
- **Sanitize trước khi log**: `sanitizeEventPayload(event)` từ `@utils` (strip password/token/otpCode). Runtime: `RequestLoggingInterceptor` tự gắn correlation ID; `sensitive-field-redactor` (logging lib) redact field nhạy cảm.
- **Permanent vs Retryable**: `markFailed(id, error, permanent: true)` cho lỗi vĩnh viễn (SMTP 550-555, FCM token-not-registered, template not found); retryable (timeout, 503) giữ PENDING cho scheduler retry.
- **DispatchResult**: dispatcher trả `{ success, error?, permanent? }`, KHÔNG throw.
- **Global filters** (`main.ts`): `LoggingExceptionFilter` (i18n), `QueryErrorFilter` (map lỗi DB).
- **Ví dụ:** `src/modules/email/mail.service.ts` (`handleMailEvent`), `libs/infrastructure/messaging/outbox/base-outbox.service.ts` (`markFailed`), `src/common/utils/sanitize-event-payload.util.ts`.

---

## 9. Code quality (áp dụng mọi file .ts)

No magic numbers (hằng UPPER_SNAKE_CASE) · tên biến có nghĩa · boolean prefix `is/has/should/can/was` · hàm < 30 dòng single-responsibility · max 4 params (gom object) · enum thay string · early return · `any` → `unknown` + generic. Xem skill `code-quality`.
