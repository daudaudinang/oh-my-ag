---
name: implement-plan
description: |
  Triển khai plan theo workflow từng phase. Dùng khi user chạy `/implement-plan <plan_file_path>`
  hoặc yêu cầu "implement/thực thi plan". Agent sẽ đọc plan, đánh giá tính khả thi,
  tuân thủ Execution Boundary, và triển khai tuần tự từng phase kèm verification.
  Kích hoạt khi user nói "implement plan", "triển khai plan", "thực thi kế hoạch".
---

# Goal

Triển khai plan thành code hoạt động đúng, tuần tự từng phase. Tuân thủ tuyệt đối Execution Boundary, có walkthrough + manual test guide sau mỗi phase — đảm bảo chất lượng qua việc tự động chạy Verify Commands của Acceptance Criteria và user confirm thủ công.

---

# Cấu trúc Folder

```
.agents/
├── plans/
│   ├── <normalized-plan-name>/          # Folder cho plan đang triển khai
│   │   ├── phase-1-walkthrough.md       # Walkthrough sau phase 1
│   │   ├── phase-2-walkthrough.md       # Walkthrough sau phase 2
│   │   └── ...
│   └── <plan-file>.md                   # File plan gốc
```

Quy tắc chuẩn hóa tên folder (normalized-plan-name): lowercase, thay khoảng trắng bằng `_`, loại bỏ ký tự đặc biệt, bỏ `.md`, tối đa 50 ký tự. VD: `PLAN_Feature.md` -> `plan_feature`.

---

# Instructions

## PHASE 0: 🚀 Khởi tạo & Đánh giá

### 0.1. Đọc file plan
1. Nhận path file plan từ lệnh `/implement-plan` hoặc từ user. Kiểm tra xem file plan có tồn tại không.
2. Đọc toàn bộ nội dung plan.
3. Xác định:
   - **Tier của plan** (S/M/L)
   - **Context Sharding (Tier L)**: Nếu plan là Tier L và có chia nhỏ thành các Story files, ghi chú lại danh sách các story files tương ứng. Trong quá trình chạy thay vì nạp toàn bộ một lúc vào context, hãy read story file tương ứng khi vào mỗi phase.
   - Các phases, tasks.

### 0.2. 🌐 Nạp Project Context
Kiểm tra xem project có skill `load-project-context` không (`.agents/skills/load-project-context/SKILL.md`). Nếu có → Đọc SKILL.md, thực thi theo instructions để nạp context theo tầng trước khi bắt tay vào eval.

### 0.3. Nhận diện Execution Boundary & Dependencies (QUAN TRỌNG)
1. **Execution Boundary**:
   - Xác định và tự ghi nhớ `Allowed Files`. Agent CHỈ được sửa/tạo các file trong scope này.
   - Xác định và tự ghi nhớ `Do NOT Modify`. TUYỆT ĐỐI KHÔNG sửa các file này. Nếu buộc phải sửa, DỪNG lại và hỏi ý kiến user.
2. **Dependencies / Prerequisites**:
   - Kiểm tra xem plan yêu cầu dependencies cụ thể nào trước không (ví dụ: package, backend server).
   - Nếu chưa đáp ứng, xử lý nó hoặc hỏi user.

### 0.4. Tạo folder & Kiểm tra Test Framework
1. Tạo output folder `.agents/plans/<normalized-name>/`.
2. Kiểm tra Test Framework trong project (jest, pytest, vitest). Ghi nhận `HAS_UNIT_TEST = true / false`.

### 0.5. ✅ Chạy Pre-flight Check (nếu plan có)
1. Tìm section "Pre-flight Check" trong plan.
2. Nếu có → thực thi TẪNg checklist item:
   - **Dependencies**: chạy `uv sync` / `bun install` / `npm install`
   - **Services**: kiểm tra backend/DB đang chạy (curl health endpoint hoặc kiểm tra port)
   - **Env vars**: kiểm tra biến cần thiết (`printenv | grep KEY`)
   - **Branch**: tạo branch nếu chưa có (`git checkout -b feat/<feature_name>`)
   - **Plan version**: xác nhận đang implement version mới nhất
3. Nếu FAIL bất kỳ item → DỮNG, báo user, KHÔNG tiếp tục.
4. Nếu plan không có Pre-flight Check → bỏ qua.

### 0.6. Đánh giá tính khả thi
> **⚠️ QUY TẮC TỰ SUY LUẬN**: Tự suy luận, CHỈ output kết quả.

Tiêu chí:
- Plan bám sát codebase hiện tại?
- File đề cập thực sự tồn tại?
- Thứ tự task hợp lý?
- Phương án rõ ràng, an toàn với hệ thống hiện tại?

**Kết quả:**
- ✅ Đạt -> "Plan đã được đánh giá, bắt đầu triển khai phase 1..."
- ⚠️ Có vấn đề -> Liệt kê vấn đề, DỪNG lại hỏi user.

---

## PHASE N: 🔨 Triển khai từng phase

### N.1. Chuẩn bị Phase (Pre-check)
1. Nếu là Tier L, `view_file` Story file tương ứng của phase này.
2. Kiểm tra **Dependencies / Prerequisites** cụ thể của phase. Nếu thiếu, cài đặt (nếu an toàn) hoặc báo user.
3. **Đọc Implementation Order** (nếu plan có):
   - Xác định files thuộc phase này từ bảng Implementation Order
   - Triển khai ĐÚNG THỨ TỰ (Order column) — file phụ thuộc implement trước
   - Nếu file phụ thuộc chưa implement → implement nó trước
   - Nếu plan KHÔNG có Implementation Order → implement theo thứ tự trong plan
4. Thông báo: "Bắt đầu Phase {N}: {phase_title}" kèm danh sách tasks.

### N.2. Triển khai Task
> **BẮT BUỘC**: Code tuân thủ danh sách Execution Boundary. Khai báo cấm những đường dẫn ngoài phạm vi.

**Progress Protocol** (báo trước/sau mỗi task):
```
Trước: 📌 Phase {N} — Task {M}/{Total}: {task_title} | Files: {file_list}
Sau:   ✅ Task {M}/{Total} done — {files_changed} files
```

**Flow A: TDD (`HAS_UNIT_TEST = true`)**
1. **Kiểm tra tests hiện có**: grep/find test files liên quan
   - Nếu đã có tests → đọc hiểu coverage hiện tại
   - Nếu chưa có tests cho file này → viết test mới
2. **Viết test cases** (trước implement):
   - Happy paths: ít nhất 2 cases
   - Edge cases: ít nhất 1 case (null, empty, boundary)
   - Test naming: `describe('feature') > it('should...')`
3. **Implement code** để pass tests
4. **Chạy test suite** (toàn bộ, không chỉ tests vừa viết):
   - PASS → tiếp tục
   - FAIL → Flow Debug
5. **Regression check**: chạy test suite toàn project để đảm bảo
   không break tests khác

**Flow B: Direct Implementation (`HAS_UNIT_TEST = false`)**
1. Implement code trực tiếp.
2. Chạy lint. Nếu lỗi -> Sửa.

### N.3. Xác thực Acceptance Criteria (QUAN TRỌNG)
Sau khi hoàn thành code của phase:
1. Chạy lint/test toàn bộ (nếu có).
2. **Verify AC**: Trích xuất các lệnh `Verify Command` từ Acceptance Criteria. Chạy thực tế trên terminal. Ghi nhận output là Pass hay Fail. Nếu Fail -> Flow Debug.

### N.4. Báo cáo Walkthrough & Confirm
Tạo file walkthrough tại `.agents/plans/<normalized-name>/phase-{N}-walkthrough.md` theo template hướng dẫn. Phải có phần **AC Verification** và đánh dấu **Boundary Check**.

Sau đó:
- Tóm tắt cho user số task hoàn thành, tình trạng AC. Hỏi user: "➡️ Tiếp tục Phase {N+1}? (user confirm)"

---

## 🐛 Flow Debug (Lỗi Lint / Test / AC)
> **⚠️ QUY TẮC TỰ SUY LUẬN**: Tự suy luận, CHỈ output kết quả.

1. Tự đọc error message từ test fail hoặc AC execution fail.
2. Xác nhận nguồn lỗi. TỰ SỬA LỖI. Đã sửa xong -> "Đã sửa lỗi: [mô tả ngắn]".
3. Nếu fix 3 vòng vẫn không xong -> Áp dụng **Rollback Protocol**.

---

## 🔙 Rollback Protocol (khi Phase thất bại)

Khi Flow Debug fail sau 3 lần:

1. **Liệt kê files đã sửa** trong phase hiện tại (từ walkthrough draft hoặc `git diff`).
2. **Hỏi user** với 3 options:
   - a) "Mình revert tất cả thay đổi của Phase N?" → `git checkout -- <files_changed>`
   - b) "Giữ lại thay đổi, tiếp tục debug thủ công?" → ghi notes cho user
   - c) "Rollback toàn bộ plan?" → `git stash` tất cả từ Phase 1
3. **Nếu user chọn revert**:
   - Chạy: `git checkout -- <files_changed_in_phase_N>`
   - Cập nhật walkthrough: "Phase N: ❌ REVERTED"
   - Hỏi: "Tiếp tục Phase N với approach khác hay dừng?"

> ⚠️ KHÔNG tự ý revert mà không hỏi user.

## Kết thúc: Báo cáo tổng kết

```
🎉 Hoàn thành triển khai plan: {plan_name}

📊 Tổng kết:
- {N} phases hoàn thành
- {M} files changed / created
- Lint: ✅ / ❌ | Test: ✅ / N/A | AC Verified: ✅ / ❌

📁 Walkthrough files:
- .agents/plans/<name>/phase-1-walkthrough.md
...

💡 Đề xuất tiếp theo: Chạy /review-implement để review code đã tạo
```

---

# Quy tắc & Ràng buộc (Constraints)

1. **Tuân thủ Boundary**: 🚫 TUYỆT ĐỐI không sửa đổi files trong danh sách `Do NOT Modify`. Sửa file ngoài vùng an toàn là vi phạm chí mạng.
2. **Data-driven Verification**: ✅ Bắt buộc phải chạy `Verify Command` để chứng thực AC. Không verify bằng lý thuyết màn hình.
3. **Internal Monologue**: 🚫 Không kể lại log suy luận dài dòng. Chỉ output hành động hay báo cáo lỗi.
4. **Hỏi user đúng lúc**: Chỉ dừng khi boundary vi phạm, debug bế tắc, plan không khả thi, hoặc khi phase xong cần xác nhận.
5. **Recovery**: Xử lý đứt gãy kết nối bằng cách đọc file walkthrough của phase gần nhất và hoàn thành tiếp tục. KHÔNG bắt đầu lại từ đầu nếu lỗi.

---

# Checklist Triển khai (Internal Use)

```markdown
## Checklist /implement-plan
- [ ] **PHASE 0: KHỞI TẠO**
  - [ ] Đọc file plan + Context sharding (nếu Tier L)
  - [ ] Check Dependencies & Ghi nhận Execution Boundary
  - [ ] Chạy Pre-flight Check (nếu plan có)
  - [ ] Đánh giá tính khả thi & Kiểm tra test framework
- [ ] **PHASE N: {title}**
  - [ ] Pre-check Phase Dependencies
  - [ ] Đọc Implementation Order (nếu có) → implement đúng thứ tự
  - [ ] Triển khai tuân thủ Execution Boundary
  - [ ] Progress tracking (báo trước/sau mỗi task)
  - [ ] Chạy Lint/Test toàn hệ thống
  - [ ] **Verify AC (Chạy Verify Commands)**
  - [ ] Tạo Walkthrough (Check boundary & AC)
  - [ ] User confirm
- [ ] **Nếu FAIL**: Flow Debug (3 lần) → Rollback Protocol
...
```

# Các Skills liên quan
- `.agents/skills/create-plan/SKILL.md` — Tạo plan 
- `.agents/skills/review-plan/SKILL.md` — Review plan
- `.agents/skills/review-implement/SKILL.md` — Review code sau implement

<!-- Generated by Skill Generator v3.2 / Updated BMAD -->
