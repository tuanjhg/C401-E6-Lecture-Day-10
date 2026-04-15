# Báo cáo cá nhân — Dũng

**Họ và tên:** Nguyễn Mạnh Dũng
**Mã học viên:** 2A202600309
**Vai trò:** Lead / Orchestration
**Ngày nộp:** 2026-04-15
**Độ dài:** ~450 từ

---

## 1. Phụ trách

Tôi chịu trách nhiệm chính cho file `etl_pipeline.py`, đóng vai trò tích hợp và điều phối (orchestration) toàn bộ luồng xử lý dữ liệu. Công việc của tôi bao gồm:
- Xây dựng khung sườn pipeline, logic sinh `run_id` để đảm bảo khả năng truy vết (traceability) cho mỗi lần chạy.
- Tích hợp các module do các thành viên khác phát triển: `transform/cleaning_rules.py` (Tuấn), `quality/expectations.py` (Hải), và logic `embed/prune` vào ChromaDB (Thuận).
- Quản lý luồng chính, cơ chế logging, và tạo file manifest cuối mỗi run để ghi nhận trạng thái của pipeline.

**Bằng chứng:** file `etl_pipeline.py` trong repo của nhóm.

---

## 2. Quyết định kỹ thuật

- **Luồng Fail-Fast:** Tôi quyết định cấu trúc pipeline theo một chuỗi các bước tuần tự (Ingest → Clean → Validate → Embed). Một `expectation` có mức độ `halt` bị vi phạm sẽ dừng toàn bộ pipeline ngay lập tức. Quyết định này nhằm ngăn chặn dữ liệu bẩn hoặc không tuân thủ contract bị publish aguas downstream cho agent sử dụng.
- **Traceability qua `run_id`:** Việc sinh `run_id` và dùng nó để định danh tất cả artifacts (logs, manifests, cleaned/quarantine CSVs) là một quyết định quan trọng. Nó cho phép team dễ dàng gỡ lỗi bằng cách liên kết tất cả output từ một lần chạy duy nhất.
- **Tích hợp logic Prune:** Tôi đã tích hợp logic của Thuận để xóa các vector không còn tồn tại trong dữ liệu sạch mới nhất (`prune logic`). Quyết định này đảm bảo collection trong Vector DB luôn là một "snapshot" chính xác của dữ liệu đã được publish, tránh được rủi ro agent truy xuất phải các "mồi cũ" đã bị loại bỏ.

---

## 3. Sự cố / anomaly

Sự cố lớn nhất xảy ra trong quá trình tích hợp Sprint 2. Khi chạy pipeline tổng thể lần đầu, pipeline đã dừng (`HALT`) với lỗi "stale refund window", dù logic fix đã được bật theo mặc định.

**Phân tích:** Tôi đã truy vết từ `etl_pipeline.py` vào `cleaning_rules.py` và phát hiện ra một lỗi logic. Rule xử lý "14 ngày" được viết để *luôn luôn* đưa record vào quarantine và đánh dấu `has_stale_refund=True`, khiến cho đoạn code fix không bao giờ được thực thi.

**Giải pháp:** Tôi đã phối hợp với team Transformation (Tuấn) để tái cấu trúc lại rule này. Logic mới sẽ kiểm tra cờ `apply_refund_window_fix` *trước*, nếu `True` thì sửa và đưa vào luồng cleaned, nếu `False` thì mới quarantine. Sự thay đổi này đã giải quyết được vấn đề pipeline bị dừng không mong muốn.

---

## 4. Before/after

- **Trước khi sửa lỗi:** Chạy `python etl_pipeline.py run` cho kết quả `Exit Code: 2` và log `PIPELINE_HALT: stale refund window (14 ngày) detected...`
- **Sau khi sửa lỗi:** Chạy lại lệnh tương tự cho kết quả `Exit Code: 0` và log `PIPELINE_OK`. Log metrics cũng cho thấy `cleaned_refund_window_fixed=1`, chứng minh rule fix đã hoạt động. Toàn bộ 7 records sạch đã được embed thành công.

---

## 5. Cải tiến thêm 2 giờ

Với vai trò điều phối, tôi sẽ ưu tiên tăng tính ổn định và linh hoạt cho pipeline:
- **Thêm cơ chế Retry:** Bổ sung một flag ví dụ `--retries=3` cho `etl_pipeline.py`. Cơ chế này sẽ cho phép pipeline tự động chạy lại nếu gặp các lỗi tạm thời (transient errors), ví dụ như lỗi mạng khi tải model embedding.
- **Tách riêng các stage:** Tái cấu trúc hàm `cmd_run` để có thể bỏ qua một số stage thông qua flags (ví dụ `--skip-embed`). Điều này sẽ giúp các thành viên khác (như team Quality) kiểm thử logic của họ nhanh hơn mà không cần chạy toàn bộ pipeline.
