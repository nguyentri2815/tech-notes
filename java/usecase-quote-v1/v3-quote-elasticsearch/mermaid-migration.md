```mermaid
graph TD
    Client[Client / Postman] -->|1. HTTP Request| Controller[QuoteController.java]
    Controller -->|2. Gọi hàm nghiệp vụ| Service[QuoteService.java]

    subgraph LUONG_GHI [Luồng Ghi - Write Pipeline]
        Service -->|3a. Tạo mới/Cập nhật| SQL_Repo[QuoteRepository]
        SQL_Repo -->|4a. Lưu vật lý| Postgres[(PostgreSQL)]
        
        %% Luồng đồng bộ tạm thời ở V3
        Service -->|3b. Đồng bộ trực tiếp TRONG CÙNG TRANSACTION| ES_Repo[QuoteSearchRepository]
        ES_Repo -->|4b. Chỉ mục hóa dữ liệu| ES[(Elasticsearch)]
    end

    subgraph LUONG_DOC [Luồng Đọc Danh Sách - Read Pipeline]
        Service -->|3c. Tìm kiếm / Phân trang / Tìm kiếm toàn văn| ES_Repo
        ES_Repo -->|4c. Truy vấn siêu tốc| ES
        ES -.->|5c. Trả về dạng Document| ES_Repo
    end

    %% Trả kết quả đầu ra
    Service -->|5. Đóng gói DTO tương ứng| DTO_Out[QuoteListItemResponse / DetailResponse]
    DTO_Out --> Controller --> Client

    %% Đánh dấu điểm nghẽn để giải quyết ở V4
    Service -.->|🔴 ĐIỂM NGHẼN HỆ THỐNG V3 <br/> Gọi đồng bộ 2 DB cùng lúc làm chậm luồng Ghi| RabbitMQ[Kiến trúc Async ở V4]

    %% Định nghĩa màu sắc
    style LUONG_GHI fill:#fff0f6,stroke:#d6336c
    style LUONG_DOC fill:#e3faffc,stroke:#1098ad,stroke-width:2px
    style ES fill:#ff922b,stroke:#d9480f
    style Postgres fill:#93c5fd,stroke:#1d4ed8