# Tổng hợp tài liệu và Quy tắc làm việc dự án OpenClaw

## 1. Tổng quan dự án (Project Overview)

**OpenClaw** là một trợ lý AI cá nhân (Personal AI Assistant) chạy trên quy mô local device, kết nối qua nhiều kênh như WhatsApp, Telegram, Slack, Discord, v.v., và có các ứng dụng đi kèm (macOS, iOS, Android).
Sứ mệnh của OpenClaw (`VISION.md`): Là một trợ lý AI "thực sự làm được việc" thông qua các máy trạm cá nhân với các quy tắc do người dùng thiết lập, song song với việc ưu tiên bảo mật và UX khởi tạo mượt mà.

## 2. Kiến trúc cốt lõi (Core Architecture)

- **Gateway (Control Plane - ws://127.0.0.1:18789)**: Trung tâm điều khiển phiên bản WebSocket. Nó quản lý mọi luồng message, sessions, config, cron, event và hooks.
- **Node Clients / Apps**:
  - macOS app (Control plane ở menu bar, Voice Wake, WebChat...).
  - Máy trạm Remote Linux.
  - iOS/Android app kết nối làm WebSocket node.
- **Agent Loop (Pi Agent)**: Xử lý inference dạng RPC, quản lý bộ nhớ (LanceDB/JSON) và công cụ.
- **Tools**: CLI (Onboard, Doctor, Gateway, Send, v.v...), Trình duyệt (Chrome/Chromium qua CDP), Media/Audio, Canvas (A2UI).
- **Plugins/Extensions**: Hệ sinh thái mở rộng qua chuẩn NPM, dùng JITI ảo hóa import.

## 2.1 Chi tiết Agent Loop & Sandbox Logic

- **Agent Loop (`src/agents/pi-embedded-runner`)**: Logic chính `runEmbeddedPiAgent` thực thi thuật toán agent. Gateway quản lý luồng bằng cách đưa vào queue (`queueEmbeddedPiMessage`) và đợi kết thúc (`waitForEmbeddedPiRunEnd`). Hỗ trợ compact message (rút gọn context khi quá dài) bằng `compactEmbeddedPiSession`.
- **Sandbox (`src/agents/sandbox`)**: Bảo vệ hệ thống khỏi mã độc hoặc câu lệnh không mong muốn sinh ra do AI. Nhờ cấu trúc module hóa, nó hỗ trợ nhiều backend thực thi như: Docker (local container mặc định), SSH (Remote), Browser (cho công cụ duyệt web).
- **System Run Approval (`src/gateway/node-invoke-system-run-approval.ts`)**: Cổng an ninh quyết định việc agent có được gọi lệnh hệ thống trên node hay không thông qua kiểm tra approval records và danh tính thiết bị yêu cầu (phòng tránh bypass device ID mismatch). DMs luôn được coi là untrusted.

## 2.2 Định tuyến (Routing) và Plugins

- **Routing (`src/routing/resolve-route.ts`)**: Xác định Agent IDs để định tuyến message xử lý qua hàng loạt các cấp bậc ưu tiên (Peer/Thread > Guild & Roles > Guild > Team > Account > Channel). Nhờ "thread inheritance", các message có khả năng kế thừa agent của cha.
- **Tính đóng gói của Extensions/Plugins (`extensions/*`)**: Mọi nền tảng nhắn tin ngoài cốt lõi (như Discord, Slack, WhatsApp, Signal) đều được thiết kế dưới dạng npm package riêng trong `extensions/*`. Chúng bắt buộc phải tích hợp với Core bằng Interface ảo `openclaw/plugin-sdk/*` và nghiêm cấm import trực tiếp code production từ `src/` nhằm đảm bảo tính toàn vẹn (Integrity) theo quy chuẩn của dự án.

## 3. Cấu trúc thư mục định hướng

- Nguồn code (Core): `src/` (bao gồm CLI ở `src/cli`, logic ở `src/commands`, provider ở `src/provider-web.ts`, logic hạ tầng ở `src/infra`, và pipeline xử lý truyền thông ở `src/media`).
- Test: Nằm song song (`colocated`) với source dưới định dạng `*.test.ts`.
- Document: `docs/`.
- Các Plugins/Extensions: Hệ thống này gọi là plugin tại UI và docs, nhưng trong code vẫn duy trì folder `extensions/*` (với định dạng `@openclaw/<id>`).
- Tool/Script Dev: `scripts/`. App platform: `apps/`.
- Public Extension API: Chỉ giao tiếp qua `openclaw/plugin-sdk/*` và các barrel `api.ts`. Tránh truy cập gián tiếp vào `src/`.

## 4. Các quy tắc Code & Phát triển (Development Rules)

- **Ngôn ngữ**: TypeScript thuần ESM. Chạy với Node 22+ (khuyên dùng Node 24) qua hệ thống `bun` hoặc `pnpm`.
- **Typing & Linting**: Bắt buộc tuân thủ typing nghiêm ngặt, không đùng `any` và không xài `@ts-nocheck`. Linting bằng **Oxlint** (`pnpm check / lint`) và Formatting bằng **Oxfmt** (`pnpm format`).
- **Giới hạn module**: KHÔNG import code production song song dạng `await import` với static `import`. Cần dùng `*.runtime.ts` boundary.
- **Extension rules**: Bên trong `extensions/<id>/**`, KHÔNG xài relative import chọc ra ngoài thư mục gốc extensions. KHÔNG tự truy cập vào thư mục của extension khác ngoại trừ việc đi qua public interface.
- **File size LOC**: Giữ số dòng trên mỗi module ở mức dưới 700 LOC khi khả thi.
- **Class Methods**: Dùng Inheritance/Composition, KHÔNG chọc hoặc vá `Prototype` (v.d `*.prototype.method = ...`).

## 5. Quy trình Kiểm thử & Xử lý lỗi (Testing & CI)

- **Framework**: Vitest, kèm V8 cho coverage (yêu cầu >= 70%).
- **Cấu trúc test**: Code test và e2e test phải được isolate state hoàn toàn (`--isolate=false` vẫn pass). Dùng stub thay vì prototype injection.
- Trước khi push lên nhánh `main`, cần chạy `pnpm check` và `pnpm test`. Nếu động đến cấu trúc build, đặc biệt cần đảm bảo `pnpm build` không fail.
- Các PR dính dáng đến cấu hình (Schema drift), dùng `pnpm config:docs:check` để check trước.

## 6. Bảo mật & Deployment

- Đăng nhập và mã hóa: Web provider tự lưu credential ở `~/.openclaw/credentials/`. Sessions sinh ra tại `~/.openclaw/sessions/` không thể thay đổi vị trí.
- Channel Policy Security: Bất kỳ tin nhắn DM (Direct Message) đều mặc định là untrusted. Hệ thống bắt buộc cần Pairing code hoặc whitelist policy (`dmPolicy="pairing"`). Sandbox Docker nên được cài làm default để thực thi session lệnh.
- Không public thông tin nhạy cảm vào CI/logs/docs.

## 7. Các tiện ích lệnh hữu dụng

- `pnpm install` / `pnpm build`: Cài đặt và biên dịch dự án.
- `pnpm format` và `pnpm format:fix`: Quản lý tiêu chuẩn fomat oxfmt.
- `pnpm dev` hoặc chạy cli bun rút gọn: `pnpm openclaw gateway --config...`
- Kiểm tra sức khỏe gateway: `openclaw doctor` / `pnpm openclaw doctor`.
- Tái tạo UI: `pnpm ui:build` và `pnpm canvas:a2ui:bundle`.
