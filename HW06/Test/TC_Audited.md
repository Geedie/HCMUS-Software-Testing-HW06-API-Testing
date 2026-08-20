# Test Case Review & Bộ Test Case Hoàn Chỉnh — FR05 / FR08 / FR14 (E-Shop API)

## 0. Phương pháp đánh giá

Mỗi test case gốc (FR05.md, FR08.md, FR14.md) được đối chiếu với **`API_Infor.txt`** (nguồn spec gốc/ground truth) để kiểm tra: đúng endpoint, đúng test data theo seed, đúng expected result theo implementation đã mô tả, không gộp nhiều điều kiện vào 1 case, và không thiếu test condition mà spec đã liệt kê rõ.

Nhãn sử dụng:

-  **VALID** — Khớp spec, test data & expected result chính xác, giữ nguyên.
-  **INVALID** — Expected result hoặc test data sai lệch/không có cơ sở từ spec → đã sửa lại bên dưới.
-  **INCOMPLETE** — Về hướng đúng nhưng mơ hồ, gộp nhiều điều kiện trong 1 case, hoặc thiếu case liên quan mà spec yêu cầu → đã tách/bổ sung.

Cột **"Test data / Expected Result"** trong các bảng dưới đây là **phiên bản đã chỉnh sửa (final)**. Cột **"Đánh giá & Lý do"** ghi rõ nhãn gốc và lý do, kể cả với case VALID.

## Tổng hợp kết quả đánh giá

| FR | Tổng case gốc |  VALID |  INVALID |  INCOMPLETE (đã tách/sửa) | Case bổ sung mới | Tổng case sau cùng |
|---|---|---|---|---|---|---|
| FR05 | 35 | 33 | 1 | 1 | 0 | 35 |
| FR08 | 35 | 34 | 1 | 0 | 5 | 40 |
| FR14 | 35 | 30 | 0 | 5 (tách thành 12 case con) | 1 | 44 |
| **Tổng** | **105** | **97** | **2** | **6** | **6** | **119** |

---

# PHẦN 1 — FR05: Product Listing and Search

**Endpoint chính:** `GET /api/products` (list + `?search=`), `GET /api/products/:id`
**Authentication:** Không yêu cầu
**Seed data:** 5 products (id 1–5), 3 categories

**Rủi ro chính (đối chiếu `API_Infor.txt`):** SQL Injection do string interpolation trực tiếp trong `LIKE '%${searchQuery}%'`; `GET /api/products/:id` trả `200 + {}` khi không tồn tại thay vì `404`; `price` là `STRING` khi `id` **chẵn** (do `row.price = row.price.toString()`) và `INTEGER` khi `id` lẻ — **nhưng hành vi này spec chỉ mô tả cho `GET /api/products/:id`, không có bằng chứng áp dụng cho `GET /api/products` (listing)**.

## Nhóm 1 — Listing: GET /api/products (không có search)

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR05-01 | Status code khi lấy toàn bộ danh sách | `GET /api/products`, không kèm Authorization → `200` | High |  VALID — khớp mục 4 spec (không yêu cầu auth). |
| TC-FR05-02 | Response trả về là JSON array | `GET /api/products` → Body là array | High |  VALID — khớp mục 5 spec. |
| TC-FR05-03 | Số lượng sản phẩm khớp seed data | `GET /api/products` → đúng `5` phần tử | High |  VALID — khớp mục 7 (5 product đã seed). |
| TC-FR05-04 | Mỗi product có đủ field theo schema | `GET /api/products` → đủ `id, name, price, description, imageUrl, category_id` | High |  VALID — khớp mục 6 (product schema). |
| TC-FR05-05 | Kiểu dữ liệu của từng field đúng schema | `GET /api/products` → `id`: number, `name`: string, `description`: string, `category_id`: number, `price`: **number (INTEGER)** theo schema mục 6 | High |  **INVALID** — Bản gốc giả định `price` là number **hoặc** string tuỳ chẵn/lẻ ngay ở endpoint listing. Nhưng `API_Infor.txt` mục 11 chỉ mô tả hành vi `row.price.toString()` cho **`GET /api/products/:id`**, không có bằng chứng áp dụng cho `GET /api/products`. Đã sửa: kỳ vọng `price` là number nhất quán theo schema; **nếu thực thi thực tế cho thấy listing cũng bị ảnh hưởng bởi cùng transform, cần cập nhật lại case này và gộp vào defect schema-inconsistency chung với TC-FR05-34/35**, không giả định trước. |
| TC-FR05-06 | Dữ liệu thực tế khớp DB — product id=1 | `GET /api/products` → `id=1`: `name="iPhone 15 Pro Max"`, `price=30000000`, `category_id=1` | High |  VALID — khớp mục 7 product 1. |
| TC-FR05-07 | Dữ liệu thực tế khớp DB — product id=3 | `GET /api/products` → `id=3`: `name="MacBook Pro M3"`, `price=45000000`, `category_id=2` | Medium |  VALID — khớp mục 7 product 3. |
| TC-FR05-08 | Liên kết category_id hợp lệ (referential integrity) | `GET /api/products` → mọi `category_id` ∈ `{1,2,3}` | Medium |  VALID — khớp mục 8, seed category. |
| TC-FR05-09 | Response Content-Type đúng chuẩn | `GET /api/products` → `Content-Type: application/json` | Low |  VALID — quy ước REST hợp lý, không mâu thuẫn spec. |
| TC-FR05-10 | API hoạt động không cần Authorization (anonymous) | `GET /api/products` không kèm header → `200`, đầy đủ danh sách | Medium |  VALID — khớp mục 4 spec. |
| TC-FR05-11 | API vẫn hoạt động nếu Authorization header rác | `GET /api/products` với `Authorization: Bearer invalid-garbage-token` → `200`, kết quả không đổi | Low |  VALID — hợp lý vì endpoint không yêu cầu auth nên header thừa không được xử lý. |

## Nhóm 2 — Search: happy path & các biến thể từ khoá hợp lệ

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR05-12 | Search bằng từ khoá tồn tại (một phần tên) | `GET /api/products?search=iPhone` → `200`, đúng 1 sản phẩm id=1 | High |  VALID. |
| TC-FR05-13 | Search bằng tên đầy đủ, khớp chính xác | `GET /api/products?search=iPhone 15 Pro Max` → `200`, id=1 | High |  VALID. |
| TC-FR05-14 | Search một phần tên ở giữa chuỗi (`LIKE %...%`) | `GET /api/products?search=Pro` → id=1 (iPhone 15 **Pro** Max), id=3 (MacBook **Pro** M3), id=4 (Tai nghe AirPods **Pro** 2) | High |  VALID — khớp mục 7, cả 3 tên đều chứa "Pro". |
| TC-FR05-15 | Search không có kết quả khớp | `GET /api/products?search=Xiaomi` → `200`, `[]` | High |  VALID. |
| TC-FR05-16 | Search không phân biệt hoa/thường — chữ thường | `GET /api/products?search=iphone` → ghi nhận kết quả thực tế (SQLite `LIKE` mặc định case-insensitive với ASCII nên khả năng cao match id=1, nhưng không giả định trước) | High |  VALID — spec mục 9 liệt kê case này nhưng không khẳng định trước behavior; cách tiếp cận "ghi nhận thực tế" là đúng. |
| TC-FR05-17 | Search không phân biệt hoa/thường — chữ hoa toàn bộ | `GET /api/products?search=IPHONE` → phải nhất quán với TC-FR05-16 | High |  VALID. |
| TC-FR05-18 | Search bằng số nằm trong tên sản phẩm | `GET /api/products?search=15` → id=1 | Medium |  VALID. |
| TC-FR05-19 | Search Unicode tiếng Việt (không khớp field name) | `GET /api/products?search=Điện thoại` → `200`, `[]` (search chỉ áp dụng trên `name`) | Medium |  VALID — khớp mục 5 spec (WHERE name LIKE...). |
| TC-FR05-20 | Search nhiều ký tự dài (partial match hợp lệ) | `GET /api/products?search=Bàn phím cơ Keychron` → id=5 | Low |  VALID — khớp tên thật "Bàn phím cơ Keychron Q1". |

## Nhóm 3 — Search: boundary, input bất thường & query parameter không hợp lệ

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR05-21 | Search với chuỗi rỗng | `GET /api/products?search=` → `200`, `LIKE '%%'` → toàn bộ 5 sản phẩm | High |  VALID. |
| TC-FR05-22 | Không truyền query parameter search | `GET /api/products` → `200`, toàn bộ 5 sản phẩm, giống hệt TC-FR05-21 | Medium |  VALID. |
| TC-FR05-23 | Search chỉ gồm khoảng trắng | `GET /api/products?search=%20%20%20` → ghi nhận có trim hay không | Medium |  VALID. |
| TC-FR05-24 | Search có khoảng trắng đầu/cuối từ khoá hợp lệ | `GET /api/products?search=%20iPhone%20` → ghi nhận có trim hay không | Medium |  VALID. |
| TC-FR05-25 | Search ký tự đặc biệt không mang tính injection | `GET /api/products?search=@#$%^&*()` → `200`, không lỗi 500, `[]` | High |  VALID. |
| TC-FR05-26 | Search ký tự `%` (trùng wildcard SQL LIKE) | `GET /api/products?search=%` → `200`, ghi nhận có trả toàn bộ 5 sản phẩm hay không (`LIKE '%%%'`) | Medium |  VALID. |
| TC-FR05-27 | Search chuỗi rất dài | `GET /api/products?search=` + chuỗi 500 ký tự ngẫu nhiên (ví dụ: 500 ký tự `a` lặp lại, `"a".repeat(500)`) → `200`, không lỗi 500/timeout, `[]` | Medium |  **INCOMPLETE** (đã sửa) — Bản gốc mô tả test data bằng lời ("chuỗi 500 ký tự ngẫu nhiên") không có giá trị cụ thể để tái lập nhất quán khi automation hoá. Đã bổ sung ví dụ cụ thể (`"a".repeat(500)`) để test có thể tái lập được. |
| TC-FR05-28 | Search query parameter dạng mảng | `GET /api/products?search[]=iPhone` → không lỗi 500; ghi nhận xử lý thực tế (ignore/`[]`/coi như không search) | High |  VALID. |
| TC-FR05-29 | Nhiều query parameter `search` trùng tên | `GET /api/products?search=iPhone&search=Samsung` → không lỗi 500, ghi nhận xử lý nhất quán | Medium |  VALID. |

## Nhóm 4 — Security: SQL Injection

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR05-30 | SQL Injection dạng tautology | `GET /api/products?search=' OR '1'='1` → không lỗi 500; nếu trả về nhiều hơn phạm vi tìm kiếm hợp lệ (ví dụ toàn bộ 5 sản phẩm) → **Critical defect** | Critical |  VALID — khớp mục 10 spec, rủi ro ưu tiên cao nhất FR05. |
| TC-FR05-31 | SQL Injection phá hoại (DROP TABLE) | `GET /api/products?search='; DROP TABLE products; --` → không lỗi 500; chạy lại TC-FR05-01 để xác nhận bảng `products` còn nguyên 5 sản phẩm | Critical |  VALID. |
| TC-FR05-32 | SQL Injection UNION SELECT | `GET /api/products?search=' UNION SELECT * FROM sqlite_master --` → không lỗi 500, không lộ dữ liệu bảng hệ thống/bảng khác | Critical |  VALID. |
| TC-FR05-33 | SQL Injection dùng comment (`--`) | `GET /api/products?search=iPhone'--` → không lỗi 500, kết quả không bị thao túng ngoài phạm vi tìm kiếm hợp lệ | High |  VALID. |

## Nhóm 5 — GET /api/products/:id

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR05-34 | Product tồn tại, id lẻ → `price` kiểu INTEGER | `GET /api/products/1` → `200`; `price` kiểu **number** = `30000000` | High |  VALID — khớp mục 11 spec (id lẻ giữ nguyên INTEGER). |
| TC-FR05-35 | Product tồn tại, id chẵn → `price` kiểu STRING | `GET /api/products/2` → `200`; `price` kiểu **string** = `"28000000"` (Samsung Galaxy S24 Ultra) → flag defect schema-inconsistency nếu không phải chủ đích | Critical |  VALID — khớp mục 11 spec chính xác (`row.price.toString()` khi id chẵn). |

**Ghi chú:** Không phát hiện thiếu test condition nào so với mục 9 `API_Infor.txt` cho FR05 (ngoại trừ sửa TC-FR05-05). Coverage đầy đủ.

---

# PHẦN 2 — FR08: Checkout

**Endpoints:** `POST /api/login`, `GET/POST /api/cart`, `POST /api/checkout`, `GET /api/orders/my-orders`, `GET /api/orders/:id`
**Authentication:** Bearer JWT, bắt buộc cho mọi endpoint cart/checkout, **trừ** `GET /api/orders/:id`

## Nhóm 1 — Login & Authentication

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-01 | Login thành công với user thường | `POST /api/login` `{"email":"test@eshop.com","password":"Test1234!"}` → `200`, có `token` và `user` | High |  VALID — khớp mục 3 spec. |
| TC-FR08-02 | Login thành công với admin | `POST /api/login` `{"email":"admin@eshop.com","password":"Admin123!"}` → `200`, `token` và `user.role="admin"` | High |  VALID — khớp mục 2 (role admin trong DB). |
| TC-FR08-03 | Login sai password | `POST /api/login` `{"email":"test@eshop.com","password":"SaiPassword"}` → `401`, không trả `token` | High |  VALID. |
| TC-FR08-04 | Login với email không tồn tại | `POST /api/login` `{"email":"khongtontai@eshop.com","password":"Test1234!"}` → `401`, không trả `token` | Medium |  VALID. |

## Nhóm 2 — Access control trên Cart/Checkout

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-05 | GET cart không có Authorization header | `GET /api/cart`, không kèm header → `401` | High |  VALID. |
| TC-FR08-06 | POST cart không có Authorization header | `POST /api/cart`, không kèm header, body hợp lệ → `401` | High |  VALID. |
| TC-FR08-07 | Checkout không có Authorization header | `POST /api/checkout`, không kèm header, body hợp lệ → `401` | High |  VALID. |
| TC-FR08-08 | Gọi API với token malformed/invalid signature | `GET /api/cart` với `Authorization: Bearer invalid.token.here` → **`403 Forbidden`** | High |  **INVALID** — Bản gốc kỳ vọng `401`. Theo `API_Infor.txt` mục FR14-8, hệ thống dùng chung cơ chế `authenticateToken` và **document rõ**: không có token → `401`; **token invalid → `403 Forbidden`**. Đây là pattern middleware JWT phổ biến (`if (!token) return 401; jwt.verify(err) => return 403`), và FR08/FR14 dùng chung middleware `authenticateToken`. Case "không có header" đã có TC-FR08-05/06/07 (đúng 401); case "có header nhưng token sai" nên tách biệt kỳ vọng `403` để nhất quán với FR14 và không gây nhầm lẫn khi debug thực tế. |

## Nhóm 3 — Cart: chức năng cơ bản

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-09 | GET cart khi user chưa từng thêm gì | Login user mới, `GET /api/cart` → `200`, `[]` | High |  VALID — khớp mục 4 spec. |
| TC-FR08-10 | Add 1 sản phẩm vào cart hợp lệ | `POST /api/cart` `{"id":1,"name":"iPhone 15 Pro Max","price":30000000,"quantity":1}` → `200`, `{"message":"Added to cart"}` | High |  VALID — khớp mục 5 spec. |
| TC-FR08-11 | GET cart sau khi add — xác nhận dữ liệu đúng | Sau TC-FR08-10, `GET /api/cart` → array chứa đúng item vừa add | High |  VALID. |
| TC-FR08-12 | Add nhiều sản phẩm liên tiếp | `POST /api/cart` 2 lần với 2 sản phẩm khác nhau → `GET /api/cart` trả về cả 2 | Medium |  VALID. |
| TC-FR08-13 | Cart cô lập theo từng user (data isolation) | User A add sản phẩm; login User B; `GET /api/cart` bằng token B → không chứa sản phẩm của A | High |  VALID — khớp mục 4 spec ("Cart được phân biệt theo userId"). |

## Nhóm 4 — Cart: lỗ hổng validation

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-14 | Add với quantity = 0 | `{"quantity":0}` → `200` chấp nhận → flag nếu nghiệp vụ không cho phép | Medium |  VALID — khớp mục 12.G. |
| TC-FR08-15 | Add với quantity âm | `{"quantity":-5}` → `200` chấp nhận → **Critical defect** | High |  VALID. |
| TC-FR08-16 | Add với quantity cực lớn | `{"quantity":999999999}` → `200` chấp nhận, không giới hạn | Medium |  VALID. |
| TC-FR08-17 | Add sản phẩm với `id` không tồn tại | `{"id":999999,"name":"Fake Product","price":1,"quantity":1}` → `200` → **Critical defect** | High |  VALID — khớp mục 12.G (ví dụ gần như y hệt spec). |
| TC-FR08-18 | Add sản phẩm với `price` không khớp giá thật | `{"id":1,"name":"iPhone 15 Pro Max","price":1,"quantity":1}` (giá thật 30000000) → `200` → **Critical defect** | High |  VALID — khớp mục 12.B/G. |

## Nhóm 5 — Checkout: Positive

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-19 | Checkout thành công với dữ liệu hợp lệ | `{"total_amount":200000,"shipping_address":"123 Le Loi, TP.HCM"}` → `200`, `{"message":"Checkout successful","orderId":<số>}` | High |  VALID — khớp mục 6 spec (ví dụ y hệt). |
| TC-FR08-20 | Order được tạo với status mặc định | Sau TC-FR08-19, `GET /api/orders/my-orders` → `status="pending"` | High |  VALID — khớp mục 7, 8. |
| TC-FR08-21 | total_amount khớp đúng request | Sau TC-FR08-19 → `total_amount=200000` | High |  VALID. |
| TC-FR08-22 | shipping_address khớp đúng request | Sau TC-FR08-19 → `"123 Le Loi, TP.HCM"` | High |  VALID. |
| TC-FR08-23 | orderId trả về hợp lệ và tra cứu được | `GET /api/orders/:id` với id từ TC-FR08-19 → `200`, đúng order | High |  VALID — khớp mục 10. |

## Nhóm 6 — Checkout: Negative & Validation

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-24 | Checkout thiếu total_amount | `{"shipping_address":"123 Le Loi"}` → ghi nhận behavior thực tế (không có validation rõ trong spec) | High |  VALID — khớp mục 12.C. |
| TC-FR08-25 | Checkout thiếu shipping_address | `{"total_amount":200000}` → ghi nhận behavior thực tế | High |  VALID — khớp mục 12.D. |
| TC-FR08-26 | Checkout với total_amount = 0 | `{"total_amount":0,...}` → `200` chấp nhận → flag | Medium |  VALID. |
| TC-FR08-27 | Checkout với total_amount âm | `{"total_amount":-500000,...}` → `200` → **Critical defect** | High |  VALID. |
| TC-FR08-28 | Checkout với total_amount không phải number | `{"total_amount":"abc",...}` → ghi nhận `500` hoặc insert giá trị không hợp lệ | High |  VALID. |
| TC-FR08-29 | Checkout với shipping_address rỗng | `{"shipping_address":""}` → `200` chấp nhận → flag | Medium |  VALID. |
| TC-FR08-30 | Field thừa trong body (mass assignment) | `{"total_amount":200000,"shipping_address":"123 Le Loi","status":"completed","user_id":9999}` → verify order tạo ra dùng `userId` từ JWT, `status="pending"`, field thừa bị bỏ qua | High |  VALID — khớp mục 11 ("Kiểm tra userId lấy từ token chứ không lấy từ request"). |

## Nhóm 7 — Critical: Business logic & data integrity

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-31 | Checkout thành công ngay cả khi cart rỗng | User mới/cart rỗng → `POST /api/checkout` body hợp lệ → `200` → **Critical defect** | High |  VALID — khớp mục 12.A chính xác. |
| TC-FR08-32 | total_amount không được server tính lại từ cart | Add sản phẩm giá thật 30000000 vào cart → checkout `total_amount:1000` → server chấp nhận tuỳ ý → **Critical defect** | High |  VALID — khớp mục 12.B chính xác. |
| TC-FR08-33 | Cart không bị xoá sau checkout thành công | Add sản phẩm → checkout thành công → `GET /api/cart` → cart vẫn còn sản phẩm cũ | Medium |  VALID — khớp mục 12.E ("Không có... cart clear"). |

## Nhóm 8 — Order verification & Authorization

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-34 | GET order với id không tồn tại | `GET /api/orders/999999` → `404`, `{"error":"Order not found"}` | Medium |  VALID — khớp mục 10 spec chính xác. |
| TC-FR08-35 | IDOR: User A xem được order của User B | User B checkout → lấy `orderId` → User A (token khác hoặc không token) gọi `GET /api/orders/<orderId B>` → trả về đầy đủ dữ liệu order B → **Critical IDOR/BOLA** | Critical |  VALID — khớp mục 10 spec, đây là rủi ro ưu tiên cao nhất được liệt kê rõ. |

## Nhóm 9 (BỔ SUNG) — Case còn thiếu theo mục 11 & 12.G của `API_Infor.txt`

| ID | Mục tiêu | Test data / Expected Result | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR08-36 | Checkout với total_amount rất lớn | `POST /api/checkout` `{"total_amount":99999999999999,"shipping_address":"123 Le Loi"}` → ghi nhận behavior thực tế: `200` chấp nhận, hoặc lỗi do vượt giới hạn cột `INTEGER` (SQLite dùng kiểu động nên khả năng cao vẫn `200`) — cần xác nhận không có overflow/crash | Medium | ➕ **Bổ sung mới** — Spec mục 11 liệt kê rõ "total_amount rất lớn" trong Negative test nhưng bản gốc chưa có case tương ứng. |
| TC-FR08-37 | Checkout với shipping_address = null | `POST /api/checkout` `{"total_amount":200000,"shipping_address":null}` → ghi nhận behavior thực tế (có thể `200` insert NULL, hoặc lỗi) | Medium | ➕ **Bổ sung mới** — Spec mục 11 liệt kê "shipping_address null" riêng biệt với "rỗng" (đã có ở TC-FR08-29) nhưng chưa có case null. |
| TC-FR08-38 | Checkout với shipping_address quá dài | `POST /api/checkout` `{"total_amount":200000,"shipping_address":"<chuỗi 5000 ký tự>"}` → ghi nhận `200` (cột `TEXT` không giới hạn) hoặc lỗi, không được `500`/timeout không rõ nguyên nhân | Low | ➕ **Bổ sung mới** — Spec mục 11 liệt kê "shipping_address quá dài" chưa có case tương ứng. |
| TC-FR08-39 | Add vào cart với `name` không khớp DB | `POST /api/cart` `{"id":1,"name":"Sản phẩm giả mạo","price":30000000,"quantity":1}` (tên thật là "iPhone 15 Pro Max") → `200` chấp nhận vì không đối chiếu DB → **Critical defect**: cart chứa tên sản phẩm sai lệch, ảnh hưởng hiển thị/hoá đơn | High | ➕ **Bổ sung mới** — Spec mục 11 (Cart) liệt kê rõ "Name không khớp database" là điều kiện riêng biệt với "Price không khớp database" (đã có ở TC-FR08-18), nhưng bản gốc chỉ test price, thiếu test name. |
| TC-FR08-40 | Checkout với request body malformed (không phải JSON hợp lệ) | `POST /api/checkout` với `Content-Type: application/json` nhưng body là chuỗi không phải JSON hợp lệ (ví dụ `"total_amount": 200000,` thiếu dấu ngoặc) → ghi nhận behavior thực tế: kỳ vọng `400 Bad Request` từ JSON body-parser; không được `500` không rõ nguyên nhân | Medium | ➕ **Bổ sung mới** — Spec mục 11 liệt kê "Request body malformed" trong Negative test, bản gốc chưa có case tương ứng. |

**Ghi chú thêm (không tính vào case, mang theo từ bản gốc và giữ nguyên):**
- TC-FR08-31, 32, 35 vẫn là 3 defect nghiêm trọng nhất, nên tách defect report riêng.
- Cart in-memory (`userCarts = {}`) nên cần thêm 1 test ngoài phạm vi API thuần để xác nhận mất cart khi restart server (phối hợp dev/infra).
- Nhóm 7 và TC-FR08-35 là Exit Criteria blocker.
- Case "token expired" (mục 11 spec) chưa có test tương ứng vì cần khả năng tạo JWT hết hạn thủ công (ký bằng secret giả lập với `exp` trong quá khứ) — khuyến nghị bổ sung khi có công cụ tạo token hỗ trợ, không đưa vào bảng chính vì cần thiết lập riêng.

---

# PHẦN 3 — FR14: Category Management CRUD

**Endpoints:** `GET/POST /api/categories`, `PUT/DELETE /api/categories/:id`
**Authentication:** `GET` không yêu cầu; `POST/PUT/DELETE` yêu cầu Bearer token
**Seed data:** `1=Điện thoại, 2=Laptop, 3=Phụ kiện`; Product `id=1,2` đang dùng `category_id=1`

## Nhóm 1 — GET /api/categories

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-01 | Status code khi lấy danh sách category | `GET /api/categories` → `200` | High |  VALID. |
| TC-FR14-02 | Response là JSON array | `GET /api/categories` → array | High |  VALID. |
| TC-FR14-03 | Dữ liệu khớp seed | `GET /api/categories` → đúng 3 phần tử: `{id:1,name:"Điện thoại"}`, `{id:2,name:"Laptop"}`, `{id:3,name:"Phụ kiện"}` | High |  VALID — khớp mục 3 spec chính xác. |
| TC-FR14-04 | Kiểu dữ liệu từng field đúng schema | `id`: number, `name`: string; không thiếu/thừa field | Medium |  VALID — khớp mục 2 spec. |
| TC-FR14-05 | API hoạt động không cần Authorization | `GET /api/categories` không header → `200`, dữ liệu bình thường | Medium |  VALID — khớp mục 4 spec. |

## Nhóm 2 — CREATE category: Positive & Validation

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-06 | Tạo category hợp lệ (token admin) | `POST /api/categories` `{"name":"Đồng hồ"}` → **`200`**, `{"message":"Category created","id":<số>}` | High |  **INCOMPLETE** (đã sửa) — Bản gốc để mơ hồ `200/201`. Toàn bộ các endpoint ghi dữ liệu khác trong spec (`checkout`, `add to cart`, `login`) đều trả `200` (không có endpoint nào trả `201` trong toàn bộ `API_Infor.txt`), nên expected result nên chốt `200` làm kỳ vọng chính, tránh mơ hồ gây khó xác định Pass/Fail. |
| TC-FR14-07 | Category mới xuất hiện sau khi tạo | Sau TC-FR14-06, `GET /api/categories` → chứa category vừa tạo đúng `id`, `name="Đồng hồ"` | High |  VALID. |
| TC-FR14-08 | Tạo category không gửi body | `POST /api/categories` không có body → ghi nhận behavior thực tế (kỳ vọng `400`, có thể `500` hoặc insert `name=null`) | High |  VALID — khớp mục 5 spec ("Không gửi body"). |
| TC-FR14-09 | Tạo category thiếu field `name` | `POST /api/categories` `{}` → ghi nhận behavior thực tế | High |  VALID — khớp mục 5 ("Không gửi name"). |
| TC-FR14-10 | Tạo category với `name` rỗng | `{"name":""}` → có khả năng vẫn `200` → flag defect nếu đúng | High |  VALID. |
| TC-FR14-11 | Tạo category với `name` là `null` | `{"name":null}` → ghi nhận behavior thực tế | Medium |  VALID. |
| TC-FR14-12 | Tạo category với `name` không phải string | `{"name":12345}` → ghi nhận behavior thực tế | Medium |  VALID. |
| TC-FR14-13 | Tạo category với `name` quá dài | `{"name":"<chuỗi 2000 ký tự>"}` → khả năng vẫn `200`, xác nhận không `500`/timeout | Medium |  VALID. |
| TC-FR14-14 | Tạo category với `name` Unicode tiếng Việt | `{"name":"Thiết bị gia dụng"}` → `200`, lưu/trả đúng nguyên văn | Medium |  VALID. |
| TC-FR14-15 | Tạo category với `name` chứa ký tự đặc biệt | `{"name":"<script>alert(1)</script>"}` → `200`, không lỗi `500`, dữ liệu lưu nguyên văn | Medium |  VALID. |

## Nhóm 3 — Duplicate category name

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-16 | Tạo category trùng tên đã tồn tại | `{"name":"Laptop"}` (đã có id=2) → khả năng vẫn `200`, tạo thêm 1 category "Laptop" thứ 2 → flag defect | High |  VALID — khớp mục 10 spec (không có `UNIQUE(name)`). |
| TC-FR14-17 | Tạo trùng tên nhiều lần liên tiếp | `POST /api/categories` `{"name":"Laptop"}` 3 lần → `GET /api/categories` có ≥3 category "Laptop" | Medium |  VALID. |

## Nhóm 4 — UPDATE category

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-18 | Update category tồn tại thành công | `PUT /api/categories/3` `{"name":"Phụ kiện điện tử"}` → `200`, `{"message":"Category updated"}`; `GET` xác nhận đổi tên | High |  VALID. |
| TC-FR14-19 | Update category không tồn tại | `PUT /api/categories/999999` `{"name":"Không tồn tại"}` → ghi nhận behavior thực tế (kỳ vọng `404`, có thể vẫn `200` do không check rowCount) | High |  VALID — khớp mục 6 spec. |
| TC-FR14-20 | Update với id = 0 | `PUT /api/categories/0` → ghi nhận behavior | Medium |  VALID. |
| TC-FR14-21 | Update với id âm | `PUT /api/categories/-1` → ghi nhận behavior, không lỗi `500` | Medium |  VALID. |
| TC-FR14-22 | Update với id không phải số | `PUT /api/categories/abc` → ghi nhận behavior, không `500` không rõ nguyên nhân | Medium |  VALID. |
| TC-FR14-23 | Update với `name` rỗng | `PUT /api/categories/2` `{"name":""}` → khả năng vẫn `200` → flag defect | High |  VALID. |
| TC-FR14-24a | Update với `name` = null | `PUT /api/categories/2` `{"name":null}` → ghi nhận behavior thực tế, không `500` | Medium |  **INCOMPLETE** (đã tách) — Bản gốc gộp "name null" và "name=123" vào chung 1 case (TC-FR14-24), vi phạm nguyên tắc 1 test case = 1 điều kiện (`references/test-design.md`), khiến kết quả Pass/Fail của case bị nhập nhằng nếu chỉ 1 trong 2 điều kiện fail. Tách thành 24a/24b để trace độc lập. |
| TC-FR14-24b | Update với `name` sai kiểu dữ liệu (number) | `PUT /api/categories/2` `{"name":123}` → ghi nhận behavior thực tế, không `500` | Medium |  **INCOMPLETE** (đã tách) — xem lý do ở TC-FR14-24a. |
| TC-FR14-25 | Update `name` trùng với category khác | `PUT /api/categories/2` `{"name":"Phụ kiện"}` (trùng id=3) → khả năng vẫn `200` → flag defect | Medium |  VALID. |

## Nhóm 5 — DELETE category

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-26 | Delete category tồn tại, không bị tham chiếu | Tạo category test riêng → `DELETE /api/categories/<id>` → `200`, `{"message":"Category deleted"}`; `GET` xác nhận biến mất | High |  VALID. |
| TC-FR14-27 | Delete category không tồn tại | `DELETE /api/categories/999999` → ghi nhận behavior thực tế (kỳ vọng `404`, có thể vẫn `200`) | High |  VALID. |
| TC-FR14-28 | Delete với id = 0 | `DELETE /api/categories/0` → ghi nhận behavior, không `500` | Medium |  VALID. |
| TC-FR14-29 | Delete với id âm | `DELETE /api/categories/-1` → ghi nhận behavior, không `500` | Medium |  VALID. |
| TC-FR14-30 | Delete với id non-numeric | `DELETE /api/categories/abc` → ghi nhận behavior, không `500` không rõ nguyên nhân | Medium |  VALID. |
| TC-FR14-31 | Delete category đang được product sử dụng — orphan reference | `DELETE /api/categories/1` (category "Điện thoại", đang được product `id=1,2` tham chiếu `category_id=1`), sau đó `GET /api/products/1` **và** `GET /api/products/2` | Critical |  VALID (đã bổ sung nhẹ) — Test data khớp chính xác seed spec (`API_Infor.txt`: product 1 và 2 đều có `category_id=1`). Bản gốc chỉ verify lại `GET /api/products/1`; đã bổ sung verify thêm `GET /api/products/2` vì cả 2 product cùng bị ảnh hưởng, giúp bằng chứng defect đầy đủ hơn. |

## Nhóm 6 — Authorization: thiếu kiểm tra role admin

| ID | Mục tiêu | Test data / Expected Result (final) | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-32a | POST category không có token | `POST /api/categories` không kèm `Authorization` → `401 Unauthorized` | High |  **INCOMPLETE** (đã tách) — Bản gốc gộp 3 endpoint (POST/PUT/DELETE) vào 1 case, không nhất quán với cách FR08 tách riêng từng endpoint (TC-FR08-05/06/07). Tách theo endpoint để dễ trace lỗi (ví dụ chỉ DELETE bị thiếu check mà POST/PUT đúng vẫn có thể xảy ra). |
| TC-FR14-32b | PUT category không có token | `PUT /api/categories/1` không kèm `Authorization` → `401 Unauthorized` | High |  INCOMPLETE (đã tách) — cùng lý do TC-FR14-32a. |
| TC-FR14-32c | DELETE category không có token | `DELETE /api/categories/1` không kèm `Authorization` → `401 Unauthorized` | High |  INCOMPLETE (đã tách) — cùng lý do TC-FR14-32a. |
| TC-FR14-33a | POST category với token invalid | `POST /api/categories` với `Authorization: Bearer invalid-token` → `403 Forbidden` | High |  INCOMPLETE (đã tách) — cùng lý do TC-FR14-32a; giá trị `403` khớp chính xác mục 8 spec. |
| TC-FR14-33b | PUT category với token invalid | `PUT /api/categories/1` với `Authorization: Bearer invalid-token` → `403 Forbidden` | High |  INCOMPLETE (đã tách). |
| TC-FR14-33c | DELETE category với token invalid | `DELETE /api/categories/1` với `Authorization: Bearer invalid-token` → `403 Forbidden` | High |  INCOMPLETE (đã tách). |
| TC-FR14-34a | User thường (role="user") POST category | Login `test@eshop.com`/`Test1234!` → `POST /api/categories` với token user → `200` thành công dù không có quyền admin → **Critical defect: Broken Function-Level Authorization** | Critical |  INCOMPLETE (đã tách) — cùng lý do TC-FR14-32a; đây là defect quan trọng nhất FR14 nên càng cần tách rõ để report riêng từng action nếu chỉ 1/3 action bị lỗi. |
| TC-FR14-34b | User thường (role="user") PUT category | Token user → `PUT /api/categories/:id` → `200` thành công dù không có quyền admin → **Critical defect** | Critical |  INCOMPLETE (đã tách). |
| TC-FR14-34c | User thường (role="user") DELETE category | Token user → `DELETE /api/categories/:id` → `200` thành công dù không có quyền admin → **Critical defect** | Critical |  INCOMPLETE (đã tách). |
| TC-FR14-35 | Admin token hợp lệ — baseline | Login admin → `POST`, `PUT`, `DELETE` category → cả 3 đều `200` thành công | High |  VALID — dùng làm baseline so sánh với TC-FR14-34a/b/c. |

## Nhóm 7 (BỔ SUNG) — Case còn thiếu theo mục 4 của `API_Infor.txt`

| ID | Mục tiêu | Test data / Expected Result | Priority | Đánh giá & Lý do |
|---|---|---|---|---|
| TC-FR14-36 | GET categories khi database rỗng (sau reset) | Yêu cầu môi trường test hỗ trợ reset DB về trạng thái rỗng → `GET /api/categories` → `200`, `[]` | Low | **Bổ sung mới** — Spec mục 4 liệt kê rõ "Kiểm tra empty database nếu có thể reset" nhưng bản gốc không có case này. Ghi chú: case này phụ thuộc khả năng reset môi trường test (thường không khả thi trên môi trường shared/staging) — cần xác nhận với team hạ tầng trước khi đưa vào automation suite chính; có thể chuyển sang test riêng chạy trên DB tạm/local. |

**Ghi chú thêm (giữ nguyên từ bản gốc):**
- TC-FR14-34a/b/c (trước đây là TC-FR14-34 gộp) vẫn là defect nghiêm trọng nhất FR14, nên tách 3 defect report riêng hoặc gộp 1 report có 3 bước reproduce.
- TC-FR14-31 nên đối chiếu chéo với kết quả FR05 (`GET /api/products/:id`).
- Theo Exit Criteria: nếu TC-FR14-31 hoặc bất kỳ case nào trong nhóm TC-FR14-34a/b/c fail (lỗ hổng tồn tại), khuyến nghị **không release** phần category management cho tới khi có bản vá.

---

# Tổng kết đánh giá chéo giữa 3 FR (phát hiện khi review tổng thể)

- **Tính nhất quán về mã lỗi token invalid:** Spec (`API_Infor.txt`, mục FR14-8) là nơi duy nhất khẳng định rõ ràng `403 Forbidden` cho token không hợp lệ, dùng chung middleware `authenticateToken` với FR08. Việc TC-FR08-08 (bản gốc) kỳ vọng `401` là điểm không nhất quán nội bộ giữa 2 bộ test case cho cùng 1 middleware — đã sửa để đồng bộ.
- **Nguyên tắc 1 test case = 1 điều kiện:** FR08 đã tách riêng từng endpoint cho authorization test (TC-FR08-05/06/07), nhưng FR14 lại gộp 3 endpoint (POST/PUT/DELETE) vào 1 case (TC-FR14-32/33/34) — đã tách lại cho nhất quán và dễ trace lỗi hơn khi chỉ một action bị lỗi.
- **Case dựa trên hành vi chưa xác nhận:** TC-FR05-05 là case duy nhất áp dụng nhầm một hành vi (`price` type theo chẵn/lẻ id) chỉ được spec xác nhận cho một endpoint khác (`GET /api/products/:id`) sang endpoint listing — đã sửa để không giả định trước.
- **Coverage gap so với `API_Infor.txt`:** đã bổ sung 6 case mới (TC-FR08-36→40, TC-FR14-36) cho các test condition được spec liệt kê rõ ràng ở mục 9/11/4 nhưng chưa có test case tương ứng trong bản gốc.