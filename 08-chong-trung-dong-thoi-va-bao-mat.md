[← Mục lục tra cứu nhanh](./00-muc-luc.md)

# 08 — Chống đặt trùng, Xung đột đồng thời & Kiểm soát phiên nhân viên

> **Trả lời nhanh 30 giây:** Clucky xử lý **3 bài toán đồng thời cốt lõi** trong F&B: (1) **Chống đặt đơn trùng (Double-click)** bằng 3 tầng gồm gộp `itemKey` MD5, khóa `submitLock` + `sessionStorage` ở FE, và bảng `web_order_operations` / `counter_order_operations` trong **PostgreSQL** (`Idempotency-Key` + SHA-256 `payload_hash`, **bắt buộc** cả Web lẫn POS); (2) **Chống ghi đè đơn & âm kho** bằng `PESSIMISTIC_WRITE` + `@Version` + sắp xếp `productId` (`TreeMap`) tránh deadlock; (3) **Chống nhiều nhân viên dùng chung 1 tài khoản lộn xộn** bằng khóa dòng `users` (`FOR UPDATE`), giới hạn phiên active (mặc định **3**, chỉnh **1–10**, **không auto-kick**), và thu hồi JWT tức thì qua **Redis denylist**.

---

## Phần 1 — Chống đặt trùng món & chống bấm đặt đơn 2 lần

### Bối cảnh rủi ro
Khách hàng bấm nút "Đặt hàng" 2 lần liên tiếp hoặc mạng chập chờn khiến trình duyệt/khách gửi lại request cũ → dễ tạo 2 đơn giống nhau, trừ tồn kho 2 lần hoặc bắt khách trả tiền 2 lần.

### Giải pháp 3 tầng thực tế trong mã nguồn Clucky

| Tầng bảo vệ | Vị trí code | Cơ chế hoạt động chi tiết |
|---|---|---|
| **Tầng 1: Gộp trùng món trong giỏ** | `CartItemDto.generateItemKey` (L59–83) | Băm **MD5 hex** từ `P:{productId}` hoặc `C:{comboId}` + `optionIds` (đã sort) + `toppingIds` (đã sort) + `comboSelections` (đã sort). Thêm trùng cấu hình sẽ **cộng dồn số lượng** vào cùng 1 field Hash thay vì tạo dòng mới. *(Không đưa `note`, `storeId`, `unitPrice` vào key).* |
| **Tầng 2: Khóa & Lưu nháp khóa ở Checkout FE** | `frontend/nextjs/app/checkout/page.tsx` (`handleSubmitOrder`, L408–481) | • Chặn ngay trên RAM bằng `submitLock.current` và state `submitting`.<br>• Tính **SHA-256 `fingerprint`** của `[userId, payload]` bằng `crypto.subtle.digest` và lưu cặp `{ fingerprint, key: UUID }` vào `sessionStorage` (`clucky:checkout-idempotency:v1`).<br>• Nếu lỗi mạng/timeout bấm thử lại **cùng nội dung** → **dùng lại đúng `Idempotency-Key` cũ**. Đổi món/đổi tiền → tự sinh key mới. Thành công mới xóa key. |
| **Tầng 3: Ràng buộc bền vững ở PostgreSQL** | `OrderController` & `OrderService.java` (`V19`, `V24`) | **Cả Web (`POST /api/orders`) và POS (`POST /api/orders/counter`) đều BẮT BUỘC có header `Idempotency-Key`** (POS kiểm tra ngay đầu `createCounterOrder`, thiếu trả `400 VALIDATION_ERROR`).<br>• Bảng `order_service.web_order_operations` (PK `customer_id, idempotency_key`) và `counter_order_operations` (PK `store_id, idempotency_key`) lưu kèm SHA-256 `payload_hash` của request.<br>• Pha 1 chạy `INSERT ... ON CONFLICT DO NOTHING`: nếu đã tồn tại thì gọi `replayWebOrder` / `replayCounterOrder`:<br>  - **Khác `payload_hash`:** Trả **`409 RESOURCE_CONFLICT`** (*"Khóa gửi đơn đã dùng cho nội dung khác"*).<br>  - **Đơn còn ở nháp `PENDING_STOCK` (hoặc `order_id IS NULL`):** Trả **`409`** (*"Đơn đang xử lý, vui lòng thử lại"*).<br>  - **Đơn đã tạo xong:** Trả lại đúng kết quả `OrderResponse` cũ, **không tạo đơn mới & không trừ kho lần hai**.<br>  - Nếu nháp thất bại (`cancelPendingStockDraft`), dòng claim bị xóa để khách có thể retry với chính key đó. |

> **Đính chính quan trọng:** Báo cáo cũ ghi `SET idempotency:... EX 120 NX` trong Redis và ghi "Web API cho bỏ key". **Code thực tế dùng PostgreSQL (`web_order_operations` / `counter_order_operations`) và bắt buộc `Idempotency-Key` ở cả 2 API HTTP.**

---

## Phần 2 — Chống xung đột đồng thời trên Đơn hàng & Tồn kho

### 1. Hai nhân viên cùng bấm cập nhật 1 đơn (`cart-order-service`)
- **Khóa bi quan (`PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`):** `OrderRepository.lockById` khóa dòng đơn hàng trong mọi hàm chuyển đổi (`updateStatus`, `cancelOrder`, `rejectOrder`, `changeCounterPaymentMethod`, `ShippingOrderProjection.apply`, `OrderPaymentService.decide`/`claimCounterPayment`). Request tới sau phải chờ nhả khóa rồi mới đọc dữ liệu mới nhất.
- **Khóa lạc quan (`@Version`):** `Order.version` (`V16__add_order_version.sql`). Giao diện Staff gửi kèm `version`; nếu `request.version() != order.getVersion()`, Backend ném **`409 RESOURCE_CONFLICT`** yêu cầu UI tải lại trạng thái mới nhất trước khi thao tác tiếp.
  - *(Giới hạn trung thực: một số đường gọi `updateStatus` ở `OrderContext` của Store Manager hiện chưa truyền `version` — khi đó backend vẫn dùng khóa dòng + kiểm tra bảng chuyển trạng thái, nhưng không phát hiện được dữ liệu cũ trên màn hình).*

### 2. Hai đơn cùng đặt 1 món cuối trong kho (`catalog-store-service`)
- **Chống trừ kho lặp:** `OrderStockService` khóa bảng `catalog_service.order_stock_ledger(order_id)` (`INSERT ... ON CONFLICT DO NOTHING` + `FOR UPDATE`), đảm bảo nếu REST đồng bộ đã trừ `DEDUCTED` thì sự kiện Kafka `order.created` tới sau sẽ bỏ qua, không trừ 2 lần.
- **Chống Deadlock AB–BA:** Trước khi khóa các dòng tồn kho, `OrderStockService` đưa `quantities` vào **`TreeMap`** để luôn khóa sản phẩm theo thứ tự **`productId` tăng dần** (Đơn 1 mua món 1+2 và Đơn 2 mua món 2+1 đều khóa món 1 trước rồi mới khóa món 2 → không bao giờ deadlock).
- **Khóa dòng & kiểm tồn không âm:** `StoreInventoryRepository.lockByStoreAndProduct` dùng `@Lock(PESSIMISTIC_WRITE)` kết hợp `@Version` trên `StoreInventory`; nếu `previous < quantity` ném ngay `400 INSUFFICIENT_INVENTORY`.

---

## Phần 3 — Kiểm soát nhiều nhân viên đăng nhập cùng 1 tài khoản

### 1. Chính sách `SESSION_POLICY.md` (`identity-service`)
Tại quầy bán hàng, nhiều nhân viên có thể đăng nhập cùng tài khoản tác nghiệp hoặc quản lý, gây rủi ro lệch két tiền mặt, xung đột thao tác đơn, mất dấu kiểm toán.
- **Đối tượng áp dụng:** `ADMIN`, `STORE_MANAGER`, `STAFF`.
- **Giới hạn phiên đồng thời (`identity_service.users.max_active_sessions`):**
  - Giá trị mặc định trong DB (`V14`) là **`3`** phiên hoạt động (quản lý `ADMIN` / `STORE_MANAGER` có thể điều chỉnh từ **`1` đến `10`** cho từng tài khoản qua `AuthSessionController`; bị chặn `409 BELOW_ACTIVE_COUNT` nếu hạ thấp hơn số phiên đang mở).
- **Quy tắc từ chối cứng (Strict Rejection — NO AUTO-KICK):**
  - Khi số phiên đang hoạt động (`revoked_at IS NULL AND refresh_expires_at > now()`) đạt giới hạn (`activeCount >= limit`), người đăng nhập sau bị từ chối thẳng bằng **`409 RESOURCE_CONFLICT`** (`fieldErrors.sessionLimit = REACHED`).
  - **Tuyệt đối không đá phiên cũ (no auto-kick / LRU)** để không làm thu ngân đang tính tiền cho khách bị văng màn hình giữa chừng. Muốn nhường máy phải đăng xuất hợp lệ hoặc nhờ Quản lý thu hồi phiên.

### 2. Chống race condition khi 2 máy bấm Login cùng mili-giây
- Trong `AuthService.login()` và `AuthSessionService`: hệ thống luôn khóa dòng **`identity_service.users`** trước bằng `@Lock(PESSIMISTIC_WRITE)` (`UserRepository.findByLogin`, `findByPhoneForUpdate`, `findByIdForUpdate`) cho tới khi transaction commit xong. Do đó không thể có 2 request login vượt qua bước đếm `activeCount` cùng một tích tắc.

### 3. Thu hồi JWT tức thì không chờ hết hạn (`Redis Token Denylist`)
- Vì JWT là stateless, nếu chỉ xóa refresh token trong DB thì Access Token (mặc định 30 phút) vẫn còn hiệu lực.
- **Giải pháp Clucky (`SessionRevocationService`):**
  - Khi đăng xuất hoặc thu hồi phiên (`AuthSessionService.revoke` / `RefreshTokenService.revokeSession`), backend ghi vào Redis **TRƯỚC khi commit DB** (cơ chế **fail-closed**: Redis lỗi thì trả `503 DEPENDENCY_UNAVAILABLE`):
    - Thu hồi 1 thiết bị: `auth:revoked:sid:{sid}` = `"revoked"` (TTL = `refreshTtl + accessTtl + 1m`).
    - Thu hồi toàn bộ tài khoản (đổi mật khẩu / khóa nick): `auth:revoked:user:{userId}` (ms) và `auth:revoked:user-ns:{userId}` (nano-giây, tránh trường hợp đăng nhập lại ngay trong cùng 1 mili-giây bị từ chối nhầm).
  - **Kiểm tra đa tầng (Defense-in-Depth):** Cả **`api-gateway`** (`AccessTokenValidator` reactive) lẫn **toàn bộ 9 service backend** đều kiểm tra 3 key Redis này trên mọi request mang JWT → chặn ngay `401 UNAUTHORIZED` trong vòng dưới 1ms.

---

## File code mở chứng minh ngay
- Chống đặt trùng đơn: `backend/cart-order-service/src/main/java/com/clucky/cartorderservice/controller/OrderController.java` (L40–55), `service/OrderService.java` (`replayWebOrder`/`replayCounterOrder` L990–1033), `V19__counter_order_idempotency.sql`, `V24__web_order_idempotency.sql`
- Checkout FE lock & sessionStorage: `frontend/nextjs/app/checkout/page.tsx` (L48, L190–191, L408–481)
- Khóa đơn & khóa kho chống deadlock: `OrderRepository.java` (L21–29), `Order.java` (L127–129 `@Version`), `backend/catalog-store-service/src/main/java/com/clucky/catalogstoreservice/service/OrderStockService.java` (L43, L81)
- Quản lý phiên & Denylist JWT: `backend/identity-service/SESSION_POLICY.md`, `service/AuthSessionService.java` (L43–91), `service/SessionRevocationService.java`, `repository/UserRepository.java`, `backend/api-gateway/src/main/java/com/clucky/apigateway/config/AccessTokenValidator.java`
