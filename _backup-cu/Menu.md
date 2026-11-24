# Clucky — Phân tích lỗi logic đặt món Combo (Customer Web)

> **Ngày lập:** 17/11/2026
> **Branch hiện tại:** `fix/logic-order`
> **Phạm vi:** `frontend/nextjs` + `backend/catalog-store-service` (có liên quan gián tiếp `backend/cart-order-service`)
> **Trạng thái:** **Chưa sửa file nào**, chưa chạy build/test/lint/typecheck.
> **Tài liệu này** tổng hợp: code hiện tại đang dùng gì · còn gì · thiếu gì · kế hoạch sửa chi tiết · checklist kiểm thử.

---

## Mục lục

1. [Tóm tắt điều hành](#1-tóm-tắt-điều-hành)
2. [Luồng code thực tế đang chạy](#2-luồng-code-thực-tế-đang-chạy)
3. [Chi tiết 4 lỗi](#3-chi-tiết-4-lỗi)
4. [Những điểm kế hoạch ban đầu còn thiếu / sai](#4-những-điểm-kế-hoạch-ban-đầu-còn-thiếu--sai)
5. [Kế hoạch sửa chi tiết theo file](#5-kế-hoạch-sửa-chi-tiết-theo-file)
6. [Thứ tự triển khai & rebuild](#6-thứ-tự-triển-khai--rebuild)
7. [Checklist kiểm thử dành cho người dùng](#7-checklist-kiểm-thử-dành-cho-người-dùng)
8. [Rủi ro & lưu ý vận hành](#8-rủi-ro--lưu-ý-vận-hành)
9. [Trạng thái hiện tại](#9-trạng-thái-hiện-tại)

---

## 1. Tóm tắt điều hành

| # | Lỗi | Mức độ | Đánh giá so với kế hoạch ban đầu |
|---|-----|--------|----------------------------------|
| 1 | Route chi tiết `app/menu/[slug]/page.tsx` nhận diện sai combo → mở nhầm món đơn hoặc 404 | **Cao** | Đúng, nhưng **seed hiện tại chưa lộ lỗi** vì tên combo đã bắt đầu bằng `Combo` |
| 2 | `ValidationService` vẫn cộng `lineTotal`/`subtotal` cho combo không khả dụng | **Trung bình (tiềm ẩn)** | Đúng, nhưng **UI chưa bị sai tiền** do `OrderService.previewOrder` return sớm subtotal = 0 |
| 3 | Thiếu `categoryId` trong `comboAsProduct` + lọc menu | **Thấp** | **Chẩn đoán chưa khớp code thực tế** — cần chỉnh lại nguyên nhân |
| 4 | Slug combo không được chuẩn hóa prefix `combo-` | **Cao** | Đúng; slug tính runtime nên **không cần migration DB** |

**Kết luận nhanh:**

- Kế hoạch gốc đúng hướng ở **lỗi 1, 2, 4**.
- **Lỗi 3 bị chẩn đoán sai cơ chế** (React batching + `Promise.all` khiến combo không "biến mất" như mô tả).
- Kế hoạch **bỏ sót bản duplicate API client** `frontend/nextjs/lib/api.ts` — nếu chỉ sửa `lib/catalog-and-stores.api.ts` thì trang `/menu` vẫn tạo link sai.
- Kế hoạch dùng **sai tên service khi rebuild** (`nextjs` không tồn tại; tên đúng là `customer`).

---

## 2. Luồng code thực tế đang chạy

| Bước | File / hàm |
|------|------------|
| Menu load sản phẩm | `frontend/nextjs/app/menu/page.tsx:12` import `catalogApi` từ **`@/lib/api`** |
| Lấy products + combos | `frontend/nextjs/lib/api.ts:571` `getProducts()` → gọi `/api/catalog/products`, sau đó `/api/catalog/combos` **nếu không truyền** `categoryId/page/size` |
| Map combo → ProductItem | `frontend/nextjs/lib/api.ts:542` `comboAsProduct()` |
| Card tạo link | `frontend/nextjs/components/product-card.tsx:15` → `` `/menu/${product.slug}` `` |
| Trang chi tiết | `frontend/nextjs/app/menu/[slug]/page.tsx:15-45` import `catalogApi` từ **`@/lib/catalog-and-stores.api`** |
| Thêm giỏ | `frontend/nextjs/components/product-detail-client.tsx:114` → `useOrder().addItem()` → `lib/cart.ts:25 toCartPayload()` → `POST /api/cart/items` |
| Giỏ hàng local | `frontend/nextjs/components/order/order-provider.tsx:530` `cartTotals(workspace.items)` |
| Preview / tạo đơn | `frontend/nextjs/app/checkout/page.tsx:159,314,391` → `orderApi.preview/create` → `OrderService.previewOrder/createOrder` |
| Validate món/combo | `CartOrderService.CatalogClient.validateItems()` → `POST /api/catalog/internal/validate-items` → `ValidationService.validateOrderItems()` |
| Trừ kho | `OrderStockService.apply()` tách combo thành các product thành phần |

### Điểm cốt lõi: có **2 bản** `catalogApi` / `comboAsProduct` gần như trùng nhau

| File | Dùng bởi |
|------|----------|
| `frontend/nextjs/lib/api.ts` | `/menu`, `/checkout`, `/vouchers`, order provider, reviews… |
| `frontend/nextjs/lib/catalog-and-stores.api.ts` | `/` (homepage), `/menu/[slug]` (detail) |

> ⚠️ Nếu chỉ sửa một file → chỉ fix được một nửa luồng. **Bắt buộc sửa cả hai** (hoặc gộp về một nguồn duy nhất).

---

## 3. Chi tiết 4 lỗi

### 3.1. Lỗi 1 — Nhận diện sai route chi tiết món/combo

**File:** `frontend/nextjs/app/menu/[slug]/page.tsx` (dòng 15–45)

#### Code hiện tại

```ts
const isCombo = slug.startsWith('combo-');
const id = Number(slug.match(/-(\d+)$/)?.[1] ?? slug);
const storeIdValue = Array.isArray(query.storeId) ? query.storeId[0] : query.storeId;
const parsedStoreId = storeIdValue === undefined ? undefined : Number(storeIdValue);
const storeId = Number.isSafeInteger(parsedStoreId) && (parsedStoreId ?? 0) > 0 ? parsedStoreId : undefined;

let product: Product | null = null;
let productIsCombo = isCombo;
try {
  if (Number.isSafeInteger(id) && id > 0) {
    product = isCombo
      ? await catalogApi.getComboProductById(id, storeId)
      : await catalogApi.getProductById(id, storeId);
  }
} catch {
  // ...
  const allProducts = await catalogApi.getProducts({ storeId, activeOnly: true });
  // tìm theo slug ... (nhánh này gần như không bao giờ chạy)
}
```

#### Nguyên nhân

Backend sinh slug:

```java
// ComboService.java:218
r.setSlug(ProductDto.toSlug(c.getName()) + "-" + c.getId());
```

Ví dụ:

| Combo | Slug backend sinh ra | `startsWith('combo-')` |
|-------|----------------------|------------------------|
| `Combo Clucky Classic` (id 1) | `combo-clucky-classic-1` | ✅ true |
| `Combo Family Feast` (id 2) | `combo-family-feast-2` | ✅ true |
| `Family Feast` (id 2, đã đổi tên) | `family-feast-2` | ❌ false |
| `Gà Rán Tiết Kiệm` (id 1) | `ga-ran-tiet-kiem-1` | ❌ false |

→ Khi `isCombo = false`, hệ thống gọi **`/api/catalog/products/{id}`**.

#### Hệ quả

| Tình huống | Kết quả |
|------------|---------|
| Product cùng ID tồn tại | Hiển thị **nhầm món đơn** (VD: click "Family Feast" 329.000đ → ra "Burger gà giòn" 55.000đ), thêm giỏ thành PRODUCT |
| Product cùng ID **không** tồn tại | API 404 → `catch` → `product = null` → hiện "Không tìm thấy món ăn" |
| Slug không kết thúc bằng `-<id>` | Regex không match → `Number(slug)` = `NaN` → nhảy vào `catch` |

#### Điểm quan trọng về dữ liệu seed

`V5__seed_sample_catalog_data.sql:113-116` seed tên:

```sql
(1, 'Combo Clucky Classic', ..., 129000, ..., TRUE),
(2, 'Combo Family Feast',  ..., 329000, ..., TRUE)
```

→ **Cả hai đều có prefix `combo-`**, nên với DB seed mặc định **lỗi 1 chưa biểu hiện**. Lỗi chỉ xảy ra khi:

1. Admin đổi tên combo bỏ chữ `Combo` ở đầu;
2. Tạo combo mới không có chữ `Combo` ở đầu;
3. Dữ liệu DB thực tế khác seed.

**Kiểm tra dữ liệu đang chạy:**

```sql
SELECT id, name, is_active, start_at, end_at
FROM catalog_service.combos
ORDER BY id;
```

#### Nhánh fallback hiện tại có an toàn không?

**Không.**

```ts
const allProducts = await catalogApi.getProducts({ storeId, activeOnly: true });
```

- `getProducts` **không truyền** `categoryId/page/size` → có gọi combos → danh sách trả về **có combo**.
- Nhưng nhánh này **chỉ chạy trong `catch`**, tức là khi API ném lỗi. Trường hợp nguy hiểm hơn — **API trả về product cùng ID thành công nhưng sai món** — lại **không** đi vào `catch`, nên fallback không bao giờ được kích hoạt.

---

### 3.2. Lỗi 2 — Tính sai `lineTotal` / `subtotal` khi combo không khả dụng

**File:** `backend/catalog-store-service/src/main/java/com/clucky/catalogstoreservice/service/ValidationService.java` (dòng 319–325)

#### Code hiện tại

```java
// sau vòng lặp kiểm tra tồn kho từng món thành phần
itemResult.setName(combo.getName());
itemResult.setUnitPrice(combo.getComboPrice());
itemResult.setExtraPrice(BigDecimal.ZERO);

BigDecimal lineTotal = combo.getComboPrice().multiply(BigDecimal.valueOf(qty));
itemResult.setLineTotal(lineTotal);
totalSubtotal = totalSubtotal.add(lineTotal);   // ❌ vẫn cộng dù combo invalid
```

So sánh nhánh product (dòng 234–243):

```java
if (optionsValid && toppingsValid && itemResult.isAvailable()) {
    BigDecimal lineTotal = unitPrice.add(totalExtra).multiply(BigDecimal.valueOf(qty));
    itemResult.setLineTotal(lineTotal);
    totalSubtotal = totalSubtotal.add(lineTotal);
} else {
    itemResult.setExtraPrice(BigDecimal.ZERO);
    itemResult.setLineTotal(BigDecimal.ZERO);
}
```

#### Đánh giá mức ảnh hưởng (khác với kế hoạch ban đầu)

Kế hoạch ban đầu nói "Order Preview trả về tổng tiền không chính xác". **Thực tế chưa đúng như vậy:**

`OrderService.previewOrder:249-256`:

```java
CatalogClient.CatalogValidationResponse catalogValidation = catalogClient.validateItems(storeId, items);
if (!catalogValidation.valid() || !catalogValidation.storeOpen()) {
    return new OrderPreviewResponse(
        BigDecimal.ZERO, BigDecimal.ZERO, ... // subtotal = 0
    );
}
BigDecimal subtotal = catalogValidation.subtotal(); // chỉ đọc khi valid
```

Tương tự `createOrder`, `previewCounterOrder`, `createCounterOrder` đều **chặn** khi `valid == false`.

→ **UI hiện tại chưa hiển thị sai tiền**, vì preview trả `valid = false` + tổng 0.

**Đây là lỗi hợp đồng nội bộ (internal contract bug):**

- Endpoint `/api/catalog/internal/validate-items` trả về `subtotal` **có cộng combo lỗi**.
- Bất kỳ consumer mới nào đọc `subtotal` mà không check `valid` sẽ tính sai tiền.
- Response không nhất quán: combo `null` / inactive / expired / rỗng items → `name`, `unitPrice`, `lineTotal` bị **null** thay vì `0`.

#### Các trường hợp biên cần xử lý

| Trường hợp | Hiện tại | Nên là |
|-----------|----------|--------|
| Combo không tồn tại | `name = null`, `unitPrice = null`, `lineTotal = null` | `lineTotal = 0`, reason rõ ràng |
| Combo `is_active = false` | `lineTotal = giá combo` (sai) | `lineTotal = 0` |
| Combo hết hạn (`end_at < now`) | `lineTotal = giá combo` (sai) | `lineTotal = 0` |
| Combo chưa bắt đầu (`start_at > now`) | `lineTotal = giá combo` (sai) | `lineTotal = 0` |
| Combo không có item | `lineTotal = giá combo` (sai) | `lineTotal = 0` |
| Thành phần hết tồn kho | `lineTotal = giá combo` (sai) | `lineTotal = 0` |

---

### 3.3. Lỗi 3 — Thiếu `categoryId` và cơ chế lọc menu

**Files:**
- `frontend/nextjs/lib/catalog-and-stores.api.ts:82` (`comboAsProduct`)
- `frontend/nextjs/lib/api.ts:542` (`comboAsProduct`)
- `frontend/nextjs/app/menu/page.tsx:65, 97–102`

#### Code hiện tại

```ts
export function comboAsProduct(combo: ComboItem): ProductItem {
  return {
    id: combo.id,
    slug: combo.slug || `combo-${combo.id}`,
    name: combo.name,
    categoryName: "Combo",
    category: "Combo",
    // ❌ categoryId không được gán → undefined
    ...
  };
}
```

Filter menu:

```ts
const selectedCategoryName = categories.find(c => c.id === selectedCategory)?.name;
const matchCat =
  selectedCategory === 0 ||
  p.categoryId === selectedCategory ||
  (selectedCategoryName?.toLocaleLowerCase() === 'combo' &&
   p.categoryName?.toLocaleLowerCase() === 'combo');
```

#### Chẩn đoán ban đầu **chưa đúng** — giải thích

Kế hoạch viết:

> "Tại thời điểm render đầu tiên, categories mới chỉ có `[{ id: 0, name: 'All' }]`... toàn bộ Combo bị ẩn cho đến khi API danh mục tải xong."

**Thực tế `menu/page.tsx:64–76`:**

```ts
const [products, store, categories] = await Promise.all([
  catalogApi.getProducts({ storeId, activeOnly: true }),  // không truyền categoryId
  catalogApi.getStoreById(storeId),
  catalogApi.getCategories(true),
]);
// sau đó setProducts / setStore / setCategories trong cùng một async continuation
```

- Sản phẩm và danh mục được **fetch song song**, set state **cùng một microtask**.
- React 19 **tự động batch** các `setState` này → không có frame trung gian nào hiển thị products mà chưa có categories.
- `loading` vẫn `true` cho đến `finally`.

→ **Combo không "biến mất" trong lúc tải danh mục.**

Ngoài ra filter hiện tại đã có **fallback theo tên**, nên với category seed `(3, 'Combo')` combo vẫn hiển thị.

#### Trường hợp **thực sự** có thể làm combo biến mất

1. Admin **đổi tên category** `Combo` thành tên khác → fallback theo tên fail, `categoryId` vẫn `undefined` → combo mất khỏi tab.
2. Category Combo bị **inactive** → `getCategories(true)` không trả về → URL `/menu?cat=3` không có tab, filter theo tên cũng fail.
3. DB có category Combo với **ID khác 3** (dữ liệu không theo seed).

#### Về đề xuất sửa `getProducts` khi `categoryId === 3`

Kế hoạch ban đầu đề xuất:

> "Nếu `params?.categoryId === 3`, không return sớm products rỗng mà vẫn gọi combos."

**Không nên làm vậy.**

Lý do:

- Hiện **không có caller nào** truyền `categoryId` vào `getProducts`.
- Hàm tên `getProducts` — trả về combo cho `categoryId = 3` là **sai contract**.
- Nếu sau này thêm phân trang (`page/size`), combo lại bị loại → hành vi không nhất quán.

**Cách đúng:** thêm `categoryId: 3` vào `comboAsProduct` (hardening), giữ nguyên early-return. Nếu cần API "lấy menu item theo category bao gồm combo" thì tách hàm mới (`getMenuItems`).

---

### 3.4. Lỗi 4 — Slug combo không có prefix `combo-`

**File:** `backend/catalog-store-service/src/main/java/com/clucky/catalogstoreservice/service/ComboService.java:218`

```java
r.setSlug(ProductDto.toSlug(c.getName()) + "-" + c.getId());
```

#### Đặc điểm quan trọng

- Slug **không được lưu DB** — `combos` table (`V2__domain_schema.sql:27-31`) **không có cột `slug`**.
- Slug được **tính runtime** trong `toResponse()` → áp dụng cho mọi endpoint list/detail/create/update.
- → **Không cần Flyway migration, không cần backfill.**

#### Va chạm ID giữa 2 bảng

| Bảng | ID bắt đầu | Ví dụ |
|------|-----------|-------|
| `products` | 1, 2, 3… | Product 1 = Gà rán, Product 2 = Cánh gà |
| `combos` | 1, 2, 3… | Combo 1 = Classic, Combo 2 = Family |

→ Slug `family-feast-2` có thể là product 2 hoặc combo 2. **Prefix `combo-` là cách phân biệt tối thiểu và rẻ nhất.**

#### Ảnh hưởng cache

- `ComboController.java:26-33`: list combos có `Cache-Control: public, max-age=60` → browser có thể giữ slug cũ tối đa 60 giây. Sau deploy cần hard refresh.
- `CatalogCacheService` chỉ cache **products** (`menu:store:*`, TTL 5 phút), **không cache combos** → không cần evict Redis.
- Không có test backend nào assert combo slug → không phải sửa test cũ.

---

## 4. Những điểm kế hoạch ban đầu còn thiếu / sai

| # | Vấn đề | Chi tiết |
|---|--------|----------|
| 1 | **Thiếu `lib/api.ts`** | Trang `/menu` dùng `@/lib/api`, không phải `catalog-and-stores.api.ts`. Bỏ sót file này → link combo vẫn sai. |
| 2 | **Sai số dòng giữa 2 file** | Early-return `categoryId` ở `catalog-and-stores.api.ts:138`, ở `lib/api.ts:584`. `comboAsProduct` ở `:82` và `:542`. |
| 3 | **Fallback `product.slug !== slug` không an toàn** | Product đổi tên → URL cũ `old-name-2`, `product.slug` mới khác → logic sẽ thử combo id 2 → hiển thị **sai combo**. Phải so khớp **legacy combo slug** cụ thể. |
| 4 | **Không xử lý link combo cũ** | Sau khi đổi sang `combo-family-feast-2`, bookmark cũ `family-feast-2` sẽ 404 hoặc ra product. |
| 5 | **`lib/api.ts` thiếu `.catch()` cho request combos** | `catalog-and-stores.api.ts:144` có catch; `lib/api.ts:589` **không** → nếu `/api/catalog/combos` lỗi, toàn bộ `getProducts` reject → menu trắng dù products vẫn OK. |
| 6 | **Sai tên service khi rebuild** | `nextjs` không tồn tại → script báo `Unknown service: nextjs`. Tên đúng là `customer` (`deploy/frontend/compose.yml:8`). |
| 7 | **Checkout fallback `prodId = 1`** | `checkout/page.tsx:163-173`: nếu không map được ID → gán product 1. Khách có thể **âm thầm đặt sai món**. |
| 8 | **Lỗi 2 chưa xử lý combo inactive/expired** | Nhánh `combo == null \|\| !active` không set `name/unitPrice/lineTotal` → response trả null. |
| 9 | **Trang chủ không bao giờ có combo** | `app/page.tsx:20` gọi `getProducts({ activeOnly: true, page: 0, size: 9 })` → có `page/size` nên **không gọi combos**. Đây là gap riêng, **không sửa** bằng cách bỏ early-return. |
| 10 | **Giá trong giỏ là giá client gửi lên** | `CartService` lưu `unitPrice` từ payload; `cartTotals` tính local. Server chỉ tính lại ở preview/create. Không phải bug combo nhưng ảnh hưởng cách test. |
| 11 | **Test UI hiện có dùng `/menu/combo-901`** | `tests/ui/cart-checkout.spec.ts:64`, fixture `tests/fixtures/server.mjs:26-33`. Route này vẫn hoạt động sau fix vì đã có prefix `combo-`. |

---

## 5. Kế hoạch sửa chi tiết theo file

### 5.1. Backend — `ComboService.java`

Thêm helper chuẩn hóa slug, dùng tại dòng 218.

```java
private static final String COMBO_SLUG_PREFIX = "combo-";

private String comboSlug(Combo combo) {
    String base = ProductDto.toSlug(combo.getName());

    if (base.isBlank()) {
        return COMBO_SLUG_PREFIX + combo.getId();
    }
    if (base.equals("combo") || base.startsWith(COMBO_SLUG_PREFIX)) {
        return base + "-" + combo.getId();
    }
    return COMBO_SLUG_PREFIX + base + "-" + combo.getId();
}
```

```java
// dòng 218
r.setSlug(comboSlug(c));
```

Kết quả:

| Tên combo | Slug mới |
|-----------|----------|
| `Combo Clucky Classic` | `combo-clucky-classic-1` (không đổi) |
| `Combo Family Feast` | `combo-family-feast-2` (không đổi) |
| `Family Feast` | `combo-family-feast-2` |
| `Gà Rán Tiết Kiệm` | `combo-ga-ran-tiet-kiem-1` |
| `Combo` (chỉ 1 từ) | `combo-7` — **không** bị `combo-combo-7` |

---

### 5.2. Frontend — tạo helper dùng chung

**File mới:** `frontend/nextjs/lib/combo.ts`

```ts
export const COMBO_SLUG_PREFIX = "combo-";

// catalog_service.categories seed: (3, 'Combo') — V5__seed_sample_catalog_data.sql
export const COMBO_CATEGORY_ID = 3;

export function normalizeComboSlug(
  slug: string | null | undefined,
  id: number,
): string {
  const value = slug?.trim();
  if (!value) return `${COMBO_SLUG_PREFIX}${id}`;
  return value.startsWith(COMBO_SLUG_PREFIX)
    ? value
    : `${COMBO_SLUG_PREFIX}${value}`;
}

export function legacyComboSlug(slug: string | null | undefined): string {
  const value = slug?.trim() ?? "";
  return value.startsWith(COMBO_SLUG_PREFIX)
    ? value.slice(COMBO_SLUG_PREFIX.length)
    : value;
}
```

Mục đích:

- `normalizeComboSlug`: frontend luôn tạo link combo có prefix, **kể cả khi backend chưa deploy**.
- `legacyComboSlug`: nhận diện link cũ dạng `family-feast-2`.

---

### 5.3. Sửa **cả hai** API client

Áp dụng đồng thời cho:
- `frontend/nextjs/lib/api.ts`
- `frontend/nextjs/lib/catalog-and-stores.api.ts`

```ts
import { COMBO_CATEGORY_ID, normalizeComboSlug } from "./combo";

export function comboAsProduct(combo: ComboItem): ProductItem {
  return {
    id: combo.id,
    slug: normalizeComboSlug(combo.slug, combo.id),
    name: combo.name,
    categoryId: COMBO_CATEGORY_ID,   // ✅ thêm mới
    categoryName: "Combo",
    category: "Combo",
    // ... giữ nguyên các field còn lại
  };
}
```

Và thêm catch cho request combos trong `lib/api.ts` (quanh dòng 589):

```ts
const combos = await request<ComboItem[]>(
  `/api/catalog/combos?${comboQuery.toString()}`
).catch(() => []);
```

> ❌ **Không sửa** `if (params?.categoryId !== undefined) return products;` ở cả hai file.

---

### 5.4. Sửa `app/menu/[slug]/page.tsx`

Thay toàn bộ block `isCombo/id/try-catch` bằng logic 2 chiều có **so khớp slug chính xác**.

```tsx
import { COMBO_SLUG_PREFIX, legacyComboSlug } from '@/lib/combo';

// ...

const idMatch = slug.match(/-(\d+)$/);
const id = Number(idMatch?.[1] ?? (/^\d+$/.test(slug) ? slug : NaN));
const hasId = Number.isSafeInteger(id) && id > 0;
const comboHint = slug.startsWith(COMBO_SLUG_PREFIX);

const loadProduct = async (productId: number): Promise<Product | null> => {
  try {
    return await catalogApi.getProductById(productId, storeId);
  } catch {
    return null;
  }
};

const loadCombo = async (comboId: number): Promise<Product | null> => {
  try {
    return await catalogApi.getComboProductById(comboId, storeId);
  } catch {
    return null;
  }
};

let product: Product | null = null;
let productIsCombo = false;

if (hasId) {
  if (comboHint) {
    // URL canonical của combo
    const combo = await loadCombo(id);

    if (combo?.slug === slug) {
      product = combo;
      productIsCombo = true;
    } else {
      const prod = await loadProduct(id);
      if (prod?.slug === slug) {
        product = prod;
      } else if (combo) {
        // combo đổi tên nhưng URL cũ vẫn dùng đúng ID
        product = combo;
        productIsCombo = true;
      } else if (prod) {
        product = prod;
      }
    }
  } else {
    // URL không có prefix: ưu tiên product khớp slug chính xác
    const prod = await loadProduct(id);

    if (prod && (prod.slug === slug || String(prod.id) === slug)) {
      product = prod;
    } else {
      const combo = await loadCombo(id);

      if (
        combo &&
        (combo.slug === slug || legacyComboSlug(combo.slug) === slug)
      ) {
        product = combo;
        productIsCombo = true;
      } else {
        // URL cũ của product đã đổi tên
        product = prod;
      }
    }
  }
}

// Fallback cuối: slug không có ID hoặc dữ liệu cũ
if (!product) {
  try {
    const allProducts = await catalogApi.getProducts({
      storeId,
      activeOnly: true,
    });
    const matched = allProducts.find(
      (p) => p.slug === slug || String(p.id) === slug,
    );

    if (matched) {
      const matchedIsCombo =
        (matched.categoryName ?? matched.category ?? '').toLowerCase() ===
          'combo' ||
        Boolean(matched.comboItems?.length) ||
        matched.slug.startsWith(COMBO_SLUG_PREFIX);

      product = matchedIsCombo
        ? await loadCombo(matched.id)
        : await loadProduct(matched.id);
      productIsCombo = matchedIsCombo;
    }
  } catch {
    // để rơi xuống trạng thái "không tìm thấy"
  }
}
```

#### Bảng hành vi sau khi sửa

| URL | Kết quả |
|-----|---------|
| `/menu/combo-family-feast-2` | ✅ combo 2 |
| `/menu/family-feast-2` (link cũ) | ✅ combo 2 nhờ `legacyComboSlug` |
| `/menu/canh-ga-cay-mat-ong-2` | ✅ product 2 (slug khớp) |
| `/menu/old-name-2` (product đổi tên) | ✅ product 2, **không** nhảy sang combo |
| `/menu/2` | ✅ product 2 |
| `/menu/slug-khong-co-id` | ✅ quét `getProducts()` và so khớp chính xác |

---

### 5.5. Sửa `ValidationService.java`

#### Bản tối thiểu (đúng như kế hoạch gốc)

```java
itemResult.setName(combo.getName());
itemResult.setUnitPrice(combo.getComboPrice());
itemResult.setExtraPrice(BigDecimal.ZERO);

if (itemResult.isAvailable()) {
    BigDecimal lineTotal = combo.getComboPrice().multiply(BigDecimal.valueOf(qty));
    itemResult.setLineTotal(lineTotal);
    totalSubtotal = totalSubtotal.add(lineTotal);
} else {
    itemResult.setLineTotal(BigDecimal.ZERO);
}
```

#### Bản khuyến nghị (đồng nhất response)

```java
} else if (itemReq.getComboId() != null) {
    itemResult.setComboId(itemReq.getComboId());
    Combo combo = comboCache.computeIfAbsent(
        itemReq.getComboId(),
        id -> comboRepository.findByIdWithItems(id).orElse(null)
    );
    itemResult.setExtraPrice(BigDecimal.ZERO);

    if (combo == null) {
        itemResult.setAvailable(false);
        itemResult.setReason("Combo không tồn tại hoặc đã ngừng áp dụng.");
        itemResult.setLineTotal(BigDecimal.ZERO);
        allValid = false;
    } else {
        itemResult.setName(combo.getName());
        itemResult.setUnitPrice(combo.getComboPrice());

        if (!Boolean.TRUE.equals(combo.getIsActive())) {
            // ... giữ nguyên reason
        } else if (combo.getStartAt() != null && now.isBefore(combo.getStartAt())) {
            // ...
        } else if (combo.getEndAt() != null && now.isAfter(combo.getEndAt())) {
            // ...
        } else if (combo.getItems() == null || combo.getItems().isEmpty()) {
            // ...
        } else {
            // ... vòng lặp kiểm tra thành phần giữ nguyên
        }

        if (itemResult.isAvailable()) {
            BigDecimal lineTotal = combo.getComboPrice()
                .multiply(BigDecimal.valueOf(qty));
            itemResult.setLineTotal(lineTotal);
            totalSubtotal = totalSubtotal.add(lineTotal);
        } else {
            itemResult.setLineTotal(BigDecimal.ZERO);
        }
    }
}
```

---

### 5.6. Optional — `checkout/page.tsx`

Bỏ fallback nguy hiểm `prodId = 1`:

```ts
if (prodId == null && comboId == null) {
  const numId = Number(item.id);
  if (Number.isSafeInteger(numId) && numId > 0) {
    if (item.itemType === "COMBO") comboId = numId;
    else prodId = numId;
  }
  // ❌ KHÔNG gán prodId = 1 nữa
}
```

Nếu không map được → `toCartPayload` sẽ throw `VALIDATION_ERROR` → checkout báo lỗi thay vì âm thầm đặt product 1.

---

## 6. Thứ tự triển khai & rebuild

### 6.1. Thứ tự thực hiện

1. **Backend** `catalog-store-service`:
   - `ComboService.java`
   - `ValidationService.java`
2. **Frontend** `nextjs`:
   - `lib/combo.ts` (file mới)
   - `lib/api.ts`
   - `lib/catalog-and-stores.api.ts`
   - `app/menu/[slug]/page.tsx`
   - (optional) `app/checkout/page.tsx`
3. **Không cần sửa**: `cart-order-service`, `api-gateway`, `frontend/react`.

### 6.2. Lệnh rebuild

```powershell
./start-docker.ps1 -Services catalog-store-service customer
```

> ⚠️ **Không dùng** `nextjs` — script sẽ báo `Unknown service: nextjs`.
> Service frontend Next.js tên là **`customer`** (`deploy/frontend/compose.yml:8`).

Script sẽ: build → up các service chỉ định → in `docker compose ps`.
Volumes/data được giữ nguyên.

---

## 7. Checklist kiểm thử dành cho người dùng

> Người dùng tự thực hiện kiểm thử trình duyệt. Không chạy Playwright/test tự động trừ khi được yêu cầu.

### 7.1. Chuẩn bị dữ liệu

```sql
-- Xem combo thực tế
SELECT id, name, combo_price, is_active, start_at, end_at
FROM catalog_service.combos ORDER BY id;

-- Xem thành phần
SELECT combo_id, product_id, quantity
FROM catalog_service.combo_items ORDER BY combo_id, product_id;

-- Xem tồn kho chi nhánh 1
SELECT store_id, product_id, quantity_available, is_sold_out
FROM catalog_service.store_inventory
WHERE store_id = 1 ORDER BY product_id;
```

### 7.2. Kịch bản 1 — Hiển thị combo trên menu

- [ ] Vào `/menu`, bấm tab **Combo**.
- [ ] Vào trực tiếp `/menu?cat=3`.
- [ ] **Kỳ vọng:** toàn bộ combo (Classic, Family Feast) hiển thị đầy đủ.
- [ ] DevTools → Elements: link combo phải có dạng `/menu/combo-family-feast-2`.
- [ ] (Hardening) Đổi tên category `Combo` trong DB → combo vẫn hiển thị nhờ `categoryId: 3`.

### 7.3. Kịch bản 2 — Điều hướng chi tiết combo

- [ ] Click combo **không** có chữ `Combo` ở đầu tên (VD: đổi tên thành `Family Feast`).
- [ ] Network phải gọi **`GET /api/catalog/combos/<id>?storeId=...`**.
- [ ] **Không** gọi `/api/catalog/products/<id>`.
- [ ] Trang hiển thị đúng tên, đúng giá combo (VD: 329.000đ), danh sách "Combo Items".
- [ ] Không hiện badge "Không tìm thấy món ăn".

### 7.4. Kịch bản 3 — Link cũ (legacy)

- [ ] Vào trực tiếp `/menu/family-feast-2` → vẫn ra combo 2.
- [ ] Vào `/menu/canh-ga-cay-mat-ong-2` → ra product, **không** nhảy sang combo.

### 7.5. Kịch bản 4 — Thêm giỏ & checkout

- [ ] Bấm "Thêm vào giỏ hàng".
- [ ] Network `POST /api/cart/items`: body có `"comboId": 2`, **không** có `productId`.
- [ ] Giỏ hiển thị đúng tên combo + giá combo.
- [ ] `/checkout`: preview trả `subtotal` = giá combo × số lượng.
- [ ] Đặt đơn thành công.
- [ ] Trang chi tiết đơn: item có `itemType = "COMBO"`, `comboId` đúng, tên đúng.

### 7.6. Kịch bản 5 — Combo hết hàng

- [ ] Set một thành phần combo về `quantity_available = 0` tại store đang chọn.
- [ ] Thêm combo vào giỏ → checkout.
- [ ] **Kỳ vọng:**
  - [ ] Preview trả `valid = false`.
  - [ ] Message nêu đúng tên món thành phần hết hàng.
  - [ ] Không tạo được đơn.
- [ ] Gọi trực tiếp `/api/catalog/internal/validate-items`:
  - [ ] `valid = false`.
  - [ ] `items[].lineTotal = 0` cho combo lỗi.
  - [ ] `subtotal` **không** cộng combo lỗi.

---

## 8. Rủi ro & lưu ý vận hành

| # | Rủi ro | Biện pháp |
|---|--------|-----------|
| 1 | Browser cache slug combo cũ | Hard refresh sau deploy. Backend có `Cache-Control: max-age=60`. |
| 2 | Redis menu cache chứa product cũ | Combos **không** nằm trong Redis cache; không cần evict. |
| 3 | Link/bookmark combo cũ `family-feast-2` | Đã xử lý bằng `legacyComboSlug`. |
| 4 | Admin đổi tên combo → link cũ chết | Logic detail ưu tiên so khớp ID + slug; nếu slug không khớp nhưng combo tồn tại cùng ID → vẫn hiển thị combo. |
| 5 | Product và combo trùng ID | Prefix `combo-` phân biệt rõ; URL không prefix luôn ưu tiên product khớp slug chính xác. |
| 6 | Sửa sót 1 trong 2 API client | Bắt buộc cập nhật cả `lib/api.ts` và `lib/catalog-and-stores.api.ts`. |
| 7 | `comboAsProduct` hardcode `categoryId = 3` | Nếu DB có category Combo ID khác → cần đọc động từ `getCategories()`. Hiện seed cố định ID 3. |
| 8 | Trang chủ không có combo | Ngoài phạm vi fix này — cần quyết định product requirement trước. |

---

## 9. Trạng thái hiện tại

- ✅ Đã đọc và đối chiếu trực tiếp toàn bộ file liên quan.
- ✅ Đã xác định chính xác dòng code và hành vi hiện tại.
- ❌ **Chưa sửa file nào.**
- ❌ **Chưa chạy** build / test / lint / typecheck / browser test (theo quy định `AGENTS.md`).

### File sẽ thay đổi khi implement

| File | Loại thay đổi |
|------|---------------|
| `backend/catalog-store-service/.../service/ComboService.java` | Thêm helper `comboSlug()` + sửa dòng 218 |
| `backend/catalog-store-service/.../service/ValidationService.java` | Sửa logic cộng `lineTotal` |
| `frontend/nextjs/lib/combo.ts` | **File mới** |
| `frontend/nextjs/lib/api.ts` | `comboAsProduct` + catch combos |
| `frontend/nextjs/lib/catalog-and-stores.api.ts` | `comboAsProduct` |
| `frontend/nextjs/app/menu/[slug]/page.tsx` | Logic resolve 2 chiều |
| `frontend/nextjs/app/checkout/page.tsx` | (optional) bỏ `prodId = 1` |

### Rebuild sau khi implement

```powershell
./start-docker.ps1 -Services catalog-store-service customer
```

---

*Tài liệu được tạo từ phân tích codebase trực tiếp ngày 17/11/2026. Không có file source nào bị thay đổi trong quá trình phân tích.*
