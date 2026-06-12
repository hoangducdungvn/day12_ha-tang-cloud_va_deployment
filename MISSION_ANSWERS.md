# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. Hardcoded API Keys và Mật khẩu trực tiếp trong code (vi phạm 12-factor app).
2. Lưu state (trạng thái ứng dụng) vào bộ nhớ RAM (In-memory) khiến hệ thống không thể nhân bản (Scale ngang) mà không làm mất phiên làm việc của user.
3. Chạy Server bằng script phát triển (`python app.py`) thay vì dùng Uvicorn/Gunicorn với số lượng worker đa luồng tiêu chuẩn.
4. Log xuất ra dạng text thuần (Unstructured), gây khó khăn cho việc truy vết và gom nhóm log trên môi trường Cloud.

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config  | Hardcoded trong code | Lấy từ biến môi trường (.env) | Tuyệt đối bảo mật, không rò rỉ mã bí mật lên GitHub và dễ dàng thay đổi cấu hình cho từng server. |
| State | Lưu trong RAM máy chủ | Stateless (Lưu chung ở Redis) | Bất kỳ máy chủ Agent nào (1, 2 hay 3) cũng có thể đọc Redis và trả lời tiếp cho user liền mạch. |
| Logging | Văn bản text thuần | Định dạng JSON (Structured) | Giúp các hệ thống quản lý Log (Datadog, Kibana) dễ truy vấn, phân tích lỗi tự động. |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: Dùng `python:3.11-slim` cho cả tầng Builder và Runtime để tối ưu dung lượng và bảo mật.
2. Working directory: Dùng `/build` để tập trung biên dịch thư viện, và `/app` làm nhà ở chính thức cho ứng dụng lúc chạy.

### Exercise 2.3: Image size comparison
- Develop: ~900 MB (Ảnh base Python đầy đủ hoặc bị ám ảnh bởi file GCC Build Tools).
- Production: < 500 MB (Chỉ lấy kết quả build, bỏ qua các công cụ compiler nhờ Multi-stage).
- Difference: Tiết kiệm được ~50% -> 80% dung lượng lưu trữ và băng thông truyền tải.

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: `https://ai-agent-production-98pj.onrender.com` *(Tôi đã dùng Render làm platform thay thế do Railway hết hạn Trial)*.
- Screenshot: Xem chi tiết các dấu tick xanh trong log Console của tài khoản Render.

## Part 4: API Security

### Exercise 4.1-4.3: Test results
- Không có khóa API Key / Khóa sai: Trả về HTTP 401 Unauthorized `{"detail":"Invalid API key."}`.
- Có khóa hợp lệ: Trả về HTTP 200 OK với đầy đủ JSON.
- Rate Limiting Test: Khi Spam 20 request liên tiếp bằng bash script vòng lặp, từ request thứ 11 trở đi hệ thống bắt đầu chửi lỗi `HTTP 429 Too Many Requests`.

### Exercise 4.4: Cost guard implementation
- Cost Guard được chèn như một màng lọc cuối cùng sau bước Validate.
- Thuật toán sẽ đếm số Input Tokens và Output Tokens thực tế, nhân với đơn giá (VD: \$0.00015/1K token của OpenAI) để cộng dồn vào hóa đơn ngày `_daily_cost`.
- Nếu hóa đơn vượt qua ngân sách `$1.0/ngày`, toàn bộ yêu cầu bị đóng sập cửa với mã lỗi `HTTP 503` (hoặc 402) để ngăn AI Agent "đốt" sạch tài khoản ngân hàng của Dev.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- **Graceful Shutdown**: Hệ thống bắt giữ tín hiệu SIGTERM từ Hệ điều hành (khi yêu cầu Tắt/Update). Lập tức đổi trạng thái đường dẫn `/ready` thành `503` để báo Load Balancer (Nginx) ngưng chuyển khách mới vào. Đồng thời, hàm `lifespan` kiên nhẫn đếm ngược chờ đến khi số lượng `_in_flight_requests == 0` (làm xong hết khách cũ) thì mới yên tâm tự sát (`Exit 0`).
- **Nginx & Redis Stack**: Cấu hình Docker Compose với `scale agent=3`. Nginx chia bài lần lượt cho 3 Agent (Round-robin). Khi test `test_stateless.py`, mỗi câu hỏi của mình rơi vào 1 instance ID khác nhau nhưng lịch sử chat vẫn được ghi nhớ nhờ cuốn sổ chung Redis.
