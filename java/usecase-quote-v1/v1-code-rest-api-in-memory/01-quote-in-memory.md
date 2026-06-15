# [QUOTE-SERVICE-EVOLUTION] VERSION 1: REST API + IN-MEMORY STORAGE

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Thiết lập cấu trúc gói theo tính năng (`package quote`)
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang định hình các file DTO chia nhỏ cho List/Detail và viết logic lưu trữ tạm thời bằng `ConcurrentHashMap` trong `QuoteService`.
- [ ] BƯỚC TIẾP THEO (TƯƠNG LAI): Preparation cho Version 2 (Thay thế Map thành JPA/PostgreSQL).

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC BAN ĐẦU (FEATURE-BASED)

Dự án được tổ chức theo từng package tính năng (`quote/`), giúp dễ quản lý và cô lập phạm vi thay đổi khi nâng cấp version.

```text
quote-service/
└── src/main/java/com/example/quoteservice/
    ├── QuoteServiceApplication.java     <-- File chạy chính của Spring Boot
    └── quote/                           <-- Toàn bộ hệ sinh thái của UseCase Quote
        ├── controller/
        │   └── QuoteController.java     <-- Tiếp nhận HTTP Request (GET, POST, PUT)
        ├── service/
        │   └── QuoteService.java        <-- [⚡ ĐIỂM DỪNG] Chứa logic + In-Memory DB (Map/List)
        ├── dto/
        │   ├── QuoteCreateRequest.java  <-- Dữ liệu khi tạo mới một Quote
        │   ├── QuoteResponse.java       <-- Dữ liệu trả về chung khi thao tác thành công
        │   ├── QuoteDetailResponse.java <-- Dữ liệu chi tiết đầy đủ (Xem chi tiết 1 bản ghi)
        │   └── QuoteListItemResponse.java<-- Dữ liệu rút gọn phục vụ hiển thị ở màn hình danh sách
        └── model/
            ├── Quote.java               <-- Thực thể (POJO) định hình thuộc tính của Quote
            └── QuoteStatus.java         <-- Enum quản lý trạng thái (DRAFT, SUBMITTED, APPROVED...)