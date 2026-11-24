# Clucky — Bản đồ tra cứu nhanh 5 giây khi bảo vệ dự án

Bộ tài liệu này tổng hợp toàn bộ kiến thức kỹ thuật, luồng nghiệp vụ, kiến trúc và giải pháp xử lý sự cố của dự án **Clucky** (đối chiếu trực tiếp với mã nguồn tại `f:\OJT\Clucky`). Toàn bộ **9 file cũ** đã được sao lưu an toàn trong thư mục [`_backup-cu/`](./_backup-cu).

> **Cách dùng nhanh trong buổi vấn đáp:**
> 1. Bấm vào file theo **Câu hỏi của Mentor** ở bảng dưới đây.
> 2. Đọc ngay khối **`> Trả lời nhanh 30 giây:`** ở đầu mỗi file để trả lời dứt khoát ý chính.
> 3. Nếu mentor hỏi sâu, kéo xuống **Sơ đồ / Bảng so sánh / Kỹ thuật xử lý** và mở **File code chứng minh** ở cuối trang.

---

## Danh mục 10 chuyên đề

| STT | Tài liệu | Chủ đề trọng tâm — Mở khi nào? |
|---|---|---|
| 01 | [01-khung-thuyet-trinh.md](./01-khung-thuyet-trinh.md) | Kịch bản thuyết trình 5–7 phút, phạm vi đóng góp M3 (Trần Đức Anh) & 5 đoạn code "vàng" |
| 02 | [02-kien-truc-va-he-thong.md](./02-kien-truc-va-he-thong.md) | 12 module backend, DB-per-service, REST vs Kafka, ngôn ngữ nghiệp vụ chuẩn & nhịp sync 3 UI |
| 03 | [03-gio-hang-redis-va-checkout.md](./03-gio-hang-redis-va-checkout.md) | **Tại sao dùng Redis cho giỏ hàng**, TTL 24h/7d, merge cart, validate giá server-side, combo & giữ tồn 3 pha |
| 04 | [04-trang-thai-don-hang-va-pos.md](./04-trang-thai-don-hang-va-pos.md) | State Machine 2 trục độc lập, ma trận 4 chiều, tab "Chờ tiền" POS, khóa đổi phương thức & 3 đơn mẫu A/B/C |
| 05 | [05-thanh-toan-sepay-pos.md](./05-thanh-toan-sepay-pos.md) | **Thanh toán VietQR + SePay** (webhook timing-safe, 2-way Kafka), thu tiền POS/SPLIT chống gian lận, sổ quỹ & ngoại lệ |
| 06 | [06-hoan-tien-refund.md](./06-hoan-tien-refund.md) | **Logic Hoàn tiền (Refund) 2 pha**: 6 trạng thái, giữ tiền `HELD`, chứng từ Cloudinary private + SHA-256, bồi hoàn Loyalty |
| 07 | [07-kafka-redis-kien-truc-phoi-hop.md](./07-kafka-redis-kien-truc-phoi-hop.md) | **Combo Kafka + Redis làm được gì?** Toàn bộ 6 ứng dụng Redis, 11 Outbox, ~19 Kafka topics, DLT & 4 kịch bản phối hợp |
| 08 | [08-chong-trung-dong-thoi-va-bao-mat.md](./08-chong-trung-dong-thoi-va-bao-mat.md) | Chống đặt trùng 3 tầng (`Idempotency-Key` Postgres), `@Version` + `PESSIMISTIC_WRITE` + `TreeMap`, chặn phiên nhân viên |
| 09 | [09-giao-hang-va-loyalty-qr.md](./09-giao-hang-va-loyalty-qr.md) | Giao hàng mô phỏng (5s/20s, ví COD `PAID_SIMULATED`) & Mã QR Clucky Club 30s (Lua CAS, `GETDEL`, `loyalty_scan_proofs`) |
| 10 | [10-hoi-dap-mentor-va-gioi-han.md](./10-hoi-dap-mentor-va-gioi-han.md) | **25 câu hỏi xoáy của Mentor** (kèm đáp án ngắn gọn) & **Bảng giới hạn trung thực** của phiên bản hiện tại |

---

## Mentor hỏi gì? → Mở ngay file đó!

| Câu hỏi / Chủ đề Mentor nhắc tới | Mở ngay file | Mục cần xem |
|---|---|---|
| "Giới thiệu luồng đặt hàng & phần em làm?" | [01-khung-thuyet-trinh.md](./01-khung-thuyet-trinh.md) | Mục 1–5 |
| "Kiến trúc dự án, các service giao tiếp kiểu gì?" | [02-kien-truc-va-he-thong.md](./02-kien-truc-va-he-thong.md) | Bảng 12 module & REST vs Kafka |
| "**Tại sao dùng Redis cho giỏ hàng** mà không lưu DB hay LocalStorage?" | [03-gio-hang-redis-va-checkout.md](./03-gio-hang-redis-va-checkout.md) | Mục 1 (Bảng so sánh 4 phương án) |
| "Giỏ guest/user sống bao lâu? Đóng tab / login merge sao?" | [03-gio-hang-redis-va-checkout.md](./03-gio-hang-redis-va-checkout.md) | Mục 2 & 3 (TTL & `mergeCart`) |
| "Giá dưới client sửa được thì sao? Tồn kho trừ lúc nào?" | [03-gio-hang-redis-va-checkout.md](./03-gio-hang-redis-va-checkout.md) | Mục 4 & 5 (Giữ tồn 3 pha) |
| "**Combo Kafka + Redis** trong Clucky kết hợp làm được những gì?" | [07-kafka-redis-kien-truc-phoi-hop.md](./07-kafka-redis-kien-truc-phoi-hop.md) | Mục 1 & Mục 4 (4 kịch bản phối hợp) |
| "Outbox Pattern là gì? Kafka gửi trùng hoặc lỗi (DLT) xử lý sao?" | [07-kafka-redis-kien-truc-phoi-hop.md](./07-kafka-redis-kien-truc-phoi-hop.md) | Mục 3 (Outbox, Idempotency, DLT & Replay) |
| "Tại sao đơn POS nằm tab **Chờ tiền** nhưng DB lại là `CONFIRMED`?" | [04-trang-thai-don-hang-va-pos.md](./04-trang-thai-don-hang-va-pos.md) | Mục 3 (2 tầng cần phân biệt ở POS) |
| "Bếp nấu trước khi trả tiền được không? Khi nào chặn bàn giao?" | [04-trang-thai-don-hang-va-pos.md](./04-trang-thai-don-hang-va-pos.md) | Mục 1 & Mục 5 (`updateStatus` 409) |
| "POS đổi tiền mặt sang QR lúc người khác đang thu thì sao?" | [04-trang-thai-don-hang-va-pos.md](./04-trang-thai-don-hang-va-pos.md) | Mục 4 (`claimCounterPayment`) |
| "**Thanh toán VietQR SePay** chạy thế nào? Chống webhook giả/lặp?" | [05-thanh-toan-sepay-pos.md](./05-thanh-toan-sepay-pos.md) | Phần A (`SepayWebhookSecurity`, 2-way Kafka) |
| "Khách chuyển **thiếu tiền / dư tiền / muộn**? Thu ngân bấm nhận chuyển khoản tay?" | [05-thanh-toan-sepay-pos.md](./05-thanh-toan-sepay-pos.md) | Phần B & Phần C (Ngoại lệ dòng tiền) |
| "**Hoàn tiền (Refund)** xử lý ra sao? Dùng phương pháp gì đảm bảo nhất quán?" | [06-hoan-tien-refund.md](./06-hoan-tien-refund.md) | Toàn bộ (State Machine 6 bước & 5 phương pháp) |
| "Hoàn tiền có gọi API ngân hàng tự động không? Điểm/voucher xử lý sao?" | [06-hoan-tien-refund.md](./06-hoan-tien-refund.md) | Mục 1 & Mục 4 (Cloudinary + Loyalty clawback) |
| "Khách **double-click Đặt hàng** / mạng lag thì chống trùng thế nào?" | [08-chong-trung-dong-thoi-va-bao-mat.md](./08-chong-trung-dong-thoi-va-bao-mat.md) | Phần 1 (`web_order_operations` / `counter_order_operations`) |
| "**2 nhân viên dùng chung 1 tài khoản** hoặc cùng bấm 1 đơn thì sao?" | [08-chong-trung-dong-thoi-va-bao-mat.md](./08-chong-trung-dong-thoi-va-bao-mat.md) | Phần 2 & Phần 3 (`max_active_sessions`, Redis denylist) |
| "Giao hàng mô phỏng chạy sao? `PAID_SIMULATED` có tính vào doanh thu?" | [09-giao-hang-va-loyalty-qr.md](./09-giao-hang-va-loyalty-qr.md) | Phần A (`MockDeliveryEngine`, `mock_wallet_ledger`) |
| "Mã **QR Loyalty 30 giây** chống chụp màn hình gửi cho bạn quét hộ kiểu gì?" | [09-giao-hang-va-loyalty-qr.md](./09-giao-hang-va-loyalty-qr.md) | Phần B (`GETDEL`, Lua scripts, `loyalty_scan_proofs`) |
| "Hệ thống hiện còn điểm yếu / giới hạn gì chưa làm xong?" | [10-hoi-dap-mentor-va-gioi-han.md](./10-hoi-dap-mentor-va-gioi-han.md) | Bảng Giới hạn trung thực ở cuối file |

---

## ⚠️ Cảnh báo: 6 điểm đính chính quan trọng so với tài liệu cũ

Một số file báo cáo cũ (trong `_backup-cu/`) ghi chưa khớp với mã nguồn hiện tại của Clucky. **Tuyệt đối nhớ 6 điểm này để không nói nhầm trước Mentor:**

1. **Tên service đúng:** Dự án **không có** `auth-service` hay `inventory-service`. Tên thật là **`identity-service`** (chứa `AuthSessionService`) và **`catalog-store-service`** (chứa `OrderStockService`, `InventoryService`).
2. **Chống đặt trùng đơn lưu ở PostgreSQL, KHÔNG phải Redis:** Clucky **không dùng** `SET idempotency:<key> "PROCESSING" EX 120 NX` trong Redis. Chống đặt trùng đơn được lưu bền vững bằng **PostgreSQL** (`order_service.web_order_operations` và `order_service.counter_order_operations`) dùng `INSERT ... ON CONFLICT DO NOTHING` + mã băm SHA-256 `payload_hash`.
3. **`Idempotency-Key` bắt buộc cho cả Web lẫn POS:** Endpoint Web `POST /api/orders` yêu cầu header `Idempotency-Key` mặc định (`@RequestHeader("Idempotency-Key")`). Endpoint POS `POST /api/orders/counter` cũng kiểm tra bắt buộc ngay đầu `OrderService.createCounterOrder()` (thiếu trả `400 VALIDATION_ERROR`). *(Chỉ có overload nội bộ 1 tham số trong Java không nhận key, còn qua HTTP đều bắt buộc).*
4. **Giới hạn phiên nhân viên mặc định là 3, KHÔNG phải 1:** Cột `identity_service.users.max_active_sessions` có giá trị mặc định **`3`** cho `ADMIN`, `STORE_MANAGER`, `STAFF` (và quản lý chỉnh được trong khoảng **`1..10`** qua API `AuthSessionController`). Khi vượt ngưỡng sẽ từ chối cứng `409 RESOURCE_CONFLICT` (`sessionLimit = REACHED`), **không auto-kick** phiên cũ.
5. **Tên bảng ví shipper mô phỏng:** Tên thật là `shipping_service.mock_wallet_ledger` (không phải `mock_rider_wallet_ledger`).
6. **Các lỗi Combo trong `Menu.md` (ngày 17/11) đa phần đã sửa:** Trong code hiện tại, `comboAsProduct()` đã gán `slug: combo-${combo.id}`, trang `/menu/[slug]` đã xử lý `combo-*` và tìm cả `getCombos()`, `ValidationService` đã cộng đúng `lineTotal` (kèm `extraPrice`) vào `totalSubtotal`. *(Chỉ còn `comboAsProduct()` chưa gán `categoryId`, nhưng `/menu` đã lọc client-side theo tên `"Combo"`).*
