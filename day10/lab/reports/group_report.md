# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Quality & Observability + Transformation + DB + Eval
**Thành viên:**
| Tên | Vai trò (Day 10) | Mã HV |
|-----|------------------|-------|
| Dũng | Lead / Orchestration | 2A202600176 |
| Tuấn | Transformation (Cleaning) | 2A202600086 |
| Quang | Transformation (Cleaning) | 2A202600285 |
| Hải | Quality & Observability | 2A202600090 |
| Long | Quality & Observability (Contract, Freshness, Docs) | 2A202600160 |
| Thuận | Vector DB (Embed/Prune) | 2A202600058 |
| Huy | Evaluation | 2A202600309 |

**Ngày nộp:** 2026-04-15
**Repo:** [Link Repository](https://github.com/tuanjhg/Lecture-Day-10)
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (~180 từ)

> Nguồn raw: `data/raw/policy_export_dirty.csv` — **15 records** (theo log run), nhiều lỗi cố ý (duplicate, unknown source, stale version, empty text, date format inconsistency).

**Tóm tắt luồng:**

Pipeline 5 stages: Ingest → Clean → Validate (Expectations) → Embed (ChromaDB) → Publish (Manifest + Freshness). Dữ liệu đi từ CSV bẩn, qua hơn 10 cleaning rules (7 baseline + 3 mới), và 9 expectations (7 baseline + 2 mới), rồi upsert idempotent vào ChromaDB collection `day10_kb`. Mỗi run tạo 4 artifacts liên kết bằng `run_id`: log, cleaned CSV, quarantine CSV, manifest JSON. Sơ đồ chi tiết: `docs/pipeline_architecture.md`.

**Lệnh chạy một dòng:**

```bash
# Pipeline chuẩn (sau khi đã fix lỗi logic)
python etl_pipeline.py run --run-id day10-final

# Inject corruption (before) để demo expectation fail
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate

# Freshness check
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_day10-final.json
```

---

## 2. Cleaning & expectation (~200 từ)

> Baseline có nhiều rules. Nhóm thêm **3 rule mới** và **2 expectation mới** để tăng cường độ tin cậy cho pipeline, đảm bảo dữ liệu đầu ra tuân thủ contract.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Metric | Trước (inject bad) | Sau (pipeline chuẩn) | Chứng cứ / Kịch bản Inject |
|-----------------------------------|--------|-------------------|---------------------|--------------------------------|
| `refund_window_fix` (rule) | `cleaned_refund_window_fixed` | 0 | 1 | **Kịch bản:** Chạy với `--no-refund-fix`. **Chứng cứ:** `logs/run_inject-bad.log` |
| `refund_no_stale_14d_window` (expectation) | `violations` | 1 (FAIL, skipped) | 0 (OK) | **Kịch bản:** Chạy với `--no-refund-fix`. **Chứng cứ:** `logs/run_inject-bad.log` |
| Retrieval cleanliness | `hits_forbidden` | **yes** | **no** | **Chứng cứ:** `eval/after_inject_bad.csv` so với `eval/run_final.csv`. |
| **(mới) `normalize_date_format`** | `quarantine_invalid_date_format` | 1 | 0 | **Kịch bản:** Thêm dòng có ngày `01-01-2025` (DD-MM-YYYY). Rule sẽ quarantine. |
| **(mới) `dedup_by_chunk_id`** | `dropped_duplicate_chunk_id` | 1 | 0 | **Kịch bản:** Thêm 2 dòng có cùng `chunk_id` nhưng `exported_at` khác nhau. Rule sẽ giữ lại dòng mới hơn. |
| **(mới) `handle_empty_chunk`** | `quarantine_empty_chunk_text` | 1 | 0 | **Kịch bản:** Thêm dòng có `chunk_text` rỗng. Rule sẽ quarantine. Expectation `no_empty_chunk_text` cũng sẽ FAIL. |


> **Note:** Các rule mới được thiết kế để "phòng thủ" cho các kịch bản lỗi thực tế. Dữ liệu mẫu `policy_export_dirty.csv` đã trigger một số rule (VD: `invalid_date_format` có sẵn), nhưng để chứng minh toàn diện, cần các kịch bản inject như trên.

**Ví dụ expectation fail:** `E3: refund_no_stale_14d_window` FAIL khi inject `--no-refund-fix` → chunk "14 ngày" còn trong cleaned → halt pipeline (trừ khi `--skip-validate`).

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (~200 từ)

**Kịch bản inject:**

Chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` → chunk stale "14 ngày làm việc" (từ v3 migration, row 3 CSV) KHÔNG bị fix, và được embed thẳng vào ChromaDB. Agent truy vấn "bao nhiêu ngày hoàn tiền?" sẽ nhận được cả chunk "7 ngày" và "14 ngày" → gây ra câu trả lời mâu thuẫn, giảm độ tin cậy.

**Kết quả định lượng:**

| Metric | Inject bad (`inject-bad`) | Pipeline chuẩn (`day10-final`) | Delta |
|--------|-----------|---------------|-------|
| expectation E3 (halt) | FAIL (skipped) | OK | Fixed |
| hits_forbidden (q_refund_window) | **yes** | **no** | ✅ |
| cleaned_refund_window_fixed (metric) | 0 | 1 | Auto-fix đã hoạt động |
| contains_expected (q_refund_window) | yes | yes | — |

> Chứng cứ: `artifacts/eval/before_after_eval.csv` (sau khi chạy `eval_retrieval.py` trên cả 2 version của DB)

---

## 4. Freshness & monitoring (~120 từ)

**SLA:** 24h (từ `data_contract.yaml`), grace period 2h.

**Đo tại 2 boundary:**
- **Ingest boundary:** Dựa vào `exported_at` trong CSV — thời điểm dữ liệu được xuất từ nguồn.
- **Publish boundary:** Dựa vào `run_timestamp` trong manifest — thời điểm dữ liệu sẵn sàng cho agent.

**Trên data mẫu:** `check_manifest_freshness` trả về **FAIL**. Lý do: `exported_at` trong CSV mẫu đã quá cũ (>24h), trong khi `run_timestamp` (publish) thì mới. Kết quả FAIL chứng minh logic kiểm tra freshness hoạt động đúng như thiết kế, cảnh báo khi dữ liệu nguồn bị stale.

**Alert:** WARN > 80% SLA (19.2h) → FAIL > SLA → CRITICAL > SLA + grace → escalate cho Lead.

---

## 5. Liên hệ Day 09 (~80 từ)

Pipeline Day 10 là nền tảng dữ liệu cho agent của Day 09. Cả hai lab cùng sử dụng bộ 5 tài liệu policy trong `data/docs/*.txt`. Trong lab này, nhóm sử dụng collection riêng (`day10_kb`) để an toàn khi thử nghiệm inject lỗi. Để tích hợp vào hệ thống Day 09, chỉ cần cập nhật biến môi trường `CHROMA_COLLECTION` trong file `.env` để trỏ về collection mà agent đang sử dụng.

---

## 6. Rủi ro còn lại & việc chưa làm

- **Logic Fix:** Logic ban đầu trong `cleaning_rules.py` có lỗi, khiến pipeline luôn halt dù có cờ `--no-refund-fix`. Nhóm đã xác định và sửa lỗi này để pipeline chạy thành công.
- **Rule Coverage:** 3 rules mới được bổ sung nhưng chưa có sẵn data mẫu để trigger, phải dùng kịch bản inject để chứng minh (như trong bảng metric_impact).
- **Hard-coded values:** Một số giá trị như `HR_LEAVE_MIN_EFFECTIVE_DATE` đang được hard-code trong `cleaning_rules.py` thay vì đọc từ `data_contract.yaml`. Đây là điểm cần cải thiện để hệ thống linh hoạt hơn.
- **Bonus:** Chưa tích hợp các thư viện nâng cao như Great Expectations hay Pydantic cho validation.
- **Eval:** Chưa có LLM-judge để đánh giá câu trả lời của agent một cách tự động và toàn diện hơn.
