# Báo Cáo: Phân Tích Sai Sót Của AI Trong Quá Trình Sinh Test Case (FR05 / FR08 / FR14)

**Đối tượng đánh giá:** AI Agent (skill `api-qa-testing-agent`) — quá trình sinh test case tự động cho 3 chức năng của E-Shop API
**Nguồn đối chiếu:** `TC_Audited.md` — kết quả rà soát thủ công 3 file gốc (`FR05`, `FR08.md`, `FR14.md`) so với `API_Infor.txt` (spec gốc/ground truth)
**Người thực hiện rà soát:** Geedie
**Mục đích báo cáo:** Ghi nhận lại các dạng sai sót AI đã mắc phải khi sinh test case, phân tích nguyên nhân gốc rễ, và rút ra khuyến nghị cải thiện quy trình cho các lần dùng AI Agent tiếp theo.

---

## 1. Tóm tắt số liệu

| FR | Tổng case gốc | VALID | INVALID | INCOMPLETE | Case bổ sung (bị bỏ sót) | Tỷ lệ có vấn đề* |
|---|---|---|---|---|---|---|
| FR05 | 35 | 33 | 1 | 1 | 0 | 5.7% |
| FR08 | 35 | 34 | 1 | 0 | 5 | 2.9% (+5 thiếu) |
| FR14 | 35 | 30 | 0 | 5 (→12 case con) | 1 | 14.3% |
| **Tổng** | **105** | **97** | **2** | **6** | **6** | **~11.4%** |

*Tỷ lệ có vấn đề = (INVALID + INCOMPLETE) / Tổng case gốc, chưa tính case bị bỏ sót hoàn toàn.

**Nhận xét chung:** Tỷ lệ case sai lệch hoàn toàn (INVALID) rất thấp (2/105 ≈ 1.9%) — AI hiểu đúng phần lớn spec. Tuy nhiên tỷ lệ case **thiết kế chưa đạt chuẩn** (INCOMPLETE — gộp nhiều điều kiện, mơ hồ) tập trung gần như toàn bộ ở FR14 (5/5), cho thấy **chất lượng không ổn định giữa các lần chạy** dù cùng một skill, cùng một nguyên tắc thiết kế.

---

## 2. Phân loại các dạng sai sót

### 2.1. Suy diễn hành vi hệ thống không có căn cứ từ spec (Overgeneralization)

- **Case liên quan:** TC-FR05-05
- **Mô tả:** AI giả định `price` có thể là `number` **hoặc** `string` tuỳ `id` chẵn/lẻ ngay ở endpoint listing (`GET /api/products`). Nhưng spec chỉ mô tả hành vi ép kiểu này (`row.price.toString()`) cho endpoint chi tiết (`GET /api/products/:id`), không có bằng chứng áp dụng cho listing.
- **Bản chất lỗi:** AI đã **mang một hành vi đã xác nhận ở context A sang áp dụng cho context B** mà không kiểm tra spec có thực sự mô tả behavior đó ở context B hay không. Đây là lỗi suy diễn thiếu căn cứ (không phải bịa hoàn toàn, mà là "lây" logic đúng chỗ này sang chỗ khác chưa được xác nhận).
- **Rủi ro nếu không phát hiện:** Một tester thực thi theo case này có thể ghi nhận sai kết luận (coi hành vi bình thường là bug hoặc ngược lại) vì expected result không phản ánh đúng phạm vi áp dụng thực tế của behavior.

### 2.2. Không đối chiếu chéo giữa các FR dùng chung cơ chế xử lý

- **Case liên quan:** TC-FR08-08
- **Mô tả:** AI kỳ vọng `401` khi gọi API với token malformed/invalid signature. Nhưng theo spec (mục FR14-8), middleware `authenticateToken` dùng chung cho cả FR08 và FR14 có quy tắc rõ ràng: **không có token → 401**, **token invalid → 403**. AI khi sinh case cho FR08 đã không tra cứu/nhất quán với quy tắc đã có ở phần spec liên quan đến FR14.
- **Bản chất lỗi:** Đây là lỗi **thiếu liên kết ngữ cảnh giữa các phần của cùng một spec** khi AI xử lý từng FR như một đơn vị độc lập, thay vì nhận diện các endpoint dùng chung middleware/logic xuyên suốt toàn bộ hệ thống.
- **Rủi ro nếu không phát hiện:** Expected result sai sẽ khiến kết quả test thực tế (`403`) bị đánh giá nhầm là "Fail" dù hệ thống hoạt động đúng thiết kế — gây báo cáo defect giả (false positive).

### 2.3. Vi phạm nguyên tắc "1 test case = 1 điều kiện"

- **Case liên quan:**
  - TC-FR14-24 (gộp `name=null` và `name=123` vào 1 case) → tách thành 24a/24b
  - TC-FR14-32, 33, 34 (mỗi case gộp cả 3 endpoint POST/PUT/DELETE) → tách thành mỗi case 3 case con (a/b/c) — tổng cộng đây là nhóm chiếm phần lớn số case bị đánh giá INCOMPLETE (5/6 toàn bộ 3 FR).
- **Bản chất lỗi:** AI gộp nhiều điều kiện kiểm thử độc lập vào cùng một test case, vi phạm nguyên tắc thiết kế test case cơ bản mà chính skill đang sử dụng (`references/test-design.md`) đã quy định. Hệ quả là khi thực thi, nếu chỉ 1 trong nhiều điều kiện fail, không thể xác định rõ Pass/Fail của case và không trace được chính xác điều kiện nào gây lỗi.
- **Điểm đáng chú ý:** Cùng một loại case (authorization theo endpoint: không token / token invalid / thiếu quyền admin), **FR08 được AI tách đúng theo từng endpoint** (TC-FR08-05/06/07), nhưng **FR14 lại gộp chung** — cho thấy đây không phải AI "không biết" nguyên tắc, mà là **áp dụng không nhất quán giữa các lần sinh case**, kể cả trong cùng một phiên làm việc.

### 2.4. Thiếu nhất quán về mức độ chi tiết (granularity) giữa các FR

- Hệ quả trực tiếp của mục 2.3: cùng một loại test condition (auth theo endpoint ghi dữ liệu) nhưng FR08 và FR14 được thiết kế ở hai mức độ chi tiết khác nhau, dù được sinh bởi cùng một skill trong cùng một tác vụ tổng thể.
- Đây là dạng lỗi **về tính nhất quán trong toàn bộ bộ test case**, chỉ phát hiện được khi review chéo giữa các FR với nhau — không thể thấy nếu chỉ đọc riêng lẻ từng file.

### 2.5. Expected Result / Test Data mơ hồ, không xác định rõ ràng

- **Case liên quan:**
  - TC-FR05-27: mô tả test data bằng lời ("chuỗi 500 ký tự ngẫu nhiên") thay vì giá trị cụ thể có thể tái lập khi automation hoá → đã bổ sung ví dụ cụ thể (`"a".repeat(500)`).
  - TC-FR14-06: để mơ hồ expected status `200/201` thay vì chốt một giá trị rõ ràng, trong khi đối chiếu toàn bộ spec cho thấy không có endpoint ghi dữ liệu nào trả `201` — AI có đủ dữ liệu trong spec để kết luận dứt khoát nhưng đã không làm.
- **Bản chất lỗi:** AI để lại sự mơ hồ ở những chỗ **đáng lẽ có thể chốt được** dựa trên thông tin đã có sẵn trong spec (khác với các case "ghi nhận behavior thực tế" hợp lý vì spec thực sự không mô tả rõ). Sự mơ hồ không cần thiết này làm giảm khả năng automation hoá và gây khó khăn khi xác định tiêu chí Pass/Fail.

### 2.6. Bỏ sót test condition đã được spec liệt kê rõ (Coverage Gap)

- **Case bổ sung:** TC-FR08-36→40 (total_amount rất lớn, shipping_address null, shipping_address quá dài, cart name không khớp DB, request body malformed), TC-FR14-36 (GET categories khi DB rỗng).
- **Bản chất lỗi:** Đây là những test condition **đã được liệt kê tường minh trong spec** (mục 11, 12.G của FR08; mục 4 của FR14) nhưng AI không sinh case tương ứng. Khác với các lỗi ở trên (sai logic), đây là lỗi **thiếu sót coverage thuần tuý** — cho thấy AI chưa đối chiếu đầy đủ 100% các điều kiện được liệt kê trong spec khi sinh case, dù các điều kiện tương tự/liền kề đã được cover (ví dụ: đã test `price` không khớp DB nhưng bỏ sót `name` không khớp DB dù spec liệt kê cả hai ngang hàng).

---

## 3. Bảng tổng hợp chi tiết từng lỗi

| # | Case ID | Loại sai sót | Mô tả ngắn gọn | Mức độ ảnh hưởng nếu không phát hiện |
|---|---|---|---|---|
| 1 | TC-FR05-05 | Suy diễn không có căn cứ | Áp hành vi price string/int (chỉ đúng cho `/:id`) sang endpoint listing | Trung bình — có thể gây kết luận sai khi review kết quả test |
| 2 | TC-FR08-08 | Thiếu đối chiếu chéo | Kỳ vọng `401` thay vì `403` cho token invalid | Cao — gây false positive defect report |
| 3 | TC-FR14-24 | Gộp nhiều điều kiện | `name=null` và `name=123` chung 1 case | Trung bình — khó trace khi 1/2 điều kiện fail |
| 4 | TC-FR14-32 | Gộp nhiều điều kiện | 3 endpoint (POST/PUT/DELETE) chung 1 case kiểm tra "không token" | Trung bình-Cao — đây là nhóm liên quan bảo mật |
| 5 | TC-FR14-33 | Gộp nhiều điều kiện | 3 endpoint chung 1 case kiểm tra "token invalid" | Trung bình-Cao |
| 6 | TC-FR14-34 | Gộp nhiều điều kiện | 3 endpoint chung 1 case kiểm tra **Critical defect** (Broken Function-Level Authorization) | **Cao** — đây là defect nghiêm trọng nhất FR14, gộp case làm giảm độ chính xác khi report |
| 7 | TC-FR05-27 | Test data mơ hồ | "Chuỗi ngẫu nhiên" không cụ thể, không tái lập được | Thấp-Trung bình — cản trở automation |
| 8 | TC-FR14-06 | Expected result mơ hồ | `200/201` không chốt rõ dù spec đủ dữ liệu để kết luận | Thấp — gây tranh cãi khi xác định Pass/Fail |
| 9 | TC-FR08-36→40 | Bỏ sót coverage | 5 điều kiện spec liệt kê rõ nhưng chưa có case | Trung bình — lỗ hổng coverage với case liên quan bảo mật/business logic |
| 10 | TC-FR14-36 | Bỏ sót coverage | Case "DB rỗng" theo spec mục 4 chưa có | Thấp |

---

## 4. Phân tích nguyên nhân gốc rễ

1. **Xử lý từng FR như đơn vị độc lập, thiếu bước đối chiếu chéo toàn cục.** Lỗi 2.1 và 2.2 đều xuất phát từ việc AI không kiểm tra xem một hành vi/quy tắc đã xác nhận ở phần này của spec có áp dụng nhất quán cho các phần khác hay không — dù cùng một hệ thống, cùng một middleware.
2. **Áp dụng nguyên tắc thiết kế không đồng đều giữa các lần sinh case trong cùng một phiên.** Lỗi 2.3/2.4 cho thấy vấn đề không phải do AI "không biết" nguyên tắc "1 case = 1 điều kiện" (vì FR08 áp dụng đúng), mà do **thiếu một bước tự-kiểm-tra nhất quán (self-consistency check)** trước khi hoàn thiện toàn bộ bộ case.
3. **Xu hướng để ngỏ sự mơ hồ ngay cả khi thông tin đủ để kết luận.** Lỗi 2.5 phản ánh việc AI đôi khi không tận dụng hết thông tin đã có trong spec để đưa ra kết luận dứt khoát, có thể do thiên hướng "an toàn" (tránh khẳng định) áp dụng sai chỗ — sự mơ hồ chỉ nên giữ lại khi spec **thực sự** không mô tả rõ, không phải mặc định cho mọi trường hợp không chắc chắn.
4. **Không có bước rà soát checklist đầy đủ 1-1 giữa từng dòng spec và test case tương ứng.** Lỗi 2.6 (bỏ sót 6 case) cho thấy quy trình sinh case chưa bao gồm bước đối chiếu tường minh "mỗi điều kiện trong spec đã có ít nhất 1 test case tương ứng chưa" — dẫn đến bỏ sót dù các điều kiện liền kề, tương tự đã được cover đầy đủ.

---

## 5. Đánh giá mức độ nghiêm trọng tổng thể

- **Về mặt nội dung/hiểu đúng nghiệp vụ:** AI hiểu đúng phần lớn các rủi ro bảo mật và business logic quan trọng nhất của cả 3 FR (SQL Injection, IDOR/BOLA, Broken Function-Level Authorization, thiếu validation, orphan reference) — đây là phần khó nhất và AI làm tốt. Không có case nào bị đánh giá sai ở tầng nhận diện rủi ro.
- **Về mặt kỹ thuật thiết kế test case:** Các lỗi phát hiện được chủ yếu nằm ở tầng **chuẩn hoá/nhất quán** (granularity, tách điều kiện, đối chiếu chéo) chứ không phải ở tầng hiểu sai spec hoàn toàn — ngoại trừ TC-FR08-08 (lỗi có khả năng gây false positive khi thực thi thật) nên cần đặc biệt lưu ý.
- **Điểm cần theo dõi nhất:** FR14 có tỷ lệ case cần chỉnh sửa cao nhất (14.3%) và tập trung ở đúng nhóm case liên quan đến **defect nghiêm trọng nhất của cả bộ** (Broken Function-Level Authorization — TC-FR14-34) — nghĩa là lỗi thiết kế case lại rơi đúng vào khu vực rủi ro cao nhất, cần ưu tiên rà soát kỹ khi AI sinh case cho các FR có rủi ro bảo mật tương tự trong tương lai.

---

## 6. Đề xuất cải thiện quy trình cho các lần dùng AI Agent tiếp theo

1. **Thêm bước đối chiếu chéo bắt buộc** giữa các FR dùng chung endpoint/middleware/logic trước khi chốt bộ test case cuối cùng (đặc biệt với các cơ chế auth dùng chung).
2. **Thêm bước self-check nguyên tắc "1 case = 1 điều kiện"** như một checklist cuối cùng, áp dụng đồng đều cho toàn bộ case trong cùng một phiên, không chỉ áp dụng cục bộ theo từng nhóm.
3. **Thêm bước đối chiếu 1-1 giữa từng test condition liệt kê trong spec và test case đã sinh**, để phát hiện coverage gap trước khi bàn giao, thay vì phát hiện qua rà soát thủ công sau đó.
4. **Phân biệt rõ 2 loại "ghi nhận behavior thực tế":** (a) hợp lý vì spec thực sự không mô tả → giữ nguyên; (b) không hợp lý vì spec đã đủ dữ liệu để kết luận → cần chốt expected result rõ ràng thay vì để ngỏ.
5. **Ưu tiên rà soát kỹ hơn ở các case liên quan defect Critical/bảo mật cao** (IDOR, Broken Function-Level Authorization, SQL Injection) — vì đây là khu vực lỗi thiết kế case gây ảnh hưởng lớn nhất nếu không được phát hiện.

---

## 7. Kết luận

Trong tổng số 105 test case AI sinh ra cho FR05/FR08/FR14, có **2 case sai logic (INVALID)**, **6 case cần tách/chuẩn hoá lại (INCOMPLETE)**, và **6 test condition bị bỏ sót hoàn toàn** so với spec — tổng cộng khoảng 11.4% case gốc cần chỉnh sửa, chưa kể phần bổ sung. Các lỗi này **không xuất phát từ việc AI hiểu sai bản chất rủi ro nghiệp vụ/bảo mật** (phần khó nhất AI làm tốt), mà chủ yếu đến từ: (1) thiếu đối chiếu chéo giữa các phần liên quan của spec, (2) áp dụng không nhất quán nguyên tắc thiết kế test case trong cùng một phiên, và (3) chưa rà soát coverage đầy đủ 1-1 với spec. Đây là những điểm có thể cải thiện bằng cách bổ sung các bước self-check/cross-check tường minh vào quy trình sử dụng AI Agent, thay vì chỉ dựa vào một lần sinh case duy nhất.