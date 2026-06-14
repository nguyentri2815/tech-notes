```mermaid
graph TD
    Client[Client / Postman] -->|Request DTO| Controller[CustomerController]
    Controller --> Service[CustomerService <br/> Chứa Validation & Mapping]
    
    subgraph GIAI_DOAN_1 [Giai đoạn 1 - Đã đập bỏ]
        Service -.->|Xóa bỏ| MemDB[(In-Memory DB: Map / List)]
    end

    subgraph GIAI_DOAN_2 [Giai đoạn 2 - Hiện tại]
        Service -->|1. Gọi hàm cứu/lưu dữ liệu| Repo[CustomerRepository <br/> Interface kế thừa JpaRepository]
        Repo -->|2. Tự sinh lệnh SQL ngầm| PostgreSQL[(Database thật: PostgreSQL)]
    end

    %% Định nghĩa màu sắc để nhận diện nhanh
    style GIAI_DOAN_1 fill:#ffec97,stroke:#f59f00
    style GIAI_DOAN_2 fill:#d3f9d8,stroke:#37b24d,stroke-width:2px
    style Repo fill:#74c0fc,stroke:#1971c2,stroke-width:2px
    style PostgreSQL fill:#93c5fd,stroke:#1d4ed8