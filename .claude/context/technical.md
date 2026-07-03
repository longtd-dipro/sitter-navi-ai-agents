# Sitter Navi — Technical Memory

> `techlead-design-agent` và `backend-agent` đọc file này trước khi thiết kế/implement. Khung ban đầu được `/init-kit` điền từ câu trả lời setup (bảng Tech Stack); phần CI/CD và Known Bugs bổ sung dần bởi Tech Lead/Dev khi phát hiện — đây là nơi ghi lại "known bug đã gặp + cách fix" để AI không lặp lại sai lầm cũ.

---

## Tech Stack

> Từ scan thực tế 2026-07-02. ⚠️ 2 điểm **khác mặc định kit**: web dùng **Next.js (App Router), KHÔNG phải Vite**; forms dùng **zod, KHÔNG phải yup**.

| Repo | Vai trò | Technology |
|---|---|---|
| `sitternavi-web-BE` | Backend | NestJS 11 · TypeORM 0.3.26 · PostgreSQL (pg 8) · Redis + BullMQ · `@nestjs/websockets` · `@dataui/crud-typeorm` · nestjs-i18n |
| `sitternavi-web` | Frontend | **Next.js 16 (App Router)** · React 19 · Ant Design 6 · TailwindCSS 4 · Redux Toolkit 2 (chỉ UI state) · TanStack Query 5 (server state) · react-hook-form 7 + **zod 4** · axios · socket.io-client 4 |
| `sitternavi-app-parents` | Mobile | Flutter · hooks_riverpod 3.0.1 (hand-written, KHÔNG codegen) · Retrofit 4 + Dio 5 · auto_route 11 · freezed 3 · injectable/get_it · slang (i18n) |
| `sitternavi-app-babysitter` | Mobile | (giống parents) |
| Database | — | PostgreSQL + TypeORM |
| Secrets | — | AWS (env-based hiện tại; xem POLICIES → chuẩn hoá Parameter Store) |
| Environments | — | DEV / STG / PROD |

### Integrations

Từ scan thực tế (chủ yếu ở backend `sitternavi-web-BE`, tiêu thụ bởi mobile/web):

- **AWS S3** (`@aws-sdk/client-s3`) — module `storage`, presigned upload/get URL; local dev dùng RustFS. Env: `AWS_ACCESS_KEY`, `S3_BUCKET`, `AWS_REGION`, `S3_ENDPOINT`.
- **AWS SES** (`client-ses`/`client-sesv2`) — module `email`, bật qua `ENABLE_AWS_SES`; fallback SMTP/Nodemailer (MailHog local).
- **Firebase Cloud Messaging** (`firebase-admin` ở BE; `firebase_messaging` ở mobile) — module `push-notification`, batch 500 token, rate-limit token-bucket, auto-unregister token chết. No-op nếu `FCM_CREDENTIAL_PATH` chưa set.
- **Socket.IO** — realtime. BE gateway `conversation.gateway.ts` (namespace `/conversations`). ⚠️ Mục đích khác nhau theo client: **web** + **app parents** dùng cho **chat** (conversation); **app babysitter** dùng socket cho **kết quả thanh toán** (`charge.succeeded`/`charge.failed`, `waitForChargeResult`) — app babysitter KHÔNG có chat.
- **Social login (mobile)** — LINE (`flutter_line_sdk`) + Apple (`sign_in_with_apple`) + email/OTP; BE exchange qua `api/v1/auth/social-login`.
- **Redis + BullMQ** — cache, distributed lock, background workers, outbox dispatch.
- **Payment gateway** — ⚠️ **CHƯA tích hợp gateway ngoài**. Module `payment`/`payout` chỉ là ledger CRUD nội bộ (`payment.entity.ts`, `payout.entity.ts`). Charge/settlement thật chưa có — coi là gap khi task yêu cầu.

---

## CI/CD

_(để trống — Tech Lead điền khi dự án có pipeline thật: build/test/deploy flow, secrets cần thiết, branch protection...)_

---

## Known Bugs / Gotchas

_(để trống — bổ sung mỗi khi Dev gặp 1 bug non-obvious đáng nhớ, kèm nguyên nhân + fix pattern, để agent không tái phạm. Ví dụ format:)_

```
### <Tên bug ngắn>
- Endpoint/File: <path>
- Nguyên nhân: <root cause>
- Fix: <pattern áp dụng>
```

---

## Doc Structure

Khi tạo DESIGN.md, xem `.claude/context/doc-structure.md` để đặt file đúng vị trí.
</content>
