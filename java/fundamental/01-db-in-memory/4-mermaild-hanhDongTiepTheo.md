```mermaid 
graph TD
    %% Kiến trúc Giai đoạn 1 (Hiện tại của bạn)
    subgraph Giai_Doan_1 [Giai đoạn 1: In-Memory]
        Service1[CustomerService.java] -->|Thao tác trực tiếp| MemDB[(RAM: Map / List)]
    end

    %% Kiến trúc Giai đoạn 2 (Khi lên DB thật)
    subgraph Giai_Doan_2 [Giai đoạn 2: Database Thật]
        Service2[CustomerService.java] -->|1. Gọi hàm cứu dữ liệu| Repo[CustomerRepository.java <br/> Interface / Class mới]
        Repo -->|2. Tự sinh lệnh SQL| RealDB[(Database Thật: <br/> MySQL / PostgreSQL)]
    end

    %% Chỉ ra điểm thay đổi
    MemDB -.->|Điểm bóc tách và thay thế| Repo

    style MemDB fill:#f9f,stroke:#333
    style Repo fill:#bbf,stroke:#333,stroke-width:2px
    style RealDB fill:#bfb,stroke:#333