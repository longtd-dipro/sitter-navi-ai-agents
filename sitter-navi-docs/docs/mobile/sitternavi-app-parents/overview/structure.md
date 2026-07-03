# sitternavi-app-parents — Structure

> Nguồn: scan repo 2026-07-02. Cập nhật khi code đổi.

App Flutter cho **phụ huynh / người giám hộ (parents/guardians)** của SitterNavi. Dùng nhóm endpoint `api/v1/client/*` (parent-profiles, children, devices).

> **Nhận diện repo:** `pubspec.yaml` khai báo `name: myapp` — **giống hệt** app babysitter kế bên. Mọi import nội bộ là `package:myapp/...`. Xác định repo bằng **đường dẫn** (`repositories/sitternavi-app-parents`), KHÔNG bằng package name.

## Stack (từ pubspec.yaml)

| Layer | Package | Version |
|---|---|---|
| State | `hooks_riverpod` / `flutter_riverpod` / `riverpod` | 3.0.1 (HAND-WRITTEN providers, không codegen) |
| Hooks | `flutter_hooks` | 0.21.3+1 |
| Routing | `auto_route` (+ `auto_route_generator`) | 11.1.0 |
| HTTP | `dio` 5.9.2 + `retrofit` 4.9.2 | |
| Models | `freezed` 3.x + `json_serializable` 6.x + `json_annotation` 4.9 | |
| DI phụ | `injectable` 3.0.0 + `get_it` 9.2.1 | (dùng rất ít — xem mục DI) |
| i18n | `slang_flutter` 4.3 (base locale `ja`, thêm `en`) | |
| Realtime | `socket_io_client` 3.1.4 (chat) | |
| Push | `firebase_core/messaging/crashlytics` + `flutter_local_notifications` 20.1 + `app_badge_plus` | |
| Social login | `flutter_line_sdk` 2.7.2 + `sign_in_with_apple` 8.1 | |
| Deep link | `app_links` 7.0.0 | |
| QR | `mobile_scanner` 7.2.0 | |
| Env | `flutter_dotenv` 6.0.0 (`.env` / `.env.stg` / `.env.prod`) | |
| Sizing | `flutter_screenutil` 5.9.3 (design size 390×844) | |

Không có payment SDK, không maps SDK. (`socket_service.dart` còn sót method `waitForChargeResult` + `deep_link_service.dart` còn nhánh `payment-success/failed` bị comment — di sản chưa dùng, **cần xác nhận** nếu định bỏ.)

## Layout `lib/`

```
lib/
├── main.dart                 ← mainCommon(flavor): dotenv, LINE SDK, ProviderContainer override, FCM/deep-link/language/socket init, runApp
├── firebase_options.dart
├── flavor/                   ← 3 entrypoints: main_development.dart · main_staging.dart · main_production.dart
├── app/                      ← hạ tầng cross-cutting
│   ├── app.dart              ← MyApp (MaterialApp.router) + appRouterProvider + LoadingOverlay/ToastOverlay mount
│   ├── configs/              ← env_config.dart · socket_config.dart · constants.dart
│   ├── routers/              ← app_router.dart (RootStackRouter) + app_router.gr.dart (generated)
│   ├── core/
│   │   ├── network/          ← provider/dio_provider.dart · provider/network_provider.dart · interceptors/app_interceptor.dart · controller+state (connectivity) · cancel/ (request cancel manager)
│   │   ├── services/         ← push_notification/ · socket/ · deep_link/ · device_info/ · permission/
│   │   ├── prefs/            ← app_prefs.dart (AppPrefs over SharedPreferences)
│   │   ├── resources/        ← res_colors.dart · res_text_styles.dart · res_dm_text_styles.dart · res.dart
│   │   ├── enums/enums.dart  ← enums dùng chung (Gender, RequireAction, ConversationKind, NetworkStatus, AppFlavor, SocketErrorCode...)
│   │   ├── extensions/       ← string_ext · time_ext · files_exts · jp_extensions · function
│   │   └── utils/            ← error_utils (getApiErrorMessage) · dialog_ultis · bottomsheet_ultis
│   ├── base/                 ← page_effect_mixin.dart · base_page_mixin.dart
│   ├── helpers/              ← validation · message · toast · formatter · regex · otp_time · vibration · api
│   ├── widgets/              ← buttons · input · calendar · loading_overlay · toast_overlay · skeletons · pages
│   └── language/             ← controller/ provider/ state/ (đổi locale, lưu vào AppPrefs)
├── data/                     ← lớp dữ liệu DUY NHẤT
│   ├── api/                  ← app_api.dart (Retrofit AppApi) + app_api.g.dart · app_endpoints.dart · provider/api_provider.dart (appApiProvider)
│   ├── repositories/         ← *_repository.dart + provider/*_repository_provider.dart
│   ├── models/               ← DTO theo domain (xem dưới)
│   └── logs/                 ← app_logger.dart + app_logger_provider.dart
├── features/                 ← feature-first (xem dưới)
├── i18n/                     ← ja.i18n.json · en.i18n.json + strings*.g.dart (slang generated)
└── gen/                      ← assets.gen.dart · fonts.gen.dart · di/ (di.dart + di.config.dart)
```

## Feature folders (`lib/features/`)

Mỗi feature (theo skill) tách 4 lớp: `controller/` · `provider/` · `state/` · `ui/` (+ tuỳ chọn `widgets/`).

| Feature | Sub-features | Ghi chú |
|---|---|---|
| `splash` | (chỉ `ui/`) | Màn khởi động, điều hướng theo trạng thái đăng nhập |
| `start` | (chỉ `ui/`) | Onboarding / màn chọn login |
| `auth` | `login`, `register`, `forgot_password`, `me` | login gồm email/pass + OTP + LINE + Apple; `me` = auth_me (fetch profile) |
| `app_shell` | controller/provider/state/widgets | Shell bao ngoài, chứa các tab + coordinator deep link/socket |
| `bottom_bar` | (chỉ `ui/`) | Bottom navigation |
| `home` | controller/provider/state | Màn chính |
| `children` | `detail`, `add_edit` (+ `widgets/`) | Quản lý hồ sơ con — `api/v1/client/children` |
| `conversation` | controller/provider/state/widgets/ui | Chat list + màn chat (Socket.IO + REST) |
| `user` | `edit`, `detail` (+ `widgets/`) | Hồ sơ phụ huynh — `api/v1/client/parent-profiles/me` |
| `scan_qr` | `scan_qr_page.dart` (1 file) | Quét QR bằng `mobile_scanner` |
| `widgets` | — | Widget dùng chung cấp features |

> **Lưu ý gap:** các domain `favorite`, `product`, `notification`, `term`, `storage` **chỉ tồn tại trong `data/models/`**, KHÔNG có feature folder tương ứng — nhiều khả năng là di sản dùng chung với app babysitter, chưa build màn ở app parents (**cần xác nhận** trước khi giả định có UI).

## Routing (auto_route)

- Cấu hình: `lib/app/routers/app_router.dart` — `class AppRouter extends RootStackRouter` với `@AutoRouterConfig(replaceInRouteName: 'Page,Route')` (annotate page bằng `@RoutePage()`, tên `XxxPage` → route `XxxRoute`).
- Generated: `app_router.gr.dart` (không sửa tay). Regenerate bằng `dart run build_runner build --delete-conflicting-outputs`.
- Provider: `appRouterProvider = Provider<AppRouter>((ref) => AppRouter())` đặt trong `lib/app/app.dart`; `MyApp` dùng `router.config()`.
- `defaultRouteType = RouteType.cupertino()`.
- Cấu trúc: các route auth (Splash, Start, Login, Register, OTP, ForgotPassword, ResetPassword, Success...) ở top-level; toàn bộ màn sau đăng nhập nằm **nested trong `AppShellRoute`** (children: BottomBar [initial], ScanQR, ChildrenList, ConversationList, MessageChat, ImagePreview, AddEditChild, ChildrenDetail, UserDetail, EditUser).
- **Không có `AutoRouteGuard`.** Auth được xử lý bằng: Splash quyết định điều hướng theo token, và `AppInterceptor` bắt 401 → `appRouterProvider.replaceAll([LoginRoute()])` (xem `app_interceptor.dart`). Socket 401/unauthorized cũng force logout tương tự (`socket_provider.dart`).
- Điều hướng: `context.router.push(...)` / `replaceAll(...)` — không dùng `Navigator.push`. Có ~22 file annotate `@RoutePage()`.

## Data layer

- **Retrofit client duy nhất:** `lib/data/api/app_api.dart` — `@RestApi() abstract class AppApi` (factory `AppApi(Dio dio)`), generated `app_api.g.dart`. Truy cập qua `appApiProvider` (`lib/data/api/provider/api_provider.dart`) — wrap `dioProvider`.
- **Endpoint paths:** `lib/data/api/app_endpoints.dart` (`abstract class AppEndpoints`, hằng `const`). Nhóm endpoint:
  - **Auth** — `api/v1/auth/login`, `.../register`, `.../reset-password`, `.../social-login`, `.../otp/{email,resend,verify}`, `.../me`, `.../logout` *(lưu ý: một số hằng cũ như `auth/user/...` không mang prefix `api/v1` — legacy, cần xác nhận)*.
  - **Parent profile** — `api/v1/client/parent-profiles/me`.
  - **Children** — `api/v1/client/children` (+ `/{id}`).
  - **Conversation** — `api/v1/conversations` (+ `/{id}/messages`, `/unread-conversation-count`).
  - **Device** — `api/v1/client/devices` (+ `/{token}`).
  - **Storage** — `api/v1/storage/presigned-url`.
  - **App version** — `app/version`; **Notification device-token (legacy)** — `user/notifications/device-token`.
- **Dio + interceptor:** `dioProvider` (`app/core/network/provider/dio_provider.dart`) — baseURL từ `EnvConfig.baseUrl`, timeout 30s; gắn `AppInterceptor` + `NetworkInterceptor` (+ `LogInterceptor` chỉ ở `kDebugMode`). Chi tiết interceptor xem `patterns.md`.
- **Repositories:** `lib/data/repositories/<name>_repository.dart` (auth, parent_profile, children, conversation, files) — mỗi cái nhận `AppApi` trong constructor; kèm `provider/<name>_repository_provider.dart` là `Provider<Repo>((ref) => Repo(ref.watch(appApiProvider)))`.

### DI (injectable/get_it) — dùng rất hạn chế

- `configureDependencies()` (`lib/gen/di/di.dart`) gọi `initGetIt` (generated `di.config.dart`). Hiện **chỉ đăng ký 1 thứ**: `OtpTimerHelper` (`@lazySingleton` trong `lib/app/helpers/otp_time_helper.dart`). Chỉ file này có annotation injectable.
- **Toàn bộ DI thực tế đi qua Riverpod hand-written providers**, không qua get_it. `main.dart` tạo 1 `ProviderContainer` với override `sharedPrefsProvider.overrideWithValue(prefs)`, rồi eagerly `read` FCM / deep-link / language / socket coordinator trước `runApp` (dùng `UncontrolledProviderScope`).

## Models & conventions

- **Vị trí:** `lib/data/models/<domain>/` — domain: `app`, `auth/{login,register,forgot_password,otp,social_login}`, `children`, `conversation`, `parent_profile`, `device`, `user`, `notification`, `storage`, `term`, `favorite`, `product`, `toast`, `ext`.
- **Serialization hỗn hợp:** phần lớn (~90 file) dùng `@JsonSerializable` (json_serializable) sinh `*.g.dart`; một số model/state dùng `freezed` sinh `*.freezed.dart` (+ `*.g.dart` nếu có JSON). Xem `patterns.md`.
- **Envelope generic** trong `models/ext/`: `ApiResponse<T>` (`{data, statusCode, message}`), `CursorResponse<T>` (`{data, nextCursor, hasMore}` — chat), `PaginatedResponse<T>` (page-based — children), `ParamsRequest`. Tất cả `genericArgumentFactories: true`.
- **State** feature nằm ở `features/<f>/state/<f>_state.dart` (Freezed). **Controller** ở `controller/`, **provider** hand-written ở `provider/`.
- **Naming:** file/folder `snake_case`; class/enum `PascalCase`; var/func `camelCase`. Không sửa file generated (`*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `strings*.g.dart`).

## i18n (slang)

- Config `slang.yaml`: `base_locale: ja`, `fallback_strategy: base_locale`, chỉ thêm `en`. `key_case: snake`, `lazy: true`, `translate_var: t`.
- Nguồn: `lib/i18n/ja.i18n.json` + `en.i18n.json` → generated `strings.g.dart`, `strings_ja.g.dart`, `strings_en.g.dart` (regenerate: `dart run slang build`).
- Đọc chuỗi: `t.<key>` hoặc `context.t` (mixin `PageEffectMixin` expose getter `t = Translations.of(context)`). `MyApp` bọc trong `TranslationProvider`, `supportedLocales = AppLocaleUtils.supportedLocales`. Không hardcode chuỗi hiển thị.

## Backend API tham chiếu

App gọi `api/v1/client/*`. Catalog endpoint backend: `sitter-navi-docs/docs/backend/sitternavi-web-BE/overview/api-catalog.md`.
