# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Latency P95 của `response_sent.latency_ms` <= 3000ms
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` duy trì liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Trải nghiệm người dùng suy giảm nghiêm trọng, thời gian phản hồi câu trả lời quá chậm
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel **Latency** để xác nhận P50/P95/P99 và khoảng thời gian latency bắt đầu tăng vọt.
  2. Lọc file log `data/logs.jsonl` trong khung giờ đó, lấy ra các `correlation_id` có `latency_ms > 3000`.
  3. Mở trace có `correlation_id` đó trên Langfuse, đối chiếu thời gian của span `retrieval` và span `generation` để xác định bước gây nghẽn.
- Mitigation tạm thời: Nếu do retrieval quá tải hoặc RAG mock lag, restart/giảm tải; nếu do model/prompt, rollback prompt label `production` về version ổn định trước đó.
- Owner: `team-oncall`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ lỗi tổng thể `error_rate_pct` <= 2%
- Điều kiện và thời gian duy trì: `count(request_failed) / count(request_received) * 100 > 2%` trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận mã lỗi HTTP 500 khi gửi câu hỏi vào API chat, làm gián đoạn dịch vụ
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel **Errors** để kiểm tra tỉ lệ lỗi và phân bố theo `error_type` (ví dụ: `RuntimeError`, `TimeoutError`).
  2. Tra cứu `event == "request_failed"` trong `data/logs.jsonl`, trích xuất `correlation_id` và trường `payload.detail`.
  3. Mở trace lỗi tương ứng trên Langfuse để xem exception stack trace tại span nào (`retrieval` hay `generation`).
- Mitigation tạm thời: Kiểm tra trạng thái incident inject (`/health`), tắt incident giả lập nếu đang diễn ra (`/incidents/{name}/disable`) hoặc chuyển hướng traffic sang backup instance.
- Owner: `team-oncall`

## Alert 3

- Tên: `LowRetrievalSuccess`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ thành công của Retrieval `retrieval_success_rate_pct` >= 90%
- Điều kiện và thời gian duy trì: `tool_success_rate_pct < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: RAG không trích xuất được tài liệu phù hợp, dẫn đến câu trả lời thiếu ngữ cảnh hoặc bị fallback
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel **Errors** kiểm tra metric `tool_success_rate_pct` và số lượt thất bại của `tool_name="retrieval"`.
  2. Lọc log có `tool_name == "retrieval"` và `tool_success == false` trong `data/logs.jsonl` để lấy `correlation_id`.
  3. Mở trace trên Langfuse, kiểm tra input query và lỗi timeout/kết nối của vector store trong span retrieval.
- Mitigation tạm thời: Kiểm tra kết nối tới Vector DB, kích hoạt fallback bộ nhớ đệm (cached context) và khởi động lại vector store service.
- Owner: `team-oncall`
