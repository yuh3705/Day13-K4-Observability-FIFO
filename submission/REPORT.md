# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: FIFO
- Repository URL: https://github.com/yuh3705/Day13-K4-Observability-FIFO
- Commit SHA cuối:
- Thành viên và vai trò:

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 100/100
- Tổng số traces: 22
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard:

## 3. Logging và tracing

- Evidence correlation ID:
- Evidence PII redaction:
- Evidence trace waterfall:
- Giải thích một span đáng chú ý:

## 4. Prompt versioning

- Prompt name:
- Version/label baseline:
- Version/label candidate:
- Trace ID của mỗi version:
- Bằng chứng đổi label hoặc rollback:

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`:
- Evidence dashboard:
- SLO đã chọn và lý do:
- Alert rules và runbook:

## 6. Điều tra challenge

- Challenge ID: `day13-k4-observability-v1`.
- Triệu chứng từ metrics: Traffic 5 request; latency p50/p95/p99 lần lượt `2886/3632/3632 ms`; error rate `0%`; avg cost `0.0021 USD`, total cost `0.0103 USD`; quality average `0.84`.
- Trace ID liên quan: `49eda383d00af1caaeeeca15a0e384090` (trace `run` trong Langfuse), `session_id=k4-challenge-s03`; waterfall xác định `retrieve=2.50s` và `generate=0.15s`.
- Log line/correlation ID liên quan: `req-d69e5378`. Log `request_received` lúc `09:05:12.695858Z` và `response_sent` lúc `09:05:15.586084Z` ghi nhận `latency_ms=2889`, `feature=monitoring`, `session_id=k4-challenge-s03`, `cost_usd=0.001575`, `quality_score=0.8`; không có `request_failed`.
- Root cause: Metrics cho thấy latency p50/p95/p99 là `2886/3632/3632 ms`, trong đó p95 vượt SLO `3000 ms`, nhưng error rate `0%` và cost không spike. Trace khoanh vùng bottleneck ở span `retrieve` (`2.50s`), không phải `generate` (`0.15s`). Log cùng correlation ID xác nhận request hoàn tất nhưng chậm (`2889 ms`). Nguyên nhân gốc là incident `rag_slow`; tại [app/mock_rag.py:19](../app/mock_rag.py:19), khi incident bật, dòng 20 gọi `time.sleep(2.5)`, làm RAG retrieval tăng khoảng 2.5 giây so với baseline 1.1--1.2 giây.
- Fix action: Trong production, rollback/tắt thay đổi gây chậm bằng `python scripts/inject_incident.py --scenario rag_slow --disable`, kiểm tra lại vector store/RAG dependency, rồi chạy cùng workload để xác nhận p95 trở về baseline.
- Preventive measure: Duy trì alert `high_latency_p95` khi p95 > 3000 ms trong 5 phút; theo dõi riêng latency của `retrieve` và `generate`, ghi bắt buộc `correlation_id`, và dùng trace-to-log correlation để điều tra nhanh các request ở tail latency.

## 7. Đóng góp cá nhân

Với mỗi thành viên, ghi rõ nhiệm vụ và link commit/PR tương ứng.


| Thành viên | Phần việc | Commit/PR | Điều đã học |
| ------------ | ----------- | --------- | ---------------- |
|              |             |           |                  |
