
## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)

[x] Tách biệt hoàn toàn luồng Đọc/Ghi tại tầng Service (QuoteCommandService và QuoteQueryService).

[x] Quy hoạch lại danh xưng cấu trúc dữ liệu (QuoteEntity cho SQL, QuoteDocument cho NoSQL).

[x] Đóng gói tập trung cấu trúc bổ trợ hệ thống (workflow/, common/).

[/] ĐIỂM DỪNG HIỆN TẠI: Đang tiến hành ánh xạ (Mapping) dữ liệu qua các Interface Mapper tập trung và cấu hình docker-compose.yml để chạy toàn bộ hạ tầng (PostgreSQL, ES, RabbitMQ).

[ ] BƯỚC TIẾP THEO: Khởi chạy môi trường Docker, kiểm thử tích hợp (Integration Test) toàn bộ các mắt xích.

🛠️ 2. MÔ PHỎNG CÂY THƯ MỤC ENTERPRISE (REFACTORED TREE)
Cấu trúc thư mục hiện tại đã được chuẩn hóa theo mô hình Enterprise, phân tách rạch ròi trách nhiệm của từng file:

## 🛠️ 2. MÔ PHỎNG CÂY THƯ MỤC ENTERPRISE (REFACTORED TREE)
Cấu trúc thư mục hiện tại đã được chuẩn hóa theo mô hình Enterprise, phân tách rạch ròi trách nhiệm của từng file:

quote-service/
├── docker-compose.yml               <-- Khởi tạo nhanh bộ 3: PostgreSQL, Elasticsearch, RabbitMQ
├── build.gradle                     <-- Quản lý Dependency tập trung (JPA, AMQP, Elasticsearch)
└── src/
    ├── main/resources/
    │   └── application.yml          <-- [CẬP NHẬT] Đổi từ .properties sang .yml để dễ quản lý cấu hình môi trường
    └── main/java/com/example/quoteservice/
        ├── QuoteServiceApplication.java
        │
        ├── quote/                   <-- BIẾN ĐỘNG LỚN: Phân rã luồng xử lý và quy hoạch lại danh xưng
        │   ├── controller/
        │   │   └── QuoteController.java
        │   ├── service/             
        │   │   ├── QuoteCommandService.java <-- [✨ MỚI] Chỉ lo xử lý nghiệp vụ Ghi (Create, Update, Submit, Approve)
        │   │   ├── QuoteQueryService.java   <-- [✨ MỚI] Chỉ lo xử lý nghiệp vụ Đọc (FindById từ PostgreSQL)
        │   │   └── QuoteIndexService.java   <-- [✨ MỚI] Cầu nối trung gian bọc logic lưu/xóa chỉ mục ES
        │   ├── dto/
        │   │   ├── QuoteCreateRequest.java
        │   │   ├── QuoteResponse.java
        │   │   ├── QuoteDetailResponse.java
        │   │   ├── QuoteListItemResponse.java
        │   │   └── QuoteSearchRequest.java  <-- [✨ MỚI] Object bọc các tham số tìm kiếm nâng cao (keyword, status...)
        │   ├── entity/
        │   │   ├── QuoteEntity.java         <-- [ĐỔI TÊN] Rõ ràng bản chất (Thay thế cho Quote.java cũ)
        │   │   └── QuoteStatus.java
        │   ├── repository/
        │   │   └── QuoteRepository.java     <-- Kết nối PostgreSQL
        │   ├── document/
        │   │   └── QuoteDocument.java       <-- [ĐỔI TÊN] Nằm riêng tại package dữ liệu NoSQL (Thay cho QuoteDoc.java)
        │   ├── search/
        │   │   ├── QuoteSearchRepository.java<-- Kết nối Elasticsearch
        │   │   └── QuoteSearchService.java  <-- [✨ MỚI] Điều khiển luồng query Search phức tạp từ ES
        │   ├── mapper/              
        │   │   ├── QuoteMapper.java         <-- [✨ MỚI] Interface chuyển đổi DTO <-> Entity (Bằng MapStruct/Thuần)
        │   │   └── QuoteSearchMapper.java   <-- [✨ MỚI] Interface chuyển đổi Entity -> Document của ES
        │   └── messaging/           
        │       ├── QuoteRabbitConfig.java   <-- [ĐỔI TÊN] Cấu hình hàng đợi tập trung
        │       ├── QuoteSyncMessage.java
        │       ├── QuoteSyncPublisher.java  <-- [ĐỔI TÊN] Chuẩn hóa danh xưng Producer đẩy tin
        │       └── QuoteSyncEsConsumer.java <-- [ĐỔI TÊN] Chuẩn hóa danh xưng Consumer nhận tin sync ES
        │
        ├── workflow/                <-- [✨ PACKAGE MỚI] Tách biệt nghiệp vụ kiểm soát phê duyệt/audit trail
        │   ├── entity/
        │   │   └── WorkflowTaskEntity.java  <-- Lưu lịch sử phê duyệt, người duyệt, thời gian duyệt
        │   ├── repository/
        │   │   └── WorkflowTaskRepository.java
        │   └── service/
        │       └── WorkflowService.java     <-- Xử lý ghi nhận log chuyển dịch trạng thái đơn hàng
        │
        └── common/                  <-- [✨ PACKAGE MỚI] Chứa hạ tầng dùng chung cho toàn hệ thống
            └── exception/
                ├── BusinessException.java   <-- Custom Exception để ném ra các lỗi nghiệp vụ
                └── GlobalExceptionHandler.java <-- Bộ lọc interceptor tự động bắt lỗi và trả về JSON chuẩn cho Client

                