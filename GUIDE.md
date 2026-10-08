# Hướng dẫn thực hành — Region Failover

## Mục lục

1. [Mục tiêu học tập](#mục-tiêu-học-tập)
2. [Phân bổ thời gian](#phân-bổ-thời-gian)
3. [Bước 0 — Chuẩn bị môi trường (10 phút)](#bước-0--chuẩn-bị-môi-trường-10-phút)
4. [Bước 1 — Baseline (15 phút)](#bước-1--baseline-15-phút)
5. [Bước 2 — Red Team (25 phút)](#bước-2--red-team-25-phút)
6. [Bước 3 — Containment (50 phút)](#bước-3--containment-50-phút)
7. [Bước 4 — Chứng minh và thu thập evidence (20 phút)](#bước-4--chứng-minh-và-thu-thập-evidence-20-phút)
8. [Các lỗi thường gặp](#các-lỗi-thường-gặp)
9. [Xử lý sự cố (Troubleshooting)](#xử-lý-sự-cố-troubleshooting)
10. [Bài tập mở rộng](#bài-tập-mở-rộng)
11. [Câu hỏi suy ngẫm](#câu-hỏi-suy-ngẫm)

## Mục tiêu học tập

Sau bài lab này, bạn sẽ:

- Quan sát một pipeline xử lý request sụp đổ khi **không có** disaster recovery, và đo chính xác hệ thống ngừng hoạt động trong bao lâu.
- Hiện thực ba thành phần: health checker, quy trình failover và runbook — ba mảnh ghép biến trạng thái "sập vĩnh viễn" thành "sập N giây".
- Chạy lại đúng cuộc tấn công ban đầu và chứng minh, từ timestamp trong log gốc, rằng RTO ≤ 300s và RPO được giới hạn, định lượng rõ ràng.
- Tạo bộ evidence (`reports/rto-evidence.md`, `reports/postmortem.md`, `reports/runbook.md`) trong đó mọi con số đều truy ngược được về một file và một dòng log thật.

Toàn bộ hệ thống chạy cục bộ: hai tiến trình FastAPI đại diện cho hai region, SQLite đại diện cho vector DB, và filesystem đại diện cho S3. Những hành vi quan trọng — polling health check, độ trễ replication (replication lag), thời gian warm-up của GPU pool, cache TTL của DNS — đều hoạt động thật; chỉ có hạ tầng bên dưới được đơn giản hoá.

## Phân bổ thời gian

| Bước | Thời lượng | Trọng tâm | Tham chiếu slide |
|---|---:|---|---|
| 0. Chuẩn bị | 10 phút | Cài thư viện, seed dữ liệu, khởi động hệ thống | — |
| 1. Baseline | 15 phút | Lần theo đường đi của request; kiểm tra Region B | §1 RTO/RPO Fundamentals |
| 2. Red Team | 25 phút | Tắt Region A khi đang có traffic | §1 Case Study · §4 Anti-Patterns |
| 3. Containment | 50 phút | Hiện thực health check, failover, runbook | §2 Multi-Region Patterns · §3 State Recovery · §4 Failover Automation |
| 4. Chứng minh | 20 phút | Tấn công lại, đo RTO/RPO, viết evidence | §4 Runbook & Postmortem · §6 Game Day |

---

## Bước 0 — Chuẩn bị môi trường (10 phút)

```bash
cd lab23-region-failover
pip install -r requirements.txt
make seed
bash scripts/up_bare.sh
curl localhost:8080/v1/infer
```

`make seed` tạo Region A với 200 tài liệu và model weights trên đĩa; Region B bắt đầu ở trạng thái **trống**. Khoảng trống này là chủ ý, và bạn sẽ xử lý nó ở Bước 3.

**Checkpoint** — cả ba service đều phải phản hồi:

```bash
curl localhost:8001/healthz   # Region A
curl localhost:8002/healthz   # Region B
curl localhost:8080/edge/state
```

> Nếu một service không khởi động được, hãy xem `run/region-a.log`, `run/region-b.log` hoặc `run/edge.log` trước tiên. Đừng chuyển sang Docker để lách lỗi của chế độ bare: khoảng 90% trường hợp nguyên nhân là một tiến trình cũ vẫn đang giữ cổng 8001/8002/8080.

---

## Bước 1 — Baseline (15 phút)

Đọc `serving/app.py` và `edge/proxy.py`, sau đó lần theo một request đi qua ba lớp:

1. **`edge/proxy.py`** — thành phần thay thế DNS/LB. Chọn upstream dựa trên `edge/active_region`, nhưng cache kết quả trong `EDGE_TTL_SECONDS`.
2. **`serving/app.py`** — API inference. Endpoint `/readyz` kiểm tra pool state, model weights *và* số lượng vector, chứ không chỉ kiểm tra "tiến trình còn sống".
3. **`state/`** — vector DB (SQLite) và model weights trên đĩa.

Kiểm tra trực tiếp Region B:

```bash
curl localhost:8002/v1/state
```

**Trước khi viết bất kỳ dòng code nào, hãy trả lời ba câu hỏi:**

1. Nếu Region A sập ngay lúc này, thành phần nào sẽ phát hiện ra, và bằng cách nào?
2. Region B hiện có dữ liệu hoặc model weights nào không?
3. Nếu chuyển `edge/active_region` sang `b` ngay lúc này, người dùng sẽ nhận được gì?

**Đáp án mong đợi:** chưa có thành phần nào phát hiện · Region B có `count:0`, `weights:false` · người dùng sẽ nhận `region_not_ready`. Đây là bài học cốt lõi của DR cho hạ tầng AI: một tiến trình còn sống không đồng nghĩa với việc region đó có khả năng phục vụ inference.

---

## Bước 2 — Red Team (25 phút)

Tạo traffic, chờ 8 giây, rồi tắt Region A trong lúc hệ thống đang phục vụ:

```bash
python3 loadgen/traffic.py --duration 40 --rps 2 --out reports/drill-1-nodr.jsonl &
sleep 8
python3 chaos/kill_region.py --region a --mode netblock --mock
```

Ở chế độ bare, `netblock --mock` gửi `SIGSTOP`: kết nối TCP vẫn mở được nhưng không có phản hồi nào — triệu chứng giống hệt một gói tin bị drop, nên client bị treo cho đến khi timeout. Ngược lại, `--mode stop` gửi `SIGKILL`, khiến request thất bại ngay lập tức với lỗi connection.

Khi load generator kết thúc, hãy đo mức độ hồi phục (vốn không tồn tại) và lưu kết quả để dùng làm evidence:

```bash
python3 tools/measure_rto.py --loadgen reports/drill-1-nodr.jsonl --target-rto 300 \
    | tee reports/measure-drill-1.json
```

**Kết quả tham khảo** (kết quả của bạn sẽ khác, điều này là bình thường): tổng cộng 32 request, request lỗi đầu tiên xuất hiện tại `+0.2s` với timeout `2017.7ms` (`reports/drill-1-nodr.jsonl:17`), 16/32 request thất bại, verdict là `"rto_verdict":"NO_RECOVERY"`.

Khôi phục Region A trước khi sang bước tiếp theo — ở chế độ bare **bắt buộc có `--backend bare`**:

```bash
python3 chaos/kill_region.py restore --region a --backend bare
```

> Lệnh `restore` không có lối tắt `--mock`. Nếu bỏ `--backend bare`, lệnh mặc định dùng Docker và sẽ lỗi khi máy không có Docker daemon. Nếu trước đó bạn dùng `--mode stop` thì tiến trình đã bị `SIGKILL`; `restore` sẽ báo `need_manual_start`, khi đó chỉ cần chạy lại `bash scripts/up_bare.sh`.

---

## Bước 3 — Containment (50 phút)

Ba file trong `dr/` là các skeleton chứa `NotImplementedError`. Mỗi file có docstring mô tả chi tiết — hãy đọc hết trước khi viết code. **Không sửa bất kỳ file nào ngoài `dr/`** để làm cho test pass.

### 3a. `dr/health_checker.py` — 12 phút

Poll `/readyz` của cả hai region sau mỗi `interval` giây. Chỉ chuyển sang `UNHEALTHY` sau `threshold` lần thất bại **liên tiếp**, và chỉ ghi một dòng JSONL khi trạng thái thực sự thay đổi. Mỗi dòng `state_change` phải có tối thiểu các trường: `event` (giá trị `"state_change"`), `ts`, `region`, `to`, `reason`, `consecutive_fails`, `interval_s`, `threshold`.

> Hai trường `event` và `consecutive_fails` không được nêu trong docstring của skeleton nhưng được test sử dụng: test chỉ nhận các dòng có `event == "state_change"` và kiểm tra `consecutive_fails >= 3`; `tools/measure_rto.py` cũng lọc theo `event == "state_change"`.

> `interval × threshold` là **ngưỡng phát hiện tối thiểu (detection floor)**, và nó luôn nằm trong RTO của bạn. Với `interval=5s, threshold=3`, ngưỡng này là 15 giây. Trong một lần chạy tham khảo, riêng phần này đã chiếm khoảng 53% trong tổng RTO 28.5s.

> Lưu ý: ngay từ đầu Region B chưa ready (chưa có dữ liệu), nên health checker có thể ghi sự kiện `UNHEALTHY` cho Region B. Điều này không sai; `tools/measure_rto.py` chỉ lọc các sự kiện của region đã bị tắt.

```bash
pytest tests/test_failover.py::test_health_checker_can_threshold_lien_tiep
```

### 3b. `dr/failover.py` — 15 phút

Hiện thực đúng năm bước sau, theo đúng thứ tự:

| # | Bước | Nội dung |
|---|---|---|
| 1 | `1_verify_target` | Kiểm tra trạng thái hiện tại của region đích |
| 2 | `2_restore_snapshot` | Restore qua `state/snapshot.py`; ghi log `rpo_seconds`, `docs_lost`, `embed_model_version` |
| 3 | `3_scale_pool` | Chuyển pool state của region đích sang `full` |
| 4 | `4_wait_ready` | Poll `/readyz` cho đến khi trả về 200 |
| 5 | `5_dns_cutover` | Chỉ đến lúc này mới ghi region đích vào `edge/active_region` |

> Nếu bước 4 bị timeout, phải **abort** — không được cutover. Chuyển DNS khi region đích chưa thực sự ready sẽ khiến người dùng nhận lỗi 503 từ *cả hai* region, làm RTO dài hơn thay vì ngắn đi.

Hàm `failover()` phải trả về một `dict` có khoá `ok` (`True` khi cutover thành công, `False` khi abort). Test dựa vào khoá này, và `dr/runbook.py` cũng dùng chính dict đó để ghi log các bước 4 và 5.

```bash
pytest tests/test_failover.py::test_failover_khong_cutover_khi_target_chua_ready
```

### 3c. `dr/runbook.py` — 13 phút

Tự động hoá bảy bước checklist trong §4 "Runbook: Region Chính Down": xác nhận outage, thông báo incident và bắt đầu bấm giờ, gọi `failover.failover(...)` **đúng một lần**, kiểm tra state replica, ghi nhận kết quả cutover, kiểm tra golden signals bằng 10 request thật, rồi ghi log tổng kết sau sự cố.

Hành vi mặc định phải hỏi xác nhận `y/N` — đây là thiết kế bán tự động có chủ ý, không phải tự động hoàn toàn (§4 cảnh báo rằng failover tự động hoàn toàn mà không có circuit breaker sẽ gây flapping giữa hai region). Cờ `--auto` chỉ dành cho drill chấm điểm hoặc CI.

- Log của runbook: `reports/runbook-run.jsonl`
- Năm bước con của failover: `reports/failover-events.jsonl`

### 3d. `reports/runbook.md` — 10 phút

Điền vào template một trang sao cho người **không** viết code này vẫn có thể thực hiện lúc 3 giờ sáng. Mỗi bước cần có: lệnh copy-paste được, tín hiệu rõ ràng để biết bước đó đã thành công, vai trò người phụ trách, và — quan trọng nhất — điều kiện rollback cùng người có thẩm quyền quyết định chuyển traffic trở lại Region A.

> Template `reports/runbook.md` liệt kê bảy bước theo cách chia thao tác thủ công, còn `dr/runbook.py` tự động hoá bảy bước theo thứ tự của §4 (xác nhận outage → thông báo → scale pool qua failover → verify state → cutover → golden signals → post-incident). Hai danh sách không trùng khớp từng dòng; hãy đảm bảo runbook của bạn bao phủ đủ cả bảy hạng mục trong §4.

---

## Bước 4 — Chứng minh và thu thập evidence (20 phút)

`dr/failover.py` restore từ một snapshot, nhưng một bản checkout mới chưa từng tạo snapshot nào. Bạn phải chạy ingest và replication **trước** khi tấn công, nếu không `failover.py` sẽ lỗi ngay tại `2_restore_snapshot`.

```bash
python3 state/ingest.py --region a --rate 0.5 --duration 150 &
python3 state/replicate.py --every 30 --duration 150 --backend fs &
sleep 5   # chờ chu kỳ replication đầu tiên hoàn tất trước khi bắt đầu các bước khác

python3 loadgen/traffic.py --duration 100 --rps 2 --out reports/drill-2-withdr.jsonl &
python3 dr/health_checker.py --interval 5 --threshold 3 --duration 100 \
    --out reports/health-events.jsonl &
sleep 12
python3 chaos/kill_region.py --region a --mode netblock --mock
python3 dr/runbook.py --primary a --target b --backend fs --auto
python3 tools/measure_rto.py --loadgen reports/drill-2-withdr.jsonl --target-rto 300 \
    | tee reports/measure-drill-2.json
```

> Hãy chờ load generator kết thúc (khoảng 100 giây kể từ lúc khởi động) trước khi chạy lệnh `measure_rto.py` cuối cùng; nếu đo quá sớm, file log chưa đủ dữ liệu.

### Đọc kết quả

Bảng dưới đây lấy từ **một** lần chạy tham khảo — để minh hoạ định dạng kết quả, không phải con số để sao chép:

| Mốc (tính từ `t_outage`) | +giây | Evidence |
|---|---:|---|
| Người dùng thấy lỗi đầu tiên | 0.2 | `reports/drill-2-withdr.jsonl:25` |
| Health check đánh dấu Region A `UNHEALTHY` | 14.9 | `reports/health-events.jsonl:3` |
| Hoàn tất restore snapshot | 17.2 | `reports/failover-events.jsonl:2` |
| Region B ready | 23.3 | `reports/failover-events.jsonl:4` |
| DNS cutover | 23.4 | `reports/failover-events.jsonl:5` |
| Request thành công đầu tiên từ B | **28.5** | `reports/drill-2-withdr.jsonl:39` |

RTO = 28.5s so với mục tiêu 300s → **PASS**. RPO trong lần chạy đó là 14.04s / mất 7 tài liệu; một lần chạy khác đo được 4.01s / mất 2 tài liệu. **RPO dao động là điều bình thường**, vì nó phụ thuộc vào thời điểm chu kỳ `state/replicate.py` gần nhất diễn ra so với thời điểm restore. RTO ổn định hơn nhiều giữa các lần chạy (thường trong khoảng 28.3–28.6s). Không sao chép bất kỳ con số mẫu nào vào báo cáo của bạn.

### Hoàn thiện các báo cáo

Điền `reports/rto-evidence.md` và `reports/postmortem.md` bằng số liệu từ **lần chạy của chính bạn**. Mỗi ô Evidence phải là một `path:line` có thật. Bảng phân rã RTO phải bao phủ cả bốn thành phần và tổng của chúng phải bằng RTO đo được:

1. Ngưỡng phát hiện của health check (detection floor)
2. Restore snapshot
3. Warm-up của GPU pool
4. Cache TTL của DNS/LB

> Giữa mốc health check phát hiện và mốc bắt đầu restore thường có một khoảng trễ nhỏ do runbook (xác nhận outage, ghi log). Nếu bốn thành phần trên không cộng đủ thành RTO, hãy thêm một dòng "Khác (độ trễ của runbook)" để bảng cân đối thay vì làm tròn số.

Chạy riêng từng evidence gate trong lúc làm bài:

```bash
pytest tests/test_rto_evidence.py::test_evidence_table_tro_vao_file_that
pytest tests/test_rto_evidence.py::test_evidence_table_da_dien_that
pytest tests/test_rto_evidence.py::test_so_trong_bang_khop_voi_so_trong_log
pytest tests/test_rto_evidence.py::test_postmortem_co_gap_analysis
```

Sau đó chạy toàn bộ:

```bash
python3 -m pytest tests/ -v
```

---

## Các lỗi thường gặp

| Lỗi | Cách khắc phục |
|---|---|
| Máy không có Docker | Dùng `bash scripts/up_bare.sh` — đây là đường chính, không phải phương án dự phòng. |
| Cài đặt MinIO tốn quá nhiều thời gian | Dùng `--backend fs` (snapshot nằm trong `state/_replica/`). MinIO chỉ là bài tập mở rộng. |
| Tắt cả hai region cùng lúc | Script chaos từ chối mặc định. `--i-really-want-both` ép được nhưng drill sẽ bị đánh dấu `INVALID`. |
| Đo RTO "theo cảm giác" | Chỉ timestamp từ `loadgen/traffic.py` mới được tính. Sự kiện kill phải nằm trong khoảng thời gian chạy của load generator. |
| Quên detection floor | `interval_s × threshold` phải là một dòng riêng trong bảng phân rã RTO — không phải tuỳ chọn. |
| Sửa tay `edge/active_region` | `measure_rto.py` sẽ cảnh báo nếu `t_cutover < t_detect`, và `test_drill2_hop_le` từ chối mọi cảnh báo. Hãy chạy lại drill tự động. |
| Cho rằng warm-up bắt đầu ngay khi tiến trình khởi động | `serving/app.py` chỉ bắt đầu đếm warm-up khi `pool_state` thay đổi *trong lúc đang chạy*. Không sửa `serving/` để né điều này. |
| Nhầm `netblock` với tiến trình đã chết | Ở chế độ bare, `netblock` là `SIGSTOP` (tiến trình bị tạm dừng, không bị kill). Khôi phục bằng `restore --region a --backend bare`. Chỉ `stop` mới thực sự kill (`SIGKILL`). |
| Gọi `failover.py` khi chưa có snapshot nào | Hãy chạy `state/replicate.py` và chờ lần `put` đầu tiên — nếu không, `state/snapshot.py get` sẽ lỗi ở `2_restore_snapshot`. |
| Dùng chế độ Docker cho drill chấm điểm | `--backend docker` hoạt động nhưng thời gian phụ thuộc máy và Docker daemon, không tái lập được. Chấm điểm luôn dùng bare `--mock`. |
| Chạy `make clean` giữa chừng | Lệnh này xoá toàn bộ `reports/*.jsonl`, `reports/*.json` và `chaos/chaos-events.jsonl` — tức là xoá luôn evidence của drill 1. Chỉ dùng khi muốn làm lại từ đầu. |

## Xử lý sự cố (Troubleshooting)

- **Cổng 8001/8002/8080 đã bị chiếm** — chạy `bash scripts/down_bare.sh`, tìm tiến trình còn giữ cổng, rồi khởi động lại. Xem chi tiết trong `run/*.log`.
- **Region A chưa ready ngay sau khi chuẩn bị** — chạy lại `make seed`; kiểm tra `state/region-a/pool_state`, file weights và file SQLite đã thực sự được tạo.
- **`restore` báo chưa từng có snapshot nào được `put`** — kiểm tra `reports/replication.jsonl` và `state/_replica/dr-artifacts/MANIFEST.json`.
- **Drill baseline không có request nào thất bại** — xác nhận traffic đi qua cổng 8080 (edge), và timestamp của sự kiện kill nằm giữa request đầu tiên và request cuối cùng của load generator.
- **Log của các drill cũ gây nhiễu** — `make clean` xoá toàn bộ state và log đã sinh ra; sau đó chạy lại `make seed` và `bash scripts/up_bare.sh`.

## Bài tập mở rộng

1. **MinIO thật** — chạy `docker compose up minio`, lặp lại luồng snapshot/failover với `--backend minio`, và so sánh độ trễ `put`/`get` với backend `fs`.
2. **Postgres PITR** — thêm một metadata store Postgres, dùng `pg_basebackup` kết hợp WAL archiving, restore về một thời điểm cụ thể (point in time), và đo riêng RTO của lớp này (§3 PITR).
3. **Active-active** — giữ cả hai region ở `pool_state=full`, chia traffic 50/50 tại edge, và thiết kế cơ chế giải quyết xung đột khi ingest đồng thời (§2 Active-Passive vs. Active-Active).
4. **Terraform (chỉ viết, không apply)** — soạn một resource `aws_s3_bucket_replication_configuration` tương ứng với `put`/`get` trong `state/snapshot.py`, và với ý nghĩa của việc đánh phiên bản `MANIFEST.json` trên một bucket thật.
5. **Chaos ngẫu nhiên** — ngẫu nhiên hoá thời điểm kill và chế độ (`stop` so với `netblock`) qua năm lần chạy; báo cáo RTO trung bình và độ lệch chuẩn (§6 Chaos Engineering).
6. **Tự đánh giá mức độ trưởng thành DR** — xếp hạng hệ thống của bạn theo thang Level 0–4 trong slide, và nêu thay đổi cụ thể cần thực hiện để lên mức tiếp theo.

## Câu hỏi suy ngẫm

1. RTO của bạn là tổng của detection floor, restore snapshot, GPU warm-up và cache TTL của DNS. Thành phần nào có thể rút ngắn mà không làm tăng nguy cơ flapping, và việc đó đánh đổi điều gì?
2. Nếu `dr/health_checker.py` chạy cùng tiến trình với serving API mà nó giám sát, ai sẽ phát cảnh báo khi tiến trình đó chết? Hãy kiểm tra: health checker của bạn có import gì từ `serving/` không?
3. Khi có người hỏi "RTO 5 phút của chúng ta có thực sự đúng không?", bạn mở log hoặc file evidence nào để trả lời bằng một con số thật thay vì phỏng đoán?
