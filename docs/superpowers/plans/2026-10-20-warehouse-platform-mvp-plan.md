# Warehouse Platform MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Chuyển đổi và tinh chỉnh có chọn lọc nền tảng sushi sang `warehouse-platform` phục vụ demo gấp cho khách hàng gồm: Quản lý hàng hóa, Nhập kho, Xuất kho, Theo dõi tồn kho thực tế và Cảnh báo hàng tồn thấp (< 5); giao diện phong cách logistics hiện đại chuẩn di động, liên kết 3 repository qua Git submodules.

**Architecture:** Kiến trúc phân tán phân lớp rõ ràng với Backend NestJS sử dụng Prisma ORM kết nối PostgreSQL (Neon), phân tách các schema `identity` và `inventory`; Frontend Next.js 15 App Router với Tailwind CSS, giao diện tối ưu responsive cho màn hình iPhone (Safari); Root repository quản lý `apps/api`, `apps/web` và `apps/mobile` dưới dạng Git submodules.

**Tech Stack:** NestJS, TypeScript, Prisma ORM, PostgreSQL (Neon), Cloudinary, Next.js 15, Tailwind CSS, Lucide Icons, Git Submodules.

**Spec:** `docs/superpowers/specs/2026-10-20-warehouse-platform-mvp-design.md`

## Global Constraints

- Database URL: `postgresql://neondb_owner:npg_VPZnr8YTo0wB@ep-dark-thunder-aodqllaa-pooler.c-2.ap-southeast-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require`
- Cloudinary: Cloud Name `hfu98tq3`, API Key `751647446247396`, API Secret `1AjAUaI1bA192IcAAt6aTLXFPWs`
- Không lưu commit thông tin mật database/cloudinary lên Git công khai; luôn dùng `.env` / `.env.local`
- Logo nguồn `logo.jpg` phải được chuyển sang định dạng PNG chất lượng cao và đặt tại các thư mục public của web
- Bảng màu: Xanh Navy / Slate (`#0F172A`, `#1E293B`, `#2563EB`), Cảnh báo tồn thấp: Industrial Amber (`#F59E0B`), Tuyệt đối không giữ màu sắc và thương hiệu sushi cũ
- ERD Invariant: Nghiêm cấm xuất vượt quá số dư hiện có (`quantity < 0`), cập nhật số dư và ghi nhận sổ cái biến động trong cùng transaction
- Ngưỡng cảnh báo tồn kho thấp mặc định là 5 đơn vị cơ sở nếu chưa cấu hình riêng

## Review Focus

1. Thử xuất kho số lượng lớn hơn số dư tồn kho hiện có -> Phải chặn đứng và trả về lỗi 400 rõ ràng, không làm biến đổi số dư
2. Nhập kho một sản phẩm -> Số dư tồn phải tăng đúng bằng số lượng nhập và sổ cái biến động `stock_movements` phải ghi nhận dòng IN
3. Xuất kho một sản phẩm -> Số dư tồn phải giảm đúng bằng số lượng xuất và sổ cái `stock_movements` phải ghi nhận dòng OUT
4. Khi số lượng tồn kho giảm xuống $\le 5$ -> Phải hiển thị badge cảnh báo Amber trên màn hình tồn kho và xuất hiện trong danh sách "Cảnh báo sắp hết hàng" ở Dashboard
5. Kiểm tra hiển thị trên viewport iPhone (375px - 430px) -> Thanh điều hướng, bảng danh sách và các nút thao tác không bị tràn viền, dễ bấm

---

### Task 1: Setup Workspace Structure, Logo Asset Conversion, and Git Submodules

**Files:**
- Create: `apps/api/.gitignore`, `apps/web/.gitignore`, `apps/mobile/README.md`
- Create: `.gitmodules`
- Create: `apps/web/public/logo.png`, `apps/web/public/favicon.ico`
- Source: `logo.jpg`

**Interfaces:**
- Consumes: `logo.jpg` tại root `warehouse-platform`
- Produces: `apps/web/public/logo.png` và cấu trúc 3 repository con đã khởi tạo git remote tương ứng

- [ ] **Step 1: Chuyển đổi `logo.jpg` sang `logo.png`**
  - Chạy script Node.js sử dụng canvas hoặc sharp / đọc buffer hình ảnh để tạo ra `logo.png` trong suốt hoặc nền sạch.
  - Lưu file tại `warehouse-platform/logo.png`.

- [ ] **Step 2: Khởi tạo thư mục và Git remotes cho 3 repository con**
  - Tạo thư mục tạm và clone hoặc init 3 repo:
    - `https://github.com/HoVietThang190704/warehouse-api.git` vào `apps/api`
    - `https://github.com/HoVietThang190704/warehouse-web.git` vào `apps/web`
    - `https://github.com/HoVietThang190704/warehouse-mobile.git` vào `apps/mobile`
  - Sao chép có chọn lọc mã nguồn backend từ `sushi-platform/apps/api` sang `warehouse-platform/apps/api` (loại bỏ thư mục `.git` cũ, loại bỏ `node_modules`, `dist`).
  - Sao chép có chọn lọc mã nguồn admin web từ `sushi-platform/apps/admin-web` sang `warehouse-platform/apps/web` (loại bỏ thư mục `.git` cũ, loại bỏ `node_modules`, `.next`).
  - Đặt README.md rõ ràng cho `apps/mobile` định hướng lộ trình React Native Expo kế thừa API từ web.

- [ ] **Step 3: Thiết lập `.gitmodules` trên root `warehouse-platform`**
  - Cấu hình file `.gitmodules` ánh xạ 3 submodules với 3 URL GitHub của tài khoản `HoVietThang190704`.

- [ ] **Step 4: Copy logo mới vào thư mục assets của Web**
  - Đặt `logo.png` vào `apps/web/public/logo.png` và cập nhật favicon.

- [ ] **Step 5: Verify cấu trúc & Commit Task 1**
  - Kiểm tra `git status` đảm bảo không có file rác thừa thãi.

---

### Task 2: Warehouse API — Database Migration & Seed Data (Neon Postgres)

**Files:**
- Modify: `apps/api/prisma/schema.prisma`
- Create: `apps/api/prisma/seed.ts`
- Create: `apps/api/.env` (được gitignore)
- Modify: `apps/api/package.json`

**Interfaces:**
- Consumes: PostgreSQL connection string từ Neon
- Produces: Schema `identity` và `inventory` đầy đủ trên Neon với seed dữ liệu kho mẫu và tài khoản quản trị

- [ ] **Step 1: Cấu hình `apps/api/.env`**
  - Tạo file `.env` chứa `DATABASE_URL` kết nối Neon:
    ```env
    DATABASE_URL="postgresql://neondb_owner:npg_VPZnr8YTo0wB@ep-dark-thunder-aodqllaa-pooler.c-2.ap-southeast-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require"
    JWT_SECRET="warehouse_super_secret_jwt_key_2026_demo"
    JWT_EXPIRATION="1d"
    REFRESH_TOKEN_SECRET="warehouse_super_secret_refresh_jwt_key_2026_demo"
    REFRESH_TOKEN_EXPIRATION="7d"
    PORT=3000
    CLOUDINARY_CLOUD_NAME="hfu98tq3"
    CLOUDINARY_API_KEY="751647446247396"
    CLOUDINARY_API_SECRET="1AjAUaI1bA192IcAAt6aTLXFPWs"
    ```

- [ ] **Step 2: Viết lại Prisma schema theo đúng ERD Quản lý kho**
  - Thay thế toàn bộ thực thể sushi (Combo, Món ăn, Nguyên liệu món) bằng thực thể kho:
    - Schema `identity`: `User`, `RefreshToken`, enum `UserRole` (`SUPER_ADMIN`, `WAREHOUSE_MANAGER`, `WAREHOUSE_STAFF`).
    - Schema `inventory`: `Branch`, `Warehouse`, `Unit`, `ProductCategory`, `Product` (có `minStock` mặc định 5), `StockBalance`, `StockDocument`, `StockDocumentItem`, `StockMovement`.

- [ ] **Step 3: Chạy `prisma db push` hoặc migration lên Neon**
  - Chạy lệnh `pnpm prisma db push` trong `apps/api` để tạo bảng trực tiếp trên Neon.
  - Xác nhận bảng đã được khởi tạo thành công không có lỗi cú pháp.

- [ ] **Step 4: Tạo script seed dữ liệu kho mẫu**
  - Viết `apps/api/prisma/seed.ts`:
    - Tạo 1 Chi nhánh chính: `Chi nhánh Trung Tâm (CN-01)`.
    - Tạo 1 Kho mặc định: `Kho Tổng Logistics (KHO-01)`.
    - Tạo 1 User Admin: `admin@warehouse.local` / mật khẩu `Admin@123`.
    - Tạo các Đơn vị tính: `Cái`, `Hộp`, `Thùng`, `Kg`, `Bao`.
    - Tạo Danh mục hàng hóa: `Thiết bị điện tử`, `Linh kiện cơ khí`, `Bao bì đóng gói`, `Hàng tiêu dùng`.
    - Tạo 8 mặt hàng mẫu đa dạng:
      1. Pin sạc dự phòng 20000mAh (Tồn: 25 - Đủ hàng)
      2. Cáp sạc nhanh Type-C (Tồn: 4 - CẢNH BÁO SẮP HẾT HÀNG)
      3. Băng keo đóng hàng 5cm (Tồn: 2 - CẢNH BÁO SẮP HẾT HÀNG)
      4. Thùng carton 3 lớp 30x20x10 (Tồn: 100 - Đủ hàng)
      5. Tai nghe Bluetooth TWS (Tồn: 0 - CẢNH BÁO HẾT HÀNG)
      6. Củ sạc đa năng 65W (Tồn: 15 - Đủ hàng)
      7. Màng xốp bóng khí 50cm (Tồn: 3 - CẢNH BÁO SẮP HẾT HÀNG)
      8. Chuột không dây văn phòng (Tồn: 18 - Đủ hàng)
  - Khởi tạo số dư ban đầu trong `StockBalance` và ghi nhận chứng từ mở đầu `StockDocument` + `StockMovement`.
  - Chạy seed: `pnpm ts-node prisma/seed.ts`.

- [ ] **Step 5: Commit Task 2 trong `apps/api`**

---

### Task 3: Warehouse API — Inventory Core Services & Endpoints

**Files:**
- Create: `apps/api/apps/inventory-service/src/dto/stock-operation.dto.ts`
- Modify: `apps/api/apps/inventory-service/src/inventory.service.ts`
- Modify: `apps/api/apps/inventory-service/src/inventory.controller.ts`
- Create: `apps/api/apps/inventory-service/src/inventory.service.spec.ts`
- Modify: `apps/api/apps/api-gateway/src/...` (đảm bảo routing thông suốt)

**Interfaces:**
- Produces:
  - `GET /api/v1/inventory/balances`
  - `POST /api/v1/inventory/receipts` (Nhập kho)
  - `POST /api/v1/inventory/issues` (Xuất kho - chặn xuất âm)
  - `GET /api/v1/inventory/kpis` (KPIs Dashboard)

- [ ] **Step 1: Viết Unit Test kiểm tra tính đúng đắn của logic Xuất kho và Nhập kho**
  - Viết test trong `inventory.service.spec.ts`:
    - Test 1: Nhập kho tăng số dư và tạo movement ledger.
    - Test 2: Xuất kho hợp lệ giảm số dư và tạo movement ledger.
    - Test 3: Xuất kho với số lượng lớn hơn số dư hiện có phải ném ra `BadRequestException("Số lượng tồn kho không đủ để xuất")`.
    - Test 4: Cảnh báo tồn thấp khi `quantity <= minStock`.

- [ ] **Step 2: Thực thi Unit Test và xác nhận FAIL trước khi implement**
  - Chạy `pnpm test apps/inventory-service/src/inventory.service.spec.ts`.

- [ ] **Step 3: Triển khai `InventoryService` chuẩn ACID transaction**
  - Hàm `recordStockReceipt(dto)`:
    - Sử dụng `prisma.$transaction`.
    - Cập nhật tăng `StockBalance` theo `(warehouseId, productId)`.
    - Tạo `StockDocument` trạng thái `POSTED`.
    - Tạo `StockMovement` loại `IN` với `quantityDelta > 0`.
  - Hàm `recordStockIssue(dto)`:
    - Sử dụng `prisma.$transaction`.
    - Đọc số dư hiện có trong `StockBalance` với pessimistic lock hoặc optimistic version.
    - Kiểm tra: nếu `balance.quantity < dto.quantity` -> ném lỗi `BadRequestException`.
    - Giảm số lượng trong `StockBalance`.
    - Tạo `StockDocument` loại `SALES_ISSUE` / `MANUAL_OUT` trạng thái `POSTED`.
    - Tạo `StockMovement` loại `OUT` với `quantityDelta = -dto.quantity`.
  - Hàm `getStockBalances(warehouseId, lowStockOnly)`:
    - Trả về danh sách tồn kèm `isLowStock: quantity <= minStock`.
  - Hàm `getDashboardKpis(warehouseId)`:
    - Tính tổng SKU, tổng số lượng tồn, số mặt hàng sắp hết, số giao dịch hôm nay.

- [ ] **Step 4: Chạy lại Unit Test và xác nhận PASS 100%**
  - Chạy `pnpm test apps/inventory-service/src/inventory.service.spec.ts`.

- [ ] **Step 5: Verify build & Commit Task 3**
  - Chạy `pnpm run build` trên `apps/api`.

---

### Task 4: Warehouse Web — Theme Redesign & Brand Identity Setup

**Files:**
- Modify: `apps/web/src/app/globals.css`
- Modify: `apps/web/src/config/navigation.config.ts`
- Modify: `apps/web/src/components/layout/sidebar.tsx`
- Modify: `apps/web/src/components/layout/header.tsx`
- Modify: `apps/web/src/app/layout.tsx`

**Interfaces:**
- Consumes: `logo.png`
- Produces: Giao diện nền tảng kho hàng tông màu Slate/Navy chuyên nghiệp, dọn sạch toàn bộ menu/text sushi

- [ ] **Step 1: Cập nhật CSS Palette & Typography**
  - Cập nhật `globals.css` định nghĩa các biến màu chuẩn Logistics:
    - Nền: Slate-50 / Slate-100
    - Sidebar & Header: Slate-900 / Slate-800
    - Accent: Blue-600 (`#2563EB`)
    - Alert/Warning: Amber-500 (`#F59E0B`), Amber-50 (`#FFFBEB`)
    - Xóa bỏ toàn bộ màu đỏ cam, phong cách ẩm thực sushi cũ.

- [ ] **Step 2: Cập nhật Cấu trúc Menu Navigation**
  - Viết lại `navigation.config.ts`:
    - 📊 **Tổng quan**: `/dashboard`
    - 📦 **Tồn kho**: `/inventory`
    - 📥 **Nhập kho**: `/inventory/in`
    - 📤 **Xuất kho**: `/inventory/out`
    - 🏷️ **Danh mục hàng hóa**: `/products`
    - 🏢 **Kho & Chi nhánh**: `/warehouses`
    - ⚙️ **Cài đặt hệ thống**: `/settings`

- [ ] **Step 3: Cập nhật Sidebar và Header với Logo mới**
  - Hiển thị `logo.png` ở Sidebar kèm tiêu đề `WAREHOUSE PLATFORM`.
  - Thêm thẻ hiển thị nhanh Kho đang thao tác (mặc định: `Kho Tổng Logistics`).
  - Hỗ trợ mobile drawer menu mở/đóng mượt mà trên Safari iPhone.

- [ ] **Step 4: Kiểm tra build Next.js**
  - Chạy `pnpm build` hoặc `pnpm lint` trong `apps/web` đảm bảo không lỗi type/import.

- [ ] **Step 5: Commit Task 4 trong `apps/web`**

---

### Task 5: Warehouse Web — Product Catalog Management

**Files:**
- Modify: `apps/web/src/features/catalog/...` -> Chuyển thành `apps/web/src/features/products/...`
- Create: `apps/web/src/app/(dashboard)/products/page.tsx`
- Create: `apps/web/src/features/products/components/product-list.tsx`
- Create: `apps/web/src/features/products/components/product-form-modal.tsx`

**Interfaces:**
- Consumes: `/api/v1/products`
- Produces: Màn hình danh sách hàng hóa và modal tạo mới sản phẩm với thiết lập ngưỡng cảnh báo `min_stock`

- [ ] **Step 1: Tạo API client & Hooks quản lý hàng hóa**
  - Tạo service gọi các API danh mục sản phẩm.

- [ ] **Step 2: Xây dựng màn hình danh sách hàng hóa**
  - Bảng dữ liệu hiển thị: Ảnh, Mã SKU, Tên hàng, Danh mục, Đơn vị tính, Ngưỡng báo động tồn (mặc định 5), Trạng thái.
  - Ô tìm kiếm nhanh theo SKU/Tên và lọc theo danh mục.

- [ ] **Step 3: Xây dựng Modal Thêm/Sửa hàng hóa**
  - Form validation bằng Zod:
    - Tên hàng (bắt buộc)
    - SKU (tự sinh hoặc nhập tay)
    - Đơn vị tính cơ sở
    - Ngưỡng cảnh báo tồn tối thiểu (`min_stock`, mặc định gợi ý 5)
    - Tích hợp tải ảnh lên Cloudinary nếu có ảnh.

- [ ] **Step 4: Verify và Commit Task 5**

---

### Task 6: Warehouse Web — Core Stock Operations & Low-Stock Alerts

**Files:**
- Create: `apps/web/src/app/(dashboard)/inventory/page.tsx`
- Create: `apps/web/src/app/(dashboard)/inventory/in/page.tsx`
- Create: `apps/web/src/app/(dashboard)/inventory/out/page.tsx`
- Create: `apps/web/src/features/inventory/components/stock-balance-table.tsx`
- Create: `apps/web/src/features/inventory/components/stock-receipt-form.tsx`
- Create: `apps/web/src/features/inventory/components/stock-issue-form.tsx`

**Interfaces:**
- Consumes: `/api/v1/inventory/balances`, `/api/v1/inventory/receipts`, `/api/v1/inventory/issues`
- Produces: Màn hình quản lý tồn kho có badge cảnh báo, form nhập kho, form xuất kho có kiểm soát tồn khả dụng

- [ ] **Step 1: Xây dựng màn hình Sổ tồn kho (`/inventory`)**
  - Bảng liệt kê: Mã SKU, Tên hàng hóa, Đơn vị, Số lượng tồn, Ngưỡng cảnh báo, Tình trạng.
  - Cột tình trạng hiển thị:
    - Badge Xanh: **Đủ hàng** (`Tồn > minStock`)
    - Badge Vàng nhấp nháy/Amber: **Sắp hết hàng (Tồn: X $\le$ minStock)**
    - Badge Đỏ: **Hết hàng (Tồn: 0)**
  - Nút Switch lọc nhanh: **"Chỉ hiển thị mặt hàng cần nhập thêm"** để lọc ngay những mặt hàng đang cảnh báo.

- [ ] **Step 2: Xây dựng màn hình Phiếu Nhập Kho (`/inventory/in`)**
  - Form chọn mặt hàng, nhập số lượng nhập (phải > 0), nhập đơn giá vốn và ghi chú phiếu nhập.
  - Nút bấm lớn "Hoàn tất nhập kho": Gửi request, hiển thị toast thông báo thành công và chuyển về màn hình tồn kho hoặc reset form.

- [ ] **Step 3: Xây dựng màn hình Phiếu Xuất Kho (`/inventory/out`)**
  - Form chọn mặt hàng cần xuất.
  - Hiển thị ngay hộp thông tin số lượng tồn hiện có: `Số lượng khả dụng: X [đơn vị]`.
  - Nhập số lượng xuất:
    - Nếu `số lượng xuất > số lượng khả dụng`: Vô hiệu hóa nút xuất, viền đỏ ô nhập, hiển thị thông báo: `Không được xuất vượt quá số tồn hiện có (tối đa X)`.
    - Nếu hợp lệ: Cho phép bấm "Xác nhận xuất kho".

- [ ] **Step 4: Verify luồng và Commit Task 6**

---

### Task 7: Warehouse Web — Dashboard Overview & iPhone Safari Responsive Check

**Files:**
- Modify: `apps/web/src/app/(dashboard)/dashboard/page.tsx`
- Create: `apps/web/src/features/dashboard/components/kpi-cards.tsx`
- Create: `apps/web/src/features/dashboard/components/low-stock-alert-card.tsx`
- Create: `apps/web/src/features/dashboard/components/recent-transactions-card.tsx`

**Interfaces:**
- Consumes: `/api/v1/inventory/kpis`
- Produces: Màn hình Dashboard trực quan làm điểm nhấn cho buổi demo và bố cục hiển thị hoàn hảo trên điện thoại

- [ ] **Step 1: Triển khai 4 KPI Cards**
  - Card 1: Tổng số mã hàng (Items)
  - Card 2: Tổng lượng hàng trong kho (Total Units)
  - Card 3: **Cảnh báo cần nhập hàng** (Số lượng mặt hàng $\le 5$ hoặc $\le$ minStock, tô màu Amber cảnh báo)
  - Card 4: Biến động xuất nhập gần nhất

- [ ] **Step 2: Widget "Mặt hàng sắp hết cần đặt hàng ngay"**
  - Danh sách top các sản phẩm tồn kho chạm ngưỡng đỏ/vàng kèm nút bấm nhanh "Tạo phiếu nhập".

- [ ] **Step 3: Tinh chỉnh Responsive cho màn hình iPhone (Safari)**
  - Đảm bảo Drawer Menu đóng mở mượt trên iOS.
  - Trên màn hình di động: Bảng tự động hiển thị dạng Card trực quan thay vì table tràn ngang.
  - Các nút hành động đạt chuẩn kích thước bấm cảm ứng $\ge 44px$.

- [ ] **Step 4: Verify build Next.js & Commit Task 7**

---

### Task 8: End-to-End Verification, Root Repository Submodules Sync & Push

**Files:**
- Root: `.gitmodules`, `README.md`
- Submodules: `apps/api`, `apps/web`, `apps/mobile`

- [ ] **Step 1: Khởi chạy cả API và Web local để kiểm tra luồng hoàn chỉnh**
  - Đăng nhập tài khoản `admin@warehouse.local`.
  - Xem Dashboard: Thấy các thẻ KPI và danh sách cảnh báo tồn thấp.
  - Vào Nhập kho: Nhập thêm 10 cái cho Cáp sạc Type-C (từ 4 lên 14) -> Quay lại Dashboard thấy số lượng mặt hàng cảnh báo giảm đi 1.
  - Vào Xuất kho: Thử xuất 100 cái Pin dự phòng (tồn 25) -> Hệ thống chặn và báo lỗi rõ ràng. Xuất 20 cái -> Tồn còn 5 -> Trở thành hàng cảnh báo vàng.

- [ ] **Step 2: Kiểm tra bảo mật**
  - Xác nhận `.env` không bị commit vào git ở cả 3 repo con và repo root.
  - Kiểm tra các file `.gitignore`.

- [ ] **Step 3: Commit và Push lên GitHub**
  - Push commit của `apps/api` lên `https://github.com/HoVietThang190704/warehouse-api.git`.
  - Push commit của `apps/web` lên `https://github.com/HoVietThang190704/warehouse-web.git`.
  - Push commit của `apps/mobile` lên `https://github.com/HoVietThang190704/warehouse-mobile.git`.
  - Commit và Push repo root `warehouse-platform` cập nhật 3 submodule pointers lên `https://github.com/HoVietThang190704/warehouse-platform.git`.
