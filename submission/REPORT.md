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
| `validate_dashboard.py` | 6/6 panel hợp lệ | | Dashboard contract đạt cấu hình chuẩn |
| `pytest` | 22 passed | | Toàn bộ unit/contract tests ban đầu đều pass |
| Số traces hợp lệ | 0 | | Mới chỉ có root span, thiếu child observations (retrieval/generation) và correlation_id là MISSING |
| Số PII leak | 0 | | Chưa phát hiện leak thô từ sample queries, nhưng PII scrubber chưa được gắn vào logging pipeline |
| Latency P95 / TTFT P95 | 5442.0 ms / 50.0 ms | | Chịu ảnh hưởng cold-start ở request đầu tiên trong load test |
| Retrieval success rate | 100% (10/10) | | 100% request retrieval thành công ở trạng thái baseline |

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
- **Cấu trúc root/retrieval/generation observations:**
- **Cách nối trace với log:**
- **Prompt name:**
- **Version/label baseline:**
- **Version/label candidate:**
- **Trace ID của mỗi version:**
- **Cách promote và rollback `production`:**

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
- **SLO và lý do chọn:**
- **Cách tính error budget:**
- **Ba alert và runbook tương ứng:**

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
