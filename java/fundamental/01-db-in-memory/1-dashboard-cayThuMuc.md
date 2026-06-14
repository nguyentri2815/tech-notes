# [JAVA-BACKEND-ROADMAP] GIAI ĐOẠN 1: IN-MEMORY DATABASE

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Cấu trúc thư mục (Packages)
- [x] Thiết lập luồng dữ liệu (Data Flow) dạng RAM
- [/] ĐIỂM DỪNG HIỆN TẠI: Đã chạy thử nghiệm lưu trữ bằng `ConcurrentHashMap` thành công trong `CustomerService.java`.
- [ ] BƯỚC TIẾP THEO (GIAI ĐOẠN 2): Bóc tách RAM DB để nâng cấp lên MySQL/PostgreSQL.

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC HIỆN TẠI
```text
src/main/java/com/example/project/
├── controller/
│   └── CustomerController.java      <-- (Nhận Request từ Client)
├── dto/
│   ├── CreateCustomerRequest.java
│   ├── UpdateCustomerRequest.java
│   └── CustomerResponse.java
├── model/
│   └── Customer.java                <-- (Đối tượng định hình bảng dữ liệu)
└── service/
    └── CustomerService.java         <-- [⚡ ĐIỂM DỪNG] Chứa logic + biến Map DB tạm thời