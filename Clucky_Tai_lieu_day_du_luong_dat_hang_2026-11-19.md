Tài liệu tham khảo bảo vệ

Luồng đặt hàng Clucky

Từ giỏ hàng khách đến nhân viên, bếp, nhận món và quản lý theo ngày. Trọng tâm là giải thích từng lựa chọn dẫn tới trạng thái nào, ai được chuyển bước và dữ liệu nào bảo vệ quyết định đó.

Phiên bản khảo sát ngày 19 tháng 11 năm 2026, commit 18c3ea4527a9090efbcd3ff6232546ff2a554552. Nhánh làm việc codex/fix-delivery-cod-settlement đã fast-forward tới origin/main. Nội dung source trước và sau fast-forward giống nhau vì main vừa merge bản sửa COD này.

## Bản đồ nội dung

Trang 2–7 giải thích dữ liệu, Redis, checkout, ma trận nhánh và sơ đồ.

Trang 8–17 lần theo UI khách/staff, tiền, giao mô phỏng, trạng thái và ba đơn mẫu.

Trang 18–25 tra đồng bộ, quyền, transaction, tích hợp, tồn, Excel, két và ngoại lệ.

Trang 26–30 dành cho phần tâm đắc, hỏi đáp, giới hạn và các đoạn code cần mở.

## Các phát hiện cần cập nhật từ bản cũ

Giỏ guest sống 24 giờ, giỏ tài khoản 7 ngày. Thêm/sửa đặt lại TTL, đọc giỏ không gia hạn. Các giá trị CartService đang dùng là hằng số.

POS chưa thanh toán được UI gom vào Chờ tiền dù Order có thể CONFIRMED, PREPARING hoặc READY. Tab công việc là phép phân loại giao diện.

POS có đổi phương thức trước khi bắt đầu thu. Order dùng counterPaymentStarted và kiểm hoạt động ở Payment để ngăn đổi sau khi đã mở phiên hoặc nhận khoản thu.

Tiền ứng COD mô phỏng được lưu merchantPaymentStatus trên Order, dùng để hiển thị nhất quán hơn ở khách, staff và quản lý. paymentStatus vẫn theo dõi khoản thanh toán thực của đơn.

## Đọc tài liệu như bộ khung

Mỗi phần có quyết định nghiệp vụ, thao tác UI và vị trí code. Dùng các ý để tự kể theo nhịp của mình. Những câu ghi đóng góp cá nhân cần đối chiếu PR/hàm đã trực tiếp làm trước buổi bảo vệ.

Bằng chứng chính · CartService · OrderService · ShippingOrderProjection · StaffOrdersPage.queueCounts/visibleOrders · CounterPaymentModal.changeMethod. Các file tra cứu được liệt kê ở cuối tài liệu.

Nền tảng nghiệp vụ

# Dữ liệu phân biệt các nhánh

| Dữ liệu | Giá trị điển hình | Quyết định |
| --- | --- | --- |
| orderSource | WEB / COUNTER | Khách đặt web hay staff nhập POS. Tự lấy do khách đặt vẫn WEB. |
| type | DELIVERY / PICKUP | Giao tận nơi hay nhận tại quầy. POS lưu PICKUP. |
| diningOption | DINE_IN / TAKE_AWAY | Phân biệt ăn tại chỗ và mang đi ở POS. |
| paymentMethod | COD / CASH / BANK_TRANSFER / SPLIT | Chọn phương thức, theo phạm vi hỗ trợ từng kênh. |
| status | OrderStatus | Tiến độ đặt, làm món, giao và kết thúc. |
| paymentStatus | PENDING / PAID / COLLECTED | Khoản thanh toán trên đơn. CONFIRMED chưa suy ra đã trả. |
| merchantPaymentStatus | PAID_SIMULATED / ADVANCE_RETURNED | Tiền ứng mô phỏng của shipper cho quán, được lưu riêng. |

“Online giao tận nơi” và “đặt trước tự lấy tại quầy” là cách kể nghiệp vụ. Nếu gọi tự lấy là offline, giải thích đó là cách nhận món. Backend phân biệt bằng WEB/PICKUP và COUNTER/PICKUP. Tiền mặt tự lấy web hiện dùng mã COD tương thích trong request, còn lời trình bày nên gọi tiền mặt tại quầy.

## Các service tham gia

Hai frontend đi qua Gateway. Cart Order sở hữu đơn, lịch sử và outbox. Catalog Store cung cấp menu, giá và tồn chi nhánh. Payment quản lý phiên và khoản thu. Promotion Loyalty kiểm voucher/ví điểm. Shipping báo phí, ghi shipment và mô phỏng giao. Review Report nhận đánh giá và dựng số liệu. Identity cung cấp actor, xác thực và liên hệ, Notification nhận sự kiện để gửi theo điều kiện.

REST nội bộ và Kafka trao đổi dữ liệu giữa các service. Mỗi service sở hữu schema, sử dụng ID để liên kết nghiệp vụ. Eureka và Config Server cung cấp discovery và cấu hình. Redis dùng cho giỏ và số thứ tự trong luồng này.

## Quy ước đường dẫn dùng trong các trang

Cart = backend/cart-order-service/src/main/java/com/clucky/cartorderservice/. Tương tự Payment = payment-service/paymentservice, Catalog = catalog-store-service/catalogstoreservice, Shipping = shipping-service/shippingservice, Loyalty = promotion-loyalty-service/promotionloyaltyservice, Review = review-report-service/reviewreportservice. Ví dụ Cart/service/OrderService.java là đường dẫn tương đối đầy đủ khi ghép với tiền tố.

config-repo/api-gateway.yml · Cart/entity/Order.java · Cart/dto/OrderStatus.java · frontend/react/src/lib/order-fulfillment.ts · AGENTS.md.

Giỏ Redis

# Danh tính giỏ và thời hạn lưu

## Tổ chức dữ liệu

CartController.resolveContext chọn giỏ theo actor. CUSTOMER dùng cart:{userId}. Guest dùng cart:guest:{sessionId}, ưu tiên header X-Guest-Session-Id, tiếp đến cookie clucky_guest_session, HTTP session có sẵn, cuối cùng tạo UUID. RedisConfig.key kiểm định danh. Staff đăng nhập vào API giỏ khách bị từ chối theo vai trò.

Một giỏ là Redis Hash. Field là itemKey, value là JSON CartItemDto. getCart đọc các field, parse món và tính CartDto. localStorage trình duyệt giữ guest session ID để nhận lại cùng giỏ, còn nội dung giỏ được tải qua API. Redis key hết hạn thì API trả giỏ rỗng.

| Loại hoặc thao tác | TTL | Hành vi tìm thấy |
| --- | --- | --- |
| Guest | 24 giờ | GUEST_CART_TTL = Duration.ofHours(24). |
| Tài khoản khách | 7 ngày | CUSTOMER_CART_TTL = Duration.ofDays(7). |
| Thêm món | Đặt lại TTL | HSET xong gọi expire(cartKey, ttl), áp dụng cả khi cộng số lượng món trùng. |
| Sửa món số lượng dương | Đặt lại TTL | updateItem hoặc updateItemQuantity lưu xong gọi expire. |
| Đọc giỏ | Giữ hạn cũ | getCart không gọi expire. |
| Xóa một món | Giữ hạn còn lại | removeItem và nhánh số lượng không dương xóa field, không gia hạn. |
| Merge có giỏ guest | 7 ngày cho giỏ đích | Gộp vào cart:{userId}, expire 7 ngày rồi xóa giỏ guest. |

## Ví dụ thời gian để giải thích

Guest thêm món lúc 10 giờ thứ Hai. Nếu chỉ mở xem lại lúc 9 giờ thứ Ba, hạn vẫn là 10 giờ thứ Ba. Nếu sửa món dương lúc 9 giờ thì hạn chuyển tới 9 giờ thứ Tư. Với giỏ tài khoản, cùng quy tắc đặt lại nhưng khoảng thời gian là 7 ngày. Đây là suy luận từ EXPIRE của các hàm ghi.

Đóng trình duyệt không dừng đồng hồ TTL. Mất guest ID có thể khiến mở một giỏ mới dù key cũ còn hạn. Đăng xuất chuyển danh tính giỏ ở UI, chưa tìm thấy lệnh xóa cart:{userId} chỉ vì đăng xuất.

Cart/cart/CartService.java · GUEST_CART_TTL, CUSTOMER_CART_TTL, getCart, addItem, updateItem, removeItem, mergeCart. Cart/cart/CartController.java · resolveContext. frontend/nextjs/lib/api.ts · getGuestSessionId.

Giỏ Redis

# Gộp giỏ tùy chọn combo và giới hạn

## Cùng món được nhận diện thế nào

CartItemDto.generateItemKey băm MD5 chuỗi loại sản phẩm hoặc combo, ID, option/topping đã sắp xếp và lựa chọn từng thành phần combo. Cùng cấu hình được cộng số lượng. Khi sửa tùy chọn, updateItem sinh key mới, gộp khi đụng field đã có rồi xóa field cũ. MD5 ở đây tạo định danh cấu hình món.

itemKey không chứa note hoặc storeId. Ghi chú khác có thể đi vào cùng dòng. Khi thêm trùng, code có thể lấy ghi chú mới không rỗng. Suy luận từ mã, cần kiểm trường hợp cùng cấu hình qua chi nhánh khác và các thao tác song song để đánh giá đủ hành vi.

## Đăng nhập và gộp

OrderProvider thấy guest ID thì gọi mergeCart, dùng navigator.locks khi trình duyệt hỗ trợ để giảm chạy gộp đồng thời giữa các tab. Backend cộng số lượng theo itemKey, ghi giỏ tài khoản, đặt TTL 7 ngày và xóa guest. Sau đó UI GET giỏ tài khoản. Client gửi guestSessionId trong body, DTO backend đọc sessionId, đường hiện tại còn truyền header X-Guest-Session-Id để backend lấy dự phòng. Caller độc lập cần theo đúng DTO/header.

## Những giới hạn quan trọng

Thêm, sửa và merge sử dụng nhiều lệnh đọc/ghi Redis. Chưa thấy Lua, WATCH/MULTI hoặc khóa Redis cho toàn bộ chuỗi này. Khóa phía trình duyệt chỉ bảo vệ trong phạm vi nó thực sự chạy. Hai request độc lập vẫn có rủi ro mất cập nhật, đây là suy luận từ mô hình read-modify-write.

Cấu hình chung có clucky.redis.ttl.cart và guest-cart, nhưng CartService dùng hằng số 7 ngày/24 giờ. Thay biến cấu hình đó chưa đủ chứng minh thay đổi TTL giỏ. spring.session.timeout=24h và namespace clucky:cart:sessions là HTTP session, khác key nội dung giỏ.

Compose backend bật Redis AOF, volume và noeviction với maxmemory 192mb. Volume giữ dữ liệu khi container khởi động lại trong điều kiện Redis lưu/khôi phục được, còn TTL vẫn có hiệu lực. Khi hết bộ nhớ, lệnh ghi có thể lỗi theo noeviction. Thêm vào giỏ chưa giữ tồn hàng và chưa khóa giá.

Cart/cart/CartItemDto.java · generateItemKey. Cart/cart/CartService.java · updateItem/mergeCart. frontend/nextjs/components/order/order-provider.tsx · effect loadCart. config-repo/application.yml:61–68 · config-repo/cart-order-service.yml · deploy/backend/compose.yml:63–70.

Checkout

# Dữ liệu đầu vào và giá backend

## Đường đi UI

Menu hoặc chi tiết món/combo → thêm giỏ → /cart → /checkout. Khách chọn người nhận, chi nhánh, Giao tận nơi hoặc Tự đến lấy tại quán, ưu đãi và thanh toán. Preview cập nhật tổng trước khi bấm Đặt hàng ngay. Giao tận nơi cần địa chỉ, tọa độ và nằm trong phạm vi phục vụ.

## Backend tin dữ liệu nào

| Trường | Client đưa vào | Backend xử lý |
| --- | --- | --- |
| Món và số lượng | productId hoặc comboId, quantity, option/topping, comboSelections | Catalog kiểm món bán, thành phần combo và định giá. Order dùng kết quả đã xác minh. |
| Giá và tổng | Giá trong giỏ, expectedTotal | Tính lại bằng BigDecimal. expectedTotal lệch trả 409 RESOURCE_CONFLICT. |
| Chi nhánh | storeId | Kiểm món trong giỏ tương thích chi nhánh, giờ bán và tồn. POS còn kiểm chi nhánh token. |
| Cách nhận | type, địa chỉ, tọa độ | Chỉ DELIVERY/PICKUP cho web. Giao cần địa chỉ và cặp tọa độ hợp lệ. Shipping trả phí/phạm vi. |
| Ưu đãi | voucherInstanceId, pointsUsed | Promotion Loyalty xác minh. Web từ chối voucherCode gõ trực tiếp. |
| Khách và trạng thái | Thông tin người nhận | customerId theo actor. Trạng thái khởi đầu do backend quyết định. |

Công thức tổng trong createOrder lấy subtotal từ Catalog, trừ voucher và giảm hạng, cộng phí giao, trừ điểm, chặn số âm về 0. Không hardcode giá từ client làm tổng cuối. Combo có snapshot option/topping theo từng thành phần để lưu và hiển thị đúng món cần làm.

## Gửi đang chờ thành công hoặc lỗi

handleSubmitOrder khóa bằng submitLock và submitting. Lần đặt dùng khóa yêu cầu gắn hash của payload, giữ qua timeout/reload trong sessionStorage. Thành công mở /orders/{id}. Lỗi mở lại thao tác, hiện lỗi và tải lại preview/điểm. 401 chuyển đăng nhập. Backend dọn giỏ Redis tại bước kết thúc tạo web.

DTO có scheduledAt và Order lưu giá trị này, nhưng checkout khảo sát chưa có chọn giờ hẹn. Đặt trước tự lấy hiện có nghĩa đặt trước khi đến quầy.

frontend/nextjs/app/checkout/page.tsx · handleSubmitOrder. Cart/service/OrderService.java · previewOrder, createOrder, applyValidatedSnapshots. Catalog/service/ValidationService.java · validateOrderItems. Cart/dto/CreateOrderRequest.java.

Nhánh đặt hàng

# Ma trận nhận món và thanh toán

Các trạng thái dưới đây là sau khi giữ tồn và hoàn tất tạo đơn. Trước đó DB có bản nháp PENDING_STOCK. Mỗi cặp viết theo status / paymentStatus trên Order.

| Nhánh | Ban đầu | Điều kiện tiếp tục |
| --- | --- | --- |
| Web giao ngân hàng | PENDING_PAYMENT / PENDING | Order chấp nhận tiền → CONFIRMED / PAID → chế biến → READY → giao mô phỏng. |
| Web tự lấy ngân hàng | PENDING_PAYMENT / PENDING | Cùng đoạn chờ tiền. READY và đã trả thì nhân viên xác nhận khách nhận tại quầy. |
| Web giao COD | CONFIRMED / PENDING | Chế biến trước. READY kích hoạt giao khi đủ điều kiện. Tiền ứng cho quán lưu merchantPaymentStatus. |
| Web tự lấy tiền mặt | CONFIRMED / PENDING | Chế biến trước. Thu tại quầy được chấp nhận → COLLECTED, rồi bàn giao. |
| POS tiền mặt | CONFIRMED / PENDING | UI gom vào Chờ tiền. Có thể thu ngay sau tạo hoặc thu sau. Khoản thu được chấp nhận → COLLECTED. |
| POS ngân hàng | CONFIRMED / PENDING | UI gom vào Chờ tiền, mở QR. SePay + Order xác nhận → PAID, đơn quay về nhóm theo tiến độ. |
| POS kết hợp | CONFIRMED / PENDING | Chốt phần tiền mặt, QR cho số còn lại. Khi được chấp nhận → COLLECTED. |

## Hai tầng cần diễn giải riêng ở POS

Backend cho CONFIRMED → PREPARING khi hợp lệ mà chưa có điều kiện riêng bắt POS phải trả tiền trước chế biến. Thẻ OrderCard ở CONFIRMED cũng còn nút Bắt đầu nấu. Tuy nhiên danh sách gom POS còn PENDING vào Chờ tiền và loại khỏi các nhóm Mới xác nhận/Đang chuẩn bị/Sẵn sàng. Vì vậy không mô tả tab Chờ tiền như bằng chứng status=PENDING_PAYMENT.

Sau thu tiền, đơn trở lại nhóm đúng status ở lần đọc tiếp theo. Theo dõi bằng mã đơn để tìm lại khi nhóm thay đổi. Với CASH thu tự động lúc tạo, effect chờ kết quả Order rồi chuyển tab Mới xác nhận khi tiền đã PAID/COLLECTED.

## Đơn miễn thanh toán

Tổng 0 được lưu CONFIRMED / COLLECTED với collectionMethod=NO_PAYMENT. Web chuẩn hóa phương thức cũ COD, POS dùng CASH. Luồng bỏ qua tạo khoản thu bằng 0.

Cart/service/OrderService.java · createOrder dòng 492–494, createCounterOrder dòng 951–966. frontend/react/src/pages/Staff/StaffOrdersPage.tsx · queueCounts/visibleOrders và effect chờ thu tiền. components/OrderCard.tsx.

Luồng tổng thể

# Từ lựa chọn đến hoàn tất

Sơ đồ thể hiện nhánh thành công với tổng dương. Vùng UI Chờ tiền của POS có thể bao trùm status CONFIRMED, PREPARING hoặc READY. Đổi tab hiển thị không phải là một lệnh chuyển trạng thái backend.

Cart/service/OrderService.java · createOrder/createCounterOrder/updateStatus. frontend/react/src/pages/Staff/StaffOrdersPage.tsx · visibleOrders. Cart/service/ShippingOrderProjection.java · apply.

UI khách hàng

# Những đoạn có thể gộp

## Tạo đơn web

Chọn món → giỏ → checkout → chọn nhận và trả tiền → Đặt hàng ngay → chi tiết đơn. Giao và tự lấy dùng chung màn checkout, chỉ khác dữ liệu cần nhập, phí và bước kết thúc. Hai nhánh ngân hàng dùng chung khung VietQR và chờ xác nhận.

| Dữ liệu Order | Nhãn khách điển hình | Nội bộ nhìn thấy |
| --- | --- | --- |
| PENDING_PAYMENT | Chờ thanh toán, khung VietQR và thông tin phiên. | Tab Chờ tiền, thẻ web có Chờ khách thanh toán QR để xem. |
| CONFIRMED | Đã xác nhận. Thông báo diễn giải còn phụ thuộc phương thức và đơn 0 đồng. | Có thể Bắt đầu nấu. Với POS, tab còn phụ thuộc trạng thái tiền. |
| PREPARING | Chế biến. | Đang chuẩn bị nếu đơn không thuộc nhóm POS chưa trả. |
| READY với PICKUP | Chờ khách đến lấy. | Thiếu tiền web → Thu tiền quầy. Đã trả → xác nhận nhận món. |
| READY với DELIVERY | Sẵn sàng giao, khung theo dõi shipment. | Xem giao hàng, đợi gán/nhận mô phỏng. |
| SHIPPING | Đang giao hàng. | Nhóm giao mô phỏng hoặc cần xử lý lượt lỗi. |
| COMPLETED | Đã nhận tại quán hoặc Đã giao hàng theo type. | Nhóm hoàn tất. Tiền mô phỏng vẫn được hiển thị riêng. |

## Trên cùng trang có nhiều nguồn dữ liệu

Chip đọc Order.status và ánh xạ theo type. Khung tiền đọc paymentStatus hoặc merchantPaymentStatus kết hợp thông tin phiên. Bản đồ đọc tracking của Shipping. Trong lúc sự kiện đang xử lý, các vùng có thể nhận dữ liệu ở các thời điểm khác nhau.

Khách xem danh sách đơn qua GET /api/orders/my và chi tiết theo ID hoặc mã. API kiểm chủ sở hữu. POS customerId=null nên đơn quầy không tự thành đơn lịch sử web của khách. Việc nhận điểm ví có thể được nối riêng qua Loyalty.

frontend/nextjs/lib/order-demo.ts · orderStatusKey. frontend/nextjs/app/orders/[id]/page.tsx · OrderTrackingPage. frontend/nextjs/components/payment/live-order-payment.tsx. Cart/service/OrderService.java · getMyOrders/buildOrderDetailsResponse.

UI nhân viên và POS

# Tạo nhanh thu tiền và đổi phương thức

## Tạo nhanh và nhóm làm việc

Đơn hàng → Nhập đơn nhanh hoặc F1 → Ăn tại chỗ/Mang đi → món/tùy chọn/combo → thanh toán → tạo. handleSubmit kiểm lại báo giá và hash, cần QR chứng thực còn hợp lệ nếu tiêu điểm. Thành công đóng modal, về ngày hiện tại. COUNTER còn PENDING chọn tab Chờ tiền, trường hợp khác vào Mới xác nhận.

Tiền mặt chọn đã thu lúc tạo thì POS gửi collection sau khi tạo thành công. Nếu lỗi, giữ đơn đã tạo và báo thu sau. Effect theo dõi kết quả thu để chuyển nhóm khi Order xác nhận. QR/kết hợp mở modal thanh toán từ đơn vừa tạo.

## Chức năng thu theo lựa chọn

| Phương thức | Modal và thao tác | Kết quả trên Order |
| --- | --- | --- |
| Tiền mặt | Tab Tiền mặt (CASH), nhập tiền khách đưa và xác nhận. | Khoản thu chấp nhận → COLLECTED. |
| Ngân hàng | SePay (VietQR), đọc phiên đã có. Lần đầu không có phiên có thể tự tạo. Có nút tạo QR khi cần. | Khớp ngân hàng và được Order chấp nhận → PAID. |
| Kết hợp | Nhập tiền mặt, khóa phần đó rồi tạo QR số còn lại. Khi có phiên, phần tiền mặt bị khóa. | Đủ phần ngân hàng và được chấp nhận → COLLECTED. |

## Đổi phương thức

CounterPaymentModal.changeMethod gọi onChangeMethod, StaffOrdersPage gửi PUT /api/orders/{id}/counter/payment-method kèm version. UI chỉ mở đổi khi chưa có phiên và chưa bận tải/ghi. Backend kiểm vai trò, chi nhánh, COUNTER còn PENDING, số tiền dương và status cho phép.

OrderService.changeCounterPaymentMethod hỏi Payment về hoạt động trước khi khóa Order, sau đó khóa và kiểm counterPaymentStarted. Payment claimCounterPayment khóa cùng dòng Order để đánh dấu đã bắt đầu thu. Khi claim thắng, đổi phương thức bị 409. Cùng phương thức vẫn có thể retry theo quy tắc phiên/khóa yêu cầu. Cờ đã claim không được hiểu như bằng chứng đã thu đủ tiền.

frontend/react/src/pages/Staff/components/CounterOrderModal.tsx · handleSubmit. CounterPaymentModal.tsx · changeMethod/createSePay. StaffOrdersPage.tsx · handleCounterOrderSuccess/handleChangeCounterPaymentMethod. Cart/service/OrderService.java:1159–1194 · OrderPaymentService.claimCounterPayment.

Thu tiền mặt và chứng từ

# Từ thao tác đến bản ghi

## Luồng thu có hai mốc

Staff mở Thu tiền quầy, nhập tiền khách đưa. POST /api/payments/counter-collection gửi orderId, orderCode, storeId, tổng phải thu, phương thức và tiền mặt. CounterPaymentService.collect kiểm quyền và tổng theo context Order, chống thu trùng, làm tròn tiền mặt tới 1.000 đồng theo HALF_UP, tính tiền thối và lưu biên lai.

Biên lai gắn collectionId, collectionStatus ban đầu có thể PENDING. Payment gửi payment.received với operationType=COLLECTION. Order khóa đơn, kiểm snapshot updatedAt, phạm vi, tổng và trạng thái, rồi ghi COLLECTED nếu chấp nhận. Sau quyết định ngược về Payment, biên lai/khoản thu được cập nhật. HTTP 201 chưa bảo đảm đã được bàn giao.

## Riêng web tự lấy tiền mặt

CodEvidencePanel trong modal yêu cầu ảnh bằng chứng. Upload kiểm loại ảnh, nội dung, tối đa 2 MiB và chi nhánh, lưu APPROVED ở đường hiện tại. PaymentOperations.collect yêu cầu ảnh phù hợp và tổng khoản thu khớp đơn. Tổng lẻ có thể vướng bước làm tròn của CounterPaymentService, đây là suy luận cần tự kiểm bằng dữ liệu lẻ.

| Loại | Dữ liệu và thời điểm | Xem hoặc in |
| --- | --- | --- |
| Biên lai khách | counter_payment_receipts có mã, đơn, tổng, tiền khách đưa/thối, thu ngân, trạng thái và printCount. | ReceiptBillModal dựng bill K80. Nếu có ID, POST /receipts/{id}/print tăng đếm. |
| Phiếu bếp KOT | Món, số lượng, option/topping, số gọi và ghi chú từ đơn. Mở sau khi chuyển PREPARING. | Xem lại trên thẻ. In mô phỏng nhả giấy, chưa thấy lưu bảng KOT riêng. |

QR POS có bản ghi Payment và bank_transactions, chưa thấy sinh biên lai tiền mặt riêng cho phiên này. In hiện là mô phỏng K80. Chưa thấy kết nối máy in nhiệt thật hoặc phát hành hóa đơn điện tử trong đường được khảo sát.

Payment/service/CounterPaymentService.java · collect/printReceipt. Payment/service/PaymentOperations.java · createCounterCollection/collect/uploadCodEvidence. Cart/service/OrderPaymentService.java · decideCollection. frontend/react/src/pages/Staff/components/ReceiptBillModal.tsx.

Chuyển khoản ngân hàng

# Phiên QR biên nhận và quyết định

## Khởi tạo

Web vào chi tiết đọc GET /api/payments/{orderId}, nếu chưa có và đang chờ thì POST /bank-transfer. PaymentFlow.initialize lấy dữ liệu đơn theo chủ, phương thức và trạng thái, ngân hàng được gán cho chi nhánh và hạn do Order mở. Phiên có transfer_content riêng, amount, tài khoản và expiresAt.

POS dùng POST /counter-bank-transfer với khóa yêu cầu và cashAmount. Tổng ngân hàng là tổng đơn trừ phần tiền mặt. Cần cấu hình SePay và counter-enabled. Payment kiểm đã có collection/biên lai/phiên khác, gọi mở hạn và claim Order, rồi lưu phiên cố định. Phiên cũ phù hợp có thể được đọc lại.

## Nhận tiền và đối soát

POST /api/payments/webhook/sepay được bảo vệ bằng khóa webhook. SepayReceiptService.receive lưu giao dịch SEPAY:{id}, fingerprint và thông tin tài chính. TransferReference đối chiếu tham chiếu, tài khoản, ngân hàng, tiền vào và tình huống cần rà soát. Đọc giao dịch hoặc scan QR chưa tự làm Order thành PAID.

PaymentFlow.reconcile khóa bản ghi, phân bổ giao dịch hợp lệ và cộng receivedAmount. Thiếu tiền giữ PENDING. Đủ hoặc dư theo điều kiện mới ghi đề xuất payment.received. OrderPaymentService.decide kiểm source, mã đơn, customer/store, tổng, hạn và quyết định cũ theo paymentId trước khi đổi Order.

## Kết quả cho mỗi nguồn

Web hợp lệ còn hạn chuyển PENDING_PAYMENT → CONFIRMED và tiền PAID. POS giữ tiến độ món hiện tại, đổi tiền PAID hoặc COLLECTED khi kết hợp. Order phát order.payment-resolved, Payment nhận ACCEPTED để chốt PAID và phát payment.confirmed. Nếu Order từ chối, theo decision EXPIRED/CANCELLED/SUPERSEDED hoặc trạng thái collection tương ứng.

## Hết hạn và nhận lặp

Hạn phiên mặc định 15 phút từ lúc mở phiên trên Order. OrderPaymentWorker quét đơn status PENDING_PAYMENT đã hết hạn mỗi 5 giây. POS QR hết hạn không đồng nghĩa worker này tự hủy mọi đơn CONFIRMED. Trùng bank_tx_id cùng nội dung chỉ tăng replay_count. Khác nội dung lưu conflict để rà soát. Tiền tới muộn giữ dấu vết phục vụ đối soát, không tự phục hồi đơn đã đóng.

config-repo/cart-order-service.yml · payment.session-ttl. Payment/service/PaymentFlow.java · initialize/initializeCounter/reconcile/orderEvent. SepayReceiptService.receive. Cart/service/OrderPaymentService.java · open/openCounter/decide/expire. OrderPaymentWorker.expire.

Giao hàng

# MVP và hành trình mô phỏng

## Tạo bản ghi shipment

ShipmentServiceTemplate.createFromOrder nhận order.created sau khi Order hoàn tất tạo và ghi outbox. DELIVERY tạo shipment CREATED với orderId, storeId, customer, người nhận, địa chỉ, tọa độ, phí và cod_amount. Ngân hàng cod_amount=0. Tự lấy không tạo shipment. order_created_receipts và fingerprint chặn lặp/xung đột theo sự kiện và đơn.

Theo AGENTS.md, MVP tạo bản ghi giao. Bản local bật SHIPPING_MOCK_ENABLED trong compose.local.yml, còn cấu hình mặc định false. Giao thật/phân công thuộc Phase 2. Mock cần Mapbox/tuyến hợp lệ, dữ liệu tọa độ và payment snapshot đúng.

## READY tới giao xong

MockDeliveryEngine.acceptOrderStatus nhận READY của DELIVERY. reconcileReady lấy tuyến, ghi lượt giao và ASSIGNED. reconcileDeadlines/advance mô phỏng lấy món sau khoảng 5 giây rồi SHIPPING, giao sau thêm khoảng 20 giây và lưu proof SIMULATED_WATERMARK. Deadline được lưu trong DB. Tick mặc định 500 ms, còn độ trễ end-to-end phụ thuộc worker và Kafka.

Thiếu tuyến hoặc upstream lỗi có thể giữ CREATED/READY, lưu last_error_code và thử lại theo nhịp tối thiểu khoảng 15 giây ở đường lấy tuyến. Staff mở Xem giao hàng để theo dõi. Khách đọc tracking riêng và nội suy vị trí trên bản đồ từ dữ liệu mô phỏng.

## Thất bại trả lại và giao lại

Một lượt bị FAILED giữ Order SHIPPING để chờ quyết định. Staff được đánh dấu thất bại và xác nhận trả món theo điều kiện. Store Manager/Admin có giao lại hoặc đóng xử lý. Request mô phỏng dùng attemptNumber, version và khóa yêu cầu để tránh áp lên lượt khác. Đóng không giao được chuyển Order DELIVERY_FAILED.

ShippingOrderProjection.apply kiểm sourceService, SIMULATOR, simulationOnly, orderId/storeId/attempt, khóa Order và kiểm lượt cũ. EventId được ghi trong mock_delivery_receipts. Bản mới còn nhận RETURNED/CLOSED để tránh sự kiện giao cũ ghi đè tình trạng tiền ứng đã hoàn.

Shipping/service/ShipmentServiceTemplate.java · createFromOrder. MockDeliveryEngine · acceptOrderStatus/reconcileReady/advance/fail/returned/retry/close. Shipping/controller/MockDeliveryController.java. Cart/service/ShippingOrderProjection.java · apply.

COD mô phỏng

# Trạng thái tiền của quán và khách

## Quyết định mới trong code

Giao COD chưa có tiền khách thanh toán trên hệ thống dùng paymentStatus=PENDING. Khi shipper giả lập ứng tiền hàng cho quán, Shipping phát simulatedAdvanceConfirmed. ShippingOrderProjection ghi merchantPaymentStatus=PAID_SIMULATED ở Order khi nhận/giao hợp lệ. Khi trả món và hoàn ứng, ghi ADVANCE_RETURNED.

Giá trị ứng lấy merchant_goods_amount từ snapshot, bằng totalAmount trừ shippingFee, chặn dưới 0. Ledger DEBIT_ADVANCE/CREDIT_RETURN mô phỏng biến động ví. QR khách đã trả trước không cần ứng tiền hàng như COD.

## UI đọc theo hai dữ liệu

displayPaymentStatus ở React ưu tiên PAID_SIMULATED/ADVANCE_RETURNED khi paymentStatus còn PENDING. Các màn staff, quản lý và tài chính đã truyền merchantPaymentStatus. Khách dùng trường Order trước, tracking làm dự phòng. Điều này giúp giữ nhãn ứng tiền khi mở lại màn hình ngay cả lúc tracking chưa tải xong.

Đơn có thể COMPLETED / PENDING và merchantPaymentStatus=PAID_SIMULATED. Khi bảo vệ, nêu đơn đã giao theo mô phỏng, quán được ghi ứng tiền giả lập, hệ thống chưa ghi bằng chứng khách thực thu tiền. Phần hiển thị khác với thanh toán thật được chốt PAID/COLLECTED.

## Ảnh hưởng số liệu và Loyalty

Các công thức doanh thu thu thật và két dùng PAID/COLLECTED hoặc chứng từ phù hợp, không tự cộng vì thấy PAID_SIMULATED. publishSimulatedDeliveryCompleted chỉ phát order.completed khi đủ settlementEligible. COD chỉ có tiền ứng thường phát status-changed. LoyaltyOrderController chiếu hoàn tất COD chưa thu thành CANCELLED cho nhả khoản giữ, tránh cộng thưởng từ tiền mô phỏng.

## Dữ liệu có từ phiên bản cũ

V26__mock_cod_merchant_payment.sql thêm cột và index. Migration backfill PAID_SIMULATED cho đơn COD đã hoàn tất có đủ dấu giao/lịch sử mô phỏng. FlywayCompatibilityConfig cùng script db/compatibility/legacy-order-v25.sql xử lý cơ sở dữ liệu legacy theo kiểm tra trong code. Đây là phần nâng cấp schema, không phải bằng chứng môi trường đang chạy đã được migrate.

Cart/service/ShippingOrderProjection.java · recordMerchantPayment/change. Cart/entity/Order.java · merchantPaymentStatus. Cart migration V26. frontend/react/src/lib/order-fulfillment.ts · displayPaymentStatus. frontend/nextjs/app/orders/[id]/page.tsx · codMockPaid. Cart/controller/LoyaltyOrderController.java.

Vòng đời đơn

# Bảng chuyển trạng thái và quyền

| Nguồn và đích | Tác nhân và kiểm tra | Ý nghĩa |
| --- | --- | --- |
| Tạo → PENDING_STOCK | Order lưu nháp và claim idempotency. | Giữ tồn, còn có thể lỗi trước hoàn tất tạo. |
| PENDING_STOCK → PENDING_PAYMENT hoặc CONFIRMED | Order hoàn tất sau Catalog và ưu đãi. | Web ngân hàng chờ tiền, nhánh khác xác nhận. |
| PENDING_PAYMENT → CONFIRMED | Xác nhận tiền hợp lệ, hoặc API trạng thái khi tiền đã PAID/COLLECTED. | Cho vào đoạn làm món. |
| CONFIRMED → PREPARING | Staff/manager đúng store hoặc Admin. | Bắt đầu nấu và mở KOT. |
| PREPARING → READY | Cùng phạm vi quyền. | Món sẵn sàng. |
| READY → COMPLETED | API trạng thái yêu cầu PICKUP và tiền PAID/COLLECTED. | Xác nhận khách nhận tại quầy. |
| READY → SHIPPING → COMPLETED | Projection nhận sự kiện giao mô phỏng đúng điều kiện. | Giao tận nơi do Shipping điều khiển. |
| PENDING_PAYMENT hoặc CONFIRMED → CANCELLED | Hủy/từ chối đúng quyền, lý do. Hoặc hết hạn web. | Ngừng xử lý và bù theo điều kiện. |
| SHIPPING → DELIVERY_FAILED | Đóng xử lý không giao được theo Shipping. | Kết thúc nhánh giao lỗi. |

## Quy tắc backend ngăn thao tác sai

updateStatus dùng lockById, requireStore, kiểm version tùy chọn, trạng thái nguồn và tiền. Staff gửi paymentStatus trong request trạng thái bị 400 VALIDATION_ERROR. Chuyển sai bước hoặc dữ liệu cũ bị 409 RESOURCE_CONFLICT. Sai chi nhánh/quyền bị 403 FORBIDDEN. ApiError gồm timestamp, status, error, code, message, path.

Hủy sau PREPARING/READY qua các đường hủy thông thường bị chặn. DELIVERY không được staff gửi SHIPPING/COMPLETED/DELIVERY_FAILED qua API status. Gửi lại cùng đích có nhánh trả đơn hiện tại sau các kiểm tra trước đó.

Cart/service/OrderService.java · updateStatus dòng 1197–1290, cancellationReason. Cart/config/AccessGuard.java. Cart/dto/OrderStatus.java · chín enum. Cart/exception/GlobalExceptionHandler.java.

Đơn mẫu A

# Web giao tận nơi bằng ngân hàng

Mẫu A theo xuyên luồng. Chuẩn bị giỏ có tổng dương, chi nhánh mở và giao được tới tọa độ chọn. VietQR/SePay, ngân hàng chi nhánh và tuyến mô phỏng phải có cấu hình.

| Thao tác UI | Hàm hoặc endpoint | Dữ liệu quyết định | Kết quả |
| --- | --- | --- | --- |
| Đặt hàng ngay | CheckoutPage.handleSubmitOrder → POST /api/orders | DELIVERY, BANK_TRANSFER, món, store, địa chỉ/tọa độ, expectedTotal, key. | Bản nháp → PENDING_PAYMENT / PENDING. Mở chi tiết. |
| Xem mã QR | LiveOrderPayment → GET /api/payments/{id}, POST /bank-transfer khi cần. | Chủ đơn, tổng Order, tài khoản chi nhánh, hạn. | Payment PENDING, QR/nội dung nếu hợp lệ. |
| Ngân hàng báo tiền | SepayReceiptService → PaymentFlow.reconcile → payment.received → OrderPaymentService.decide. | Mã, account/bank, tổng, store/customer, đủ tiền và còn hạn. | CONFIRMED / PAID. UI staff vào nhóm nhận bếp. |
| Làm món | PUT /api/orders/{id}/status | CONFIRMED → PREPARING → READY, quyền và version. | KOT rồi sẵn sàng giao. |
| Mô phỏng giao | MockDeliveryEngine → shipping.started/delivered → projection. | READY, PAID, tuyến và proof. | SHIPPING → COMPLETED, Shipment DELIVERED. |

## Tại sao trạng thái đầu hợp lý

Lựa chọn ngân hàng trên web cần xác nhận trả trước. PENDING_PAYMENT giúp staff nhìn Chờ tiền và chưa dùng nhánh bắt đầu nấu. Catalog giữ tồn sau lưu nháp để tránh báo đặt thành công khi không còn món. expectedTotal là điều kiện phát hiện giá đã đổi.

## Ba màn hình cập nhật

Khách đọc chi tiết Order, khung QR và tracking riêng. Staff poll đơn đúng ngày/chi nhánh, sau thao tác có response để sửa state. Quản lý xem cùng mã trên danh sách, báo cáo tổng hợp có nguồn riêng. Khi chưa đến chu kỳ tải, nhãn có thể còn cũ.

frontend/nextjs/app/checkout/page.tsx · handleSubmitOrder. Cart/service/OrderService.java · createOrder. Payment/service/PaymentFlow.java · initialize/reconcile. Cart/service/OrderPaymentService.java · decide. ShippingOrderProjection.apply.

Đơn mẫu B

# Web tự lấy bằng tiền mặt

Mẫu B dùng Tự đến lấy tại quán và Thanh toán tiền mặt khi nhận tại quán. Chọn tổng tròn 1.000 đồng để tập đường thu hiện tại. request vẫn dùng giá trị COD nhưng nghiệp vụ là tiền mặt tại quầy.

| Thao tác UI | Hàm hoặc endpoint | Dữ liệu quyết định | Kết quả |
| --- | --- | --- | --- |
| Đặt hàng ngay | POST /api/orders → createOrder | WEB theo mặc định, PICKUP, phương thức tiền khi nhận. | CONFIRMED / PENDING, không tạo shipment. |
| Bắt đầu nấu | Staff.handleStatusChange → PUT …/status | Chi nhánh đúng, CONFIRMED, version. | PREPARING, mở KOT. |
| Đã xong món | PUT …/status | PREPARING → READY. | Khách thấy Chờ khách đến lấy. Staff Thu tiền quầy. |
| Lưu bằng chứng và thu | POST …/operations/cod-evidence; POST /api/payments/counter-collection. | Ảnh hợp lệ, tiền nhận, tổng và mã đơn. | Biên lai/collection có thể còn PENDING. Order chấp nhận → COLLECTED. |
| Xác nhận khách đã nhận | PUT …/status với COMPLETED | READY + PICKUP + PAID/COLLECTED. | COMPLETED. Khách Đã nhận tại quán. |

## Trường hợp lỗi để chỉ vào code

Gửi COMPLETED khi READY nhưng tiền PENDING bị 409. Đây là chốt độc lập với việc UI có hiện nút hay không. Khi khoản thu hợp lệ, bàn giao mới được lưu. Nếu collection tới khi updatedAt đã đổi, Order có thể trả STALE_VERSION cho khoản thu thay vì chấp nhận dữ liệu cũ.

## Giá trị cho phần tâm đắc

Cùng một đơn được làm trước, thu sau và bàn giao sau cùng. Tách status/paymentStatus giúp thể hiện đúng cả món và tiền. Các màn hình cùng đọc kết quả backend qua response/polling, không cần suy tiền từ tiến độ.

Cart/service/OrderService.java · createOrder/updateStatus. Payment/service/CounterPaymentService.java · collect. Payment/service/PaymentOperations.java · collect/uploadCodEvidence. Cart/service/OrderPaymentService.java · decideCollection.

Đơn mẫu C

# POS và thay đổi phương thức

Mẫu C bắt đầu từ staff cùng chi nhánh, Mang đi và tổng dương. Nhân viên chọn CASH và chưa thu ngay để quan sát rõ nhóm Chờ tiền. Có thể đổi sang ngân hàng trước khi phát sinh hoạt động thu.

| Thao tác UI | Hàm hoặc endpoint | Dữ liệu quyết định | Kết quả |
| --- | --- | --- | --- |
| Nhập đơn nhanh | CounterOrderModal.handleSubmit → POST /api/orders/counter. | COUNTER, PICKUP, TAKE_AWAY, món, báo giá, key bắt buộc. | CONFIRMED / PENDING trong DB, UI Chờ tiền. |
| Mở thu và đổi trước thu | CounterPaymentModal.changeMethod → PUT …/counter/payment-method. | Chưa có phiên/claim/hoạt động, version hợp lệ. | Phương thức lưu mới, modal ánh xạ response. |
| Thu tiền mặt hoặc QR | counter-collection, hoặc counter-bank-transfer. | Claim snapshot Order, tiền mặt/tổng QR, ngân hàng và key. | Đợi Order. COLLECTED cho tiền mặt/kết hợp, PAID cho QR toàn phần. |
| Theo dõi nhóm bếp | fetchOrders và queueCounts. | Tiền đã được chấp nhận, status hiện có. | Trở lại nhóm theo tiến độ. CONFIRMED hiện Mới xác nhận. |
| Nấu và bàn giao | PUT …/status. | Thứ tự trạng thái, scope, tiền. | PREPARING → READY → COMPLETED. |

## Nếu hai người đổi và thu gần nhau

Đổi phương thức hỏi Payment trước, rồi khóa Order và kiểm counterPaymentStarted. Thu tiền gọi claimCounterPayment khóa Order để so mã, store, phương thức và tổng rồi đặt cờ. Nếu claim thắng, đổi sang phương thức khác bị 409. Nếu đổi thắng, claim gửi phương thức cũ bị từ chối và không được âm thầm áp tiền cho phương thức mới.

Đây là cơ chế phối hợp xuyên service theo bản ghi Order. Không tuyên bố mọi thao tác đều ở một transaction chung. Nếu claim đã ghi nhưng bước sau lỗi, cùng phương thức có thể retry, cờ bảo vệ có thể vẫn giữ phương thức bị khóa.

OrderService.createCounterOrder/changeCounterPaymentMethod. OrderPaymentService.claimCounterPayment. PaymentFlow.initializeCounter/claimCounterPayment. PaymentOperations.createCounterCollection. CounterPaymentModal.changeMethod.

Đồng bộ ba giao diện

# Lúc nào dữ liệu được đọc lại

| Bề mặt | Nguồn và nhịp | Giới hạn |
| --- | --- | --- |
| Khách danh sách | OrderProvider.refreshOrders đọc /api/orders/my khi tải/session hoặc được gọi lại. | Không gán cơ chế poll chi tiết cho mọi danh sách. |
| Khách chi tiết | fetchCurrentOrder mỗi 5 giây tới trạng thái cuối. | Lỗi mạng có thể giữ dữ liệu cũ, trạng thái cuối ngừng poll Order. |
| Khung QR | LiveOrderPayment poll 5 giây lúc chờ, 15 giây khi đã đổi, backoff tới 30 giây khi lỗi. | Order chấp nhận tiền và Payment nhận quyết định là hai thời điểm. |
| Tracking khách | MockDeliveryTracking poll khi active khoảng 1,5 giây. | Map nội suy theo serverNow/deadline, không phải GPS tài xế thật. |
| Staff | fetchOrders tải đúng ngày/store, response sau thao tác. Poll thường 5 giây hoặc 15 giây nếu tab ẩn lúc lập interval. | Nhóm POS chưa trả tính từ source + paymentStatus + status. |
| Quản lý | Store Manager OrderContext và màn Orders poll khoảng 5 giây khi hiện. Admin list khoảng 15 giây khi không mở drawer. | Chi tiết mở có vòng đời tải riêng, cần đọc lại khi đổi. |
| Két và báo cáo | Két tải khi mở/đổi lọc/tải lại. Báo cáo dùng projection sự kiện. | Có độ trễ từ Kafka hoặc thời điểm tải. Tổng theo nguồn khác nhau. |

## Vì sao có thể thấy khác chữ hoặc khác thời điểm

READY/PICKUP của khách là Chờ khách đến lấy, staff nhóm Sẵn sàng, còn quản lý có badge theo nhãn của màn mình. Một dòng POS còn PENDING có thể nằm ở Chờ tiền dù badge tiến độ CONFIRMED. COD mô phỏng dùng displayPaymentStatus với merchantPaymentStatus riêng.

Trước khi kết luận xung đột, so mã đơn, chi nhánh, ngày lọc, timestamp đọc gần nhất, status, paymentStatus và merchantPaymentStatus. Nếu có collection hoặc shipment đang chờ, kiểm nguồn đó. Các màn hình hội tụ ở lần đọc sau theo dữ liệu đã lưu.

frontend/nextjs/app/orders/[id]/page.tsx · effect fetchCurrentOrder. LiveOrderPayment/MockDeliveryTracking. frontend/react/src/pages/Staff/StaffOrdersPage.tsx · effects polling/queueCounts. pages/Orders/AdminOrdersPage.tsx. IncomingTransfersPage.loadPaymentData.

Quyền và thao tác đồng thời

# Backend kiểm gì trước khi ghi

## Phạm vi dữ liệu

CUSTOMER tạo web và xem/hủy đơn của mình theo điều kiện. Staff/Store Manager xem, cập nhật và thu theo chi nhánh được gán. Admin có phạm vi hệ thống. AccessGuard.requireOwner/requireStore kiểm lại ở backend. List dùng effectiveStore, không tin storeId tùy ý từ staff để mở rộng truy cập.

Tạo POS chọn storeId hiệu lực từ actor và kiểm scope. Preview counter hiện từ chối requested store khác token. Các endpoint nội bộ phục vụ thanh toán có chính sách service credential riêng trong SecurityConfig, khác API cho người dùng.

## Hai người cập nhật một đơn

OrderRepository.lockById dùng PESSIMISTIC_WRITE. Trong updateStatus @Transactional, request tới sau lấy khóa sau rồi kiểm dữ liệu mới. Order có @Version, staff gửi version. Nếu version cũ, 409 và UI cảnh báo tải lại. Nếu bỏ version, backend vẫn kiểm bảng chuyển và quyền nhưng không so được ý định dựa trên phiên bản UI cũ.

OrderContext của quản lý có đường gọi updateStatus chưa gửi version. Điều này là giới hạn cần nêu khi nói chống xung đột. Bản ghi thu còn dùng snapshot updatedAt trong decideCollection, trả STALE_VERSION nếu đã đổi.

## Gửi lặp và phiên trùng

Web và POS có khóa thao tác phía UI, key + hash khi retry. DB web_order_operations khóa (customer_id, idempotency_key), counter_order_operations khóa (store_id, idempotency_key). Cùng key/hash trả lại đơn hoàn tất. Nháp còn PENDING_STOCK trả 409. Khác nội dung cùng key bị 409, POS thiếu key bị 400.

Web header vẫn tùy chọn, request trực tiếp thiếu key chưa có bảo vệ claim này. Key mới có thể tạo đơn mới. Biên nhận SePay chống lặp bằng bank_tx_id/fingerprint. Payment decisions và collection decisions ngăn chấp nhận tiền lặp. Giỏ Redis có phạm vi đồng thời riêng như phần đầu, không kế thừa khóa dòng Order.

Cart/config/AccessGuard.java · OrderRepository.lockById · entity/Order.java · OrderService.updateStatus/replayWebOrder/replayCounterOrder. Cart migrations V16/V19/V24. OrderPaymentService.decideCollection. frontend/react/src/context/OrderContext.tsx.

Transaction và phục hồi

# Phạm vi nguyên tử thực tế

## Khi tạo web hoặc POS

Bước 1 TransactionTemplate ghi nháp PENDING_STOCK và claim khóa yêu cầu. Bước 2 gọi Catalog reserveStock ngoài transaction DB Order. Bước 3 TransactionTemplate khóa nháp, gọi giữ voucher/điểm, lưu items, dữ liệu ưu đãi, trạng thái, lịch sử và outbox. Web xóa Redis cart ở bước kết thúc này.

Lưu PostgreSQL Order trong transaction chỉ bảo vệ các ghi ở DB Order. Ghi Redis, transaction Catalog hoặc Loyalty ở service khác không tự rollback cùng Order. Đây là điểm cần hiểu khi giải thích “đã dùng @Transactional”.

## Nếu bước sau thất bại

Reserve stock lỗi thì hủy nháp và phát hủy. Lỗi kết thúc thì rollback items/lịch sử/outbox của lần kết thúc, thử releaseStock, hủy nháp trong REQUIRES_NEW và ghi sự kiện bù. Công việc voucher release được theo dõi riêng theo orderId và được retry.

PendingStockRecoveryScheduler quét mỗi 60 giây, mặc định xử lý nháp quá 15 phút, giá trị timeout có chặn tối thiểu 10 phút. Nó khóa và kiểm lại nháp trước khi hủy, dọn claim cũ chưa gắn order. Vì giữ tồn và nháp có thể tồn tại sau lỗi, trả lỗi cho khách chưa đồng nghĩa mọi tài nguyên đã được trả tức thời.

## Outbox và Kafka

OrderEventPublisher.publish ghi outbox trong transaction bắt buộc. OutboxProcessor gửi sau commit, có job quét PENDING mỗi 5 giây. Event chứa eventId/correlationId, consumer dùng ID/ledger hoặc quyết định nghiệp vụ để xử lý lặp. Kafka không tự đảm bảo mọi hiệu ứng business chỉ chạy một lần.

Catalog ledger chống trừ/hoàn lặp theo orderId. Loyalty/voucher có bản ghi giữ và đối chiếu trạng thái Order. Payment có receipt/decision và outbox riêng. Khi phụ thuộc còn lỗi, worker có thể tiếp tục retry và người dùng thấy trạng thái trung gian.

Cart/service/OrderService.java · createOrder/createCounterOrder/markOrderCancelledWithCompensation/cancelPendingStockDraft. OrderEventPublisher.publish. OutboxProcessor. PendingStockRecoveryScheduler. VoucherCheckoutRecovery. Catalog/service/OrderStockService.java.

Voucher điểm đánh giá

# Kích hoạt dữ liệu và ảnh hưởng

## Voucher

Web chọn voucherInstanceId đã nhận vào tài khoản. Order từ chối voucherCode trực tiếp cho checkout web. POS có voucher theo báo giá quầy, kể cả nhánh voucher legacy cho khách vô danh theo quy tắc service. PromotionClient gửi storeId và giá món để kiểm. LoyaltyClient.reserveVoucher gửi orderId và snapshot giảm.

VoucherReservationService/PersonalVoucherService giữ lượt theo đơn và chốt khi COMPLETED đủ điều kiện. Hủy/đóng nhánh không đủ điều kiện nhả theo projection/recovery. VoucherCheckoutRecovery bản mới xem customer_voucher_instance_id, không chỉ voucher_code, để đưa đúng công việc nhả vào hàng chờ.

## Điểm giảm và điểm nhận

Web preview theo ví và số điểm yêu cầu, áp policy tỷ lệ, giảm hạng, trần mỗi đơn/ngày và số dư. Khi tạo giữ điểm cùng orderId. Hoàn thành hợp lệ mới chốt trừ điểm dùng và EARN, hủy nhả. Tiền COD chỉ mô phỏng không được tự thành đơn đủ điều kiện thưởng.

POS tra liên hệ có thể hỗ trợ nhận điểm, tiêu điểm cần QR/chứng thực gắn giỏ và còn hiệu lực. counterDraftHash chuẩn hóa món và sắp xếp để đối chiếu bản nháp. Nếu có ví loyalty thì ghi liên kết riêng dù Order.customerId vẫn null. Nếu không có ví/định danh phù hợp thì không có điểm để chốt cho khách.

GuestLoyaltyClaimService bản mới có đường nhận điểm guest vào tài khoản qua danh tính liên hệ đã xác minh. Đây là chức năng Loyalty được luồng đặt sử dụng gián tiếp, không tự biến đơn POS vô danh thành đơn lịch sử web của khách.

## Đánh giá và báo cáo

OrderItemReviews gọi POST /api/reviews với orderId/productId/rating/comment. ReviewService.createReview yêu cầu đúng chủ, COMPLETED, sản phẩm thuộc đơn và chưa trùng khách/đơn/món. Review gọi OrderVerification để kiểm lại. Lỗi đánh giá không làm đổi đơn. Report nhận order.completed/payment.confirmed theo cấu hình, projection có bảo vệ xử lý lặp.

Các module có owner và DB riêng. Khi trình bày đóng góp, nói rõ phần gọi API/phối hợp và phần trực tiếp viết, tránh quy hết implementation của module liên quan cho mình.

Cart/service/PromotionClient.java · LoyaltyClient. Loyalty/service/VoucherReservationService.java · PersonalVoucherService · LoyaltyService · CounterLoyaltyService · LoyaltyEvents · GuestLoyaltyClaimService. Review/service/ReviewService.java.

Tồn chi nhánh và combo

# Giữ hàng đúng một lần

## Món đơn và thành phần combo

ValidationService kiểm món, chi nhánh, giờ bán, option/topping và comboSelections. Combo có lựa chọn theo từng thành phần. Catalog trả subtotal, các snapshot đã xác thực và stockQuantities theo product thực tế cần trừ. Order lưu snapshot để bill/KOT thể hiện đúng cấu hình khách chọn.

CartItemDto.itemKey chứa cấu hình combo nên hai lựa chọn khác nhau có thể thành hai dòng. Khi Order chốt, snapshot từ Catalog mới là cơ sở định giá và làm món. UI/giỏ có thể giữ giá tạm để hiển thị, không quyết định số tiền cuối.

## Giữ tồn

Sau nháp, POST /api/catalog/internal/reserve-stock gửi orderId, storeId, quantities và expectedVersions nếu có. OrderStockService.reserveOrderStock tạo/khóa ledger theo orderId, kiểm store và quantities tương thích, duyệt sản phẩm theo thứ tự đã sắp. InventoryService.reserveItemForOrder khóa tồn store/product, kiểm version nếu gửi và kiểm số lượng đủ.

Sau giảm, ledger DEDUCTED. Nếu đã DEDUCTED cùng snapshot thì retry không trừ thêm. Event order.created cũng gặp ledger này và bỏ qua phần trừ lặp. POS gửi expectedVersions rỗng trong đường tạo, vẫn có khóa tồn và kiểm đủ.

## Hết món và bù

Nếu tồn không đủ, transaction Catalog rollback phần giữ của lần gọi. Order hủy nháp theo đường lỗi. Nếu đã giữ mà Order không hoàn tất, releaseStock hoặc consumer order.cancelled hoàn theo snapshot ledger và đánh dấu để tránh hoàn lặp. Hủy thất bại tạm thời có thể chờ job/retry.

Tồn bị trừ ở bước giữ khi đặt, không đợi khách nhận hay bếp xong. Vì vậy thêm giỏ không bảo đảm món còn khi bấm đặt. Reorder cũng cần kiểm và báo giá lại trước lần đặt tiếp theo.

Catalog/service/ValidationService.java · validateOrderItems. Catalog/service/OrderStockService.java · reserveOrderStock/apply/releaseOrderStock. InventoryService.reserveItemForOrder. Cart/service/OrderService.java · applyValidatedSnapshots/createOrder/createCounterOrder.

Đơn theo ngày và Excel

# Nguồn ngày và tập xuất

## Ngày và số gọi

OrderService.getOrders chuyển date thành [00:00, 00:00 ngày kế) ở Asia/Ho_Chi_Minh, lọc createdAt. Staff loadStoreDayOrders gọi các trang size 200 tới khi hết. Store Manager dùng dữ liệu OrderContext theo store và lọc ngày trên UI. Admin gửi query về backend, giữ phân trang.

generateDailyQueueNumber dùng key queue:store:{storeId}:{yyyyMMdd}, Redis INCR và mốc max trong DB, TTL 2 ngày khi khởi tạo/reset theo nhánh. Giá trị quay vòng 1–999. Nó phục vụ gọi món. orderCode/ID mới là định danh để đối chiếu, có thể có số khuyết sau lỗi/hủy.

## Xuất danh sách

Staff handleExportDailyExcel dùng đơn theo ngày đang nạp, thêm biên lai theo orderCode khi đọc được rồi gọi exportStaffDailyOrdersExcel. File là SpreadsheetML với đuôi .xls tạo client. Tổng đã thu lấy PAID/COLLECTED và cộng total. Tiền mặt/chuyển khoản dựa receipt hoặc fallback trường đơn.

exportAdminOrdersByDateExcel lấy tập truyền vào. Admin có thể chỉ xuất các đơn trang hiện tại hoặc nhóm ngày trong trang. Người dùng cần xem đúng filter và tập đã tải trước khi diễn giải con số như số toàn hệ thống.

## Khác báo cáo tổng hợp

Trang Đơn hàng đọc từng đơn. Reports dùng /api/reports/dashboard, /revenue, /export và projection riêng. config-repo/review-report-service.yml mặc định doanh thu theo order.completed, có policy payment.confirmed. Đơn vừa chốt có thể đợi event được xử lý trước khi số báo cáo đổi.

PAID_SIMULATED giúp đọc tiền ứng trên UI nhưng các bộ lọc total đã thu bằng PAID/COLLECTED vẫn loại nó. Chưa được lấy chữ hiển thị “đã ứng” thay cho công thức tổng tiền thật.

Cart/service/OrderService.java · getOrders/generateDailyQueueNumber. frontend/react/src/pages/Staff/StaffOrdersPage.tsx · loadStoreDayOrders/handleExportDailyExcel. utils/orderExcelExport.ts · exportStaffDailyOrdersExcel/exportAdminOrdersByDateExcel. Review/service/ReportProjectionService.java.

Sổ quỹ và két ngày

# Nguồn tiền và công thức

## Vai trò và phạm vi

UI /admin/transfers cho Store Manager/Admin. API GET /api/payments/shift-summary còn cho STAFF cùng chi nhánh. CounterPaymentService.getShiftSummary lấy chi nhánh token cho người dùng chi nhánh, từ chối yêu cầu khác scope. Admin chọn store, nếu bỏ có fallback theo code. Ngày dùng Asia/Ho_Chi_Minh.

| Chỉ số | Nguồn được cộng | Mốc thời gian |
| --- | --- | --- |
| Tiền mặt | Biên lai không VOIDED, collection ACCEPTED/LEGACY. Cộng cash của POS phiên PAID và collection độc lập ACCEPTED chưa có receipt hiệu lực. | createdAt receipt, paid_at phiên, resolved_at collection. |
| Tiền thối | cashChange trên chứng từ/khoản thu đủ điều kiện. | Theo cùng nguồn tương ứng. |
| Ngân hàng đã đối soát | bankAmountReconciled trên receipt RECONCILED, amount phiên live PAID, bằng chứng collection tối đa số khai báo. | Các mốc chứng từ hoặc trả tiền. |
| Ngân hàng chờ | Receipt PENDING_RECONCILIATION, thiếu của POS còn hạn, phần khai collection chưa có bằng chứng. | Theo điều kiện ngày của từng nguồn. |
| Chênh lệch | discrepancyAmount từ receipt đủ điều kiện. | Ngày receipt. |
| Tiền trong két | netCashInDrawer = totalCashCollected. | Tiền thối thống kê riêng. Chưa có số dư đầu ca. |

## “Ca” hiện được tính thế nào

Hàm nhận storeId/date, không có shiftId và không nối entity Shift của Identity. Đây là tổng tiền theo ngày/chi nhánh. Danh sách receipt dùng Between của repository, các query SQL tiền dùng giới hạn đầu/cuối theo điều kiện từng nguồn. Cần tự kiểm ca sát nửa đêm khi phát triển thêm.

IncomingTransfersPage đọc /transactions theo ngày, /receipts rồi lọc ngày và /shift-summary. Khi mở, đổi bộ lọc hoặc bấm tải lại thì lấy mới. Nếu backend summary lỗi, UI có fallback theo danh sách và thông báo tải lỗi, số dự phòng có thể khác backend. Xuất cash ledger dùng dữ liệu đang nạp.

Đơn tạo hôm qua trả hôm nay làm ngày đơn và ngày tiền khác nhau. Đối chiếu bằng mã và mốc giao dịch trước khi kết luận sai thống kê.

Payment/service/CounterPaymentService.java · getShiftSummary dòng 301–410. frontend/react/src/pages/Staff/IncomingTransfersPage.tsx · loadPaymentData/shiftCashStats. utils/orderExcelExport.ts · exportCashLedgerExcel.

Ngoại lệ và đường quay lại

# Lỗi hủy hoàn tiền và đặt lại

| Tình huống | Kiểm tra hoặc thay đổi | UI và giới hạn |
| --- | --- | --- |
| Đặt lặp/mất mạng | Key/hash và DB claim replay đơn, nháp đang chạy trả 409. | UI giữ key cho retry phù hợp. Web bỏ key vẫn có thể sinh đơn riêng. |
| Sai store/bước/version | Backend scope, lock, source status và version nếu có. | 403/409. Staff cảnh báo, tải lại để chọn thao tác mới. |
| QR sai/thiếu/lặp | Match tài khoản/ngân hàng/tham chiếu, receivedAmount, bank_tx_id/fingerprint. | Chờ đủ, hiện số thiếu hoặc cần rà soát. Không tự xác nhận tiền từ click. |
| Thu tiền đã có nhưng đơn đổi | Order so snapshot và decision cũ. | 201 receipt có thể còn PENDING hoặc bị từ chối sau. Xem collection trước khi thử lại. |
| Voucher/điểm/tồn lỗi | Chặn trước tạo hoặc hủy nháp, release/consumer/job phục hồi sau. | Có thể cần tải lại giỏ/giá/điểm. Phục hồi liên service không tức thì. |
| Đã sửa nhưng màn cũ | Response, polling, projection và mapping nhãn riêng. | So đúng ID/store/ngày rồi tải lại. Không suy từ chỉ một nhãn. |

## Hủy và đặt lại

Khách hủy đơn sở hữu ở PENDING_PAYMENT/CONFIRMED theo lý do. Nội bộ có reject/cancel với scope. Sau chế biến thông thường không hủy qua các đường này. Chuyển CANCELLED ghi lịch sử/outbox và đưa công việc nhả voucher theo instance/mã vào recovery. Đơn đã trả cần xử lý tiền riêng.

POST /api/orders/{id}/reorder trả CartDto để đưa món về giỏ. Nó kiểm lại dữ liệu phù hợp trước khi thêm. Tạo đơn mới vẫn phải checkout và có thể thay giá/tồn/ưu đãi. Không tạo ngay một đơn mới chỉ từ nút đặt lại.

## Hoàn tiền

RefundRequestService.createRefundRequest kiểm khách, quyền sở hữu, lịch sử hủy hoặc kết quả giao thất bại đủ điều kiện và số tiền Payment cho phép hoàn. Yêu cầu thông tin ngân hàng, phiên bản thông tin và bằng chứng theo quy trình. Duyệt, nhận xử lý, yêu cầu bổ sung và xác nhận là các bước riêng qua /api/refund-requests.

Mã yêu cầu hoàn khác ID đơn. API không được coi orderId như refundRequestId. UI cho khách tạo yêu cầu và quản lý duyệt/xử lý theo quyền. Chưa được nói hủy là tiền đã tự về tài khoản hoặc giao mô phỏng thành công là có khoản hoàn ngân hàng thực.

Cart/service/OrderService.java · cancelOrder/rejectOrder/reorder. RefundRequestService · cancellation/createRefundRequest/claimProcessing/confirmRefund. Payment/service/PaymentFlow.java · reconcile. Cart/controller/RefundRequestController.java.

Phần tâm đắc

# Điều phối theo lựa chọn và giữ nhất quán

## Khó ở đâu

Một luồng phục vụ nhiều tổ hợp. Web ngân hàng chờ tiền trước bếp, web tự lấy tiền mặt làm trước thu sau, POS có thể đổi cách trả trước khi thu. DELIVERY do Shipping kết thúc còn PICKUP do staff xác nhận. Ba UI đọc các nguồn và thời điểm khác nhau, tab Chờ tiền lại có thể chứa đơn CONFIRMED.

Khi hai người thu/đổi phương thức gần nhau, hoặc event giao tới trễ sau trả món, ghi thiếu kiểm tra có thể làm trạng thái hoặc nhãn sai. Phần khó là giữ các quyết định liên quan chặt chẽ dù nhiều tác nhân và service cùng tham gia.

## Đã làm gì cần xác nhận theo đóng góp

[Tôi xác nhận phần đóng góp này] Phân nhánh bằng orderSource, type, paymentMethod và diningOption, dùng status, paymentStatus và merchantPaymentStatus cho các ý nghĩa tương ứng.

[Tôi xác nhận phần đóng góp này] Xây hoặc hoàn thiện các quyết định lúc tạo, chốt chuyển bước, xử lý kết quả tiền và đọc lại UI. Ghi đúng hàm/PR mình trực tiếp làm, phần phối hợp với người viết Payment/Shipping/Loyalty và phần chỉ sử dụng API.

## Giải quyết ra sao và chứng minh bằng đơn nào

| Vấn đề | Điểm giải quyết | Bằng chứng nên mở |
| --- | --- | --- |
| Đơn nào được làm trước | createOrder/createCounterOrder quyết trạng thái, UI gom POS theo tiền. | Mẫu A chờ QR, B làm trước, C vào Chờ tiền nhưng status CONFIRMED. |
| Giao thiếu tiền tại quầy | updateStatus yêu cầu READY/PICKUP và PAID/COLLECTED. | Mẫu B gửi COMPLETED khi PENDING bị 409. |
| Đổi cách trả khi đang thu | claimCounterPayment và changeCounterPaymentMethod khóa cùng Order. | Mẫu C mở phiên rồi thử đổi bị chặn. |
| Nhầm ứng tiền COD với thu thật | Projection lưu merchantPaymentStatus, UI dùng displayPaymentStatus. | Đơn giao COD hiện ứng mô phỏng và paymentStatus PENDING riêng. |
| Hai màn hình thấy khác | API là nơi quyết định, response/polling và event phản ánh sau. | Đối chiếu cùng mã đơn và trạng thái nguồn trước/sau tải lại. |

Giá trị có thể trình bày là giải thích được mỗi thao tác hợp lệ, dữ liệu kiểm tra và kết quả bị từ chối bằng code thật. Tránh hứa tuyệt đối không có độ trễ hoặc mọi cạnh tranh đều đã khóa.

OrderService.createOrder/changeCounterPaymentMethod/updateStatus · OrderPaymentService.claimCounterPayment · ShippingOrderProjection.recordMerchantPayment · StaffOrdersPage.visibleOrders · order-fulfillment.displayPaymentStatus.

Hỏi đáp tra cứu

# Mười hai điểm mentor thường hỏi

1 Tại sao tính lại giá

Giá/tồn/ưu đãi có thể thay đổi và request có thể được gửi trực tiếp. Catalog kiểm giá, Order tính lại và so expectedTotal. Tra createOrder, ValidationService.validateOrderItems.

2 Giỏ Redis sống bao lâu

Guest 24 giờ, tài khoản 7 ngày. Thêm/sửa dương đặt lại TTL, đọc và xóa riêng không gia hạn. Tra CartService và resolveContext.

3 Đóng trình duyệt hoặc đăng nhập thì sao

TTL tiếp tục chạy. Guest ID trong localStorage nhận lại giỏ. Đăng nhập gọi merge, cộng món trùng vào giỏ tài khoản và xóa guest khi có dữ liệu gộp. Tra OrderProvider.loadCart/CartService.mergeCart.

4 Bấm đặt hai lần

UI khóa gửi, key/hash cùng nội dung replay. POS bắt buộc key, web API còn cho bỏ. Tra replayWebOrder/replayCounterOrder và V19/V24.

5 Tại sao POS Chờ tiền nhưng status đã xác nhận

queueCounts/visibleOrders gom theo source + tiền còn PENDING + status. Đây là nhóm UI. Backend vẫn có bảng chuyển của status riêng.

6 Tiền chưa vào có được làm không

Web ngân hàng PENDING_PAYMENT chưa vào bếp. POS backend CONFIRMED có thể làm, UI đưa vào Chờ tiền. PICKUP hoàn tất luôn cần đã ghi nhận tiền.

7 Đổi phương thức khi nào

Khi còn chờ, chưa có hoạt động và chưa claim. Claim Order và đổi phương thức dùng cùng khóa dòng. Tra changeCounterPaymentMethod/claimCounterPayment.

8 Hai nhân viên cùng cập nhật

Có khóa dòng, @Version và staff gửi version. Caller bỏ version chỉ còn kiểm dữ liệu hiện tại. Collection kiểm updatedAt riêng.

9 Tồn bị trừ khi nào

Sau nháp, reserveStock giảm kho chi nhánh trong Catalog. Ledger theo orderId chống trừ lặp. Tra OrderStockService và InventoryService.

10 Khách thấy thay đổi khi nào

Chi tiết poll 5 giây tới trạng thái cuối, QR và tracking có nhịp riêng. Thêm thời gian worker/Kafka nếu là tích hợp.

11 COD mô phỏng đã có tiền chưa

Có thể ghi PAID_SIMULATED cho tiền ứng quán nhưng paymentStatus vẫn PENDING. Chưa có chứng từ khách thực thu. Tra ShippingOrderProjection/displayPaymentStatus.

12 Két theo ca và Excel lấy đâu

Két Payment tổng theo ngày/store và nhiều mốc tiền. Excel danh sách theo tập nạp và ngày tạo đơn. Hiện chưa dùng shiftId.

Giới hạn và hướng hoàn thiện

# Phân biệt điều đã có với điều cần bổ sung

| Phần | Hiện trạng khảo sát | Hướng cải thiện |
| --- | --- | --- |
| Giỏ và cấu hình TTL | Hằng 24h/7d, có cấu hình TTL nhưng CartService chưa dùng. Read-modify-write nhiều lệnh. | Gắn policy cấu hình, thêm nguyên tử Redis cho cập nhật/gộp nếu yêu cầu. |
| Chờ tiền POS | UI gom theo tiền, backend vẫn cho CONFIRMED → PREPARING chưa thu. | Nếu sản phẩm yêu cầu trả trước, cần quy tắc backend thống nhất. Hiện phải diễn giải đúng hai tầng. |
| Claim phương thức | Claim bền trên Order bảo vệ sau khi bắt đầu thu. Lỗi bước sau vẫn có thể giữ cờ. | Thiết kế rõ phục hồi phiên lỗi/hết hạn và khả năng retry cùng phương thức. |
| Đặt trước theo giờ | PICKUP có, scheduledAt lưu được nhưng UI checkout chưa có chọn giờ. | Bổ sung slot nhận nếu nằm trong yêu cầu sản phẩm. |
| In và giao thật | Bill/KOT in mô phỏng, giao mô phỏng. Phase 2 cho giao thật. | Máy in và chứng từ thực, hợp đồng giao/thu tiền ngoài mô phỏng. |
| Két và tổng | Theo ngày, chưa mở/đóng ca. Excel Admin theo tập trang hiện tại. | Phiên ca, đối chiếu nhiều mốc và xuất full dataset theo yêu cầu. |
| Tổng tiền mặt lẻ | Có làm tròn và kiểm tổng giữa service. Rủi ro suy luận ở web tự lấy. | Kiểm lại số lẻ và quy tắc tổng được chấp nhận. |
| Version và độ trễ | Version tùy chọn, một số caller manager chưa gửi. Polling/event có trễ. | Gửi version nhất quán và hiển thị trạng thái chờ tích hợp. |

## Những khác biệt với tài liệu cũ

docs/M3_CART_ORDER_FIX_FLOW.md còn ghi giỏ tài khoản 30 ngày, code CartService hiện 7 ngày. Các mô tả POS mock trong tài liệu sprint là giai đoạn trước, đường hiện tại gọi API thật trong code. Tài liệu cũ ghi bỏ endpoint refund trên Order, nhưng OrderController hiện vẫn còn mapping tương thích.

Bản trình bày cũ mô tả POS luôn vào Mới xác nhận sau tạo chưa phản ánh nhóm Chờ tiền hiện tại. MerchantPaymentStatus mới được lưu trên Order và ánh xạ ở UI thay cho dựa tracking đơn thuần.

## Những điểm người trình bày tự xác nhận

Phạm vi đóng góp theo hàm/PR, tài khoản và store demo, trạng thái cấu hình QR/Mapbox và các ca chức năng đã tự kiểm. Tài liệu mô tả bằng chứng source, không phải kết quả kiểm thử môi trường chạy.

docs/M3_CART_ORDER_FIX_FLOW.md · CartService · StaffOrdersPage · OrderController · Cart migration V26. Các đề xuất là hướng hoàn thiện, chưa được triển khai trong phiên soạn tài liệu.

Code mở khi bảo vệ

# Năm quyết định có thể chỉ ngay

## 1 Hạn giỏ

backend/cart-order-service/src/main/java/com/clucky/cartorderservice/cart/CartService.java · CartService · dòng 21–22

public static final Duration GUEST_CART_TTL = Duration.ofHours(24);

public static final Duration CUSTOMER_CART_TTL = Duration.ofDays(7);

Phần giỏ nhận TTL theo guest hoặc CUSTOMER từ controller. Kiểm thêm expire ở hàm ghi.

## 2 Trạng thái đầu web

backend/cart-order-service/src/main/java/com/clucky/cartorderservice/service/OrderService.java · OrderService.createOrder · dòng 492–494

String initialStatus = "COD".equalsIgnoreCase(paymentMethod)

? OrderStatus.CONFIRMED.name()

: OrderStatus.PENDING_PAYMENT.name();

Đơn web tiền khi nhận được xác nhận, ngân hàng chờ tiền. Bản nháp được hoàn tất sau giữ tồn.

## 3 Chốt tiền khi bàn giao

backend/cart-order-service/src/main/java/com/clucky/cartorderservice/service/OrderService.java · OrderService.updateStatus · dòng 1227

if ("COMPLETED".equals(target)) {

if (order.getPaymentStatus() == null || "PENDING".equalsIgnoreCase(order.getPaymentStatus())

|| !Set.of("PAID", "COLLECTED").contains(order.getPaymentStatus())) {

throw new ApiException(ErrorCode.RESOURCE_CONFLICT, "Đơn hàng chưa được thanh toán đầy đủ trước khi bàn giao.");

}

}

Điều kiện chưa thanh toán trả 409. Mở tiếp switch để thấy giới hạn READY/PICKUP.

## 4 Khóa cách trả POS

backend/cart-order-service/src/main/java/com/clucky/cartorderservice/service/OrderService.java · OrderService.changeCounterPaymentMethod · dòng 1187

if (order.isCounterPaymentStarted() || existingPaymentActivity)

Nếu hoạt động thu đã bắt đầu thì đổi cách trả bị chặn. Đối chiếu claimCounterPayment ở OrderPaymentService.

## 5 UI tiền ứng mô phỏng

frontend/react/src/lib/order-fulfillment.ts · displayPaymentStatus · dòng 5–9

const merchantStatus = order.merchantPaymentStatus ?? trackingStatus;

if (order.paymentStatus === 'PENDING' && (merchantStatus === 'PAID_SIMULATED' || merchantStatus === 'ADVANCE_RETURNED')) {

return merchantStatus;

}

return order.paymentStatus ?? 'UNKNOWN';

Khi paymentStatus=PENDING và merchant là PAID_SIMULATED/ADVANCE_RETURNED, UI hiển thị theo trường merchant.

Tra cứu nguồn

# Các file chính và phạm vi đã đọc

Nguồn sản phẩm và kiến trúc gồm AGENTS.md, docs/M3_CART_ORDER_FIX_FLOW.md, các báo cáo sprint trong docs/ và config-repo. Nội dung khác code được nêu tại phần giới hạn. 

| Điểm tra cứu | Đường dẫn tương đối trong repo |
| --- | --- |
| Giỏ | backend/cart-order-service/src/main/java/com/clucky/cartorderservice/cart/CartController.java, CartService.java, CartItemDto.java, MergeCartRequest.java. |
| Tạo và thay đổi đơn | backend/cart-order-service/src/main/java/com/clucky/cartorderservice/service/OrderService.java. Controller và DTO cùng service. |
| Tiền và giao trên Order | Cùng thư mục service có OrderPaymentService.java, ShippingOrderProjection.java, OrderPaymentWorker.java, OrderEventPublisher.java. |
| Thanh toán | backend/payment-service/src/main/java/com/clucky/paymentservice/service/PaymentFlow.java, PaymentOperations.java, CounterPaymentService.java, SepayReceiptService.java. |
| Kho và combo | backend/catalog-store-service/src/main/java/com/clucky/catalogstoreservice/service/ValidationService.java, OrderStockService.java, InventoryService.java. |
| Ưu đãi và đánh giá | backend/promotion-loyalty-service/.../service/LoyaltyEvents.java, VoucherReservationService.java, CounterLoyaltyService.java. backend/review-report-service/.../service/ReviewService.java. |
| Giao mô phỏng | backend/shipping-service/src/main/java/com/clucky/shippingservice/service/ShipmentServiceTemplate.java, MockDeliveryEngine.java. |
| UI khách | frontend/nextjs/app/checkout/page.tsx, app/cart/page.tsx, app/orders/[id]/page.tsx, components/order/order-provider.tsx, lib/api.ts, lib/order-demo.ts. |
| UI nội bộ | frontend/react/src/pages/Staff/StaffOrdersPage.tsx, Staff/components/CounterOrderModal.tsx, CounterPaymentModal.tsx, OrderCard.tsx, pages/Orders, lib/order-fulfillment.ts. |
| Số liệu và cấu hình | frontend/react/src/utils/orderExcelExport.ts, pages/Staff/IncomingTransfersPage.tsx, config-repo/*.yml, deploy/backend/compose.yml, compose.local.yml. |

## Migration nên nhớ

Cart V7 payment_deadline_and_decisions, V8 collection_decisions, V16 add_order_version, V19 counter_order_idempotency, V24 web_order_idempotency, V25 counter_payment_claim và V26 mock_cod_merchant_payment. Nằm trong src/main/resources/db/migration. Catalog/Payment/Loyalty có ledger và migration riêng trong service tương ứng.

Việc soạn không chạy test, lint, typecheck độc lập hoặc kiểm thử trình duyệt. Kết quả runtime, cấu hình môi trường đang dùng và phạm vi đóng góp theo tác giả cần người trình bày tự xác nhận.

	Clucky  ·  Tài liệu bảo vệ luồng đặt hàng	1
