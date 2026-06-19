# Tech Note — Ngày 45: Kafka Integration Test cho Outbox → Kafka → Consumer → Projection/DLT

> **Chủ đề:** Event Sourcing / CQRS nâng cao  
> **Vai trò kiến trúc:** Software Architect  
> **Mục tiêu đọc lại:** nắm ngữ cảnh trong **30 giây**

---

## 1. DASHBOARD TIẾN ĐỘ

| Hạng mục | Trạng thái |
|---|---|
| Event Store + Outbox | ✅ Đã có |
| Kafka Producer thật | ✅ `OutboxMessagePublisher -> KafkaTemplate` |
| Kafka Consumer thật | ✅ `@KafkaListener` consume `quote-events` |
| Projection `quote_state` | ✅ Consumer update qua Kafka thật |
| Idempotency | ✅ `processed_messages` |
| Retry + DLT | ✅ Retry lỗi, đẩy sang `quote-events-dlt` |
| Integration Test | ✅ Test chạy qua Kafka broker thật |
| CDC/Debezium | ⏭️ Chưa làm, là bước tiếp theo |

### ⚡ ĐIỂM DỪNG HIỆN TẠI

```text
Đang dừng ở trạng thái:

Command/Aggregate tạo event
  -> ghi event_store
  -> ghi outbox_events
  -> OutboxMessagePublisher publish Kafka thật
  -> QuoteKafkaConsumer consume Kafka thật
  -> DomainEventMessageProcessor dispatch handler
  -> quote_state update đúng version
  -> processed_messages ghi nhận message đã xử lý

Nếu consumer lỗi:
  -> retry theo Kafka DefaultErrorHandler
  -> sau retry vẫn lỗi thì message vào quote-events-dlt
```

**File trọng tâm hôm nay:**

```text
QuoteKafkaIntegrationTest.java
```

Lý do: test này chứng minh flow không còn là unit/integration giả lập bằng cách gọi consumer trực tiếp, mà đã đi qua Kafka broker thật.

### 🎯 BƯỚC TIẾP THEO

```text
Ngày 46:
Hiểu CDC/Debezium architecture
  -> thay OutboxPublisher polling bằng Debezium đọc outbox_events từ PostgreSQL WAL
  -> Debezium/Kafka Connect publish Kafka
```

**Điểm nâng cấp tiếp theo:**

```text
Hiện tại:
  App tự polling outbox_events rồi publish Kafka

Mục tiêu tiếp theo:
  App chỉ ghi outbox_events
  Debezium CDC tự đọc DB log và publish Kafka
```

---

## 2. MÔ PHỎNG CÂY THƯ MỤC

```text
src/
└── main/
    └── java/com/example/quoteservice/
        ├── command/
        │   └── quote/
        │       └── infrastructure/
        │           ├── outbox/
        │           │   ├── OutboxMessagePublisher.java        // [EXISTING][TESTED] đọc outbox PENDING, publish Kafka, mark SENT
        │           │   ├── OutboxEventEntity.java             // [EXISTING] dữ liệu outbox event
        │           │   └── OutboxEventRepository.java         // [EXISTING] query outbox_events
        │           └── kafka/
        │               └── KafkaDomainEventPublisher.java     // [NEW-D42] wrapper KafkaTemplate.send(topic, key, value)
        │
        ├── flow/
        │   └── quote/
        │       ├── consumer/
        │       │   ├── DomainEventMessageProcessor.java       // [REFACTOR-D43/D45] xử lý message chung: dedup -> deserialize -> dispatch -> markProcessed
        │       │   ├── KafkaPoisonMessageTestHook.java        // [NEW-D45] test hook để ép lỗi DLT sạch hơn
        │       │   └── NoopKafkaPoisonMessageTestHook.java    // [NEW-D45] production no-op implementation
        │       └── consumer/kafka/
        │           ├── QuoteKafkaConsumer.java                // [NEW-D43][TESTED] @KafkaListener đọc quote-events
        │           └── QuoteKafkaDltConsumer.java             // [NEW-D44] đọc quote-events-dlt để log/debug
        │
        ├── readmodel/
        │   └── quote/state/
        │       ├── QuoteStateEntity.java                      // [EXISTING][ASSERTED] read model được projection update
        │       └── QuoteStateRepository.java                  // [EXISTING][ASSERTED] assert DRAFT/SUBMITTED + version
        │
        └── shared/
            └── messaging/
                ├── DomainEventMessage.java                    // [EXISTING] message chuẩn đưa qua Kafka
                ├── dedup/
                │   ├── ProcessedMessageEntity.java            // [EXISTING][ASSERTED] chống xử lý trùng
                │   └── ProcessedMessageRepository.java        // [EXISTING][ASSERTED] kiểm tra message đã xử lý
                └── kafka/
                    ├── QuoteKafkaTopicNames.java              // [UPDATED-D44] quote-events + quote-events-dlt
                    ├── KafkaTopicConfig.java                  // [NEW-D45] auto-create topic cho local/test
                    └── KafkaConsumerErrorHandlerConfig.java   // [NEW-D44][TESTED] DefaultErrorHandler + DLT recoverer

└── test/
    ├── java/com/example/quoteservice/kafka/
    │   └── QuoteKafkaIntegrationTest.java                     // [NEW-D45][CORE] KafkaContainer + PostgreSQLContainer + Awaitility
    └── resources/
        └── application-test.yml                               // [NEW/UPDATED-D45] group-id test, retry nhanh, disable scheduled publisher
```

---

## 3. SƠ ĐỒ LUỒNG DỮ LIỆU

```mermaid
graph LR
    subgraph CMD[Command Side]
        API[Command API\nCreate/Submit/Approve]
        AGG[QuoteAggregate\nprocess command -> event]
        OUTPUB[OutboxMessagePublisher\nPENDING -> Kafka]
        API --> AGG
    end

    subgraph DB[PostgreSQL]
        ES[(event_store)]
        OUT[(outbox_events)]
        STATE[(quote_state)]
        PM[(processed_messages)]
    end

    subgraph KAFKA[Kafka Broker]
        TOPIC[topic: quote-events\nkey = aggregateId]
        DLT[topic: quote-events-dlt]
    end

    subgraph FLOW[Flow Service]
        CONSUMER[QuoteKafkaConsumer\n@KafkaListener]
        PROCESSOR[DomainEventMessageProcessor\ndedup -> deserialize -> dispatch]
        HANDLER[Projection / Workflow Handlers]
        ERROR[DefaultErrorHandler\nretry -> DLT]
    end

    subgraph TEST[Integration Test Boundary]
        TESTCASE[QuoteKafkaIntegrationTest\nKafkaContainer + PostgreSQLContainer + Awaitility]
    end

    AGG --> ES
    AGG --> OUT
    OUT --> OUTPUB
    OUTPUB --> TOPIC
    TOPIC --> CONSUMER
    CONSUMER --> PROCESSOR
    PROCESSOR --> HANDLER
    HANDLER --> STATE
    PROCESSOR --> PM
    CONSUMER -. failure .-> ERROR
    ERROR -. after retries .-> DLT
    TESTCASE -. assert async .-> STATE
    TESTCASE -. assert dedup .-> PM
    TESTCASE -. assert failure path .-> DLT

    OUTPUB:::hotspot
    TOPIC:::hotspot
    TESTCASE:::hotspot

    classDef hotspot fill:#ffdddd,stroke:#cc0000,stroke-width:2px,color:#000;
```

### 🔴 ĐIỂM THAY THẾ/NÂNG CẤP CHỐT YẾU

```text
Trước Ngày 45:
  Test gọi consumer trực tiếp -> chưa chứng minh Kafka thật.

Ngày 45:
  Test đi qua Kafka broker thật -> chứng minh Producer/Topic/Consumer/Retry/DLT hoạt động.

Bước nâng cấp tiếp theo:
  OutboxMessagePublisher polling -> Debezium CDC đọc PostgreSQL WAL.
```

---

## 4. CHI TIẾT SỰ DỊCH CHUYỂN LOGIC

**File bị tác động mạnh nhất:**

```text
src/test/java/com/example/quoteservice/kafka/QuoteKafkaIntegrationTest.java
```

### TRƯỚC ĐÓ — test còn bypass Kafka

```java
// Bài cũ: test xử lý event bằng cách gọi consumer trực tiếp
DomainEventMessage message = buildMessageFromOutbox(outboxEvent);

rabbitMqDomainEventConsumer.consume(message);

QuoteStateEntity state = quoteStateRepository.findById(quoteId)
        .orElseThrow();

assertThat(state.getStatus()).isEqualTo(QuoteStatus.DRAFT);
assertThat(state.getLastProjectedVersion()).isEqualTo(1L);
```

**Vấn đề kiến trúc:**

```text
Test này chỉ chứng minh processor/projection chạy được.
Nó chưa chứng minh Kafka producer, topic, consumer group, offset, retry, DLT hoạt động.
```

### BÂY GIỜ — test đi qua Kafka thật

```java
// Ngày 45: test đi qua Kafka broker thật
AggregateCommandResult<QuoteAggregate> created =
        quoteAggregateRepository.create(createCommand());

String quoteId = created.getAggregateId();

// Publish outbox PENDING vào Kafka thật
outboxMessagePublisher.publishPendingEvents();

// Consumer async tự consume từ Kafka topic quote-events
await().atMost(10, SECONDS)
        .untilAsserted(() -> {
            QuoteStateEntity state = quoteStateRepository.findById(quoteId)
                    .orElseThrow();

            assertThat(state.getStatus().name()).isEqualTo("DRAFT");
            assertThat(state.getLastProjectedVersion()).isEqualTo(1L);
        });

await().atMost(10, SECONDS)
        .untilAsserted(() -> {
            assertThat(processedMessageRepository.count()).isEqualTo(1L);
        });
```

### BÂY GIỜ — test DLT path

```java
// Ép lỗi trong processor bằng test hook
doThrow(new RuntimeException("Forced failure for DLT test"))
        .when(poisonMessageTestHook)
        .beforeProcess(argThat(message ->
                "poison-001".equals(message.getEventId())
        ));

DomainEventMessage poisonMessage = new DomainEventMessage(
        "poison-001",
        "FAIL_TEST",
        "Quote",
        "QuoteSubmittedEvent",
        "{}",
        2L,
        "test-dlt-correlation",
        LocalDateTime.now()
);

kafkaTemplate.send(
        QuoteKafkaTopicNames.QUOTE_EVENTS,
        poisonMessage.getAggregateId(),
        poisonMessage
).get();

await().atMost(20, SECONDS)
        .untilAsserted(() -> {
            ConsumerRecord<String, DomainEventMessage> record =
                    pollOneRecord(dltConsumer);

            assertThat(record).isNotNull();
            assertThat(record.key()).isEqualTo("FAIL_TEST");
            assertThat(record.value().getEventId()).isEqualTo("poison-001");
        });
```

### Vì sao kiến trúc đổi?

```text
1. CQRS/Event-driven system là async system.
2. Test đúng phải kiểm tra đường đi thật của event qua message broker.
3. Direct method call che mất lỗi Kafka config, serializer, topic, consumer group, retry, DLT.
4. Kafka integration test là lớp bảo vệ kiến trúc trước khi nâng cấp sang CDC/Debezium.
```

---

## 5. QUY LUẬT ĐỌC LẠI 30 GIÂY

Khi mở lại file này, đọc theo thứ tự:

```text
1. Nhìn DASHBOARD TIẾN ĐỘ
   -> biết hệ thống đang xong phần nào, còn thiếu phần nào.

2. Nhìn ⚡ ĐIỂM DỪNG HIỆN TẠI
   -> biết code đang dừng ở flow OutboxPublisher -> Kafka -> Consumer -> Projection/DLT.

3. Nhìn Mermaid FLOW
   -> khôi phục topology: Command / DB / Kafka / Flow / Test Boundary.

4. Nhìn 🔴 ĐIỂM THAY THẾ/NÂNG CẤP CHỐT YẾU
   -> nhớ trọng tâm: bỏ direct consumer call, test qua Kafka thật; bước sau thay OutboxPublisher bằng Debezium.

5. Nhìn code TRƯỚC ĐÓ vs BÂY GIỜ
   -> nhớ chính xác logic test đã dịch chuyển từ direct call sang broker-mediated integration test.
```

**Câu chốt để nhớ:**

```text
Ngày 45 không thêm business mới.
Ngày 45 nâng độ thật của kiến trúc test:
  từ gọi consumer trực tiếp
  sang Outbox -> Kafka Broker -> Consumer -> Projection/DLT.
```

---

## 6. NEXT CONTEXT TOKEN

```text
NEXT:
Ngày 46/47/48 sẽ thay nguồn publish Kafka.
Không còn OutboxPublisher polling nữa.
Debezium sẽ đọc outbox_events từ PostgreSQL WAL và publish Kafka.

Điểm cần giữ nguyên:
  QuoteKafkaConsumer
  DomainEventMessageProcessor
  Projection/Workflow handlers
  processed_messages

Điểm sẽ thay:
  OutboxMessagePublisher
  KafkaDomainEventPublisher trong command app
```
