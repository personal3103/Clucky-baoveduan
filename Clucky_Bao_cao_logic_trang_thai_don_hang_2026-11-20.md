# BÁO CÁO KỸ THUẬT: ĐIỀU PHỐI VÀ KIỂM SOÁT TRẠNG THÁI ĐƠN HÀNG THEO TỪNG LỰA CHỌN TRONG DỰ ÁN CLUCKY

> **Dự án:** Clucky – Nền tảng Đặt Món Ăn Nhanh (Vietnamese Fast-Food Ordering Platform)  
> **Chủ đề báo cáo:** Kiến trúc & Giải pháp kiểm soát trạng thái đơn hàng theo kênh (Online/Offline), hình thức nhận món (Giao tận nơi/Mang về/Ăn tại quầy) và phương thức thanh toán (COD/VietQR/Tiền mặt/Kết hợp).  
> **Ngày lập báo cáo:** 20/11/2026  
> **Mã nguồn tham chiếu:** `backend/cart-order-service`, `backend/payment-service`, `docs/sprint3-phases/`, `docs/Khung_thuyet_trinh_logic_dat_hang_Clucky_UI.docx`.

---

## 1. TỔNG QUAN & TRIẾT LÝ THIẾT KẾ CỐT LÕI (Core Architectural Insight)

Trong các hệ thống đặt món F&B thông thường, lỗi nghiệp vụ phổ biến nhất là cố gắng nhồi nhét tất cả quy trình vào một máy trạng thái (State Machine) tuyến tính duy nhất. Điều này dẫn đến sự cố nghiêm trọng:
- Đơn giao hàng bị nhân viên quầy tự tiện bấm hoàn tất khi shipper chưa giao.
- Khách đặt online chuyển khoản chưa thanh toán nhưng bếp đã chiên gà làm lãng phí nguyên vật liệu.
- Đơn tự lấy hoặc ăn tại quầy bị bỏ quên khâu thu tiền mặt trước khi giao thức ăn.

Tại **Clucky**, bài toán này được giải quyết triệt để thông qua triết lý: **Tách đôi hai trục trạng thái độc lập** và kiểm soát bằng **Chốt chặn điều kiện bàn giao (Handover Gates)**:

1. **Trục Tiến độ Chế biến & Bàn giao (`status`)**:
   `PENDING_STOCK` → `PENDING_PAYMENT` / `CONFIRMED` → `PREPARING` → `READY` → `SHIPPING` → `COMPLETED` (hoặc `CANCELLED`, `DELIVERY_FAILED`).
2. **Trục Dòng tiền & Tài chính (`paymentStatus`)**:
   `PENDING` → `PAID` (ngân hàng xác nhận) hoặc `COLLECTED` (tiền mặt vào két / đối soát xong).

> **Quy tắc vàng:**  
> *Tiến độ chế biến của Bếp có thể đi trước Dòng tiền (tùy vào kênh đặt và độ tin cậy để tối ưu tốc độ ra món), nhưng **Thời điểm Bàn giao cho khách (`COMPLETED`) bắt buộc phải hội tụ: Tiền phải vào két hoặc tài khoản thì hàng mới được xuất đi!***

---

## 2. MA TRẬN 4 CHIỀU PHÂN NHÁNH NGHIỆP VỤ

Mỗi đơn hàng được định danh và điều phối xuyên suốt qua 4 chiều dữ liệu:

| Chiều phân nhánh | Các giá trị hỗ trợ | Ý nghĩa nghiệp vụ |
| :--- | :--- | :--- |
| **1. Nguồn tạo (`orderSource`)** | `WEB`, `COUNTER` | Phân định khách hàng tự đặt trên web (`WEB`) hay nhân viên thu ngân nhập tại quầy POS (`COUNTER`). |
| **2. Hình thức nhận (`type` / `diningOption`)** | `DELIVERY`, `PICKUP` / `TAKE_AWAY`, `DINE_IN` | Giao hàng tận nơi (`DELIVERY` - chỉ có trên Web), Đặt trước mang về (`PICKUP`), Ăn tại chỗ (`DINE_IN` - cấp `queueNumber` từ 1-999). |
| **3. Phương thức thanh toán (`paymentMethod`)** | `COD`, `BANK_TRANSFER`, `CASH`, `SPLIT` | Trả sau khi nhận (`COD`), Chuyển khoản VietQR tự động qua SePay (`BANK_TRANSFER`), Tiền mặt tại quầy (`CASH`), Kết hợp Tiền mặt + VietQR (`SPLIT`). |
| **4. Thẩm quyền quyết định (Actor Authority)** | `STAFF`, `SHIPPING_SIMULATOR`, `SYSTEM_PAYMENT` | Phân định rõ ai có quyền chuyển trạng thái: Nhân viên chỉ chuyển bước bếp, Vận chuyển độc quyền chuyển `SHIPPING`/`COMPLETED`, Ngân hàng độc quyền cập nhật `PAID`. |

---

## 3. PHÂN TÍCH CHI TIẾT TỪNG NHÁNH NGHIỆP VỤ

### 3.1. Nhánh Online - Giao tận nơi (Web Delivery) + Chuyển khoản VietQR
* **Khởi tạo:** `status = PENDING_PAYMENT`, `paymentStatus = PENDING`.
* **Luồng xử lý:**
  - Khách chưa trả tiền thì quán **chưa nấu**. Trên giao diện khách hiển thị khung mã VietQR động cùng đồng hồ đếm ngược 15 phút.
  - Khách quét mã chuyển khoản → Webhook SePay gửi thông báo về `payment-service`.
  - `OrderPaymentService.decide()` xác thực snapshot số tiền, mã đơn → Tự động cập nhật `paymentStatus = PAID` và đẩy `status = CONFIRMED`.
  - Đơn xuất hiện trên màn hình Bếp: Nhân viên chuyển `CONFIRMED` → `PREPARING` → `READY`.
  - Khi món đã sẵn sàng, shipper mô phỏng nhận hàng → `ShippingOrderProjection` chuyển `SHIPPING`. Khi giao tận tay khách có bằng chứng giao hàng → Tự động chuyển `COMPLETED`.
  - Nhân viên quầy bị **chặn tuyệt đối** không thể bấm hoàn tất thủ công cho đơn giao hàng.

### 3.2. Nhánh Online - Giao tận nơi (Web Delivery) + Thu tiền khi nhận hàng (COD)
* **Khởi tạo:** `status = CONFIRMED`, `paymentStatus = PENDING`.
* **Luồng xử lý:**
  - Khách được bảo chứng làm món ngay. Bếp nhận đơn và chế biến: `CONFIRMED` → `PREPARING` → `READY`.
  - **Bài toán tài chính tinh tế:** Khi shipper đến lấy đơn, hệ thống ghi nhận `merchantPaymentStatus = PAID_SIMULATED` (mô phỏng shipper ứng tiền hàng lại cho chi nhánh trước khi rời quán).
  - Khi giao hàng thành công, `ShippingOrderProjection` cập nhật `status = COMPLETED`. Tiền mặt khách trả shipper nằm ngoài hệ thống Clucky (`paymentStatus` của khách vẫn `PENDING` theo đúng nguyên tắc kế toán COD).

### 3.3. Nhánh Online - Đặt trước tự đến lấy (Web Pickup) + Tiền mặt tại quầy
* **Khởi tạo:** `status = CONFIRMED`, `paymentStatus = PENDING`.
* **Luồng xử lý:**
  - Đơn tự lấy được ưu tiên vào bếp làm trước (`CONFIRMED` → `PREPARING` → `READY`) để khi khách bước vào quán là có món nóng hổi ngay.
  - **Điểm chốt chặn Handover Gate:**
    * Khi món đã `READY`, khách đến quầy nhận món. Lúc này `paymentStatus` vẫn là `PENDING`.
    * Trên giao diện Staff, nút "Xác nhận khách đã nhận" bị ẩn/khóa; thay vào đó là nút màu cam: **"Thu tiền quầy"**.
    * Nhân viên bắt buộc phải mở modal thu tiền mặt → lưu biên lai `payment-service` → đơn chuyển sang `paymentStatus = COLLECTED`.
    * Ngay khi tiền đã vào két (`COLLECTED`), nút bàn giao mới sáng lên để nhân viên đưa đơn về `COMPLETED`. Nếu nhân viên cố tình gọi API kết thúc đơn, backend ném ngay lỗi **HTTP 409 Conflict**: *"Đơn hàng chưa được thanh toán đầy đủ trước khi bàn giao."*

### 3.4. Nhánh Offline - Tại quầy POS (Counter Order) + Tiền mặt / VietQR / Kết hợp (Split)
* **Khởi tạo:**
  - Tự động cấp số thứ tự gọi món `queueNumber` (1–999) tăng dần mỗi ngày theo từng chi nhánh qua Redis `INCR`.
  - Đơn vào thẳng `status = CONFIRMED` để in phiếu chế biến KOT (Kitchen Order Ticket) cho bếp.
* **Linh hoạt thời điểm thu tiền:**
  - Thu ngân có thể thu tiền ngay lúc tạo đơn hoặc để bếp nấu rồi thu tiền lúc bàn giao.
* **Ba phương thức thanh toán tại quầy:**
  1. **Tiền mặt (CASH):** Thu ngân nhận tiền → hệ thống tính tiền thối tự động → in bill nhiệt K80 → cập nhật `paymentStatus = COLLECTED`.
  2. **VietQR:** Mở mã SePay POS, hệ thống polling mỗi 6 giây chờ webhook ngân hàng thật → cập nhật `paymentStatus = PAID`.
  3. **Kết hợp (SPLIT):** Khách trả một phần tiền mặt, phần còn lại quét QR. Hệ thống khóa phần tiền mặt, tạo mã QR cho đúng số tiền chuyển khoản còn lại. Khi tiền vào tài khoản → cập nhật `paymentStatus = COLLECTED`, `collectionMethod = MIXED`.
* **Chống gian lận thu ngân:** Thu ngân **không có quyền** bấm xác nhận chuyển khoản thủ công cho đơn POS `BANK_TRANSFER` hay `SPLIT`; bắt buộc phải có webhook ngân hàng thật để tránh thất thoát tài chính.

---

## 4. SƠ ĐỒ ĐIỀU PHỐI TỔNG THỂ (STATE MACHINE MATRIX)

```
[Khách / Thu ngân gửi đơn]
          │
          ▼
   [PENDING_STOCK] ──(Kiểm tra tồn kho & Đặt giữ Saga)──┐
          │                                              │
          ▼                                              ▼ (Hết tồn / Lỗi)
┌─────────────────────────────────┐               [CANCELLED]
│ Phân nhánh theo Nguồn & Cách trả│
└────────────────┬────────────────┘
                 │
                 ├─► [Web + VietQR] ──► [PENDING_PAYMENT] ──(Webhook SePay khớp)──┐
                 │                                                                 │
                 └─► [Web COD / Web Pickup / POS Quầy] ───────────────────────────┼──► [CONFIRMED]
                                                                                  │         │
                                                                                  │         ▼
                                                                                  │    [PREPARING] (Bếp làm - CHẶN HỦY)
                                                                                  │         │
                                                                                  │         ▼
                                                                                  │      [READY]
                                                                                  │         │
                                      ┌───────────────────────────────────────────┴─────────┴───────────────────────┐
                                      │                                                                             │
                                      ▼ (Đơn Tự Lấy / POS)                                                          ▼ (Đơn Giao Hàng DELIVERY)
                        [Kiểm tra: Đã PAID hoặc COLLECTED?]                                          [Shipper nhận: merchantPaymentStatus=PAID_SIMULATED]
                         ├── Chưa ──► Bắt buộc Thu tiền quầy                                                        │
                         └── Rồi  ──► [COMPLETED] (Bàn giao tại quầy)                                                ▼
                                                                                                               [SHIPPING]
                                                                                                                    │
                                                                                                                    ▼ (Giao có bằng chứng)
                                                                                                               [COMPLETED]
```

---

## 5. NHỮNG ĐIỂM SÁNG KỸ THUẬT "ĐẮT GIÁ" ĐÃ GIẢI QUYẾT

1. **Bảo vệ tính toàn vẹn trạng thái tại API Boundary (`OrderService.java:1225–1227`)**:
   Chặn đứng việc client hay nhân viên tự tiện gửi kèm `paymentStatus` khi bấm chuyển trạng thái làm món. Trạng thái thanh toán chỉ được thay đổi thông qua sự kiện xác nhận tiền chính thống.
2. **Chặn hủy sai quy trình bếp (`OrderService.java:1216–1221`)**:
   Khi đơn đã ở `PREPARING` hoặc `READY`, hệ thống từ chối mọi yêu cầu hủy đơn từ cả khách lẫn nhân viên để tránh lãng phí nguyên vật liệu.
3. **Phân định quyền hạn giữa Người và Hệ thống vận chuyển (`OrderService.java:1212–1215`)**:
   Nhân viên thu ngân không thể "ngồi tại quầy bấm hoàn tất" cho một đơn đang giao ngoài đường. Trạng thái `SHIPPING`, `COMPLETED`, `DELIVERY_FAILED` của đơn `DELIVERY` hoàn toàn do `ShippingOrderProjection` phụ trách.
4. **Xử lý tinh tế đơn hàng 0 VNĐ (`OrderService.java:535–543, 831–836`)**:
   Khi khách dùng Voucher hoặc Điểm Loyalty giảm 100%, hệ thống tự động gán `paymentStatus = COLLECTED` và `collectionMethod = NO_PAYMENT`, cho phép đơn đi thẳng vào bếp mà không tạo ra các khoản thu 0 đồng rác.
5. **Tính toán cấn trừ ưu đãi chặt chẽ (`OrderService.java:474–480`)**:
   Điểm thưởng Loyalty và Voucher chỉ được cấn trừ vào giá trị món ăn, **tuyệt đối không bao giờ được trừ vào phí vận chuyển**.

---

## 6. DANH MỤC FILE & DÒNG CODE DẪN CHỨNG TRONG DỰ ÁN

| Nghiệp vụ / Quyết định Logic | Vị trí File Code | Dòng Code / Hàm cụ thể |
| :--- | :--- | :--- |
| Khởi tạo trạng thái ban đầu (Online Web) | `backend/cart-order-service/.../OrderService.java` | `createOrder()`, dòng 476–496 |
| Khởi tạo trạng thái ban đầu (Offline POS) | `backend/cart-order-service/.../OrderService.java` | `createCounterOrder()`, dòng 804–855 |
| Chốt chặn điều kiện bàn giao & cấm hủy bếp | `backend/cart-order-service/.../OrderService.java` | `updateStatus()`, dòng 1200–1260 |
| Tiếp nhận webhook SePay & khớp tiền ngân hàng | `backend/cart-order-service/.../OrderPaymentService.java` | `decide()`, `decideCounterTransfer()` |
| Vận chuyển mô phỏng độc quyền kết thúc đơn | `backend/cart-order-service/.../ShippingOrderProjection.java` | `apply()`, dòng 81–154 |
| Giao diện POS & thu tiền quầy | `frontend/react/src/pages/Staff/components/` | `CounterOrderModal.tsx`, `CounterPaymentModal.tsx` |
