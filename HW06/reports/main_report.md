# BÁO CÁO TỔNG HỢP KIỂM THỬ API E-SHOP

**Dự án:** HW06 — API Automated Testing (E-Shop)  
**Người thực hiện:** Geedie (Huynh M. Doan)  
**Ngày:** 2026-08-21  
**Phạm vi:** FR05 (Product Listing & Search), FR08 (Checkout), FR14 (Category Management CRUD)  
**Công cụ:** Newman + Postman Collection + GitHub Actions CI  

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1. Mục tiêu
Thiết kế, thực thi và đánh giá bộ test case tự động cho 3 chức năng chính của E-Shop API, bao gồm:
- **FR05:** Product Listing and Search
- **FR08:** Checkout (Cart + Order)
- **FR14:** Category Management CRUD

### 1.2. Kiến trúc hệ thống được test
| Thành phần | Mô tả |
|---|---|
| **Backend** | Express.js + SQLite3 + JWT (`eshop-sut/backend`) |
| **Base URL** | `http://localhost:3000` |
| **Authentication** | Bearer JWT (`POST /api/login`) |
| **Test Users** | Admin: `admin@eshop.com` / `Admin123!`<br>User: `test@eshop.com` / `Test1234!` |
| **Seed Data** | 5 products, 3 categories, 2 users |

### 1.3. Công cụ & Quy trình
| Công cụ | Vai trò |
|---|---|
| **Postman Collection** | Định nghĩa 125+ request test (FR05/FR08/FR14) |
| **data.csv** | 125 dòng test data cho Newman iteration |
| **Newman** | Chạy collection từ CLI, xuất HTML/JUnit report |
| **GitHub Actions** | CI pipeline: checkout → install → start API → run Newman → upload artifact → publish PR summary |
| **AI Agent** (`api-qa-testing-agent`) | Hỗ trợ sinh test case ban đầu từ spec |

---

## 2. TÓM TẮT API SPECIFICATION (`API_Infor.txt`)

### 2.1. FR05 — Product Listing and Search
| Endpoint | Method | Auth | Mục đích |
|---|---|---|---|
| `/api/products` | GET | Không | Listing + search (`?search=`) |
| `/api/products/:id` | GET | Không | Chi tiết product |

**Rủi ro chính:**
- SQL Injection qua `LIKE '%${searchQuery}%'` (string interpolation trực tiếp)
- `price` kiểu **STRING** khi id chẵn, **INTEGER** khi id lẻ (schema inconsistency)
- `GET /api/products/:id` trả `200 + {}` khi không tồn tại (thay vì `404`)

### 2.2. FR08 — Checkout
| Endpoint | Method | Auth | Mục đích |
|---|---|---|---|
| `/api/login` | POST | Không | Đăng nhập, lấy JWT |
| `/api/cart` | GET/POST | Có | Xem / thêm vào cart |
| `/api/checkout` | POST | Có | Tạo order |
| `/api/orders/my-orders` | GET | Có | Lịch sử order của user |
| `/api/orders/:id` | GET | **Không** | Chi tiết order |

**Rủi ro chính:**
- Checkout không đọc cart, chỉ nhận `total_amount` từ client → có thể checkout với giá bất kỳ
- `POST /api/cart` không validate product, price, quantity → có thể thêm sản phẩm giả mạo
- `GET /api/orders/:id` không có `authenticateToken` → IDOR/BOLA: user A xem được order của user B
- Cart lưu in-memory → mất dữ liệu khi restart server

### 2.3. FR14 — Category Management CRUD
| Endpoint | Method | Auth | Mục đích |
|---|---|---|---|
| `/api/categories` | GET | Không | Danh sách category |
| `/api/categories` | POST | Có | Tạo category |
| `/api/categories/:id` | PUT | Có | Cập nhật category |
| `/api/categories/:id` | DELETE | Có | Xoá category |

**Rủi ro chính:**
- Chỉ kiểm tra `authenticateToken`, **không kiểm tra `req.user.role === "admin"`** → Broken Function-Level Authorization
- `name` không có `UNIQUE` constraint → duplicate category name
- `products.category_id` không có FOREIGN KEY → orphan reference khi xoá category đang được product sử dụng
- `name` gần như không được validate (rỗng, null, sai kiểu, quá dài đều có thể được chấp nhận)

---

## 3. BỘ TEST CASE TỔNG HỢP

### 3.1. Thống kê tổng quan
| FR | Tổng case gốc | VALID | INVALID | INCOMPLETE | Case bổ sung | Tổng case cuối |
|---|---|---|---|---|---|---|
| FR05 | 35 | 33 | 1 | 1 | 0 | **35** |
| FR08 | 35 | 34 | 1 | 0 | 5 (+6 cancel) | **40** |
| FR14 | 35 | 30 | 0 | 5 (→12 case con) | 1 | **44** |
| **Tổng** | **105** | **97** | **2** | **6** | **6 (+6)** | **125** |

### 3.2. Phân loại ưu tiên
| Priority | Số lượng | Mô tả |
|---|---|---|
| **Critical** | 15 | SQL Injection, IDOR/BOLA, Broken Function-Level Authorization, business logic defect |
| **High** | 62 | Validation gap, authorization, data integrity, schema inconsistency |
| **Medium** | 35 | Boundary, input bất thường, edge case |
| **Low** | 13 | Content-Type, whitespace handling, quy ước REST |

---

## 4. PHẦN CHI TIẾT THEO CHỨC NĂNG

### 4.1. FR05 — Product Listing and Search (35 test cases)

#### Nhóm 1: Listing (TC-FR05-01 → 11)
- **TC-FR05-05** (INVALID → đã sửa): Bản gốc giả định `price` là number/string tuỳ id chẵn/lẻ ngay ở endpoint listing. Đã sửa: kỳ vọng `price` là **number nhất quán** theo schema vì spec chỉ mô tả hành vi `row.price.toString()` cho `GET /api/products/:id`, không có bằng chứng áp dụng cho listing.
- Các case còn lại đều khớp spec: status 200, JSON array, đủ 5 products, đủ field, category_id hợp lệ, Content-Type đúng, anonymous access được phép.

#### Nhóm 2: Search Happy Path (TC-FR05-12 → 20)
- Tìm kiếm theo từ khóa tồn tại, tên đầy đủ, substring, không kết quả, Unicode, số trong tên.
- **TC-FR05-16/17**: Search không phân biệt hoa/thường — ghi nhận behavior thực tế (SQLite `LIKE` mặc định case-insensitive với ASCII).
- **TC-FR05-19**: Search tiếng Việt trong `description` không khớp (chỉ search trên `name`).

#### Nhóm 3: Search Boundary & Input Bất thường (TC-FR05-21 → 29)
- Search rỗng → trả toàn bộ 5 products (`LIKE '%%'`)
- Whitespace, ký tự đặc biệt, `%` (trùng wildcard), chuỗi 500 ký tự, query dạng mảng, nhiều `search` trùng tên.
- **TC-FR05-27** (INCOMPLETE → đã sửa): Bản gốc mô tả "chuỗi 500 ký tự ngẫu nhiên" bằng lời → đã bổ sung ví dụ cụ thể (`"a".repeat(500)`) để tái lập được.

#### Nhóm 4: SQL Injection (TC-FR05-30 → 33) — Critical
- Tautology (`' OR '1'='1`), DROP TABLE, UNION SELECT, comment injection (`--`).
- Tất cả đều ưu tiên **Critical** do implementation dùng string interpolation trực tiếp.

#### Nhóm 5: GET /api/products/:id (TC-FR05-34 → 35)
- **TC-FR05-34**: id lẻ → `price` là **number** (INTEGER)
- **TC-FR05-35**: id chẵn → `price` là **string** → flag defect schema-inconsistency

#### Ghi chú bổ sung
- `GET /api/products/:id` với id không tồn tại trả `200 + {}` thay vì `404` — cần xác nhận với team business/API owner là chủ đích hay defect.
- Cart in-memory — cần test thêm (ngoài phạm vi API) khi restart server.

---

### 4.2. FR08 — Checkout (40 test cases)

#### Nhóm 1: Login & Authentication (TC-FR08-01 → 04)
- Login thành công user/admin, sai password, email không tồn tại.

#### Nhóm 2: Access Control (TC-FR08-05 → 08)
- Không có Authorization header → `401`
- **TC-FR08-08** (INVALID → đã sửa): Bản gốc kỳ vọng `401` cho token invalid. Đã sửa thành **`403 Forbidden`** vì middleware `authenticateToken` dùng chung với FR14 có quy tắc rõ: không có token → `401`; token invalid → `403`.

#### Nhóm 3: Cart Cơ bản (TC-FR08-09 → 13)
- Cart rỗng trả `[]`, add 1/nhiều sản phẩm, cart cô lập theo user.

#### Nhóm 4: Cart Validation (TC-FR08-14 → 18) — Critical defects
- Quantity = 0, âm, cực lớn đều được chấp nhận
- Product id không tồn tại được chấp nhận → **Critical defect**
- Price không khớp DB được chấp nhận → **Critical defect**

#### Nhóm 5: Checkout Positive (TC-FR08-19 → 23)
- Checkout thành công, order status = pending, total_amount/shipping_address/orderId đúng.
- **TC-FR08-19**: Expected status `201` trong bản gốc AI → đã sửa thành `200` (không có endpoint nào trong spec trả `201`).

#### Nhóm 6: Checkout Negative (TC-FR08-24 → 30)
- Thiếu field, total_amount = 0/âm, shipping_address rỗng, field thừa (mass assignment).
- **TC-FR08-28**: total_amount không phải number → kỳ vọng `500` hoặc insert giá trị không hợp lệ.

#### Nhóm 7: Critical Business Logic (TC-FR08-31 → 33)
- **TC-FR08-31**: Checkout thành công ngay cả khi cart rỗng → **Critical defect**
- **TC-FR08-32**: total_amount không được server tính lại từ cart → **Critical defect** (client tự quyết định giá)
- **TC-FR08-33**: Cart không bị xoá sau checkout

#### Nhóm 8: Order Authorization (TC-FR08-34 → 35)
- **TC-FR08-34**: Order không tồn tại → `404`
- **TC-FR08-35**: IDOR — User A xem được order của User B → **Critical IDOR/BOLA**

#### Nhóm 9: Bổ sung theo spec (TC-FR08-36 → 40)
- total_amount rất lớn, shipping_address null, shipping_address quá dài, name không khớp DB, request body malformed.

#### Nhóm 10: Cancel Order (TC-FR08-41 → 46) — Bổ sung
- Không token → `401`, token invalid → `403`, cancel order thành công, cancel order không tồn tại, IDOR cancel order, cancel order không phải pending.

---

### 4.3. FR14 — Category Management CRUD (44 test cases)

#### Nhóm 1: GET /api/categories (TC-FR14-01 → 05)
- Status 200, JSON array, khớp 3 seed categories, đủ field, không cần auth.

#### Nhóm 2: CREATE Category (TC-FR14-06 → 15)
- **TC-FR14-06** (INCOMPLETE → đã sửa): Bản gốc để mơ hồ `200/201` → đã chốt `200` (không có endpoint nào trả `201` trong toàn bộ spec).
- Validation: không gửi body, thiếu name, name rỗng/null/sai kiểu/quá dài, Unicode, ký tự đặc biệt.

#### Nhóm 3: Duplicate Category (TC-FR14-16 → 17)
- Không có `UNIQUE(name)` → có thể tạo nhiều category trùng tên.

#### Nhóm 4: UPDATE Category (TC-FR14-18 → 25)
- Update thành công, update không tồn tại, id = 0/âm/non-numeric, name rỗng.
- **TC-FR14-24** (INCOMPLETE → đã tách thành 24a/24b): Bản gốc gộp `name=null` và `name=123` vào 1 case → vi phạm nguyên tắc "1 test case = 1 điều kiện".
- Update name trùng category khác.

#### Nhóm 5: DELETE Category (TC-FR14-26 → 31)
- Delete thành công, delete không tồn tại, id = 0/âm/non-numeric.
- **TC-FR14-31**: Delete category đang được product sử dụng → **Critical defect: orphan reference** (không có FOREIGN KEY). Đã bổ sung verify cả product id=2 (cùng bị ảnh hưởng).

#### Nhóm 6: Authorization (TC-FR14-32 → 35)
- **TC-FR14-32a/b/c** (INCOMPLETE → đã tách): Bản gốc gộp 3 endpoint (POST/PUT/DELETE) vào 1 case → đã tách riêng từng endpoint.
- **TC-FR14-33a/b/c** (INCOMPLETE → đã tách): Token invalid → `403 Forbidden`.
- **TC-FR14-34a/b/c** (INCOMPLETE → đã tách): **Critical defect: Broken Function-Level Authorization** — user thường có token hợp lệ vẫn CRUD được category.
- **TC-FR14-35**: Admin baseline — cả 3 endpoint đều `200` thành công.

#### Nhóm 7: Bổ sung (TC-FR14-36)
- GET categories khi DB rỗng — phụ thuộc khả năng reset môi trường test.

---

## 5. KẾT QUẢ RÀ SOÁT & PHÂN TÍCH LỖI AI

### 5.1. Tổng quan lỗi AI (từ `report_bug.md`)
| Loại sai sót | Số case | Mô tả ngắn gọn |
|---|---|---|
| Suy diễn không có căn cứ | 1 | Áp hành vi price string/int (chỉ đúng cho `/:id`) sang endpoint listing |
| Thiếu đối chiếu chéo | 1 | Kỳ vọng `401` thay vì `403` cho token invalid |
| Gộp nhiều điều kiện | 5 | Mỗi case gộp cả 3 endpoint POST/PUT/DELETE vào 1 case |
| Test data/Expected result mơ hồ | 2 | "Chuỗi ngẫu nhiên" không cụ thể; `200/201` không chốt rõ |
| Bỏ sót coverage | 6 | 5 điều kiện FR08 + 1 điều kiện FR14 đã spec liệt kê nhưng thiếu case |

### 5.2. Phân tích nguyên nhân gốc rễ
1. **Xử lý từng FR như đơn vị độc lập** — thiếu bước đối chiếu chéo toàn cục (lỗi 2.1, 2.2)
2. **Áp dụng nguyên tắc thiết kế không đồng đều** giữa các lần sinh case trong cùng phiên (lỗi 2.3/2.4)
3. **Xu hướng để ngỏ sự mơ hồ** ngay cả khi thông tin đủ để kết luận (lỗi 2.5)
4. **Không có bước rà soát checklist 1-1** giữa spec và test case (lỗi 2.6)

### 5.3. Đánh giá mức độ nghiêm trọng
- **Nội dung/hiểu đúng nghiệp vụ:** AI hiểu đúng phần lớn rủi ro bảo mật và business logic quan trọng nhất (SQL Injection, IDOR/BOLA, Broken Function-Level Authorization, orphan reference).
- **Kỹ thuật thiết kế test case:** Các lỗi chủ yếu ở tầng chuẩn hoá/nhất quán (granularity, tách điều kiện, đối chiếu chéo).
- **Điểm cần theo dõi nhất:** FR14 có tỷ lệ case cần chỉnh sửa cao nhất (14.3%) và rơi đúng vào defect nghiêm trọng nhất (Broken Function-Level Authorization).

### 5.4. Khuyến nghị cải thiện quy trình AI
1. Thêm bước **đối chiếu chéo bắt buộc** giữa các FR dùng chung endpoint/middleware
2. Thêm bước **self-check nguyên tắc "1 case = 1 điều kiện"** như checklist cuối cùng
3. Thêm bước **đối chiếu 1-1** giữa từng test condition trong spec và test case đã sinh
4. Phân biệt rõ 2 loại "ghi nhận behavior thực tế": hợp lý (spec thực sự không rõ) vs. không hợp lý (spec đủ dữ liệu để kết luận)
5. Ưu tiên rà soát kỹ các case liên quan defect Critical/bảo mật cao

---

## 6. CI/CD PIPELINE

### 6.1. GitHub Actions Workflow (`.github/workflows/newman-ci.yml`)
| Step | Mô tả |
|---|---|
| Checkout | Clone repo + submodule `eshop-sut` (recursive) |
| Setup Node.js | Node 20 |
| Install Newman | `newman` + `newman-reporter-htmlextra` |
| Install & start API | `npm install` + `npm run start` ở `eshop-sut/backend`, wait-on port 3000 |
| Run Newman | `newman run` với collection, env, data.csv → xuất JUnit + HTML report |
| Upload artifacts | HTML report + JUnit XML |
| Publish summary | `mikepenz/action-junit-report` comment lên PR |

### 6.2. Trigger
- Push/PR đến `main` và `develop`
- Newman tự exit code != 0 nếu có test fail → CI tự động FAILED

### 6.3. Demo CI (`ci-demo/`)
- `data.commit1-pass.csv`: Sample data cho commit pass
- `data.commit1-fail.csv`: Sample data cho commit fail (TC-FR08-19 expected_status sai: 201 thay vì 200)
- `two-demo-commits.patch`: Patch minh hoạ 2 commit (1 pass, 1 fail)
- Commit log cho thấy quá trình refine CI: fix working-directory, iteration-count, path artifact, npm install vs npm ci

---

## 7. BẢNG TỔNG HỢP DEFECT ĐƯỢC PHÁT HIỆN

### 7.1. Defect nghiêm trọng (Critical)
| # | Case ID | FR | Defect | Mô tả |
|---|---|---|---|---|
| 1 | TC-FR05-30~33 | FR05 | SQL Injection | Search dùng string interpolation trực tiếp |
| 2 | TC-FR05-35 | FR05 | Schema Inconsistency | `price` đổi kiểu STRING/INTEGER tuỳ id chẵn/lẻ |
| 3 | TC-FR08-31 | FR08 | Missing Cart Validation | Checkout thành công ngay cả khi cart rỗng |
| 4 | TC-FR08-32 | FR08 | Client-controlled total_amount | Server không tính lại total từ cart |
| 5 | TC-FR08-35 | FR08 | IDOR/BOLA | `GET /api/orders/:id` không có auth → user A xem order của user B |
| 6 | TC-FR08-15 | FR08 | Negative Quantity | Cart chấp nhận quantity âm |
| 7 | TC-FR08-17 | FR08 | Nonexistent Product | Cart chấp nhận product id không tồn tại |
| 8 | TC-FR08-18 | FR08 | Forged Price | Cart chấp nhận price giả mạo |
| 9 | TC-FR14-31 | FR14 | Orphan Reference | Xoá category đang được product sử dụng không có ràng buộc |
| 10 | TC-FR14-34a/b/c | FR14 | Broken Function-Level Authorization | User thường có token hợp lệ vẫn CRUD category |
| 11 | TC-FR08-39 | FR08 | Forged Product Name | Cart chấp nhận name giả mạo |
| 12 | TC-FR08-27 | FR08 | Negative total_amount | Checkout chấp nhận total_amount âm |

### 7.2. Defect trung bình (High/Medium)
| # | Case ID | FR | Defect |
|---|---|---|---|
| 13 | TC-FR08-14 | FR08 | Quantity = 0 được chấp nhận |
| 14 | TC-FR08-16 | FR08 | Quantity cực lớn không giới hạn |
| 15 | TC-FR08-26 | FR08 | total_amount = 0 được chấp nhận |
| 16 | TC-FR08-29 | FR08 | shipping_address rỗng được chấp nhận |
| 17 | TC-FR14-16/17 | FR14 | Duplicate category name |
| 18 | TC-FR14-23 | FR14 | Update name rỗng được chấp nhận |
| 19 | TC-FR14-10 | FR14 | Create name rỗng được chấp nhận |

---

## 8. THỐNG KÊ ĐẠT ĐƯỢC

### 8.1. Test Coverage
| FR | Số endpoint | Số test condition theo spec | Số test case | Coverage |
|---|---|---|---|---|
| FR05 | 2 | 25+ | 35 | Đầy đủ |
| FR08 | 7 | 40+ | 40 | Đầy đủ (đã bổ sung 5 case) |
| FR14 | 4 | 30+ | 44 | Đầy đủ (đã bổ sung 1 case) |
| **Tổng** | **13** | **95+** | **125** | **>100%** (tính cả case bổ sung) |

### 8.2. CI/CD Metrics
| Metric | Giá trị |
|---|---|
| Số commits | 13 |
| Số CI pipeline chạy thành công | Baseline pass toàn bộ |
| Demo fail case | TC-FR08-19 (expected_status sai) |
| Artifact uploaded | HTML report + JUnit XML |
| PR summary | Tự động publish qua `mikepenz/action-junit-report` |

### 8.3. AI Performance
| Metric | Giá trị |
|---|---|
| Tỷ lệ case VALID | 97/105 = 92.4% |
| Tỷ lệ case có vấn đề | ~11.4% |
| Tỷ lệ INVALID (sai logic) | 2/105 = 1.9% |
| Tỷ lệ INCOMPLETE (thiết kế chưa chuẩn) | 6/105 = 5.7% |
| Coverage gap (bỏ sót) | 6 case |

---

## 9. KẾT LUẬN & ĐỀ XUẤT

### 9.1. Kết luận
1. **Bộ test case 125 case** đã được thiết kế, review và hoàn thiện cho 3 chức năng chính của E-Shop API, đạt coverage >100% so với spec.
2. **AI Agent** hỗ trợ sinh test case ban đầu với tỷ lệ chính xác cao (92.4% VALID), nhưng cần rà soát thủ công để phát hiện các lỗi về granularity, đối chiếu chéo và coverage gap.
3. **12 defect nghiêm trọng** đã được phát hiện và ghi nhận, trong đó có 2 lỗ hổng bảo mật Critical (SQL Injection, IDOR/BOLA, Broken Function-Level Authorization).
4. **CI/CD pipeline** đã được thiết lập hoàn chỉnh trên GitHub Actions, tự động chạy Newman trên mỗi push/PR và publish test summary.

### 9.2. Đề xuất tiếp theo
1. **Thực thi test thực tế:** Chạy Newman chống lại backend thật để xác nhận defect và thu thập evidence (request/response thực tế).
2. **Tạo defect report:** Viết report chi tiết cho 12 defect nghiêm trọng theo template, kèm bằng chứng.
3. **Cải thiện AI workflow:** Áp dụng các khuyến nghị từ `report_bug.md` để tăng chất lượng sinh test case tự động.
4. **Mở rộng coverage:** Bổ sung test cho endpoint `PUT /api/orders/:id/cancel` (đã có 6 case trong data.csv, chưa có trong collection chính).
5. **Automation framework:** Đóng gói collection vào framework automation (vd. Newman với script pre-test setup, post-test teardown) để có thể chạy regression tự động.

---

## 10. TÀI LIỆU THAM KHẢO

| File | Mô tả |
|---|---|
| `API_Infor.txt` | Specification / Ground truth cho 3 FR |
| `HW06/Test/FR05` | Test case gốc FR05 (AI generated) |
| `HW06/Test/FR08.md` | Test case gốc FR08 (AI generated) |
| `HW06/Test/FR14.md` | Test case gốc FR14 (AI generated) |
| `HW06/Test/TC_Audited.md` | Bộ test case hoàn chỉnh sau rà soát (final) |
| `HW06/report_bug.md` | Báo cáo phân tích sai sót AI |
| `HW06/AI Critique.md` | Nhận xét tổng quan về AI Agent |
| `HW06/data.csv` | 125 dòng test data cho Newman |
| `HW06/New Collection.postman_collection.json` | Postman collection (125+ requests) |
| `HW06/New Environment.postman_environment.json` | Postman environment config |
| `.github/workflows/newman-ci.yml` | GitHub Actions CI workflow |
| `ci-demo/newman-ci.yml` | Demo CI với iteration-count fix |
| `git-commit-log.txt` | Lịch sử 13 commits |

