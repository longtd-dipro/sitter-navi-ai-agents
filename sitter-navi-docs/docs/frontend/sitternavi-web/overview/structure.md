# sitternavi-web — Structure

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

## Tổng quan

`sitternavi-web` là **dashboard cho ADMIN/OPERATOR** của Sitter Navi (KHÔNG phải site cho parent/sitter).

- **Framework:** Next.js 16 (App Router) · React 19 · TypeScript strict.
- **UI:** Ant Design 6 (`@ant-design/nextjs-registry`) + TailwindCSS 4 + Sass modules.
- **State:** Redux Toolkit 2 (chỉ UI/client state) + TanStack Query 5 (server state).
- **Form:** react-hook-form 7 + zod 4 (`@hookform/resolvers`).
- **HTTP:** axios (wrapper `src/services/client/api.ts`) + `@dataui/crud-request` (build query string cho NestJS crud BE).
- **Realtime:** socket.io-client 4 (`src/services/socket/`).
- **Dev port:** 3001 · **locale:** `ja_JP` · **font:** Noto Sans JP (`next/font/google`).
- **Package manager:** pnpm.
- **Auth login endpoint:** `POST /v1/admin/auth/login`.

Path alias: `@/*` → `src/*` (bắt buộc dùng, ESLint cấm import `../*` ra ngoài — xem convention §4.4).

---

## Directory layout (`src/`)

```
src/
├── app/                     # Next.js App Router (layouts, pages, route groups)
│   ├── layout.tsx           # Root layout: providers + AntdRegistry + ConfigProvider(ja_JP) + font
│   ├── page.tsx             # Root "/" → redirect(ROUTE.PROJECT_MANAGEMENT)
│   ├── error.tsx            # Root error boundary
│   ├── not-found.tsx        # 404
│   ├── (auth)/              # Route group: màn hình chưa đăng nhập (layout căn giữa)
│   │   ├── layout.tsx
│   │   ├── components/auth-header/
│   │   ├── hooks/           # vd use-login (auth mutations)
│   │   ├── sign-in/         # + components/forgot-password-modal
│   │   ├── reset-password/
│   │   └── reset-password-success/
│   └── (dashboard)/         # Route group: khu vực đã đăng nhập (Sidebar + Header)
│       ├── layout.tsx       # 'use client' — Sidebar + DashboardHeader, đọc sidebar slice
│       ├── project-management/     # + [id] (detail, edit, children/[childId]/care-report...), _components, hook
│       ├── sitters-management/     # + [id] (basic-info, calendar, qualification, internal-memo...), hooks, _components/invite-sitter
│       ├── jobs-management/        # + [id]/edit, _components/manual-job
│       ├── billing-management/     # + [id] (edit), receipts/[id], _components/invoice|receipt
│       └── chat-room/              # + [id], hooks (socket + mappers), _components/list|create-room
│
├── components/
│   ├── common/              # Shared UI: sidebar, dashboard-header, breadcrumb-common,
│   │   │                    #   table-common, modal-common, modal-confirm, loading,
│   │   │                    #   toast-container, one-line-text, RouterInitializer.tsx
│   │   └── form-field/      # RHF-integrated wrappers: input, textarea, select, radio-group,
│   │                        #   date-picker, time-picker, picker, tags (+ *.module.scss)
│   └── icons/
│
├── configs/                 # config.ts (env), provider-theme.ts (AntD ThemeConfig)
├── constants/               # route.ts, query-key.ts, slice-name.ts, storage-key.ts, common.ts, errors.ts
├── hooks/
│   ├── queries/             # TanStack Query hooks dùng chung: use-user, use-upload-file, use-download-urls
│   ├── use-pagination.ts    # page/pageSize state cho list
│   ├── use-export-job.ts    # CSV export async: export → poll background job → download
│   ├── use-adress.ts
│   └── use-certifications.ts
├── libs/
│   ├── providers/           # QueryProvider.tsx (QueryClient), StoreProvider.tsx (Redux)
│   └── schema/              # zod schemas: auth, billing, chat-room, child, job, parent, sitter (*.schema.ts)
├── models/
│   ├── interfaces/          # request/response types (auth, sitters, jobs, children, conversation,
│   │                        #   adress, certifications, users, pagination) + index.ts
│   └── types/               # index.ts
├── services/
│   ├── client/              # API service layer (axios): api.ts + <name>.service.ts (xem dưới)
│   └── socket/              # socket.ts (io client), socket-events.ts (event names + payload types)
├── statics/                 # icons/, images/ (logo...)
├── stores/                  # store.ts, hooks.ts (typed hooks), slices/, sidebar-storage.ts (localStorage persist)
├── styles/                  # globals.scss
└── utils/
    ├── crud-query.ts        # buildListQuery() — wrap @dataui/crud-request
    ├── format-utils.ts, string-utils.ts
    └── client/              # storage.ts (cookie/token), router-utils.ts (globalNavigate), toast-utils.ts
```

### Ghi chú co-location (App Router)
- Mỗi route module co-locate UI riêng trong thư mục `_components/` (prefix `_` = private folder, không thành route).
- Hooks đặc thù của 1 route đặt trong `<route>/hooks/` (vd `sitters-management/hooks/`, `chat-room/hooks/`). Hooks dùng chung nhiều route đặt ở `src/hooks/queries/`.
- Dynamic segment: `[id]`, `[childId]`, `[reportId]`.
- Lưu ý: có 1 thư mục `project-management/hook/` (số ít) và các nơi khác dùng `hooks/` (số nhiều) — không đồng nhất `(cần xác nhận nếu chuẩn hóa)`.

---

## Routing model

- **App Router** với **route groups** `(auth)` và `(dashboard)` — dấu ngoặc không xuất hiện trong URL, chỉ để nhóm layout.
- `(auth)` → layout căn giữa, không sidebar. `(dashboard)` → layout có `Sidebar` + `DashboardHeader`.
- Root `/` redirect sang `ROUTE.PROJECT_MANAGEMENT` (`/project-management`) — đây là màn hình mặc định sau khi vào app.

### Route constants — `src/constants/route.ts`

| Constant | Path | Đã build page? |
|---|---|---|
| `INDEX` | `/` | ✅ (redirect → project-management) |
| `SIGN_IN` | `/sign-in` | ✅ |
| `RESET_PASSWORD` | `/reset-password` | ✅ |
| `RESET_PASSWORD_SUCCESS` | `/reset-password-success` | ✅ |
| `PROJECT_MANAGEMENT` | `/project-management` | ✅ (+ `[id]`, edit, children...) |
| `JOB_MANAGEMENT` | `/jobs-management` | ✅ (+ `[id]/edit`) |
| `SITTER_MANAGEMENT` | `/sitters-management` | ✅ (+ `[id]`) |
| `CHAT` | `/chat-room` | ✅ (+ `[id]`) |
| `BILLING` | `/billing-management` | ✅ (+ `[id]`, receipts) |
| `SIGN_UP` | `/sign-up` | ❌ reserved (chưa có page) |
| `ATTENDANCE_MANAGEMENT` | `/attendance` | ❌ reserved |
| `RESERVATION_MANAGEMENT` | `/reservations` | ❌ reserved |
| `CONTACT_BOOK` | `/contact-book` | ❌ reserved |
| `PAYROLL` | `/payroll` | ❌ reserved |
| `NOTIFICATION` | `/notifications` | ❌ reserved |

> Các route "reserved" đã khai báo trong `ROUTE` (dùng cho sidebar menu) nhưng chưa có thư mục page tương ứng trong `src/app/(dashboard)/` tại thời điểm scan.

---

## Naming conventions (từ `sitternavi-convention.md` — authoritative)

- **Thư mục:** kebab-case (`components/common/form-field`, `hooks/queries`).
- **Component file:**
  - PascalCase cho component standalone/nổi bật hoặc initializer (`RouterInitializer.tsx`).
  - kebab-case/lowercase cho component utility trong sub-folder (`form-field/input.tsx`).
- **Hooks:** prefix `use-`, kebab-case (`use-auth.ts`, `use-sitters.ts`).
- **Services:** suffix `.service.ts`, kebab-case (`sitters.service.ts`).
- **Utilities:** suffix `-utils.ts` hoặc kebab-case (`toast-utils.ts`, `router-utils.ts`, `storage.ts`).
- **Constants file:** kebab-case (`query-key.ts`, `route.ts`).
- **Biến/hàm:** camelCase; **React component:** PascalCase; **global constant:** UPPER_SNAKE_CASE (`QUERY_KEYS`, `APIs`).
- **Import order** (ESLint auto-sort): `react` → `next/**` → `next-auth` → external libs → `@/**` internal → `./**` sibling (mỗi nhóm cách 1 dòng trống, alphabet).
- Cấm import `../*` (dùng `@/*`); `no-console` = warn; index làm key = error.

---

## Where things live

| Cần tìm | Ở đâu |
|---|---|
| API service functions | `src/services/client/<name>.service.ts` — mỗi file 1 domain: `auth`, `sitters`, `parent`, `children`, `jobs`, `conversations`, `certifications`, `adress`, `users`, `storage`. Endpoint gom trong object `APIs` đầu file. |
| Axios wrapper + interceptor | `src/services/client/api.ts` (class `Requester`, export default `API`). |
| Query keys | `src/constants/query-key.ts` (`QUERY_KEYS`, `as const`). |
| TanStack Query hooks (chung) | `src/hooks/queries/*` — hook đặc thù route: `src/app/(dashboard)/<route>/hooks/*`. |
| zod schemas | `src/libs/schema/*.schema.ts` (export cả schema + `z.infer` type). |
| Redux slices | `src/stores/slices/*` (`sidebar-slice`, `counter-slice`); store `src/stores/store.ts`; typed hooks `src/stores/hooks.ts`. |
| Request/response models | `src/models/interfaces/*.model.ts` (+ `auth.interface.ts`, `pagination.interface.ts`). |
| Route constants | `src/constants/route.ts` (`ROUTE`). |
| Env config | `src/configs/config.ts` (`serverConfig`). |
| AntD theme | `src/configs/provider-theme.ts` (`providerTheme`). |
| Socket client | `src/services/socket/socket.ts` + `socket-events.ts`. |
| Token/cookie helpers | `src/utils/client/storage.ts`. |
| List query builder | `src/utils/crud-query.ts` (`buildListQuery`). |

> Lưu ý typo trong tên file/model: `adress.service.ts` / `adress.model.ts` (thiếu chữ "d"), và `children.modal.ts` / `users.modal.ts` (dùng "modal" thay vì "model"). Đây là tên thật trong repo — import phải khớp đúng.
