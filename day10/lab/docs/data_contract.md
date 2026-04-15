# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| **policy_refund_v4** | CSV export từ Document Management System | Migration error: chunk_id=3 có text "14 ngày" (từ v3 cũ) thay vì "7 ngày" canonical; duplicate chunks (id 1-2 giống nhau) | Quarantine chunk với text chứa "14 ngày"; `quarantine_records` > 0 là alert |
| **hr_leave_policy** | CSV export từ HR Management System | Version conflict: hr_leave_policy có 2 bản (row 7: "10 ngày" HR 2025, row 8: "12 ngày" HR 2026); effective_date không consistent | Chọn version mới nhất theo effective_date; flag records nếu `effective_date` NULL |
| **it_helpdesk_faq** | CSV export từ Knowledge Base | Date format inconsistency: chunk_id=10 có effective_date="01/02/2026" (DD/MM/YYYY) thay vì ISO 8601 | Parse error khi normalize effective_date; SLA tracking metric: `p1_response_sla_minutes` = 15 |
| **sla_p1_2026** | CSV export từ SLA Policy DB | Single policy chunk; kiểm chứng payload đầy đủ | Verify chunk_text không empty; metric: `p1_resolution_sla_hours` = 4 |
| **legacy_catalog_xyz_zzz** | CSV export từ Legacy system (deprecated) | Unknown source — không trong allowlist; có thể obsolete | Quarantine tất cả records từ source này; flag trong logs |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | ID ổn định; trong cleaned tương ứng 1:1 với record (không dedupe) |
| doc_id | string | Có | Khóa logic: one of {policy_refund_v4, hr_leave_policy, it_helpdesk_faq, sla_p1_2026}; unknown → quarantine |
| chunk_text | string | Có | Text nội dung; min 8 chars; non-empty; trim whitespace |
| effective_date | date | Có | ISO 8601 (YYYY-MM-DD); parse "DD/MM/YYYY" → ISO; NULL → quarantine |
| exported_at | datetime | Có | ISO 8601 timestamp (UTC); từ ingest time; không modify |

---

## 3. Quy tắc quarantine vs drop

| Điều kiện | Action | Lý do | Approval để merge lại |
|----------|--------|-------|----------------------|
| `chunk_text IS NULL` hoặc độ dài < 8 ký tự | **DROP** | Text không đủ thông tin; không khôi phục được | Không cần (xóa vĩnh viễn) |
| `doc_id NOT IN allowlist` (vd legacy_catalog_xyz_zzz) | **QUARANTINE** | Unknown source; cần verify trước publish | Data Owner duyệt metadata + source confirmation |
| `effective_date IS NULL` | **QUARANTINE** | Missing key metadata; không biết version nào active | Data Owner xác nhận version + backfill date |
| `effective_date` format invalid (trừ DD/MM/YYYY parse được) | **QUARANTINE** | Parse error; cần thủ công check | DevOps check log + format correction |
| Text chứa "14 ngày" + `doc_id = 'policy_refund_v4'` | **QUARANTINE + HALT** | Stale refund window từ v3 migration; xung đột canonical | Policy Owner: confirm fix refund window v4 = 7 ngày |
| Duplicate `chunk_id` | **DROP tất cả except latest** | Dedup by chunk_id; giữ record mới nhất theo `exported_at` | Tự động; log số records dropped |
| `chunk_id` + `doc_id` + hash(`chunk_text`) trùng | **DROP tất cả except first occurrence** | Semantic duplicate; giảm noise vector store | Tự động; metric `dedup_count` |

**Quarantine location:** `artifacts/quarantine/quarantine_<run-id>.csv` — reviewed weekly, approval từ Data Owner trước merge vào cleaned.

---

## 4. Phiên bản & canonical

### 4.1 Source of truth (canonical)

| doc_id | Canonical file | Owner | Version | Note |
|--------|----------------|-------|---------|------|
| **policy_refund_v4** | `data/docs/policy_refund_v4.txt` | Policy Team | v4 (2026-02-01) | **Migration risk:** v3 chunks (refund 14 ngày) còn trong raw → quarantine + halt |
| **hr_leave_policy** | `data/docs/hr_leave_policy.txt` | HR Team | 2026 (2026-02-01+) | Prefer effective_date >= 2026-01-01; drop version 2025 (row 7: "10 ngày") |
| **it_helpdesk_faq** | `data/docs/it_helpdesk_faq.txt` | IT Service Desk | Latest | No versioning; reviewed weekly |
| **sla_p1_2026** | `data/docs/sla_p1_2026.txt` | Service Management | 2026 | Static policy; no changes expected |

### 4.2 Version resolution

- **HR Leave Policy conflict:** Bộ dữ liệu chứa 2 versions (10 vs 12 ngày). **Canonical:** chỉ giữ >= 2026-01-01 → drop row 7 (2025-01-01).
- **Refund window conflict:** Chunk_id=3 có "14 ngày" từ migration lỗi. **Canonical:** policy_refund_v4 = 7 ngày → quarantine chunk_id=3 + fix trong cleaning_rules.
- **Effective date:** Quy tắc: khi có multiple versions của cùng content, chọn newest by `effective_date` (tie-breaker: latest `exported_at`).

### 4.3 Update frequency & SLA

| doc_id | Frequency | Lead time | Validation |
|--------|-----------|-----------|-----------|
| policy_refund_v4 | Daily | 12h | Check "14 ngày" giảm → 0 quarantine_records |
| hr_leave_policy | Monthly | 48h | Verify effective_date >= contract.policy_versioning.hr_leave_min_effective_date |
| it_helpdesk_faq | Weekly | 24h | Freshness check: exported_at within 7 days |
| sla_p1_2026 | Static | Static | Manual review; no automation required |
