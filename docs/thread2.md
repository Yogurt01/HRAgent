# HƯỚNG DẪN CÀI ĐẶT THREAD2

## NOTE: Có nhiều bug lắm nên anh em đọc có gì làm theo không đúng thì AI hoặc hỏi tui nhé

## PREREQUISITE: Vô cái file n8n trong group tải tới cái đoạn docker thôi, sau đó
- Tạo một thư mục tên là n8n-ai (ví dụ ở ổ C:\n8n-ai)
- Trong thư mục n8n-ai, Copy cái "docker-compose.yml" trong cái docs bỏ vô
- Mở Terminal
- cd đường/dẫn/đến/thư-mục/n8n-ai
- chạy: docker compose up -d
- Mở trình duyệt web và truy cập: http://localhost:5678
- Đăng nhập và copy cái thread2.json bỏ vào
## 1. Chuẩn bị API Keys (External Services)

Ông cần đăng ký và lấy Key dán vào mục Credentials của n8n (n8n không lưu key trong file JSON vì lý do bảo mật):

- **Groq API**: Dùng để chạy "não" AI cho Agent 3 & 4  
  *(Model khuyến nghị: Llama-3-70b hoặc Mixtral-8x7b)*

- **Airtable**: Dùng để quản lý số lượng chỉ tiêu (Quota) ( Phần này tui đang update)

- **PostgreSQL**: Database để lưu trữ thông tin ứng viên đã xử lý (Này thì chưa cần nhma làm sẵn)

- **Gmail API (OAuth2)**: Dùng để gửi mail phản hồi tự động

---

## 2. Cấu hình Gmail OAuth2 (Có thể search AI để hiểu)

Vì đây là ứng dụng tự tạo, ông phải thiết lập trên Google Cloud Console:
- Tạo project mới trên link: https://console.cloud.google.com/welcome?project=n8nhragent-491906
- Enable **Gmail API** trong dự án của ông (chỗ cái ô Search)
- **OAuth Consent Screen (cái taskbar bên trái -> API)**: Thiết lập là `External`
- **Test Users (Chỗ này hình như là cái Audience)**:
     Bắt buộc thêm email cá nhân của ông vào  
  *(không có bước này sẽ bị lỗi 403 Access Denied)*

- **Credentials**: Tạo OAuth Client ID (Web application)

- **Redirect URI**:  
  Copy link từ node Gmail trong n8n dán vào Google Cloud  
  *(thường là: http://localhost:5678/rest/oauth2-callback)*

- Khi đăng nhập n8n báo `"App chưa xác minh"`:  
   Nhấn **Advanced → Go to n8n (unsafe)**

---

## 3. Thiết lập cho các AI Agent

Hệ thống sử dụng cấu trúc **Agent - Parser**, ông cần check kỹ cấu hình output:

### Agent 3 (Resume Matcher)
So khớp CV với JD. Node **Structured Output Parser** phải có đủ các trường:

{
  "full_name": "Nguyen Van A",
  "email": "candidate@example.com",
  "fit_score": 85,
  "recommendation": "Pass",
  "top_3_strengths": ["Python", "Deep Learning", "Problem Solving"],
  "top_3_weaknesses": ["Docker", "Communication", "English Speaking"]
}

### Agent 4 (Reviewer)
Rà soát định kiến và soạn mail. Parser phải có:

{
  "bias_check": "Clean",
  "email_content": "Chào bạn, chúng tôi rất ấn tượng với kỹ năng Python của bạn...",
  "recommendation": "Pass"
}

---

## 4. Cấu hình Logic Điều hướng (Switch Node)

Node `Switch1` sẽ điều hướng dựa trên kết quả từ Agent:
- **Data to compare**
{{ $json.output.recommendation }}

- **Nhánh 0 (Fail)**  
  → Gmail (Gửi mail từ chối)  
  → PostgreSQL (Lưu dữ liệu)

- **Nhánh 1 (Pass)**  
  → Airtable (Cập nhật quota)  
  → HTTP Request (Gỡ bài nếu đủ người)

---

### TIP
Nếu node Gmail báo lỗi dữ liệu:

- Vào mục **Expression**
- Kéo trường `email_content` từ Agent 4 vào phần **Message**

---

## 5. Hướng dẫn Test bằng Postmann (Tải cái bản desktop về máy nhé)

Để kích hoạt luồng chạy, ông không thể nhấn nút Play thông thường mà phải gửi request thật vào Webhook:

### Cấu hình:

- **URL**: Lấy từ node Webhook (Production URL hoặc Test URL)
- **Method**: `POST`

### Body:

- Chọn **form-data**
- Key: `data`
- Type: chuyển từ `Text` → `File`
- Value: chọn file CV dạng **PDF**

### Chạy:

- Nhấn **Send**
- Quan sát dữ liệu "chảy" trong n8n

---
