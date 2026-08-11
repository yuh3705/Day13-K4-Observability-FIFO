# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: high_latency_p95
- Severity: warning
- SLI/SLO liên quan: latency_p95_ms (objective ≤ 3000ms, target 99.5%)
- Điều kiện và thời gian duy trì: latency_p95 > 3000ms, duy trì liên tục 5 phút
- Ảnh hưởng tới người dùng: Người dùng chờ phản hồi chat lâu hơn bình thường, cảm giác app bị treo/lag.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel "Latency percentiles" để xác nhận p95 vượt ngưỡng và không phải nhiễu từ một request đơn lẻ.
  2. Lọc `data/logs.jsonl` theo `event == "response_sent"` với `latency_ms` cao nhất, lấy `correlation_id` rồi tra trace tương ứng để xem span nào (RAG retrieve hay LLM generate) chiếm phần lớn thời gian.
  3. Kiểm tra panel "Request traffic" xem có tăng đột biến traffic cùng thời điểm không (loại trừ nguyên nhân do tải).
- Mitigation tạm thời: Bật/tăng timeout ngắn hơn cho bước retrieval chậm hoặc tạm chuyển sang model/flow nhanh hơn để giảm tải; thông báo team nếu cần rollback thay đổi gần nhất.
- Owner: on-call-engineer

## Alert 2

- Tên: elevated_error_rate
- Severity: critical
- SLI/SLO liên quan: error_rate_pct (objective < 2%, target 99.0%)
- Điều kiện và thời gian duy trì: error_rate_pct > 5%, duy trì liên tục 3 phút
- Ảnh hưởng tới người dùng: Một phần request trả lỗi, người dùng không nhận được câu trả lời hoặc thấy thông báo lỗi.
- Ba bước kiểm tra đầu tiên:
  1. Mở panel "Error rate and breakdown" để xem `error_type` nào chiếm đa số (vd. tool timeout, LLM exception).
  2. Lọc log theo `event == "request_failed"`, lấy vài `correlation_id` gần nhất và xem trace đầy đủ để xác định span/service gây lỗi.
  3. Kiểm tra xem lỗi có tập trung vào một `feature` hoặc `model` cụ thể không (regression cục bộ) hay lan rộng toàn hệ thống (sự cố hạ tầng).
- Mitigation tạm thời: Nếu lỗi đến từ một tool/dependency cụ thể (vd. vector store timeout), bật fallback hoặc tạm vô hiệu hoá tool đó; nếu do một model/feature cụ thể, rollback về phiên bản trước.
- Owner: on-call-engineer

## Alert 3

- Tên: cost_budget_exceeded
- Severity: warning
- SLI/SLO liên quan: daily_cost_usd (objective ≤ $2.5/ngày, target 100%)
- Điều kiện và thời gian duy trì: daily_cost_usd > 2.5 (tính theo cửa sổ trong ngày, không cần duy trì)
- Ảnh hưởng tới người dùng: Không ảnh hưởng trực tiếp/tức thời tới người dùng, nhưng cảnh báo rủi ro vượt ngân sách vận hành nếu không xử lý.
- Ba bước kiểm tra đầu tiên:
  1. Mở panel "Cost over time" để xem chi phí tăng dần đều hay có một đợt tăng đột biến (spike) tại thời điểm cụ thể.
  2. Đối chiếu với panel "Input and output tokens" để biết chi phí tăng do nhiều request hơn hay do request tốn nhiều token hơn bình thường (vd. prompt dài bất thường).
  3. Lọc log theo `event == "response_sent"` sắp theo `cost_usd` giảm dần để tìm `correlation_id`/`feature` cụ thể gây tốn chi phí nhất.
- Mitigation tạm thời: Giới hạn độ dài prompt/response hoặc tạm chuyển feature tốn kém nhất sang model rẻ hơn cho đến khi xác định nguyên nhân gốc.
- Owner: team-lead
