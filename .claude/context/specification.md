# Sitter Navi — Business Specification Memory

> Long-term business memory cho dự án. `ba-agent` và `pm-agent` đọc file này trước khi tạo SPEC/PLAN. Khung ban đầu được `/init-kit` điền từ câu trả lời setup; nội dung chi tiết bổ sung dần theo thời gian bởi BA/PM — không phải sinh 1 lần rồi bỏ.

---

## Business Context

**Sitter Navi** — nền tảng kết nối (matching) phụ huynh với người trông trẻ (babysitter/caregiver) cho thị trường **Nhật Bản** (UI mặc định `ja`, font Noto Sans JP). Phụ huynh tìm và đặt sitter, quản lý con & lịch chăm sóc; sitter đăng ký hồ sơ + lịch làm việc mong muốn và nhận job; admin vận hành toàn bộ hệ thống qua web dashboard. Có chat realtime (Socket.IO), push notification (FCM), quản lý booking / attendance / care-report, membership plan, billing & payout.

> Actors, integration nghiệp vụ, business rule chi tiết bổ sung dần bởi BA khi làm SPEC từng feature. Hiện tại **chưa có document feature nào** — cần scan code để dựng feature docs (task riêng, không thuộc init-kit).

---

## Epics / Repos

_(đồng bộ với bảng Ecosystem trong `AGENTS.md` — không lặp lại thủ công, chỉ tham chiếu nếu cần thêm mô tả nghiệp vụ mà bảng Ecosystem không có chỗ)_

---

## Actors

> Nguồn: scan thực tế `RoleType` enum backend + endpoint namespace + feature folders (2026-07-02). Giá trị enum backend: `admin` / `parent` / `caregiver`.

| Actor | Mô tả | Repo chính |
|---|---|---|
| **Parent** (`parent`) | Phụ huynh/người giám hộ — đăng ký & quản lý thông tin con, tìm/xem sitter, đặt booking, chat với sitter | `sitternavi-app-parents` (mobile) · backend `api/v1/client/*` |
| **Caregiver / Sitter** (`caregiver`) | Người trông trẻ — quản lý hồ sơ sitter, đăng ký lịch làm việc mong muốn (希望日登録), xem & nhận job, work schedule, chat | `sitternavi-app-babysitter` (mobile) · backend `api/v1/sitter/*` |
| **Admin / Operator** (`admin`) | Vận hành back-office — quản lý sitter, job, project/booking, children care-report, chat, billing/payout qua web dashboard | `sitternavi-web` (frontend) · backend `*.controller.admin.ts` (`/v1/admin/*`) |

> Lưu ý naming: enum backend là `caregiver` nhưng module/domain dùng tiền tố `sitter-*` — cùng 1 actor.

---

## Phase Gate

_(điền bởi `pm-agent` khi có timeline/milestone thật — để trống nếu dự án không theo mô hình phase-gate)_

| Gate | Milestone | Ngày dự kiến |
|---|---|---|
| | | |

---

## Ghi chú nghiệp vụ khác

_(BA bổ sung dần: quy tắc nghiệp vụ đặc thù, integration bên ngoài liên quan trực tiếp tới nghiệp vụ, v.v. — không phải chi tiết kỹ thuật, đó là việc của `technical.md`)_
</content>
