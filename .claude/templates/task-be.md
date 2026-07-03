# Template task-x-y.md — Backend / Phase 1 & 2

> Template dùng bởi `techlead-tasks-agent.md` (Bước 6). Áp dụng cho task **Phase 1 (DB migration)** và **Phase 2 (API + Service)** của repo `backend`. Task Phase 3 FE/Mobile → dùng `task-fe-mobile.md`.

```markdown
# [ROLE] [Category] — <Mô tả ngắn gọn>

## Backlog Info
- **Issue Type:** Task
- **Category:** <Category theo Ecosystem AGENTS.md>
- **Parent Issue:** <User Story title hoặc Epic ID>
- **Version:** <release/version của dự án>
- **Milestone:** <Released xxx>
- **Estimate Hour:** Xh
- **Actual Hour:** — _(điền khi Resolved)_
- **Status:** Open

## Metadata
| Thuộc tính | Giá trị |
|---|---|
| Phase | X — <tên phase> |
| Repo | `<repo-name>` |
| Depends on | task-Y-Z / none |
| Song song với | task-A-B / none |
| Estimate | ~Xh |

## Mục tiêu
[1-2 câu: task này làm gì và tại sao cần]

## Context (đọc trước khi code)
- SPEC.md: `<DOCS_ROOT>/features/<feature>/SPEC.md>`
- DESIGN.md: `<DOCS_ROOT>/features/<feature>/<repo>/DESIGN.md>`
- Screen Code: `<XX_FEAT_001>` _(FE/Mobile task — lấy từ SPEC.md ## Screens)_
- Figma URL: `<path_figma>` _(FE/Mobile task — lấy từ cột Figma Link trong SPEC.md ## Screens; FE/Mobile agent gọi MCP đọc trực tiếp)_
- File liên quan:
  - `<path/to/file>` — xem pattern inject dependency / service structure
  - `<path/to/entity>` — xem column conventions

## Yêu cầu implement

### Tạo / Sửa: `<đường dẫn chính xác>`

```typescript
// Pseudocode / code snippet cụ thể
```

## Unit Tests (BẮT BUỘC)

### Test file: `<path>.spec.ts`

```typescript
describe('<ClassName>', () => {
  let service: <ClassName>;
  let mockDep: jest.Mocked<<Dependency>>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [<ClassName>, { provide: <Dependency>, useValue: createMock<<Dependency>>() }],
    }).compile();
    service = module.get<<ClassName>>(<ClassName>);
    mockDep = module.get(<Dependency>);
  });

  it('should <mô tả behavior>', async () => {
    // Arrange → Act → Assert — BẮT BUỘC có ít nhất 1 expect()
  });
});
```

**Coverage target:**
| File | Target |
|---|---|
| `*.service.ts` | ≥ 80% |
| `*.controller.ts` | ≥ 70% |

**Verify:** `npm run test -- <file>`

## API Definition _(chỉ điền cho task Phase 2 — API endpoint; bỏ section này nếu task Phase 1)_

> Điền sau khi implement xong. FE/Mobile sẽ copy bảng này vào task-3-x trước khi bắt đầu code.

| Method | Endpoint | Request | Response |
|---|---|---|---|
| GET | `/api/<resource>` | `?page=1&limit=10` | `{ items: [], total }` |
| POST | `/api/<resource>` | `{ field: type }` | `{ id, field }` |

**Base URL:** `VITE_API_URL` (env var — không hard-code)
**Auth:** Bearer JWT (trừ khi endpoint public)

## Non-Regression Table
| Tính năng | File liên quan | Cách verify |
|---|---|---|
| <feature đang dùng entity/service này> | `<path>` | <bước test> |

## Không được làm
- Không sửa `<file khác>` — ngoài scope, sẽ làm ở task khác
- Không thay đổi response format API đang có — consumer đang dùng
- Không refactor code lân cận dù thấy cần cải thiện

## Definition of Done
- [ ] Build pass (`npm run build`)
- [ ] Lint pass (`npm run lint`)
- [ ] Unit Tests pass — coverage đạt target
- [ ] Non-Regression verify đủ
- [ ] **API Definition điền đủ** _(Phase 2 only)_ — FE/Mobile có thể bắt đầu task-3-x
- [ ] Actual Hour cập nhật
- [ ] Status → Request Review
```
