# Template — QC Automation Output Summary

> Dùng bởi `qc-automation-agent.md` (Output tổng kết). Post vào chat sau khi chạy Playwright xong.

```
## QC Automation Output — <Feature> | <target-app> | <ngày>

### Nguồn TC
- Chế độ: <TC-driven (<file path>) | SPEC-driven>
- Số TC: N

### Artifacts đã tạo
- Spec files:       e2e/<target-app>/<feature>/ (N files)
- Execution report: reports/<feature>/execution-report.md
- Screenshots:      reports/<feature>/screenshots/ (chỉ FAIL)

### Kết quả
- ✅ PASS: X / N
- ❌ FAIL: Y / N  → xem report để biết chi tiết
- ⏭ SKIP: Z / N

### Bước tiếp theo
→ Fix FAIL: Dev xem reports/<feature>/execution-report.md
→ Chạy lại sau fix: "Hãy là QC Automation, test feature: <path>, Figma: <url>, app: <target-app>, website: <url>, testcases: <tc-path>"
→ Thêm TC mới: thêm row vào TC file rồi chạy lại
```
