---
alwaysApply: true
description: >
  Quy tắc bắt buộc tham chiếu skill terminal-mastery khi agent cần chạy terminal commands.
  Áp dụng khi agent thực thi lệnh terminal (run_command, command_status, send_command_input).
---

# Terminal Execution Rule

## Yêu cầu BẮT BUỘC

Khi gọi `run_command`, `command_status`, `send_command_input`, hoặc `read_terminal`,
agent **PHẢI** tuân thủ skill `terminal-mastery` (`.agents/skills/terminal-mastery/SKILL.md`):

1. **Phân loại** command theo Decision Tree (Bước 1)
2. **Chọn đúng** Execution Pattern A/B/C/D/E (Bước 2)
3. **Set đúng** `WaitMsBeforeAsync` theo bảng Parameter Reference
4. **Verify** kết quả sau mỗi command (Bước 3)
5. **Escalate** khi gặp giới hạn — KHÔNG tự ý retry vô hạn (Bước 5)

## Hard Constraints

- 🚫 `WaitDurationSeconds` ≥ `5` (KHÔNG BAO GIỜ = 0)
- 🚫 `OutputCharacterCount` ≥ `2000`
- 🚫 `SafeToAutoRun = false` cho DESTRUCTIVE commands — **LUÔN LUÔN**
- 🚫 Max 3 poll cho cùng 1 command

## Khi gặp edge case

Tham chiếu: `.agents/skills/terminal-mastery/resources/edge_cases.md`
