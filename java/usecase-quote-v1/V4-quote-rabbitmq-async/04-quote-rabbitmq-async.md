# [QUOTE-SERVICE-EVOLUTION] VERSION 4: ASYNC ELASTICSEARCH SYNC VIA RABBITMQ

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Giữ nguyên luồng Đọc danh sách tốc độ cao từ Elasticsearch (V3).
- [x] Giữ nguyên cấu trúc các file DTO đầu vào/đầu ra (V1).
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang gỡ bỏ đoạn code gọi trực tiếp `QuoteSearchRepository` bên trong `QuoteService`. Tiến hành cấu hình RabbitMQ (Exchange/Queue) và viết lớp Consumer chạy ngầm.
- [ ] BƯỚC TIẾP THEO (TƯƠNG LAI): Chuẩn bị cho Version 5 (Thiết lập Workflow giả lập trạng thái Submit/Approve tuần tự).

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC SAU KHI THÊM MESSAGE BROKER

Mã nguồn xuất hiện thêm gói `messaging/` để quản lý cấu trúc gửi/nhận tin nhắn bất đồng bộ, giúp giải phóng hoàn toàn áp lực cho tầng `service/` chính.

```text
quote-service/
└── src/main/java/com/example/quoteservice/
    ├── QuoteServiceApplication.java
    └── quote/
        ├── controller/              (GIỮ NGUYÊN)
        ├── dto/                     (GIỮ NGUYÊN)
        ├── model/                   (GIỮ NGUYÊN)
        ├── repository/              (GIỮ NGUYÊN)
        ├── search/                  
        │   ├── QuoteSearchRepository.java
        │   └── QuoteDoc.java
        ├── service/                 
        │   └── QuoteService.java    <-- [CẬP NHẬT] Giải phóng ES: Chỉ lưu PostgreSQL + Publish Message rồi thoát luôn
        └── messaging/               <-- [✨ PACKAGE MỚI] Đảm nhận luồng truyền tin bất đồng bộ
            ├── QuoteEventPublisher.java <-- Producer: Hàm đẩy tin nhắn vào RabbitMQ Exchange
            ├── QuoteEventConsumer.java  <-- Consumer: Worker chạy ngầm hứng tin nhắn để sync sang ES
            └── QuoteSyncMessage.java    <-- DTO bọc tin nhắn truyền đi (Ví dụ: Chứa ID, Action Type)

### C. Cấu trúc so sánh Code logic xử lý tại Service (V3 Sync vs V4 Async)

Để thấy rõ sự giải phóng áp lực cho luồng chạy API chính, hãy nhìn vào sự thay đổi trong cấu trúc viết code của hàm `createQuote`:

#### 🔴 Ở Version 3 (Đồng bộ - Trực tiếp)
Service phải gánh vác cả 2 DB, luồng xử lý bị nghẽn (Blocking) cho đến khi cả 2 lưu xong:

```java
@Transactional
public QuoteResponse createQuote(QuoteCreateRequest request) {
    // 1. Validation nghiệp vụ
    validationService.validate(request);
    
    // 2. Mapping DTO -> Entity
    Quote quote = modelMapper.map(request, Quote.class);
    
    // 3. Ghi vào PostgreSQL
    Quote savedQuote = quoteRepository.save(quote); 
    
    // 4. Đồng bộ trực tiếp sang Elasticsearch (ĐIỂM NGHẼN!)
    QuoteDoc doc = quoteSearchMapper.toDoc(savedQuote);
    quoteSearchRepository.save(doc); // Nếu ES phản hồi chậm, API sẽ bị treo/chậm theo
    
    // 5. Mapping sang Response DTO và trả về
    return modelMapper.map(savedQuote, QuoteResponse.class);
}

#### 🟢 Ở Version 4 (Bất đồng bộ - Qua Message)
 @Transactional
public QuoteResponse createQuote(QuoteCreateRequest request) {
    // 1. Validation nghiệp vụ (GIỮ NGUYÊN)
    validationService.validate(request);
    
    // 2. Mapping DTO -> Entity (GIỮ NGUYÊN)
    Quote quote = modelMapper.map(request, Quote.class);
    
    // 3. Ghi vào PostgreSQL (GIỮ NGUYÊN)
    Quote savedQuote = quoteRepository.save(quote); 
    
    // 4. Bắn sự kiện sang RabbitMQ rồi THOÁT LUỒNG (CẬP NHẬT MỚI)
    // Hệ thống chỉ mất khoảng 1-2ms để ném message vào hàng đợi rồi đi tiếp
    quoteEventPublisher.publishQuoteSyncEvent(
        new QuoteSyncMessage(savedQuote.getId(), "CREATE")
    );
    
    // 5. Mapping sang Response DTO và trả về (GIỮ NGUYÊN)
    return modelMapper.map(savedQuote, QuoteResponse.class);
}