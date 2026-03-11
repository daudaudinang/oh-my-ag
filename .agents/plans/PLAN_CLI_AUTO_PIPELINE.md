# PLAN: CLI Auto Pipeline — Single-Agent Automated Lifecycle

> **Version**: 1.3
> **Tier**: L
> **Type**: 🆕 Feature mới
> **Created**: 2026-03-11
> **Updated**: 2026-03-11 *(polished to 100%)*

## Mục tiêu

- Xây dựng hệ thống pipeline tự động chạy toàn bộ lifecycle của 1 Jira task: **Jira → create-plan → review-plan → implement-plan → review-implement**
- Agent tự động chạy qua từng bước, chỉ dừng khi phát hiện vấn đề nghiêm trọng cần user confirm
- Sử dụng **gemini-cli** làm runtime, **filesystem** làm state bus (thay thế Serena MCP)
- Có cơ chế **retry** khi bước fail, **escalation** khi gặp vấn đề critical
- Thiết kế dạng pipeline runner có thể mở rộng sang **multi-agent parallel** trong tương lai (Phase 2)

## Acceptance Criteria

| # | Criteria | Verify Command / Method | Pass/Fail |
|---|---------|------------------------|-----------| 
| AC-1 | Runner nhận Jira issue key / URL / manual desc, chạy tuần tự 5 steps | `bun run pipeline PRES-99 --dry-run` — kiểm tra log output đúng thứ tự 5 steps | ☐ |
| AC-2 | Mỗi step ghi output vào memory directory `.agents/memories/<RUN_ID>/` | `ls -la .agents/memories/PRES-99/` — thấy đúng các files: `task.md`, `plan-ref.md`, `review-plan.md`, `progress.md`, `review-implement.md` | ☐ |
| AC-3 | Step N+1 đọc được output của Step N qua memory files | Grep prompt output: verify `plan-ref.md`, `review-plan.md` contents injected vào step tiếp theo | ☐ |
| AC-4 | review-plan phát hiện Blocker → retry create-plan tối đa 2 lần | `bun run pipeline TEST-1 --test-retry` — simulate blocker, verify retry logic | ☐ |
| AC-5 | Step fail sau max retry → pipeline dừng, log rõ failure reason, exit code 2 | Kiểm tra exit code: `echo $?` = 2 | ☐ |
| AC-6 | Escalation: phát hiện keyword ESCALATE → pipeline dừng, ghi context file cho user | `grep "ESCALATE" .agents/memories/<RUN_ID>/escalation.md` | ☐ |
| AC-7 | Pipeline chạy end-to-end thành công trên 1 Jira ticket thực | User verify: chạy pipeline trên 1 ticket thực, kiểm tra code changes + Jira comment | ☐ |
| AC-8 | Hỗ trợ `--step` flag để chạy từ 1 step cụ thể (resume) | `bun run pipeline PRES-99 --step=3` — chạy từ implement-plan | ☐ |
| AC-9 | Hỗ trợ input dạng Jira URL: `https://xxx.atlassian.net/browse/PRES-99` | `bun run pipeline "https://xxx.atlassian.net/browse/PRES-99"` — extract key, chạy bình thường | ☐ |
| AC-10 | Hỗ trợ input manual (không Jira): `--desc "..." --name "AUTH_API"` | `bun run pipeline --desc "Build auth" --name "AUTH_API"` — skip step 1, bắt đầu từ step 2 | ☐ |
| AC-11 | Step 0.5 tạo `project-context.md` từ `load-project-context` skill | `cat .agents/memories/<RUN_ID>/project-context.md` — chứa architecture overview + rules + conventions | ☐ |

## Non-goals (chưa làm ở phase này)

- ❌ Multi-agent parallel orchestration (Phase 2 tương lai)
- ❌ QA orchestrator review tổng thể (Phase 2)
- ❌ Antigravity IDE extension/integration
- ❌ Web dashboard / terminal dashboard real-time
- ❌ Git branch management tự động (commit, PR creation)
- ❌ Jira status transition tự động (chỉ ghi comment)

## Bối cảnh hiện trạng

### Hệ thống skills hiện có (`.agents/skills/`)

| Skill | Lines | Mô tả | Input | Output |
|-------|-------|-------|-------|--------|
| `jira-workflow-bridge` | 563 | Đọc Jira → structured context | Jira issue key | Structured Context Block (markdown) |
| `load-project-context` | — | Nạp project context (global → epic → task) | Project path | Architecture + rules + decisions context |
| `create-plan` | 410 | Tạo plan triển khai | Feature requirements | `PLAN_<NAME>.md` tại `.agents/plans/` |
| `review-plan` | 352 | Review plan, chấm điểm 12 tiêu chí | Plan file path | Report: PASS / CẦN BỔ SUNG / CẦN XEM LẠI |
| `implement-plan` | 220 | Triển khai plan thành code | Plan file path | Code changes + walkthrough files |
| `review-implement` | 176 | Review code đã implement | Plan file path | Report: APPROVED / APPROVED WITH NOTES / CHANGES REQUESTED |

### oh-my-ag CLI source (tham khảo)

- **`cli/commands/agent.ts`** (1052 lines): Spawn agent via `gemini -p "..." --approval-mode=yolo`, auto-detect workspace, multi-vendor support (gemini, claude, codex, qwen)
- **`cli-config.yaml`**: Vendor config templates với prompt_flag, auto_approve_flag, output_format
- **`subagent-prompt-template.md`**: Self-contained prompt template với memory protocol + charter preflight
- **Pattern**: CLI → spawn detached process → log to file → check result files

### Thách thức thiết kế

1. **Skill adaptation**: Skills hiện tại design cho IDE (interactive, dùng `view_file`, `grep_search` tools) → cần wrap lại prompt cho CLI non-interactive
2. **Output parsing**: Cần convention để gemini output kết quả parseable (exit code + marker tags)
3. **Context size**: Mỗi step khi chạy qua CLI có context window riêng → phải truyền state qua filesystem
4. **Escalation detection**: Cần define rõ khi nào pipeline DỪNG vs khi nào auto-retry
5. **MCP tools**: gemini-cli có access MCP tools (Jira) nhưng khác tool ecosystem so với Antigravity IDE

## Dependencies / Prerequisites

- **Bun runtime** ≥ 1.1 đã cài đặt (`which bun` → có output)
- **gemini-cli** đã cài đặt (`which gemini` → có output)
- **Jira MCP server** đã cấu hình trong gemini settings (optional — chỉ cần nếu dùng Jira mode)
- **Project codebase** phải có `.agents/skills/` directory đầy đủ

## Yêu cầu nghiệp vụ (đã chốt)

- **D1**: Pipeline chạy 5 steps tuần tự: Jira (optional) → create-plan → review-plan → implement-plan → review-implement
- **D2**: State truyền giữa steps qua filesystem (`.agents/memories/<RUN_ID>/`)
- **D3**: review-plan Blocker → retry create-plan (max 2 lần), review-implement CHANGES REQUESTED → retry implement-plan (max 2 lần)
- **D4**: Escalation khi: (a) 3 retry fail, (b) agent output chứa ESCALATE marker, (c) review phát hiện vấn đề ảnh hưởng luồng chính hệ thống
- **D5**: Logging đầy đủ mỗi step vào `.agents/memories/<RUN_ID>/pipeline.log`
- **D6**: Mỗi step dùng 1 lần gọi `gemini -p` riêng biệt (fresh context)
- **D7**: Runner viết bằng **TypeScript + Bun** — production-ready từ đầu
- **D8**: Model mặc định: **gemini-3.1-pro-preview**, configurable qua CLI flag hoặc config
- **D9**: Input modes: (a) Jira issue key `PRES-99`, (b) Jira URL `https://xxx.atlassian.net/browse/PRES-99`, (c) manual `--desc "..." --name "..."`
- **D10**: Approval mode: **yolo** cho tất cả steps — tin tưởng pipeline + review step kiểm tra sau

## Thiết kế kỹ thuật / Kiến trúc

### Architecture Overview

```
bun run pipeline <INPUT> [options]
│
│  INPUT modes:
│  ├── Jira key:  bun run pipeline PRES-99
│  ├── Jira URL:  bun run pipeline "https://xxx.atlassian.net/browse/PRES-99"
│  └── Manual:    bun run pipeline --desc "Build auth API" --name "AUTH_API"
│
├── Step 1: JIRA_FETCH (skipped if --desc mode)
│   ├── gemini -p "<jira-fetch-prompt>" -m gemini-3.1-pro-preview --approval-mode=yolo
│   ├── Input: ISSUE_KEY (extracted from key or URL)
│   ├── Output: .agents/memories/<RUN_ID>/task.md
│   └── Exit: 0=success, 1=fail, 2=escalate
│
├── Step 2: CREATE_PLAN  
│   ├── gemini -p "<create-plan-prompt>" -m gemini-3.1-pro-preview --approval-mode=yolo
│   ├── Input: task.md (from step 1 or --desc)
│   ├── Output: .agents/plans/PLAN_<NAME>.md
│   │          + .agents/memories/<RUN_ID>/plan-ref.md (path reference)
│   └── Exit: 0=success, 1=fail, 2=escalate
│
├── Step 3: REVIEW_PLAN
│   ├── gemini -p "<review-plan-prompt>" -m gemini-3.1-pro-preview --approval-mode=yolo
│   ├── Input: .agents/plans/PLAN_<NAME>.md
│   ├── Output: .agents/memories/<RUN_ID>/review-plan.md
│   ├── Logic: 
│   │   ├── PASS → continue
│   │   ├── CẦN BỔ SUNG (no Blocker) → auto-fix, continue
│   │   └── CẦN XEM LẠI / Blocker → retry Step 2 (max 2x)
│   └── Exit: 0=pass, 1=needs-retry, 2=escalate
│
├── Step 4: IMPLEMENT_PLAN
│   ├── gemini -p "<implement-plan-prompt>" -m gemini-3.1-pro-preview --approval-mode=yolo
│   ├── Input: .agents/plans/PLAN_<NAME>.md
│   ├── Output: Code changes + .agents/memories/<RUN_ID>/progress.md
│   └── Exit: 0=success, 1=fail, 2=escalate
│
├── Step 5: REVIEW_IMPLEMENT
│   ├── gemini -p "<review-impl-prompt>" -m gemini-3.1-pro-preview --approval-mode=yolo
│   ├── Input: .agents/plans/PLAN_<NAME>.md + code changes
│   ├── Output: .agents/memories/<RUN_ID>/review-implement.md
│   ├── Logic:
│   │   ├── APPROVED → DONE
│   │   ├── APPROVED WITH NOTES → DONE (log notes)
│   │   └── CHANGES REQUESTED → retry Step 4 (max 2x)
│   └── Exit: 0=approved, 1=needs-retry, 2=escalate
│
└── Final: LOG + SUMMARY
    ├── Ghi tổng kết vào .agents/memories/<RUN_ID>/summary.md
    └── (Optional) Post Jira comment
```

### Memory Directory Structure

```
.agents/memories/
└── <RUN_ID>/              # VD: PRES-99/ hoặc AUTH_API/ (manual mode)
    ├── pipeline.log       # Full pipeline log (append)
    ├── pipeline-state.json # Pipeline state for resume
    ├── project-context.md # Step 0.5 output (chi tiết bên dưới)
    ├── task.md            # Step 1 output: Jira context / manual desc
    ├── plan-ref.md        # Step 2 output: Path reference to .agents/plans/PLAN_<NAME>.md
    ├── review-plan.md     # Step 3 output: Review result
    ├── progress.md        # Step 4 output: Implementation progress
    ├── review-implement.md # Step 5 output: Review result
    ├── summary.md         # Final summary
    └── escalation.md      # (if escalated) Context for user
```

#### `project-context.md` — Nội dung từ `load-project-context` skill

File này được tạo ở Step 0.5 bằng cách đọc và tổng hợp từ các nguồn sau:

| Source | Path | Nội dung |
|--------|------|----------|
| Global rules | `.agents/rules/*.md` | Coding conventions, quy tắc chung |
| AGENTS.md | `AGENTS.md` (project root) | Kiến trúc tổng thể, tech stack, conventions |
| Context rule | `.agents/rules/context-rule.md` | Decisions đã chốt, mục lục tài liệu |
| Epic context | `.agents/plans/PLAN_*.md` (related) | Plans liên quan đến tính năng đang làm |

> **Size control:** `prompt-builder.ts` sẽ truncate `project-context.md` nếu vượt **4000 tokens** để tránh chiếm quá nhiều context window. Chỉ giữ: architecture overview + active rules + relevant decisions.
```

### pipeline-state.json (cho resume)

```json
{
  "issueKey": "PRES-99",
  "startedAt": "2026-03-11T21:00:00Z",
  "currentStep": 3,
  "steps": {
    "1": { "status": "completed", "exitCode": 0, "duration": "45s" },
    "2": { "status": "completed", "exitCode": 0, "duration": "120s", "retries": 0 },
    "3": { "status": "running", "startedAt": "2026-03-11T21:05:00Z" }
  },
  "planFile": ".agents/plans/PLAN_SELLER_DASHBOARD.md",
  "featureName": "SELLER_DASHBOARD"
}
```

### Project Structure

Runner viết bằng TypeScript/Bun, cấu trúc production-ready:

```
.agents/pipeline/
├── package.json           # Bun project config
├── tsconfig.json          # TypeScript config
├── src/
│   ├── index.ts           # CLI entry point (argument parsing)
│   ├── pipeline.ts        # Pipeline orchestrator (main loop)
│   ├── config.ts          # Configuration (retry limits, model, timeouts per step)
│   ├── types.ts           # Type definitions
│   ├── steps/
│   │   ├── base-step.ts   # Abstract base class for steps
│   │   ├── jira-fetch.ts  # Step 1: Jira context extraction
│   │   ├── create-plan.ts # Step 2: Plan creation
│   │   ├── review-plan.ts # Step 3: Plan review
│   │   ├── implement.ts   # Step 4: Implementation
│   │   └── review-impl.ts # Step 5: Implementation review
│   ├── lib/
│   │   ├── gemini.ts      # gemini CLI wrapper (spawn, parse output)
│   │   ├── logger.ts      # Structured logging (console + file)
│   │   ├── state.ts       # State persistence (pipeline-state.json)
│   │   ├── memory.ts      # Memory bus (read/write memory files)
│   │   ├── retry.ts       # Retry logic with backoff
│   │   ├── escalation.ts  # Escalation detection & handling
│   │   ├── input-parser.ts # Parse input (Jira key, URL, manual desc)
│   │   └── prompt-builder.ts # Build prompts from templates + context
│   └── prompts/
│       ├── step1-jira-fetch.md
│       ├── step2-create-plan.md
│       ├── step3-review-plan.md
│       ├── step4-implement-plan.md
│       └── step5-review-implement.md
└── __tests__/
    ├── input-parser.test.ts
    ├── result-parser.test.ts
    ├── retry.test.ts
    └── pipeline.test.ts
```

### Prompt Template Pattern

Mỗi prompt template chứa:

```markdown
# SYSTEM CONTEXT (đọc từ skill tương ứng)
You are running as part of an automated pipeline.
You MUST follow these instructions exactly.

## Project Context
<Nội dung project-context.md — inject từ load-project-context output>
<Bao gồm: architecture overview, coding conventions, existing patterns, decisions đã chốt>

## Your Expertise
<Nội dung SKILL.md tương ứng — inject vào prompt>

## Input Context
<Cat nội dung từ memory files — inject vào prompt>

## Task
<Mô tả cụ thể task của step này>

## Output Requirements
You MUST write your output to these files:
- Primary output: `.agents/memories/<RUN_ID>/<output-file>.md`
- Pipeline marker: Output EXACTLY one of these on the LAST LINE:
  - `PIPELINE_RESULT:PASS` — Step completed successfully
  - `PIPELINE_RESULT:FAIL:<reason>` — Step failed, can retry
  - `PIPELINE_RESULT:ESCALATE:<reason>` — Needs human intervention

## Rules
1. Do NOT ask questions — make reasonable assumptions
2. Do NOT wait for user input — complete autonomously
3. If you encounter a critical issue affecting main system flow → output ESCALATE
4. Work within the project at: <WORKSPACE_PATH>
```

> **Note:** `prompt-builder.ts` sẽ hỗ trợ `--output-format json` flag của gemini CLI trong tương lai để parse structured output thay vì regex marker. Hiện tại dùng text + PIPELINE_RESULT marker cho đơn giản.

### Output Parsing Strategy

Pipeline runner parse exit từ gemini output bằng:

1. **gemini exit code**: 0 = process thành công, non-0 = crash
2. **PIPELINE_RESULT marker**: Dòng cuối cùng trong output — regex pattern
3. **Memory file existence**: Kiểm tra file output có được tạo không
4. **Fallback**: Nếu không có marker → check memory file content → infer result

```typescript
// Parse result from gemini output
function parseResult(output: string): StepResult {
  const markerMatch = output.match(/PIPELINE_RESULT:(PASS|FAIL|ESCALATE)(?::(.+))?/);
  
  if (markerMatch) {
    return {
      status: markerMatch[1] as 'PASS' | 'FAIL' | 'ESCALATE',
      reason: markerMatch[2] || undefined,
    };
  }
  
  // Fallback: check if expected output file was created
  return { status: 'UNKNOWN', reason: 'No PIPELINE_RESULT marker found' };
}
```

### Retry & Escalation Logic

```typescript
async function retryStep(step: PipelineStep, maxRetries = 2): Promise<StepResult> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const result = await step.execute();
    if (result.status === 'PASS') return result;
    if (result.status === 'ESCALATE') return result;
    
    logger.warn(`Retry ${attempt}/${maxRetries} for ${step.name}...`);
    step.injectFeedback(result.reason);
  }
  
  return { status: 'ESCALATE', reason: `Max retries (${maxRetries}) exhausted` };
}

async function escalate(stepNum: number, reason: string, context: PipelineContext): Promise<void> {
  await memory.write('escalation.md', {
    step: stepNum,
    reason,
    retryOutputs: context.retryHistory,
    collectedFiles: context.memoryFiles,
    suggestedActions: inferSuggestedActions(reason),
  });
  logger.error(`🚨 ESCALATED at Step ${stepNum}: ${reason}`);
  process.exit(2);
}
```

### Feedback Loop Pattern (retry)

Khi step fail và retry, prompt step tiếp theo inject thêm:

```markdown
## 🔄 RETRY CONTEXT (Attempt 2/3)

Previous attempt failed. Here is the feedback:

### Previous Review Result
<Cat nội dung review-plan.md hoặc review-implement.md>

### Identified Issues
<Extract Blockers/Majors từ review>

### Your task
Fix the issues identified above and regenerate the output.
IMPORTANT: Address EVERY blocker listed. Do NOT ignore any.
```

## Execution Boundary

### Allowed Files (chỉ được sửa/tạo mới)

- `.agents/pipeline/package.json` (🆕): Bun project config
- `.agents/pipeline/tsconfig.json` (🆕): TypeScript config
- `.agents/pipeline/src/index.ts` (🆕): CLI entry point
- `.agents/pipeline/src/pipeline.ts` (🆕): Pipeline orchestrator
- `.agents/pipeline/src/config.ts` (🆕): Configuration
- `.agents/pipeline/src/types.ts` (🆕): Type definitions
- `.agents/pipeline/src/steps/base-step.ts` (🆕): Abstract step class
- `.agents/pipeline/src/steps/jira-fetch.ts` (🆕): Step 1
- `.agents/pipeline/src/steps/create-plan.ts` (🆕): Step 2
- `.agents/pipeline/src/steps/review-plan.ts` (🆕): Step 3
- `.agents/pipeline/src/steps/implement.ts` (🆕): Step 4
- `.agents/pipeline/src/steps/review-impl.ts` (🆕): Step 5
- `.agents/pipeline/src/lib/gemini.ts` (🆕): CLI wrapper
- `.agents/pipeline/src/lib/logger.ts` (🆕): Structured logging
- `.agents/pipeline/src/lib/state.ts` (🆕): State persistence
- `.agents/pipeline/src/lib/memory.ts` (🆕): Memory bus
- `.agents/pipeline/src/lib/retry.ts` (🆕): Retry logic
- `.agents/pipeline/src/lib/escalation.ts` (🆕): Escalation handling
- `.agents/pipeline/src/lib/input-parser.ts` (🆕): Input parsing
- `.agents/pipeline/src/lib/prompt-builder.ts` (🆕): Prompt building
- `.agents/pipeline/src/prompts/*.md` (🆕): 5 prompt templates
- `.agents/pipeline/__tests__/*.test.ts` (🆕): Unit tests
- `.agents/memories/` (🆕 directory): Runtime memory state

### Do NOT Modify (CẤM TUYỆT ĐỐI)

- `.agents/skills/` — Tất cả skill files hiện có (chỉ reference, không sửa)
- `.agents/rules/` — Tất cả rule files
- `.agents/plans/**` (trừ memory directory) — Plans hiện có
- `.agent/skills/orchestrator/` — oh-my-ag orchestrator gốc (chỉ tham khảo)
- `cli/` — oh-my-ag CLI source code (không sửa)

## Implementation Order

| Order | File | Depends On | AC(s) | Est. Effort |
|-------|------|------------|-------|-------------|
| 1 | `package.json` + `tsconfig.json` | - | - | Small |
| | `package.json` cần define: `"scripts": { "pipeline": "bun run src/index.ts" }` | | | |
| 2 | `src/types.ts` | #1 | - | Small |
| | Bao gồm `StepTimeoutMs` type cho per-step timeout config | | | |
| 3 | `src/config.ts` | #2 | AC-1 | Small |
| | Thêm `STEP_TIMEOUTS: { fetch: 120_000, plan: 300_000, review: 300_000, implement: 900_000 }` | | | |
| 4 | `src/lib/logger.ts` | #2, #3 | AC-5 | Small |
| 5 | `src/lib/input-parser.ts` | #2 | AC-9, AC-10 | Medium |
| 6 | `src/lib/memory.ts` | #2, #4 | AC-2 | Medium |
| 7 | `src/lib/state.ts` | #2, #6 | AC-2, AC-8 | Medium |
| 8 | `src/lib/gemini.ts` | #2, #4 | AC-1 | Medium |
| | Spawn `gemini -p` với AbortSignal timeout từ config | | | |
| 9 | `src/lib/prompt-builder.ts` | #2, #6 | AC-3 | Medium |
| | Inject `project-context.md` vào `## Project Context` section của mọi prompt | | | |
| 10 | `src/lib/retry.ts` | #2, #4, #8 | AC-4 | Medium |
| 11 | `src/lib/escalation.ts` | #2, #4, #6 | AC-5, AC-6 | Medium |
| 12 | `src/steps/base-step.ts` | #2–#11 | - | Medium |
| 13 | `src/steps/jira-fetch.ts` | #12 | AC-1, AC-9 | Medium |
| 14 | `src/steps/create-plan.ts` | #12 | AC-1, AC-3 | Large |
| 15 | `src/steps/review-plan.ts` | #12 | AC-4 | Large |
| 16 | `src/steps/implement.ts` | #12 | AC-1, AC-3 | Large |
| 17 | `src/steps/review-impl.ts` | #12 | AC-4 | Large |
| 18 | `src/prompts/*.md` (5 files) | - | AC-3 | Large |
| 19 | `src/pipeline.ts` | #12–#17 | AC-1–AC-8, AC-11 | Large |
| | Step 0.5: gọi `load-project-context` → ghi `project-context.md` trước step 2 | | | |
| 20 | `src/index.ts` | #19 | AC-1, AC-8–AC-10 | Medium |
| 21 | `__tests__/*.test.ts` | #5, #8, #10 | AC-4, AC-9 | Medium |

## Pre-flight Check

> Implement agent chạy checklist này TRƯỚC khi bắt đầu code.

- [ ] Bun ≥ 1.1 đã cài: `bun --version`
- [ ] gemini CLI đã cài: `which gemini`
- [ ] Jira MCP server available (optional): `gemini -p "test Jira connection"`
- [ ] `.agents/skills/` directory tồn tại với đầy đủ 6 skills (incl. `load-project-context`)
- [ ] Branch đã tạo: `git checkout -b feat/cli-auto-pipeline`
- [ ] Plan version đang implement: `v1.3` (check latest version!)

## Rủi ro / Edge cases

| Risk | Likelihood | Impact | Trigger | Mitigation | Owner |
|------|-----------|--------|---------|------------|-------|
| gemini CLI output format thay đổi | Medium | High | CLI update | Pin version, validate output structure | Agent |
| Prompt quá dài vượt context window | High | High | Large skill files | Trim skill content, chỉ inject sections cần thiết | Agent |
| gemini crash giữa step | Medium | Medium | Network/API issues | State persistence → resume from last step | Agent |
| review-plan luôn fail (vòng lặp vô hạn) | Low | High | Bad plan template | Max 2 retry + escalate | Agent |
| Memory files bị corrupt/thiếu | Low | Medium | Disk issue | Validate file existence + content trước mỗi step | Agent |
| Jira MCP không available trong gemini | Medium | High | Config issue | Pre-flight check + clear error message | User |
| gemini không output PIPELINE_RESULT marker | High | Medium | Agent ignores instructions | Fallback: check memory file existence → infer result | Agent |
| Step timeout (gemini chạy quá lâu) | Medium | Medium | Network lag / complex task | Per-step timeout config: fetch=2m, plan=5m, review=5m, implement=15m | Agent |
| project-context.md quá lớn | Medium | Medium | Dự án lớn, nhiều rules/plans | Truncate ở 4000 tokens trong `prompt-builder.ts`, chỉ giữ sections thiết yếu | Agent |

## Test plan

### Happy paths

- **T1: Dry run end-to-end**: `bun run pipeline PRES-99 --dry-run` → in ra 5 steps đúng thứ tự, không gọi gemini thật
- **T2: Single step**: `bun run pipeline PRES-99 --step=1 --dry-run` → chỉ chạy step 1
- **T3: Resume**: Tạo mock state file (step 1-2 done) → `bun run pipeline PRES-99 --resume` → chạy từ step 3
- **T4: Jira URL input**: `bun run pipeline "https://xxx.atlassian.net/browse/PRES-99"` → extract key, chạy bình thường
- **T5: Manual input**: `bun run pipeline --desc "Build auth" --name "AUTH_API"` → skip step 1, chạy từ step 2
- **T6: Real Jira ticket**: Chạy thực pipeline trên 1 ticket Jira đơn giản (Tier S)
- **T7: Project context loading**: Verify `project-context.md` được tạo đúng ở Step 0.5 và inject vào mọi prompt từ Step 2+

### Edge cases

- **E1: Invalid issue key**: `bun run pipeline INVALID` → exit 1 với error message
- **E2: Missing gemini CLI**: Unset PATH → exit 1 với "gemini not found"
- **E3: Memory dir already exists**: Chạy lại pipeline trên cùng issue → hỏi overwrite/resume
- **E4: Retry exhausted**: Mock review-plan luôn output FAIL → pipeline escalate sau 2 retries
- **E5: Step timeout**: Gemini chạy vượt timeout config → kill process, log timeout, retry hoặc escalate
- **E6: Project context quá lớn**: `.agents/rules/` có >20 files → verify truncation hoạt động đúng, prompt không vượt context window

### Regression

- Kiểm tra `.agents/skills/` files KHÔNG bị sửa sau khi chạy pipeline
- Kiểm tra `.agents/plans/` files hiện có KHÔNG bị overwrite

## Decisions đã chốt

| # | Quyết định | Source |
|---|-----------|--------|
| 1 | Dùng **TypeScript + Bun** cho runner — production-ready từ đầu | User — "chỉn chu cẩn thận, sản phẩm production ready" |
| 2 | State truyền qua filesystem (`.agents/memories/`) thay vì Serena MCP | User + analysis — không phụ thuộc external MCP server |
| 3 | Mỗi step = 1 lần gọi `gemini -p` (fresh context) | Technical — tránh context overflow |
| 4 | Skill files chỉ reference, không sửa | Architecture — giữ backward compatibility với IDE mode |
| 5 | Model mặc định: **gemini-3.1-pro-preview** | User — tài khoản ultra, không lo token |
| 6 | Max retry = 2 cho review steps | User — đủ cho hầu hết cases, escalate nếu quá |
| 7 | PIPELINE_RESULT marker là convention cho output parsing | Technical — đơn giản, regex-parseable |
| 8 | Input modes: Jira key + Jira URL + manual desc | User — linh hoạt, Jira không bắt buộc |
| 9 | **yolo** approval mode cho tất cả steps | User — "cần vs lỗi đâu thì tối ưu pipeline" |
| 10 | gemini CLI là CLI mặc định, support multi-vendor sau | User — hiện tại dùng gemini, mở rộng sau |

## Những điểm dễ thay đổi trong tương lai

- **Multi-vendor**: Thay `gemini` bằng `claude` / `codex` → chỉ cần sửa `config.ts`
- **Multi-agent parallel**: Chạy N pipeline instances song song → `parallel-pipeline.ts` wrapper
- **QA orchestrator**: Thêm Step 6 (QA review) sau review-implement
- **Antigravity extension**: Bridge giữa IDE và pipeline CLI → extension gọi pipeline subprocess
- **Git automation**: Thêm step 0 (create branch) + step 6 (commit + PR)
- **Jira sync**: Tự động transition Jira status theo pipeline progress
- **Structured JSON output**: Migrate từ text + PIPELINE_RESULT marker sang `--output-format json` cho reliable parsing

## Change Log

| Version | Date | Changes | Reason |
|---------|------|---------|--------|
| 1.3 | 2026-03-11 | Polish to 100%: fix AC-3 ref consistency, +AC-11 (project-context), +project-context.md content sources & size control, +context-overflow risk, +test cases T7/E5/E6 | All 12 criteria → 10/10 |
| 1.2 | 2026-03-11 | Auto-fix: +load-project-context, fix plan-ref.md naming, fix bash→TS refs, +timeout config, +JSON output note, +bun entry point | Review 84.8% → 5M + 3m fixed |
| 1.1 | 2026-03-11 | TypeScript/Bun runner, model gemini-3.1-pro-preview, Jira optional + URL, yolo all steps, AC-9/AC-10 | User decisions confirmed |
| 1.0 | 2026-03-11 | Initial plan | - |
