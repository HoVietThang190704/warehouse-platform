# Realtime Inventory Socket Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
> **USER CONSTRAINT:** Không được commit và không được push Git. Chỉ chỉnh sửa/tạo file cục bộ để user tự kiểm tra code.

**Goal:** Triển khai realtime Socket.IO giữa API Gateway và Next.js Web để khi bất kỳ ai nhập/xuất kho thành công, toàn bộ màn hình liên quan của người khác tự động cập nhật số liệu tồn kho, KPI và chứng từ ngay lập tức.

**Architecture:** 
- `apps/api-gateway`: Đăng ký `SocketModule` / `InventoryEventsGateway` (NestJS WebSockets + Socket.IO) trên cùng HTTP server cổng 3000. Cung cấp API cấp `socket ticket` (1-2 phút) cho người dùng đã đăng nhập và endpoint nội bộ nhận sự kiện từ `inventory-service` để broadcast theo room kho (`warehouse:<warehouseId>`).
- `apps/inventory-service`: Sau khi giao dịch nhập/xuất kho hoàn tất thành công trong database, gọi HTTP nội bộ sang API Gateway để phát tín hiệu `inventory:changed`.
- `apps/web`: Thêm `SocketProvider` và hook lắng nghe sự kiện `inventory:changed`. Khi nhận sự kiện, tự động gọi `queryClient.invalidateQueries({ queryKey: inventoryKeys.all })` để cập nhật bảng tồn, KPI và sổ chứng từ.

**Tech Stack:** NestJS 11, `@nestjs/websockets`, `@nestjs/platform-socket.io`, `socket.io`, `socket.io-client`, Next.js 15, TanStack Query v5.

**Spec:** Đã thống nhất trực tiếp trong hội thoại kỹ thuật (không đổi cơ chế REST/Prisma, socket chỉ làm nhiệm vụ invalidation signal).

## Global Constraints
- Không chạy bất kỳ lệnh `git commit` hay `git push` nào.
- Giữ nguyên cơ chế bảo mật HttpOnly cookie của Web và API.
- Không đưa Access Token dài hạn lên query string của WebSocket URL; dùng ticket ngắn hạn hoặc handshake auth.
- Socket không mang số dư tính toán sẵn; client luôn lấy số dư chính xác từ API REST.

## Review Focus
1. Client mất kết nối rồi nối lại có tự refetch dữ liệu mới nhất không?
2. Nhập/xuất thất bại (lỗi validation, xuất âm) có đảm bảo KHÔNG phát socket không?
3. Khi người dùng đổi kho được chọn trên web, socket có chuyển room tương ứng không?
4. Đăng xuất hoặc hết phiên có dọn dẹp sạch socket connection không?
5. Giao tiếp nội bộ giữa inventory-service và api-gateway có được bảo vệ bằng secret key không?

---

### Task 1: Cài đặt dependencies và cấu hình môi trường cho Socket backend & frontend

**Files:**
- Modify: `apps/api/package.json`
- Modify: `apps/web/package.json`
- Modify: `apps/api/.env.example` (hoặc `.env`)
- Modify: `apps/web/.env.example` (hoặc `.env.local`)

- [ ] **Step 1: Thêm thư viện Socket vào backend và frontend**
  - Backend `apps/api`: `@nestjs/websockets`, `@nestjs/platform-socket.io`, `socket.io`
  - Frontend `apps/web`: `socket.io-client`
- [ ] **Step 2: Cập nhật biến môi trường**
  - `apps/api`: `INTERNAL_SOCKET_SECRET`, `NEXT_PUBLIC_SOCKET_URL` (nếu cần)
  - `apps/web`: `NEXT_PUBLIC_SOCKET_URL` (mặc định trỏ về Gateway `http://localhost:3000` ở local)

---

### Task 2: Xây dựng Socket Gateway và Endpoint cấp Ticket tại API Gateway

**Files:**
- Create: `apps/api/apps/api-gateway/src/socket/socket.module.ts`
- Create: `apps/api/apps/api-gateway/src/socket/inventory-events.gateway.ts`
- Create: `apps/api/apps/api-gateway/src/socket/socket-ticket.service.ts`
- Create: `apps/api/apps/api-gateway/src/socket/internal-events.controller.ts`
- Modify: `apps/api/apps/api-gateway/src/app.module.ts`

- [ ] **Step 1: Xây dựng service quản lý ticket kết nối ngắn hạn**
  - Sinh vé ngẫu nhiên có TTL ngắn (60-120 giây), gắn kèm thông tin `userId`, `role`, `branchId`.
  - Cung cấp endpoint HTTP `GET /api/socket/ticket` cho web gọi khi đã có session đăng nhập.
- [ ] **Step 2: Xây dựng `InventoryEventsGateway`**
  - Khởi tạo Socket.IO server gắn cùng HTTP server của Gateway.
  - Xác thực kết nối qua ticket ở `handshake.auth.ticket`.
  - Hỗ trợ client join/leave room theo `warehouse:<id>` và room cá nhân `user:<id>`.
- [ ] **Step 3: Xây dựng `InternalEventsController`**
  - Nhận HTTP POST `/api/internal/events/inventory-changed` từ các service nội bộ.
  - Kiểm tra Header bí mật `x-internal-secret`.
  - Broadcast sự kiện `inventory:changed` tới room kho tương ứng.
- [ ] **Step 4: Đăng ký `SocketModule` vào `AppModule` của API Gateway**

---

### Task 3: Phát sự kiện từ Inventory Service sau khi giao dịch kho thành công

**Files:**
- Create: `apps/api/apps/inventory-service/src/events/inventory-event-emitter.service.ts`
- Modify: `apps/api/apps/inventory-service/src/inventory.service.ts`
- Modify: `apps/api/apps/inventory-service/src/inventory-service.module.ts`

- [ ] **Step 1: Xây dựng HTTP emitter gửi tín hiệu sang Gateway**
  - Gửi POST sang `${API_GATEWAY_URL}/api/internal/events/inventory-changed` kèm payload `{ warehouseId, productId, documentCode, type, timestamp }`.
  - Bọc try/catch không làm sập luồng chính nếu Gateway tạm thời không phản hồi.
- [ ] **Step 2: Tích hợp vào `recordStockReceipt` và `recordStockIssue`**
  - Đặt lệnh gọi emitter CHỈ SAU KHI khối `prisma.$transaction` commit thành công.
- [ ] **Step 3: Đăng ký emitter vào `InventoryServiceModule`**

---

### Task 4: Tích hợp Socket Client và cơ chế Auto-Invalidate trên Web

**Files:**
- Create: `apps/web/src/lib/socket.ts`
- Create: `apps/web/src/providers/socket-provider.tsx`
- Create: `apps/web/src/features/inventory/hooks/use-inventory-socket.ts`
- Modify: `apps/web/src/app/(dashboard)/layout.tsx`
- Modify: `apps/web/src/features/inventory/hooks/use-inventory.ts`

- [ ] **Step 1: Tạo module quản lý vòng đời socket client (`apps/web/src/lib/socket.ts`)**
  - Tự động gọi API lấy ticket trước khi kết nối.
  - Cấu hình reconnect tự động và timeout hợp lý.
- [ ] **Step 2: Tạo `SocketProvider` bọc trong `(dashboard)/layout.tsx`**
  - Chỉ kích hoạt khi user đã authenticated.
  - Tự động ngắt kết nối khi logout.
- [ ] **Step 3: Tạo hook `useInventorySocket`**
  - Nhận `warehouseId` đang xem để join đúng room kho.
  - Khi nhận sự kiện `inventory:changed`:
    - Gọi `queryClient.invalidateQueries({ queryKey: inventoryKeys.all })`.
    - Hiển thị toast nhẹ/thông báo góc màn hình nếu người thực hiện không phải là chính mình (để người thao tác không bị spam 2 lần).
- [ ] **Step 4: Tinh chỉnh refetch interval của `use-inventory.ts`**
  - Nâng `refetchInterval` từ 10s lên 60s làm lưới an toàn dự phòng khi mất kết nối socket.

---

### Task 5: Kiểm thử tự động & Kiểm tra toàn diện hệ thống

**Files:**
- Create/Run: Unit/E2E test hoặc script kiểm tra socket kết nối và broadcast
- Check: TypeScript compile không lỗi ở cả `apps/api` và `apps/web`

- [ ] **Step 1: Kiểm tra build TypeScript**
  - Chạy `pnpm --filter warehouse-api build`
  - Chạy `pnpm --filter warehouse-web build` (hoặc type-check)
- [ ] **Step 2: Kiểm tra kịch bản hoạt động**
  - Xác thực ticket hợp lệ/không hợp lệ.
  - Nhập kho phát đúng sự kiện `inventory:changed`.
  - Xuất âm báo lỗi và không phát sự kiện.
  - Web nhận sự kiện và trigger query invalidation.
- [ ] **Step 3: Xác nhận không có commit hay push nào được thực hiện**
