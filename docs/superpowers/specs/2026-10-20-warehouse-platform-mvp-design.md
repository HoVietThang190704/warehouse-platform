# Warehouse Platform MVP — Design Specification

**Date:** 2026-10-20  
**Project:** Warehouse Platform (Quản lý kho hàng)  
**Status:** Approved for Implementation Planning

---

## 1. Context & Business Intent

Dự án chuyển đổi từ nền tảng `sushi-platform` sang `warehouse-platform` nhằm phục vụ nhu cầu quản lý kho hàng thực tế của khách hàng. Khách hàng đang cần gấp bản demo tập trung vào bài toán cốt lõi:
- **Nhập hàng vào kho**
- **Theo dõi số lượng tồn kho theo thời gian thực**
- **Xuất kho có kiểm soát (không cho phép xuất âm/xuất vượt tồn)**
- **Cảnh báo tức thì khi hàng tồn kho xuống thấp hoặc chạm ngưỡng tối thiểu (mặc định dưới 5 sản phẩm)** để không bị hụt hàng khi có đơn tiếp theo.
- Giao diện Admin Web chuyên nghiệp, phong cách logistics hiện đại (thay thế giao diện quán ăn/sushi), hỗ trợ tốt thao tác trên thiết bị di động (Safari iPhone).
- Chuẩn bị sẵn cấu trúc 3 repository độc lập (`warehouse-api`, `warehouse-web`, `warehouse-mobile`) liên kết qua Git submodules trên `warehouse-platform`. Phiên bản mobile native (Expo) sẽ được phát triển ở giai đoạn sau, tái sử dụng các business logic và API này.

---

## 2. Global Constraints & Credentials

1. **Database:** PostgreSQL trên Neon Serverless:
   - Connection URL: `postgresql://neondb_owner:npg_VPZnr8YTo0wB@ep-dark-thunder-aodqllaa-pooler.c-2.ap-southeast-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require`
   - Chỉ lưu trữ trong biến môi trường `.env` / `.env.local`, tuyệt đối không commit thông tin mật lên Git.
2. **Media Storage (Cloudinary):**
   - Cloud Name: `hfu98tq3`
   - API Key: `751647446247396`
   - API Secret: `1AjAUaI1bA192IcAAt6aTLXFPWs`
   - Dùng cho upload hình ảnh sản phẩm kho hàng.
3. **Logo:**
   - Sử dụng file gốc `logo.jpg` ở thư mục gốc `warehouse-platform`, chuyển đổi sang định dạng `logo.png` chất lượng cao, tối ưu dùng cho sidebar, favicon, navbar và màn hình đăng nhập.
4. **Git Repository Architecture:**
   - Root: `https://github.com/HoVietThang190704/warehouse-platform.git`
   - API Submodule (`apps/api`): `https://github.com/HoVietThang190704/warehouse-api.git`
   - Web Submodule (`apps/web`): `https://github.com/HoVietThang190704/warehouse-web.git`
   - Mobile Submodule (`apps/mobile`): `https://github.com/HoVietThang190704/warehouse-mobile.git`

---

## 3. Architecture & Data Model (Aligning with Warehouse ERD)

Khác với `sushi-platform` (quản lý công thức combo và nguyên liệu chế biến món ăn), `warehouse-platform` triển khai mô hình quản lý kho theo ERD chuẩn `warehouse_management_erd.dbml`:

### 3.1. Schemas trong PostgreSQL
1. **`identity` schema:**
   - `users`: Quản lý tài khoản đăng nhập (Admin, Quản lý kho, Thủ kho).
   - `refresh_tokens`: Quản lý JWT refresh token an toàn (HttpOnly cookie).
   - `roles`, `permissions`: Phân quyền RBAC.
2. **`inventory` schema:**
   - `branches`: Chi nhánh quản lý kho (mặc định tạo 1 chi nhánh chính).
   - `warehouses`: Kho hàng (mặc định 1 kho trung tâm `is_default = true`).
   - `units`: Đơn vị tính cơ sở (`Cái`, `Hộp`, `Thùng`, `Kg`, `Bao`).
   - `product_categories`: Danh mục nhóm hàng hóa.
   - `products`: Thông tin hàng hóa (`sku`, `barcode`, `name`, `category_id`, `base_unit_id`, `min_stock` mặc định = 5, `cost_price`, `selling_price`, `image_url`, `is_active`).
   - `stock_balances`: Bảng số dư tức thời theo `(warehouse_id, product_id)` gồm `quantity`, `average_unit_cost`, `version` (optimistic locking), `updated_at`.
   - `stock_documents`: Chứng từ nhập/xuất kho (`PURCHASE_RECEIPT`, `MANUAL_IN`, `MANUAL_OUT`, `SALES_ISSUE`). Trạng thái: `DRAFT`, `POSTED`, `VOIDED`.
   - `stock_document_items`: Chi tiết từng mặt hàng và số lượng trên chứng từ.
   - `stock_movements`: Sổ cái kho bất biến (`immutable movement ledger`), ghi nhận `quantity_delta` (dương cho nhập, âm cho xuất), `movement_type` (`IN`/`OUT`), `unit_cost`.

### 3.2. Quy tắc nghiệp vụ kho bất biến (ERD Invariants)
- **Độc quyền ghi nhận:** Chỉ khi chứng từ chuyển sang trạng thái `POSTED`, `stock_balances` mới được cập nhật đồng thời sinh `stock_movements` trong cùng 1 database transaction.
- **Chặn xuất âm:** Nghiêm cấm xuất vượt quá số dư hiện có (`quantity < 0`). Bất kỳ giao dịch xuất nào thiếu tồn phải bị reject ngay lập tức với lỗi rõ ràng `400 Bad Request: Insufficient stock`.
- **Cảnh báo tồn thấp:** Khi `stock_balances.quantity <= products.min_stock` (hoặc ngưỡng mặc định 5 khi chưa cài đặt), hàng hóa lập tức được gắn cờ `LOW_STOCK` trên cả API và Web dashboard.

---

## 4. API Endpoints Specification (`warehouse-api`)

Backend viết bằng NestJS, module hóa theo Clean Architecture:

1. **Authentication (`/api/v1/auth`):**
   - `POST /login`: Đăng nhập email/password, cấp Access Token & Refresh Token.
   - `GET /me`: Lấy thông tin user hiện tại.
   - `POST /refresh`: Làm mới token.
2. **Products & Catalog (`/api/v1/products`):**
   - `GET /`: Danh sách hàng hóa, hỗ trợ tìm kiếm theo SKU, tên, barcode, lọc danh mục.
   - `POST /`: Tạo hàng hóa mới (kèm đơn vị tính, min_stock mặc định 5, ảnh Cloudinary).
   - `GET /:id`: Chi tiết hàng hóa.
   - `PUT /:id`: Cập nhật thông tin hàng hóa.
   - `DELETE /:id`: Vô hiệu hóa hàng hóa (`is_active = false`).
3. **Warehouses & Units (`/api/v1/warehouses`, `/api/v1/units`, `/api/v1/categories`):**
   - Danh mục kho hàng và đơn vị tính hỗ trợ luồng nhập xuất.
4. **Stock Operations (`/api/v1/inventory`):**
   - `GET /balances`: Danh sách tồn kho theo từng kho hàng. Hỗ trợ query `lowStockOnly=true` để lấy ngay danh sách hàng cần cảnh báo nhập thêm.
   - `POST /receipts`: Tạo và duyệt phiếu nhập kho (`POSTED`), cập nhật tăng số dư tức thì và ghi nhận ledger movement.
   - `POST /issues`: Tạo và duyệt phiếu xuất kho (`POSTED`), kiểm tra số dư khả dụng, trừ tồn an toàn và ghi nhận ledger movement.
   - `GET /documents`: Lịch sử các chứng từ nhập/xuất kho.
   - `GET /movements`: Sổ nhật ký biến động kho (Audit trail).
   - `GET /dashboard-kpis`: Tổng hợp nhanh: Tổng mã hàng, Tổng lượng tồn, Số mặt hàng sắp hết/chạm ngưỡng, Giá trị tồn kho ước tính.

---

## 5. Frontend Specification (`warehouse-web`)

Admin Web xây dựng bằng Next.js 15 App Router, Tailwind CSS, Lucide Icons:

### 5.1. Bảng màu & Visual Identity
- **Primary:** Deep Slate Navy (`#0F172A`, `#1E293B`) kết hợp Modern Blue Accent (`#2563EB`, `#3B82F6`). Mang lại cảm giác chắc chắn, chuyên nghiệp của hệ thống quản lý chuỗi cung ứng.
- **Warning/Alert:** Industrial Amber (`#F59E0B`, nền nhẹ `#FEF3C7`, viền `#FCD34D`) dùng để làm nổi bật tức thì các mặt hàng tồn kho $\le 5$ hoặc $\le$ min_stock.
- **Success:** Emerald (`#10B981`) cho chứng từ hoàn thành, nhập kho thành công.
- **Danger:** Rose/Red (`#EF4444`) cho cảnh báo hết hàng hoàn toàn (tồn = 0) hoặc xuất lỗi.
- **Logo:** `logo.png` hiển thị đồng bộ ở header, sidebar thu gọn/mở rộng, favicon và trang login.

### 5.2. Các màn hình chính
1. **Trang đăng nhập (`/login`):**
   - Logo thương hiệu mới, form đăng nhập gọn gàng, hỗ trợ tài khoản demo có sẵn (ví dụ: `admin@warehouse.local` / mật khẩu khởi tạo).
2. **Bảng điều khiển kho (`/dashboard`):**
   - 4 KPI cards: Tổng mã hàng hóa, Tổng lượng tồn trong kho, **Cảnh báo sắp hết hàng (< 5)** (nổi bật màu Amber), Biến động nhập xuất hôm nay.
   - Bảng top 5 mặt hàng sắp hết cần đặt gấp để tránh hụt hàng.
   - Bảng 5 giao dịch nhập/xuất gần nhất.
3. **Danh mục hàng hóa (`/products`):**
   - Bảng danh sách hàng hóa với ảnh, SKU, Tên, Đơn vị tính, Ngưỡng cảnh báo tồn (mặc định 5).
   - Nút "Thêm hàng hóa" với modal nhập thông tin nhanh và upload ảnh lên Cloudinary.
4. **Sổ theo dõi tồn kho (`/inventory`):**
   - Lọc theo kho hàng, tìm kiếm theo tên/SKU/barcode.
   - Switch toggle: **"Chỉ hiện hàng sắp hết / hết hàng"** để thủ kho xem ngay danh sách cần nhập bổ sung.
   - Hiển thị badge trực quan: Xanh (Đủ hàng), Vàng (Sắp hết $\le 5$), Đỏ (Hết hàng = 0).
5. **Phiếu nhập kho (`/inventory/in`):**
   - Giao diện tạo phiếu nhập đơn giản: Chọn kho nhập, chọn sản phẩm, nhập số lượng, đơn giá, ghi chú.
   - Nút "Xác nhận nhập kho": Gửi lệnh lên backend, cập nhật tồn kho tức thời.
6. **Phiếu xuất kho (`/inventory/out`):**
   - Chọn kho xuất, chọn sản phẩm, nhập số lượng xuất, lý do xuất.
   - Hiển thị ngay số lượng tồn hiện có của sản phẩm.
   - Nếu người dùng nhập số lượng xuất > tồn hiện có: Chặn nút xuất, hiển thị cảnh báo đỏ "Không thể xuất quá số lượng tồn kho hiện tại (Tồn: X)".
7. **Tối ưu hóa thiết bị di động (iPhone / Safari):**
   - Giao diện responsive 100%, drawer menu điều hướng mượt mà, kích thước nút bấm tối thiểu 44px chuẩn Apple HIG, bảng dữ liệu tự chuyển card view gọn gàng trên màn hình hẹp.

---

## 6. Migration & Repository Setup Strategy

1. **Khởi tạo và dọn dẹp:**
   - Dọn toàn bộ từ khóa, tài nguyên, assets liên quan đến "sushi" trong code copy sang.
   - Giữ nguyên cấu trúc code chuẩn NestJS và Next.js đã có sẵn cấu hình tốt (eslint, prettier, tsconfig, packages).
2. **Cấu trúc Git Submodules tại root `warehouse-platform`:**
   - Cấu hình `.gitmodules` liên kết 3 repo:
     - `apps/api` -> `https://github.com/HoVietThang190704/warehouse-api.git`
     - `apps/web` -> `https://github.com/HoVietThang190704/warehouse-web.git`
     - `apps/mobile` -> `https://github.com/HoVietThang190704/warehouse-mobile.git`
3. **Triển khai Database:**
   - Chạy Prisma migration tạo các bảng và schema trong database Neon.
   - Chạy seed data khởi tạo: 1 Branch mặc định, 1 Warehouse chính, 1 User Admin, các Units cơ bản, và 5-10 sản phẩm mẫu (với các mức tồn khác nhau: đủ tồn, sắp hết < 5, hết hàng = 0) để sẵn sàng demo ngay lập tức cho khách hàng.

---

## 7. Acceptance Criteria for MVP Demo

- [ ] Logo mới `logo.png` xuất hiện đồng bộ ở `warehouse-web`.
- [ ] Bảng màu giao diện xanh navy chuyên nghiệp, không còn dấu vết màu sắc thương hiệu sushi cũ.
- [ ] Đăng nhập thành công vào hệ thống quản lý kho.
- [ ] Dashboard hiển thị đúng 4 thông số: Tổng mã hàng, Tổng tồn, Cảnh báo tồn thấp, Biến động gần đây.
- [ ] Thực hiện tạo phiếu **Nhập kho** -> Số tồn kho của mặt hàng tương ứng tăng lên chính xác.
- [ ] Thực hiện tạo phiếu **Xuất kho** hợp lệ -> Số tồn kho giảm chính xác.
- [ ] Thực hiện thử **Xuất kho vượt quá tồn** -> Hệ thống chặn và báo lỗi rõ ràng, không làm tồn âm.
- [ ] Khi số lượng tồn của một mặt hàng rớt xuống $\le 5$ (hoặc $\le$ min_stock) -> Mặt hàng lập tức xuất hiện trong mục cảnh báo sắp hết hàng trên Dashboard và danh sách tồn kho.
- [ ] Giao diện web hiển thị tốt và thao tác mượt mà trên iPhone.
- [ ] 3 submodules được cấu hình chuẩn xác trong Git của `warehouse-platform`.
