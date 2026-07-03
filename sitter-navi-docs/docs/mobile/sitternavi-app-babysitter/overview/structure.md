# sitternavi-app-babysitter — Structure

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

App Flutter dành cho **BABYSITTER / CAREGIVER (sitter)** trong hệ Sitter Navi. Gọi backend qua nhóm endpoint `api/v1/sitter/*` (calendar/events, working-patterns 希望日登録) + `api/v1/client/*` (master data, sitter-profiles/me) + `api/v1/auth/*`.

> **Lưu ý nhận diện repo:** `pubspec.yaml` có `name: myapp` (trùng với app parents anh em) → mọi import là `package:myapp/...`. Nhận diện repo qua **đường dẫn**, không qua tên package. File `.claude/CLAUDE.md` bị dán nhãn nhầm là "parents app" (boilerplate copy) — **CODE mới là nguồn chính xác**, đây là app SITTER.

---

## 1. Stack (từ pubspec.yaml)

| Layer | Package | Ghi chú |
|---|---|---|
| State | `hooks_riverpod` 3.0.1 + `flutter_hooks` | Provider **VIẾT TAY**, KHÔNG codegen (không `@riverpod`, không `*.g.dart` cho provider) |
| Routing | `auto_route` 11.1.0 | Config `@AutoRouterConfig`, file `.gr.dart` generated |
| Networking | `retrofit` 4.9.2 + `dio` 5.9.2 | `AppApi` (Retrofit) + interceptor tự viết |
| Models | `freezed_annotation` 3.1.0 + `json_annotation` 4.9.0 | freezed cho state; json_serializable cho model DTO |
| DI phụ | `injectable` + `get_it` | Chỉ dùng rất hạn chế (xem mục 6) |
| i18n | `slang_flutter` 4.3.0 | `baseLocale: AppLocale.ja` (default **ja**), có en |
| Push | `firebase_core/messaging/crashlytics` + `flutter_local_notifications` + `app_badge_plus` | |
| Realtime | `socket_io_client` 3.1.4 | Dùng cho **kết quả thanh toán (charge)**, không phải chat — xem patterns.md |
| Deep link | `app_links` | |
| QR | `mobile_scanner` 7.2.0 | Màn scan_qr (check-in) |
| Calendar UI | `table_calendar` | Dùng trong `monthly_calendar_widget.dart` |
| Sizing | `flutter_screenutil` 5.9.3 | designSize 390×844 |
| Prefs | `shared_preferences` | |

Không có payment SDK (thanh toán chỉ hiển thị dạng data + chờ kết quả qua socket). Không có maps SDK.

---

## 2. Directory layout `lib/`

```
lib/
├── main.dart                 ← mainCommon(): dotenv, ProviderContainer, FCM/deeplink/language init, DI, runApp
├── firebase_options.dart
├── flavor/                   ← main_development.dart / main_staging.dart / main_production.dart (entrypoint per flavor)
├── app/                      ← hạ tầng dùng chung (KHÔNG phải feature)
│   ├── app.dart              ← MyApp (MaterialApp.router) + appRouterProvider; mount LoadingOverlay + ToastOverlayWidget
│   ├── base/                 ← BasePageMixin (Scaffold/AppBar config) · PageEffectMixin (onInit/onDispose + getter `t`)
│   ├── configs/              ← constants · env_config · socket_config
│   ├── core/
│   │   ├── enums/            ← enums.dart (RequireAction, BootstrapStatus, AppFlavor, NetworkStatus...)
│   │   ├── extensions/       ← function · jp_extensions · string_ext · time_ext
│   │   ├── network/
│   │   │   ├── interceptors/app_interceptor.dart   ← AppInterceptor + NetworkInterceptor
│   │   │   ├── provider/dio_provider.dart           ← dioProvider (Dio + interceptors)
│   │   │   ├── provider/network_provider.dart       ← trạng thái online/offline
│   │   │   ├── cancel/…, cancel_token_mixin.dart    ← CancelToken per-screen
│   │   │   └── controller/, state/
│   │   ├── prefs/app_prefs.dart          ← appPrefsProvider (token, flags) + sharedPrefsProvider
│   │   ├── resources/        ← res · res_colors · res_text_styles · res_dm_text_styles
│   │   ├── services/         ← deep_link · device_info · permission · push_notification(fcm) · socket
│   │   └── utils/            ← bottomsheet · date_picker · dialog · error_utils
│   ├── helpers/              ← api · file_pick · formatter · message · mime · otp_time · regex · toast · validation · vibration …
│   ├── language/             ← language controller/provider/state (đổi locale slang)
│   ├── routers/              ← app_router.dart (AppRouter) · app_router.gr.dart (gen) · startup_router.dart
│   └── widgets/              ← buttons, calendar, input, loading_overlay, toast_overlay, skeletons, pages/success_page
├── data/
│   ├── api/
│   │   ├── app_api.dart      ← @RestApi() AppApi (Retrofit interface)
│   │   ├── app_api.g.dart    ← generated
│   │   ├── app_endpoints.dart← hằng đường dẫn endpoint
│   │   └── provider/api_provider.dart   ← appApiProvider
│   ├── models/               ← DTO theo domain (auth, calendar, working_pattern, sitter_profile, storage, ext, …)
│   ├── repositories/         ← ApiRepository · AuthRepository · FilesRepository + provider/
│   └── logs/                 ← app_logger + provider
├── i18n/                     ← ja.i18n.json · en.i18n.json · strings.g.dart (+ strings_ja/en.g.dart) slang gen
├── gen/                      ← di/di.dart + di.config.dart (injectable) · assets.gen.dart · fonts.gen.dart
└── features/                 ← 1 thư mục / feature (xem mục 3)
```

---

## 3. Feature folders (`lib/features/`)

Mỗi feature "đủ 4 lớp" theo cấu trúc: `state/` (freezed) · `controller/` (Notifier) · `provider/` (NotifierProvider viết tay) · `ui/` (page `@RoutePage()`).

| Feature | Nội dung | Lớp |
|---|---|---|
| `splash/` | Bootstrap gate (chỉ ui) | ui |
| `start/` | Màn khởi động/onboarding (chỉ ui) | ui |
| `auth/login/` | Đăng nhập | 4 lớp |
| `auth/change_password/` | Đổi mật khẩu (+ success page) | controller/state/ui |
| `auth/forgot_password/` | Quên MK: email → OTP → reset → success | 4 lớp |
| `edit_sitter_profile/` | Sửa hồ sơ sitter + confirm page (onboarding/complete profile) | 4 lớp |
| `app_shell/` | Vỏ app sau đăng nhập (+ modal_update_app, vibration_state) | 4 lớp |
| `bottom_bar/` | Bottom navigation (chỉ ui) | ui |
| `home/` | Trang chủ | 4 lớp |
| `work_schedule/` | Lịch làm việc (予約・研修 + 希望日一覧, table_calendar) | 4 lớp |
| `shift_register/` | Đăng ký ca/希望日 + confirm page | 4 lớp |
| `booking_detail/` | Chi tiết booking (chỉ ui) | ui |
| `session/` | Session/`/me` bootstrap + requireAction (không có ui) | controller/state/provider |
| `notification/` | Danh sách/thông báo | 4 lớp |
| `scan_qr/` | Quét QR (check-in), `mobile_scanner` (chỉ ui, state cục bộ) | ui |
| `user/` | Hồ sơ/tài khoản user (+ modal delete account, action buttons) | 4 lớp |
| `widgets/` | Widget dùng chung cấp feature (app_bar, cached_image, dialog, status_badge, step_indicator, notification_tile…) | — |

---

## 4. Routing (auto_route)

- Cấu hình: `lib/app/routers/app_router.dart` — `class AppRouter extends RootStackRouter` với `@AutoRouterConfig(replaceInRouteName: 'Page,Route')`. `defaultRouteType = RouteType.cupertino()`.
- Generated: `app_router.gr.dart` (chạy build_runner để cập nhật).
- Provider: `appRouterProvider = Provider<AppRouter>((ref) => AppRouter())` (khai trong `app.dart`). Điều hướng ngoài widget qua `ref.read(appRouterProvider)` (interceptor dùng cách này để `replaceAll([LoginRoute()])` khi 401).
- Route khai trong danh sách `routes` của AppRouter: `SplashRoute` (initial) → `StartRoute`, `LoginRoute`, `ChangePasswordRoute(+Success)`, `EditSitterProfileRoute`, `ConfirmSitterProfileRoute`, forgot-password chain, `ScanQRRoute`, `NotificationRoute`, `ShiftRegisterRoute(+Confirm)`, `BookingDetailRoute`, `SuccessRoute`, và `AppShellRoute` chứa child `BottomBarRoute` (initial).
- **Guard/gate:** không dùng `AutoRouteGuard` class. Gate điều hướng làm thủ công qua `startup_router.dart` → hàm `routesForRequireAction(RequireAction)`: map `RequireAction` từ server (`/me`) sang stack đích — `changePassword` → ChangePasswordRoute; `completeProfile` → EditSitterProfileRoute(isOnboarding:true); `registerInfo`/`none` → AppShellRoute. Splash + login đọc kết quả này. Auth-gate 401 xử lý trong `AppInterceptor.onError` (clear prefs/session, về LoginRoute).

---

## 5. Data layer

- **Retrofit client:** `lib/data/api/app_api.dart` — `@RestApi() abstract class AppApi { factory AppApi(Dio dio, {String? baseUrl}) = _AppApi; }`. Mỗi method annotate `@GET/@POST/@PATCH` + `@Extra({...})` (cờ `showLoading`/`skipShowError` cho interceptor) + `@CancelRequest()` cho CancelToken.
- **Endpoints:** `lib/data/api/app_endpoints.dart` — `abstract class AppEndpoints` chứa hằng string. Nhóm:
  - Auth: `api/v1/auth/login`, `.../otp/email|resend|verify`, `.../change-password`, `.../me`; (một số path cũ không có prefix v1: `auth/user/forgot-password`, `resend-otp`, `reset-password`, `logout`).
  - App version: `app/version`. Notification: `user/notifications/device-token`.
  - Master data (client): `api/v1/client/prefectures|municipalities|postal-codes|care-services`.
  - Sitter profile: `api/v1/client/sitter-profiles/me` (GET + PATCH).
  - Storage: `api/v1/storage/presigned-url`.
  - **Sitter (đặc thù app này):** `api/v1/sitter/calendar/events` (GET, query `from/to/mode`), `api/v1/sitter/working-patterns` (POST — 希望日登録).
- **Dio + interceptor:** `dioProvider` (`lib/app/core/network/provider/dio_provider.dart`) tạo Dio (baseUrl `EnvConfig.baseUrl`, timeout 30s), add `AppInterceptor` + `NetworkInterceptor` (+ `LogInterceptor` ở debug). `AppInterceptor` (`app_interceptor.dart`): gắn `Authorization: Bearer <token>` từ `appPrefs.token`; bật/tắt `globalLoadingProvider` theo `extra.showLoading`; xử lý 401 (clear + về login); hiện toast lỗi qua `ToastHelper` trừ khi `extra.skipShowError`. `NetworkInterceptor` reject request khi offline.
- **Repositories:** 3 lớp repository, mỗi lớp có provider viết tay trong `data/repositories/provider/`:
  - `ApiRepository` (`repository.dart`) + `repositoryProvider` — bọc `appApiProvider`, unwrap envelope (`.data` / `.items`), build filter CRUD-style (`prefectureId||$eq||...`), map `CalendarEventMode` → query.
  - `AuthRepository` + `authRepositoryProvider` — login, getMe…
  - `FilesRepository` + `filesRepositoryProvider` — upload/download qua presigned URL (dùng Dio trần, bypass interceptor).
- **DI (get_it/injectable):** `lib/gen/di/di.dart` → `configureDependencies()` gọi `initGetIt` (gen ở `di.config.dart`). Hiện chỉ đăng ký `OtpTimerHelper` (`@lazySingleton`). Phần lớn wiring đi qua **Riverpod provider**, không qua get_it.

---

## 6. Naming & conventions

- Import luôn là `package:myapp/...` (do `name: myapp`).
- **Feature layout chuẩn:** `features/<feature>/{state,controller,provider,ui}/<feature>_<layer>.dart`. State freezed đặt tại `state/<feature>_state.dart` (+ `.freezed.dart` generated). Controller `extends Notifier<State>`. Provider viết tay `NotifierProvider.autoDispose<C,S>(C.new)`.
- **Model/DTO** nằm ở `data/models/<domain>/*.dart` (+ `*.g.dart` json_serializable). State (freezed) nằm trong feature. Không lẫn hai loại.
- File generated: `*.g.dart` (json/retrofit/injectable), `*.freezed.dart` (freezed), `*.gr.dart` (auto_route), `strings*.g.dart` (slang) — không sửa tay, chạy `fvm dart run build_runner build --delete-conflicting-outputs`.
- **i18n (slang):** default `AppLocale.ja`, có `en`. Truy cập qua getter `t` (mixin `PageEffectMixin`: `Translations.of(context)`) → `t.<key>` trong build/callback. Nguồn: `lib/i18n/ja.i18n.json`, `en.i18n.json`.
- Resources: màu qua `ResColors()`, typography qua `ResTextStyles()`; kích thước qua `flutter_screenutil` (`.w/.h/.sp`, design 390×844).
- Page dùng `@RoutePage()` + `ConsumerWidget`/`ConsumerStatefulWidget`/`HookConsumerWidget`; page có state mix `PageEffectMixin`/`BasePageMixin`.
- Flavor: 3 entrypoint `lib/flavor/main_*.dart` gọi `mainCommon(AppFlavor.x)`; env qua `flutter_dotenv` (`flavor.envFileName`).
