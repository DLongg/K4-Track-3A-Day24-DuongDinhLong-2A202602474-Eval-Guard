# Nhóm thực hiện K4 Track 3A

Tên nhóm: DuongDinhLong-Eval-Guard

Repo chung: https://github.com/DLongg/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard

Trưởng nhóm/người nộp: Dương Đình Long - 2A202602474

Ngày lab/deadline áp dụng: 2026-10-12 23:59 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: b9f8d5b53bc4f94899aa30fdc52c30be0e96eb6d

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Dương Đình Long | 2A202602474 | Trưởng nhóm, phụ trách toàn bộ pipeline m1-m5, Phase A, Phase B, Phase C và báo cáo | Commit git, artifacts JSON và 3 báo cáo Markdown |

Người tích hợp/chạy batch/nộp link: Dương Đình Long - Thực hiện chạy setup answers, batch runner A/B/C và tổng hợp báo cáo.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11, Python 3.11.9, MSVC C++ Build Tools v14.0.
Virtualenv/dependencies/imports: Đã cài đặt requirements.txt với constraints-core.txt, import ragas, nemoguardrails, presidio_analyzer thành công.
Models/Qdrant/50 answers đã chuẩn bị: Qdrant in-memory mode, BGE-M3 embedding, CrossEncoder reranker, 50 answers tạo bởi gemini-2.5-flash-lite tại artifacts/answers_50q.json.

## INDIVIDUAL

### DuongDinhLong-2A202602474

- Phần được giao và phần thực tế đã làm: Phụ trách toàn bộ dự án bao gồm: kết nối pipeline Day 18, sinh 50 answers cho test set; triển khai Phase A (RAGAS 4 metrics, bottom 10 và ma trận phân tích lỗi); triển khai Phase B (pairwise judge hoán đổi vị trí, tính Cohen kappa và phân tích bias); triển khai Phase C (Presidio PII, NeMo input/output rails, chạy adversarial suite và đo P95 latency); hoàn thiện toàn bộ báo cáo kỹ thuật failure_clusters.md, bias_report.md, blueprint.md.
- Bằng chứng: Các commit git trên repository, 4 tệp dữ liệu kết quả thực nghiệm trong artifacts/ (answers_50q.json, ragas_50q.json, judge_results.json, guard_results.json) và 3 bài báo cáo kỹ thuật trong doc/reports/.
- Cách kiểm tra: Đã chạy toàn bộ unit tests offline với 86/86 test passed (python -m pytest), chạy live batch runner cho cả 3 phase (scripts/run_experiments.py --phase a/b/c --live) và vượt qua 100% các tiêu chí kiểm tra của scripts/check_lab.py.
- Điều đã học: Hiểu sâu sắc phương pháp LLM-as-a-Judge và cách khử Position Bias bằng kỹ thuật swap-and-average; đo lường độ tin cậy của mô hình đánh giá bằng Cohen's Kappa; thiết kế hệ thống Guardrails đa tầng (Defense-in-Depth) kết hợp giữa Regex/NLP local (Presidio) và Semantic Guard (NeMo) để tối ưu cả độ an toàn và độ trễ latency.
- Giới hạn/cách xử lý: Độ trễ P95 của tầng NeMo Input Rails vượt budget 500ms do phụ thuộc lời gọi API qua mạng; giải pháp đề xuất là chuyển đổi sang mô hình phân loại cục bộ on-premise.
- Nguồn/AI hỗ trợ nếu có: Sử dụng Antigravity AI Assistant hỗ trợ rà soát kiểm thử và cấu hình Multi-Agent workflow; tự kiểm chứng toàn bộ mã nguồn và dữ liệu đo lường bằng test suite tự động và script check_lab.py.

## Cập nhật sau khi nộp

Không có cập nhật bổ sung sau deadline. Phiên bản nộp là bản commit hoàn thiện đúng hạn.
