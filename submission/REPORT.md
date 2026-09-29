# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Phan Trọng Hoàn
- **MSSV:** 2A202602954
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/naoh-pt/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps
- **Commit SHA cuối:**
- **Challenge ID:**
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602954`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 (3 FAILED, 1 PASSED) | 100/100 | Đạt toàn bộ sau CP1 (0 thiếu trường, 10 unique correlation IDs, 100% enriched, 0 PII leak) |
| `validate_dashboard.py` | 6/6 panel hợp lệ | 6/6 panel hợp lệ | Dashboard contract thỏa mãn đầy đủ cấu hình chuẩn 6 panel |
| `pytest` | 22 passed | 24 passed | 24/24 tests pass (bổ sung tests CCCD & Credit card) |
| Số traces hợp lệ | 0 | 24 traces | Có đầy đủ cấu trúc root (agent), child 1 (retrieval), child 2 (generation), usage & cost |
| Số PII leak | 0 | 0 | 0 leak, PII đã được scrub sạch trước khi ghi log/trace |
| Latency P95 / TTFT P95 | 5442.0 ms / 50.0 ms | 168.7 ms / 54.0 ms | Tối ưu độ trễ ổn định sau khi cache prompt và warm up model |
| Retrieval success rate | 100% (10/10) | 100% (10/10) | Toàn bộ các lượt tra cứu tri thức đều thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:**
  - `CorrelationIdMiddleware` ([app/middleware.py](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/app/middleware.py)) trước tiên gọi `clear_contextvars()` để xóa sạch ngữ cảnh của request trước, chống rò rỉ dữ liệu giữa các luồng.
  - Trích xuất header `x-request-id` từ request đến; nếu không có hoặc rỗng sẽ sinh mới theo format chuẩn `req-<8-hex>` thông qua `f"req-{uuid.uuid4().hex[:8]}"`.
  - Gọi `bind_contextvars(correlation_id=correlation_id)` của `structlog` để mọi log trong request tự động gắn ID này. Đồng thời lưu vào `request.state.correlation_id` để chuyển tiếp xuống `agent.run()`, trace metadata và trả về qua response header `x-request-id`, `x-response-time-ms` cũng như response body JSON.
- **Các metadata được ghi vào structured log:**
  - Định dạng JSON theo chuẩn ISO-8601 timestamp (`ts`), cấp độ (`level`: info/error/warning), dịch vụ (`service`: api/control), sự kiện (`event`: `request_received`, `response_sent`, `request_failed`).
  - Metadata ngữ cảnh request: `correlation_id`, `env` (môi trường), `user_id_hash` (băm SHA-256 12 ký tự), `session_id`, `feature` (`qa`/`summary`), `model` (`claude-sonnet-4-5`).
  - Metadata hiệu năng và kết quả: `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`, và `payload` (chứa preview rút gọn đã lọc PII).
- **Cách bảo đảm PII được scrub trước khi ghi:**
  - Hoàn thiện regex trong [app/pii.py](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/app/pii.py) cho 4 nhóm thông tin: Email (`[\w\.-]+@[\w\.-]+\.\w+`), Số điện thoại VN (`(?<!\d)(?:\+84|0)(?:[ .-]?\d){9}(?!\d)`), CCCD (`\b\d{12}\b`), và Thẻ thanh toán (`\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b`).
  - Đăng ký processor `scrub_event` trong chuỗi pipeline `structlog.configure` của [app/logging_config.py](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/app/logging_config.py) ngay trước `JsonlFileProcessor` và `JSONRenderer`. Processor này quét đệ quy mọi trường chuỗi/dict/list trong `event_dict` để thay thế thông tin nhạy cảm bằng `[REDACTED_...]` trước khi dữ liệu được ghi vào file `data/logs.jsonl`.
- **Cách kiểm chứng kết quả:**
  - Chạy `python scripts/load_test.py` với 10 queries thực tế chứa PII thô (email, số điện thoại, thẻ tín dụng).
  - Chạy `python scripts/validate_logs.py` đạt điểm tuyệt đối **100/100**:
    + Basic JSON schema: **PASSED** (0 record thiếu trường bắt buộc).
    + Correlation ID propagation: **PASSED** (10/10 unique correlation IDs).
    + Log enrichment: **PASSED** (100% records có đủ user_id_hash, session_id, feature, model, env).
    + PII scrubbing: **PASSED** (0 PII leaks).
  - Chạy `python -m pytest -q` đạt **24/24 passed** (đã bổ sung đầy đủ unit tests cho CCCD và Thẻ thanh toán trong `tests/test_pii.py`).

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  - Toàn bộ traces được đẩy trực tiếp lên project Langfuse Cloud cá nhân mang tên `day13-k4-l3a-2A202602954` (ID: `cmumeaizk024wad0d90gn19r9`) thuộc tổ chức `naoh-pt's Organization` trên host `https://cloud.langfuse.com`.
  - Traces được xác thực qua API Key cá nhân trong `.env` (`pk-lf-64718c48...`).
  - Mọi trace đều mang `environment="dev"`, tag `["lab", feature, "claude-sonnet-4-5"]`, `user_id` đã băm tương ứng với workload chạy từ máy học viên.
- **Cấu trúc root/retrieval/generation observations:**
  - Quan hệ phân cấp cha - con rõ ràng chuẩn OpenTelemetry & Langfuse v4:
    + **Root observation**: `lab-agent-run` (type `AGENT`, capture input/output tắt để bảo mật dữ liệu, chứa metadata `correlation_id`, `feature`, `model`, `prompt_name`, `prompt_version`, `prompt_label`).
    + **Child observation 1**: `retrieval` (type `RETRIEVER`, quan sát bước tra cứu tri thức `retrieve()`, ghi nhận metadata `doc_count`, `query_preview`).
    + **Child observation 2**: `llm-generation` (type `GENERATION`, quan sát lệnh gọi mô hình `llm.generate()`, ghi nhận `model`, `prompt`, `usage_details` với input/output tokens, `cost_details` tính theo đơn giá $3/M input và $15/M output, và `ttft_ms`).
- **Cách nối trace với log:**
  - `correlation_id` được sinh từ middleware (`req-<8-hex>`) được truyền đồng bộ vào `propagate_attributes(metadata={"correlation_id": correlation_id})` của Langfuse trace và ghi vào mọi event trong [data/logs.jsonl](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/data/logs.jsonl).
  - Khi điều tra sự cố, ta tra cứu `correlation_id` từ log line rồi tìm trực tiếp trên Langfuse để mở trace waterfall tương ứng.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (labels: `baseline`, `production`)
- **Version/label candidate:** Version 2 (labels: `candidate`)
- **Trace ID của mỗi version:**
  - Trace ID dùng Version 1 (`baseline`/`production`): `e9e9c74043d5144da5909cf49b651124`
  - Trace ID dùng Version 2 (`candidate` / promoted): `d8380513489743591a5e0a1b25a0bdd2`
  - Trace ID sau khi rollback về Version 1: `655354aec62d283217a3a0920c2e0396`
- **Cách promote và rollback `production`:**
  - **Promote**: Chuyển nhãn `production` sang Version 2 thông qua Langfuse API `lf.update_prompt(name="day13-chat", version=2, new_labels=["candidate", "production"])` (hoặc qua giao diện Langfuse Prompts). Hệ thống sẽ chuyển nhãn `production` trỏ sang v2.
  - **Rollback**: Khi cần hoàn nguyên về Version 1, gán lại nhãn `production` cho Version 1 bằng lệnh `lf.update_prompt(name="day13-chat", version=1, new_labels=["baseline", "production"])`. Ứng dụng tự động nhận diện version 1 khi hết cache TTL mà không cần sửa code hay redeploy.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  - 1. `latency`: Độ trễ P50, P95, P99 và TTFT P95 từ sự kiện `response_sent`. Ngưỡng cảnh báo: P95 <= 3000ms.
  - 2. `traffic`: Lưu lượng request theo phút (`rate_per_minute`) từ sự kiện `request_received`. Ngưỡng: >= 1 req/min.
  - 3. `errors`: Tỷ lệ lỗi (`error_rate_pct`) và tỷ lệ retrieval thành công (`tool_success_rate_pct`) từ `request_received`/`request_failed`. Ngưỡng: error rate <= 2%.
  - 4. `cost`: Chi phí ước tính theo phút và tổng chi phí lũy kế từ `response_sent`. Ngưỡng: total cost <= $2.5.
  - 5. `tokens`: Tổng số lượng token tiêu thụ (input và output) từ `response_sent`. Ngưỡng: sum <= 50,000 tokens.
  - 6. `quality`: Điểm đánh giá chất lượng trung bình từ `response_sent`. Ngưỡng: mean score >= 0.75.
- **SLO và lý do chọn:**
  - Tên SLO: `fast_successful_requests` với mục tiêu **99.5%** trong chu kỳ 28 ngày (28d).
  - SLI: Tỷ lệ request thành công có thời gian phản hồi `latency_ms <= 3000ms` trên tổng số request nhận vào (`request_received`).
  - Lý do chọn: Ứng dụng chat AI yêu cầu phản hồi nhanh (< 3 giây) để giữ trải nghiệm tương tác liền mạch của người dùng và ngăn ngừa client timeout. Ngưỡng 99.5% đảm bảo tính sẵn sàng cao nhưng vẫn cho phép biên độ lỗi thực tế khi có biến động tải mạng.
- **Cách tính error budget:**
  - Error budget = `100% - Target SLO = 100% - 99.5% = 0.5%`.
  - Trong 28 ngày, nếu hệ thống nhận được 100,000 requests thì ngân sách lỗi cho phép tối đa 500 requests bị chậm (> 3000ms) hoặc gặp lỗi 500. Nếu vượt quá con số này, error budget bị cháy, đội ngũ bắt buộc phải dừng release tính năng mới để tập trung cải thiện độ ổn định.
- **Ba alert và runbook tương ứng:**
  - 1. `high_tail_latency` (Severity: Warning, điều kiện `p95_latency_ms > 3000` trong `5m`, kênh Slack `#alerts-llmops-l3a`, runbook: [docs/alerts.md#alert-1](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/docs/alerts.md#alert-1)).
  - 2. `high_error_rate` (Severity: Critical, điều kiện `error_rate_pct > 2%` trong `3m`, kênh Slack `#alerts-llmops-l3a`, runbook: [docs/alerts.md#alert-2](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/docs/alerts.md#alert-2)).
  - 3. `retrieval_failure_rate` (Severity: Warning, điều kiện `retrieval_success_rate_pct < 90%` trong `5m`, kênh Slack `#alerts-llmops-l3a`, runbook: [docs/alerts.md#alert-3](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/docs/alerts.md#alert-3)).

## 7. Điều tra challenge

- **Challenge ID:**
- **Khoảng thời gian điều tra:**
- **Triệu chứng từ metrics:**
- **Log line và correlation ID liên quan:**
- **Trace ID và span gây ảnh hưởng:**
- **Root cause:**
- **Fix action:**
- **Preventive measure:**

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
- **Một lỗi/blocker đã gặp:**
- **Cách tìm nguyên nhân và xử lý:**
- **Cách hiểu luồng Metrics → Logs → Traces:**
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
- **Điều quan trọng nhất đã học:**
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
