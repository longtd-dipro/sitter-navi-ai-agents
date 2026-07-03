# sitternavi-web — Patterns

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

Các pattern chuẩn của dashboard ADMIN/OPERATOR. AI dev agent theo đúng các pattern này thay vì tự chế mới. Mỗi mục có "how to apply" + file ví dụ.

---

## 1. Data fetching — TanStack Query v5 (object syntax)

**Quy tắc:**
- Component KHÔNG gọi axios trực tiếp. Luồng: component → hook (`use*`) → service function → `API`.
- Luôn dùng **object syntax** v5 (`useQuery({ queryKey, queryFn })`), không positional.
- `queryKey` luôn bắt đầu bằng 1 hằng từ `QUERY_KEYS` (`src/constants/query-key.ts`), thêm các biến ảnh hưởng kết quả (id, page, keyword, filter...) vào mảng.
- List: dùng `placeholderData: keepPreviousData` để giữ data cũ khi đổi trang.
- Detail/query phụ thuộc điều kiện: dùng `enabled: Boolean(id)`.
- Mutation → `invalidateQueries` theo `queryKey` liên quan trong `onSuccess`; báo kết quả bằng `showToast` (tiếng Nhật).
- Realtime cập nhật cache trực tiếp bằng `queryClient.setQueryData` (không refetch) — xem mục 5.

**How to apply:** tạo hook trong `src/hooks/queries/` (chung) hoặc `src/app/(dashboard)/<route>/hooks/` (đặc thù), import service function + `QUERY_KEYS`.

**Ví dụ:**
- Query đơn giản: `src/hooks/queries/use-user.ts`
  ```ts
  useQuery({ queryKey: [QUERY_KEYS.USER_ME], queryFn: fetchUserInfo });
  ```
- List + detail + create/update mutation + invalidate + export: `src/app/(dashboard)/sitters-management/hooks/use-sitters.ts`
  ```ts
  const listQuery = useQuery({
    queryKey: [QUERY_KEYS.SITTER_LIST, page, pageSize, keyword, certificationIds],
    queryFn: () => getSitterList({ page, limit: pageSize, ...}),
    placeholderData: keepPreviousData,
  });
  const updateMutation = useMutation({
    mutationFn: ({ id, data }) => updateSitter(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.SITTER_LIST] });
      queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.SITTER_DETAIL, id] });
      showToast.success('更新しました');
    },
  });
  ```
- Query có `enabled`: `src/app/(dashboard)/sitters-management/hooks/use-sitter-calendar.ts`

**QueryClient defaults** (`src/libs/providers/QueryProvider.tsx`): `queries.refetchOnWindowFocus: false`, `queries.retry: 1`, `mutations.retry: false`.

---

## 2. API layer — axios wrapper + interceptors

File: `src/services/client/api.ts` — class `Requester`, export default singleton `API` với `get/post/put/patch/delete`. `handleSuccess` trả về `response.data` (service function nhận thẳng data, không phải `AxiosResponse`).

**Request interceptor:**
- Set `Accept-Language` mỗi request từ `document.documentElement.lang` (fallback `'ja'`).
- Endpoint chứa `/auth/` (login, forgot/reset, refresh) = public → không gắn token, bỏ qua session guard.
- Endpoint khác: kiểm tra session hết hạn (`isSessionExpired` — 24h), gắn `Authorization: Bearer <idToken>` (`getIdToken()` từ cookie).

**Response interceptor — silent refresh single-flight trên 401:**
- Nếu 401 (và không phải auth endpoint, chưa retry): set `_retry = true`, gọi `refreshAccessToken()` rồi replay request gốc với token mới.
- **Single-flight:** nhiều 401 đồng thời share chung 1 `refreshPromise` → chỉ gọi refresh 1 lần; clear promise trong `finally`.
- Refresh dùng `POST /v1/auth/refresh-token` bằng **bare axios** (không qua interceptor để tránh vòng lặp), lưu token mới qua `saveTokensOnRefresh`.
- Refresh fail → `forceSignOut()`: clear token + `globalNavigate(ROUTE.SIGN_IN)`.

**Endpoints:** khai báo trong object `APIs` đầu mỗi service file. Auth ở `/v1/admin/auth/*` (login/forgot/reset); token refresh/revoke ở `/v1/auth/refresh-token*`; resource admin ở `/v1/admin/*`.

**How to apply:** viết service function trong `src/services/client/<domain>.service.ts`, gọi `API.get/post/...`, cast kiểu trả về từ `models/interfaces`. Không tự tạo axios instance mới.

**Ví dụ:** `src/services/client/sitters.service.ts`, `src/services/client/auth.service.ts`.

### 2.1. List query params — `@dataui/crud-request`

Backend là NestJS crud → dùng `buildListQuery()` (`src/utils/crud-query.ts`) build object query truyền vào axios `params`.

- `keyword` + `searchFields` → OR `$cont` (contains, case-insensitive) trên nhiều field.
- `filters` → mỗi filter 1 field + operator (`$cont` mặc định · `$eq` · `$contIntArr` · `$contArr` · `$intersectsArr`); value rỗng bị skip.
- Nhiều điều kiện → gộp dưới `$and` trong `s`; `page`/`limit` → `setPage`/`setLimit`.
- Trả về `qb.queryObject` (plain object) — pass thẳng làm axios params.

```ts
buildListQuery({ page, limit, keyword, searchFields: ['fullNameKanji'],
  filters: [{ field: 'certificationIds', value: ids, operator: '$contIntArr' }] });
```
> Lưu ý: `use-sitters.ts` hiện truyền `keyword`/`certificationIds` dạng phẳng (server tự xử lý), không phải lúc nào cũng qua `buildListQuery`. Dùng `buildListQuery` khi endpoint cần cú pháp crud `s`/`filter`. `(cần xác nhận per-endpoint)`

---

## 3. Forms — react-hook-form + zodResolver

**Quy tắc:**
- Schema zod ở `src/libs/schema/<domain>.schema.ts`, export cả schema + type (`z.infer`). Message lỗi bằng tiếng Nhật.
- `useForm({ resolver: zodResolver(schema), defaultValues })`.
- Bọc component AntD (`Input`, `Select`, `DatePicker`...) bằng `Controller` để tránh lệch state — đã có sẵn wrapper trong `src/components/common/form-field/` (nhận `control` + `name`, generic `<T extends FieldValues>`).

**How to apply:** định nghĩa/định vị schema trong `libs/schema`, dùng `FormFieldInput`/`FormFieldSelect`... với `control` từ `useForm`.

**Ví dụ:**
- Schema: `src/libs/schema/auth.schema.ts` (`signInSchema`, `resetPasswordSchema` có `.refine` so khớp password).
- Form dùng Controller: `src/app/(auth)/sign-in/page.tsx`.
- Wrapper RHF+AntD: `src/components/common/form-field/input.tsx`.

---

## 4. State — Redux Toolkit chỉ cho UI/client state

**Quy tắc:** Redux CHỈ giữ UI/client state, KHÔNG giữ server data (server data thuộc TanStack Query).

- Slice tên qua `SLICE_NAMES` (`src/constants/slice-name.ts`).
- Store `src/stores/store.ts` (`makeStore`); typed hooks `useAppDispatch`/`useAppSelector`/`useAppStore` ở `src/stores/hooks.ts`.
- Provider `src/libs/providers/StoreProvider.tsx` (lazy `useState(() => makeStore())`).
- Persist localStorage qua `store.subscribe` → `saveSidebarCollapsed` (`src/stores/sidebar-storage.ts`); guard SSR (`typeof window === 'undefined'`), hydrate sau mount trong `(dashboard)/layout.tsx`.

**Ví dụ:** `src/stores/slices/sidebar-slice.ts` (collapsed desktop + mobileOpen drawer). `counter-slice.ts` là mẫu boilerplate.

---

## 5. Realtime — socket.io-client (chat)

File: `src/services/socket/socket.ts` (singleton + ref-count) + `socket-events.ts` (event names + payload types).

**Quy tắc:**
- 1 socket singleton, namespace `/conversations`, `path` từ `serverConfig.socket_path` (`/socket.io`), auth callback gửi `{ token: getIdToken() }`, reconnect vô hạn.
- Ref-count lifecycle: `acquireSocket()` khi mount, `releaseSocket()` khi unmount (disconnect khi refCount về 0). `destroySocket()` để teardown toàn cục.
- Events (`SOCKET_EVENTS`): server→client `group:new-message`, `message:read`; client→server `group:conversation-opened`.
- Nhận message mới → cập nhật **trực tiếp** TanStack Query cache bằng `queryClient.setQueryData` trên `InfiniteData<MessagesResponse>` (dedupe theo id, prepend vào page mới nhất) — không refetch.

**How to apply:** trong hook chat, `acquireSocket()` trong `useEffect`, đăng ký listener, `releaseSocket()` ở cleanup; cập nhật cache thay vì invalidate.

**Ví dụ:** `src/app/(dashboard)/chat-room/hooks/use-chat-socket.ts`, `use-chat-list-socket.ts`.

---

## 6. UI — Ant Design 6 + Tailwind 4 + SSR registry

- **Root layout** `src/app/layout.tsx`: `AntdRegistry` (SSR style, `@ant-design/nextjs-registry`) bọc `ConfigProvider theme={providerTheme} locale={jaJP}` + `<App>` (context message/notification của AntD) + `RouterInitializer` + `ToastProviderContainer`. `button={{ autoInsertSpace: false }}` (giữ nguyên chữ Nhật).
- **Provider order** (ngoài → trong): `StoreProvider` → `QueryProvider` → `AntdRegistry` → `ConfigProvider` → `App`.
- **Theme** `src/configs/provider-theme.ts`: `colorPrimary: #FD8BA6` (hồng), `colorError: #F82635`, font `var(--font-noto-sans-jp)`.
- **Font:** `Noto_Sans_JP` qua `next/font/google`, expose CSS var `--font-noto-sans-jp` (weight 400/500/700).
- **Styling:** Tailwind 4 (config qua PostCSS `postcss.config.mjs`, không `tailwind.config.js`) cho layout/spacing; Sass module (`*.module.scss`) cho component riêng (vd form-field). `globals.scss` là stylesheet chung.
- **Convention:** dùng `Controller` để nối AntD component với RHF; toast qua `showToast` (`src/utils/client/toast-utils.ts`).

**Ví dụ:** `src/app/layout.tsx`, `src/app/(dashboard)/layout.tsx` (Sidebar + Header + Tailwind), `src/components/common/form-field/input.tsx` (AntD + scss module).

---

## 7. Consume backend API contract

Nguồn sự thật cho endpoint (path/method/response) là api-catalog của backend:

`sitter-navi-docs/docs/backend/sitternavi-web-BE/overview/api-catalog.md` (đã có — nguồn contract chính thức).

- Admin dashboard chỉ gọi endpoint admin `/v1/admin/*` (trừ token refresh/revoke `/v1/auth/refresh-token*`).
- FE KHÔNG tự đoán endpoint — đọc api-catalog trước, rồi khai báo path trong object `APIs` của service file tương ứng.
- Kiểu request/response đặt trong `src/models/interfaces/*` để khớp shape backend; list trả về `PaginatedResponse<T>` (`src/models/interfaces/pagination.interface.ts`).
