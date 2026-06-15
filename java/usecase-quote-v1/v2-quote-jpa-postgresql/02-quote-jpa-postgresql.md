# [QUOTE-SERVICE-EVOLUTION] VERSION 2: REST API + POSTGRESQL / JPA MIGRATION

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Giữ nguyên 100% cấu trúc: Controller, các file DTO (Request/Response)
- [x] Giữ nguyên tầng Validation và Mapping thủ công tại Service
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang thực hiện đập bỏ bộ nhớ tạm `ConcurrentHashMap` ở V1. Tiến hành cấu hình PostgreSQL và tạo tầng `Repository`.
- [ ] BƯỚC TIẾP THEO (TƯƠNG LAI): Chuẩn bị cho Version 3 (Tích hợp Elasticsearch cho luồng Read/Search).

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC SAU KHI NÂNG CẤP (MIGRATION TREE)

Hãy chú ý vào gói `repository/` mới được khai sinh và file `Quote.java` được tái cấu trúc thành Entity kết nối DB.

```text
quote-service/
└── src/main/java/com/example/quoteservice/
    ├── QuoteServiceApplication.java
    └── quote/
        ├── controller/              (GIỮ NGUYÊN)
        │   └── QuoteController.java
        ├── service/                 
        │   └── QuoteService.java    <-- [CẬP NHẬT] Xóa biến Map DB, tiêm QuoteRepository vào xử lý
        ├── repository/              <-- [✨ PACKAGE MỚI] Interface giao tiếp SQL tự động sinh lệnh
        │   └── QuoteRepository.java
        ├── dto/                     (GIỮ NGUYÊN 100%)
        │   ├── QuoteCreateRequest.java
        │   ├── QuoteResponse.java
        │   ├── QuoteDetailResponse.java
        │   └── QuoteListItemResponse.java
        └── model/                   
            ├── Quote.java           <-- [CẬP NHẬT] Gắn Annotation @Entity để mapping xuống bảng PostgreSQL
            └── QuoteStatus.java     <-- [CẬP NHẬT] Gắn Annotation Enum để lưu string vào DB (@Enumerated)