# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: high_tail_latency
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: `fast_successful_requests` (SLO 99.5% requests hoàn thành trong <= 3000ms trên cửa sổ 28 ngày)
- Điều kiện và thời gian duy trì: `p95_latency_ms > 3000` liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ đợi lâu khi chat (>3 giây), có nguy cơ timeout client và tiêu hao error budget nhanh chóng.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel **Latency percentiles and TTFT** để xem P95/P99 tăng từ thời điểm nào và TTFT có bị tăng tương ứng không.
  2. Tra cứu [data/logs.jsonl](data/logs.jsonl) lọc các dòng `response_sent` có `latency_ms > 3000` để lấy các `correlation_id` đại diện.
  3. Mở Langfuse tìm trace có `correlation_id` đó, xem waterfall để xác định bước chậm nằm ở span `retrieval` (RAG vector DB) hay `llm-generation` (API provider/mô hình).
- Mitigation tạm thời:
  - Nếu do retrieval chậm (ví dụ do incident `rag_slow`): Chuyển tạm thời sang chế độ fallback retrieval hoặc giảm `top_k` documents.
  - Nếu do mô hình LLM bị nghẽn: Chuyển tạm thời sang model dự phòng (fallback model) hoặc tăng timeout client.
- Owner: oncall-engineer

## Alert 2

- Tên: high_error_rate
- Severity: critical
- Duration: 3m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: `fast_successful_requests` (Tỷ lệ lỗi tối đa cho phép là 0.5% theo error budget)
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` liên tục trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận mã lỗi HTTP 500 (`Internal Server Error`), không nhận được câu trả lời chat.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel **Error rate and retrieval success** để xem số lượng lỗi và phân loại lỗi (`count_by_value(error_type)`).
  2. Lọc log sự kiện `request_failed` trong [data/logs.jsonl](data/logs.jsonl) để xem `error_type` (ví dụ `RuntimeError`, `TimeoutError`, v.v.) và lấy `correlation_id`.
  3. Truy vết Langfuse bằng `correlation_id` để kiểm tra span nào bị gãy hoặc fail (ví dụ mock_rag lỗi `tool_fail`).
- Mitigation tạm thời:
  - Nếu do lỗi tích hợp công cụ/retrieval: Tắt tính năng retrieval bị lỗi (`tool_fail`) hoặc chuyển sang trả lời trực tiếp không qua retrieval kèm cảnh báo disclaimer.
  - Nếu do sự cố hệ thống: Restart pod/container dịch vụ API, kiểm tra tình trạng kết nối mạng.
- Owner: oncall-engineer

## Alert 3

- Tên: retrieval_failure_rate
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: Guardrail `retrieval_success_rate_pct_min >= 90%`
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90%` liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Mô hình không nhận được tài liệu bối cảnh liên quan, chất lượng câu trả lời suy giảm, có thể dẫn đến hallucination hoặc thông tin thiếu chính xác.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel **Error rate and retrieval success** trên dashboard, quan sát đường `tool_success_rate_pct`.
  2. Tìm trong log các bản ghi có `tool_name == "retrieval"` và `tool_success == false`.
  3. Xem span `retrieval` trên Langfuse trace để xác định mã lỗi từ vector store / database.
- Mitigation tạm thời:
  - Kiểm tra kết nối dịch vụ retrieval/knowledge base.
  - Chuyển hướng truy vấn sang cụm vector search replica dự phòng hoặc fallback về static knowledge docs.
- Owner: oncall-engineer
