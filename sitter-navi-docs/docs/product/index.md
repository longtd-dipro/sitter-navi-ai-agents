# Sitter Navi — Product / Business Docs

> Tài liệu **nghiệp vụ** (không phải kỹ thuật) mô tả sản phẩm Sitter Navi làm gì, cho ai, và các luồng chính. Tổng hợp ngược từ source code 2026-07-02. Chi tiết kỹ thuật (endpoint/entity) xem mục **backend/frontend/mobile overview**.

## Sản phẩm là gì

**Sitter Navi** là nền tảng **kết nối phụ huynh với người trông trẻ (babysitter/sitter)** cho thị trường Nhật Bản. Phụ huynh tìm và đặt sitter chăm con; sitter đăng ký lịch rảnh và nhận việc; đội vận hành (admin) điều phối toàn bộ qua web dashboard.

## Ai dùng (Actors)

| Actor | Là ai | Dùng gì |
|---|---|---|
| **Phụ huynh (Parent)** | Gia đình cần người trông trẻ | App mobile `sitternavi-app-parents` |
| **Sitter (Caregiver)** | Người trông trẻ cung cấp dịch vụ | App mobile `sitternavi-app-babysitter` |
| **Admin / Vận hành** | Đội back-office điều phối, kiểm duyệt, đối soát | Web dashboard `sitternavi-web` |

## Bản đồ tính năng (Feature Map)

| # | Nhóm tính năng | Mô tả ngắn |
|---|---|---|
| 01 | Đăng nhập & Onboarding | Đăng ký/đăng nhập email+OTP, LINE, Apple; định tuyến theo trạng thái hồ sơ |
| 02 | Tài khoản & Hồ sơ | Hồ sơ phụ huynh, hồ sơ sitter, quản lý thông tin con |
| 03 | Dịch vụ & Năng lực Sitter | Dịch vụ sitter cung cấp, chứng chỉ, đào tạo, lịch rảnh |
| 04 | Đặt lịch & Xếp ca | Phụ huynh đặt booking, sitter đăng ký ca (希望日), lịch làm việc, đánh giá |
| 05 | Check-in & Chăm sóc | Quét QR check-in/out, chấm công, báo cáo chăm sóc |
| 06 | Gói thành viên & Giá | Membership plan của phụ huynh, bảng giá dịch vụ |
| 07 | Thanh toán & Chi trả | Thanh toán của phụ huynh, chi trả (payout) cho sitter |
| 08 | Nhắn tin realtime | Chat giữa phụ huynh và sitter/admin |
| 09 | Thông báo | Push (FCM) + Email |
| 10 | Vận hành Back-office | Web admin quản lý sitter/job/project/billing + dữ liệu nền |

> **Sơ đồ hành trình tổng quan (End-to-end journey)** đã chuyển lên [trang Documentation chính](../index.md). Dùng menu trái để đọc chi tiết từng tính năng bên dưới.
