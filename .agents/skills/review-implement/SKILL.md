---
name: implement-review
description: |
  Review code đã triển khai theo workflow 4 pha. Đóng vai trò Senior Developer để đánh giá
  tính đúng đắn, Execution Boundary, AC Verification, ảnh hưởng và rủi ro của code.
  Kích hoạt khi user chạy `/review-implement`, hoặc yêu cầu "review implementation",
  "review code đã implement", "kiểm tra code triển khai", "code này ổn chưa".
---

# Goal

Đóng vai **Senior Developer (Lead)** review code đã triển khai. Trọng tâm là bắt vi phạm Execution Boundary (Guardrails), đối chiếu bằng chứng Verify AC, phát hiện lỗ hổng logic/architecture. Đưa ra report xếp theo severity cùng Auto-fix suggestions.

---

# Instructions

## Bước 0: Khởi tạo & Định vị Scope

### 0.1. Triage Depth & Nạp Plan
1. Nhận path từ `/review-implement <plan_file_path>`. Kiểm tra xem file plan tồn tại không.
2. Đọc file plan bằng `view_file` -> Xác định **Tier của Plan (S/M/L)**.
3. Trích xuất ngay & Ghi nhớ 2 Guardrails:
   - **Allowed Files:** Nơi được phép sửa.
   - **Do NOT Modify:** Nơi CẤM sửa.
4. Xác định thư mục Walkthrough (VD: `.agents/plans/<normalized_name>/`).

### 0.2. Nạp Project Context
> **Hook tự động:** Nếu project có `.agents/skills/load-project-context/SKILL.md`, đọc và nạp context theo tầng kiến trúc. Nếu không có thì bỏ qua.

### 0.3. Context Sharding (đối với Tier L)
- Nếu là plan Tier L có nhiều Phase chia ra các Story files, **KHÔNG** đọc mọi story file 1 lúc. Trích xuất list file changed tổng → rẽ nhánh đọc Story file tương ứng lúc soi code.

---

## PHASE 1: 📥 Thu thập Bằng chứng & Check Guardrails

1. **Tìm & đọc Walkthrough files:** Đọc tất cả `phase-*-walkthrough.md`.
2. **Tổng hợp:**
   - Files Changed hiện tại.
   - Trạng thái **Execution Boundary Check** từ walkthrough.
   - Trạng thái **AC Verification** từ bảng Walkthrough.
3. **Execution Boundary Check (Crucial):**
   - Đối chiếu danh sách `Files Changed` thực tế vs `Do NOT Modify`.
   - Nếu vi phạm → Đánh dấu 🔴 **Critical: Vi phạm Boundary**.
4. **Chuẩn bị review:** Dùng `view_file` xem nội dung HIỆN TẠI của các `Files Changed`.

---

## PHASE 1.5: 📝 Plan v3 Compliance Check

Nếu plan dùng template v3, kiểm tra thêm:

| Check | Cách verify | Severity nếu thiếu |
|-------|------------|--------------------|
| **Implementation Order** | Code changes có match dependency order trong plan? | 🟡 Minor |
| **Risk Matrix** | Mỗi risk High×High có mitigation code tương ứng? | 🟠 Major |
| **Pre-flight Check** | Walkthrough ghi nhận đã chạy Pre-flight? | 🟡 Minor |
| **Change Log** | Plan version khớp với version đã implement? | 🔵 Info |

> Nếu plan không có sections này → ghi N/A, không phạt.

---

## PHASE 2: 🔍 Review Code & Logic (Deep Dive)

Review từng file dựa trên 5 bộ tiêu chí nằm trong thư mục resources (`review_criteria.md`).
*(Lưu ý: Tuỳ thuộc vào Tier S -> Review nhanh, Tier M/L -> Deep dive).*

### Weighted Scoring

| # | Tiêu chí | Hệ số |
|---|---------|-------|
| 0 | Boundary Compliance & Guardrails | ×4 |
| 1 | Traceability & AC | ×3 |
| 2 | Correctness | ×3 |
| 3 | Robustness | ×2 |
| 4 | Clarity & Style | ×1 |

```text
Max: (10×4) + (10×3) + (10×3) + (10×2) + (10×1) = 130 pts

Thresholds:
- ≥ 117 (90%) → ✅ APPROVED
- 78–116 (60–89%) → ⚠️ APPROVED WITH NOTES
- < 78 (<60%) → ❌ CHANGES REQUESTED

Override:
- Boundary ≤ 5/10 → auto ❌ CHANGES REQUESTED
- Bất kỳ Critical issue → auto ❌ CHANGES REQUESTED
```

### 2.1. Traceability & AC (Bám sát Plan)
- Đảm bảo mọi Requirement trong Plan có code xử lý.
- Mọi AC đều phải có bằng chứng từ Bước 1. Nếu walkthrough không có test command → 🟠 Major/Minor.
- Check Scope Creep (Implement thừa).

### 2.2. Correctness (Tính đúng đắn)
- Trace flow logic, edge cases, async handles, missing awaits, race conditions.

### 2.3. Robustness (Tính bền vững)
- Check Error handling, bounds checking, input validations, null safeties.

### 2.4. Clarity & Style
- Naming convention, code length, hardcoded magic numbers/strings, duplicate codes.

---

## PHASE 3: ⚡ Impact & Risk Analysis

Đặt câu hỏi "Senior":
1. **Regression:** Có sửa util/base component hay interface dùng chung không? Có khả năng gãy ở module không liên quan hông?
2. **Integration:** API Contract thay đổi nhưng consumer chưa đổi?
3. **Data:** Sửa Schema có migration chưa? Backward map ổn không?
4. **Performance:** N+1 Query? Heavy renders? Vòng lặp lồng?

---

## PHASE 4: 📝 Report & Auto-fix

1. Phân loại Severity (xem `severity_levels.md`):
   - 🔴 **Critical:** Viết boundary, Lỗ hổng bảo mật, Crash. (PHẢI fix trước merge).
   - 🟠 **Major:** Logic fail, missing validation, ko handle error.
   - 🟡 **Minor:** Naming bad, typo, scope creep nhẹ.
   - 🔵 **Info:** Best practice refactoring.

2. **Auto-fix / Quick fix Generation:** 
   - Với các lỗi **Minor** hoặc Info (ví dụ: đổi tên hàm, thêm export, fix magic numbers), CUNG CẤP NGAY ĐOẠN CODE SNIPPET (Patch) trong Recommendation của Report để user copy-paste fix lẹ.

3. **Xuất Report:** Sử dụng đúng format `report_template.md`. Báo cáo cụ thể Boundary Check & AC Verification.

4. **Kết luận:**
   - ✅ **APPROVED**
   - ⚠️ **APPROVED WITH NOTES**
   - ❌ **CHANGES REQUESTED**

---

## 🔁 Re-review Protocol (khi user fix xong)

Nếu user yêu cầu re-review sau khi fix:

1. **ĐỌC report cũ** → trích xuất danh sách issues CHƯA fix.
2. **CHỈ review lại files có thay đổi** (`git diff` hoặc user chỉ ra).
3. **Verify từng issue đã fix**:
   - ✅ Fixed → đánh dấu resolved
   - ❌ Vẫn sai / fix sai → giữ nguyên severity
   - 🆕 New issue phát sinh → thêm vào report
4. **Cập nhật weighted score** dựa trên điểm mới.
5. **Output:** "Re-review Report" (compact, chỉ delta changes).

> ⚠️ KHÔNG chạy lại toàn bộ Phase 0→4 nếu chỉ cần verify fix.

---

# Constraints (Quy tắc "Senior")

- ✅ LUÔN check danh sách **Do NOT Modify** trước khi đào sâu vào logic thuật toán. Phá rào là lỗi nặng nhất.
- ✅ Bắt lỗi phải dùng `file path : line number` cụ thể. Không vu vơ.
- 🚫 KHÔNG tự tiện ghi đè file code (CHỈ là reviewer, output là report markdown). Auto-fix chỉ nằm trong Report.
- 🚫 Không Hallucination: Chỉ review trên file thật đọc qua `view_file`. Lỗi tool -> Báo User, không được bịa logic.

# Checklist Review Implement (tự kiểm)

```markdown
- [ ] Xong Bước 0: Triage Depth (S/M/L) & Nạp Plan. Ghi nhận `Do NOT Modify`.
- [ ] Xong Bước 1: Thu thập Files Changed & Verify Boundary + AC từ walkthrough.
- [ ] Xong Bước 1.5: Plan v3 Compliance (Impl Order, Risk Matrix, Pre-flight, Change Log).
- [ ] Xong Bước 2: Review Code qua 5 tiêu chí + chấm weighted score.
- [ ] Xong Bước 3: Phân tích Impact & Risks (Regression, Performance).
- [ ] Xong Bước 4: Viết Report kèm Auto-fix + weighted score (Template + Severity check).
- [ ] (Nếu re-review): Chỉ verify delta files + issues đã fix.
```

<!-- Generated by Skill Generator v3.2 / Updated BMAD -->
