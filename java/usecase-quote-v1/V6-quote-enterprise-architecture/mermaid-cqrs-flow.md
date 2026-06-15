```mermaid
graph TD
    Client[Client / Postman] -->|1. HTTP Request| Controller[QuoteController]

    %% XA LỘ QUERY (READ)
    subgraph XA_LO_QUERY [Xa Lộ Query - Đọc Dữ Liệu]
        Controller -->|2a. Search/List| Q_Service[QuoteSearchService]
        Q_Service --> ES_Repo[QuoteSearchRepository]
        ES_Repo --> ES[(Elasticsearch)]
        
        Controller -->|2b. Get Detail| Query_Service[QuoteQueryService]
        Query_Service --> SQL_Repo[QuoteRepository]
        SQL_Repo --> Postgres[(PostgreSQL)]
    end

    %% XA LỘ COMMAND (WRITE)
    subgraph XA_LO_COMMAND [Xa Lộ Command - Thay Đổi Dữ Liệu]
        Controller -->|2c. Create / Submit / Approve| Cmd_Service[QuoteCommandService]
        Cmd_Service -->|3. Gọi kiểm tra chéo| WorkflowService[WorkflowService]
        WorkflowService --> WF_Repo[WorkflowTaskRepository] --> Postgres
        
        Cmd_Service -->|4. Lưu thay đổi trạng thái| SQL_Repo
        Cmd_Service -->|5. Đẩy tin nhắn qua Mapper| Publisher[QuoteSyncPublisher]
        Publisher -->|6. Chớp nhoáng| RabbitMQ[[RabbitMQ]]
    end

    %% LUỒNG ĐỒNG BỘ NGẦM (BACKGROUND SYNC)
    subgraph SYNC_NGAM [Hạ Tầng Đồng Bộ Ngầm]
        RabbitMQ -.->|7| Consumer[QuoteSyncEsConsumer]
        Consumer -->|8. Tìm Entity gốc| QuoteQueryService
        Consumer -->|9. Đẩy chỉ mục qua IndexService| IndexService[QuoteIndexService]
        IndexService --> ES_Repo
    end

    %% Quy định màu sắc cấu trúc
    style XA_LO_QUERY fill:#e3fafc,stroke:#1098ad,stroke-width:2px
    style XA_LO_COMMAND fill:#fff0f6,stroke:#d6336c,stroke-width:2px
    style SYNC_NGAM fill:#fff9db,stroke:#f59f00
    style RabbitMQ fill:#ff8000,stroke:#cc6600