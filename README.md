# Warehouse Platform

Hệ thống quản lý kho hàng và chuỗi cung ứng tập trung: Quản lý danh mục hàng hóa, Nhập kho, Xuất kho, Theo dõi tồn kho thời gian thực và Cảnh báo hàng tồn thấp (< 5) chống hụt hàng.

## 1. Cấu trúc Repository (Git Submodules)

Dự án được phân tách thành 3 repository độc lập liên kết qua Git Submodules:

- **`apps/api`**: [warehouse-api](https://github.com/HoVietThang190704/warehouse-api.git) — Backend NestJS microservices (Gateway, Identity, Catalog, Inventory) kết nối PostgreSQL (Neon Cloud) và Cloudinary.
- **`apps/web`**: [warehouse-web](https://github.com/HoVietThang190704/warehouse-web.git) — Admin Web Next.js 15 App Router, Tailwind CSS, tối ưu hóa giao diện di động (Safari iPhone).
- **`apps/mobile`**: [warehouse-mobile](https://github.com/HoVietThang190704/warehouse-mobile.git) — Ứng dụng di động Expo / React Native (kế thừa nghiệp vụ từ API web).

## 2. Hướng dẫn khởi chạy Local

### Bước 1: Khởi chạy Backend (`warehouse-api`)
```bash
cd apps/api
pnpm install
pnpm prisma db push
pnpm ts-node prisma/seed.ts  # Khởi tạo dữ liệu kho mẫu và tài khoản admin
pnpm dev                     # Khởi chạy đồng thời API Gateway và các microservices
```
- API Gateway lắng nghe tại: `http://localhost:3000`

### Bước 2: Khởi chạy Giao diện Quản trị (`warehouse-web`)
```bash
cd apps/web
pnpm install
pnpm dev                     # Lắng nghe tại http://localhost:3004
```
- Mở trình duyệt truy cập: `http://localhost:3004`
- Tài khoản quản trị mặc định: `admin@warehouse.local` / `Admin@123`

## 3. Các tính năng cốt lõi cho Demo
1. **Sổ Theo Dõi Tồn Kho (`/dashboard/inventory`)**: Hiển thị số lượng tồn thực tế theo từng mặt hàng, badge trạng thái (Đủ hàng, Sắp hết, Hết hàng), nút lọc nhanh mặt hàng cần đặt bổ sung.
2. **Cảnh Báo Hàng Sắp Hết (&le; 5)**: Tự động phát hiện và đưa vào danh sách cảnh báo trên Dashboard và sổ tồn khi số lượng &le; ngưỡng tối thiểu.
3. **Nhập Kho Hàng Hóa (`/dashboard/inventory/in`)**: Tăng số dư tức thì, ghi sổ cái biến động bất biến.
4. **Xuất Kho Chống Xuất Âm (`/dashboard/inventory/out`)**: Kiểm soát chặt chẽ số lượng khả dụng, nghiêm cấm xuất âm.
5. **Danh Mục Hàng Hóa (`/dashboard/products`)**: Quản lý SKU, mã vạch, đơn vị tính cơ sở, ngưỡng báo động và ảnh Cloudinary.
