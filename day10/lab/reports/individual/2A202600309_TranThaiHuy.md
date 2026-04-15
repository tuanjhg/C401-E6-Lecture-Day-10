# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** TranThaiHuy  
**Mã học viên:** 2A202600309  
**Vai trò:** Evaluation Lead (Evidence & Retrieval Eval)  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào?

Tôi phụ trách mảng **evaluation/evidence** để chứng minh pipeline Day 10 “feed đúng data” cho tầng retrieval/agent. Cụ thể, tôi chịu trách nhiệm chạy và lưu các artifacts cho 2 kịch bản:

- **Pipeline chuẩn (after / good run)**: chạy `etl_pipeline.py run` để clean + expectation pass + embed Chroma, sau đó chạy `eval_retrieval.py` để lấy kết quả retrieval theo bộ golden questions.
- **Inject corruption (before / inject-bad)**: chạy pipeline với `--no-refund-fix --skip-validate` để cố ý tạo tình huống “dirty index”, rồi chạy `eval_retrieval.py` để thấy khác biệt định lượng trước/sau.

Tôi cũng tổng hợp evidence vào báo cáo nhóm/quality report (run_id + đường dẫn artifacts + số liệu trước/sau) và hỗ trợ xử lý xung đột/logic liên quan đến scenario inject để kết quả eval phản ánh đúng ý đồ Sprint 3.

**Bằng chứng artifacts (run_id):**
- `run_id=good`: `artifacts/logs/run_good.log`, `artifacts/manifests/manifest_good.json`
- `run_id=inject-bad`: `artifacts/logs/run_inject-bad.log`, `artifacts/manifests/manifest_inject-bad.json`

---

## 2. Một quyết định kỹ thuật

Quyết định quan trọng của tôi là **tách rõ 2 mode đánh giá** để vừa có “index canonical” cho Sprint 2, vừa có “bad scenario” cho Sprint 3:

- **Normal mode (pipeline chuẩn)** phải **auto-fix** lỗi “refund window 14→7 ngày” để expectation pass và embed được index sạch (đảm bảo hệ tầng trên không còn đọc policy stale).
- **Inject mode** phải **giữ 14 ngày** (không fix) và cho phép bypass halt (`--skip-validate`) để cố ý embed “dirty index”. Nhờ vậy, eval có thể đo `hits_forbidden=yes` cho câu `q_refund_window` và tạo được before/after evidence.

Trade-off: logic phức tạp hơn so với chỉ “halt” ngay, nhưng phù hợp mục tiêu lab: vừa vận hành được pipeline chuẩn, vừa chứng minh tác động data quality lên retrieval.

---

## 3. Một lỗi/anomaly đã xử lý

**Triệu chứng:** Pipeline chuẩn bị **HALT** vì phát hiện stale refund window (“14 ngày”) trong raw export, khiến Sprint 2 không đi tới embed/eval được.

**Dấu hiệu trong log:** `PIPELINE_HALT: stale refund window (14 ngày) detected...` (pipeline dừng trước embed).

**Cách xử lý:** Điều chỉnh logic cleaning/refund để normal run tự fix về “7 ngày” (expectation E3 pass), còn inject-bad thì giữ “14 ngày” để E3 fail (nhưng vẫn embed khi `--skip-validate`). Sau khi xử lý, pipeline `run_id=good` có:
- `expectation[refund_no_stale_14d_window] OK (halt) :: violations=0`
- `embed_upsert count=7 collection=day10_kb`

---

## 4. Bằng chứng trước / sau

**Before (inject-bad):** `artifacts/logs/run_inject-bad.log`
- `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1`
- `WARN: expectation failed but --skip-validate -> tiếp tục embed`

**After (good):** `artifacts/logs/run_good.log`
- `expectation[refund_no_stale_14d_window] OK (halt) :: violations=0`

**Retrieval eval (điền số liệu từ CSV):**
- File after (good): `artifacts/eval/before_after_eval.csv` → `q_refund_window`: `hits_forbidden=no`
- File before (inject-bad): `artifacts/eval/after_inject_bad.csv` → `q_refund_window`: `hits_forbidden=yes`

> Nếu thiếu 2 file CSV trên do merge/cleanup, chỉ cần chạy lại `python eval_retrieval.py --out ...` cho từng scenario để regenerate và paste 2 dòng tương ứng vào report.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ (1) mở rộng eval theo “slice” nhiều câu hơn (≥5) để giảm phụ thuộc keyword; (2) thêm một bước “eval snapshot” ghi `run_id + commit_sha + collection_count` để trace evidence chắc chắn; và (3) chuẩn hoá output CSV có cột `scenario` để gộp before/after vào một file phục vụ đọc nhanh trong group report.

