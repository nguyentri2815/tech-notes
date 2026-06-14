```mermaid 
graph TD
    %% Luồng chạy hiện tại
    Client[Client / Postman] --> Controller[CustomerController]
    Controller --> Service[CustomerService]
    
    subgraph RAM_ZONE [Vùng dữ liệu tạm thời - Hiện tại]
        Service -->|Thao tác trực tiếp| MemDB[(In-Memory DB: Map / List)]
    end

    %% Luồng nâng cấp trong tương lai
    subgraph REAL_DB_ZONE [Vùng dữ liệu thật - Tương lai]
        Repo[CustomerRepository <br/> Interface mới] --> RealDB[(Database thật: MySQL)]
    end

    %% Điểm bóc tách
    MemDB -.->|🔴 ĐIỂM CẮT & THAY THẾ| Repo

    %% Đổ màu trạng thái
    style RAM_ZONE fill:#fff9c4,stroke:#fbc02d
    style REAL_DB_ZONE fill:#e8f5e9,stroke:#388e3c
    style MemDB fill:#ffcdd2,stroke:#c62828