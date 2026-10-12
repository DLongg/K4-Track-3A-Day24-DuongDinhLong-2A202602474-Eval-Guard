# Blueprint CI/CD cho RAG và guardrails

Nhóm: DuongDinhLong-Eval-Guard

Người phụ trách: Dương Đình Long - 2A202602474

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T15:17:00.650481+00:00; guard models: ["gemini-2.5-flash-lite"].
Models generation/evaluator/judge từ A/B: generation: ["gemini-2.5-flash-lite"], evaluator: ["BAAI/bge-m3", "gemini-2.5-flash-lite"], judge: ["gemini-2.5-flash-lite"].

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Kiến trúc bảo vệ đa lớp (Defense-in-Depth) được thiết kế theo luồng dữ liệu tuần tự:
1. **Lớp 1 - Presidio PII Detection & Anonymization:** Quét và lọc thông tin định danh cá nhân (CCCD 12 số, CMND 9 số, số điện thoại Việt Nam, email). Nếu phát hiện PII nhạy cảm thuộc diện cấm đưa vào prompt, hệ thống lập tức ngắt chuỗi và trả về thông báo từ chối (Short-circuit). Với dữ liệu được phép xử lý, Presidio thay thế PII bằng token anonymized `<TYPE>` trước khi chuyển tiếp.
2. **Lớp 2 - NeMo Input Rails:** Kiểm tra câu hỏi người dùng có vi phạm chính sách phạm vi hỗ trợ (chỉ giải đáp chính sách HR/IT nội bộ), phòng chống Jailbreak, Prompt Injection và mạo danh quyền hạn. Quyết định block được xác định dựa trên cờ `stop` của `activated_rails`, kích hoạt hành động từ chối chuẩn mực mà không gọi đến mô hình RAG đắt đỏ.
3. **Lớp 3 - RAG Core Pipeline:** Thực hiện Hybrid Search (BM25 + Dense Qdrant) và Cross-Encoder Reranker để lấy Top-3 ngữ cảnh xác thực, sau đó gọi LLM thế hệ mới với prompt siết chặt căn cứ tài liệu.
4. **Lớp 4 - NeMo Output Rails:** Kiểm tra câu trả lời sinh ra từ LLM để đảm bảo không rò rỉ bí mật nội bộ, thông tin cá nhân của nhân sự cụ thể hoặc dữ liệu tài chính ngoài thẩm quyền. Nếu phát hiện vi phạm, hệ thống thay thế câu trả lời bằng thông điệp từ chối hoặc redact nội dung vi phạm trước khi gửi tới người dùng cuối.

Hệ thống bảo mật quản lý toàn bộ API Key và thông tin đăng nhập trong file biến môi trường `.env` hoặc hệ thống Secret Manager (như HashiCorp Vault / AWS Secrets Manager), tuyệt đối không commit vào mã nguồn. Cơ chế Fallback được thiết lập: khi dịch vụ LLM hoặc Guardrail gặp sự cố quá tải mạng, hệ thống chuyển sang chế độ extractive fallback an toàn hoặc trả về thông báo bảo trì có cấu trúc.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 8.02 | 11.13 | 11.89 | Đã đo thực tế Task 12 (chạy local cực nhanh) |
| NeMo input | 474.79 | 793.81 | 808.11 | Đã đo thực tế Task 12 (gọi LLM self-check qua mạng) |
| Input guard sequential | 483.67 | 802.63 | 815.86 | Đã đo thực tế Task 12 (tổng 2 tầng sequential) |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: P95 đạt 802.63ms, budget_ok = False.
Phân tích điểm nghẽn hiệu năng (Bottleneck):
- Tầng Presidio chạy hoàn toàn bằng regex và spaCy trên CPU local nên đạt hiệu năng lý tưởng (P95 chỉ 11.13ms, chiếm chưa tới 2% tổng thời gian).
- Điểm nghẽn chính nằm hoàn toàn ở tầng NeMo Input Rails (P95 lên tới 793.81ms) do cần thực hiện một lời gọi API mạng đến mô hình LLM để thực hiện prompt self_check_input. Độ trễ mạng (Network RTT) và thời gian sinh token của API bên thứ ba là nguyên nhân khiến P95 vượt ngưỡng mục tiêu 500ms.
- Kế hoạch đo lường cho các tầng ngoài phạm vi: Trong giai đoạn nâng cấp tiếp theo, nhóm sẽ triển khai thêm phép đo end-to-end có short-circuit và output guard rails bằng cách gắn timestamp tại API Gateway trước khi đưa vào pipeline chính thức.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.
Đề xuất thiết lập CI/CD Workflow (GitHub Actions / GitLab CI):
- Tạo một job tự động `evaluation-and-guardrails` chạy trên các pull request vào nhánh main.
- Script kiểm tra sẽ đọc artifacts hoặc chạy batch rút gọn, so sánh điểm số với các biến ngưỡng:
  - `MIN_FAITHFULNESS=0.75`
  - `MIN_AVG_SCORE=0.65`
  - `MIN_ATTACKS_PASSED=18`
  - `MAX_CLEAN_FALSE_POSITIVE=0`
- Nếu bất kỳ tiêu chí nào không đạt, script thoát với mã lỗi `exit 1`, tự động chặn pull request và không cho phép deploy lên môi trường Production.
- Lưu ý: Đây là workflow đề xuất dựa trên kết quả thực nghiệm, hiện tại quy trình được kiểm tra thủ công qua `scripts/check_lab.py`.

## Monitoring và cải tiến

Chiến lược giám sát vận hành liên tục (Production Monitoring):
1. **Thu thập Metrics và Log:** Ghi nhận mọi sự kiện chặn (blocked events) bao gồm lý do chặn (presidio vs nemo_input vs nemo_output), thời gian phản hồi của từng tầng, và tỷ lệ các câu hỏi bị từ chối. Dữ liệu nhạy cảm được che giấu (redact) trước khi ghi log để đảm bảo tuân thủ GDPR và Nghị định 13/2023.
2. **Cảnh báo tự động (Alerting):** Thiết lập ngưỡng cảnh báo khi tỷ lệ chặn vượt quá 15% trong 1 giờ (dấu hiệu tấn công diện rộng hoặc prompt drift) hoặc khi P95 latency vượt quá 1000ms.
3. **Quy trình xử lý False Positives:** Định kỳ hàng tuần lấy mẫu 5% các câu hỏi bị chặn để đội ngũ HR & Bảo mật đánh giá chéo. Nếu phát hiện câu hỏi hợp lệ bị chặn nhầm, cập nhật lại prompt và danh sách từ khóa của NeMo.

Số liệu đo lường thực tế làm cơ sở:
RAGAS mean/worst distribution: Mean toàn bộ đạt 0.7374; nhóm yếu nhất là multi_hop đạt 0.5815.
Cohen’s κ: 1.0 (đồng thuận hoàn hảo 10/10 với chuyên gia).
Attacks passed: 20/20; clean false_positive_rate: 0.0 (0/8).

Đề xuất 4 cải tiến kỹ thuật cụ thể:
1. **Nâng cấp công nghệ Input Rails sang mô hình phân loại cục bộ:** Thay thế lời gọi LLM từ xa của NeMo bằng mô hình phân loại chuyên biệt chạy on-premise (như DeBERTa/DistilBERT fine-tuned) để giảm P95 latency từ 800ms xuống dưới 50ms.
2. **Cải thiện bài toán Multi-hop Retrieval:** Bổ sung phương pháp Multi-Query Generator và HyDE để phân rã các câu hỏi phức tạp thành nhiều truy vấn con, giúp tăng `context_recall` và `faithfulness` cho nhóm câu hỏi kết hợp.
3. **Cơ chế lọc phiên bản chính sách (Temporal Filtering):** Thêm trường metadata `is_active` và `effective_year` vào index Qdrant để tự động lọc bỏ các tài liệu v2022/v2023 đã hết hiệu lực khi người dùng không yêu cầu tra cứu lịch sử.
4. **Bộ nhớ đệm Guardrail (Semantic Caching):** Sử dụng Redis hoặc semantic cache để lưu trữ kết quả kiểm tra an toàn cho các câu hỏi phổ biến, bỏ qua bước kiểm tra lặp lại đối với các truy vấn an toàn thường gặp.
