## OpenClaw Web UI - Kiến trúc gọi API (Client - Gateway)

Quá trình khảo sát source code trong `ui/src/ui` cho thấy OpenClaw sử dụng kiến trúc **WebSocket-first**.
Không có các REST API truyền thống (như `GET /chat`, `POST /message`). Thay vào đó:

1. Trình duyệt mở **1 kết nối WebSocket duy nhất** trực tiếp tới Gateway (thường là `ws://127.0.0.1:18789`).
2. Mọi giao tiếp diễn ra dưới dạng bắt tay **JSON-RPC** tùy chỉnh.
3. Khi Client gửi Request:
   ```json
   { "type": "req", "id": "<UUID>", "method": "<Tên_hàm>", "params": { ... } }
   ```
4. Khi Server trả lời Response:
   ```json
   { "type": "res", "id": "<UUID>", "ok": true, "payload": { ... } }
   ```
5. Khi Server đẩy Event (Stream tin nhắn, thông báo):
   ```json
   { "type": "event", "event": "<Tên_sự_kiện>", "payload": { ... } }
   ```

### Các Logic chính (Luồng nghiệp vụ)

#### 1. Bootstrap & Kết nối (Connect)

- Khi giao diện vừa bật, Client nhận sự kiện `connect.challenge` từ Gateway kèm một `nonce`.
- Client tính toán chữ ký số (Device Identity RSA/Ed25519) và gửi method `connect` chứa:
  - `role`: `operator`
  - `scopes`: `["operator.admin", "operator.read", "operator.write", ...]`
  - `device`: Thông tin Public Key và chữ ký số ký cái `nonce`.
  - `auth`: Chứa token truy cập (lấy từ cookie/localStorage hoặc nhập tay).

#### 2. Lấy danh sách Chat (History)

- Gọi method: `chat.history`
- Params: `{ "sessionKey": "<room_id>", "limit": 200 }`
- Logic: Nhận về mảng `messages` gồm role (`user`, `assistant`) và nội dung đã phân mảnh (`text`, `image`).

#### 3. Gửi tin nhắn mới (Send Message)

- Gọi method: `chat.send`
- Params:
  ```json
  {
    "sessionKey": "<room_id>",
    "message": "Nội dung cần hỏi",
    "deliver": false,
    "idempotencyKey": "<UUID_của_lần_chạy>",
    "attachments": [{ "type": "image", "mimeType": "image/jpeg", "content": "<base64_string>" }]
  }
  ```
- Logic sau khi gửi: Giao diện sẽ không chờ response trả về text ngay, mà sẽ lắng nghe các gói tin `{ type: "event", event: "chat.event" }` từ Gateway bắn về (chứa state: `delta`, `final`, `aborted`, `error`) để tạo hiệu ứng streaming text.

#### 4. Các API phụ trợ quan trọng khác

- `models.list`: Lấy danh sách AI Model đang cấu hình.
- `skills.status` / `skills.update`: Bật tắt nhanh các công cụ (Web Search, File System, ...).
- `config.apply`: Thay đổi thiết lập của Gateway trực tiếp từ UI.
- `sessions.subscribe`: Lắng nghe thay đổi danh sách cuộc hội thoại bên sidebar.
