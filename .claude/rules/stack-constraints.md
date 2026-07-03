# Stack Constraints — Không được vi phạm

## Tech Stack Cố định

| Layer | Bắt buộc dùng | Tuyệt đối không dùng |
|---|---|---|
| Database | PostgreSQL + TypeORM | MySQL, MongoDB, SQLite, Prisma |
| API Style | REST | GraphQL, gRPC, tRPC |
| Payment | **Chưa tích hợp gateway ngoài** — `payment`/`payout` là ledger CRUD nội bộ. Khi cần charge/settlement thật → xác nhận gateway với PM trước | Tự ý thêm Stripe/PayPal/VNPay khi chưa chốt |
| Mobile State | `hooks_riverpod` 3 (hand-written, KHÔNG codegen) | Provider, BLoC, GetX, MobX, `@riverpod` annotation/codegen |
| Mobile HTTP | Retrofit + Dio | `http` package, `chopper` |
| Mobile Routing | `auto_route` | `go_router`, `Navigator.push` trực tiếp |
| Web Framework | **Next.js 16 (App Router)** | Vite, Pages Router, react-router-dom |
| Web Server State | TanStack Query v5 | Redux Toolkit cho server data |
| Web Client State | Redux Toolkit v2 (chỉ UI state) | Context API cho auth/global state |
| Web Forms | react-hook-form + **zod** | Formik, yup, AntD Form.Item rules |
| Secrets | AWS Parameter Store | `.env` production, hard-code |

## Version Cố định (không tự nâng cấp)

| Package | Version | Ghi chú |
|---|---|---|
| TypeORM | 0.3.x | Có known bug với `orderBy` — xem postgresql.md |
| Ant Design | v6 | Breaking changes từ v5 — check migration guide |
| TanStack Query | v5 (5.100.x) | Object syntax, không positional |
| Next.js | 16.x | App Router (`src/app/`), route groups `(auth)`/`(dashboard)` |
| React | 19.x | |
| Redux Toolkit | 2.x | Chỉ dùng cho UI state (sidebar...), KHÔNG cho server data |
| zod | 4.x | Schema tại `src/libs/schema/*.schema.ts`, dùng với `@hookform/resolvers` |
| TailwindCSS | v4 | Config via PostCSS, không `tailwind.config.js` cũ |
| hooks_riverpod | 3.0.1 | Hand-written providers, không `.g.dart` cho provider |

## Mobile Version Convention

```
DEV:  0.0.<build_number>
STG:  0.1.<build_number>
PROD: 1.0.<build_number>
```

Không đảo ngược, không bỏ qua STG để lên thẳng PROD.
