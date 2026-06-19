# Tech Note — Ngày 39: Integration Test Elasticsearch cho Quote CQRS

> **Chủ đề:** Create / Submit / Approve → Workflow sync Elasticsearch → List API search đúng `status` / `productCode` / `keyword`  
> **Kiến trúc:** Event Sourcing + CQRS + Projection + Elasticsearch Read Model

---

## 1. DASHBOARD TIẾN ĐỘ

### ✅ Trạng thái tổng quan

```text
Status: DONE - Elasticsearch Integration Test đã đi qua flow Create/Submit/Approve
Scope : Command -> Event -> Flow/Workflow -> Elasticsearch -> Query API
Level : Integration Test, không còn chỉ mock SearchRepository
```

### ⚡ ĐIỂM DỪNG HIỆN TẠI

```text
Code hiện đang dừng tại tầng Query/Search verification:

CreateQuoteCommand
  -> QuoteCreatedEvent
  -> QuoteSyncWorkflow
  -> QuoteDocument(status=DRAFT)
  -> List API search status=DRAFT đúng

SubmitQuoteCommand
  -> QuoteSubmittedEvent
  -> QuoteSyncWorkflow
  -> QuoteDocument(status=SUBMITTED)
  -> List API search status=SUBMITTED đúng

ApproveQuoteCommand
  -> QuoteApprovedEvent
  -> QuoteSyncWorkflow
  -> QuoteDocument(status=APPROVED)
  -> List API search status=APPROVED đúng
```

**Điểm đã chứng minh:**

```text
[✓] Workflow sync ES chạy thật
[✓] Elasticsearch document được update theo event version mới
[✓] List API không đọc Aggregate
[✓] List API đọc Elasticsearch read model
[✓] Search filter theo status/productCode/keyword hoạt động
```

### 🎯 BƯỚC TIẾP THEO

```text
Ngày 40 — E2E + Runbook:
Command API -> EventStore -> Outbox -> Consumer -> Projection -> ES -> Query API
Mục tiêu: dựng checklist debug end-to-end khi quote_state hoặc ES bị lệch.
```

---

## 2. MÔ PHỎNG CÂY THƯ MỤC

```text
src/main/java/com/example/quoteservice
├── command
│   └── quote
│       ├── api
│       │   └── QuoteCommandController.java          // Command API: create/submit/approve
│       └── application
│           └── QuoteCommandService.java             // gọi AggregateRepository, append event
│
├── flow
│   └── quote
│       ├── workflow
│       │   └── QuoteSyncWorkflow.java               // [REFACTOR] nơi điều phối sync ES sau event
│       └── search
│           ├── QuoteIndexService.java               // [NEW/FOCUS] service ghi QuoteDocument vào ES
│           └── QuoteDocumentMapper.java             // [NEW/FOCUS] map quote_state/event -> QuoteDocument
│
├── query
│   └── quote
│       ├── api
│       │   └── QuoteQueryController.java            // Query API: list/detail/search
│       ├── application
│       │   └── QuoteQueryService.java               // [REFACTOR] list dùng ES thay vì DB state trực tiếp
│       └── search
│           ├── QuoteSearchRepository.java           // [NEW/FOCUS] ElasticsearchRepository
│           ├── QuoteSearchCriteria.java             // [NEW] status/productCode/keyword/page/sort
│           └── QuoteSearchResultMapper.java         // [NEW] map QuoteDocument -> ListItemResponse
│
├── readmodel
│   └── quote
│       └── state
│           ├── QuoteStateEntity.java                // DB read model/projection state
│           └── QuoteStateRepository.java            // projection source để rebuild/sync ES
│
└── shared
    └── search
        └── ElasticsearchTestConfig.java             // [TEST] cấu hình ES/Testcontainers nếu tách riêng

src/test/java/com/example/quoteservice
└── integration
    └── QuoteElasticsearchIntegrationTest.java       // [NEW/FOCUS] test Create/Submit/Approve -> ES -> List API
```

---

## 3. SƠ ĐỒ LUỒNG DỮ LIỆU

```mermaid
graph LR
    subgraph Command_Service["Command Service"]
        A[Command API<br/>Create / Submit / Approve]
        B[QuoteAggregate<br/>process(command)]
        C[DomainEvent<br/>Created / Submitted / Approved]
        D[(EventStore)]
    end

    subgraph Flow_Service["Flow Service"]
        E[Event Consumer / Test Dispatch]
        F[Projection Handler]
        G[(quote_state)]
        H[QuoteSyncWorkflow]
    end

    subgraph Search_Read_Model["Elasticsearch Read Model"]
        I[QuoteIndexService]
        J[(quote_index / QuoteDocument)]
    end

    subgraph Query_Service["Query Service"]
        K[QuoteQueryService]
        L[List API<br/>status/productCode/keyword]
    end

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    E --> H
    H --> I
    I --> J

    L --> K
    K --> J

    X[🔴 ĐIỂM THAY THẾ/NÂNG CẤP CHỐT YẾU:<br/>List API search chuyển sang đọc Elasticsearch<br/>thay vì đọc Aggregate/EventStore]:::hot
    K -.-> X
    X -.-> J

    classDef hot fill:#ffdddd,stroke:#cc0000,stroke-width:2px,color:#000;
```

---

## 4. CHI TIẾT SỰ DỊCH CHUYỂN LOGIC

### TRƯỚC ĐÓ — Query/List còn gần DB read model

```java
// QuoteQueryService.java - BEFORE
public Page<QuoteListItemResponse> search(QuoteSearchCriteria criteria) {
    Page<QuoteStateEntity> page = quoteStateRepository.search(
            criteria.getStatus(),
            criteria.getProductCode(),
            criteria.getKeyword(),
            criteria.toPageable()
    );

    return page.map(quoteStateMapper::toListItem);
}
```

### BÂY GIỜ — Query/List đọc Elasticsearch document

```java
// QuoteQueryService.java - NOW
public Page<QuoteListItemResponse> search(QuoteSearchCriteria criteria) {
    Page<QuoteDocument> page = quoteSearchRepository.search(
            criteria.getStatus(),
            criteria.getProductCode(),
            criteria.getKeyword(),
            criteria.toPageable()
    );

    return page.map(quoteSearchResultMapper::toListItem);
}
```

### Vì sao kiến trúc đổi?

```text
TRƯỚC:
  quote_state phù hợp cho detail/projection state,
  nhưng search/filter/keyword phức tạp sẽ bị giới hạn.

BÂY GIỜ:
  Elasticsearch trở thành read model chuyên dụng cho List/Search.
  Query Service không cần replay Aggregate, không đọc EventStore.
  Workflow chịu trách nhiệm sync QuoteDocument sau mỗi business event.
```

**Enterprise rule:**

```text
Command side quyết định sự thật nghiệp vụ.
Flow side dựng read model.
Query side đọc read model tối ưu cho màn hình.
```

---

## 5. QUY LUẬT ĐỌC LẠI 30 GIÂY

```text
Bước 1 - Nhìn DASHBOARD TIẾN ĐỘ
  -> Biết hôm nay đang dừng ở Integration Test Elasticsearch.

Bước 2 - Nhìn ĐIỂM DỪNG HIỆN TẠI
  -> Nhớ flow đã test: Create/Submit/Approve -> Workflow -> ES -> List API.

Bước 3 - Nhìn SƠ ĐỒ Mermaid
  -> Xác định ranh giới Command / Flow / Search Read Model / Query.

Bước 4 - Nhìn 🔴 ĐIỂM THAY THẾ/NÂNG CẤP CHỐT YẾU
  -> Nhớ thay đổi chính: List API đọc Elasticsearch.

Bước 5 - Nhìn code BEFORE/NOW
  -> Khôi phục nhanh file bị tác động mạnh nhất: QuoteQueryService.java.

Bước 6 - Nhìn BƯỚC TIẾP THEO
  -> Chuyển sang Ngày 40: E2E + Runbook debug toàn tuyến.
```

---

## GHI NHỚ 1 DÒNG

```text
Ngày 39 chứng minh Elasticsearch là Query Read Model thật:
Event xảy ra -> Workflow sync ES -> List API search đúng theo status/productCode/keyword.
```
