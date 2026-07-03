# Template — Designer Agent Handover & Design Notes

> Dùng bởi `designer-agent.md` (Bước 5b + Bước 6).

---

## Design Notes — append vào ## Open Questions của SPEC.md (Bước 5b)

Khi Designer phát hiện design gap (state chưa rõ, flow chưa confirm, token không map được, ambiguity về component, modal pattern chưa có) → append vào section `## Open Questions` trong SPEC.md với prefix `[Design]`:

```markdown
## Open Questions

... (BA's existing questions) ...

### Design notes (Designer Agent — <YYYY-MM-DD>)

- [Design] State "loading" của <Component> chưa được mô tả trong SPEC. Đề xuất: skeleton rows 3-5 dòng. Cần BA confirm.
- [Design] Button "<label>" trong filter bar — hover state? Đề xuất: darken 8% theo design system.
- [Design] Màu `<hex>` trên Figma không có trong design token của dự án. Designer dùng nearest semantic token. Cần Design Lead confirm.
- [Design] Modal <Screen Code> chưa có pattern trong component library. Designer đề xuất tạo mới. Cần Designer Lead duyệt.
```

Mục đích: BA / Tech Lead / Dev đọc được những gap design ngay trong SPEC.md, không phải xem comment Figma.

---

## Handover output — post vào chat (Bước 6)

```
✅ Figma HIGH-FIDELITY hoàn thành: <feature>

Files đã update:
  - SPEC.md ## Screens — Figma Link đã điền cho <N> screens

Figma frames đã tạo (high-fi):
  - <list Screen Code + node-id>

Components reuse từ library:
  - <list components đã import>
Components đề xuất mới (đã được user duyệt):
  - <list new components nếu có>
Components SKIP wireframe (user đồng ý):
  - <list nếu có>

Quality check:
  ✅ Tất cả screens dùng component instance (không rectangle thủ công)
  ✅ Sample data realistic đúng domain/ngôn ngữ dự án
  ✅ Icons + status badges + pagination đầy đủ
  ✅ Text styles + color variables bind đúng

Bước tiếp theo (chờ Tech Lead Design xong DESIGN.md — chạy song song):
→ "Hãy là Tech Lead Tasks, phân rã tasks từ DESIGN.md tại:
   <DOCS_ROOT>/features/<feature>/"

Sau khi có task files — implement theo repo:
→ FE: "Hãy là Frontend Developer, implement task: <task-x-y.md>"
   FE Agent tự gọi MCP đọc Figma từ URL trong task file.
→ Mobile (nếu có screens mobile): "Hãy là Mobile Developer, implement task: <task-x-y.md>"
```
