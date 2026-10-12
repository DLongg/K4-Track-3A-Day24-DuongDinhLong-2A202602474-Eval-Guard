# Phân tích lỗi Phase A

Nhóm: K4-Track-3A-DuongDinhLong

Người phụ trách: Dương Đình Long - 2A202602474

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T15:13:21.422952+00:00.
Generation mode/fallback và điều kiện chạy: LLM generation mode (`gemini-2.5-flash-lite`), không sử dụng extractive fallback; chạy live qua API Vilao.
Models generation/evaluator: Generation model: `gemini-2.5-flash-lite`; Evaluator models: `gemini-2.5-flash-lite`, `BAAI/bge-m3` từ m4.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.9500 | 0.4212 | 0.6333 |
| answer_relevancy | 0.8615 | 0.6422 | 0.6812 |
| context_precision | 0.9333 | 0.6000 | 0.8500 |
| context_recall | 0.8583 | 0.6625 | 0.7250 |
| avg_score | 0.9008 | 0.5815 | 0.7224 |

## Bottom 10 và nguyên nhân

Bảng dưới đây là danh sách chính xác 10 câu hỏi có `avg_score` thấp nhất từ Task 3:

| Rank | ID | Phân phối | Câu hỏi | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 27 | multi_hop | Nhân viên có thâm niên 7 năm theo v2024 được nghỉ bao nhiêu ngày phép? Nếu bị trừ thêm 4 ngày ốm không có giấy y tế thì còn lại bao nhiêu? | 0.1413 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 39 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | 0.1703 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 33 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.2819 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 21 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.3173 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 5 | 50 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 6 | 48 | adversarial | Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không? | 0.4167 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 7 | 26 | multi_hop | Nhân viên vừa kết hôn và cùng tuần đó có con kết hôn. Tổng số ngày nghỉ đặc biệt có lương là bao nhiêu? | 0.4354 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 35 | multi_hop | Nhân viên Junior P1 có lương cơ bản 12 triệu vừa vào thử việc: nhận lương bao nhiêu và phụ cấp gì tháng đầu tiên? | 0.4711 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 9 | 22 | multi_hop | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | 0.4739 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 10 | 25 | multi_hop | Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu? | 0.4918 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

### Đối chiếu bằng chứng chi tiết các câu tiêu biểu:

1. **Câu ID 27 (multi_hop - Lỗi suy luận/tổng hợp đa tài liệu):**
   - *Câu hỏi:* Tính phép năm thâm niên 7 năm (v2024) và số ngày còn lại sau khi trừ 4 ngày ốm không giấy y tế.
   - *Ground Truth:* 15 ngày cơ bản + 2 ngày thâm niên = 17 ngày; 17 - 4 = 13 ngày.
   - *Thực tế:* Chunks trả về từ retrieval chỉ có đoạn quy định 15 ngày cơ bản và quy định nghỉ ốm bị trừ vào phép năm, nhưng thiếu chunk công thức tính thâm niên (cứ 3 năm thêm 1 ngày) hoặc LLM không tổng hợp được hai điều kiện này, dẫn tới trả lời ngắn hoặc rơi vào "Không tìm thấy", kéo `faithfulness` xuống 0.0.

2. **Câu ID 39 (multi_hop - Lỗi version conflict giữa policy v1.0 và v2.0):**
   - *Câu hỏi:* So sánh chính sách mật khẩu giữa policy v1.0 (hết hiệu lực) và v2.0 (hiện hành) về độ dài tối thiểu, chu kỳ đổi và MFA.
   - *Ground Truth:* v1.0: tối thiểu 8 ký tự, đổi mỗi 90 ngày, không MFA. v2.0: tối thiểu 12 ký tự, đổi mỗi 120 ngày, MFA bắt buộc.
   - *Thực tế:* Reranker ưu tiên chunk v2.0 do điểm semantic cao hơn, làm mất chunk v1.0; LLM thiếu dữ liệu so sánh hai phiên bản nên không đưa ra được bảng đối chiếu đầy đủ, dẫn đến điểm trung bình chỉ đạt 0.1703.

3. **Câu ID 50 & 48 (adversarial - Negation trap và Policy contradiction):**
   - *Câu hỏi ID 50:* Hỏi Manager có thể dùng VPN cá nhân (NordVPN) khi WFH để tăng bảo mật không.
   - *Ground Truth:* KHÔNG. Chính sách VPN cấm dùng VPN cá nhân trong mạng công ty; bắt buộc dùng WireGuard.
   - *Câu hỏi ID 48:* Hỏi nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không.
   - *Ground Truth:* KHÔNG. Nhân viên thử việc chưa được hưởng gói PVI (chỉ có BHXH bắt buộc).
   - *Thực tế:* Các câu bẫy phủ định và mâu thuẫn chính sách khiến retriever lấy các đoạn văn nói về lợi ích chung của VPN hoặc bảo hiểm PVI mà không trích xuất được mệnh đề loại trừ (exclusion). Do đó, LLM trả lời "Không tìm thấy" hoặc phỏng đoán sai lệch, làm giảm điểm `faithfulness`.

## Matrix và nhóm yếu nhất

Dưới đây là ma trận phân bố số lượng câu hỏi có `worst_metric` tương ứng theo từng distribution (tổng theo cột bằng 20/20/10):

| Worst Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 1 | 12 | 4 | 17 |
| context_recall | 5 | 2 | 3 | 10 |
| context_precision | 3 | 2 | 3 | 8 |
| answer_relevancy | 11 | 4 | 0 | 15 |

- Nhóm yếu theo mean thấp nhất: **multi_hop**; mean avg_score: **0.5815** (so với factual: 0.9008 và adversarial: 0.7224).
- Worst metric xuất hiện nhiều nhất: **faithfulness**; số câu: **17 câu** (chiếm 34% toàn bộ test set, đặc biệt tập trung ở multi_hop với 12 câu).

### Phân tích nguyên nhân và đề xuất cải tiến:
- **Vì sao multi_hop là nhóm yếu nhất:** Các câu hỏi multi_hop đòi hỏi phải kết hợp thông tin từ 2 đến 3 tài liệu khác nhau (ví dụ: kết hợp chính sách phép năm với quy định nghỉ ốm, hoặc kết hợp cấp bậc Junior với thang bảng lương). Khi retrieval chỉ lấy top-3 chunks qua dense/hybrid search mà không có Query Expansion hoặc Sub-query Decomposition, hệ thống thường bỏ sót 1 trong 2 mảng thông tin cần thiết. Hệ quả là LLM phải suy diễn ngoài ngữ cảnh dẫn đến suy giảm `faithfulness`.
- **Đề xuất khắc phục cụ thể:**
  1. Bổ sung kỹ thuật Multi-Query / Sub-question decomposition trong pipeline trước khi truy vấn dense & BM25.
  2. Nâng số lượng retrieval chunk lên Top 5 hoặc Top 8 kết hợp Cross-Encoder Reranker để không làm rơi rụng context giữa các tài liệu phụ.
  3. Thắt chặt prompt: Yêu cầu LLM chỉ trả lời dựa trên context được cung cấp và nêu rõ các phần còn thiếu nếu không đủ thông tin.
- **Cách chạy lại để kiểm chứng:** Sau khi cập nhật module retrieval, sinh lại câu trả lời và chạy lại `python scripts/run_experiments.py --phase a --live` để đối chiếu xem avg_score của `multi_hop` có vượt ngưỡng 0.75 hay không.

## Bẫy adversarial và giới hạn

- **So sánh với factual và multi_hop:**
  - Nhóm factual đạt điểm rất cao (avg_score 0.9008, faithfulness 0.9500) do câu hỏi rõ ràng, thông tin tập trung trong một đoạn văn bản đơn lẻ.
  - Nhóm adversarial có điểm trung bình 0.7224, cao hơn multi_hop nhưng thấp hơn factual đáng kể. Cụ thể có 2 câu adversarial (ID 50 và ID 48) rơi vào Bottom 10.
- **Bẫy xung đột phiên bản (Version conflict) và Phủ định (Negation trap):**
  - Trong các câu hỏi bẫy (như ID 48 về bảo hiểm thử việc, ID 50 về VPN cá nhân), pipeline thường gặp khó khăn do tài liệu chứa nhiều câu khẳng định về quyền lợi chung, trong khi điều khoản loại trừ (exclusion clause) nằm rải rác hoặc bị cắt ngang chunk.
  - Điểm số thấp hơn ở nhóm adversarial phản ánh việc pipeline dễ bị nhầm lẫn khi người dùng hỏi các câu mang tính giả định sai hoặc mâu thuẫn chính sách.
- **Giới hạn thực nghiệm:**
  - Số lượng mẫu adversarial còn khiêm tốn (10 câu), chưa bao quát hết các kịch bản prompt injection hoặc jailbreak trong pha generation (các kịch bản này được xử lý độc lập ở Phase C với Guardrails).
  - Model generator và evaluator là `gemini-2.5-flash-lite`, sử dụng embedding `BAAI/bge-m3`. Độ nhạy với các sắc thái tiếng Việt phức tạp trong câu phủ định kép vẫn còn hạn chế nhất định.
