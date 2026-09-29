# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Phan Trọng Hoàn
- **MSSV:** 2A202602954
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/naoh-pt/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps
- **Commit SHA cuối:** `8be59da`
- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
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

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1` (Cohort: K4, Seed: 1311, Incident: `rag_slow`, Affected Feature: `monitoring`, Latency Threshold: 2000ms)
- **Khoảng thời gian điều tra:** `2026-09-29T10:12:31Z` đến `2026-09-29T10:13:17Z`
- **Triệu chứng từ metrics:**
  - P95 latency của hệ thống tăng vọt từ mức bình thường ~165ms lên tới **2,656ms** (thời gian xử lý nội bộ server) và **10,684ms – 13,357ms** (đo từ phía client khi chạy đồng thời với concurrency = 5).
  - Vi phạm nghiêm trọng ngưỡng độ trễ challenge quy định (`latency_threshold_ms: 2000`) và mục tiêu SLO (`3000ms`).
  - Trong khi đó, tỷ lệ lỗi vẫn giữ 0% (error rate = 0%) và điểm chất lượng vẫn đạt ~0.80–0.90, chứng tỏ hệ thống không bị crash mà bị nghẽn hiệu năng nghiêm trọng cục bộ trên tính năng `monitoring`.
- **Log line và correlation ID liên quan:**
  - Correlation ID đại diện được chọn để đối chiếu xuyên suốt: `req-f49a0b74` (User hash: `4570299f37e2`, Session: `k4-l3a-challenge-s04`, Feature: `monitoring`).
  - Request nhận lúc: `2026-09-29T10:12:42.410132Z`
  - Response gửi lúc: `2026-09-29T10:12:45.067789Z` (latency: 2655ms)
  - Log line trích xuất từ `data/logs.jsonl`:
    ```json
    {"service": "api", "latency_ms": 2655, "ttft_ms": 50, "tokens_in": 36, "tokens_out": 110, "cost_usd": 0.001758, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "payload": {"answer_preview": "Starter answer. You should improve this output logic and add better quality chec..."}, "event": "response_sent", "env": "dev", "correlation_id": "req-f49a0b74", "session_id": "k4-l3a-challenge-s04", "user_id_hash": "4570299f37e2", "model": "claude-sonnet-4-5", "feature": "monitoring", "level": "info", "ts": "2026-09-29T10:12:45.067789Z"}
    ```
- **Trace ID và span gây ảnh hưởng:**
  - Trace ID tương ứng trên Langfuse Cloud: `2894f380a141bb94edc29b1482fa2f35` (chứa metadata `correlation_id: req-f49a0b74`).
  - Phân rã thời gian thực thi của các span (Span timing breakdown):
    + Root span `lab-agent-run`: **2.65s** (100% thời gian agent)
    + Child span `retrieval`: **2.50s** (chiếm tới **94.3%** tổng thời gian thực thi của trace!)
    + Child span `llm-generation`: **0.15s** (chỉ chiếm 5.7%, tốc độ sinh token của LLM hoàn toàn bình thường)
  - Span gây chậm trực tiếp chính là child span `retrieval` (bước tra cứu RAG vector/tài liệu).
- **Root cause:**
  - Sự cố mô phỏng `rag_slow` đã can thiệp vào tầng tri thức/retriever, gây ra độ trễ nhân tạo 2.5 giây cho mỗi lần gọi hàm `retrieve()` đối với feature `monitoring`. Khi có tải đồng thời (5 concurrent requests), hàng đợi xử lý bị dồn ứ khiến độ trễ tổng thể từ phía client tăng vọt lên hơn 13 giây.
- **Fix action:**
  - Vô hiệu hóa ngay sự cố bằng lệnh `python scripts/inject_incident.py --disable` (gọi endpoint `POST /incidents/rag_slow/disable`), hệ thống phục hồi latency về mức bình thường ~165ms.
  - Trong môi trường production thực tế: Tối ưu chỉ mục vector (HNSW index), mở rộng connection pool tới cơ sở dữ liệu vector/tri thức, áp dụng semantic caching cho các truy vấn tra cứu lặp lại.
- **Preventive measure:**
  - Cấu hình hard timeout cho retrieval span (ví dụ `timeout=1500ms`) với cơ chế fallback trả về tài liệu tĩnh hoặc thông báo tra cứu suy giảm thay vì để request bị block vô thời hạn.
  - Thiết lập alert `high_tail_latency` (P95 > 3000ms trong 5m) và `retrieval_failure_rate` (< 90%) gửi trực tiếp về kênh trực ban Slack `#alerts-llmops-l3a` kèm theo runbook xử lý sự cố tại [docs/alerts.md](file:///e:/AIinActoin/K4-L3-DAY13-PhanTrongHoan-2A202602954-Monitoring-LLMOps/docs/alerts.md).

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
  - Thiết kế kiến trúc quan sát phân cấp (hierarchical tracing) tương thích chuẩn Langfuse v4 và OpenTelemetry: sử dụng context manager `start_observation` kết hợp với `propagate_attributes`.
  - Lý do: Tách biệt rõ ràng giữa bước `retrieval` (type `RETRIEVER`) và bước `llm-generation` (type `GENERATION`). Nhờ đó, khi có sự cố hiệu năng, ta lập tức cô lập được thời gian nghẽn xảy ra ở tầng cơ sở dữ liệu/RAG hay ở tầng gọi mô hình LLM mà không phải phỏng đoán.
- **Một lỗi/blocker đã gặp:**
  - Khi bắt đầu CP2, quá trình gửi trace và fetch prompt gặp lỗi xác thực 401 do cấu hình nhầm `LANGFUSE_BASE_URL` trỏ về máy chủ US (`https://us.cloud.langfuse.com`) trong khi tài khoản và project được tạo trên máy chủ EU (`https://cloud.langfuse.com`).
- **Cách tìm nguyên nhân và xử lý:**
  - Kiểm tra log chi tiết từ API response và mã lỗi HTTP 401, đối chiếu URL trong dashboard Langfuse trên trình duyệt và nhận thấy prefix tổ chức nằm trên cụm EU. Xử lý bằng cách cập nhật biến môi trường `LANGFUSE_BASE_URL=https://cloud.langfuse.com` trong file `.env`, tạo lại API key hợp lệ và khởi động lại uvicorn server.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - **Metrics** là tín hiệu phát hiện đầu tiên (*Symptom Detection*): cho biết *hệ thống có đang gặp vấn đề không, vấn đề gì (latency/error), và bắt đầu từ thời điểm nào*.
  - **Logs** là công cụ lọc và khoanh vùng (*Event Correlation*): dựa vào timestamp và triệu chứng từ metrics, ta tra cứu các dòng log bất thường, xác định `user_id_hash`, `session_id`, `feature`, và đặc biệt là lấy được mã định danh duy nhất `correlation_id` (`req-...`).
  - **Traces** là công cụ chẩn đoán sâu và xác định nguyên nhân gốc rễ (*Root Cause Diagnosis*): dùng `correlation_id` để mở cây thác trace (waterfall tree), xem chi tiết thời gian và trạng thái của từng span con (`retrieval`, `generation`) để chỉ ra đích xác hàm hay dịch vụ bên dưới đang là điểm nghẽn.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  - **Prompt version & rollback**: Prompt là thành phần thường xuyên thay đổi nhất trong ứng dụng GenAI. Việc quản lý phiên bản (v1, v2) và gán nhãn môi trường (`production`, `candidate`) cho phép A/B testing an toàn và thu hồi ngay lập tức (zero-downtime rollback) khi prompt mới gây hallucination hoặc suy giảm chất lượng mà không cần sửa mã nguồn hay build lại container.
  - **Token & cost**: Giúp kiểm soát chi phí API theo thời gian thực, phát hiện rò rỉ token (ví dụ prompt quá dài hoặc vòng lặp vô tận) trước khi ngân sách bị cạn kiệt.
  - **SLO & error budget**: Đặt ra ranh giới định lượng giữa tốc độ phát triển tính năng và độ ổn định hệ thống. Khi error budget bị cạn, đội ngũ kỹ thuật buộc phải dừng deploy để khắc phục nợ kỹ thuật.
- **Điều quan trọng nhất đã học:**
  - Khắc sâu tư duy quan sát toàn diện (*Observability-driven engineering*) cho hệ thống AI: không chỉ log input/output đơn thuần mà phải bọc lót bảo vệ dữ liệu nhạy cảm (PII redaction), gắn kết ngữ cảnh xuyên suốt bằng correlation ID, và phân tách chi tiết từng giai đoạn thực thi trong pipeline LLM.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**
  - Cơ chế retry khi gọi API vector DB / LLM hiện tại vẫn là logic cơ bản, có thể nâng cấp thêm exponential backoff và circuit breaker để tự động ngắt tải khi downstream service bị nghẽn kéo dài.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
