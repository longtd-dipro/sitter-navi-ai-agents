# sitternavi-app-babysitter — Patterns

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

Quy ước code thực tế của app SITTER (babysitter). Nguồn quy ước Riverpod chính thức: `.claude/skills/flutter-riverpod-expert/SKILL.md` (áp dụng cho mọi file trong repo này). Dưới đây tóm tắt + ví dụ file thật.

---

## 1. Riverpod 3 — provider VIẾT TAY (QUAN TRỌNG NHẤT)

**KHÔNG dùng codegen cho provider.** Tuyệt đối không thêm `@riverpod`, `riverpod_annotation`, `riverpod_generator`, hay `part 'xxx.g.dart'` cho provider. `build_runner` chỉ dùng cho freezed / json_serializable / retrofit / auto_route / slang. README/GEMINI rule trong repo mô tả style codegen "aspirational" — **bỏ qua**, code không theo.

### Provider selection
- `Provider<T>` — DI / giá trị thuần: `dioProvider`, `appApiProvider`, `repositoryProvider`, `appRouterProvider`, `appPrefsProvider`.
- `NotifierProvider.autoDispose<C,S>(C.new)` — **pattern chính** cho controller feature (screen-scoped).
- `NotifierProvider` (không autoDispose) — state app-lifetime (network, session).
- `StateProvider`/`StateNotifierProvider` (import `hooks_riverpod/legacy.dart`) — chỉ dùng cho global cũ (`globalLoadingProvider`, `toastOverlayProvider`); KHÔNG dùng cho feature mới.

### Controller (business logic) — `extends Notifier<State>`
Ví dụ thật: `lib/features/work_schedule/controller/work_schedule_controller.dart`
- `build()` trả state khởi tạo; kick load async bằng `Future.microtask(loadX)`; gọi mixin `registerCancelOnDispose()`.
- Mutate qua `state = state.copyWith(...)` (không sửa field tại chỗ).
- Đọc provider khác bằng `ref.read(...)` trong action; `ref.watch(...)` chỉ trong `build()`.
- Guard `if (!ref.mounted) return;` trước/sau await (vì autoDispose).
- try/catch trong controller, đẩy lỗi qua state `isLoading/errorMessage` — **không throw lên UI, không navigate/show dialog từ controller** (UI `ref.listen` rồi mới điều hướng).
- Đi qua **repository**, không gọi `AppApi`/Dio trực tiếp.

Ví dụ tối giản: `lib/features/session/controller/session_controller.dart` (`bootstrap()` đọc `/me`, set `BootstrapStatus`).

**Cách áp dụng:** thêm feature = tạo đủ 4 file: `state/<f>_state.dart` (freezed) · `controller/<f>_controller.dart` (Notifier) · `provider/<f>_provider.dart` (`NotifierProvider.autoDispose<C,S>(C.new)`, ví dụ `lib/features/work_schedule/provider/work_schedule_provider.dart`) · `ui/<f>_page.dart` (`@RoutePage()` Consumer). Rồi chạy build_runner cho freezed.

### ref.watch / select / read / listen
- `ref.watch(provider)` subscribe trong build.
- `ref.watch(provider.select((s) => s.field))` — subscribe 1 field, giảm rebuild (dùng nhiều).
- `ref.read(provider.notifier).action()` — trong callback/sự kiện.
- `ref.listen<State>(provider, (prev,next){...})` — side-effect (navigation/toast).

---

## 2. State — Freezed 3 (immutable)

Ví dụ: `lib/features/*/state/*_state.dart` (+ `.freezed.dart` generated).
- `@freezed abstract class XxxState with _$XxxState`; mọi field có `@Default(...)` để controller trả `const XxxState()`.
- Thêm `const XxxState._();` **chỉ khi** có getter/method.
- Cập nhật qua `copyWith`; `copyWith(field: null)` set null tường minh.
- Enum-state để nguyên `enum` (vd `app_shell/state/vibration_state.dart`), không bọc freezed.
- **Cách áp dụng:** sửa field xong chạy `fvm dart run build_runner build --delete-conflicting-outputs`, không sửa tay `.freezed.dart`.

---

## 3. Model / DTO — json_serializable

Ví dụ: `lib/data/models/calendar/calendar_event_model.dart`.
- `@JsonSerializable() class XModel { ... factory X.fromJson(...) => _$XFromJson(json); toJson() => _$XToJson(this); }` + `part 'x.g.dart';`.
- Enum có value string dùng constructor enum (vd `CalendarEventMode { registerShift('register_shift'), bookingTraining('booking_training') }`).
- Envelope: `data/models/ext/` chứa `api_response.dart` (`ApiResponse<T>`), `list_response.dart` (`{data:[...]}`), `paginated_response.dart`, `params_request.dart`. Repository unwrap `.data`/`.items` trước khi trả UI.
- **Phân biệt:** freezed dùng cho **state** (feature); json_serializable dùng cho **model DTO** (data/models). `.g.dart`/`.freezed.dart` tồn tại cho model/state — KHÔNG cho provider.

---

## 4. Networking — Retrofit + Dio

- Interface: `lib/data/api/app_api.dart` (`@RestApi() AppApi`). Endpoint→model: method trả kiểu model, path lấy từ `AppEndpoints`. Ví dụ:
  ```dart
  @GET(AppEndpoints.calendarEvents)
  Future<CalendarEventsResponse> getCalendarEvents(
    @Query('from') String from, @Query('to') String to, @Query('mode') String mode,
    {@CancelRequest() CancelToken? cancelToken});
  ```
- **Cờ `@Extra`** interceptor đọc: `@Extra({'showLoading': true})` → bật `LoadingOverlay` (global); `@Extra({'skipShowError': true})` → tắt toast lỗi tự động. → **Đừng tự viết spinner/toast per-screen** cho call API chuẩn; chỉ dùng `isLoading/errorMessage` cục bộ cho validation/button.
- **Interceptor** (`lib/app/core/network/interceptors/app_interceptor.dart`): `AppInterceptor` gắn Bearer token, toggle loading, 401 → clear prefs+session+notifications rồi `appRouterProvider.replaceAll([LoginRoute()])`, toast lỗi qua `getApiErrorMessage(err)`. `NetworkInterceptor` reject khi offline (`networkProvider`).
- **CancelToken:** dùng `CancelTokenMixin` (`lib/app/core/network/cancel_token_mixin.dart`) trong controller (`registerCancelOnDispose()`), truyền `@CancelRequest()` xuống endpoint để hủy request khi screen dispose.
- **Repository unwrap:** UI/controller gọi repository, không gọi AppApi. Ví dụ `ApiRepository.getCalendarEvents(...)` (`lib/data/repositories/repository.dart`) gọi `api.getCalendarEvents` rồi trả `res.items`.
- **DI provider:** repo mới → thêm `Provider` viết tay bọc `appApiProvider` trong `data/repositories/provider/`.

---

## 5. Routing — auto_route

- `@RoutePage()` trên page; khai route trong `AppRouter.routes` (`lib/app/routers/app_router.dart`); chạy build_runner cập nhật `app_router.gr.dart`.
- Điều hướng trong widget: `context.router.push(const XRoute())` / `replaceAll([...])`. Ngoài widget (interceptor/service): `ref.read(appRouterProvider).replaceAll([...])`.
- Không dùng guard class; gate theo `RequireAction` qua `routesForRequireAction()` (`lib/app/routers/startup_router.dart`).

---

## 6. Integrations

| Tính năng | File | Ghi chú áp dụng |
|---|---|---|
| **FCM / Push** | `lib/app/core/services/push_notification/` (`fcm_service.dart`, `fcm_controller.dart`, provider/state) | Init trong `main.dart` (`fcmServiceProvider.init()` + `requestPermission()` + đọc `fcmControllerProvider`). Background handler top-level `@pragma('vm:entry-point')`; cập nhật badge qua `app_badge_plus` từ `message.data['unreadCount']`; local notification qua `flutter_local_notifications`. Đăng ký device token qua `AppApi.updateDeviceToken`. |
| **Socket (charge/payment result)** | `lib/app/core/services/socket/socket_service.dart` + `configs/socket_config.dart` | Singleton `SocketService`, `socket_io_client` transport websocket, `disableAutoConnect`. Lắng nghe `charge.succeeded`/`charge.failed` → `waitForChargeResult(chargeId)` (timeout 60s). **Đây là realtime cho kết quả thanh toán, không phải chat** (không có feature chat trong repo). |
| **Deep links** | `lib/app/core/services/deep_link/deep_link_service.dart` | `app_links`; init trong `main.dart`. Cold-start dùng `getInitialLink()`, foreground/background dùng `uriLinkStream`. Chống xử lý lặp bằng cờ `_consumedKey` trong SharedPreferences; giữ `_pendingInitialUri` tới khi AppShell mount. |
| **QR scan (check-in)** | `lib/features/scan_qr/scan_qr_page.dart` | `mobile_scanner` `MobileScannerController` (chỉ `BarcodeFormat.qrCode`, `autoStart:false`). Quản lý lifecycle qua `WidgetsBindingObserver`, permission qua `permission_handler`, rung qua `vibration_helper`. State cục bộ trong `State` (không Notifier). |
| **table_calendar (lịch)** | `lib/app/widgets/calendar/monthly_calendar_widget.dart` | Widget lịch tháng dùng lại; work_schedule feed data từ `getCalendarEvents` (chuyển UTC→JST +9h, xem `work_schedule_controller.dart`). |
| **Presigned upload (S3)** | `lib/data/repositories/files_repository.dart` | 2 bước: `AppApi.createPresignedUrl` lấy `uploadUrl`+`fileUrl` → `Dio().put(uploadUrl, bytes)` bằng **Dio trần** (bypass `AppInterceptor` để không gắn Authorization đụng chữ ký query-string). Trả `fileUrl` (key, vd `pr/<uuid>.png`) để lưu vào profile. Download cũng dùng Dio trần. |

---

## 7. Error handling

- Validate + catch trong controller; đẩy message qua state; UI render `if (state.errorMessage.isNotEmpty) Text(...)`.
- Đa số lỗi API đã tự hiện toast qua `AppInterceptor` trừ khi endpoint set `skipShowError`. `on DioException catch (e)` → lấy `e.response?.data?['message']` với fallback tiếng Nhật (`'失敗しました'`).
- Toast dùng `ToastHelper` + `toastOverlayProvider`; loading dùng `globalLoadingProvider` + `LoadingOverlay` (mount 1 lần trong `MyApp`). Đừng chèn overlay mới.

---

## 8. Cách consume backend API

- App này gọi nhóm **`api/v1/sitter/*`** (calendar/events, working-patterns 希望日登録) + `api/v1/client/*` (master data, sitter-profiles/me) + `api/v1/auth/*`.
- Contract endpoint/response chính thức: **`sitter-navi-docs/docs/backend/sitternavi-web-BE/overview/api-catalog.md`** — đọc file này trước khi thêm/sửa endpoint, không tự đoán path/shape.
- Thêm endpoint mới: (1) thêm hằng vào `app_endpoints.dart` → (2) thêm method `@GET/@POST...` vào `app_api.dart` (+`@Extra` cờ loading/error, `@CancelRequest`) → (3) thêm method unwrap trong repository tương ứng → (4) controller gọi qua `ref.read(repositoryProvider)` → (5) build_runner cho `app_api.g.dart`.

---

## 9. Anti-patterns (tránh)

- ❌ Thêm codegen provider `@riverpod` / `part '*.g.dart'` cho provider.
- ❌ `ref.read` trong `build()` để "né rebuild" (dùng `watch`/`select`).
- ❌ Sửa state tại chỗ (`state.x = ...`) — dùng `copyWith`.
- ❌ Navigate / show dialog từ controller — emit state, UI `ref.listen`.
- ❌ Gọi Dio/AppApi trực tiếp từ controller — qua repository.
- ❌ Thêm `StateNotifier/StateProvider` cho feature mới — dùng `NotifierProvider`.
- ❌ Hard-code màu/size — dùng `ResColors()`/`ResTextStyles()`/`flutter_screenutil`.
