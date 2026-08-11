# Yêu cầu dashboard

Contract có thể kiểm tra bằng máy nằm tại `config/dashboard.yaml`. Hướng dẫn dựng và kiểm tra runtime nằm tại [DASHBOARD_SETUP.md](DASHBOARD_SETUP.md).

Dashboard chính cần đủ 6 nhóm thông tin:

1. Latency P50/P95/P99.
2. Traffic: request count hoặc QPS.
3. Error rate và breakdown theo loại lỗi.
4. Cost theo thời gian.
5. Tổng token input/output.
6. Quality proxy.

Tiêu chuẩn trình bày:

- Khoảng thời gian mặc định: 1 giờ.
- Tự refresh mỗi 15–30 giây nếu công cụ hỗ trợ.
- Có threshold hoặc SLO line.
- Ghi rõ đơn vị.
- Chỉ giữ 6–8 panel quan trọng ở lớp chính.
- Screenshot phải nhìn được tên panel và khoảng thời gian.

Kiểm tra contract trước khi chụp evidence:

```bash
python scripts/validate_dashboard.py
```

## Phần B — Đặc tả 6 panel

**Công cụ sử dụng: Langfuse Custom Dashboards** (Dashboards → Widgets → New Widget), đọc trực tiếp từ traces/observations/scores mà app đã gửi lên qua `app/agent.py`. Dashboard title: `Day 13 AI Observability`. Time range mặc định cho toàn bộ dashboard: **Last 1 hour (60 phút)**, khớp `dashboard.time_range_minutes: 60` trong contract.

> Lưu ý công cụ: Langfuse Custom Dashboards không có cấu hình auto-refresh interval hay threshold/goal-line vẽ sẵn trên chart (đã kiểm tra tài liệu chính thức, không thấy tính năng này). Cách xử lý: reload trang thủ công trước khi chụp evidence, và ghi ngưỡng SLO ngay trong **tên widget** (vd: "Latency P95 (SLO ≤ 3000ms)") để ảnh chụp vẫn thể hiện threshold theo đúng yêu cầu.
>
> Để panel Tokens và Quality có dữ liệu thật, `app/agent.py` đã được chỉnh: `usage_details` dùng key `input`/`output` (Langfuse chỉ gộp vào bucket input/output nếu tên key chứa đúng chuỗi đó — key cũ `prompt_tokens`/`completion_tokens` không được nhận diện, khiến usage luôn ra 0/0) và mỗi trace được gắn thêm `score_current_trace(name="quality", value=quality_score)`.

| # | Panel (tên widget) | Đơn vị | Time range | Threshold/SLO | Nguồn & cấu hình widget trong Langfuse |
|---|---|---|---|---|---|
| 1 | Latency percentiles (P50/P95/P99) | ms | Last 1h | P95 ≤ 3000ms | Data source: **Traces** (hoặc Observations, filter `name = run`) · Metric: Latency · Aggregation: p50, p95, p99 · Chart: Line (3 series) hoặc 3 Big Number tile |
| 2 | Request traffic | requests / phút | Last 1h | ≥ 1 request/phút | Data source: **Traces** · Metric: Count · Dimension: time (bucket 1 phút) · Chart: Line/Bar time series |
| 3 | Error rate and breakdown | percent (%) | Last 1h | ≤ 2% | Widget A — "Error rate": Data source: **Observations**, filter `level = ERROR` · Metric: Count(level=ERROR) / Count(total) × 100 · Chart: Big Number. Widget B — "Error breakdown": cùng data source, filter `level = ERROR` · Dimension: `statusMessage` (hoặc `name` của observation lỗi, vd `retrieve`) · Chart: Bar/Pie. *Lưu ý: breakdown lấy theo `statusMessage` do OTel tự ghi khi exception raise trong span, không phải field `error_type` riêng như trong `data/logs.jsonl`.* |
| 4 | Cost over time | USD | Last 1h | Tổng ≤ $2.5 **trong cửa sổ 1h của dashboard** (khác với `daily_cost_usd` 28 ngày trong `config/slo.yaml` — hai ngưỡng khác cửa sổ thời gian, không dùng lẫn) | Data source: **Observations/Generations** · Metric: Cost (sum totalCost) · Dimension: time (line) + 1 Big Number "Total cost" |
| 5 | Input and output tokens | tokens | Last 1h | Tổng ≤ 50.000 tokens | Data source: **Observations/Generations** · Metric: Sum(usageDetails.input), Sum(usageDetails.output) · Chart: Bar/Line 2 series |
| 6 | Quality proxy | score 0–1 | Last 1h | Trung bình ≥ 0.75 | Data source: **Scores**, filter `name = quality` · Metric: Average value · Chart: Line hoặc Big Number |

Evidence: chụp ảnh từng widget (hoặc cả dashboard) sao cho thấy rõ tên panel, time range và ngưỡng trong tiêu đề, lưu vào `submission/evidence/`.
