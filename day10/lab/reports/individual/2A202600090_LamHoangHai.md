# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lâm Hoàng Hải
**Vai trò:** Quality & Observability Owner
**Ngày nộp:15/04/2026**
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- quality/expectations.py

**Kết nối với thành viên khác:**

Tôi phối hợp chặt chẽ với Long để thống nhất các luật kiểm tra, đồng thời trao đổi với Dũng (Lead) để tích hợp expectation vào luồng ETL tổng thể. Ngoài ra, tôi làm việc với nhóm Transformation (Tuấn, Quang) để xác nhận các trường hợp dữ liệu biên và cập nhật expectation phù hợp với cleaning rules mới. Tôi cũng tham gia review log, kiểm thử stress test và đóng góp vào báo cáo chất lượng cuối cùng.

---

**Bằng chứng (commit / comment trong code):**

214fe86785516e4e18a83be3870356e2f456fdd1
efcebec017938a9c6c126390d3eaec9b816cb624

---

---

## 2. Một quyết định kỹ thuật (100–150 từ)

> VD: chọn halt vs warn, chiến lược idempotency, cách đo freshness, format quarantine.

---

Một quyết định kỹ thuật quan trọng tôi thực hiện là chủ động bổ sung thêm ba expectation mức `halt` vào quality suite: kiểm tra chunk_text không rỗng (`no_empty_chunk_text`), doc_id hợp lệ (`no_unknown_doc_id`), và chunk_id duy nhất (`no_duplicate_chunk_id`). Tôi nhận thấy nếu chỉ dựa vào cleaning rules thì vẫn có nguy cơ lọt dữ liệu rỗng, doc_id lạ hoặc trùng chunk_id vào downstream, gây lỗi ngầm hoặc làm sai lệch kết quả embedding. Việc bổ sung các expectation này giúp pipeline dừng ngay khi phát hiện lỗi nghiêm trọng, đảm bảo dữ liệu đầu ra luôn đạt chuẩn tối thiểu. Tôi cũng chọn mức `halt` thay vì `warn` cho các lỗi này để tăng tính an toàn.

---

---

## 3. Sự cố / anomaly

Tôi phát hiện một số bản ghi có doc_id viết hoa sai chuẩn (ví dụ: `policy_refund_V4`) hoặc chunk_id bị trùng vẫn lọt qua cleaning, dẫn đến nguy cơ dữ liệu bẩn downstream. Tôi đã bổ sung expectation kiểm tra doc_id hợp lệ (`no_unknown_doc_id`) và chunk_id duy nhất (`no_duplicate_chunk_id`) ở mức halt vào `quality/expectations.py`. Sau khi bổ sung, log expectation luôn OK với các batch dữ liệu hợp lệ, đảm bảo pipeline chỉ chạy khi dữ liệu đã sạch hoàn toàn.

---

---

## 4. Bằng chứng trước / sau

**Log expectation:**

- Trước khi bổ sung: không có log `expectation[no_unknown_doc_id]` hoặc `expectation[no_duplicate_chunk_id]`.
- Sau khi bổ sung (run_id=adv-stress):
  `expectation[no_unknown_doc_id] OK (halt) :: unknown_doc_id_count=0`
  `expectation[no_duplicate_chunk_id] OK (halt) :: duplicate_chunk_ids=[]`

---

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ bổ sung thêm expectation kiểm tra PII (số điện thoại, email) trong chunk_text ở mức halt. Điều này giúp phát hiện và loại bỏ triệt để các thông tin nhạy cảm còn sót lại sau bước cleaning, đảm bảo dữ liệu đầu ra an toàn tuyệt đối trước khi đưa vào vector DB hoặc chia sẻ cho downstream.

---
