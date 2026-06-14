# [JAVA-BACKEND-ROADMAP] GIAI ĐOẠN 2: DỊCH CHUYỂN SANG JPA & POSTGRESQL

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Giữ nguyên các lớp: Controller, DTO (Request/Response)
- [x] Giữ nguyên vị trí Validation & Mapping tại Service
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang thực hiện bóc tách phần lưu trữ dữ liệu. Chuẩn bị tạo lớp Repository và cấu hình kết nối PostgreSQL.
- [ ] BƯỚC TIẾP THEO: Viết code cho `CustomerRepository` và cấu hình file `application.properties`.

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC SAU KHI THAY THẾ
Hãy chú ý vào thư mục `repository` mới được thêm vào và file `Customer.java` được gắn nhãn Entity.

```text
src/main/java/com/example/project/
├── controller/              (GIỮ NGUYÊN)
│   └── CustomerController.java
├── dto/                     (GIỮ NGUYÊN)
│   ├── CreateCustomerRequest.java
│   ├── UpdateCustomerRequest.java
│   └── CustomerResponse.java
├── model/                   
│   └── Customer.java        <-- [CẬP NHẬT] Thêm @Entity để ánh xạ thành bảng trong PostgreSQL
├── repository/              <-- [✨ THƯ MỤC MỚI] Nơi lo việc giao tiếp với Database thật
│   └── CustomerRepository.java
└── service/                 
    └── CustomerService.java <-- [CẬP NHẬT CODE] Xóa biến Map DB, gọi sang CustomerRepository