# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Trần Đình Hinh
- **MSSV:** 2A202602399
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/hinhtran/K4-L3B-Day13-TranDinhHinh-2A202602399-Monitoring-LLMOps
- **Commit SHA cuối:** `7573eed451e27f754bd7a5801907dccdf58eb227`
- **Challenge ID:** `day13-k4-l3b-practice-rag_slow`
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602399`

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
| `validate_logs.py` | 30/100 | 100/100 | Vượt qua toàn bộ checklist: schema chuẩn, correlation ID, log context và PII scrubbing. |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Hợp lệ 100% contract specs (latency, traffic, errors, cost, tokens, quality). |
| `pytest` | 22 passed | 22 passed | Bộ test suite hoàn tất 100% trong 2.16s không có lỗi. |
| Số traces hợp lệ | 0 | 12 | Đủ cây observation (root trace -> agent run -> retrieval + generation). |
| Số PII leak | 0 | 0 | Không còn PII nguyên văn; toàn bộ email, sđt VN, thẻ ngân hàng, CCCD đều được che. |
| Latency P95 / TTFT P95 | 159ms / 50ms | 159ms / 50ms | Latency P95 ổn định ở mức ~159ms, TTFT 50ms; khi xảy ra sự cố rag_slow đạt 2650ms. |
| Retrieval success rate | 100% | 100% | Đạt tỷ lệ thành công 100% trong điều kiện bình thường, vượt guardrail 90%. |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, middleware xóa context cũ qua `clear_contextvars()`, đọc header `x-request-id` từ client gửi đến hoặc tự sinh theo chuẩn `req-<8-char-hex>` (dùng `uuid.uuid4().hex[:8]`), bind vào structlog contextvars (`bind_contextvars(correlation_id=correlation_id)`), lưu vào `request.state.correlation_id` và trả lại cho client trong header `x-request-id` cùng `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** Mỗi bản ghi log trong `data/logs.jsonl` bao gồm các trường chuẩn: `ts` (ISO format), `level`, `service`, `event`, `correlation_id`, `user_id_hash` (băm SHA256 lấy 12 ký tự), `session_id`, `feature`, `model`, `env`, `payload` (chứa preview an toàn), `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Cấu hình processor `scrub_event` trong `app/logging_config.py` đặt trước `JsonlFileProcessor()` và `JSONRenderer()`. Hàm `_scrub_value` duyệt đệ quy qua toàn bộ dict/list/string trong `event_dict` và dùng regex từ `app/pii.py` để thay thế email, số điện thoại Việt Nam, CCCD 12 số, số thẻ thanh toán thành các token `[REDACTED_...]`.
- **Cách kiểm chứng kết quả:** Chạy script `python scripts/validate_logs.py` đọc toàn bộ `data/logs.jsonl`. Script quét độc lập bằng bộ regex PII detectors và kiểm tra các trường bắt buộc, đạt kết quả tuyệt đối 100/100 (0 missing required, 0 missing enrichment, 0 PII leaks).

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Cấu hình API key kết nối đến project cá nhân `day13-k4-l3b-2A202602399` trên Langfuse Cloud. Mỗi trace được gắn tag môi trường `env=dev`, `tags=['lab', feature, model]`, và trường metadata `correlation_id` khớp chính xác với `data/logs.jsonl`.
- **Cấu trúc root/retrieval/generation observations:**
  - Root observation: `day13-agent-request` (trace-level context với user, session, tags, metadata).
  - Agent observation: `lab-agent-run` (as_type="agent", bọc toàn bộ luồng thực thi của agent).
  - Child observation 1: `retrieval` (as_type="retriever", đo thời gian truy vấn tài liệu RAG).
  - Child observation 2: `generation` (as_type="generation", bọc cuộc gọi LLM generate, ghi nhận model, prompt, usage_details và cost_details).
- **Cách nối trace với log:** Sử dụng trường `correlation_id` duy nhất được gán từ middleware, ghi đồng thời vào log sự kiện `request_received`/`response_sent` và trong metadata của trace trên Langfuse.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 gắn nhãn `baseline` và `production`
- **Version/label candidate:** Version 2 gắn nhãn `candidate`
- **Trace ID của mỗi version:** Trace `req-f1a64b03` cho Version 1 (label production) và trace `req-a32dc114` cho Version 2 (label candidate).
- **Cách promote và rollback `production`:**
  - Promote: Trên Langfuse UI, chuyển nhãn `production` từ Version 1 sang Version 2; app tự động lấy Version 2 qua `LANGFUSE_PROMPT_LABEL=production` mà không cần thay đổi source code.
  - Rollback: Khi phát hiện chất lượng giảm hoặc chi phí tăng, chuyển nhãn `production` quay trở lại Version 1; app tức thì phục hồi prompt an toàn mà không cần downtime hay redeploy.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Dựng đủ 6 panel theo đúng contract `config/dashboard.yaml`:
  1. *Latency*: P50, P95, P99 và TTFT P95 với đường threshold P95 $\le$ 3000ms.
  2. *Traffic*: Request count và rate_per_minute với threshold $\ge$ 1 req/min.
  3. *Errors*: Tỉ lệ lỗi `error_rate_pct` ($\le$ 2%) và tỷ lệ thành công của retrieval `tool_success_rate_pct`.
  4. *Cost*: Tổng chi phí theo phút và tổng cửa sổ thời gian (threshold $\le$ $2.50).
  5. *Tokens*: Tổng input tokens và output tokens (threshold $\le$ 50,000 tokens).
  6. *Quality*: Mean quality proxy score (threshold $\ge$ 0.75).
- **SLO và lý do chọn:** SLO chính `fast_successful_requests`: 99.5% request có `event == "response_sent"` và `latency_ms <= 3000` trong cửa sổ 28 ngày. Lý do chọn: Đáp ứng trải nghiệm tương tác trực tuyến của người dùng (dưới 3s), đồng thời phù hợp với năng lực baseline của hệ thống (P95 đạt ~159ms).
- **Cách tính error budget:** Với target SLO là 99.5%, error budget là $100\% - 99.5\% = 0.5\%$. Trong chu kỳ 28 ngày với giả định 10,000 requests, tối đa 50 requests được phép thất bại hoặc phản hồi chậm hơn ngưỡng 3000ms trước khi vi phạm SLO.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95` (Warning): `p95(latency_ms) > 3000ms` kéo dài 5m $\rightarrow$ Runbook: `docs/alerts.md#alert-1`.
  2. `HighErrorRate` (Critical): `error_rate_pct > 2%` kéo dài 3m $\rightarrow$ Runbook: `docs/alerts.md#alert-2`.
  3. `LowRetrievalSuccess` (Warning): `tool_success_rate_pct < 90%` kéo dài 5m $\rightarrow$ Runbook: `docs/alerts.md#alert-3`.

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-practice-rag_slow`
- **Khoảng thời gian điều tra:** 2026-09-30 03:50:00 UTC – 03:55:00 UTC
- **Triệu chứng từ metrics:** Dashboard panel Latency ghi nhận P95 tăng vọt bất thường từ 159ms lên 2650ms (vượt ngưỡng cảnh báo), trong khi TTFT vẫn giữ nguyên ở mức 50ms và không phát sinh lỗi HTTP 500 (Error Rate = 0%).
- **Log line và correlation ID liên quan:** Lọc `data/logs.jsonl` phát hiện request `correlation_id="req-d72cef59"` có sự kiện `response_sent` với `latency_ms=2650`, `ttft_ms=50`, `tool_name="retrieval"`, `tool_success=true`.
- **Trace ID và span gây ảnh hưởng:** Mở trace `req-d72cef59` trên Langfuse cho thấy span con `retrieval` bị nghẽn mất 2500ms, trong khi span con `generation` chỉ mất 140ms.
- **Root cause:** Module Vector Store / RAG bị suy giảm hiệu năng nghiêm trọng (mô phỏng bởi kịch bản `rag_slow` gây delay 2.5s trong hàm `retrieve()`), dẫn tới toàn bộ thời gian phản hồi của request bị kéo dài, dù LLM sinh token vẫn nhanh bình thường.
- **Fix action:** Tắt sự cố mô phỏng qua lệnh `python scripts/inject_incident.py --scenario rag_slow --disable`, giải phóng tài nguyên và kiểm tra lại connection pool của Vector Store.
- **Preventive measure:** Bật alert `HighLatencyP95` để phát hiện suy thoái sớm; cấu hình client timeout tối đa 1.5s cho bước vector retrieval kèm cơ chế fallback trả tài liệu mặc định nếu quá hạn để đảm bảo request không bao giờ bị nghẽn quá lâu.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Triển khai PII scrubbing ở cấp độ logging processor đệ quy trong Structlog thay vì che thủ công ở từng hàm controller. Lý do: Giúp che chắn tập trung, không phụ thuộc vào việc lập trình viên có nhớ scrub hay không, đồng thời đảm bảo dữ liệu nhạy cảm được làm sạch ngay trước khi ghi xuống đĩa hoặc gửi qua network.
- **Một lỗi/blocker đã gặp:** Ban đầu khi chạy baseline, điểm `validate_logs.py` chỉ đạt 30/100 do `CorrelationIdMiddleware` chưa trích xuất `x-request-id` và chưa bind contextvars, dẫn đến thiếu correlation ID và các trường ngữ cảnh `user_id_hash`, `session_id`, `model`, `feature`.
- **Cách tìm nguyên nhân và xử lý:** Đọc mã nguồn `scripts/validate_logs.py` để hiểu rõ các trường bắt buộc; hoàn thiện logic trích xuất/sinh ID `req-<8-hex>` trong middleware, gọi `bind_contextvars` trong middleware và `main.py`, sau đó xóa log cũ và đo lại đạt điểm tối đa 100/100.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - *Metrics* là chỉ báo tầm cao giúp phát hiện triệu chứng và khoanh vùng khung giờ xảy ra sự cố.
  - *Logs* thu hẹp phạm vi để xác định chính xác request nào bị lỗi/chậm thông qua `correlation_id`.
  - *Traces* phân rã toàn bộ tiến trình của request đó thành từng span độc lập để định vị chính xác bước/hàm nào là nguyên nhân gốc rễ.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt trong LLMOps đóng vai trò như mã nguồn cấu hình; một thay đổi nhỏ có thể làm bùng nổ token, chi phí và latency. Quản lý prompt có version và nhãn `production` cho phép đội ngũ vận hành phản ứng nhanh: khi phiên bản mới có dấu hiệu bất thường, việc rollback về phiên bản cũ diễn ra trong vài giây mà không cần deploy lại toàn bộ ứng dụng.
- **Điều quan trọng nhất đã học:** Kỹ năng phân tích và xử lý sự cố có hệ thống dựa trên bằng chứng định lượng (Observability Data-Driven) thay vì phỏng đoán mò mẫm.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Bài lab hiện đang sử dụng Mock LLM và Mock RAG local để mô phỏng sự cố; trong môi trường production thực tế, cần tích hợp thêm OpenTelemetry collector và kết nối trực tiếp với LLM provider/Vector DB chuyên dụng.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
