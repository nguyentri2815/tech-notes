# [QUOTE-SERVICE-EVOLUTION] VERSION 5: STATE-DRIVEN WORKFLOW (SUBMIT / APPROVE)

## 📊 1. BẢNG ĐIỀU KHIỂN TRẠNG THÁI (DASHBOARD)
- [x] Giữ nguyên luồng ghi PostgreSQL (V2) và luồng đọc Elasticsearch (V3).
- [x] Giữ nguyên cơ chế đồng bộ bất đồng bộ qua RabbitMQ (V4).
- [/] ĐIỂM DỪNG HIỆN TẠI: Tiến hành bẻ nhánh logic cập nhật trạng thái đơn Quote. Xây dựng các hàm kiểm tra điều kiện chuyển đổi trạng thái (Transition Validation).
- [ ] BƯỚC TIẾP THEO (TƯƠNG LAI): Tối ưu hóa bảo mật phân quyền (RBAC/ABAC) - Chỉ có Manager mới được duyệt đơn từ trạng thái SUBMITTED.

---

## 🛠️ 2. MÔ PHỎNG CẤU TRÚC THƯ MỤC SAU KHI THÊM WORKFLOW

Mã nguồn không thêm cấu hình hạ tầng mà bổ sung package `workflow/` hoặc mở rộng trực tiếp trong các file hiện tại để hiện thực hóa quy trình kiểm soát trạng thái.

```text
quote-service/
└── src/main/java/com/example/quoteservice/
    ├── QuoteServiceApplication.java
    └── quote/
        ├── controller/              
        │   └── QuoteController.java     <-- [CẬP NHẬT] Thêm 2 Endpoint: POST /submit và POST /approve
        ├── service/                 
        │   └── QuoteService.java        <-- [CẬP NHẬT] Chứa logic cốt lõi điều khiển việc chuyển đổi trạng thái
        ├── model/                   
        │   ├── Quote.java               (GIỮ NGUYÊN cấu trúc - thuộc tính status thay đổi động)
        │   └── QuoteStatus.java         <-- [TỎA SÁNG] Định nghĩa danh sách trạng thái hợp lệ
        ├── dto/                     
        │   ├── QuoteCreateRequest.java  (GIỮ NGUYÊN)
        │   ├── QuoteResponse.java       (GIỮ NGUYÊN)
        │   ├── QuoteApprovalRequest.java<-- [✨ FILE MỚI] DTO nhận thông tin phê duyệt (note, action: APPROVE/REJECT)
        │   └── ...                      (Các DTO khác giữ nguyên)
        ├── repository/              (GIỮ NGUYÊN)
        └── messaging/               (GIỮ NGUYÊN - Vẫn phát tin nhắn sync sang ES mỗi khi status đổi)