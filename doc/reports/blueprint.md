# Blueprint CI/CD cho RAG và guardrails

Nhóm: Duo-Eval-Guard

Người phụ trách: Nguyễn Việt Hoàng Hải - 2A202602967

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T20:07:39.192573+00:00; guard models: ["gpt-4o-mini"].
Models generation/evaluator/judge từ A/B: generation: gpt-4o-mini (artifacts/answers_50q.json); evaluator: gpt-4o-mini, text-embedding-ada-002 (artifacts/ragas_50q.json); judge: gpt-4o-mini (artifacts/judge_results.json); guard: gpt-4o-mini (artifacts/guard_results.json).

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Kiến trúc bảo vệ đa tầng theo mô hình defense-in-depth:
1. Data flow trong production:
   - Bước 1 (Presidio PII Layer): Nhận user input thô. Chạy đồng bộ qua regex và pattern recognizers (VN_CCCD, VN_PHONE, EMAIL, PHONE_NUMBER). Nếu phát hiện PII nhạy cảm, chặn ngay lập tức (short-circuit) và trả về thông báo từ chối. Nếu an toàn, dữ liệu được chuyển tiếp sang bước 2.
   - Bước 2 (NeMo Input Rail): Chạy bất đồng bộ qua LLM guardrail (gpt-4o-mini với prompt self_check_input). Kiểm tra prompt injection, jailbreak, off-topic và yêu cầu thông tin mật. Nếu vi phạm, dừng luồng (short-circuit) và trả về câu từ chối chuẩn hóa.
   - Bước 3 (RAG Pipeline - DAY18): Nếu input hợp lệ, query được chuyển vào Search Engine (hybrid BM25 + dense Qdrant), rerank bằng FlashRank, và tổng hợp câu trả lời qua LLM generator (gpt-4o-mini).
   - Bước 4 (NeMo Output Rail): LLM output được kiểm tra đối chiếu qua NeMo self_check_output trong ngữ cảnh hội thoại để ngăn chặn rò rỉ dữ liệu nhạy cảm hoặc mật khẩu ngoài ý muốn. Nếu bị cờ vi phạm, câu trả lời bị redact hoặc thay thế hoàn toàn bằng câu từ chối an toàn.
2. Failure actions và Fallback:
   - Khi Presidio hoặc NeMo Input Rail kích hoạt chặn: Hệ thống trả mã lỗi nghiệp vụ 400 kèm câu thông báo thân thiện: "Xin lỗi, tôi không thể thực hiện yêu cầu này. Tôi chỉ hỗ trợ chính sách nội bộ của công ty...".
   - Khi LLM API của Guardrails gặp sự cố timeout (quá 2000ms) hoặc mạng gián đoạn: Hệ thống kích hoạt fail-closed đối với các request nghi vấn hoặc fallback sang regex rule-based engine để đảm bảo tính an toàn tối đa.
3. Logging và Privacy:
   - Các query bị chặn được log vào hệ thống giám sát tập trung với PII đã được ẩn danh (anonymized) bởi Presidio. Tuyệt đối không lưu CCCD, số điện thoại hoặc mật khẩu người dùng ở dạng plain-text trong application log.
   - Quản lý Secrets: Toàn bộ API keys (OPENAI_API_KEY) được quản lý qua AWS Secrets Manager / Vault hoặc GitHub Actions Secrets được inject dưới dạng runtime environment variables, tuyệt đối không hardcode trong mã nguồn hoặc log.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 24.85 | 30.27 | 30.79 | Đạt ngân sách riêng lẻ (<50ms) |
| NeMo input | 599.29 | 688.96 | 696.17 | Vượt ngân sách do mạng gọi OpenAI API |
| Input guard sequential | 619.59 | 716.24 | 719.93 | Vượt ngân sách target 500ms |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: 716.24ms / False.
Phân tích bottleneck:
- Presidio xử lý rất nhanh (P50 = 24.85ms, P95 = 30.27ms), chiếm chưa tới 5% tổng thời gian bảo vệ.
- NeMo input rail là điểm nghẽn chính (P50 = 599.29ms, P95 = 688.96ms) do phải thực hiện một lượt gọi mạng HTTPS từ client đến OpenAI API endpoint (gpt-4o-mini) với RTT quốc tế trung bình 500-650ms.
- Đối với các phạm vi "chưa đo" (RAG latency, NeMo output, Full guard short-circuit): Kế hoạch đo thực tế trong tương lai là thiết lập benchmark suite sử dụng 50 câu hỏi chuẩn, thực hiện 5 lượt warm-up, ghi nhận timestamp độ chính xác cao `perf_counter` tại từng entry/exit middleware, và tổng hợp số liệu percentile P50/P95/P99.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.
Đề xuất cấu hình CI Workflow (.github/workflows/rag_eval_guard_gate.yml - chưa triển khai, chỉ là đề xuất thiết kế):
- Lệnh chạy kiểm thử và đánh giá:
  1. `pytest tests/ -m "not integration" -q` (Unit tests gate: đảm bảo 100% contracts và logic guardrails không lỗi hồi quy).
  2. `python scripts/run_experiments.py --phase all --live --latency-samples 20` (Live experiment runner trên staging environment).
  3. `python scripts/check_lab.py --with-tests` (Quality gate tổng quát: kiểm tra ngưỡng pass của tất cả artifacts).
- Cơ chế chặn (Fail-stop):
  - Script kiểm tra trả về exit code 1 (non-zero) khi:
    + RAGAS faithfulness < 0.75 hoặc avg_score < 0.65.
    + Tỷ lệ chặn tấn công Adversarial < 18/20 (< 90%).
    + Tỷ lệ chặn nhầm (False Positive Rate) trên bộ câu hỏi sạch > 0.0 (phải đạt 8/8 clean allowed).
    + Phát hiện bất kỳ ca output nhạy cảm nào bị lọt lưới trong output cases.
  - Khi script trả về exit code 1, GitHub Actions tự động đánh dấu build FAIL, chặn thao tác merge pull request vào nhánh main/production.

## Monitoring và cải tiến

Hệ thống giám sát vận hành production (Proposed Observability Stack):
- Metric cần theo dõi liên tục:
  1. Guardrail Latency (P50, P95, P99) với ngưỡng cảnh báo P95 > 800ms.
  2. Tỷ lệ chặn (Block Rate) theo từng tầng (Presidio vs NeMo Input vs Output Rail) với ngưỡng cảnh báo đột biến > 15% tổng request.
  3. Tỷ lệ phản hồi từ người dùng (Negative Feedback / Thumbs down) với ngưỡng cảnh báo > 5%.
  4. Token consumption và chi phí API OpenAI hàng ngày.
- Xử lý False Positives và Data Drift:
  - Thiết lập kênh tiếp nhận khiếu nại (Feedback Loop): Người dùng có thể bấm "Báo cáo câu trả lời bị chặn nhầm". Đội ngũ Ops/AI Engineer sẽ rà soát log đã ẩn danh trong vòng 24 giờ.
  - Định kỳ hàng tuần lấy mẫu ngẫu nhiên 5% query thực tế (sau khi loại bỏ PII) để bổ sung vào Regression Test Set, tái huấn luyện hoặc tinh chỉnh prompt cho NeMo rails.
- Giới hạn của bộ dữ liệu hiện tại: Bộ test 50 câu RAGAS, 20 câu adversarial, 8 câu clean và 4 câu output là tập dữ liệu mẫu tập trung vào chính sách nhân sự và IT nội bộ; chưa bao quát toàn bộ ngữ cảnh phức tạp và tiếng lóng trong thực tế doanh nghiệp.

RAGAS mean/worst distribution: mean 0.7645 / worst distribution là adversarial với mean 0.6442 (factual: 0.8481, multi_hop: 0.7412).
Cohen’s κ: 1.0 (khớp hoàn hảo 10/10 với nhãn con người).
Attacks passed: 18/20; clean false_positive_rate: 0.0.
Các hướng cải tiến cụ thể có thể kiểm tra:
1. Triển khai mô hình Local LLM nhỏ (như Qwen2.5-0.5B hoặc Llama-3.2-1B được fine-tune hoặc quantized via vLLM/ONNX) để thay thế OpenAI API trong NeMo input rail, giúp giảm P95 latency từ 716ms xuống dưới 80ms, giải quyết dứt điểm vấn đề latency budget.
2. Tối ưu hóa pipeline Presidio với mô hình tiếng Việt chuyên dụng (như PhoBERT NER) để nhận diện tên người và tổ chức tiếng Việt chính xác hơn mà không gây false positives.
3. Mở rộng bộ quy tắc Regex và Keyword Blacklist cho các mẫu tấn công phổ biến để chặn ngay tại Layer 1 (chỉ tốn ~2ms) trước khi phải chuyển tiếp sang NeMo LLM rail, giúp tiết kiệm hơn 60% chi phí gọi OpenAI API.
4. Tăng cường kỹ thuật contextual chunking và reranking ngưỡng thích ứng trong RAG DAY18 để cải thiện điểm số RAGAS trên nhóm câu hỏi đa bước (multi_hop) từ 0.7412 lên trên 0.85.
