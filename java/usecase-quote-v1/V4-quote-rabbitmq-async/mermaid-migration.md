```mermaid
graph TD
    Client[Client / Postman] -->|1. Create Request| Controller[QuoteController]
    Controller -->|2. Gọi nghiệp vụ| Service[QuoteService]
    
    subgraph LUONG_CHINH_API [Luồng Xử Lý Chính - Siêu Tốc]
        Service -->|3. Lưu vật lý cứng| Postgres[(PostgreSQL)]
        Service -->|4. Ném tin nhắn đi| Publisher[QuoteEventPublisher]
        Publisher -->|5. Đẩy rất nhanh| RabbitMQ[[RabbitMQ Broker]]
        Service -->|6. Trả kết quả ngay| Controller --> Return[Return Response]
    end

    %% Luồng chạy ngầm đứt đoạn, tách biệt hoàn toàn với API chính
    subgraph LUONG_CHAY_NGAM [Luồng Đồng Bộ Ngầm - Async Pipeline]
        RabbitMQ -.->|7. Tự động đẩy tin| Consumer[QuoteEventConsumer]
        Consumer -->|8. Chuyển đổi dữ liệu| ES_Repo[QuoteSearchRepository]
        ES_Repo -->|9. Chỉ mục hóa dữ liệu| ES[(Elasticsearch)]
    end

    %% Định nghĩa màu sắc hệ thống
    style LUONG_CHINH_API fill:#f0f4f8,stroke:#102a43
    style LUONG_CHAY_NGAM fill:#fef3c7,stroke:#d97706
    style RabbitMQ fill:#ff8000,stroke:#cc6600,stroke-width:2px
    style ES fill:#ff922b,stroke:#d9480f