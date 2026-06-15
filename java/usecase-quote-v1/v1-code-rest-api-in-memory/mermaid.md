```mermaid
graph TD
    Client[Client / Postman] -->|1. POST / Get List / Get Detail| Controller[QuoteController.java]
    
    subgraph DTO_Input_Layer [Luồng Nhận Dữ Liệu]
        Controller -->|Đọc body| ReqCreate[QuoteCreateRequest.java]
    end

    Controller -->|2. Chuyển DTO đã Validate| Service[QuoteService.java]

    subgraph Memory_Data_Store [Vùng Lưu Trữ Tạm Thời V1]
        Service -->|3. Thao tác RAM bằng Java Code| MemDB[(In-Memory DB: Map / List)]
        Service -->|4. Tạo / Đọc| Entity[Quote.java]
        Entity -.->|Đẩy vào / Lấy ra| MemDB
        Entity -.->|Đọc trạng thái| EnumStatus[QuoteStatus.java]
    end

    subgraph DTO_Output_Layer [Luồng Trả Dữ Liệu]
        Service -->|5a. Trả dữ liệu danh sách| ResList[QuoteListItemResponse.java]
        Service -->|5b. Trả dữ liệu chi tiết| ResDetail[QuoteDetailResponse.java]
        Service -->|5c. Trả về thông tin chung| ResGeneric[QuoteResponse.java]
    end

    DTO_Output_Layer -->|6. Trả về JSON| Controller
    Controller -->|7. HTTP Response Status| Client

    %% Đánh dấu điểm bóc tách cho Version sau
    MemDB -.->|🔴 ĐIỂM SẼ THAY THẾ Ở V2 <br/> Đổi sang PostgreSQL| Repository[QuoteRepository.java]

    %% Định nghĩa màu sắc
    style Memory_Data_Store fill:#fff9c4,stroke:#fbc02d
    style DTO_Output_Layer fill:#e1f5fe,stroke:#0288d1
    style MemDB fill:#ffcdd2,stroke:#c62828