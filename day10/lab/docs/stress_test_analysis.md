# Phân tích Kịch bản Stress Test - Lab Day 10
**Người thực hiện:** Quang (Transformation Engineer)
**Mục tiêu:** Kiểm tra độ bền của Pipeline và khả năng tự chữa lành dữ liệu trước khi nạp vào Vector DB.

---

## 1. Chi tiết kịch bản & Kỳ vọng xử lý

| ID | Vấn đề (Issue) | Kỳ vọng xử lý (Expected Handling) | Metric Impact |
|:---|:---|:---|:---|
| **st_01** | **Logic sai:** Ghi 14 ngày (v4 chuẩn là 7 ngày). | Rule `fix_stale_refund` phải đổi 14 -> 7 ngày. | `cleaned_refund_window_fixed` |
| **st_02** | **Duplicate chéo:** Nội dung Refund nhưng gán nhầm Doc_ID là Helpdesk. | Rule `dedupe_chunk_text` phải phát hiện nội dung này đã tồn tại và loại bỏ dòng này. | `dropped_duplicate_chunk_text` |
| **st_03** | **Encoding bẩn:** Chứa mã Hex `\x00\xff...` | Rule `clean_trash_chars` phải xóa bỏ các ký tự không in được, giữ lại text sạch. | `cleaned_records` |
| **st_04** | **Dữ liệu cũ (Stale):** Chính sách năm 2024. | Rule `stale_hr_policy` phải đẩy dòng này vào Quarantine vì contract yêu cầu >= 2026. | `quarantine_stale_hr_policy` |
| **st_05** | **Dữ liệu rỗng:** `chunk_text` để trống. | Phải đẩy vào Quarantine ngay lập tức. | `quarantine_empty_chunk_text` |
| **st_06** | **Doc_ID lạ:** `unknown_system_log`. | Phải đẩy vào Quarantine vì không nằm trong `allowed_doc_ids`. | `quarantine_unknown_doc_id` |
| **st_07** | **Sai định dạng ngày:** `15/04/2026`. | Rule `normalize_effective_date` phải convert về đúng ISO `2026-04-15`. | `cleaned_records` |
| **st_08** | **Duplicate tuyệt đối:** Trùng y hệt dòng `st_01`. | Rule `dropped_duplicate_chunk_id` phải loại bỏ để đảm bảo Idempotency. | `dropped_duplicate_chunk_id` |

---

## 2. Ảnh hưởng dự kiến lên Agent (Before vs. After)

### Kịch bản: Câu hỏi về Hoàn tiền (Refund Window)
*   **Trước khi fix (Before):** Huy (Eval) sẽ thấy Agent trả lời "14 ngày" hoặc tệ hơn là trả lời lẫn lộn giữa 7 và 14 ngày (Hallucination). Cột `hits_forbidden` sẽ báo **YES**.
*   **Sau khi fix (After):** Pipeline của Quang & Tuấn sẽ sửa 14 thành 7. Agent chỉ nhận được context 7 ngày duy nhất. Cột `hits_forbidden` phải báo **NO**.

### Kịch bản: Câu hỏi về Nghỉ phép (HR Leave)
*   **Trước khi fix (Before):** Agent có thể lấy nhầm thông tin 10 ngày (của năm 2024) thay vì 12 ngày (của năm 2026).
*   **Sau khi fix (After):** Dòng 2024 bị Quang "nhốt" vào Quarantine. Agent chỉ thấy dữ liệu 2026. Cột `top1_doc_matches` phải báo **YES**.

---

## 3. Kết luận cho Observability
Kết quả chạy file Stress Test này sẽ được dùng để điền vào bảng `metric_impact` trong báo cáo nhóm. 
*   Nếu mọi kỳ vọng trên đều khớp với Log thực tế -> Hệ thống đạt chuẩn **Distinction**.
*   Nếu có bất kỳ dòng bẩn nào lọt vào Vector DB -> Cần cập nhật lại `cleaning_rules.py`.
