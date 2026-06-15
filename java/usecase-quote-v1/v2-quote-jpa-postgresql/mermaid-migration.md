```mermaid
graph TD
    Client[Client / Postman] -->|POST / GET| Controller[QuoteController.java]
    Controller -->|Truyền DTO| Service[QuoteService.java <br/> Chứa Validation & Mapping]

    subgraph DATA_MIGRATION_ZONE [Tầng dữ liệu đã nâng cấp]
        Service -->|1. Gọi hàm cứu/ghi| Repo[QuoteRepository.java <br/> Kế thừa JpaRepository]
        Repo -->|2. Ánh xạ Object| Entity[Quote.java <br/> @Entity]
        Entity -.->|Đọc trạng thái dạng String| EnumStatus[QuoteStatus.java]
        Repo -->|3. Thực thi SQL ngầm| PostgreSQL[(Database thật: PostgreSQL)]
    end

    Service -->|4. Trả ra các DTO phù hợp| DTO_Output[QuoteDetail / ListItemResponse]
    DTO_Output --> Controller --> Client

    %% Chuẩn bị tâm thế cho Version 3
    PostgreSQL -.->|🔴 ĐIỂM SẼ ĐỒNG BỘ Ở V3/V4 <br/> Chuyển luồng đọc danh sách| ES[(Elasticsearch)]

    %% Định nghĩa màu sắc
    style DATA_MIGRATION_ZONE fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Repo fill:#74c0fc,stroke:#1971c2,stroke-width:2px
    style PostgreSQL fill:#93c5fd,stroke:#1d4ed8