---
trigger: model_decision
description: Context triển khai Multi‑Merchant Marketplace (source of truth, quyết định chốt, mục lục tài liệu). Sử dụng context rule này khi người dùng yêu cầu tạo plan hoặc triển khai epic/task hoặc review plan hoặc review phần code đã tạo trước đó
---

# Context Rule — Multi‑Merchant B2B Marketplace (Medusa v2, B2B Starter)

Mục tiêu của rule này: giúp LLM **nắm nhanh bối cảnh + quyết định nghiệp vụ đã chốt + vị trí tài liệu thiết kế** để triển khai từng epic mà **không phải nhắc lại toàn bộ plan**.

## Source of truth (ưu tiên đọc theo thứ tự)

- **Chốt nghiệp vụ (contract/decision + edge cases)**: `.agent/note_flow/CHOT_NGHIEP_VU_MULTI_MERCHANT_MARKETPLACE.md`
- **Implementation plan (scope/phase + mapping + hướng triển khai)**: `.agent/note_flow/PLAN_MULTI_MERCHANT_MARKETPLACE.md`

Nguyên tắc: khi có mâu thuẫn, **CHOT** là “đã chốt để implement”, PLAN là “định hướng triển khai & breakdown”.

## Danh mục tài liệu thiết kế trong repo (để tra nhanh)

- **C4**:
  - `.agent/note_flow/C4_SYSTEM_CONTEXT_MULTI_MERCHANT_MARKETPLACE.md`
  - `.agent/note_flow/C4_CONTAINER_DIAGRAM_MULTI_MERCHANT_MARKETPLACE.md`
  - `.agent/note_flow/C4_COMPONENT_BACKEND_MODULES_MULTI_MERCHANT_MARKETPLACE.md`
- **ADR**: `.agent/note_flow/ADR_MULTI_MERCHANT_MARKETPLACE.md`
- **NFR**: `.agent/note_flow/NFR_MULTI_MERCHANT_MARKETPLACE.md`
- **ERD/DBML**:
  - `.agent/note_flow/ERD_TO_BE_MULTI_MERCHANT_MARKETPLACE.md`
  - `.agent/note_flow/ERD_TO_BE_MULTI_MERCHANT_MARKETPLACE.dbml`
- **API specs**: `.agent/note_flow/api_specs/`
- **DB specs**: `.agent/note_flow/db_specs/`
- **Flow specs**: `.agent/note_flow/flow_specs/`
- **Inventory API (đã đặc tả riêng)**: `.agent/note_flow/API_INVENTORY.md`

## Bối cảnh hệ thống & mapping thuật ngữ (để tránh hiểu sai)

- **Super Admin Portal**: **chưa có** trong repo (sẽ là Next.js codebase riêng). Super Admin gọi **Medusa Admin API** với role `super_admin`.
- **Merchant Admin Portal**: chính là **Medusa Admin hiện tại** (Admin UI) nhưng sẽ bị **scope quyền** theo role `merchant_admin` (actor type `user`).
- **Buyer/Customer Admin**: “Company Admin” hiện tại (actor type `customer`) với role `company_admin` (đang dùng pattern metadata).

## Quyết định chốt quan trọng (MVP)

### 1) Kiến trúc & dữ liệu

- **1 DB chung**, **1 Medusa instance**.
- **Global (shared)**: Customers, Companies, Marketplace Categories, Regions/Tax Regions/Store settings, các cấu hình ops (sales channels/stock locations/shipping...).
- **Merchant‑scoped**: merchant profile/certificates, products của merchant, merchant collections (custom), carts theo merchant (MVP), data hiển thị trong merchant admin (scoped).

### 2) Roles/Permission contract (để implement guard đúng)

- **Bảng user dùng chung**: dùng core `user` của Medusa.
- **Role source of truth** (đã chốt): `provider_identity.user_metadata.role` (provider `emailpass`).
- **Cách đọc role** (đã chốt): từ `req.auth_context.auth_identity_id` → query `provider_identity` theo `auth_identity_id`.
- **Super admin**: role `super_admin`, **không thuộc merchant nào** (không có merchant link).
- **Merchant admin**: role `merchant_admin`, thuộc **đúng 1 merchant**.

### 3) Invariants bám DB/Medusa hiện tại (đừng “đoán” khác đi)

- **`product.handle` unique global** (soft-delete aware) → merchants **không** thể trùng handle.
- **`product.status` đã có `proposed`** (draft/proposed/published/rejected) → dùng cho “Immediate hide khi pending”.

### 4) Categories & Collections

- **Marketplace Categories (global)**: Super Admin CRUD; merchant chỉ assign; hỗ trợ hierarchy; avatar/symbol lưu `metadata`.
- **Merchant Collections (custom)**: dùng module `CollectionGroup` (tree n cấp, `rank` reorder); **many-to-many** product ↔ collection groups.
- **Không dùng Medusa core collections** cho merchant (core: 1 product chỉ thuộc 1 collection).

### 5) Checkout (MVP Option B)

- **Multi-cart**: 1 customer có N carts (mỗi merchant 1 cart).
- **Checkout**: checkout **từng cart** (N carts → N lần checkout → N orders). Unified checkout là **Post‑MVP**.
- **Currency (MVP)**: 1 currency chung toàn marketplace.

### 6) Product Approval (MVP Option B: Immediate hide khi update)

- Update product đã published:
  - tạo `ProductApproval(action=update, pending_data snapshot full payload)`
  - set `product.status = proposed` để ẩn storefront
  - approve: apply pending_data + publish
  - reject: discard pending_data + publish version cũ
- Inventory updates **bypass approval** (không đổi sang proposed).
- Edge case quan trọng (đã chốt): **1 pending approval/product**, resubmit sẽ **update cùng ticket** (overwrite pending_data).

### 7) “Related Customers/Companies” trong Merchant Admin (đã chốt)

- “Related” **chỉ tính khi có Order** (không tính Cart).

### 8) Response strategy khi scope mismatch (đã chốt)

- Nếu `merchant_admin` truy cập resource không thuộc mình → **trả 404** (không leak existence), không trả 403.

## Edge cases (MVP) — bắt buộc tuân theo

Danh sách EC‑1..EC‑12 nằm ở:
` .agent/note_flow/CHOT_NGHIEP_VU_MULTI_MERCHANT_MARKETPLACE.md ` (Section 3.1)

## Cách dùng rule này khi triển khai từng epic

- Luôn đọc **CHOT** trước để lấy “contract”, sau đó dùng **PLAN** để biết scope/phase & mapping.
- Khi cần chi tiết:
  - Endpoint/response/request → tra `.agent/note_flow/api_specs/`
  - Schema/migration/constraints → tra `.agent/note_flow/db_specs/` + ERD/DBML
  - Luồng nghiệp vụ/sequence → tra `.agent/note_flow/flow_specs/`
- Nếu gặp điểm chưa đủ rõ để implement, **phải hỏi lại** (không tự suy diễn) và đề xuất scenario test thủ công.
- Không tự ý thêm logging nếu chưa được yêu cầu rõ; tuyệt đối không log secrets/credentials/token/env/db details.
