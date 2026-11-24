[← Mục lục tra cứu nhanh](./00-muc-luc.md)

# 10 — 25 Câu hỏi xoáy của Mentor & Bảng Giới hạn trung thực

> **Trả lời nhanh 30 giây:** Tài liệu này gom **25 câu hỏi kỹ thuật hay bị mentor hỏi vặn nhất** theo 5 nhóm chủ đề kèm câu trả lời đi thẳng vào trọng tâm, và **Bảng giới hạn hiện tại** để trả lời tự tin, trung thực nếu mentor hỏi *"Hệ thống của em còn điểm yếu/thiếu sót nào?"*.

---

## Phần 1 — 25 Câu hỏi vấn đáp thường gặp (Đọc là trả lời được ngay)

### Nhóm A — Giỏ hàng Redis & Checkout (5 câu)

1. **Tại sao lưu giỏ hàng ở Redis mà không lưu PostgreSQL hay LocalStorage?**
   → Giỏ hàng biến động liên tục, tỷ lệ bỏ giỏ cao: Redis ghi trên RAM siêu nhanh giảm tải I/O cho Postgres, tự dọn rác bằng TTL (`EXPIRE`) không cần cron job, vừa hỗ trợ Guest (`cart:guest:{sessionId}`) vừa đồng bộ đa thiết bị khi đăng nhập (`cart:{userId}`) — điều LocalStorage không làm được. *(Xem [03-gio-hang-redis-va-checkout.md](./03-gio-hang-redis-va-checkout.md)).*
2. **Giỏ hàng sống bao lâu? Chỉ mở xem lại hoặc đóng trình duyệt thì có gia hạn không?**
   → Guest **24 giờ**, tài khoản **7 ngày** (hằng số trong `CartService`). Thêm món hoặc sửa số lượng dương thì đặt lại TTL; **chỉ xem giỏ (`getCart`) hoặc xóa 1 món (`removeItem`) KHÔNG gia hạn**; đóng trình duyệt không dừng đồng hồ TTL.
3. **Tại sao đặt hàng phải gọi Catalog tính lại giá thay vì lấy giá từ giỏ?**
   → Vì giỏ không khóa giá và client có thể sửa payload request. Backend dùng `ValidationService` tính lại toàn bộ món/combo/topping bằng `BigDecimal`, so với `expectedTotal` — lệch thì trả `409 RESOURCE_CONFLICT`.
4. **Giỏ hàng Redis có an toàn cập nhật đồng thời tuyệt đối không?**
   → **Thẳng thắn là chưa tuyệt đối:** `CartService` đang đọc hash lên Java rồi ghi lại nhiều lệnh (`read-modify-write`), chưa bọc Lua script. Phía trình duyệt dùng `navigator.locks` khi merge lúc login để giảm race giữa các tab.
5. **Tồn kho bị trừ lúc nào? Nếu trừ kho xong mà bước sau lỗi thì sao?**
   → Trừ sau khi lưu nháp `PENDING_STOCK` qua REST đồng bộ (`reserveStock` khóa `order_stock_ledger`). Nếu Pha 3 lỗi, hệ thống gọi đồng bộ `releaseStock` (retry 3 lần) + hủy nháp + phát `order.cancelled`; nếu sập nguồn giữa chừng có `PendingStockRecoveryScheduler` (15p) và `StockReconciliationScheduler` (10p) thu hồi bù.

### Nhóm B — Combo Kafka + Redis & Kiến trúc (5 câu)

6. **Combo Kafka + Redis trong Clucky kết hợp giải quyết được gì?**
   → Redis xử lý tầng **tốc độ/tạm thời/nguyên tử** (giỏ, cache menu 5p, queue POS, QR 30s Lua+`GETDEL`, denylist JWT, rate-limit); Kafka + Outbox xử lý tầng **bền vững/liên service** (đơn hàng, thanh toán, hoàn tiền, trả kho kèm xóa cache menu Redis, cộng/thu hồi điểm, email, audit). *(Xem [07-kafka-redis-kien-truc-phoi-hop.md](./07-kafka-redis-kien-truc-phoi-hop.md)).*
7. **Tại sao không lưu DB xong rồi gọi thẳng `kafkaTemplate.send()` mà phải dùng Transactional Outbox?**
   → Vì gọi trực tiếp không nằm trong transaction SQL: nếu DB commit xong mà mạng/Kafka rớt đúng tích tắc đó thì mất vĩnh viễn sự kiện! Outbox ghi vào bảng `*_outbox` **cùng transaction DB** (`Propagation.MANDATORY`), rồi worker dùng `FOR UPDATE SKIP LOCKED` gửi và đợi ACK đồng bộ mới đánh dấu đã gửi.
8. **Outbox gửi lặp (`at-least-once`) hoặc Kafka gửi trùng thì consumer có bị trừ kho / cộng tiền 2 lần không?**
   → **Không.** Mọi consumer đều cài idempotency bằng bảng dedup với `INSERT ... ON CONFLICT DO NOTHING` (ví dụ `payment_decisions`, `processed_events`, `order_stock_ledger`, `mock_delivery_receipts`, `loyalty_refunds`). Riêng Loyalty còn gọi ngược REST xác minh snapshot mới nhất từ Order trước khi chốt điểm.
9. **Message bị lỗi parse hoặc chết consumer thì đi đâu? Dữ liệu khách hàng có lộ vào DLT không?**
   → Sau 3 lần retry (`1s`) hoặc lỗi JSON/schema sẽ vào topic `<topic>.DLT`. Clucky **không ném nguyên payload PII** vào DLT mà chỉ đóng gói **Quarantine Envelope** (metadata + `sha256` + các ID/số tiền an toàn). Muốn chạy lại dùng `scripts/replay-kafka-dlt.py` bắt buộc đối chiếu SHA-256 bản ghi gốc và lưu audit file.
10. **Tại sao chống đặt trùng đơn và thanh toán lại lưu PostgreSQL chứ không lưu Redis?**
    → Vì đơn hàng và tiền bạc cần **ACID**, ràng buộc duy nhất (`PK`/`UNIQUE`) gắn chặt transaction tạo đơn/nhận tiền, và lịch sử đối soát lâu dài; không đánh đổi rủi ro mất dữ liệu trong RAM hoặc bị chặn ghi khi Redis đầy (`noeviction`).

### Nhóm C — Trạng thái đơn & POS (5 câu)

11. **Tại sao phải tách 2 trục `status` và `paymentStatus`? Có phức tạp thừa không?**
    → Không thừa vì F&B có đơn nấu trước trả sau (POS, Web tự lấy tiền mặt, COD) và đơn trả trước mới nấu (Web VietQR). Tách 2 trục giúp bếp vẫn nấu mà ở bước **bàn giao `COMPLETED`** backend vẫn ép được `paymentStatus` phải là `PAID` hoặc `COLLECTED` (`409` nếu thiếu). *(Xem [04-trang-thai-don-hang-va-pos.md](./04-trang-thai-don-hang-va-pos.md)).*
12. **Tại sao đơn POS nằm ở tab `Chờ tiền` nhưng dưới DB `status` lại là `CONFIRMED`?**
    → Vì `Chờ tiền` là cách UI (`StaffOrdersPage.visibleOrders`) gom các đơn `COUNTER` còn `paymentStatus = PENDING` để thu ngân không quên thu tiền, còn `Order.status` thực chất là tiến độ món (`CONFIRMED`/`PREPARING`/`READY`). Khi thu tiền xong, đơn tự quay về đúng tab tiến độ bếp.
13. **2 nhân viên POS: người bấm đổi từ Tiền mặt sang QR, người kia bấm Thu tiền cùng lúc thì sao?**
    → `changeCounterPaymentMethod` hỏi Payment trước (ngoài lock), rồi khóa dòng Order kiểm `counterPaymentStarted`. `claimCounterPayment` từ Payment cũng khóa đúng dòng Order đó để bật cờ. Ai lấy lock trước thắng, người sau nhận **`409`**, không deadlock chéo service.
14. **2 nhân viên cùng bấm chuyển trạng thái 1 đơn (ví dụ cùng bấm hoàn thành hoặc 1 hủy 1 nấu) thì sao?**
    → Backend khóa bi quan `PESSIMISTIC_WRITE` (`lockById`) xử lý tuần tự + kiểm `@Version` (với Staff có gửi version, lệch trả `409`) + kiểm bảng chuyển trạng thái và cấm hủy khi đã vào `PREPARING`/`READY` (`409`).
15. **Đơn 0 đồng (dùng voucher/điểm hết tiền món) và đơn COD mô phỏng xử lý tiền ra sao?**
    → Đơn 0đ tự gán `CONFIRMED / COLLECTED` với `collectionMethod = NO_PAYMENT` và bỏ qua bước thu. Đơn COD giao mô phỏng lưu tiền ứng ở `merchantPaymentStatus = PAID_SIMULATED` còn `paymentStatus` giữ `PENDING`, nên sổ quỹ thu thật và cộng điểm Loyalty không tự cộng nhầm.

### Nhóm D — Thanh toán & Hoàn tiền (6 câu)

16. **Làm sao chống giả mạo webhook SePay và chống SePay gửi lặp cộng tiền 2 lần?**
    → `SepayWebhookSecurity` kiểm `Authorization: Apikey` bằng **`MessageDigest.isEqual`** chống timing attack, giới hạn body 32KB, parse JSON strict. `SepayReceiptService` dùng `bank_tx_id = SEPAY:{id}` `UNIQUE` + SHA-256 `webhook_fingerprint`: trùng nội dung chỉ tăng `replay_count`, trùng ID mà lệch tiền/nội dung thì đưa vào `sepay_receipt_conflicts` (`409`). *(Xem [05-thanh-toan-sepay-pos.md](./05-thanh-toan-sepay-pos.md)).*
17. **Thu ngân ở quầy có bấm "Khách đã chuyển khoản" bằng tay khi khách đưa ảnh màn hình chuyển khoản không?**
    → **Tuyệt đối không!** Backend chặn cứng xác nhận chuyển khoản thủ công đối với đơn `COUNTER` ở 4 nơi (`CounterPaymentService`, `PaymentOperations` x2, `OrderPaymentService`). Bắt buộc mở phiên VietQR POS và chờ webhook SePay thật về khớp tiền.
18. **Khách chuyển thiếu tiền, chuyển dư tiền, hoặc chuyển muộn sau khi hết hạn 15 phút thì sao?**
    → **Thiếu:** cộng vào `received_amount`, chưa phát `payment.received`, UI khách vẽ lại QR đúng số tiền thiếu `missingAmount`. **Dư:** vẫn chấp nhận đủ đơn, báo `excessAmount`, nhưng **loại khỏi số tiền được hoàn tự động**. **Muộn (sau 15p):** ghi `LATE_RECEIPT`, không tự mở lại đơn đã đóng.
19. **Hoàn tiền (Refund) khác hủy đơn chỗ nào? Hệ thống có gọi API ngân hàng tự chuyển tiền trả khách không?**
    → **Không gọi API chi hộ ngân hàng — chuyển khoản thủ công 100%.** Tách bạch rõ `APPROVED` (mới đồng ý và giữ tiền `HELD` trong Payment) → `PROCESSING` (khóa cho 1 Quản lý thao tác chuyển ngoài) → `REFUNDED` (đã chuyển thật và đối chiếu xong). *(Xem [06-hoan-tien-refund.md](./06-hoan-tien-refund.md)).*
20. **Làm sao ngăn Quản lý bấm "Đã hoàn tiền" khống khi chưa chuyển khoản hoặc sửa số tài khoản giữa chừng?**
    → Có 3 chốt: (1) `claimProcessing` tạo `RefundAttempt` khóa cho đúng 1 người và đóng băng STK thành `current_attempt_bank_snapshot`; (2) Bắt buộc tải ảnh chuyển khoản (re-encode PNG lên Cloudinary private `authenticated`); (3) Băm **`SHA-256(evidencePublicId)`** lưu vào `refund_reservations.evidence_reference` với **DB `CHECK` constraint (`^[a-f0-9]{64}$`)** bắt buộc có chứng từ mới cho `CONFIRMED`.
21. **Khi hoàn tiền thành công (`refund.processed`), điểm thưởng và voucher của đơn đó xử lý thế nào?**
    → Nếu đơn hủy trước bếp (`loyalty_orders != COMPLETED`): đánh dấu `NOOP` vì điểm đã nhả từ lúc `order.cancelled`. Nếu đơn đã từng `COMPLETED`: `LoyaltyRefundService` hoàn lại điểm đã tiêu (`RESTORE`), thu hồi điểm đã tích (`EARN`, thiếu thì ghi nợ `pending_clawback_points` qua `DEFER`), còn voucher cá nhân đã dùng thì **không hoàn lại** (`REFUND_NO_RESTORE`).

### Nhóm E — Bảo mật phiên, Vận chuyển & QR Loyalty (4 câu)

22. **Tại sao giới hạn phiên nhân viên không auto-kick phiên cũ khi người mới đăng nhập?**
    → Theo `SESSION_POLICY.md`, để bảo vệ thu ngân đang đứng quầy tính tiền không bị đá văng giữa chừng. Khi đủ giới hạn (`max_active_sessions`, mặc định **3**, chỉnh **1–10**), hệ thống khóa dòng `users` bằng `FOR UPDATE` và trả thẳng `409` (`sessionLimit = REACHED`).
23. **Nhân viên bấm Đăng xuất nhưng Access Token JWT vẫn còn hạn 30 phút thì chặn bằng cách nào?**
    → `SessionRevocationService` ghi `auth:revoked:sid:{sid}` (và `auth:revoked:user:{id}` / `user-ns:{id}` khi đổi mật khẩu) vào **Redis trước khi commit DB**. Cả `api-gateway` lẫn 9 service đều kiểm tra Redis trên mọi request (`fail-closed`) → chặn ngay `401`. *(Xem [08-chong-trung-dong-thoi-va-bao-mat.md](./08-chong-trung-dong-thoi-va-bao-mat.md)).*
24. **Mã QR Loyalty 30s khác gì Mã chữ `LY-...`? Chụp màn hình gửi cho người khác quét hộ có dùng được không?**
    → QR Vector có entropy 256-bit, vừa tích vừa tiêu điểm và bị xóa ngay sau **1 lần quét** bằng Redis **`GETDEL`** (cộng thêm giới hạn 30s xoay mã, xóa khỏi DOM đúng giây 0, `loyalty_scan_proofs` khóa đúng `staff_id + store_id + draft_hash` trong 30s). Mã chữ `LY-...` **chỉ được tích điểm**, bị cấm trừ điểm để tránh nhìn trộm rút điểm. *(Xem [09-giao-hang-va-loyalty-qr.md](./09-giao-hang-va-loyalty-qr.md)).*
25. **Giao hàng mô phỏng (`MockDeliveryEngine`) chạy thế nào và làm sao biết không làm sai báo cáo thật?**
    → Tick 500ms: chờ 5s chuyển `SHIPPING`, thêm 20s sinh `SIMULATED_WATERMARK` chuyển `DELIVERED` → `COMPLETED`. Ví ứng COD ghi riêng ở `shipping_service.mock_wallet_ledger` và `Order.merchantPaymentStatus = PAID_SIMULATED` trong khi `paymentStatus` giữ `PENDING`, tách biệt hoàn toàn khỏi doanh thu thu thật.

---

## Phần 2 — Bảng Giới hạn hiện tại & Hướng hoàn thiện (Nói thẳng thắn để ghi điểm tư duy)

Nếu mentor hỏi *"Dự án này còn hạn chế hay nợ kỹ thuật (technical debt) gì?"*, trả lời rõ 8 điểm đã đối chiếu từ code dưới đây:

| Hạng mục | Hiện trạng thực tế trong code | Hướng hoàn thiện tiếp theo |
|---|---|---|
| **1. Giỏ Redis & TTL Config** | `CartService` hardcode hằng `24h`/`7d` và `CatalogCacheService` hardcode `5m` chưa đọc `RedisConfig.Policy` (`7d`/`24h`/`10m`). Thao tác giỏ là read-modify-write chưa dùng Lua. | Đọc TTL từ `Policy` và bọc thao tác sửa/merge giỏ bằng Redis Lua script. |
| **2. Tab "Chờ tiền" POS** | UI gom POS `PENDING` vào tab Chờ tiền, nhưng backend vẫn cho `CONFIRMED → PREPARING` khi chưa thu tiền. | Thống nhất lại rule sản phẩm: giữ như hiện tại (ăn trước trả sau) hoặc chặn hẳn ở backend nếu đổi chính sách. |
| **3. `@Version` ở Manager UI** | Giao diện Staff đã gửi `version` khi đổi trạng thái, nhưng một số đường gọi từ Store Manager (`OrderContext`) chưa truyền `version`. | Bổ sung truyền `version` đồng nhất ở mọi caller nội bộ. |
| **4. DLT trên 2 service** | `catalog-store-service` & `identity-service` chưa cấu hình DLT producer (mới dừng retry/giữ offset để không mất event). | Bổ sung quarantine `.DLT` an toàn PII đồng bộ với 6 service còn lại. |
| **5. Sổ quỹ ca trực & Excel Admin** | `getShiftSummary` tổng hợp theo ngày/chi nhánh, chưa nối `shiftId` mở/đóng ca & số dư đầu ca; xuất Excel Admin đang xuất theo tập trang đã tải. | Gắn `shiftId` với ca làm việc ở `identity-service` và thêm API export toàn bộ theo bộ lọc. |
| **6. Làm tròn tiền mặt lẻ ở Web Tự lấy** | `CounterPaymentService` làm tròn bội 1.000đ cho tiền mặt, nhưng `PaymentOperations.collect` đối chiếu tổng gốc khi kiểm tra collection Web Pickup COD. | Chuẩn hóa đồng nhất quy tắc tổng làm tròn giữa 2 đường thu nếu phát sinh đơn lẻ dưới 1.000đ. |
| **7. Mô phỏng & Tích hợp Phase 2** | Giao hàng dùng `MockDeliveryEngine`, in K80/KOT là mô phỏng trên trình duyệt, hoàn tiền là chuyển khoản tay, checkout chưa có UI chọn `scheduledAt`. | Tích hợp API hãng vận chuyển thật (Grab/Ahamove), máy in nhiệt/hóa đơn điện tử, payout API & chọn giờ hẹn lấy. |
| **8. `comboAsProduct` `categoryId`** | `comboAsProduct()` ở Next.js chưa gán `categoryId` (mới có `categoryName: "Combo"`), `/menu` đang workaround lọc theo tên ở client. | Bổ sung map `categoryId` động từ danh mục Combo nếu API `getProducts({ categoryId })` mở lọc server-side cho combo. |
