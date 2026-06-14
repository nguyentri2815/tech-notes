```mermaid
graph TD
    Client[Client / Postman] -->|1. Gửi Request Payload| Controller[CustomerController.java]
    
    subgraph DTO Layer [Lớp chuyển đổi dữ liệu]
        Controller -->|Validate dữ liệu| ReqCreate[CreateCustomerRequest.java]
        Controller -->|Validate dữ liệu| ReqUpdate[UpdateCustomerRequest.java]
    end

    Controller -->|2. Chuyển Request DTO| Service[CustomerService.java]

    subgraph Data Layer [Lớp Dữ Liệu & Đối Tượng]
        Service -->|3. Thao tác / Lưu trữ vào RAM| MapDB[(In-Memory DB: Map / List)]
        Service -->|4. Tạo / Cập nhật| Entity[Customer.java]
        Entity -.->|Đẩy vào| MapDB
    end

    Service -->|5. Chuyển đổi thành| Res[CustomerResponse.java]
    Res -->|6. Trả về JSON| Controller
    Controller -->|7. Phản hồi thành công| Client

    %% Định nghĩa màu sắc trực quan
    style MapDB fill:#ff9,stroke:#333,stroke-width:2px
    style DTOLayer fill:#e1f5fe,stroke:#0288d1
    style DataLayer fill:#f1f8e9,stroke:#558b2f