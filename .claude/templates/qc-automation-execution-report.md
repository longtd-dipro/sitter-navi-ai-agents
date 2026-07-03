# Template — QC Automation Execution Report

> Dùng bởi `qc-automation-agent.md` (Bước 7). Output path: `<e2e-repo>/reports/<feature-name>/execution-report.md`.

```markdown
## Execution Report — <Feature> | <target-app> | <ngày giờ>

**URL:** <website-url>
**Browser:** Chromium
**Nguồn TC:** <TC-driven: <testcases path> | SPEC-driven: SPEC.md>
**Total:** X passed / Y failed / Z skipped

---

| TC ID | Mô tả | Status | Duration | Ghi chú |
|---|---|---|---|---|
| TC_SO_001 | Đăng nhập thành công | ✅ PASS | 1.2s | |
| TC_SO_002 | Đăng nhập sai mật khẩu | ❌ FAIL | 3.5s | Screenshot: reports/.../TC_SO_002.png |
| TC_SO_003 | Đăng xuất | ⏭ SKIP | — | Cần data động |

---

## Lỗi cần xử lý

| TC ID | Error | Khả năng nguyên nhân |
|---|---|---|
| TC_SO_002 | Expected text "..." not found | Selector sai hoặc toast chưa implement |

---

## Bước tiếp theo
→ FAIL: Dev xem screenshot + error, fix rồi báo chạy lại
→ PASS toàn bộ: Sẵn sàng demo / release
→ SKIP: Ghi nhận, implement sau
```
