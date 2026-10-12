# Đánh giá LLM Judge Phase B

Nhóm: DuongDinhLong-Eval-Guard

Người phụ trách: Dương Đình Long - 2A202602474

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T15:11:47.845764+00:00.
Models generation/judge: generation: ["gemini-2.5-flash-lite"], judge: ["gemini-2.5-flash-lite"].

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ
fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

| ID | Candidate A (Pipeline) | Candidate B (Supplied) | Winner Pass 1 | Winner Pass 2 (quy về A/B) | Final Winner | Position Consistent | Scores Pass 1 (A/B) | Scores Pass 2 (A/B) | Phân tích lý do chính |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 3 ngày làm việc. | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | B | B | B | True | A: 0.6, B: 0.9 | A: 0.7, B: 1.0 | Answer B đầy đủ hơn khi nêu rõ quyền lợi hưởng lương và không trừ phép năm theo đúng chính sách tham chiếu; Answer A chỉ nêu số ngày. |
| 5 | Tổng Giám đốc (CEO) | Cần Giám đốc phòng ban phê duyệt. | A | A | A | True | A: 1.0, B: 0.0 | A: 1.0, B: 0.2 | Answer A chính xác tuyệt đối theo quy định mua sắm trên 50 triệu VNĐ cần CEO duyệt; Answer B sai thẩm quyền phê duyệt. |
| 12 | 1 tháng lương. | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | tie | B | tie | False | A: 0.9, B: 0.95 | A: 0.6, B: 0.9 | Cốt lõi cả hai đều đúng "1 tháng lương". Pass 1 đánh giá ngang nhau (A súc tích, B đủ ngữ cảnh); Pass 2 ưu tiên ngữ cảnh của B dẫn đến không nhất quán vị trí, consensus quy về tie. |
| 21 | Trả lời dài dòng, suy diễn 16-18 ngày phép, thiếu lương. | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | B | B | B | True | A: 0.5, B: 1.0 | A: 0.7, B: 1.0 | Answer B súc tích, tính đúng 18 ngày và có khoảng lương Senior (20-35 triệu). Answer A dài dòng, suy diễn nhiều cách hiểu và thiếu hẳn thông tin lương. |
| 41 | Mỗi nhân viên chính thức được hưởng 15 ngày phép năm có lương. | Nhân viên được nghỉ 12 ngày phép năm. | A | A | A | True | A: 1.0, B: 0.2 | A: 0.9, B: 0.2 | Answer A chính xác theo chính sách hiện hành v2024 (15 ngày có lương). Answer B trích dẫn chính sách cũ v2023 (12 ngày) đã hết hiệu lực. |

Position bias count/rate: 1 / 5 (tỷ lệ 0.2, tức 20.0%).
Verbosity win rate: 0.5 (tức 50.0%); decisive cases: 4.

Phân tích hiện tượng thiên lệch vị trí và độ dài:
- Về Position Bias: Tỷ lệ bất nhất vị trí chỉ xảy ra ở 1/5 trường hợp (câu ID 12). Ở câu này, hai câu trả lời có chất lượng tương đương nhau về mặt nội dung cốt lõi ("1 tháng lương"). Khi chất lượng quá sát nhau, LLM judge có xu hướng dao động nhẹ theo thứ tự trình bày. Kỹ thuật swap-and-average đã chứng minh tính hiệu quả khi tự động gán nhãn "tie", ngăn chặn việc thiên vị ngẫu nhiên.
- Về Verbosity Bias: Trong 4 trường hợp có người thắng rõ ràng (decisive cases), có đúng 2 trường hợp câu dài hơn thắng (ID 1 và ID 41) và 2 trường hợp câu ngắn hơn thắng (ID 5 và ID 21), tạo nên tỷ lệ verbosity win rate là 50.0%. Đặc biệt tại câu ID 21, câu A dài hơn rất nhiều do liệt kê và suy diễn các giả định nhưng vẫn bị xử thua câu B vì thiếu thông tin lương và dài dòng không cần thiết. Điều này chứng minh mô hình judge không bị thiên lệch bởi độ dài văn bản mà tập trung vào độ chính xác và đầy đủ của thông tin.

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json;
không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference.
Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

| Question ID | Model Answer gốc | Nhãn Judge (0/1) | Human Label (0/1) | Đồng thuận (Agree) | Lý giải của LLM Judge |
|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | 1 | Đồng thuận | Câu trả lời chính xác, đầy đủ ý và bám sát reference. |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | 0 | Đồng thuận | Câu trả lời sai vì theo quy định, đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt, không phải Giám đốc phòng ban. |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | 1 | Đồng thuận | Câu trả lời đã nêu đúng mức thưởng Tết tối thiểu là 1 tháng lương, phù hợp với điều kiện nhân viên chính thức có từ 6 tháng trở lên theo reference. |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | 1 | Đồng thuận | Câu trả lời chính xác, đầy đủ các ý của câu hỏi và bám sát thông tin trong reference. |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | 1 | Đồng thuận | Câu trả lời chính xác, đầy đủ và bám sát reference. Nhân viên nghỉ việc sau 8 tháng là chưa đủ 1 năm cam kết nên phải hoàn trả 100% chi phí là 25 triệu VNĐ. |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | 0 | Đồng thuận | Câu trả lời sai về người phê duyệt (thiếu Kế toán trưởng) và sai về phí phạt (chỉ tính 15 ngày thay vì 30 ngày và không tính theo tỉ lệ). |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | 1 | Đồng thuận | Câu trả lời chính xác và đầy đủ theo tham chiếu, bao gồm cả tổng phụ cấp hàng tháng và số ngày phép năm cho nhân viên có thâm niên 12 năm. |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | 0 | Đồng thuận | Câu trả lời sử dụng thông tin từ chính sách cũ (v2023) là 12 ngày, trong khi chính sách hiện hành (v2024) quy định là 15 ngày cơ bản. Câu trả lời cần ưu tiên chính sách hiện hành. |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | 1 | Đồng thuận | Câu trả lời khẳng định đúng nhân viên thử việc không được nghỉ phép năm và cần xin nghỉ không lương, khớp với thông tin trong reference. |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | 0 | Đồng thuận | Câu trả lời cho phép sử dụng VPN cá nhân khi WFH, nhưng chính sách rõ ràng cấm điều này và yêu cầu sử dụng VPN của công ty. |

Cohen’s κ: 1.0 (Đồng thuận hoàn hảo / Almost Perfect theo thang đo Landis-Koch).
Observed/chance agreement và diễn giải:
- Observed agreement: $p_o = 10 / 10 = 1.0$ (100% các câu hỏi khớp nhãn giữa LLM judge và Human).
- Phân bố nhãn: 6 nhãn tốt (label 1) và 4 nhãn xấu (label 0) ở cả hai bên.
- Chance agreement: $p_e = (0.6 \times 0.6) + (0.4 \times 0.4) = 0.36 + 0.16 = 0.52$.
- Tính toán hệ số Cohen's κ: $\kappa = \frac{p_o - p_e}{1 - p_e} = \frac{1.0 - 0.52}{1 - 0.52} = 1.0$.
- Nhận xét: LLM judge thể hiện khả năng nhận diện xuất sắc các lỗi nghiêm trọng như vi phạm bảo mật VPN cá nhân (câu 50), nhầm lẫn phiên bản chính sách cũ v2023 (câu 41), sai quy trình phê duyệt tài chính (câu 5, 29).

## Giới hạn và cách dùng judge

1. Giới hạn quy mô mẫu:
   - Thử nghiệm hiệu chuẩn được thực hiện trên tập mẫu nhỏ (10 câu nhãn nhị phân và 5 cặp so sánh pairwise). Mặc dù đạt $\kappa = 1.0$, kết quả này phản ánh năng lực phân biệt các trường hợp sai rõ ràng (lỗi phiên bản, vi phạm quy định) hơn là các sắc thái tinh vi trong văn phong thực tế.
2. Cách dùng judge an toàn trong production:
   - Áp dụng kỹ thuật Swap-and-Average bắt buộc cho các so sánh pairwise nhằm loại bỏ triệt để position bias khi LLM gặp các câu trả lời tương đương về độ chính xác.
   - Luôn sử dụng temperature = 0 và cung cấp tài liệu tham chiếu (ground truth / reference policy) cụ thể; không để judge phán đoán dựa trên kiến thức tự do ngoài tài liệu.
   - Không coi LLM-as-a-Judge là chân lý tuyệt đối: Cần áp dụng mô hình Human-in-the-loop (chuyên gia đối soát định kỳ các ca "tie" hoặc điểm số dao động lớn) và sử dụng judge như một bộ lọc tự động phân loại rủi ro (risk-based triaging) trong pipeline CI/CD.
