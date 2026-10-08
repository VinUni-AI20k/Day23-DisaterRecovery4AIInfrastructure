# Day 23 — Disaster Recovery & High Availability cho hạ tầng AI

**Hình thức:** 2 giờ · **Môi trường:** chạy hoàn toàn trên máy cá nhân, không cần tài khoản cloud · **Ý tưởng cốt lõi:** tự làm sập region của mình trước, rồi học cách vượt qua sự cố đó.

> RTO/RPO là những con số bạn **đo được từ log**, không phải con số đọc từ slide.

---

## Tổng quan

Lab mô phỏng một hệ thống AI serving hai region ngay trên máy của bạn:

| Thành phần | Tương đương thực tế | Cách mô phỏng trong lab |
|---|---|---|
| Region A / Region B | AWS us-east-1 / us-west-2 | Hai tiến trình FastAPI chạy ở hai cổng khác nhau |
| Vector DB | Pinecone / Weaviate / Qdrant | Một file SQLite cho mỗi region |
| Replication model weights | S3 Cross-Region Replication | Snapshot trên filesystem (`state/_replica/`) |
| DNS / Global Load Balancer | Failover theo health check của Route53 | Một proxy cục bộ đọc file con trỏ `edge/active_region` |
| Region outage | Network partition / sự cố AZ | `chaos/kill_region.py` (`SIGSTOP` hoặc `SIGKILL`) |

Các bước chính: khởi động hệ thống, quan sát hệ thống phục vụ bình thường, **tắt Region A khi đang có traffic**, rồi đo bằng timestamp thật (không dựa vào cảm giác) mất bao lâu để người dùng nhận ra sự cố và bao lâu để hệ thống hồi phục. Sau đó bạn tự xây dựng health check, quy trình failover và runbook tự động hoá, để chuyển từ trạng thái "không có khả năng hồi phục" sang một chỉ số RTO đo được và đạt yêu cầu.

Bài lab không cần tài khoản AWS, không phát sinh chi phí cloud, và đường chấm điểm không yêu cầu Docker.

## Bắt đầu nhanh

```bash
pip install -r requirements.txt
make seed              # seed Region A (200 tài liệu + weights), Region B để trống
bash scripts/up_bare.sh
curl localhost:8080/v1/infer
```

Kết quả mong đợi: `"edge_region":"a"` và câu trả lời bắt đầu bằng `"[a] ..."`.

Dừng toàn bộ hệ thống:

```bash
bash scripts/down_bare.sh
```

> **Chế độ Docker** (tuỳ chọn, `docker compose up -d`) dùng được để tìm hiểu thêm, nhưng **không** dùng cho bài diễn tập (drill) được chấm điểm, vì thời gian đo trên các máy khác nhau không tái lập được. Mọi drill trong [GUIDE.md](GUIDE.md) đều chạy ở chế độ bare với cờ `--mock`.

## Cấu trúc thư mục

```
serving/    API inference giả lập cho mỗi region (FastAPI). Trạng thái ready phụ thuộc vào
            pool state, model weights trên đĩa và số lượng vector — không chỉ là
            "tiến trình còn sống".
edge/       Thành phần thay thế DNS/load balancer. Định tuyến bằng cách đọc
            edge/active_region, kết quả được cache trong EDGE_TTL_SECONDS để mô phỏng
            hành vi cache DNS thực tế.
state/      Seed, ingest, snapshot/restore và replication cho vector DB và
            model weights.
chaos/      kill_region.py — gây lỗi kiểu `stop` (SIGKILL) hoặc `netblock` (SIGSTOP),
            có cờ --mock để kết quả chấm điểm tái lập được.
loadgen/    traffic.py — bộ sinh request liên tục. Đây chính là "đồng hồ" đo RTO:
            mỗi request là một dòng JSONL có timestamp.
dr/         ★ PHẦN BÀI TẬP CỦA BẠN — ba file khung (skeleton) cần hoàn thiện:
            health_checker.py, failover.py, runbook.py
tools/      measure_rto.py — tính RTO/RPO từ log và kiểm tra tính hợp lệ của drill.
tests/      Unit test cho code trong dr/, cùng các test "evidence gate" dùng để
            chấm điểm.
reports/    Các template bạn cần điền: runbook.md, rto-evidence.md, postmortem.md.
scripts/    up_bare.sh / down_bare.sh — khởi động/dừng hệ thống không cần Docker.
```

## Tài liệu liên quan

| Tài liệu | Mục đích |
|---|---|
| **[GUIDE.md](GUIDE.md)** | Hướng dẫn chi tiết 2 giờ — chuẩn bị, baseline, tấn công, khắc phục, chứng minh |
| **[RUBRIC.md](RUBRIC.md)** | Tiêu chí chấm điểm, điều kiện trượt cứng và lệnh kiểm tra cụ thể |

## Lưu ý an toàn

- Mọi mục tiêu trong lab đều là `127.0.0.1` / `localhost`. Không trỏ `URL`, `UPSTREAM` hoặc load generator tới bất kỳ máy chủ bên ngoài nào.
- Chế độ `--mock` chỉ gửi tín hiệu tiến trình (`SIGSTOP`/`SIGKILL`) và thao tác trên file cục bộ — không có tác động nào lên một cloud region thật.
- Script chaos từ chối tắt một region nếu region còn lại đã ngừng hoạt động, nhằm tránh vô tình gây sự cố kép (double outage). Nếu ép bỏ qua bằng `--i-really-want-both`, drill sẽ bị đánh dấu không hợp lệ khi chấm điểm.
