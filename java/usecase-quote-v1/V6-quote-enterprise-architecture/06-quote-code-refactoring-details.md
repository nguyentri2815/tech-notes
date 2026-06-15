# [REFAC-DETAILS] CHI TIẾT DỊCH CHUYỂN CODE LOGIC (VERSION 5 -> VERSION 6)

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Hiểu tư duy phân rã God Class thành cặp Command/Query.
- [/] ĐIỂM DỪNG HIỆN TẠI: Đang đối chiếu cấu trúc code cũ (V5) để bóc tách sang cấu trúc Enterprise (V6).
- [ ] BƯỚC TIẾP THEO: Viết các Interface Mapper (MapStruct) và cấu hình Transaction Isolation Level phù hợp cho từng Service.

---

## 💻 2. CHI TIẾT BÓC TÁCH CODE TẠI TẦNG SERVICE (TỪ V5 SANG V6)

### A. Luồng Thay Đổi Dữ Liệu (Write Pipeline) -> Chuyển vào `QuoteCommandService`
Toàn bộ các hàm `create`, `update`, `submit`, `approve` được gom về đây. Điểm cốt lõi là **sự xuất hiện của `WorkflowService`** để gánh bớt trách nhiệm lưu Log/Audit Trail.

#### 🔴 Trước đó (Version 5 - Code ôm đồm trong một Service)
```java
@Service
@RequiredArgsConstructor
@Transactional
public class QuoteService { // Luồng Ghi và Luồng Đọc dính chung một file
    private final QuoteRepository quoteRepository;
    private final WorkflowTaskRepository workflowTaskRepository; // Tiêm trực tiếp repository của module khác (Sai nguyên tắc cô lập)

    public QuoteResponse approveQuote(Long id, QuoteApprovalRequest request) {
        Quote quote = quoteRepository.findById(id).orElseThrow();
        if (quote.getStatus() != QuoteStatus.SUBMITTED) throw new IllegalStateException();

        quote.setStatus(QuoteStatus.APPROVED);
        Quote updatedQuote = quoteRepository.save(quote);

        // Logic lưu lịch sử workflow viết thô thiển ngay tại đây
        WorkflowTaskEntity task = new WorkflowTaskEntity();
        task.setQuoteId(id);
        task.setAction("APPROVE");
        task.setOperator("MANAGER");
        workflowTaskRepository.save(task); 

        quoteSyncPublisher.publish(new QuoteSyncMessage(id, "UPDATE"));
        return modelMapper.map(updatedQuote, QuoteResponse.class);
    }
}

---

#### 🟢 Bây giờ (Version 6 - Chuẩn Enterprise Architecture)

```java
@Service
@RequiredArgsConstructor
@Transactional // Đảm bảo tính nhất quán (Atomicity): Lỗi ở Workflow thì SQL rollback luôn
public class QuoteCommandService {
    private final QuoteRepository quoteRepository;
    private final WorkflowService workflowService; // Gọi qua Service của Module Workflow (Đúng chuẩn cô lập)
    private final QuoteSyncPublisher quoteSyncPublisher;
    private final QuoteMapper quoteMapper; // Thay modelMapper bằng Mapper chuyên biệt

    public QuoteResponse approveQuote(Long id, QuoteApprovalRequest request) {
        // 1. Tìm kiếm thực thể gốc
        QuoteEntity quote = quoteRepository.findById(id)
            .orElseThrow(() -> new BusinessException("QUOTE_NOT_FOUND"));

        // 2. Kiểm tra quy trình nghiệp vụ (Chuyển sang dùng Custom BusinessException)
        if (quote.getStatus() != QuoteStatus.SUBMITTED) {
            throw new BusinessException("INVALID_WORKFLOW_STATE");
        }

        // 3. Thực thi thay đổi trạng thái
        quote.setStatus(QuoteStatus.APPROVED);
        QuoteEntity updatedQuote = quoteRepository.save(quote);

        // 4. Ủy quyền ghi nhận lịch sử cho Workflow Module
        workflowService.logTrace(id, "APPROVE", "MANAGER", request.getNote());

        // 5. Bắn tín hiệu đồng bộ qua Message Broker
        quoteSyncPublisher.publishQuoteSyncEvent(new QuoteSyncMessage(updatedQuote.getId(), "UPDATE"));

        return quoteMapper.toResponse(updatedQuote);
    }
}

B. Luồng Truy Vấn Dữ Liệu (Read Pipeline) -> Chuyển vào QuoteQueryService
File này được giải phóng hoàn toàn khỏi logic nghiệp vụ phức tạp, chỉ tập trung tối ưu tốc độ đọc dữ liệu từ SQL để phục vụ màn hình xem chi tiết (Detail).

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // Tối ưu hiệu năng: Hibernate bỏ qua kiểm tra thay đổi dữ liệu (Dirty Checking)
public class QuoteQueryService {
    private final QuoteRepository quoteRepository;
    private final QuoteMapper quoteMapper;

    public QuoteDetailResponse getQuoteDetail(Long id) {
        QuoteEntity quote = quoteRepository.findById(id)
            .orElseThrow(() -> new BusinessException("QUOTE_NOT_FOUND"));
        return quoteMapper.toDetailResponse(quote);
    }
}

🛡️ 4. XỬ LÝ LỖI TẬP TRUNG TẠI GÓI common/exception
Thay vì viết các khối try-catch thủ công ở Controller gây bẩn code, ta cấu hình bộ lọc toàn cục để hệ thống tự động bốc lỗi nghiệp vụ ra trả về cho Client:

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        // Tự động bắt các lỗi như "INVALID_WORKFLOW_STATE" và bọc lại thành JSON sạch sẽ
        ErrorResponse error = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }
}

