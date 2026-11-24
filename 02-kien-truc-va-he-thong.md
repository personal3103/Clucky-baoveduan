[← Mục lục tra cứu nhanh](./00-muc-luc.md)

# 02 — Kiến trúc Microservices, Giao tiếp & Ngôn ngữ nghiệp vụ

> **Trả lời nhanh 30 giây:** Clucky dùng kiến trúc **Microservices + DB-per-service** gồm 2 Frontend (Next.js khách hàng & React vận hành), đi qua **1 cổng duy nhất `api-gateway`**, khám phá dịch vụ qua **Eureka (`discovery-server`)**, lấy cấu hình qua **Config Server (`config-server`)**, và giao tiếp **REST đồng bộ** cho nghiệp vụ cần xác minh tức thì / **Kafka Outbox bất đồng bộ** cho sự kiện liên service.

---

## 1. Tech Stack tổng thể

| Thành phần | Công nghệ thực tế | Mục đích |
|---|---|---|
| **Customer UI** | Next.js 15, React 19, TypeScript | Xem menu, giỏ hàng, checkout, VietQR, theo dõi đơn, Clucky Club QR |
| **Staff / Manager / Admin UI** | React 19, Vite 6, Ant Design, TypeScript | POS nhập đơn nhanh, bếp KOT, thu tiền quầy, sổ quỹ, hoàn tiền, quản trị |
| **Backend** | Java 21, Spring Boot 3.4, Spring Cloud 2024, Spring Security | 12 module microservice (Gateway, Config, Eureka, 9 business services) |
| **Cơ sở dữ liệu** | PostgreSQL 16 (mỗi service 1 schema riêng) + Flyway | Lưu dữ liệu bền vững, migration độc lập từng service |
| **In-Memory / Cache / Token** | Redis 7 (`redis:7-alpine`, AOF + `noeviction`) | Giỏ hàng, số gọi quầy, cache menu, Live Loyalty QR, JWT denylist, rate-limit |
| **Message Broker** | Apache Kafka 3.9 (KRaft mode, 3 partitions) | Truyền sự kiện bất đồng bộ bền vững kết hợp Transactional Outbox & DLT |
| **Hạ tầng & Tích hợp** | Docker Compose (**18 containers** local), Firebase, SePay, Mapbox, Cloudinary, Gmail SMTP | Xác thực Google, đối soát ngân hàng, bản đồ/tuyến đường, lưu ảnh chứng từ |

---

## 2. Sơ đồ kiến trúc & 12 Module Backend

```mermaid
flowchart TB
    FE1["Customer UI<br/>(Next.js 15 · :3000)"] --> GW["API Gateway<br/>(:8080 · JWT + Redis Denylist)"]
    FE2["Staff / Manager / Admin UI<br/>(React 19 + Vite · :3001)"] --> GW

    CS["config-server"] -.->|"Cấu hình tập trung"| GW
    EU["discovery-server<br/>(Eureka)"] -.->|"Service Discovery lb://"| GW

    GW --> ID["identity-service<br/>(schema: identity_service)"]
    GW --> CAT["catalog-store-service<br/>(schema: catalog_service)"]
    GW --> ORD["cart-order-service<br/>(schema: order_service)"]
    GW --> PAY["payment-service<br/>(schema: payment_service)"]
    GW --> PRO["promotion-loyalty-service<br/>(schema: promotion_service)"]
    GW --> SHP["shipping-service<br/>(schema: shipping_service)"]
    GW --> NOT["notification-service<br/>(schema: notification_service)"]
    GW --> REV["review-report-service<br/>(schema: review_report_service)"]
    GW --> AUD["audit-service<br/>(schema: audit_service)"]

    ORD <-->|"REST nội bộ"| CAT
    ORD <-->|"REST nội bộ"| PAY
    ORD <-->|"REST nội bộ"| PRO
    ORD <-->|"REST nội bộ"| SHP

    ORD --- PG[("PostgreSQL 16<br/>(DB-per-schema)")]
    PAY --- PG
    CAT --- PG
    ID --- PG
    PRO --- PG

    ORD <--> RD[("Redis 7")]
    PRO <--> RD
    ID <--> RD
    CAT <--> RD
    GW <--> RD

    ORD <-->|"Outbox -> Kafka -> DLT"| KF["Apache Kafka 3.9 (KRaft)"]
    PAY <--> KF
    CAT <--> KF
    PRO <--> KF
    SHP <--> KF
    ID <--> KF
    NOT <-- KF
    REV <-- KF
    AUD <-- KF
```

### Bảng 12 module backend thực tế

| Nhóm | Service (`backend/*`) | Schema PostgreSQL / Nhiệm vụ chính |
|---|---|---|
| **Hạ tầng Spring Cloud** | `config-server` | Đọc `config-repo/*.yml` phân phối cấu hình tập trung cho toàn hệ thống |
| | `discovery-server` | Eureka Service Registry để Gateway & các service gọi nhau qua tên dịch vụ |
| | `api-gateway` | Điểm vào duy nhất (`/api/**`), validate JWT + Redis denylist, rate-limit IP, gắn `X-User-*` |
| **Nghiệp vụ lõi đặt hàng** | `identity-service` | `identity_service`: Tài khoản, RBAC, OTP, Google OAuth, quản lý phiên thiết bị & thu hồi JWT |
| | `catalog-store-service` | `catalog_service`: Món ăn, danh mục, combo, chi nhánh, tồn kho & `order_stock_ledger` |
| | `cart-order-service` | `order_service`: Giỏ hàng Redis, đặt đơn Web/POS, State Machine, hoàn tiền, đơn tiệc |
| | `payment-service` | `payment_service`: VietQR SePay webhook, thu tiền quầy, đối soát, két ca, giữ tiền hoàn |
| | `promotion-loyalty-service` | `promotion_service`: Khuyến mãi, voucher cá nhân, ví điểm, Live Loyalty QR 30s |
| **Hậu cần & Hỗ trợ** | `shipping-service` | `shipping_service`: Tính phí/phạm vi giao, tạo bản ghi shipment, `MockDeliveryEngine` |
| | `notification-service` | `notification_service`: Thông báo in-app & pipeline gửi email SMTP có chống lặp |
| | `review-report-service` | `review_report_service`: Đánh giá món (`COMPLETED`) & báo cáo thống kê doanh thu |
| | `audit-service` | `audit_service`: Lưu trữ nhật ký kiểm toán tập trung từ topic `audit.recorded` |

---

## 3. Quy tắc kiến trúc bất biến

1. **Thứ tự khởi động chuẩn (`AGENTS.md`):** Hạ tầng (Postgres, Redis, Kafka) → `config-server` → `discovery-server` (Eureka) → Business services → `api-gateway` → Frontends.
2. **Gateway là cổng duy nhất cho Frontend:** Next.js và React chỉ gọi `api-gateway` (`:8080`), tuyệt đối không gọi thẳng từng business service.
   - `IdentityHeaderPropagationFilter` (`Order = -100`) ở Gateway **xóa sạch header giả mạo** (`X-User-Id`, `X-User-Role`, `X-Store-Id`) từ client gửi lên và chỉ inject lại từ JWT đã kiểm chữ ký + kiểm Redis denylist.
3. **DB-per-service — Cấm JOIN chéo schema:** Service nào sở hữu schema của service đó, chỉ liên kết qua ID (`orderId`, `storeId`, `customerId`). Mọi dữ liệu liên quan qua **REST nội bộ** hoặc **Kafka Event**.
4. **Phân quyền chi nhánh (`storeId`) ép tại Backend:** `STAFF` và `STORE_MANAGER` bị ép dùng `storeId` lấy từ JWT (`AccessGuard.requireStore`), không tin query param `storeId` từ client.
5. **Tách biệt MVP vs Phase 2:** MVP tạo bản ghi `shipment` thật và chạy mô phỏng giao hàng ở local (`SHIPPING_MOCK_ENABLED=true`). Tích hợp hãng giao hàng thật (Grab/Ahamove/Be) thuộc Phase 2.

---

## 4. Khi nào dùng REST đồng bộ vs Kafka bất đồng bộ?

| Tiêu chí | REST nội bộ (Synchronous) | Kafka + Outbox (Asynchronous) |
|---|---|---|
| **Mục đích** | Cần kết quả **tức thì** mới đi tiếp được bước hiện tại | Phát sự kiện sau khi transaction đã chốt; không cần chờ service kia trả lời ngay |
| **Ví dụ trong Clucky** | - `CartOrder` → `Catalog`: validate giá/giờ bán & `reserveStock`<br>- `CartOrder` → `Shipping`: tính phí ship & phạm vi giao<br>- `CartOrder` → `Loyalty`: preview/giữ voucher & điểm<br>- `Loyalty` → `CartOrder`: `GET /internal/loyalty/orders/{id}` xác minh trạng thái thật (SSOT) trước khi chốt điểm | - `order.created`, `order.status-changed`, `order.completed`, `order.cancelled`<br>- `payment.received` ↔ `order.payment-resolved` ↔ `payment.confirmed`/`failed`<br>- `shipping.started`, `shipping.delivered`, `shipping.failed`, `shipping.resolved`<br>- `refund.approved`/`rejected`/`waiting-bank-details`/`processed`<br>- `audit.recorded`, `email.requested`, `inventory.low-stock` |
| **Rủi ro & Cách xử lý** | Nếu service kia lỗi/chậm → chặn/rollback ngay hoặc trả lỗi cho người dùng | Độ trễ vài trăm ms – vài giây → dùng Outbox + Retry + Idempotent Consumer + DLT |

---

## 5. Ngôn ngữ nghiệp vụ chuẩn khi vấn đáp (`CONTEXT.md`)

Dùng sai từ chuyên môn dễ bị mentor hỏi vặn. Bảng dưới đây là thuật ngữ thống nhất của dự án:

| Thuật ngữ chuẩn nên nói | Ý nghĩa đúng trong Clucky | Từ **TRÁNH** nói nhầm |
|---|---|---|
| **Tài khoản tác nghiệp chi nhánh** | Tài khoản `STAFF` dùng chung cho nhân viên tại quán (tối đa 3 phiên mặc định) | Không gọi là "tài khoản nhân viên cá nhân" |
| **Tài khoản quản lý chi nhánh** | `STORE_MANAGER` chỉ quản lý 1 chi nhánh được gán | Không gọi là "Admin chi nhánh" hay "Chủ hệ thống" |
| **Đơn tại quầy (`COUNTER`)** | Nhân viên nhập nhanh tại POS cho khách ở quán (`customerId = null`) | Không gọi là "đơn offline mất mạng" |
| **Thanh toán tại quầy** | Nhân viên trực tiếp thu tiền mặt / VietQR / SPLIT | Không gọi "offline" theo nghĩa mất kết nối Internet |
| **Thu tiền (`PAID` / `COLLECTED`)** | Ghi nhận tiền thực tế đã vào két/ngân hàng | Không đánh đồng là "hoàn thành đơn (`COMPLETED`)" |
| **Duyệt hoàn tiền (`APPROVED`)** | Chấp thuận yêu cầu và giữ tiền (`HELD`) trên sổ cái | Không nói là "đã hoàn tiền xong" (`REFUNDED`) |
| **Tiền chuyển dư (`excessAmount`)** | Phần tiền khách chuyển vượt giá trị đơn | Không tính là "doanh thu tăng thêm" |
| **Xác nhận thu thủ công** | Thao tác nghiệp vụ của nhân sự quầy | Không nói là "sao kê ngân hàng đã đối soát" |

---

## 6. Nhịp đồng bộ dữ liệu trên 3 giao diện

Khi mentor hỏi *"Làm sao khách, nhân viên và quản lý thấy trạng thái khớp nhau mà không bị lag?"*:

| Màn hình | Cơ chế & Chu kỳ đọc dữ liệu | Ghi chú kỹ thuật |
|---|---|---|
| **Khách — Chi tiết đơn** (`/orders/[id]`) | Poll `GET /api/orders/{id}` mỗi **5 giây** | Tự dừng khi đơn chạm trạng thái cuối (`COMPLETED`/`CANCELLED`/`DELIVERY_FAILED`) |
| **Khách — Khung VietQR** (`LiveOrderPayment`) | Poll `GET /api/payments/{orderId}` mỗi **5 giây** | Đổi sang **15 giây** khi không còn `PENDING`; lỗi mạng backoff tối đa **30 giây** |
| **Khách — Bản đồ giao hàng** (`MockDeliveryTracking`) | Poll mỗi **1.5 giây** khi đang giao | Nội suy vị trí theo `serverNow` và `due_at` trên tuyến mô phỏng |
| **Staff Workspace** (`StaffOrdersPage`) | Cập nhật ngay từ **HTTP response** + poll ngầm mỗi **5 giây** (15s nếu tab ẩn) | Có âm thanh báo đơn mới (**Audio chime**) và cảnh báo viền đỏ SLA nếu chờ **> 10 phút** |
| **Store Manager / Admin** | Manager poll mỗi **5 giây**; Admin list mỗi **15 giây** (tạm dừng khi mở drawer) | Sổ quỹ & Báo cáo tải khi mở/đổi lọc hoặc cập nhật theo projection Kafka |

---

## File code mở chứng minh ngay
- Định tuyến Gateway & filter bảo mật: `config-repo/api-gateway.yml`, `backend/api-gateway/src/main/java/com/clucky/apigateway/config/IdentityHeaderPropagationFilter.java`
- Cấu hình dùng chung toàn hệ thống: `config-repo/application.yml`, `config-repo/cart-order-service.yml`
- Ngôn ngữ nghiệp vụ & quy tắc kiến trúc: `CONTEXT.md`, `AGENTS.md`
