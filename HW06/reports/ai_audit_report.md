# AI Audit Report - E-Shop API Test Case Generation

**Dự án:** HW06 - API Automated Testing (E-Shop)  
**Người thực hiện audit:** Geedie (Huynh M. Doan)  
**Ngày:** 2026-08-21  
**Phạm vi:** FR05, FR08, FR14 - đối chiếu `API_Infor.txt`  

---

## 1. Tóm tắt prompting

AI Agent (`api-qa-testing-agent`) được dùng để sinh 105 test case ban đầu cho 3 FR. Sau rà soát thủ công đối chiếu 1-1 với `API_Infor.txt`:

- **97/105 VALID** (92,4%)
- **2/105 INVALID** (1,9%) - sai logic/expected result
- **6/105 INCOMPLETE** (5,7%) - thiết kế chưa chuẩn (gộp điều kiện, mơ hồ)
- **6 case bổ sung** do coverage gap so với spec

→ **Tỷ lệ cần chỉnh sửa: ~11,4%**

---

## 2. Phân loại lỗi chi tiết

| # | Loại lỗi | Số case | Ví dụ cụ thể |
|---|---|---|---|
| 1 | Suy diễn không căn cứ | 1 | TC-FR05-05: áp dụng hành vi `price` string/int (chỉ đúng cho `GET /api/products/:id`) sang endpoint listing |
| 2 | Thiếu đối chiếu chéo | 1 | TC-FR08-08: kỳ vọng `401` thay vì `403` cho token invalid (không đối chiếu với FR14 mục 8) |
| 3 | Gộp nhiều điều kiện | 5 | FR14-32/33/34: gộp 3 endpoint POST/PUT/DELETE vào 1 case |
| 4 | Expected result mơ hồ | 2 | TC-FR05-27: "chuỗi 500 ký tự ngẫu nhiên" không cụ thể; TC-FR14-06: `200/201` không chốt rõ |
| 5 | Bỏ sót coverage | 6 | FR08 thiếu 5 case (total_amount rất lớn, shipping_address null/quá dài, name giả mạo, body malformed); FR14 thiếu 1 case (DB rỗng) |

---

## 3. Phân tích nguyên nhân gốc rễ

1. **Xử lý từng FR như đơn vị độc lập** - thiếu bước đối chiếu chéo toàn cục (lỗi #2).
2. **Áp dụng nguyên tắc thiết kế không đồng đều** giữa các lần sinh case trong cùng phiên (lỗi #3).
3. **Xu hướng để ngỏ sự mơ hồ** ngay cả khi thông tin đủ để kết luận (lỗi #4).
4. **Không có bước rà soát checklist 1-1** giữa spec và test case (lỗi #5).

---

## 4. Đánh giá mức độ nghiêm trọng

- **Hiểu đúng nghiệp vụ:** AI nhận diện đúng phần lớn rủi ro bảo mật và business logic quan trọng nhất (SQL Injection, IDOR/BOLA, Broken Function-Level Authorization, orphan reference).
- **Thiết kế test case:** Lỗi chủ yếu ở tầng chuẩn hoá/nhất quán (granularity, tách điều kiện, đối chiếu chéo).
- **Điểm rủi ro cao nhất:** FR14 có tỷ lệ chỉnh sửa cao nhất (14,3%) và rơi đúng vào defect nghiêm trọng nhất (Broken Function-Level Authorization).

---

## 5. Khuyến nghị cải thiện quy trình AI

1. Thêm bước **đối chiếu chéo bắt buộc** giữa các FR dùng chung endpoint/middleware.
2. Thêm bước **self-check nguyên tắc "1 case = 1 điều kiện"** như checklist cuối cùng.
3. Thêm bước **đối chiếu 1-1** giữa từng test condition trong spec và test case đã sinh.
4. Phân biệt rõ 2 loại "ghi nhận behavior thực tế": hợp lý (spec thực sự không rõ) vs. không hợp lý (spec đủ dữ liệu để kết luận).
5. Ưu tiên rà soát kỹ các case liên quan defect Critical/bảo mật cao.

---

## 6. Kết luận

AI hiệu quả cao (seasons) - hỗ trợ sinh nội dung ban đầu nhanh và phát hiện rủi ro tốt - nhưng **không thể thay thế** bước rà soát chuyên môn bài bản. Mô hình tối ưu là **"coworking"**: AI sinh thô → human kiểm soát ngữ cảnh, đối chiếu chéo, hiệu chỉnh → bộ test case cuối cùng đạt chuẩn.

---

