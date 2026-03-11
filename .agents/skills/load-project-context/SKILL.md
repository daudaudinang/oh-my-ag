---
name: load-project-context
description: |
  Nạp project context theo tầng (global → epic → task) trước khi thực thi
  các skill khác (create-plan, review-plan, implement-plan, debug...).
  Đảm bảo agent hiểu kiến trúc, quy tắc, decisions đã chốt, và codebase
  layout TRƯỚC khi bắt tay vào công việc cụ thể.
  Kích hoạt tự động bởi các skill khác (hook step 0.5), hoặc khi user
  yêu cầu "nạp context", "load context", "hiểu project trước".
---

# Goal

Nạp đúng và đủ context cho agent trước khi thực thi task, theo thứ tự ưu tiên:
**Global rules → Architecture docs → Epic/Phase plan → Task-specific context**.

Agent sau khi chạy skill này phải trả lời được:
1. Project này là gì, dùng tech stack gì?
2. Kiến trúc tổng thể ra sao? (layers, services, ports)
3. Có những quyết định/contract nào đã chốt?
4. Task hiện tại nằm ở đâu trong roadmap?
5. Có ràng buộc/rule nào PHẢI tuân thủ?

---

# Instructions

## Bước 1: Xác định task context

Xác định agent đang làm gì để nạp context phù hợp:

```
Từ lệnh/message user, xác định:
  - TASK_TYPE: create-plan | review-plan | implement-plan | debug | review-implement | general
  - SCOPE_HINT: Phase/Epic nào? (VD: "Phase 1", "T3 Skills", "Chat UI")
  - DOMAIN_HINT: agent | backend | frontend | infra | cross-layer
```

## Bước 2: Nạp Global Context (LUÔN LUÔN)

> Đọc theo thứ tự, dừng lại nếu file không tồn tại.

### 2.1: Project overview

Đọc các file sau để hiểu project:

```
PHẢI đọc (nếu tồn tại):
  - README.md              → Hiểu project là gì
  - AGENTS.md              → Agent-specific context (nếu có)
  - CLAUDE.md              → Claude-specific instructions (nếu có)
  - package.json           → Dependencies, scripts
  - docker-compose.yml     → Services, ports, env vars
```

**Output cần ghi nhận:**
- Project name, description
- Tech stack: languages, frameworks, runtimes
- Services: tên, ports, startup commands
- Package manager: bun / npm / uv / pip

### 2.2: Rules & Constraints

```
PHẢI đọc (nếu tồn tại):
  - .agents/rules/*.md     → Tất cả rule files
  - .cursor/rules/*.mdc    → Cursor rules (nếu có)
  - .agent/rules/*.md      → Legacy rules (nếu có)
```

**Output cần ghi nhận:**
- Precedence rules (cái nào ưu tiên hơn cái nào)
- Naming conventions
- Forbidden patterns (KHÔNG ĐƯỢC làm gì)
- Domain-specific contracts (VD: Multi-Merchant decisions)

### 2.3: Codebase layout

```
PHẢI chạy:
  - list_dir trên project root (depth 1)
  - list_dir trên servers/ (nếu có)
  - list_dir trên agent/ (nếu có)
```

**Output cần ghi nhận:**
- Folder structure overview
- Vị trí backend, frontend, agent code
- Vị trí tests, configs, migrations

## Bước 3: Nạp Architecture Context (nếu có)

> Chỉ nạp nếu tồn tại. Đây là tài liệu kiến trúc chi tiết.

### 3.1: Slide Agent context (presenton project)

```
NẾU project là Presenton (Slide Agent), đọc:
  - .agents/plans/brainstorm-slide-agent-v3.md       → Architecture & design decisions
  - .agents/plans/production-readiness-slide-agent.md → Quality standards & SLOs
```

**Output cần ghi nhận (Slide Agent):**

| Khía cạnh           | Source                       | Ghi nhận                                                                                                                                   |
| ------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 7 tools Phase 1     | brainstorm v3.2 mục 5        | list_presentations, get_presentation_detail, get_slide_content, search_in_presentation, edit_slide, export_presentation, get_slide_preview |
| Phase 2 additions   | brainstorm v3.2 mục 10       | ask_about_presentation, Chat UI, Session, Retry/Idempotency                                                                                |
| Nguyên tắc thiết kế | brainstorm v3.2 mục 0.1      | Backend = source of truth, Skills ≠ business rules, Rollback = whole-slide snapshot                                                        |
| Response format     | production-readiness mục 2.3 | ToolResponse schema, MutatingToolResponse                                                                                                  |
| SLOs                | production-readiness mục 1.1 | Tool accuracy ≥ 95%, P95 latency < 5s                                                                                                      |
| Agent port          | Global preconditions         | 6004 (A2A + MCP)                                                                                                                           |
| Runtime             | Global preconditions         | Python 3.11+ (uv run), Node (bun)                                                                                                          |

### 3.2: Multi-Merchant context

```
NẾU task liên quan Multi-Merchant, đọc:
  - .agents/rules/context-rule.md            → Decisions đã chốt
  - .agent/note_flow/CHOT_NGHIEP_VU_*.md     → Business contracts
  - .agent/note_flow/PLAN_MULTI_MERCHANT_*.md → Implementation plan
```

## Bước 4: Nạp Epic/Phase Context (theo SCOPE_HINT)

> Dựa trên SCOPE_HINT từ Bước 1, đọc đúng plan file.

```
NẾU SCOPE_HINT chứa "Phase 1" hoặc task T0-T6/R0-R7 Phase 1:
  → Đọc: .agents/plans/phase1-epic-plan.md

NẾU SCOPE_HINT chứa "Phase 2" hoặc task T0-T7/R0-R7 Phase 2:
  → Đọc: .agents/plans/phase2-epic-plan.md

NẾU SCOPE_HINT chứa "parallel" hoặc "sprint":
  → Đọc: .agents/plans/parallel-execution-plan.md

NẾU không rõ phase:
  → Đọc CẢ HAI phase plans (scan headers + metadata tables)
```

**Output cần ghi nhận:**
- Task dependencies (cái nào phụ thuộc cái nào)
- Execution contract (allowed files, do NOT modify)
- Acceptance criteria
- Precedence rules (plan > production-readiness > brainstorm > code)
- Global stop rules

## Bước 5: Nạp Task-Specific Context (theo TASK_TYPE)

### Nếu TASK_TYPE = `create-plan`:

```
Đọc thêm:
  - .agents/skills/create-plan/resources/plan_template.md     (nếu có)
  - .agents/skills/create-plan/resources/question_template.md (nếu có)
  - Các plan cũ tại .agents/plans/ hoặc .agent/plans/        (scan tên file)
```

### Nếu TASK_TYPE = `review-plan`:

```
Đọc thêm:
  - Plan file cần review (user sẽ chỉ định path)
  - .agents/skills/review-plan/SKILL.md (để biết tiêu chí review)
```

### Nếu TASK_TYPE = `implement-plan`:

```
Đọc thêm:
  - Plan file cần implement (user sẽ chỉ định path)
  - Execution contract + allowed files từ plan
  - Codebase files liên quan (grep/find theo task scope)
```

### Nếu TASK_TYPE = `debug`:

```
Đọc thêm:
  - Error logs / stack trace (user cung cấp)
  - Related source files (từ stack trace)
  - Recent git changes: git log -n 10 --oneline
```

### Nếu TASK_TYPE = `review-implement`:

```
Đọc thêm:
  - Plan gốc (để đối chiếu)
  - Code đã implement (git diff hoặc file list)
  - Test results
```

## Bước 6: Output — Context Summary

Sau khi nạp xong, agent PHẢI tổng hợp thành **Context Summary** ngắn gọn trong memory (KHÔNG tạo file), bao gồm:

```markdown
## 📋 Project Context Summary

### Project
- **Name**: [tên project]
- **Stack**: [tech stack]
- **Services**: [services + ports]

### Architecture Highlights
- [3-5 bullet points quan trọng nhất]

### Current Scope
- **Phase/Epic**: [phase nào]
- **Task**: [task cụ thể nếu có]
- **Dependencies**: [task nào phải xong trước]

### Constraints & Rules
- [Precedence rules]
- [Stop rules]
- [Execution contract nếu có]

### Key Decisions (đã chốt)
- [2-3 decisions quan trọng nhất cho task hiện tại]
```

> [!IMPORTANT]
> Context Summary giữ trong memory, KHÔNG tạo file.
> Sau bước này, agent tiếp tục workflow của skill gọi (create-plan, review-plan...).

---

# Examples

## Ví dụ 1: Nạp context cho `/create-plan` Phase 1 task

**Trigger:** `create-plan` skill gọi tại Bước 0.5

**Agent thực hiện:**
1. TASK_TYPE = `create-plan`, SCOPE_HINT = "Phase 1", DOMAIN_HINT = "agent"
2. Đọc: README.md → "Presenton, AI presentation tool"
3. Đọc: docker-compose.yml → FastAPI:6001, Next.js:3000
4. Đọc: .agents/rules/*.md → context-rule, agentic-rule
5. Đọc: brainstorm-slide-agent-v3.md → 7 tools, architecture
6. Đọc: production-readiness-slide-agent.md → SLOs, quality criteria
7. Đọc: phase1-epic-plan.md → T0-T6, dependencies, execution contracts
8. Output: Context Summary → tiếp tục Phase 1 của create-plan

## Ví dụ 2: Nạp context cho `/debug` lỗi agent

**Trigger:** `debug-investigator` skill gọi load-project-context

**Agent thực hiện:**
1. TASK_TYPE = `debug`, SCOPE_HINT = "agent tool error", DOMAIN_HINT = "agent"
2. Đọc: README.md, docker-compose.yml (nhanh)
3. Đọc: .agents/rules/*.md
4. KHÔNG đọc full plan files (debug không cần roadmap chi tiết)
5. Đọc: agent/ folder structure, relevant tool files
6. Output: Context Summary → tiếp tục debug workflow

## Ví dụ 3: Nạp context cho `/implement-plan` Phase 2 T3

**Trigger:** `implement-plan` skill gọi load-project-context

**Agent thực hiện:**
1. TASK_TYPE = `implement-plan`, SCOPE_HINT = "Phase 2 T3 Chat UI", DOMAIN_HINT = "frontend"
2. Đọc: README.md, docker-compose.yml
3. Đọc: .agents/rules/*.md
4. Đọc: brainstorm + production-readiness (skim)
5. Đọc: phase2-epic-plan.md → focus T3 section (Chat UI, SSE, allowed files)
6. Đọc: servers/nextjs/ folder structure (frontend codebase)
7. Output: Context Summary → tiếp tục implement workflow

---

# Constraints

## Luôn tuân thủ

- ✅ **LUÔN** đọc README.md + rules trước mọi thao tác khác
- ✅ **LUÔN** xác định TASK_TYPE và SCOPE_HINT trước khi nạp context chi tiết
- ✅ **LUÔN** tổng hợp Context Summary sau khi nạp xong
- ✅ **LUÔN** ưu tiên đọc file plan/doc TRƯỚC khi analyze code

## Không được làm

- 🚫 **KHÔNG** đọc toàn bộ mọi file — chỉ đọc theo scope
- 🚫 **KHÔNG** tạo file output — context giữ trong memory
- 🚫 **KHÔNG** mất quá 3 phút nạp context — nếu quá lâu, skip docs ít liên quan
- 🚫 **KHÔNG** tự suy diễn khi docs mâu thuẫn — tuân theo precedence rules
- 🚫 **KHÔNG** bỏ qua rules files — đây là constraints bắt buộc

## Performance

- Dùng `list_dir` (depth 1) thay vì scan toàn bộ tree
- Dùng `view_file_outline` trước `view_file` cho files lớn
- Parallel đọc files khi không phụ thuộc nhau (VD: README + rules cùng lúc)
- Skip files > 30KB khi chỉ cần overview (đọc headers/first 100 lines)

---

# Các Skills sử dụng skill này

- `.agents/skills/create-plan/SKILL.md` — Bước 0.5
- `.agents/skills/review-plan/SKILL.md` — Bước đầu
- `.agents/skills/implement-plan/SKILL.md` — Bước đầu
- `.agents/skills/debug-investigator/SKILL.md` — Thu thập context
- `.agents/skills/review-implement/SKILL.md` — Nạp context plan gốc

<!-- Generated by AI Agent -->
