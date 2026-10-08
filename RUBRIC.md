# Rubric chấm điểm — 100 điểm

Mọi điểm số đều được tính từ code trong `dr/`, các file log sinh ra khi **bạn tự chạy drill**, và ba báo cáo trong `reports/`. Một con số không truy ngược được về timestamp thật thì không được tính là bằng chứng (evidence).

## Bảng điểm

| Tiêu chí | Điểm | Cách đánh giá |
|---|---:|---|
| Bước 2: tấn công thành công, region không tự hồi phục | 15 | `pytest tests/test_rto_evidence.py::test_drill1_ton_tai_va_khong_phuc_hoi` — đồng thời `tools/measure_rto.py --loadgen reports/drill-1-nodr.jsonl --target-rto 300` phải cho thấy có request thất bại và `"rto_verdict":"NO_RECOVERY"`. |
| `dr/health_checker.py` — ngưỡng chống flapping đúng | 15 | `pytest tests/test_failover.py::test_health_checker_can_threshold_lien_tiep` — một lần probe thất bại không được làm đổi trạng thái; lần chuyển sang `UNHEALTHY` đầu tiên cần ≥ 3 lần thất bại liên tiếp. |
| `dr/failover.py` — không cutover trước khi target sẵn sàng | 20 | `pytest tests/test_failover.py::test_failover_khong_cutover_khi_target_chua_ready` — `reports/failover-events.jsonl` phải thể hiện đúng thứ tự `1_verify_target → 2_restore_snapshot → 3_scale_pool → 4_wait_ready → 5_dns_cutover`. |
| RTO ≤ 300s, log hợp lệ, request phục hồi do Region B phục vụ | 20 | `tools/measure_rto.py --loadgen reports/drill-2-withdr.jsonl --target-rto 300` phải trả về `"valid":true` và `"rto_verdict":"PASS"` — cùng với `pytest tests/test_rto_evidence.py::test_drill2_hop_le` và `::test_rto_do_duoc_bang_timestamp`. |
| RPO được báo cáo cả theo giây và theo số tài liệu bị mất | 10 | `pytest tests/test_rto_evidence.py::test_rpo_duoc_do_chu_khong_uoc_luong` — kết quả đo phải có `rpo_at_restore_s` và `docs_lost` khác null. Phép tính nền tảng được kiểm tra bằng unit test `pytest tests/test_failover.py::test_rpo_dem_dung_so_doc_bi_mat`. |
| `reports/rto-evidence.md` — thật, đầy đủ, số liệu khớp log | 10 | `pytest tests/test_rto_evidence.py::test_evidence_table_tro_vao_file_that`, `::test_evidence_table_da_dien_that`, `::test_so_trong_bang_khop_voi_so_trong_log` — cùng với `::test_health_check_interval_duoc_ghi_lai` để kiểm tra cấu hình health check. |
| `reports/runbook.md` và `reports/postmortem.md` hoàn chỉnh | 10 | `pytest tests/test_rto_evidence.py::test_postmortem_co_gap_analysis` kiểm tra postmortem. `runbook.md` được chấm thủ công: đủ bảy bước, lệnh copy-paste được, tín hiệu hoàn thành, người phụ trách và điều kiện rollback. Không có test tự động nào cho file này. |
| **Tổng** | **100** | Kiểm tra toàn bộ: `python3 -m pytest tests/ -v` |

## Điều kiện trượt cứng (Hard-Fail)

Các điều kiện sau khiến bài nộp **bị trượt bất kể tổng điểm**:

1. **`tools/measure_rto.py` trả về `"valid":false`** cho drill ở Bước 4. Các nguyên nhân thường gặp: ép sự cố kép (`--i-really-want-both`), tắt một region khi region còn lại đã ngừng hoạt động, không có request nào thất bại sau thời điểm kill, load generator và script chaos không chạy trùng khoảng thời gian, hoặc request "phục hồi" vẫn do chính region đã bị tắt phục vụ.
   Gate: `pytest tests/test_rto_evidence.py::test_drill2_hop_le`, `::test_rto_do_duoc_bang_timestamp`, `::test_chaos_khong_giet_ca_hai_region`.

2. **RTO trong `reports/rto-evidence.md` lệch quá 1 giây** so với `rto_measured_s` do `tools/measure_rto.py` tính ra — hoặc bảng RPO không khớp với log.
   Gate: `pytest tests/test_rto_evidence.py::test_so_trong_bang_khop_voi_so_trong_log`.

> `test_drill2_hop_le` còn yêu cầu danh sách `warnings` phải rỗng. Cụ thể, nếu cutover xảy ra *trước* khi health checker phát hiện sự cố (`t_cutover < t_detect`), bạn phải chạy lại quy trình tự động cho đúng — drill đó không được nộp như một drill hợp lệ.

## Kiểm tra toàn bộ

```bash
python3 -m pytest tests/ -v
```

## Lưu ý khi nộp bài

- Các file bằng chứng bắt buộc: `reports/drill-1-nodr.jsonl`, `reports/drill-2-withdr.jsonl`, `reports/health-events.jsonl`, `reports/failover-events.jsonl`, `reports/replication.jsonl` và `chaos/chaos-events.jsonl`. `.gitignore` đã cho phép commit các file này (riêng `chaos-events.jsonl` không bị bỏ qua ngay từ đầu).
- Các file kết quả đo như `reports/measure-drill-1.json` được tham chiếu trong template evidence nhưng **không** nằm trong danh sách ngoại lệ của `.gitignore` (mẫu `reports/*.json` đang bị bỏ qua). Hãy thêm vào bằng `git add -f`, nếu không file sẽ không được commit và các tham chiếu `path:line` tới nó sẽ không kiểm chứng được.
- Không chạy `make clean` sau khi đã hoàn thành drill: lệnh này xoá toàn bộ log và file evidence trong `reports/`.
