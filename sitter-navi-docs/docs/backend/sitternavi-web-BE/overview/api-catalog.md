# sitternavi-web-BE — API Catalog

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi (xem Memory Update Gate).

**Đây là API CONTRACT source of truth cho FE/mobile agents.** Không đoán endpoint — đọc file này. Nếu FE thấy thiếu, báo BE cập nhật.

---

## Conventions chung

- **Base prefix:** `app.setGlobalPrefix('api')` + URI versioning default `1` → **mọi route dưới `/api/v1/...`**. Controller KHÔNG lặp lại version. Các base path bên dưới là tương đối — prepend `/api/v1/`.
- **Auth header:**
  - JWT: `Authorization: Bearer <accessToken>`. `@Auth(...roles)` = passport `AuthGuard('jwt')` (+ `RolesGuard` nếu có role). `@Public()` bypass. `@AuthOptional()` không throw. `@AuthRefreshToken()` đọc refresh token trong **body** (không phải header).
  - API key: `ApiKeyGuard` đọc header `x-api-key` (DB-backed + scopes, fallback legacy env key). **Hiện chưa gắn vào controller nào** — `admin/api-keys` chỉ *quản lý* key và tự bảo vệ bằng `@Auth(RoleType.ADMIN)`.
- **Roles** (`RoleType`): `admin` / `parent` / `caregiver`. Admin surface = `@Auth(RoleType.ADMIN)`; client surface = `@Auth()` (bất kỳ user đăng nhập — parent/caregiver, một số owner-scoped qua `@CrudAuth`).
- **`@dataui/crud` route map:** `getManyBase`→`GET /` · `getOneBase`→`GET /:id` · `createOneBase`→`POST /` · `createManyBase`→`POST /bulk` · `updateOneBase`→`PATCH /:id` · `replaceOneBase`→`PUT /:id` · `deleteOneBase`→`DELETE /:id`.
- **Pagination / filter (crud-request query params):** `?page=1&limit=20` (hoặc `per_page`), `?sort=field,DESC`, `?filter=field||$eq||value` (operators: `$eq,$ne,$gt,$lt,$gte,$lte,$cont,$in,$isnull...`), `?s={"field":{"$gt":1}}` (search JSON), `?fields=a,b`, `?join=relation`. `alwaysPaginate: true` + `maxLimit: 100` ở đa số controller → response dạng `{ data, count, total, page, pageCount }`.

### ⚠️ Auth gaps / disabled (cần xác nhận — có thể là tạm thời)
- `admin/parent-profiles` — `@Auth(ADMIN)` **bị comment** → hiện KHÔNG bảo vệ.
- `admin/permissions` — `@Auth(ADMIN)` **bị comment** → hiện KHÔNG bảo vệ.
- `client/postal-codes` — `@Auth()` **bị comment** → public.
- `client/parent-profiles`, `client/users` — controller **bị comment toàn bộ** → không đăng ký route.
- `admin/users POST /reset-password` — bị comment → không active.

---

## Standalone / infra

| Method | Path | Audience | Role | Purpose |
|---|---|---|---|---|
| GET | `/api/v1/ping` | public | — | Health ping (echo query) |
| GET | `/api/v1/health` | public | — | Full health check |
| GET | `/api/v1/health/live` | public | — | Liveness probe |
| GET | `/api/v1/health/ready` | public | — | Readiness (DB+lock, 503 nếu fail) |
| GET | `/api/v1/health/deep` | public | — | Deep sequential check |
| POST | `/api/v1/storage/presigned-url` | client | any auth | Presigned POST upload URL |
| POST | `/api/v1/storage/presigned-put-url` | client | any auth | Presigned PUT upload URL |
| POST | `/api/v1/storage/download-url` | client | any auth | Presigned GET download (batch) |
| DELETE | `/api/v1/storage/file` | client | any auth | Delete files by keys |
| POST | `/api/v1/storage/commit` | client | any auth | Commit uploaded files |
| PUT | `/api/v1/upload/local-upload/:key` | public | — | Local dev file write |
| GET | `/api/v1/upload/files/:key` | public | — | Serve local file |

---

## auth

| Method | Path | Audience | Role | Purpose |
|---|---|---|---|---|
| POST | `/api/v1/auth/register` | client | any auth | Register user (onboarding token) |
| POST | `/api/v1/auth/register-info` | client | any auth | Submit parent onboarding info |
| GET | `/api/v1/auth/me` | client | any auth | Current user info |
| POST | `/api/v1/auth/login` | public | — | Email+password login (parent → OTP challenge) |
| POST | `/api/v1/auth/social-login` | public | — | Social login (Apple/LINE) |
| POST | `/api/v1/auth/reset-password` | public | — | Reset password via OTP token |
| POST | `/api/v1/auth/otp/verify` | public | — | Verify OTP → access token |
| POST | `/api/v1/auth/otp/resend` | public | — | Resend/rotate OTP |
| POST | `/api/v1/auth/otp/email` | public | — | Issue email-verification OTP |
| POST | `/api/v1/auth/refresh-token` | public | — | Rotate refresh token → new access token |
| POST | `/api/v1/auth/refresh-token/revoke` | public | — | Revoke refresh token (logout, 204) |
| POST | `/api/v1/admin/auth/login` | admin | — | Admin login |
| POST | `/api/v1/admin/auth/forgot-password` | admin | — | Email admin reset link |
| POST | `/api/v1/admin/auth/reset-password` | admin | — | Reset admin password (single-use token) |

## api-key (admin, custom — không @Crud)

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET | `/api/v1/admin/api-keys` | admin | List keys |
| POST | `/api/v1/admin/api-keys` | admin | Create key (raw shown once) |
| PATCH | `/api/v1/admin/api-keys/:id/revoke` | admin | Revoke key |
| PATCH | `/api/v1/admin/api-keys/:id/activate` | admin | Re-activate key |
| DELETE | `/api/v1/admin/api-keys/:id` | admin | Soft delete key |

## permission (admin, custom — ⚠️ auth commented → UNPROTECTED)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/admin/permissions` | All permissions |
| GET | `/api/v1/admin/permissions/discovered` | Permissions từ code registry |
| GET | `/api/v1/admin/permissions/groups` | All groups |
| POST | `/api/v1/admin/permissions/groups` | Create group |
| PUT | `/api/v1/admin/permissions/groups/:id/permissions` | Update group's permissions |
| DELETE | `/api/v1/admin/permissions/groups/:id` | Delete group |
| GET | `/api/v1/admin/permissions/roles/:role` | Permissions of role |
| PUT | `/api/v1/admin/permissions/roles/:role` | Replace role permissions |
| GET | `/api/v1/admin/permissions/users/:userId` | User permission detail |
| GET | `/api/v1/admin/permissions/users/:userId/effective` | Effective perms (`?role=`) |
| POST | `/api/v1/admin/permissions/users/:userId/grant` | Grant perms |
| POST | `/api/v1/admin/permissions/users/:userId/revoke` | Revoke perms |
| DELETE | `/api/v1/admin/permissions/users/:userId/permissions` | Remove overrides |
| POST | `/api/v1/admin/permissions/users/:userId/groups` | Assign groups |
| DELETE | `/api/v1/admin/permissions/users/:userId/groups` | Remove groups |

## conversation (client chat — schema `conversation`)

Base `/api/v1/conversations`, `@UseGuards(AuthGuard)` (any auth JWT). Per-route `ParticipantGuard` / `GroupRoleGuard`.

| Method | Path | Guard extra | Purpose |
|---|---|---|---|
| POST | `/api/v1/conversations` | — | Create group |
| GET | `/api/v1/conversations` | — | List my groups |
| GET | `/api/v1/conversations/unread-count` | — | Total unread |
| GET | `/api/v1/conversations/:id` | Participant | Group detail |
| PATCH | `/api/v1/conversations/:id` | Owner/Admin | Update group |
| DELETE | `/api/v1/conversations/:id` | Owner | Dissolve (204) |
| POST | `/api/v1/conversations/:id/members` | Owner/Admin | Add member (204) |
| DELETE | `/api/v1/conversations/:id/members/:userId` | Owner/Admin | Remove member (204) |
| POST | `/api/v1/conversations/:id/leave` | Participant | Leave (204) |
| GET | `/api/v1/conversations/:id/messages` | Participant | Message history |
| POST | `/api/v1/conversations/:id/messages` | Participant | Send message (201 new / 200 dup) |
| POST | `/api/v1/conversations/:id/upload-url` | Participant | Presigned upload URL |

> Note: chat real-time qua WebSocket gateway (`conversation.gateway.ts`) — không phải REST, không liệt kê ở đây.

## push-notification

| Method | Path | Audience | Role | Purpose |
|---|---|---|---|---|
| POST | `/api/v1/client/devices` | client | any auth | Register device token |
| DELETE | `/api/v1/client/devices/:token` | client | any auth | Unregister device token |
| POST | `/api/v1/admin/push-notifications/test` | admin | admin | Test send by template |
| POST | `/api/v1/admin/push-notifications/test/raw` | admin | admin | Test send raw |
| GET | `/api/v1/admin/push-templates` (+ `/:id`) | admin | admin | crud getMany/getOne |
| POST/PATCH/DELETE | `/api/v1/admin/push-templates` (`/`, `/:id`) | admin | admin | crud create/update/delete (overridden: cache) |

## email

| Method | Path | Audience | Role | Purpose |
|---|---|---|---|---|
| GET | `/api/v1/admin/email-templates` (+ `/:id`) | admin | admin | crud getMany/getOne |
| POST/PATCH/DELETE | `/api/v1/admin/email-templates` (`/`, `/:id`) | admin | admin | crud create/update/delete (overridden: cache) |

## children

| Method | Path | Audience | Role | Purpose |
|---|---|---|---|---|
| GET/POST/PATCH/DELETE | `/api/v1/admin/children` (+`/:id`) | admin | admin | crud 5-route (getMany overridden) |
| GET | `/api/v1/client/children` (+`/:id`) | parent-client | any auth (owner-scoped `@CrudAuth`) | crud getMany/getOne (owner-scoped) |
| POST | `/api/v1/client/children` | parent-client | any auth | crud createOne (overridden: forces status=DRAFT) |
| PATCH | `/api/v1/client/children/:id` | parent-client | any auth | crud updateOne (overridden: strips status) |
| DELETE | `/api/v1/client/children/:id` | parent-client | any auth | crud deleteOne |
| PATCH | `/api/v1/client/children/:id/submit` | parent-client | any auth | **custom**: validate → transition COMPLETE |

## Standard @Crud admin/client pairs

Trừ khi ghi chú, mỗi cặp có: **admin** `admin/<path>` (`@Auth(ADMIN)`) + **client** `client/<path>` (`@Auth()`), `@Crud only: [getOne, getMany, createOne, updateOne, deleteOne]` → active `GET /`, `GET /:id`, `POST /`, `PATCH /:id`, `DELETE /:id` (crud-auto; mỗi cái override `getMany` để filter).

| Module | Admin base | Client base |
|---|---|---|
| attendance-log | `admin/attendance-logs` | `client/attendance-logs` |
| booking | `admin/bookings` | `client/bookings` |
| care-service | `admin/care-services` | `client/care-services` |
| membership-plan | `admin/membership-plans` | `client/membership-plans` |
| parent-membership | `admin/parent-memberships` | `client/parent-memberships` |
| payment | `admin/payments` | `client/payments` |
| payout | `admin/payouts` | `client/payouts` |
| review | `admin/reviews` | `client/reviews` |
| service-price | `admin/service-prices` | `client/service-prices` |
| sitter-availability | `admin/sitter-availabilities` | `client/sitter-availabilities` |
| sitter-certification | `admin/sitter-certifications` | `client/sitter-certifications` |
| sitter-profile | `admin/sitter-profiles` | `client/sitter-profiles` |
| sitter-service | `admin/sitter-services` | `client/sitter-services` |
| sitter-training-session | `admin/sitter-training-sessions` | `client/sitter-training-sessions` |
| training-course | `admin/training-courses` | `client/training-courses` |
| training-session | `admin/training-sessions` | `client/training-sessions` |

### Deviations

| Module | Note |
|---|---|
| category | admin+client **read-only** (`only: [getOne, getMany]` → `GET /`, `GET /:id`) |
| emergency-contact | admin 5-route; client **read-only** |
| product | admin **all 7 routes** (+ `POST /bulk`, `PUT /:id`) + `@ResourcePermission('product')`; client **read-only** |
| user | admin `only:[getOne,getMany,createOne,updateOne,deleteOne,createMany]` → thêm `POST /admin/users/bulk`; client controller **commented (inactive)** |
| faq-category / faq-item / faq-sub-category | admin only, **all 7 routes** (incl. `POST /bulk`, `PUT /:id`); no override |
| parent-profile | admin 5-route nhưng **`@Auth(ADMIN)` commented → UNPROTECTED**; client **commented (inactive)** |

### japan-address (read-only master data)

| Entity | Admin base | Client base | Auth |
|---|---|---|---|
| prefecture | `admin/prefectures` | `client/prefectures` | admin ADMIN; client any auth. `only:[getOne,getMany]` |
| municipality | `admin/municipalities` | `client/municipalities` | admin ADMIN; client any auth. `only:[getOne,getMany]` |
| postal-code | `admin/postal-codes` | `client/postal-codes` | admin ADMIN; **client `@Auth()` commented → PUBLIC**. `only:[getOne,getMany]` |

---

## Tóm tắt

~63 controller active (2 commented: `parent-profile.controller.client`, `user.controller.client`). Tất cả dưới `/api/v1/`. Public thực sự: auth endpoints, health, ping, upload, `client/postal-codes`. Public do bug (cần xác nhận): `admin/parent-profiles`, `admin/permissions`.
