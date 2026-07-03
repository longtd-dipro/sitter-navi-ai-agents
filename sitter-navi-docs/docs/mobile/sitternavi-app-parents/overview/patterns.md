# sitternavi-app-parents — Patterns

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

Canonical reference cho conventions: `.claude/skills/flutter-riverpod-expert/SKILL.md` + `.claude/CLAUDE.md` trong repo (authoritative). Tài liệu này tóm tắt + trỏ tới ví dụ code thật.

Luồng dữ liệu một chiều:
```
UI (Consumer) → Provider → Controller (Notifier) → Repository → AppApi (Retrofit/Dio)
                                  ↓
                          State (immutable) → rebuild UI
```

---

## 1. Riverpod 3 — HAND-WRITTEN providers (KHÔNG codegen)

**Quy tắc số một:** KHÔNG dùng `@riverpod`, `riverpod_annotation`, `riverpod_generator`, hay `part 'xxx.g.dart'` cho provider. Provider viết tay. `build_runner` trong repo này chỉ chạy cho Freezed / json_serializable / retrofit / auto_route / slang — **không bao giờ cho Riverpod**. (README/`.agents/rules/GEMINI.md` mô tả style codegen — bỏ qua, code không theo.)

### Các loại provider đang dùng

| Loại | Khi nào | Ví dụ |
|---|---|---|
| `Provider<T>` | DI / giá trị thuần | `appApiProvider`, `dioProvider`, `*_repositoryProvider`, `appRouterProvider`, `socketServiceProvider` |
| `NotifierProvider<C,S>` | state app-lifetime | `networkProvider` |
| `NotifierProvider.autoDispose<C,S>` | **pattern chính** cho controller theo màn | `loginControllerProvider` |
| `StateNotifierProvider` / `StateProvider` (legacy, `hooks_riverpod/legacy.dart`) | chỉ global widget nhỏ có sẵn | `toastOverlayProvider`, `globalLoadingProvider` — **không dùng cho feature mới** |

### Controller pattern (business logic ở đây)

Ví dụ thật: `lib/features/auth/login/controller/login_controller.dart`

```dart
class LoginController extends Notifier<LoginState> {
  @override
  LoginState build() => const LoginState();          // initial state; ref.watch dep phản ứng ở đây

  void updateEmail(String email) =>
      state = state.copyWith(email: email, errorMessage: '', isSuccess: false);

  Future<bool> login() async {
    final err = ValidationHelper.validateEmail(state.email);
    if (err != null) { state = state.copyWith(errorMessage: err); return false; }
    try {
      state = state.copyWith(isLoading: true, errorMessage: '');
      final res = await ref.read(authRepositoryProvider).login(LoginRequest(...));
      state = state.copyWith(isLoading: false, otpIdentifier: res.identifier ?? '');
      return true;
    } catch (e) {
      state = state.copyWith(isLoading: false);
      return false;
    }
  }

  void reset() => ref.invalidateSelf();              // reset về initial state
}
```
Provider (viết tay): `lib/features/auth/login/provider/login_provider.dart`
```dart
final loginControllerProvider =
    NotifierProvider.autoDispose<LoginController, LoginState>(LoginController.new);
```
**How to apply:** feature mới cần đủ 4 file: `state/<f>_state.dart` · `controller/<f>_controller.dart` · `provider/<f>_provider.dart` · `ui/<f>_page.dart`. Rồi chạy build_runner cho Freezed state.

Rules controller:
- `ref.read(...)` trong action; `ref.watch(...)` chỉ cho dep phản ứng trong `build()`.
- Try/catch + validation **trong controller**, surface qua state — **không throw ra UI**.
- **Không navigate / show dialog từ controller** — emit state, UI `ref.listen` rồi điều hướng.
- Luôn đi qua Repository — không gọi Dio/`AppApi` trực tiếp.
- Dọn timer/subscription bằng `ref.onDispose(...)` trong `build()` (xem `conversation_controller.dart` — subscribe socket + tạo `CancelToken` trong `build`).

### ref.watch / select / read / listen (UI)

- `ref.watch(provider)` — subscribe (trong `build`).
- `ref.watch(provider.select((s) => s.field))` — subscribe 1 field, giảm rebuild (dùng nhiều).
- `ref.read(provider.notifier).action()` — one-shot trong event handler.
- `ref.listen<State>(provider, (prev, next) {...})` — side-effect: navigate/toast reacting to state.
- Không `ref.read` trong `build()` để "tối ưu"; không watch trong `ListView.builder` (mỗi item 1 `ConsumerWidget`).

---

## 2. State — Freezed (immutable) cho feature state

Ví dụ thật: `lib/features/auth/login/state/login_state.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'login_state.freezed.dart';                     // Freezed — KHÔNG phải .g.dart của Riverpod

@freezed
abstract class LoginState with _$LoginState {         // freezed 3.x → abstract class
  const LoginState._();                               // CHỈ thêm khi state có getter/method

  const factory LoginState({
    @Default('') String email,
    @Default(false) bool isLoading,
    @Default(RequireAction.none) RequireAction requireAction,
  }) = _LoginState;

  bool get isEmpty => email.isEmpty;                  // getter → cần constructor private phía trên
}
```
**How to apply:** mọi field có `@Default(...)` để controller `build()` trả `const XxxState()`. Update chỉ qua `state = state.copyWith(...)`, không mutate field. Enum-style state để plain `enum`, không bọc Freezed. Regenerate: `dart run build_runner build --delete-conflicting-outputs`. Không sửa tay `*.freezed.dart`.

---

## 3. Models — json_serializable (chủ đạo) + freezed (một phần)

Hai kiểu cùng tồn tại — **match theo file đang sửa**:

- **`@JsonSerializable`** (phổ biến, ~90 file) → sinh `*.g.dart`. Ví dụ DTO đầy đủ getter tiện ích: `lib/data/models/children/child_model.dart` (`factory ChildModel.fromJson` + `toJson`, getter `fullName`, `registrationStatus`...).
- **Envelope generic** (`lib/data/models/ext/`): `@JsonSerializable(genericArgumentFactories: true)` — `ApiResponse<T>`, `CursorResponse<T>`, `PaginatedResponse<T>`. `fromJson` nhận thêm `T Function(Object?) fromJsonT`.
- **Một số model dùng freezed** (đa phần auth request/response, vài model user/term). Ví dụ freezed request: xem các file trong `lib/data/models/auth/` (grep `freezed`).

> **Điểm mấu chốt:** `*.freezed.dart` và `*.g.dart` **DO exist** cho MODEL/STATE (do build_runner sinh). Nhưng **KHÔNG** có `*.g.dart` cho PROVIDER (provider viết tay). Đừng nhầm.

**How to apply:** thêm model → viết `@JsonSerializable` + `part 'x.g.dart'` + `fromJson/toJson`, rồi build_runner. Field JSON đặt qua `@JsonKey(name: '...')` khi cần.

---

## 4. Networking — Retrofit + Dio, đấu nối endpoint → model

### Wiring một call
1. Thêm method vào `AppApi` (`lib/data/api/app_api.dart`) trỏ hằng trong `app_endpoints.dart`:
```dart
@GET(AppEndpoints.children)
Future<PaginatedResponse<List<ChildModel>>> getChildren(
  @Query('page') int page, @Query('limit') int limit,
  @CancelRequest() CancelToken? cancelToken,
);
```
2. Repository wrap (`lib/data/repositories/children_repository.dart`) — nhận `AppApi`, gọi thẳng.
3. Provider repo hand-written (`children_repository_provider.dart`): `Provider((ref) => ChildrenRepository(ref.watch(appApiProvider)))`.
4. Controller `ref.read(childrenRepositoryProvider).getChildren(...)`.

### `@Extra` flags điều khiển UI toàn cục (interceptor đọc)
- `@Extra({'showLoading': true})` → bật/tắt `globalLoadingProvider` (`LoadingOverlay`).
- `@Extra({'skipShowError': true})` → tắt toast lỗi tự động (`toastOverlayProvider`).

→ **Với call API chuẩn, KHÔNG tự viết spinner / error toast per-screen** — set `@Extra` phù hợp. `isLoading`/`errorMessage` trong state chỉ cho UX cục bộ (validate, disable nút).

### Interceptors — `lib/app/core/network/interceptors/app_interceptor.dart`
- `AppInterceptor`: gắn `Authorization: Bearer <token>` từ `appPrefsProvider`; log (chỉ debug); toggle loading theo `showLoading`; onError: nuốt `DioExceptionType.cancel`; **401 → `_handleUnauthorized`** (disconnect socket, clear prefs, cancel notifications, `appRouterProvider.replaceAll([LoginRoute()])`); lỗi khác → toast qua `getApiErrorMessage(err)` trừ khi `skipShowError`.
- `NetworkInterceptor`: nếu `networkProvider == offline` → reject sớm với `connectionError`.
- `dioProvider` (`network/provider/dio_provider.dart`): baseURL `EnvConfig.baseUrl`, timeout 30s, thêm `LogInterceptor` chỉ khi `kDebugMode`.
- **Cancel token:** `request_cancel_manager.dart` + `requestCancelManagerProvider` — controller tạo token trong `build()`, tự huỷ khi dispose (xem `conversation_controller.dart`).

### Error handling controller
Bắt trong controller, đọc message từ `e.response?.data['message']` (DioException), đưa vào state; UI render. Nhớ đa số lỗi đã tự toast qua interceptor trừ khi `skipShowError`.

---

## 5. Routing pattern (auto_route)

- Annotate page: `@RoutePage()` trên `class XxxPage extends Consumer[Stateful]Widget` (ví dụ `lib/features/scan_qr/scan_qr_page.dart`).
- Khai báo route trong `lib/app/routers/app_router.dart` (nested trong `AppShellRoute` cho màn sau login). Regenerate `app_router.gr.dart`.
- Điều hướng: `context.router.push(const XxxRoute())` / `context.router.replaceAll([...])` — **không** `Navigator.push`. Điều hướng "hệ thống" (logout) qua `ref.read(appRouterProvider).replaceAll(...)`.
- Không có guard; auth-gating ở Splash + interceptor 401.

---

## 6. UI conventions

- Widget: `ConsumerWidget` / `ConsumerStatefulWidget` / `HookConsumerWidget` (flutter_hooks có sẵn).
- Page mix-in từ `lib/app/base/`: `PageEffectMixin` (`onInit`/`onDispose` + getter `t`), `BasePageMixin` (Scaffold/AppBar cấu hình qua `buildBody()`).
- `LoadingOverlay` + `ToastOverlayWidget` mount 1 lần trong `MyApp` (`lib/app/app.dart`, trong `Stack` builder) — điều khiển qua provider, không chèn overlay mới.
- Màu/typography: `ResColors()` / `ResTextStyles()` (`app/core/resources/`). Kích thước: `flutter_screenutil` (design 390×844) — `.w/.h/.sp`.
- `TextEditingController`/`FocusNode` để trong `State` của widget, dispose ở `onDispose`; controller chỉ expose `updateXxx(value)`.

---

## 7. Integrations

| Tích hợp | File chính | Pattern / how to apply |
|---|---|---|
| **Push (FCM)** | `app/core/services/push_notification/fcm_service.dart` (+ `fcm_controller.dart`, `fcm_provider.dart`, `fcm_state.dart`, `fcm_degate.dart`) | `FCMService` init Firebase + `flutter_local_notifications`, background handler top-level `@pragma('vm:entry-point')`, cập nhật badge `app_badge_plus` từ `message.data['unreadCount']`. `main.dart` gọi `init()` + `requestPermission()` + eager `fcmControllerProvider`. Đăng ký device token backend qua `client/devices` sau login (xem `_registerDevice` trong `login_controller.dart`). |
| **Socket (chat)** | `app/core/services/socket/socket_service.dart` · `socket_provider.dart` · `socket_event.dart` · `configs/socket_config.dart` | `SocketService` singleton, `socket_io_client`, URL `${EnvConfig.socketUrl}conversations`, auth `{'token': token}`. Sự kiện in ra **broadcast Stream** (`onNewMessage`, `onMessageSent/Read/Delivered`, `onTotalUnread`, `onConversationListUpdated`, `onError`...). `SocketCoordinator` (`socketCoordinatorProvider`, eager) xử lý badge + error toàn cục + `connectIfAuthenticated()` / force-logout khi `unauthorized`. Controller feature subscribe stream trong `build()`, huỷ ở `onDispose`. |
| **Deep link** | `app/core/services/deep_link/deep_link_service.dart` | `app_links`: `getInitialLink()` (cold start, chống double-consume qua SharedPreferences key) + `uriLinkStream`. `processPendingLink()` gọi từ AppShell sau khi mount (double post-frame). Nhánh payment hiện bị comment (**cần xác nhận**). |
| **Social login** | `login_controller.dart` (`loginWithLine`, `loginWithApple`) | LINE: `flutter_line_sdk` (`LineSDK.instance.setup` trong `main.dart` nếu có `lineChannelId`; login scope `profile/openid/email`, lấy `idTokenRaw`). Apple: `sign_in_with_apple`. Cả hai đổi `idToken` qua `authRepository.socialLogin(...)`; nếu `isLinked` → lưu token + `authMe.fetchMe()` + connect socket; chưa link → lưu temp token + `requireAction` để tiếp tục đăng ký. Bắt cancel (PlatformException/`AuthorizationErrorCode.canceled`) im lặng. |
| **Presigned upload** | `data/repositories/files_repository.dart` | 2 bước: (1) `api.createPresignedUrl(PresignedUrlRequest)` lấy `uploadUrl` + `fileUrl`; (2) **`Dio()` trần** (bỏ qua `AppInterceptor`) `PUT` bytes lên S3 (không gắn `Authorization` để tránh đụng chữ ký query-string). Trả `fileUrl` (key, vd `pr/<uuid>.png`) để lưu vào profile. `downloadFile` cũng dùng Dio trần. |
| **QR scan** | `features/scan_qr/scan_qr_page.dart` | `mobile_scanner` — `MobileScannerController` trong `State`, widget `MobileScanner`. |

---

## 8. Tiêu thụ backend API

App parents dùng `api/v1/client/*` (parent-profiles, children, devices) + `api/v1/auth/*`, `api/v1/conversations`, `api/v1/storage/*`. Danh mục endpoint backend (path, method, request/response): **`sitter-navi-docs/docs/backend/sitternavi-web-BE/overview/api-catalog.md`**. Đối chiếu contract ở đó thay vì đoán — endpoint khai báo tại `lib/data/api/app_endpoints.dart` + `app_api.dart`.
