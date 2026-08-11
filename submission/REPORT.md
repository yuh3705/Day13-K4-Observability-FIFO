# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: FIFO
- Repository URL: https://github.com/yuh3705/Day13-K4-Observability-FIFO
- Commit SHA cuối: `8ed65d2804543874d5ec23242ed7a12db97f1432`
- Thành viên và vai trò:
  - **Nguyen Mai Huy** — Vai trò A, Tech Lead/Backend Engineer (CP1: middleware, correlation ID, log enrichment, PII).
  - **Bui Minh Long** — Vai trò B, SRE & Alerts Engineer (CP2: Langfuse tracing, prompt versioning, SLO/Alert Rules, runbook).
  - **Nguyen Quang Huy** — Vai trò C, QA & Chief Investigator (Dashboard spec, load test, Challenge CP3, tổng hợp báo cáo).

  Chi tiết từng đầu việc và commit tương ứng: xem mục [7. Đóng góp cá nhân](#7-đóng-góp-cá-nhân).

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 100/100 (78 log records, 0 field thiếu, 0 enrichment thiếu, 33 correlation ID duy nhất)
- Tổng số traces: 62 (kiểm tra qua Langfuse API lúc viết báo cáo; con số tăng dần theo mỗi lần chạy `/chat` hoặc `scripts/load_test.py`)
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard: https://jp.cloud.langfuse.com/project/cmsocv6n7002sad0iijbq5imz/dashboards (dashboard "Day 13 AI Observability")

## 3. Logging và tracing

- Evidence correlation ID: `submission/evidence/correlation_id.png`
- Evidence PII redaction: `submission/evidence/PII_log.png`
- Evidence trace waterfall: `submission/evidence/trace_detail_optional.png` (waterfall `run` → `retrieve` → `generate`, kèm Graph view). *Lưu ý: `submission/evidence/trace_detail.png` là ảnh cũ, chụp trước khi gắn `@observe` lên `retrieve`/`generate` nên chỉ có một span `run` duy nhất — không dùng ảnh này làm evidence waterfall.*
- Giải thích một span đáng chú ý: Span `retrieve` (kiểu `RETRIEVER`, con của `run`) là span quan trọng nhất để chẩn đoán sự cố — nó bọc lệnh gọi `app/mock_rag.py:retrieve()`, nơi hai incident `rag_slow` (thêm `time.sleep(2.5)`) và `tool_fail` (raise `RuntimeError`) được mô phỏng. Vì `retrieve` và `generate` là hai span tách biệt (không còn gộp chung vào một `generation` như instrumentation ban đầu), khi bật `rag_slow` thì đúng span `retrieve` hiển thị latency tăng đột biến (~2.5s) trong khi `generate` không đổi (~0.15s) — đây chính là bằng chứng dùng trong mục 6.

## 4. Prompt versioning

- Prompt name: `day13-chat`
- Version/label baseline: v1 — labels `baseline`, `production` (nội dung: `Feature={{feature}}\nDocs={{docs}}\nQuestion={{message}}`)
- Version/label candidate: v2 — labels `candidate`, `latest` (nội dung: thêm dòng `Answer in at most 2 concise sentences.`)
- Trace ID của mỗi version:
  - v1 / label `baseline`: `1393ad88cf8dfb9ec1c5ec41a455c59f`
  - v2 / label `candidate`: `3eaea44105dba5921a53be422e96d316`
  - v2 khi tạm giữ label `production`: `93c239cc251c354dfb364b6d181c0996`
  - v1 sau khi rollback `production`: `c8af889f3fb6ae5058f43c0dc18bf99d`
- Bằng chứng đổi label hoặc rollback: `submission/evidence/version_before.png` (production đang ở v2: `latest, candidate` | v1 chỉ có `baseline`) → `submission/evidence/version_after.png` (rollback: production quay lại v1: `production, baseline` | v2 còn `latest, candidate`). Đã xác minh khớp với trạng thái thật qua API.

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`: HỢP LỆ — 6/6 panel có trong dashboard contract
- Evidence dashboard: `submission/evidence/dashboard.jpeg` (Latency, Traffic, Error, Cost, Tokens, Quality — công cụ: Langfuse Custom Dashboards, chi tiết từng widget xem [docs/dashboard-spec.md](../docs/dashboard-spec.md))
- SLO đã chọn và lý do (`config/slo.yaml`, window 28 ngày):
  - `latency_p95_ms` ≤ 3000ms (target 99.5%) — ngưỡng chấp nhận được cho một chat response trước khi cảm giác "treo".
  - `error_rate_pct` ≤ 2% (target 99.0%) — hệ thống lab dùng mock LLM/RAG nên lỗi phải hiếm, hầu hết request phải thành công.
  - `daily_cost_usd` ≤ 2.5 (target 100%) — giới hạn ngân sách demo/lab, dễ phát hiện cost spike do incident hoặc bug.
  - `quality_score_avg` ≥ 0.75 (target 95%) — heuristic quality score phải duy trì mức chấp nhận được, không để một prompt/model change làm giảm chất lượng âm thầm.
- Alert rules và runbook (`config/alert_rules.yaml`, chi tiết từng bước điều tra tại [docs/alerts.md](../docs/alerts.md)):
  1. `high_latency_p95` (warning) — `latency_p95 > 3000ms` duy trì 5 phút, owner `on-call-engineer`.
  2. `elevated_error_rate` (critical) — `error_rate_pct > 5%` duy trì 3 phút, owner `on-call-engineer`.
  3. `cost_budget_exceeded` (warning) — `daily_cost_usd > 2.5`, owner `team-lead`.

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
| Nguyen Mai Huy | - Middleware: correlation ID (extract/generate `x-request-id`, response headers)<br>- `bind_contextvars`/`clear_contextvars` để enrich log theo request<br>- Regex PII (`email`, `phone_vn`, `passport`, `address_vn`) và wiring `scrub_text` vào logging pipeline<br>- Generic exception handler trả `x-request-id` khi lỗi | [`4af4b1c`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/4af4b1c) "cp1", [`d2691db`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/d2691db) "pii_test+evidence" | Ban đầu mình quên `clear_contextvars()` ở đầu middleware, log của request trước bị dính sang request sau khi test với concurrency > 1 — mới hiểu context var không tự reset theo request nếu không dọn tay. Viết regex PII cũng dễ chủ quan: số điện thoại `090 123 4567` (có khoảng trắng) từng lọt qua vì mình chỉ test với format liền số, phải quay lại viết test case cho từng kiểu định dạng thật thì mới yên tâm. |
| Bui Minh Long | - Tách waterfall trace `run` → `retrieve` (RETRIEVER) → `generate` (GENERATION) thay vì gộp một span<br>- Sửa `usage_details` dùng key `input`/`output` để Langfuse tính đúng token; gắn `score_current_trace` cho quality<br>- Tạo prompt `day13-chat` v1/v2 trên Langfuse, gắn nhãn `baseline`/`candidate`/`production`, thực hiện đổi label và rollback<br>- Viết SLO (`config/slo.yaml`) và 3 alert rule kèm runbook (`config/alert_rules.yaml`, `docs/alerts.md`) | [`8c84c6b`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/8c84c6b) "Fill in SLOs, symptom-based alert rules, and runbooks", [`1540bdb`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/1540bdb) "Add tracing spans to mock LLM/RAG calls", [`65219a2`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/65219a2) "Add baseline and prompt version before/after evidence" | Bài học đau nhất: gộp cả retrieve + generate vào một span `generation` duy nhất tưởng là gọn, nhưng khi bật incident `rag_slow` thì hoàn toàn không biết chỗ nào chậm — phải tách span ra mới "nhìn" được root cause thay vì đoán. Cũng mất một lúc mới phát hiện Langfuse chỉ nhận diện token input/output nếu tên key chứa đúng chữ `input`/`output`; đặt tên `prompt_tokens`/`completion_tokens` (kiểu OpenAI quen tay) khiến dashboard token cứ hiện 0/0 dù tổng cost vẫn đúng — phải tự soi trace thật qua API mới bắt được, không đoán suông được. |
| Nguyen Quang Huy | - Viết `docs/dashboard-spec.md` (6 nhóm panel: name/unit/time range/threshold/tool)<br>- Dựng 8 widget trên Langfuse Custom Dashboards và gộp thành dashboard "Day 13 AI Observability"<br>- Chạy `scripts/load_test.py` để tạo baseline traffic<br>- Điều tra Challenge CP3 (`rag_slow`), viết mục 6 của báo cáo<br>- Tổng hợp và hoàn thiện `submission/REPORT.md` | [`3965a2f`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/3965a2f) "Update report and observability evidence", [`4c05b26`](https://github.com/yuh3705/Day13-K4-Observability-FIFO/commit/4c05b26) "Add dashboard evidence and update traces screenshot" | Trước giờ mình nghĩ "điều tra incident" là nhìn dashboard thấy số đỏ là xong, nhưng làm CP3 mới thấy error rate 0% không có nghĩa là hệ thống ổn — p95 latency âm thầm vượt SLO trong khi mọi request vẫn trả về 200 OK, chỉ có trace waterfall mới chỉ đúng span nào (`retrieve`) đang gánh 2.5s đó. Cũng học được là dashboard "tốt" không phải cái đẹp mà là cái có threshold/SLO rõ ràng ngay trên panel, để nhìn một cái biết ngay có đang vượt ngưỡng hay không, không phải đoán bằng mắt. |
