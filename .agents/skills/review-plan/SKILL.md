---
name: review-plan
description: |
  Review file plan theo workflow 4 bước (đọc plan, đối chiếu codebase, chấm theo tiêu chí, xuất report).
  Dùng khi user chạy `/review-plan` hoặc yêu cầu "review plan/kế hoạch triển khai".
  Chỉ review, không implement và không tự ý sửa plan.
  Kích hoạt khi user nói "review plan", "đánh giá plan", "soát plan",
  "check plan có thiếu/sai gì không", "plan này ổn chưa".
---

# Goal

Đánh giá plan triển khai bằng cách đối chiếu với codebase thực tế và reference documents, phát hiện sai sót / thiếu sót / rủi ro chưa cover, phân loại severity (Blocker/Major/Minor), và đưa ra đề xuất cụ thể có dẫn chứng — giúp plan đạt mức actionable trước khi implement.

---

# Instructions

## Bước 0: Xác định file plan cần review

1. Nhận path file plan từ lệnh `/review-plan <plan_file>` hoặc từ yêu cầu user.
2. Kiểm tra file tồn tại:
   - **Nếu không** → Báo user: "File plan không tồn tại tại đường dẫn này."
   - **Nếu có** → Tiếp tục.

## Bước 0.1: 🎯 Triage Depth (theo Tier plan)

Đọc header plan → xác định Tier → chọn mức review phù hợp:

| Tier | Review scope | Tiêu chí chấm |
|------|-------------|---------------|
| **S (Simple)** | Quick review — chỉ files được nhắc trong plan | 4 tiêu chí: Độ chính xác, Độ đầy đủ, AC, Execution Boundary |
| **M (Medium)** | Standard review — files + grep liên quan | 12 tiêu chí đầy đủ |
| **L (Large)** | Deep review — files + grep + cross-reference docs + story files consistency | 12 tiêu chí + Cross-reference checking |

1. Đọc header plan → xác định `Tier` (nếu plan không có Tier → mặc định M).
2. Ghi Tier + Review depth vào report header.
3. Tier S → bỏ qua các tiêu chí trọng số Thấp, rút gọn Bước 2.

## Bước 0.2: 📝 Plan Format Detection

1. Đọc plan → kiểm tra có header format chuẩn (Version, Tier, Type) không?
   - ✅ Có → plan template v2/v3, proceed bình thường.
   - ❌ Không → plan custom/cũ.
2. Nếu plan custom/cũ:
   - Ghi rõ trong report: "⚠️ Plan không theo template chuẩn"
   - Vẫn review theo tiêu chí nhưng ghi **N/A** cho sections không tồn tại
   - Đề xuất: "Nên migrate plan sang template chuẩn trước khi implement"

## Bước 0.5: 🌐 Nạp Project Context (nếu có)

> **Hook tự động:** Kiểm tra xem project có skill `load-project-context` không.
> Nếu có → đọc `.agents/skills/load-project-context/SKILL.md` và thực thi instructions.
> Skill này giúp agent nắm đúng source-of-truth, contract nghiệp vụ, và scope trước khi review.
> Nếu không có → bỏ qua, tiếp tục bình thường.

1. Tìm file `.agents/skills/load-project-context/SKILL.md` trong workspace.
2. **Nếu tồn tại** → Đọc SKILL.md, thực thi theo instructions (xác định task từ plan, nạp context theo tầng).
3. **Nếu không** → Bỏ qua bước này.

## BƯỚC 1: 📖 Đọc và phân tích plan

1. **Đọc toàn bộ file plan** bằng `view_file`.
2. **Kiểm tra sections theo ma trận Tier:**

   | Section | Bắt buộc (S) | Bắt buộc (M/L) | Có trong plan? |
   |---------|-------------|-----------------|----------------|
   | Mục tiêu | ✅ | ✅ | {✅/❌} |
   | Acceptance Criteria | ✅ | ✅ | {✅/❌} |
   | Non-goals | - | ✅ | {✅/❌} |
   | Bối cảnh hiện trạng | - | ✅ | {✅/❌} |
   | Dependencies / Prerequisites | - | ✅ | {✅/❌} |
   | Yêu cầu nghiệp vụ | - | ✅ | {✅/❌} |
   | Execution Boundary (Allowed + Do NOT) | ✅ | ✅ | {✅/❌} |
   | Implementation Order | - | ✅ | {✅/❌} |
   | Pre-flight Check | - | ✅ | {✅/❌} |
   | Thiết kế kỹ thuật | - | ✅ | {✅/❌} |
   | Test plan | ✅ | ✅ | {✅/❌} |
   | Rủi ro / Edge cases (Risk Matrix) | - | ✅ | {✅/❌} |
   | Decisions đã chốt | - | ✅ (nếu có Phase 3) | {✅/❌} |
   | Change Log | - | ✅ (nếu cập nhật plan) | {✅/❌} |

3. **Trích xuất danh sách** cần đối chiếu:
   - **File paths** được nhắc đến → danh sách kiểm tra tồn tại
   - **Components / functions / types** được nhắc → danh sách kiểm tra định nghĩa
   - **API endpoints / workflows** được nhắc → danh sách kiểm tra logic
   - **Reference documents** (nếu plan có "Tham chiếu" section) → danh sách cross-reference

## BƯỚC 1.5: 📝 Structural Validation (quick format check)

Với mỗi section tồn tại, kiểm tra format bên trong:

| Section | Format chuẩn | Check |
|---------|-------------|-------|
| Acceptance Criteria | Bảng có cột: #, Criteria, Verify Command, Pass/Fail | ✅/❌ |
| Execution Boundary | Có 2 sub-sections: "Allowed" + "Do NOT Modify" | ✅/❌ |
| Risk Matrix | Bảng có cột: Likelihood, Impact (không phải flat list) | ✅/⚠️ |
| Implementation Order | Bảng có cột: Order, File, Depends On, AC(s) | ✅/❌ |
| Pre-flight Check | Checklist items (- [ ]) | ✅/❌ |
| Change Log | Bảng có cột: Version, Date, Changes, Reason | ✅/N/A |

> **Mục đích:** Phát hiện plan dùng format cũ hoặc sai cấu trúc — implement agent có thể
> misread plan nếu format không chuẩn.

## BƯỚC 2: 🔍 Đối chiếu với codebase

### 2.1. Kiểm tra sự tồn tại của files
- Với mỗi file path trong plan:
  - `view_file_outline` hoặc `view_file` để xác nhận → ✅ tồn tại / ❌ không tồn tại

### 2.2. Xác nhận cấu trúc code khớp
- Đọc types/interfaces được đề cập → so sánh với plan
- Đọc functions/components được đề cập → so sánh với plan
- Ghi nhận bất kỳ sai lệch nào (tên field khác, structure khác, ...)

### 2.3. Tìm thiếu sót
- `grep_search` để tìm occurrences liên quan mà plan chưa nhắc
- Tìm vị trí code bị ảnh hưởng nhưng chưa có trong "Execution Boundary"
- Xác định impacts tiềm ẩn: breaking changes, data migration, auth scope, ...

### 2.4. Cross-reference checking (Tier M/L, nếu plan có references)

1. Tìm section "Tham chiếu" hoặc "References" trong plan.
2. Nếu có reference documents:
   - Đọc reference docs (chỉ sections liên quan).
   - Kiểm tra: plan có trích dẫn đúng source không?
   - Kiểm tra: có mâu thuẫn giữa plan và source docs không?
   - Kiểm tra: có quyết định trong source docs mà plan chưa phản ánh không?
3. Nếu plan không có references → bỏ qua.

> **Hook Multi-Merchant:** Nếu plan liên quan Multi-Merchant Marketplace, đối chiếu thêm với `.agents/rules/context-rule.md` để kiểm tra alignment với contract/decision đã chốt.

## BƯỚC 3: 📊 Đánh giá theo tiêu chí

Chấm điểm theo tiêu chí (tham khảo chi tiết: `resources/review_criteria.md`):

### Tier S — Chấm 4 tiêu chí bắt buộc:

| # | Tiêu chí | Trọng số |
|---|----------|----------|
| 1 | Độ chính xác | 🔴 Cao |
| 2 | Độ đầy đủ | 🔴 Cao |
| 3 | Acceptance Criteria | 🔴 Cao |
| 4 | Execution Boundary | 🔴 Cao |

### Tier M/L — Chấm 12 tiêu chí:

| # | Tiêu chí | Trọng số | Hệ số |
|---|----------|----------|--------|
| 1 | Độ chính xác | 🔴 Cao | ×3 |
| 2 | Độ đầy đủ | 🔴 Cao | ×3 |
| 3 | Acceptance Criteria | 🔴 Cao | ×3 |
| 4 | Execution Boundary | 🔴 Cao | ×3 |
| 5 | Risks coverage (Risk Matrix) | 🔴 Cao | ×3 |
| 6 | Backward compatibility | 🔴 Cao | ×3 |
| 7 | Thiết kế rõ ràng | 🟡 Trung bình | ×2 |
| 8 | Test plan | 🟡 Trung bình | ×2 |
| 9 | Dependencies | 🟡 Trung bình | ×2 |
| 10 | Điểm mở rộng | 🟢 Thấp | ×1 |
| 11 | Implementation Order | 🟢 Thấp | ×1 |
| 12 | Pre-flight Check | 🟢 Thấp | ×1 |

### Scoring Formula (Tier M/L)

```text
Weighted Score = Σ(score_i × hệ_số_i)

- Cao (×3):   tiêu chí 1-6  → max 180 pts
- TB (×2):    tiêu chí 7-9  → max 60 pts
- Thấp (×1): tiêu chí 10-12 → max 30 pts
- Total: max 270 pts

Thresholds:
- ≥ 243 (90%) → ✅ PASS
- 162–242 (60–89%) → ⚠️ CẦN BỔ SUNG
- < 162 (<60%) → ❌ CẦN XEM LẠI

Override: Bất kỳ tiêu chí Cao nào ≤ 5/10 → auto-downgrade
          Bất kỳ Blocker nào → auto-downgrade xuống CẦN XEM LẠI
```

> Tier S: chỉ chấm 4 tiêu chí Cao, max 120 pts. PASS ≥ 108, BỔ SUNG 72–107, XEM LẠI <72.

**Thang điểm:**
- ✅ **ĐẠT (9–10/10)**: Không có vấn đề
- ⚠️ **CẦN BỔ SUNG (6–8/10)**: Có vấn đề nhỏ, cần bổ sung
- ❌ **CẦN XEM LẠI (<6/10)**: Có vấn đề nghiêm trọng

**Phân loại findings theo severity** (bắt buộc):
1. 🔴 **Blocker** — Plan sai sót nghiêm trọng, implement sẽ gây bug/regression
2. 🟠 **Major** — Plan thiếu sót quan trọng, nên fix trước khi implement
3. 🟡 **Minor** — Cải thiện nhỏ, fix sau cũng không sao

## BƯỚC 4: 📝 Xuất report theo template

1. **Tổng hợp** theo template output tại `resources/review_template.md`.
2. **Kết luận** rõ ràng 1 trong 3:
   - ✅ **PASS** — Plan đủ cơ sở để triển khai
   - ⚠️ **CẦN BỔ SUNG** — Cần bổ sung X điểm trước khi implement
   - ❌ **CẦN XEM LẠI** — Có vấn đề nghiêm trọng cần xem lại
3. **Câu hỏi** cho user (nếu có chỗ "chưa chốt được" hoặc mơ hồ trong plan).

### Bước 4.1: 🔧 Đề xuất Auto-fix (khi CẦN BỔ SUNG)

Nếu kết luận là CẦN BỔ SUNG **VÀ** không có Blocker:

1. Hỏi user: "Muốn mình tự bổ sung {X} điểm vào plan không?"
2. Liệt kê rõ:
   - **Sẽ sửa:** {sections/nội dung cụ thể}
   - **Không sửa (cần user quyết định):** {nếu có}
3. Nếu user đồng ý → sửa trực tiếp file plan (chỉ bổ sung, không thay đổi kiến trúc/logic)
4. Nếu user từ chối → chỉ giữ report

> ⚠️ **KHÔNG auto-fix nếu:**
> - Có Blocker (cần user quyết định hướng xử lý)
> - Thay đổi Execution Boundary / Do NOT Modify (cần user confirm scope)
> - Thay đổi thiết kế kiến trúc (cần user review)

---

# Examples

## Ví dụ 1: Review plan — Happy path (Tier M)

**Input:**
```
/review-plan .agents/plans/PLAN_FONT_COMBOBOX.md
```

**Agent thực hiện:**

1. **Bước 0:** Đọc header → Tier M
2. **Bước 1:** Đọc plan → sections checklist → thiếu "Dependencies"
3. **Bước 2:** Đối chiếu:
   - `types.ts` ✅ tồn tại, có `TemplateCellStyle.fontFamily`
   - `font-select.tsx` ❌ chưa tồn tại (plan đề xuất tạo mới → đúng)
   - `grep_search "fontFamily"` → phát hiện 1 vị trí nữa tại `forEach child segment`
4. **Bước 3:** Chấm điểm 10 tiêu chí → phân loại findings:
   - 🟠 Major: thiếu 1 file position
   - 🟡 Minor: thiếu Dependencies section
5. **Bước 4:** Output report + đề xuất auto-fix

**Output (rút gọn):**
```markdown
## 📋 Kết quả Review Plan: Font Combobox

> **Tier**: M | **Review depth**: Standard

### ✅ Những điểm tốt
| Bối cảnh hiện trạng | ✅ Chính xác — `TemplateCellStyle.fontFamily` tại `types.ts:45` |
| Acceptance Criteria | ✅ SMART, có verify command |

### ⚠️ Cần bổ sung

#### 🟠 Major
##### 1. Thiếu vị trí forEach child segment
**Vấn đề:** `grep_search "fontFamily"` → `template-editor-dialog.tsx:2077-2083`
**Đề xuất:** Bổ sung vào Execution Boundary

#### 🟡 Minor
##### 1. Thiếu Dependencies section
**Đề xuất:** Thêm: Node.js 18+, shadcn/ui Combobox component

### 🔧 Auto-fix
> Muốn mình tự bổ sung 2 điểm vào plan không?
> - Sửa: thêm file position + Dependencies section
> - Không sửa: (không có)

**Kết luận:** ⚠️ CẦN BỔ SUNG — 2 điểm
```

---

# Constraints

## Guardrails

- 🚫 **CHỈ REVIEW** — KHÔNG implement code, KHÔNG tạo PR, KHÔNG chạy thay đổi ngoài review.
- 🚫 **KHÔNG SỬA FILE PLAN** nếu chưa hỏi user — trừ khi user chấp nhận auto-fix.
- 🚫 **KHÔNG BỊA THÔNG TIN** — Mọi nhận xét PHẢI có cơ sở từ codebase. Nếu không tìm thấy → ghi rõ "Không tìm thấy thông tin về X".

## Chất lượng nhận xét

- ✅ LUÔN LUÔN kèm **dẫn chứng cụ thể**: file path + line number/range + (optional) trích đoạn code.
- ✅ LUÔN LUÔN đề xuất **khả thi, actionable**: chỉ rõ thêm/sửa section nào, bổ sung file nào, test case nào.
- ✅ LUÔN LUÔN **phân loại severity**: Blocker / Major / Minor cho mọi finding.
- ✅ LUÔN LUÔN **acknowledge điểm tốt** — không chỉ chê.
- 🚫 KHÔNG ĐƯỢC nhận xét chung chung: "cần cải thiện", "nên xem lại" mà không có bước cụ thể.
- 🚫 KHÔNG ĐƯỢC suy luận/giả định về codebase — PHẢI dùng tools (`view_file`, `grep_search`, `view_file_outline`) để xác minh.

## Output

- ✅ LUÔN LUÔN dùng template output review tại `resources/review_template.md`.
- ✅ LUÔN LUÔN kết luận rõ ràng 1 trong 3: **PASS** / **CẦN BỔ SUNG** / **CẦN XEM LẠI**.
- ✅ LUÔN LUÔN đề xuất auto-fix khi CẦN BỔ SUNG + không có Blocker.

---

# Review Checklist (tự kiểm trước khi xuất report)

```markdown
## Review Checklist

- [ ] **BƯỚC 0: KHỞI TẠO**
  - [ ] Xác định file plan
  - [ ] Triage depth (Tier S/M/L)
  - [ ] Plan Format Detection (chuẩn/custom)
  - [ ] Nạp project context (nếu có)

- [ ] **BƯỚC 1: ĐỌC PLAN**
  - [ ] Đọc toàn bộ file plan
  - [ ] Kiểm tra sections checklist theo Tier (15 sections)
  - [ ] Trích xuất danh sách files/components/references

- [ ] **BƯỚC 1.5: STRUCTURAL VALIDATION**
  - [ ] Kiểm tra format AC (bảng + verify command)
  - [ ] Kiểm tra Execution Boundary (Allowed + Do NOT)
  - [ ] Kiểm tra Risk Matrix format (✅ bảng / ⚠️ flat list)
  - [ ] Kiểm tra Implementation Order format
  - [ ] Kiểm tra Pre-flight Check format
  - [ ] Kiểm tra Change Log (nếu plan cập nhật)

- [ ] **BƯỚC 2: ĐỐI CHIỀU CODEBASE**
  - [ ] Kiểm tra tất cả files đề cập có tồn tại
  - [ ] Xác nhận cấu trúc code khớp với mô tả
  - [ ] Tìm các vị trí liên quan nhưng chưa đề cập
  - [ ] Cross-reference checking (Tier M/L, nếu có refs)

- [ ] **BƯỚC 3: ĐÁNH GIÁ**
  - [ ] Chấm điểm theo tiêu chí (4 cho S, 12 cho M/L)
  - [ ] Tính weighted score theo formula
  - [ ] Phân loại findings: Blocker / Major / Minor
  - [ ] Liệt kê điểm tốt (có dẫn chứng)
  - [ ] Liệt kê điểm cần bổ sung (có đề xuất cụ thể)

- [ ] **BƯỚC 4: OUTPUT**
  - [ ] Tổng hợp theo template v3
  - [ ] Weighted score + kết luận rõ ràng (PASS / CẦN BỔ SUNG / CẦN XEM LẠI)
  - [ ] Đề xuất auto-fix (nếu CẦN BỔ SUNG + không có Blocker)
  - [ ] Câu hỏi cho user (nếu có)
```

---

# Các Skills liên quan

- `.agents/skills/create-plan/SKILL.md` — Tạo plan (workflow trước review)
- `.agents/skills/implement-plan/SKILL.md` — Triển khai plan (workflow sau review)
- `.agents/skills/review-implement/SKILL.md` — Review code sau implement

<!-- Upgraded by review-plan v2 — synced with plan_template v2 -->
