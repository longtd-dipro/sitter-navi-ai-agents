# Screen Code Rule — Sitter Navi

> Quy tắc đặt **Screen Code** cho cột `Screen Code` + `App` trong `SPEC.md ## Screens`. Vì team **không dùng Epic code** (chốt khi `/init-kit`), Module prefix lấy theo **repo có màn hình** (backend không có screen).

## Format

```
<Module(2)>_<Feature(4)>_<Seq(3)>
```

- **Module (2 ký tự)** — repo đích chứa màn hình:

  | Prefix | Repo | Actor |
  |---|---|---|
  | `PA` | `sitternavi-app-parents` | Parent |
  | `SB` | `sitternavi-app-babysitter` | Sitter/Caregiver |
  | `AD` | `sitternavi-web` (web admin) | Admin |

  > `sitternavi-web-BE` không có màn hình → không có prefix. Cột `App` trong SPEC điền `PA`/`SB`/`AD` (thay cho Epic code).

- **Feature (4 ký tự HOA)** — viết tắt tên feature. Bảng khuyến nghị (đồng bộ business-flow-index):

  | Domain | Code |
  |---|---|
  | Đăng nhập & Onboarding | `AUTH` |
  | Tài khoản & Hồ sơ | `PROF` |
  | Dịch vụ & Năng lực Sitter | `SITT` |
  | Đặt lịch & Xếp ca | `BOOK` |
  | Check-in & Chăm sóc | `ATTN` |
  | Gói thành viên & Giá | `MEMB` |
  | Thanh toán & Chi trả | `PAYM` |
  | Nhắn tin realtime | `CHAT` |
  | Thông báo | `NOTI` |
  | Vận hành Back-office | `ADMN` |

- **Seq (3 số)** — `001`, `002`... theo thứ tự màn hình trong feature.

## Ví dụ

| Screen Code | Màn hình | App | Actor |
|---|---|---|---|
| `PA_AUTH_001` | Đăng nhập (app phụ huynh) | PA | Parent |
| `PA_BOOK_001` | Chọn sitter & đặt lịch | PA | Parent |
| `SB_SITT_001` | Đăng ký ca rảnh 希望日 | SB | Sitter |
| `AD_ADMN_001` | Danh sách quản lý sitter | AD | Admin |

## Ràng buộc

- **Unique toàn dự án** — không trùng Screen Code giữa các feature.
- 1 màn hình xuất hiện ở 2 app (vd cùng luồng chat) → 2 code riêng theo prefix repo (`PA_CHAT_001` vs `AD_CHAT_001`).
- Feature code chưa có trong bảng → BA tự đặt 4 ký tự HOA rõ nghĩa + thêm vào bảng này.
