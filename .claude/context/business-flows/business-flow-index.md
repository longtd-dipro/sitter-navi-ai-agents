# Business Flow Index — Sitter Navi

> Master index các domain nghiệp vụ **đang có** trong sản phẩm (reverse-engineer từ source code 2026-07-03). `ba-agent` đọc file này ở Bước 1 để biết feature mới thuộc domain nào, rồi **đọc doc chi tiết tương ứng** trước khi hỏi user — tránh hỏi lại thứ đã có sẵn.
>
> **Doc chi tiết** (flow đầy đủ + sơ đồ Mermaid + màn hình + gaps) nằm ở `sitter-navi-docs/docs/product/`. File này chỉ là bảng tra cứu — đọc on-demand đúng domain, không load hết.

## Actors (nhắc lại — chi tiết ở `specification.md`)

- **Parent** (`parent`) — phụ huynh, app `sitternavi-app-parents`, endpoint `api/v1/client/*`
- **Caregiver/Sitter** (`caregiver`) — sitter, app `sitternavi-app-babysitter`, endpoint `api/v1/sitter/*`
- **Admin** (`admin`) — vận hành, web `sitternavi-web`, endpoint `/v1/admin/*`

## Bảng tra cứu domain

| Domain | Actors chính | BE modules liên quan | Doc chi tiết |
|---|---|---|---|
| **Đăng nhập & Onboarding** | Parent, Sitter, Admin | auth, user, permission | `docs/product/01-auth-onboarding.md` |
| **Tài khoản & Hồ sơ** | Parent, Sitter | parent-profile, sitter-profile, children, emergency-contact | `docs/product/02-accounts-profiles.md` |
| **Dịch vụ & Năng lực Sitter** | Sitter, Admin | sitter-service, sitter-certification, sitter-availability, training-course, training-session, care-service | `docs/product/03-sitter-services.md` |
| **Đặt lịch & Xếp ca** | Parent, Sitter, Admin | booking, sitter-availability, review (+ sitter working-patterns 希望日) | `docs/product/04-booking-scheduling.md` |
| **Check-in & Chăm sóc** | Sitter, Parent, Admin | attendance-log, care-service (care report) | `docs/product/05-attendance-care.md` |
| **Gói thành viên & Giá** | Parent, Admin | membership-plan, parent-membership, service-price | `docs/product/06-membership-pricing.md` |
| **Thanh toán & Chi trả** | Parent, Sitter, Admin | payment, payout | `docs/product/07-payment-payout.md` |
| **Nhắn tin realtime** | Parent, Admin *(sitter app KHÔNG có chat)* | conversation | `docs/product/08-messaging-chat.md` |
| **Thông báo** | tất cả | push-notification, email | `docs/product/09-notifications.md` |
| **Vận hành Back-office + Master data** | Admin | (web admin) + japan-address, faq, category, product, permission | `docs/product/10-admin-operations.md` |

## Gaps nghiệp vụ nổi bật (BA cần biết khi làm SPEC)

- **Chưa có cổng thanh toán ngoài** — payment/payout mới là sổ nội bộ; luồng charge thật chưa hoàn chỉnh (xem doc 07).
- **App sitter không có chat** — socket app sitter dùng cho kết quả thanh toán, không phải chat (xem doc 08).
- **Route admin reserved chưa build**: attendance, reservations, contact-book, payroll, notifications (xem doc 10).
- **2 endpoint admin hở auth** (parent-profiles, permissions — `@Auth` bị comment).

## Nguồn & cập nhật

- Doc chi tiết sinh reverse từ overview docs (`docs/backend/.../api-catalog.md`, `erd.md`) + source. Khi code đổi → cập nhật overview trước, rồi doc product.
- File index này = bảng tra cứu; thêm domain mới → thêm 1 dòng + tạo `docs/product/NN-<slug>.md`.
