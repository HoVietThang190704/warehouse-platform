# Quản lý tài khoản nhân viên và phân quyền theo kho

## Mục tiêu và phạm vi

Quản trị viên tự tạo và quản lý tài khoản cho nhân viên; nhân viên cấp dưới chỉ được xem dữ liệu cần thiết và xuất hàng tại **một kho được gán**, không được nhập kho hay chỉnh sửa danh mục. Quyền phải được bảo vệ tại API, kể cả khi người dùng gọi trực tiếp không qua giao diện. Không xây dựng luồng tự đăng ký, mời qua email, hoặc bộ cấu hình quyền tùy ý theo từng người.

Đã xác nhận: chỉ `SUPER_ADMIN` được cấp tài khoản và đổi quyền; quản trị viên đặt mật khẩu khi tạo tài khoản, nhân viên không bắt buộc đổi ở lần đăng nhập đầu. Quản trị viên bàn giao mật khẩu qua kênh riêng, không hiển thị lại mật khẩu đã lưu.

## Hiện trạng cần xử lý

- `apps/api/prisma/schema.prisma` có `User` với `role`, `branchId`, `isActive`; `branchId` là chi nhánh, **không phải** kho. Kho nằm ở `Warehouse`.
- `apps/api/apps/identity-service/src/auth.controller.ts` mở công khai `POST /auth/register`; tài khoản mới mặc định `WAREHOUSE_STAFF`.
- `apps/api/apps/inventory-service/src/inventory.controller.ts` chưa chặn quyền trên đọc tồn, nhập và xuất; service chọn kho mặc định nếu request không có `warehouseId`.
- `apps/api/apps/catalog-service/src/guards/roles.guard.ts` cho phép đi qua khi thiếu role; một số đường ghi, gồm upload ảnh, chưa được bảo vệ thống nhất.
- Gateway gắn các header `x-user-*` từ JWT nhưng chưa loại bỏ header cùng tên do client tự gửi, và token chứa role có thể cũ sau khi đổi quyền/khóa tài khoản.
- Web có trang nhập/xuất kho, menu dùng chung, nhưng kiểu vai trò trong `apps/web/src/types/auth.ts` vẫn là bộ vai trò cũ, khác backend.

## Mô hình quyền

| Vai trò | Quản lý tài khoản | Đọc dữ liệu cần cho vận hành | Nhập kho | Xuất kho | Ghi danh mục |
| --- | --- | --- | --- | --- | --- |
| `SUPER_ADMIN` | Có | Có | Có | Có | Có |
| `WAREHOUSE_MANAGER` | Không | Có | Có | Có | Có (giữ hành vi hiện có) |
| `WAREHOUSE_STAFF` | Không | Chỉ trong kho được gán, cộng dữ liệu danh mục cần để xuất | Không | Chỉ kho được gán | Không |

Không cho tài khoản mới tự chọn vai trò; quản trị viên tạo tài khoản với `WAREHOUSE_STAFF` hoặc `WAREHOUSE_MANAGER`. Giữ vai trò quản lý kho theo hành vi hiện có trong giai đoạn này; không mặc định thay đổi quyền/phạm vi của quản lý kho khi giới hạn nhân viên. Các endpoint danh mục dùng chung cho phép nhân viên **đọc** thông tin cần thiết để xuất nhưng không cho ghi.

## Tài khoản, dữ liệu và API

- Thêm `warehouseId` tùy chọn vào `User`, tham chiếu đến kho hợp lệ, đang hoạt động; bắt buộc đối với `WAREHOUSE_STAFF`. Không dùng `branchId` để kiểm tra quyền kho. Khi chi nhánh vẫn được lưu, chi nhánh của tài khoản nhân viên phải khớp chi nhánh của kho được gán.
- Bỏ khả năng tự tạo tài khoản qua `POST /auth/register` công khai: chuyển sang endpoint chỉ admin hoặc loại bỏ route đó và dùng `POST /users` riêng. Tạo mới xác thực, kiểm tra email trùng, hash mật khẩu và chỉ trả các trường công khai.
- API admin gồm danh sách tài khoản, tạo, cập nhật tên/vai trò/kho, khóa/mở và đặt lại mật khẩu. Không cho vô hiệu hóa/hạ quyền `SUPER_ADMIN` cuối cùng; không cho người dùng tự nâng quyền; tránh thao tác tự khóa tài khoản đang dùng. Không trả hash/mật khẩu trong response hoặc log.
- Tài khoản có thể đổi mật khẩu bằng luồng riêng sau này; phạm vi hiện tại tối thiểu là admin đặt lại mật khẩu. Không buộc đổi mật khẩu lần đầu và không gửi email tự động.
- Các thay đổi vai trò, kho, trạng thái và mật khẩu phải làm phiên cũ mất hiệu lực hoặc được kiểm tra lại bằng trạng thái hiện tại trước mỗi yêu cầu; token bị thu hồi không được dùng tiếp, kể cả refresh. Thiết kế cần xử lý cả cookie Web và bearer token Mobile.

## Ranh giới xác thực và thực thi quyền

- Ở mọi API nhạy cảm, xác thực JWT và đối chiếu `User` hiện tại (`isActive`, role, `warehouseId`) bằng dữ liệu đáng tin cậy; từ chối khi thiếu xác thực hoặc thông tin quyền. Không tin `x-user-*` do client gửi. Gateway phải loại bỏ các header ủy quyền do client tự đặt trước khi proxy và các service không được nhận chúng làm căn cứ quyền nếu không có nguồn xác thực đáng tin cậy. Việc cấu hình service chỉ nhận traffic nội bộ là lớp bảo vệ bổ sung, không thay thế kiểm tra quyền.
- Nhân viên đọc danh sách kho chỉ nhận kho đã gán; xem tồn, KPI và chứng từ phải bị giới hạn theo kho. Nếu request nêu kho khác thì trả `403`, không âm thầm trả dữ liệu kho khác. Nếu thiếu `warehouseId`, mặc định dùng kho được gán cho nhân viên thay vì kho mặc định toàn hệ thống.
- `POST /inventory/receipts` chỉ cho admin/quản lý; `POST /inventory/issues` cho admin/quản lý và nhân viên xuất đúng kho. Quyết định và phạm vi kho lấy từ phía server, không từ DTO hoặc header tự khai. Giữ các kiểm tra nghiệp vụ hiện có như chống xuất âm.
- Bảo vệ thao tác ghi danh mục và upload ảnh; nơi thiếu role/trạng thái hiện phải trả 401/403, không fail-open. Nếu các service có thể được gọi trực tiếp, áp dụng kiểm tra JWT/trạng thái tại service hoặc một cơ chế nội bộ được xác thực có hiệu lực tương đương.

## Web và trải nghiệm sử dụng

- Thêm `/dashboard/employees` và mục “Nhân viên” chỉ hiện với admin. Danh sách có trạng thái, vai trò, kho; biểu mẫu tạo/chỉnh sửa, khóa/mở và đặt lại mật khẩu có xác nhận thao tác nhạy cảm.
- Đồng bộ type vai trò Web với backend. Nhân viên chỉ thấy màn hình phù hợp để xem tồn/xuất, không thấy menu hoặc nút nhập và sửa danh mục. Truy cập URL trực tiếp phải hiện trạng thái không có quyền; API vẫn là ranh giới bảo mật cuối cùng.
- Giao diện xử lý trạng thái tải, lỗi trùng email, mật khẩu không hợp lệ, kho đã ngừng hoạt động, phiên hết hạn và `403`. Không hiển thị password hash hoặc bật lại mật khẩu sau khi lưu.
- Nếu điều hướng mobile dùng cùng `NAV_ITEMS`, mục chỉ dành admin không được xuất hiện trong thanh điều hướng/gesture của nhân viên.

## Kiểm thử và nghiệm thu

1. Không xác thực không thể tạo tài khoản hoặc gọi API nhập/xuất; nhân viên không thể tự đăng ký hoặc nâng quyền.
2. Admin tạo tài khoản nhân viên với kho hợp lệ; nhập bằng tài khoản đó xem tồn và xuất tại kho được gán thành công.
3. Nhân viên gọi trực tiếp API nhập kho, ghi danh mục, upload ảnh, hoặc xem/xuất kho khác bị từ chối; thiếu kho không trượt về kho mặc định toàn hệ thống.
4. Header `x-user-role`, `x-user-id`, `x-user-branch-id` giả mạo không mở được quyền; token cũ không giữ quyền sau khi admin đổi role/kho, khóa tài khoản hoặc đặt lại mật khẩu.
5. Tài khoản bị khóa không thể đăng nhập, refresh hoặc dùng phiên cũ; không thể vô hiệu hóa quản trị viên cuối cùng.
6. UI admin tạo/sửa/khóa/đặt lại mật khẩu hoạt động; UI nhân viên không thấy chức năng bị cấm; truy cập URL trực tiếp và lỗi API được xử lý đúng.

## Ngoài phạm vi

Không thêm phân quyền checkbox tùy ý, luồng email/invite, đăng ký công khai, quy trình duyệt xuất kho, phân quyền quản lý kho theo từng chi nhánh mới, hoặc thiết kế lại ứng dụng Mobile. API phải tiếp tục hỗ trợ client Mobile hiện có ở mức xác thực/bearer token.
