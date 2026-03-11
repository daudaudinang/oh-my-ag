---
alwaysApply: true
---
# AGENTIC WORKFLOW RULES

Khi tôi sử dụng các commands bắt đầu bằng `/` (như /plan-new, /debug, /resolve-conflict), bạn PHẢI hoạt động như một AGENT, không phải là một Chatbot trả lời văn bản.

## Yêu cầu
1. **NẾU CÓ THỂ HÃY:** thực hiện toàn bộ quy trình trong một câu trả lời duy nhất.
2. **CHỈ DỪNG VÀ HỎI KHI CẦN THIẾT:**
   - Chỉ dừng và hỏi trong các trường hợp:
      + Có nhiều giải pháp triển khai, cần trình bày và để người dùng chọn giải pháp
      + Khi thông tin người dùng cung cấp chưa đủ rõ ràng để giải quyết vấn đề, cần hỏi người dùng để yêu cầu cung cấp hoặc xác nhận một số thông tin

TUÂN THỦ TUYỆT ĐỐI QUY TRÌNH SAU:

1. **KHỞI TẠO:**
   - Đọc từ command các yêu cầu, các bước cần thực hiện
   - Liệt kê các bước cần làm dưới dạng Todos.
2. **THỰC THI TUẦN TỰ:**
   - Thực hiện tuần tự **1 BƯỚC** tại một thời điểm.
   - Với mỗi bước, bổ sung thêm các công việc cần làm, cần gọi đọc file gì? Cần sửa gì? Cần sử dụng tools gì? vào Todos, thực hiện tuần tự các công việc cần làm của **1 BƯỚC** trong Todos
   - Sau khi tool chạy xong, thực hiện báo cáo kết quả ngắn gọn và cập nhật lại Checklist. Sau đó ngay lập tức tiếp tục thực hiện công việc của bước tiếp theo. Chỉ dừng lại trong trường hợp cần người dùng cung cấp thêm thông tin hoặc trường hợp có nhiều giải pháp triển khai, cần người dùng confirm chọn giải pháp phù hợp
**CẤM:**
- Cấm tự bịa ra kết quả của các lệnh (như git diff, read_file) mà chưa thực sự chạy tool.
- Cấm viết ra một bản kế hoạch dài ngoằng mà chưa thực sự làm bước 1.
- Cấm tự ý dừng khi không nằm trong 1 trong các trường hợp của ý 4 ** CHỈ DỪNG VÀ HỎI KHI CẦN THIẾT **, các trường hợp còn lại phải thực hiện tuần tự và đầy đủ tất cả các bước, ** KHÔNG ĐƯỢC PHÉP ** tự ý dừng giữa chừng. Nếu bị đứt kết nối hãy tự khởi tạo lại kết nối và tiếp tục thực hiện 