# AI Agents Workflow — Sitter Navi

> **Mục đích:** tài liệu tra cứu nhanh cho user/PM audit từng agent làm gì trong pipeline BMAD của **Sitter Navi**. Nội dung tổng hợp từ 12 file trong `.claude/agents/`. Khi sửa workflow của agent nào → **sửa file agent tương ứng, không sửa file này** (sync lại sau).
>
> **Không phải canonical** — canonical là `.claude/agents/<agent-name>.md`. File này chỉ là bản tổng hợp.
>
> **Domain:** Sitter Navi — dịch vụ kết nối phụ huynh với babysitter/caregiver (thị trường Nhật Bản). Actors: `admin` (web) · `parent` (mobile app parents) · `caregiver` (mobile app babysitter — enum backend là `caregiver` nhưng module dùng naming `sitter-*`).

---

## 0. Ecosystem tham chiếu nhanh

| Repo | Path | Vai trò | Stack chính |
|---|---|---|---|
| `sitternavi-web-BE` | `repositories/sitternavi-web-BE` | backend | NestJS 11 · TypeORM 0.3.26 · PostgreSQL · Redis/BullMQ · WebSockets |
| `sitternavi-web` | `repositories/sitternavi-web` | frontend (**ADMIN dashboard**) | **Next.js 16 App Router** · React 19 · AntD 6 · Tailwind 4 · TanStack Query 5 · react-hook-form + **zod** |
| `sitternavi-app-parents` | `repositories/sitternavi-app-parents` | mobile (phụ huynh) | Flutter · hooks_riverpod 3 · Retrofit/Dio · auto_route · freezed |
| `sitternavi-app-babysitter` | `repositories/sitternavi-app-babysitter` | mobile (caregiver) | Flutter · hooks_riverpod 3 · Retrofit/Dio · auto_route · freezed |

**Docs root (`<DOCS_ROOT>`):** `sitter-navi-docs/docs`

### Gotchas nhớ kỹ

1. **2 mobile app Flutter tên gần giống** — `sitternavi-app-parents` vs `sitternavi-app-babysitter`, cả hai `package name: myapp`. **Xác nhận đúng repo bằng đường dẫn**, không dựa vào package name. Endpoint phân biệt: parents dùng `api/v1/client/parent-profiles|children`, babysitter dùng `api/v1/sitter/*`.
2. **`.claude/CLAUDE.md` trong `sitternavi-app-babysitter` bị dán nhầm** (mô tả là "parents app"). Code là nguồn chân lý.
3. **`sitternavi-web` là ADMIN dashboard**, KHÔNG phải web cho phụ huynh/sitter. Auth qua `POST /v1/admin/auth/login`. Phụ huynh & sitter được quản lý như dữ liệu, không phải user của web.
4. **Role enum backend = `RoleType`** (`admin` / `parent` / **`caregiver`**). Module/naming dùng `sitter-*` (sitter-profile, sitter-service...) — cùng 1 actor `caregiver`.
5. **Chưa có payment gateway ngoài** — module `payment`/`payout` chỉ là ledger CRUD nội bộ. Đừng giả định đã tích hợp Stripe/elepay/PayPay.

---

## 1. Bảng tổng quan — Phase-Gate BMAD

| # | Phase | Agent | Trigger | Song song với |
|---|---|---|---|---|
| 0 | Setup | `init-agent` | `/init-kit` (1 lần) | — |
| 1 | Discovery | `ba-agent` | `/create-spec <feature>` | — |
| 2a | Design | `techlead-design-agent` | `/create-design <SPEC.md>` | 2b, 2c |
| 2b | Design | `qc-manual-agent` (TC lần 1) | `/test/generate_manual_testcases_rbt` | 2a, 2c |
| 2c | Design | `designer-agent` | `/create-ui-design <SPEC.md>` | 2a, 2b |
| 3 | Planning | `techlead-tasks-agent` | `/create-tasks <feature/>` | — |
| 4 | Planning | `pm-agent` | `/create-plan <feature/>` (+ optional `/create-backlog`) | — |
| 5a | Build | `backend-agent` | Implement task Phase 1 → 2 | — |
| 5b | Build | (manual) copy API Contract → task-3-x | — | — |
| 5c | Build | `frontend-agent` | Implement task Phase 3 (`sitternavi-web`) | 5d |
| 5d | Build | `mobile-agent` | Implement task Phase 3 (`sitternavi-app-parents` / `-babysitter`) | 5c |
| 5e | Integration | BE + FE + Mobile | task-4-x integration test | — |
| 6 | Verify | `qa-agent` | `"Hãy là QA, verify task: <path>"` | — |
| 7a | Test | `qc-manual-agent` (execution lần 2) | `/test/generate_test_execution_checklist` | 7c |
| 7b | Test | `qc-manual-agent` (regression optional) | `/test/generate_regression_suite` | — |
| 7c | Test | `qc-automation-agent` | `"Hãy là QC Automation, test feature: ..."` | 7a |

**Handover chain:** natural language (copy-paste vào turn kế) hoặc slash command — cả hai cùng load file agent.

---

## 2. Nguyên tắc chung

| Rule | Áp dụng cho |
|---|---|
| Bước 1 = đọc context + skill trước khi hành động | Mọi agent |
| Chỉ tạo/sửa `.md` (trừ Dev) | BA, PM, Tech Lead, QC-manual, QA, Designer, QC-Auto |
| `tilth_deps` blast radius BẮT BUỘC trước khi đổi public interface | Tech Lead Design/Tasks, Backend, Frontend, Mobile |
| Không tự đoán khi thiếu context — phải hỏi user | Mọi agent |
| Handover message = natural language + slash command song song | Mọi agent |
| Không commit / push khi user không yêu cầu rõ ràng | Mọi agent (đặc biệt Dev) |
| **Xác nhận đúng repo mobile bằng đường dẫn** (2 app tên gần giống) | Mobile agent |

---

## 3. Chi tiết per agent

### 3.0 `init-agent` — Kit Setup Assistant

| | |
|---|---|
| **Đọc** | `AGENTS.md` (root) — check placeholder Ecosystem |
| **Thực thi** | Hỏi 8 câu: tên dự án, docs root, repos + vai trò + stack, Epic code, actors, payment/integration, gotchas, cross-repo features |
| **Output** | `AGENTS.md` + `.claude/context/specification.md` + `technical.md` + `.claude/rules/project-structure.md` + `stack-constraints.md` (nếu stack khác default) + `POLICIES.md` (nếu payment thay đổi) |
| **Handover** | → BA (`/create-spec <feature đầu tiên>`) |
| **Không** | Sửa source code · Ghi đè khi đã init (phải hỏi user chọn a/b/c) |

### 3.1 `ba-agent` — Business Analyst

| | |
|---|---|
| **Đọc Bước 1** | `.claude/context/specification.md` · `doc-structure.md` · `.claude/context/business-flows/business-flow-index.md` (bản đồ domain đang có) · `.claude/skills/business-analyst/SKILL.md` · `tilth_files("**/SPEC.md")` |
| **Đọc thêm sau Bước 1** | `sitter-navi-docs/docs/product/NN-<slug>.md` — doc chi tiết theo domain (flow + actors + màn hình + gaps) → hiểu hệ thống hiện tại trước khi hỏi user (KHÔNG hỏi lại thứ đã có trong doc) |
| **Thực thi** | Bước 2: hỏi 10 câu discovery (actor, problem, precondition, happy path, edge case, AC, related feature, mobile, real-time, integration) → Bước 3: xác định path `sitter-navi-docs/docs/features/<feature>/SPEC.md` → Bước 4: viết SPEC theo template |
| **Output** | `SPEC.md` — sections: Mô tả nghiệp vụ · Actors & Preconditions · Happy Path · Alternative Flows & Edge Cases · AC · Out of Scope · **Screens table** (Screen Code + Screen + Actor + App + Screen Type + Mô tả + Figma Link) |
| **Handover** | → Tech Lead Design (2a) + Designer (2c) + QC Manual (2b) chạy song song |
| **Không** | Đưa giải pháp kỹ thuật · Sửa source code · Tự đoán khi thiếu info |

### 3.2 `techlead-design-agent` — Tech Lead Design

| | |
|---|---|
| **Đọc Bước 1** | SPEC.md · `.claude/context/technical.md` · `doc-structure.md` · `.claude/skills/solution-architect/SKILL.md` |
| **Đọc Figma (optional)** | `get_design_context` + `get_screenshot` (nếu SPEC.md ## Screens có Figma URL) |
| **Đọc code Bước 3** | `tilth_search` · `tilth_read` · **BẮT BUỘC `tilth_deps`** blast radius |
| **Thực thi** | Bước 2: map nghiệp vụ → repo (Sitter Navi: BE luôn `sitternavi-web-BE`; FE admin `sitternavi-web`; mobile 1 trong 2 app tùy actor) · Bước 3: blast radius · Bước 4: viết DESIGN per repo |
| **Output** | `<DOCS_ROOT>/features/<feature>/<repo>/DESIGN.md` — 7 sections: Tổng quan · Database Changes · **API Definition** (nguồn Contract Lock) · Service Layer · Interface cross-repo (WebSocket / Push) · Luồng xử lý · Non-Regression Risks |
| **Handover** | → Tech Lead Tasks (sau khi Designer xong Figma URL) |
| **Không** | Viết source code · Tự đoán khi SPEC mơ hồ · Bypass `tilth_deps` |

### 3.3 `qc-manual-agent` — QC Manual Tester

> **Chạy 3 lần trong pipeline:** lần 1 = sinh TC sau SPEC (2b) · lần 2 = execution checklist trước release (7a) · lần 3 song song với 7c (QC Automation)

| | |
|---|---|
| **Đọc Bước 1** | SPEC.md · `.claude/skills/rbt_manual_testing/SKILL.md` |
| **Đọc Figma (optional)** | `get_screenshot` + `get_design_context` (từ ## Screens Figma Link) |
| **Thực thi** | Chọn mode: **QUICK** · **FULL RBT 6 bước** · **Cross-module Pairwise** · **Update delta** · **Regression** · **Execution** · **Exploratory** · **Bug report** · **Test data** · **Onboarding** |
| **Output** | `<feature>/test-cases/tc_<module>.md` · `regression_<release>.md` · `checklist_<release>.md` · `bug-reports/BUG_*.md` · `cross_module_plan.md` — mỗi TC có visual states (Normal/Focus/Filled/Error/Disabled/Loading) + Traceability Matrix AC ↔ TC (100% cover) |
| **Không** | Sinh TC khi chưa đọc SPEC · Test data placeholder · Gộp validation nhiều field · Auto-submit bug · Lẫn vai trò với qa-agent (KHÔNG chạy test suite) |

### 3.4 `designer-agent` — UI/UX Designer

| | |
|---|---|
| **Đọc Bước 1** | SPEC.md · `.claude/context/designer-context.md` (**BẮT BUỘC**) · `.claude/rules/design_rule.md` · `.claude/skills/figma-design/SKILL.md` · `skill://figma/figma-use/SKILL.md` (MCP) · `skill://figma/figma-generate-design/SKILL.md` (MCP) |
| **Đọc Figma Bước 3** | `search_design_system` cho ~10 components (Sidebar, Header, Table, Button, Input, DatePicker, StatusBadge, Pagination, Modal, Icon) · `get_design_context` + `get_metadata` + `get_variable_defs` của reference screen |
| **Thực thi** | Bước 2: phân tích ## Screens · Bước 3: Component Discovery — **Gate check** (thiếu component → dừng, hỏi user A/B/C) · Bước 4: import component + tạo frame + bind variables + sample data thật (tiếng Nhật cho ADMIN dashboard, không placeholder) · Bước 5: update SPEC.md |
| **Output** | Figma frames HIGH-FIDELITY (cloud) + **cột Figma Link trong SPEC.md ## Screens** + append `[Design]` notes vào SPEC ## Open Questions (nếu phát hiện gap) |
| **Handover** | → Tech Lead Tasks (chờ DESIGN.md xong) → FE/Mobile đọc Figma URL từ task file |
| **Không** | Tạo file `.md` riêng (UI-SPEC / figma-context) · Vẽ wireframe · Tự generate component khi thiếu · Commit / push code |

### 3.5 `techlead-tasks-agent` — Tech Lead Tasks

| | |
|---|---|
| **Đọc Bước 1** | `tilth_files("*/DESIGN.md")` · `doc-structure.md` · `.claude/skills/task-decomposition/SKILL.md` · SPEC.md ## Screens (Screen Code + Figma URL cho FE/Mobile task) |
| **Đọc Figma (optional)** | `get_metadata` + `get_screenshot` (estimate complexity) |
| **Đọc code Bước 2** | `tilth_search` + `tilth_read` + `tilth_deps` (blast radius vào Non-Regression table) |
| **Thực thi** | Bước 3: map repo → Backlog Category + ROLE tag `[BE]`/`[FE]` · Bước 4: phase numbering (1: DB migration, 2: Service+API, 3: FE+Mobile song song, 4: Integration) · Bước 6: template BE · Bước 6b: template FE/Mobile (3 sub-steps: service → hooks → wire UI) |
| **Output** | `<DOCS_ROOT>/features/<feature>/<repo>/tasks/task-{phase}-{index}.md` — mỗi task ~4-8h · có: Backlog Info · Context · Yêu cầu implement · **Unit Tests BẮT BUỘC** · API Definition (Phase 2) · Non-Regression Table · DoD |
| **Handover** | → PM (`/create-plan`) |
| **Không** | Task > 8h · Task thiếu Unit Test · Đoán khi DESIGN mơ hồ · Sửa source code |

### 3.6 `pm-agent` — Project Manager

| | |
|---|---|
| **Đọc Bước 1** | SPEC.md · `.claude/context/specification.md` · `.claude/skills/project-planning/SKILL.md` · `tilth_files` list DESIGN.md + `tasks/task-*.md` · check ## Screens Figma Link |
| **Đọc Figma (optional)** | `get_screenshot` + `get_metadata` (estimate complexity) |
| **Đọc Backlog Bước 4.2** | `get_project_list` · `get_issue` (parent) · `get_users` · `get_categories` · `get_version_milestone_list` · `get_issue_types` · `get_priorities` |
| **Thực thi** | Bước 2: hỏi 5 câu (deadline + phase-gate, dev available, dependency, deploy STG/PROD, QA riêng hay dev tự test) · Bước 3: viết PLAN · Bước 4 (optional): sync Backlog — tạo issue thử → user confirm → batch tạo N-1 issues |
| **Output** | `<DOCS_ROOT>/features/<feature>/PLAN.md` — sections: Summary · Phase-Gate · Timeline (Gantt ASCII) · **Contract Lock** (REST + WebSocket + Push) · Dependencies & Risks · Assignees · DoD · (optional) Backlog issue keys |
| **Handover** | → Dev (BE Phase 1 trước, sau đó FE/Mobile Phase 3 song song) · optional `/create-backlog` |
| **Không** | Phân tích yêu cầu (BA việc) · Design kỹ thuật · Ghi số giả (dùng `TBD`) · Sửa source code |

### 3.7 `backend-agent` — Backend Developer (`sitternavi-web-BE`)

| | |
|---|---|
| **Đọc Bước 1** | task-x-y.md · SPEC.md · DESIGN.md · **overview docs** (`sitter-navi-docs/docs/backend/sitternavi-web-BE/overview/`): `structure.md` + `patterns.md` + `api-catalog.md` + `erd.md` (55 entity + quan hệ — reuse, không tạo trùng) · `.claude/skills/nestjs-best-practices/SKILL.md` + `postgresql/SKILL.md` (+ `redis-development/SKILL.md` nếu Redis optimization) |
| **Đọc Figma (optional)** | `get_design_context` + `get_screenshot` (nếu API response cần khớp UI) |
| **Đọc code Bước 2-3** | `tilth_search` pattern hiện có · `tilth_deps` blast radius |
| **Thực thi** | Implement (module/service/entity/DTO/migration/guard/interceptor) · Chạy `npm run lint` + `test` + `test:cov` · Self-review (snake_case column, migration `up`+`down`, không N+1, Redis TTL, DTO class-validator, không hard-code secret) |
| **Output** | Code + Unit Test coverage ≥ 80% service / ≥ 70% controller · **API Contract table** (handoff FE/Mobile) · Memory Update Gate: `api-catalog.md` / `erd.md` / `patterns.md` |
| **Handover** | → QA verify → sau QA PASS: copy API Contract vào task-3-x FE/Mobile |
| **Không** | Sửa migration/linter/test config không được yêu cầu · Commit khi không được yêu cầu · Hard-code secret · N+1 query · Đưa module `payment`/`payout` thành gateway thật (chỉ ledger CRUD) |

### 3.8 `frontend-agent` — Frontend Developer (`sitternavi-web` — ADMIN)

> ⚠️ Stack **KHÔNG phải Vite** — dùng **Next.js 16 App Router**. Đừng viết `import.meta.env.VITE_*`, đừng dùng `react-router-dom`, đừng dùng `yup`.

| | |
|---|---|
| **Đọc Bước 1** | task-x-y.md · **BE task-2-X.md để lấy API Contract** (không tự đoán endpoint) |
| **Đọc Bước 2-3** | SPEC.md · DESIGN.md · **overview docs** (`sitter-navi-docs/docs/frontend/sitternavi-web/overview/`): `structure.md` + `patterns.md` · `.claude/skills/react-expert/SKILL.md` + `frontend-review/SKILL.md` |
| **Đọc Figma (ưu tiên cao)** | Song song 4 tools: `get_metadata` + `get_design_context` + `get_variable_defs` + `get_screenshot` (nếu có URL trong task Context / SPEC ## Screens / user paste) |
| **Đọc code Bước 4** | `tilth_search` pattern hiện có |
| **Thực thi** | 3 sub-steps: **Step 1** `src/services/client/<domain>.service.ts` (gọi qua singleton `API`) → **Step 2** `src/hooks/queries/use-<feature>.ts` (TanStack Query v5 object syntax, `queryKey` bắt đầu bằng hằng `QUERY_KEYS`) → **Step 3** `src/app/(dashboard)/<route>/page.tsx` + `_components/` wire hooks · Form: schema zod ở `src/libs/schema/<domain>.schema.ts` · Chạy `lint` + `type-check` + `build` + `test --coverage` · Integration check FE-localhost (**port 3001**) + BE-localhost |
| **Output** | Code + Unit Tests ≥ 70% · Memory Update Gate: `patterns.md` per repo |
| **Handover** | → QA verify |
| **Không** | `import.meta.env`/`VITE_*` (dùng `NEXT_PUBLIC_*` qua `src/configs/config.ts`) · `useNavigate`/`useHistory`/`useParams` của `react-router-dom` (dùng `next/navigation`) · `yup` (dùng `zod`) · AntD `Form.Item rules` (dùng RHF + `zodResolver`) · Import `../*` (dùng alias `@/*`) · Tự tạo axios instance mới · Mock data production · Hard-code URL |

### 3.9 `mobile-agent` — Mobile Developer (Flutter)

> ⚠️ **2 repo mobile tên gần giống** — luôn xác nhận đúng repo bằng đường dẫn task file, không dựa vào `package name: myapp`.

| | |
|---|---|
| **Đọc Bước 1-2** | task-x-y.md · SPEC.md · DESIGN.md · **overview docs** (`sitter-navi-docs/docs/mobile/<repo>/overview/`): `structure.md` + `patterns.md` · `.claude/skills/flutter-review/SKILL.md` |
| **Đọc Figma (ưu tiên cao)** | Song song 4 tools: `get_metadata` + `get_design_context` + `get_variable_defs` + `get_screenshot` (nếu có URL) |
| **Đọc code Bước 4** | `tilth_search` pattern hiện có |
| **Thực thi** | Implement với `hooks_riverpod` 3 + Retrofit `@RestApi()` + `freezed` + `auto_route` + Socket.IO cleanup · Chạy `flutter analyze` + `flutter test` · Self-review (Riverpod, không Dio trực tiếp, socket `off()` trong dispose, `flutter_screenutil` `.w`/`.h`/`.sp`) |
| **Output** | Code + Unit Tests ≥ 75% provider · Memory Update Gate: `structure.md` / `patterns.md` |
| **Handover** | → QA verify |
| **Endpoint theo repo** | `sitternavi-app-parents` → `api/v1/client/parent-profiles\|children\|...` · `sitternavi-app-babysitter` → `api/v1/sitter/*` |
| **Không** | Provider/BLoC/GetX · Hard-code pixel/hex · Sửa `.g.dart`/`.freezed.dart` thủ công · Đảo version convention (DEV `0.0.x` · STG `0.1.x` · PROD `1.0.x`) · `Navigator.push` trực tiếp · Nhầm repo (2 app tên gần giống) · Tin `package name` để phân biệt repo |

### 3.10 `qa-agent` — QA Engineer

| | |
|---|---|
| **Đọc Bước 1** | task-x-y.md (coverage target + Non-Regression table) · SPEC.md (AC + ## Screens) · `.claude/skills/requirements_analyzer/SKILL.md` |
| **Đọc Figma (optional)** | `get_screenshot` + `get_design_context` (nếu verify UI, so sánh với code thực) |
| **Thực thi** | Bước 2: chạy `lint` + `build` + `test` + `test:cov` per repo (NestJS: `npm run test:cov` · Next.js: `pnpm lint` + `type-check` + `build` · Flutter: `flutter analyze` + `test`) · Bước 3: validate từng AC (happy + edge + boundary) · Bước 4: check Non-Regression table |
| **Output** | **QA Report** per task — sections: Test Results (unit/coverage/lint/build) · AC table (pass/fail per AC + lý do) · Non-Regression table · Kết luận PASS/FAIL với issue list (file:line + đề xuất fix) |
| **Handover** | PASS → status Testing Request → QC execution checklist (7a) · FAIL → dev fix → gọi lại QA |
| **Không** | Sửa source code · Sinh manual TC (qc-manual-agent việc) · So với assumption thay vì SPEC · Thay đổi test cases đã approve |

### 3.11 `qc-automation-agent` — QC Automation Tester

| | |
|---|---|
| **Đọc Bước 1** | Kiểm tra `<e2e-repo>/.env.test` tồn tại · Đọc `<ROLE>_URL` env · `curl` probe website · `npx playwright --version` |
| **Đọc Bước 2** | SPEC.md (Actors, Preconditions, Out of Scope, Screen Code) · optional `<testcases>` TC file (**ưu tiên cao hơn SPEC nếu có** — chế độ TC-driven, không tự thêm scenario) |
| **Đọc Figma Bước 3** | `get_design_context` + `get_screenshot` (extract labels cho selectors: button, placeholder, heading, toast) |
| **Thực thi** | Bước 4: danh sách scenarios (**max 10 nếu SPEC-driven**) · Bước 5: sinh `.spec.ts` per TC (1 TC = 1 file, tự lập, dùng `storageState` + `getByRole`/`getByPlaceholder`/`getByText`/`getByTestId`) · Bước 6: `npx playwright test --headed --project=<target-app> --no-deps` · Bước 7: parse output |
| **Output** | `<e2e-repo>/e2e/<target-app>/<feature>/*.spec.ts` + `reports/<feature>/execution-report.md` (bảng PASS/FAIL/SKIP + screenshots FAIL + Lỗi cần xử lý + Bước tiếp theo) |
| **Không** | Sửa source code app · CSS class selector (`.btn-primary`) · `waitForTimeout` cứng · Hard-code credentials trong `.spec.ts` (dùng `process.env`) · > 20 TC lần đầu (SPEC-driven) · Test phụ thuộc nhau · TC-driven mà tự thêm scenario |
| **Lưu ý Sitter Navi** | Chưa có repo E2E riêng — khi khởi tạo, khai báo path vào `AGENTS.md` section E2E Testing. Auth admin qua `POST /v1/admin/auth/login` |

---

## 4. Điểm audit thường gặp

| # | Điểm cần check | Cách verify |
|---|---|---|
| 1 | BA có đọc `business-flow-index.md` + `product/NN-<slug>.md` trước khi hỏi không? | Bước 1 của `ba-agent.md` phải có 2 file đó · SPEC không hỏi lại thứ đã có trong doc |
| 2 | Tech Lead có `tilth_deps` blast radius không? | DESIGN.md ## Non-Regression Risks phải có nội dung thật, không placeholder |
| 3 | Task có Unit Test không? | Mỗi task-x-y.md phải có section "Unit Tests (BẮT BUỘC)" với coverage target |
| 4 | FE có tự đoán endpoint không? | task-3-x.md ## API Contract phải copy từ BE task-2-X, không tự viết |
| 5 | FE có nhầm stack Vite/Router không? | Check import — phải là `next/navigation` không phải `react-router-dom`; phải là `NEXT_PUBLIC_*` không phải `VITE_*` |
| 6 | Mobile có nhầm 2 repo không? | Verify đường dẫn task file khớp với repo — endpoint parents (`client/parent-*`) vs babysitter (`sitter/*`) |
| 7 | Designer có vẽ wireframe không? | Mỗi frame Figma phải dùng component instance, không rectangle + text; sample data tiếng Nhật cho ADMIN |
| 8 | QA có so với SPEC.md AC hay assumption? | QA Report phải trích AC ID cụ thể từ SPEC, không mô tả chung |
| 9 | QC Automation có headed mode không? | `execution-report.md` phải có screenshot; command có `--headed` |
| 10 | Memory Update Gate có bị skip không? | Sau mỗi dev task, output phải liệt kê `api-catalog.md` / `erd.md` / `patterns.md` — updated hay skipped |
| 11 | Handover message có natural language không? | Output cuối mỗi agent phải có `"Hãy là <role>, ..."` để user copy-paste |
| 12 | Agent có commit tự động không? | KHÔNG được — chỉ commit khi user yêu cầu rõ ràng |
| 13 | BE có giả định payment gateway thật không? | Module `payment`/`payout` là ledger CRUD; nếu task cần charge thật → flag là gap chưa làm |

---

## 5. Common failure modes

| Failure | Nguyên nhân | Cách phòng |
|---|---|---|
| BA sinh SPEC trùng lặp / lệch domain | Không đọc `business-flow-index.md` + `product/NN-*.md` | Bước 1 `ba-agent` bắt buộc đọc index + doc domain tương ứng |
| Tech Lead design conflict với 55 entity đã có | Không đọc `erd.md` overview | `backend-agent`/`techlead-design-agent` phải đọc `overview/erd.md` trước khi thêm entity |
| FE code endpoint không tồn tại | Không đọc BE task-2-X ## API Contract | Frontend Agent Bước 2 BẮT BUỘC đọc BE task |
| FE dùng syntax React+Vite (import.meta.env, react-router-dom, yup) | Copy code từ dự án cũ | Self-review checklist có kiểm tra Next.js App Router + zod + `NEXT_PUBLIC_*` |
| Mobile lẫn code giữa 2 app tên gần giống | Tin `package name: myapp` để phân biệt | Xác nhận bằng đường dẫn task file; endpoint parents vs sitter |
| Tech Lead map actor `caregiver` thành module `caregiver-*` | Chỉ nhìn enum không đọc code | Verify: enum `RoleType.caregiver` nhưng module dùng `sitter-*` naming |
| BE giả định đã tích hợp Stripe/elepay khi làm payment | Không đọc gotcha #5 | Flag: payment gateway ngoài chưa có, module hiện tại chỉ là ledger CRUD |
| QC Automation sinh selector CSS class | Không đọc Figma labels | qc-automation-agent Bước 3 đọc Figma trước khi viết spec |
| QA PASS nhưng vẫn miss AC | So với assumption thay vì SPEC | qa-agent Bước 3 đối chiếu AC ID từ SPEC.md |
| Task quá lớn, dev không xong trong session | techlead-tasks-agent ước lượng sai | Enforce 4-8h/task, chia nhỏ nếu > 8h |

---

## 6. Xem thêm

- **Canonical workflow từng agent:** `.claude/agents/<agent-name>.md`
- **BMAD pipeline chi tiết:** `.claude/workflows/new-feature.md`
- **Bug fix workflow:** `.claude/workflows/bug-fix.md`
- **AI behavior policy:** `POLICIES.md` (always-loaded qua `CLAUDE.md`)
- **Ecosystem (repos + actors + docs root):** `AGENTS.md` section `<ecosystem>`
- **Business flow index:** `.claude/context/business-flows/business-flow-index.md` — bản đồ domain nghiệp vụ hiện có
- **Product domain docs:** `sitter-navi-docs/docs/product/NN-<slug>.md` — chi tiết flow + actors + gaps per domain
- **Overview docs per repo:** `sitter-navi-docs/docs/{backend,frontend,mobile}/<repo>/overview/` — bản đồ code hiện có (structure/patterns/api-catalog/erd)
- **Context files (agent load on-demand):** `.claude/context/` (đọc theo cột "Ai đọc" trong `AGENTS.md`)
- **Skills:** `.claude/skills/README.md`
