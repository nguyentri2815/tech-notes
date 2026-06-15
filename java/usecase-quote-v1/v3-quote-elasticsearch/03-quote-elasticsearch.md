# [QUOTE-SERVICE-EVOLUTION] VERSION 3: REST API + ELASTICSEARCH FOR LIST/SEARCH

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Giữ nguyên luồng Ghi (Write) vào PostgreSQL qua JPA.
- [x] Giữ nguyên cấu trúc `QuoteController` và các DTO đầu vào.
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang cấu hình Spring Data Elasticsearch. Tiến hành bẻ nhánh luồng Đọc danh sách/Tìm kiếm (List/Search) sang Elasticsearch thay vì chọc vào PostgreSQL.
- [ ] BƯỚC TIẾP THEO (TƯƠNG LAI): Nhận diện rủi ro nghẽn mạch khi Sync đồng bộ (Sync/Blocking) $\rightarrow$ Chuẩn bị cho Version 4 (Dùng RabbitMQ để sync Async).

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC SAU KHI TÍCH HỢP ES

Mã nguồn bắt đầu xuất hiện sự phân hóa rõ rệt ở tầng dữ liệu. Ta thêm package `search/` để cô lập các cấu trúc liên quan đến Elasticsearch.

```text
quote-service/
└── src/main/java/com/example/quoteservice/
    ├── QuoteServiceApplication.java
    └── quote/
        ├── controller/              (GIỮ NGUYÊN - Bổ sung endpoint search nếu cần)
        │   └── QuoteController.java
        ├── service/                 
        │   └── QuoteService.java    <-- [CẬP NHẬT] Điều hướng: Tạo/Sửa gọi cả JPA + ES; Tìm kiếm chỉ gọi ES
        ├── repository/              (GIỮ NGUYÊN)
        │   └── QuoteRepository.java <-- Giao tiếp PostgreSQL (Chuyên lo luồng GHI và đọc DETAIL)
        ├── search/                  <-- [✨ PACKAGE MỚI] Hệ sinh thái phục vụ tìm kiếm tốc độ cao
        │   ├── QuoteSearchRepository.java <-- Interface Spring Data Elasticsearch
        │   └── QuoteDoc.java        <-- Đối tượng ánh xạ thành Document trong Elasticsearch index
        ├── dto/                     (GIỮ NGUYÊN 100%)
        │   ├── QuoteCreateRequest.java
        │   ├── QuoteResponse.java
        │   ├── QuoteDetailResponse.java
        │   └── QuoteListItemResponse.java <-- [TỎA SÁNG] Map trực tiếp từ dữ liệu lấy từ ES ra
        └── model/                   (GIỮ NGUYÊN)
            ├── Quote.java
            └── QuoteStatus.java