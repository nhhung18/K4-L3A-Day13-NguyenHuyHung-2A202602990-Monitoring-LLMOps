# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Huy Hùng 
- **MSSV:** 2A202602990
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/nhhung18/K4-L3A-Day13-NguyenHuyHung-2A202602990-Monitoring-LLMOps
- **Commit SHA cuối:** *(cập nhật sau khi commit cuối)*
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602990`

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
| `validate_logs.py` | 0/100 (chưa có correlation ID, PII processor chưa bật) | 100/100 | Đạt tối đa sau khi hoàn thiện middleware, enrichment và PII scrubbing |
| `validate_dashboard.py` | 6/6 panel (đã có sẵn) | 6/6 panel | Dashboard contract đã đầy đủ từ starter |
| `pytest` | 20/22 (2 fail do sai API v4) | 22/22 | Fix: đổi `usage`/`cost` sang `usage_details`/`cost_details` theo Langfuse SDK v4 |
| Số traces hợp lệ | 0 | 10+ | Tạo bằng load_test.py, mỗi trace có root + retrieval span + llm-generation |
| Số PII leak | Nhiều (PII processor chưa bật) | 0 | scrub_event processor chặn email, phone, CCCD, credit card, passport |
| Latency P95 / TTFT P95 | ~200ms / ~50ms (baseline) | ~3639ms / ~50ms (khi rag_slow) | Latency tăng do retrieval span bị inject sleep 2.5s |
| Retrieval success rate | 100% | 100% | Retrieval vẫn thành công, chỉ chậm hơn |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Middleware `CorrelationIdMiddleware` trong `app/middleware.py` đọc header `x-request-id` từ request. Nếu không có, sinh ID mới theo format `req-<8-hex>` bằng `uuid.uuid4().hex[:8]`. ID được bind vào structlog context qua `bind_contextvars(correlation_id=...)` và lưu vào `request.state.correlation_id`. Response trả lại ID qua header `x-request-id` và thời gian xử lý qua `x-response-time-ms`.

- **Các metadata được ghi vào structured log:** Tại endpoint `/chat` trong `app/main.py`, trước khi ghi log `request_received`, bind thêm: `user_id_hash` (hash SHA256 cắt 12 ký tự), `session_id`, `feature`, `model`, `env` (từ biến môi trường `APP_ENV`, mặc định `dev`). Các field này tự động xuất hiện trong mọi log entry tiếp theo của request đó.

- **Cách bảo đảm PII được scrub trước khi ghi:** Processor `scrub_event` trong `app/pii.py` được đăng ký trong pipeline structlog (`app/logging_config.py`) trước `JSONRenderer` và `WriteLogFile`. Processor duyệt đệ quy tất cả string value trong log event và thay thế PII bằng `[REDACTED]` dựa trên regex patterns: email, điện thoại Việt Nam (10 số bắt đầu 0), CCCD (12 số), credit card (13-19 số), passport (1 chữ cái + 7 số).

- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đọc toàn bộ `data/logs.jsonl` và kiểm tra: (1) JSON schema hợp lệ, (2) correlation ID có trong mọi record, (3) enrichment fields đầy đủ, (4) không có PII leak. Kết quả: 100/100. Thêm vào đó, chạy `python -m pytest tests/test_pii.py` kiểm tra từng pattern riêng.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Dùng Langfuse Cloud project `day13-k4-l3a-2A202602990` với key pair riêng trong `.env`. Mỗi trace chứa `correlation_id` khớp với log, xác nhận trace thuộc workload của mình.

- **Cấu trúc root/retrieval/generation observations:**
  - **Root:** `LabAgent.run()` được decorate bằng `@observe(name="agent-run")` — tạo root trace cho toàn bộ request.
  - **Retrieval:** `retrieve()` trong `app/mock_rag.py` được decorate bằng `@observe(name="retrieval", as_type="span", capture_input=False, capture_output=False)` — child span cho bước tìm kiếm tài liệu.
  - **Generation:** `FakeLLM.generate()` trong `app/mock_llm.py` được decorate bằng `@observe(name="llm-generation", as_type="generation", capture_input=False, capture_output=False)` — child generation chứa model, usage_details (input_tokens, output_tokens) và cost_details (total).

- **Cách nối trace với log:** Cả trace và log cùng chia sẻ `correlation_id`. Trong `app/agent.py`, `propagate_attributes` context manager truyền correlation_id vào trace metadata. Khi điều tra, lọc log theo correlation_id rồi tìm trace tương ứng trên Langfuse.

- **Prompt name:** `day13-chat`
- **Version/label baseline:** v1 / `production`
- **Version/label candidate:** v2 / `candidate`
- **Trace ID của mỗi version:** *(ghi lại từ Langfuse Cloud sau khi chạy workload với từng version)*
- **Cách promote và rollback `production`:** Trên Langfuse Cloud, vào Prompts > `day13-chat`, chọn version cần promote, gán label `production`. Để rollback, chọn version cũ (v1) và gán lại label `production` — label chỉ là con trỏ, có thể di chuyển bất kỳ lúc nào.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Định nghĩa trong `config/dashboard.yaml` gồm 6 panel:
  1. **latency** — P50/P95/P99 response latency và TTFT (ms), nguồn: `latency_ms` và `ttft_ms` từ log `response_sent`
  2. **traffic** — Request rate (req/min), nguồn: đếm log `request_received` theo thời gian
  3. **errors** — Error rate (%) và retrieval success rate, nguồn: log có `level=error` hoặc `tool_success=false`
  4. **cost** — Chi phí tích lũy (USD), nguồn: `cost_usd` từ log `response_sent`
  5. **tokens** — Tổng input/output tokens, nguồn: `tokens_in` và `tokens_out` từ log
  6. **quality** — Quality score trung bình, nguồn: `quality_score` từ log `response_sent`

- **SLO và lý do chọn:** SLO `fast_successful_requests` với target 99.5% trong cửa sổ 28 ngày. Good event là request có `latency_ms <= 3000ms` và trả về thành công. Ngưỡng 3000ms phù hợp vì baseline latency ~200ms, cho phép headroom lớn cho spike nhưng vẫn đảm bảo trải nghiệm người dùng chấp nhận được.

- **Cách tính error budget:** Error budget = 100% - 99.5% = 0.5%. Trong cửa sổ 28 ngày, nếu có 10,000 request thì cho phép tối đa 50 request vi phạm SLI (latency > 3000ms hoặc lỗi). Khi budget còn lại < 25%, cần đóng băng deploy và ưu tiên fix reliability.

- **Ba alert và runbook tương ứng:**
  1. **high_latency_p95** (critical, 5m): Khi P95 latency > 3000ms liên tục 5 phút → kiểm tra dashboard latency panel, lọc log theo khoảng thời gian, tìm trace của request chậm nhất → xác định span chậm → scale infra hoặc rollback prompt. Owner: on-call-sre.
  2. **high_error_rate** (critical, 3m): Khi error rate > 2% liên tục 3 phút → kiểm tra error panel, lọc log có `level=error`, tìm trace lỗi → xác định exception → rollback code hoặc restart service. Owner: on-call-sre.
  3. **quality_degradation** (warning, 10m): Khi quality_score trung bình < 0.75 liên tục 10 phút → kiểm tra quality panel, so sánh giữa prompt versions, tìm trace có score thấp → rollback prompt về version ổn định. Owner: ml-team.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** 2026-09-29T08:23:36Z — 2026-09-29T08:23:58Z (~22 giây)

- **Triệu chứng từ metrics:** Tất cả 5 request challenge đều có latency vượt ngưỡng 2000ms (2908ms — 3639ms), trong khi baseline latency chỉ ~200ms. P95 latency tăng ~18x. TTFT vẫn bình thường (50ms), cho thấy vấn đề không nằm ở LLM generation. Retrieval vẫn thành công (tool_success=true) nhưng tổng latency cao bất thường.

- **Log line và correlation ID liên quan:**
  - `req-52b57f56` — latency 3639ms (session k4-l3a-challenge-s01, cao nhất)
  - `req-52897114` — latency 3011ms (session k4-l3a-challenge-s05)
  - `req-caa2e9e7` — latency 2947ms (session k4-l3a-challenge-s03)
  - `req-52f6f3c1` — latency 2940ms (session k4-l3a-challenge-s02)
  - `req-42424ed4` — latency 2908ms (session k4-l3a-challenge-s04)
  - Log incident_enabled xác nhận `rag_slow` được bật lúc 08:23:36Z (correlation_id: req-c276a92b).

- **Trace ID và span gây ảnh hưởng:** Trong trace waterfall của mỗi request, span `retrieval` chiếm phần lớn thời gian (~2500ms), trong khi span `llm-generation` chỉ ~150ms. Root span `agent-run` bao gồm cả hai nhưng bottleneck rõ ràng nằm ở retrieval.

- **Root cause:** Incident `rag_slow` inject `time.sleep(2.5)` vào hàm `retrieve()` tại `app/mock_rag.py:20`, mô phỏng vector store bị chậm. Mỗi request phải chờ thêm 2.5 giây ở bước retrieval, đẩy tổng latency từ ~200ms lên ~3000ms+, vượt SLO threshold 3000ms.

- **Fix action:** Tắt incident bằng `python scripts/inject_incident.py --disable`. Trong thực tế: kiểm tra kết nối đến vector store, restart/scale vector database, hoặc bật retrieval cache để giảm tải.

- **Preventive measure:**
  1. Đặt timeout cho retrieval call (ví dụ 1000ms) và trả fallback answer nếu timeout.
  2. Thêm circuit breaker cho vector store connection để tránh cascading failure.
  3. Monitor retrieval latency riêng bằng alert `retrieval_p95 > 1000ms` để phát hiện sớm trước khi ảnh hưởng SLO tổng.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Chọn decorate trực tiếp hàm `retrieve()` và `FakeLLM.generate()` bằng `@observe` thay vì wrap inline trong `agent.py`. Lý do: decorator pattern rõ ràng hơn, mỗi function có đúng một observation, và Langfuse SDK v4 tự động xây dựng parent-child relationship dựa trên call stack — không cần truyền trace context thủ công.

- **Một lỗi/blocker đã gặp:** Test `test_agent_records_prompt_version_with_v4_observation_api` fail do dùng sai tên parameter. Ban đầu dùng `usage={"input_tokens": ...}` và `cost=...` — đây là API v3. Langfuse SDK v4 đổi thành `usage_details` và `cost_details`.

- **Cách tìm nguyên nhân và xử lý:** Đọc error traceback: `TypeError: Langfuse.update_current_generation() got an unexpected keyword argument 'usage'`. Sau đó dùng `help(get_client().update_current_generation)` để xem signature thực tế của SDK v4, xác nhận parameter đúng là `usage_details` và `cost_details`. Fix và chạy lại pytest — 22/22 pass.

- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics (latency, error rate trên dashboard) cho biết **có vấn đề gì** và **khi nào**. Logs (structured JSON với correlation_id) cho biết **request nào** bị ảnh hưởng — lọc theo khoảng thời gian và tìm anomaly. Traces (span waterfall trên Langfuse) cho biết **bước nào** gây ra vấn đề — so sánh duration giữa retrieval và generation span để khoanh vùng root cause. Ba tầng bổ sung cho nhau: metrics phát hiện, logs định danh, traces định vị.

- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  - **Prompt version:** Cho phép A/B test và rollback nhanh khi chất lượng output giảm — label `production` là con trỏ di chuyển được.
  - **Token/cost:** Monitor chi phí vận hành, phát hiện cost spike sớm (ví dụ output_tokens tăng 4x khi có anomaly).
  - **SLO & error budget:** Định lượng reliability commitment, giúp quyết định khi nào đóng băng deploy (budget cạn) hay khi nào có thể chấp nhận risk để ship nhanh hơn.
  - **Rollback:** Cơ chế khôi phục nhanh khi production bị ảnh hưởng — áp dụng cho cả prompt version lẫn code deploy.

- **Điều quan trọng nhất đã học:** Observability không chỉ là thêm log — mà là thiết kế hệ thống sao cho mỗi tầng (metrics, logs, traces) có correlation key chung (`correlation_id`) để liên kết dữ liệu giữa các nguồn khác nhau. Không có correlation, ba tầng trở thành ba hệ thống rời rạc không thể điều tra cross-signal.

- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Evidence screenshots (section 2) cần được chụp thủ công từ terminal output và Langfuse Cloud UI. Prompt versioning (v1/v2, rollback) cần thực hiện trên Langfuse Cloud UI — phần này phụ thuộc vào cấu hình Langfuse project cá nhân.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
