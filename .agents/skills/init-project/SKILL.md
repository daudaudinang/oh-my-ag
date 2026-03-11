---
name: init-project
description: Khởi tạo và cập nhật AGENTS.md với context đầy đủ từ .agent/, .agents/, .cursor/. Dùng khi user chạy /init hoặc yêu cầu "init project/tạo AGENTS.md/cập nhật context cho agent".
---

# Init Project (Workflow cho `/init`)

## Khi nào áp dụng

- User dùng command **`/init`** hoặc yêu cầu tương tự.
- Cần tạo hoặc cập nhật AGENTS.md với context đầy đủ.
- Muốn agent hiểu project structure, commands, code style.

## Mục tiêu

Tạo file `AGENTS.md` (~200 lines) chứa:
1. **Project Structure** - Cấu trúc thư mục
2. **Commands** - Build, lint, test commands
3. **Code Style Guidelines** - Import, formatting, types, naming, error handling
4. **Skills Overview** - Danh sách skills có sẵn
5. **Context Rules** - Tổng hợp từ .agent/, .agents/, .cursor/

## Workflow

### PHASE 1: Thu thập Context

```
1. Đọc AGENTS.md hiện tại (nếu có)
2. Thu thập project info:
   → package.json (scripts)
   → tsconfig.json (paths)
   → eslint, prettier config
3. Thu thập rules:
   → .agents/rules/*.md, *.mdc
   → .cursor/rules/*.md, *.mdc
4. Thu thập skills:
   → .agents/skills/*/SKILL.md
```

### PHASE 2: Tổng hợp

```
1. Tổng hợp commands từ package.json
2. Tổng hợp code style từ config files
3. Tổng hợp rules từ .agents/rules/ và .cursor/rules/
4. Tổng hợp skills từ .agents/skills/
```

### PHASE 3: Tạo AGENTS.md

Viết file AGENTS.md theo template bên dưới.

---

## Template AGENTS.md (~200 lines)

```markdown
# AGENTS.md - Agentic Coding Guidelines

Multi-Merchant B2B Marketplace (Medusa v2) codebase.

---

## 1. Project Structure

```
b2b-starter-medusa/
├── backend/          # Medusa v2 server (Node.js/TypeScript)
├── storefront/      # Next.js 15 storefront (React 19)
├── merchant-admin/ # Vite + React admin (port 7001)
└── super-admin/     # Vite + React super admin (port 7002)
```

---

## 2. Commands

### Backend

| Command | Description |
|---------|-------------|
| `yarn dev` | Start Medusa development server |
| `yarn build` | Build for production |
| `yarn test:unit` | Run unit tests |
| `yarn test:unit --testPathPattern="filename.test"` | Run single test |

### Storefront

| Command | Description |
|---------|-------------|
| `yarn dev` | Start dev (port 8000) |
| `yarn lint` | ESLint + Prettier check |

### Admin (Merchant & Super)

| Command | Description |
|---------|-------------|
| `yarn dev` | Start Vite dev server |
| `yarn test filename.test.ts` | Run single Vitest test |

---

## 3. Code Style Guidelines

### TypeScript

- **Always use TypeScript** - No plain JavaScript except config files
- **Explicit types** for function parameters and return types
- **Avoid `any`** - Use `unknown` or proper generics
- **Use interfaces** for objects, types for unions/aliases

```typescript
// Good
interface User {
  id: string
  name: string
  role: 'admin' | 'user'
}

function getUser(id: string): Promise<User> {
  return db.user.findUnique({ where: { id } })
}
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Files | kebab-case | `user-service.ts` |
| Components | PascalCase | `UserProfile.tsx` |
| Hooks | camelCase with `use` | `useAuth()` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |

### Imports

- **Absolute imports** using `@/` alias
- **Group imports**: external → internal → relative
- **Avoid barrel files** (`index.ts`) - import directly

```typescript
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { UserService } from '@/services/user'
import { Order } from '../types'
```

### React Patterns

- Use functional components with explicit props typing
- Use `memo()` for expensive components
- Use functional updates: `setItems(prev => [...prev, newItem])`

### Error Handling

- Use try-catch for async operations
- Create custom error classes for domain errors
- Return error results rather than throwing in non-critical paths

```typescript
class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`)
    this.name = 'NotFoundError'
  }
}
```

### Formatting

- 2 spaces indentation
- Semicolons at end of statements
- Single quotes for strings
- Trailing commas in multi-line objects/arrays

---

## 4. Key Architectural Decisions

1. **Single DB, single Medusa instance** - merchants share infrastructure
2. **Role source of truth**: `provider_identity.user_metadata.role`
3. **merchant_admin** scope: access only their merchant's data
4. **Product handles** globally unique (stored as namespaced)
5. **merchant_admin accessing wrong merchant** → return 404 (not 403)

---

## 5. Skills

| Skill | Description | Trigger |
|-------|-------------|---------|
| create-plan | Tạo plan triển khai | `/create-plan` |
| implement-plan | Triển khai plan | `/implement-plan` |
| implement-review | Review implementation | `/review-implement` |
| review-plan | Review plan | `/review-plan` |
| create-skill | Tạo skill mới | `/create-skill` |
| sync-agents-context | Sync context vào AGENTS.md | `/sync-agents-context` |

> 💡 **Tip**: Run `/sync-agents-context` to update this file with latest rules from `.agent/`, `.agents/`, `.cursor/`.

---

## 6. Rules & Guidelines

### Project Rules (.agents/rules/)

- **context-rule.md**: Multi-Merchant context, source of truth
- **responses-rules.md**: Response formatting rules
- **doc-rule.md**: Documentation rules
- **agentic-rule.md**: Agentic workflow rules

### Cursor Rules (.cursor/rules/)

- Tương tự .agents/rules/

---

## 7. Response Language

- **Respond in Vietnamese** for discussions with users
- Write code/documentation in Vietnamese unless specified
- Use English for technical terms
```

---

## Checklist /init

```markdown
## Checklist /init

- [ ] **PHASE 1: THU THẬP**
  - [ ] Đọc AGENTS.md hiện tại
  - [ ] Thu thập commands từ package.json
  - [ ] Thu thập code style từ config
  - [ ] Thu thập rules từ .agents/, .cursor/

- [ ] **PHASE 2: TỔNG HỢP**
  - [ ] Tổng hợp commands
  - [ ] Tổng hợp code style
  - [ ] Tổng hợp skills
  - [ ] Tổng hợp rules

- [ ] **PHASE 3: TẠO AGENTS.md**
  - [ ] Viết ~200 lines
  - [ ] Bao gồm sync-agents-context tip
  - [ ] Verify nội dung
```

---

## Gợi ý sử dụng

Sau khi tạo AGENTS.md, khuyến khích user chạy:

```
/sync-agents-context
```

Để cập nhật thêm các context rules từ `.agent/`, `.agents/`, `.cursor/`.
