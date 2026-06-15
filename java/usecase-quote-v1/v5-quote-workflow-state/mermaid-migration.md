```mermaid
graph TD
    Manager[Manager / User] -->|1. POST /quotes/id/approve| Controller[QuoteController]
    Controller -->|2. Gọi hàm approveQuote| Service[QuoteService]
    
    subgraph WORKFLOW_VALIDATION [Lớp Kiểm Tra Nghiệp Vụ Chuyển Trạng Thái]
        Service -->|3. Lấy dữ liệu cũ| Repo[QuoteRepository]
        Repo -->|4. Trả về Entity| Entity[Quote]
        Service -->|5. Kiểm tra logic| Verify{Trạng thái hiện tại <br/> có phải SUBMITTED?}
    end

    Verify -->|HỢP LỆ| Save_DB[6. Đổi status sang APPROVED <br/> Lưu PostgreSQL]
    Verify -->|SAI QUY TRÌNH| Throw[7. Ném lỗi: IllegalStateException]

    subgraph SYNC_SYSTEM_V4 [Hạ Tầng Giao Tiếp Đã Làm Ở V4]
        Save_DB -->|8. Phát Message| Publisher[QuoteEventPublisher]
        Publisher -->|9. Đẩy ngầm| RabbitMQ[[RabbitMQ]]
        RabbitMQ -.->|10. Hứng tin ngầm| Consumer[QuoteEventConsumer]
        Consumer -->|11. Ghi đè chỉ mục mới| ES[(Elasticsearch)]
    end

    Save_DB -->|12. Trả về thành công| Controller --> Client[Manager nhận kết quả OK]

    style WORKFLOW_VALIDATION fill:#f3f0ff,stroke:#7048e8
    style SYNC_SYSTEM_V4 fill:#f1f3f5,stroke:#495057
    style Verify fill:#fff3bf,stroke:#f59f00