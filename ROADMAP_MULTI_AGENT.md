# KẾ HOẠCH TRIỂN KHAI LAB D24 — MULTI-AGENT WORKFLOW

Tài liệu này phân chia các giai đoạn và nhiệm vụ độc lập nhằm đảm bảo **nhiều agent (hoặc nhiều luồng phát triển) có thể làm việc song song mà không xung đột file (no merge conflicts/file contention)**, tuân thủ 100% hợp đồng dữ liệu và tiêu chí chấm điểm của Lab.

---

## I. MA TRẬN PHÂN QUYỀN VÀ TRÁCH NHIỆM FILE (OWNERSHIP MATRIX)

Mỗi Agent được phân quyền sở hữu độc quyền các tệp tin cụ thể. **Tuyệt đối không có 2 agent cùng ghi vào 1 file**.

| Agent / Luồng | File sở hữu độc quyền (Write Access) | File tham chiếu (Read Only) | Sản phẩm đầu ra (Artifacts/Reports) |
|---|---|---|---|
| **Agent 0: Setup & Data Prep** | `.env`, `artifacts/answers_50q.json`, `src/m1_chunking.py` ... `m5_enrichment.py` | `fixtures/test_set_50q.json`, `config.py` | `artifacts/answers_50q.json` |
| **Agent 1: Phase A (RAGAS)** | `src/phase_a_ragas.py`, `doc/reports/failure_clusters.md` | `artifacts/answers_50q.json`, `fixtures/test_set_50q.json` | `artifacts/ragas_50q.json`, `failure_clusters.md` |
| **Agent 2: Phase B (LLM-Judge)** | `src/phase_b_judge.py`, `doc/reports/bias_report.md` | `artifacts/answers_50q.json`, `fixtures/human_labels_10q.json`, `test_set_50q.json` | `artifacts/judge_results.json`, `bias_report.md` |
| **Agent 3: Phase C (Guardrails)** | `src/phase_c_guard.py` | `fixtures/adversarial_set_20.json`, `clean_set_8.json`, `output_set_4.json`, `guardrails/*` | `artifacts/guard_results.json` |
| **Agent 4: Integrator & Lead** | `doc/reports/blueprint.md`, `doc/TEAM.md` | Đọc toàn bộ 4 file trong `artifacts/` | `blueprint.md`, `TEAM.md`, kết quả `check_lab.py` |

---

## II. LƯỢC ĐỒ PHỤ THUỘC (DEPENDENCY GRAPH)

```text
[Giai đoạn 0: Setup & Pipeline Answers]
        │
        ├──> Sinh artifacts/answers_50q.json
        │           │
        │           ├──────────────────────────────┐
        ▼                                          ▼
[Giai đoạn 1A: Agent 1 - Phase A]       [Giai đoạn 1B: Agent 2 - Phase B]       [Giai đoạn 1C: Agent 3 - Phase C]
  - Coder: src/phase_a_ragas.py           - Coder: src/phase_b_judge.py           - Coder: src/phase_c_guard.py
  - Test: pytest tests/test_phase_a.py    - Test: pytest tests/test_phase_b.py    - Test: pytest tests/test_phase_c.py
  - Run: run_experiments --phase a        - Run: run_experiments --phase b        - Run: run_experiments --phase c
  - Doc: failure_clusters.md              - Doc: bias_report.md                   (CHẠY ĐỘC LẬP HOÀN TOÀN)
        │                                          │                                       │
        └──────────────────────────┬───────────────┴───────────────────────────────────────┘
                                   │
                                   ▼
                   [Giai đoạn 2: Agent 4 - Integrator]
                     - Thu thập latency từ guard_results.json
                     - Hoàn thành doc/reports/blueprint.md
                     - Cập nhật doc/TEAM.md (thành viên, cá nhân)
                     - Chạy scripts/check_lab.py --with-tests
```

*Nhận xét quan trọng:*
- **Agent 3 (Phase C)** hoàn toàn độc lập với `answers_50q.json`, có thể khởi chạy ngay từ giây đầu tiên!
- **Agent 1 (Phase A)** và **Agent 2 (Phase B)** chỉ cần chờ `artifacts/answers_50q.json` được sinh ra, sau đó chạy song song mà không chạm vào code của nhau.

---

## III. CHI TIẾT CÁC GIAI ĐOẠN TRIỂN KHAI

### Giai đoạn 0: Chuẩn bị Môi trường & Dữ liệu Nguồn (Agent 0)
- **Mục tiêu:** Đảm bảo có `OPENAI_API_KEY`, Qdrant docker (nếu cần), và sinh ra `artifacts/answers_50q.json`.
- **Nhiệm vụ:**
  1. Kiểm tra `.env` có `OPENAI_API_KEY`.
  2. Bổ sung các module RAG `src/m1_chunking.py`, `src/m2_search.py`, `src/m3_rerank.py`, `src/m4_eval.py`, `src/m5_enrichment.py` (từ Day 18 hoặc adapter chuẩn theo [`doc/PIPELINE_INTERFACE.md`](file:///d:/AI_20K/Day%2024%20-%20Track%203%20-%20RAGAS%20LLM-as-Judge%20and%20Guardrails/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard/doc/PIPELINE_INTERFACE.md)).
  3. Khởi động Qdrant: `docker compose -f infra/docker-compose.yml up -d`.
  4. Thực thi sinh dữ liệu:
     ```bash
     python scripts/setup_answers.py --live
     ```
  5. Kiểm tra hợp lệ: File `artifacts/answers_50q.json` đủ 50 câu (20 factual, 20 multi_hop, 10 adversarial), có đầy đủ `generation_mode` và `generation_model`.

---

### Giai đoạn 1A: Chuyên trách Phase A — RAGAS (Agent 1)
- **File phụ trách:** `src/phase_a_ragas.py`, `doc/reports/failure_clusters.md`.
- **Nhiệm vụ Coding:**
  - `Task 1: group_by_distribution(test_set)`: Gom 50 câu thành dict 3 keys `factual` (20), `multi_hop` (20), `adversarial` (10).
  - `Task 2: run_ragas_50q(answers)`: Tích hợp với `m4_eval.evaluate_ragas`, trả về `RagasEvaluation(results, raw["metadata"]["evaluator_models"])`. Đảm bảo đủ 4 metrics hữu hạn trong $[0, 1]$.
  - `Task 3: bottom_10(results)`: Lấy 10 câu có `avg_score` thấp nhất (tăng dần), tra cứu `DIAGNOSTIC_TREE` gán `diagnosis` và `suggested_fix`.
  - `Task 4: cluster_analysis(results)`: Xây dựng ma trận $4 \times 3$, xác định `dominant_failure_distribution` bằng **phân phối có avg_score trung bình thấp nhất** (loại bỏ nhóm rỗng, không chọn theo số lượng 20/20/10), xác định `dominant_failure_metric` và `insight`.
- **Quy trình nghiệm thu Phase A:**
  1. Kiểm thử unit test offline:
     ```bash
     python -m pytest tests/test_phase_a.py -v
     ```
  2. Chạy thực nghiệm live:
     ```bash
     python scripts/run_experiments.py --phase a --live
     ```
  3. Mở `artifacts/ragas_50q.json`, điền số liệu thật vào [`doc/reports/failure_clusters.md`](file:///d:/AI_20K/Day%2024%20-%20Track%203%20-%20RAGAS%20LLM-as-Judge%20and%20Guardrails/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard/doc/reports/failure_clusters.md). Xóa bỏ toàn bộ `{{...}}`.

---

### Giai đoạn 1B: Chuyên trách Phase B — LLM-as-a-Judge (Agent 2)
- **File phụ trách:** `src/phase_b_judge.py`, `doc/reports/bias_report.md`.
- **Nhiệm vụ Coding:**
  - `Task 5: pairwise_judge(q, A, B, reference)`: Prompt gọi `gpt-4o-mini` trả JSON `{winner: "A"|"B"|"tie", reasoning: str, scores: {A: float, B: float}}`. Đánh giá dựa trên `reference`.
  - `Task 6: swap_and_average(q, A, B, reference)`: Chạy Pass 1 (A vs B) và Pass 2 (B vs A). Đảo chiều kết quả Pass 2 về không gian A/B gốc. Đồng thuận thì giữ nguyên, bất nhất thì trả `"tie"`. Trả về `JudgeResult`.
  - `Task 7: cohen_kappa(judge_labels, human_labels)`: Tính toán hệ số Cohen's $\kappa$ giữa 10 nhãn nhị phân từ LLM và nhãn người:
    $$\kappa = \frac{p_o - p_e}{1 - p_e}$$
    (Hỗ trợ cả trường hợp perfect agreement $\kappa = 1.0$).
  - `Task 8: bias_report(judge_results)`: Tính tỷ lệ bất nhất vị trí (`position_bias_rate`), đếm số lượng câu dài hơn thắng (`verbosity_bias`), chi tiết `verbosity_details` và diễn giải.
- **Quy trình nghiệm thu Phase B:**
  1. Kiểm thử unit test offline:
     ```bash
     python -m pytest tests/test_phase_b.py -v
     ```
  2. Chạy thực nghiệm live:
     ```bash
     python scripts/run_experiments.py --phase b --live
     ```
  3. Mở `artifacts/judge_results.json`, điền số liệu thật vào [`doc/reports/bias_report.md`](file:///d:/AI_20K/Day%2024%20-%20Track%203%20-%20RAGAS%20LLM-as-Judge%20and%20Guardrails/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard/doc/reports/bias_report.md). Xóa bỏ toàn bộ `{{...}}`.

---

### Giai đoạn 1C: Chuyên trách Phase C — Guardrails Stack (Agent 3)
- **File phụ trách:** `src/phase_c_guard.py`.
- **Nhiệm vụ Coding:**
  - `Task 9a: pii_scan(text, analyzer, anonymizer)`: Quét PII (CCCD/CMND, SĐT VN, Email) bằng Presidio. Trả về `{has_pii, entities, anonymized}`.
  - `Task 9b: check_input_rail(text, rails)` (async): Gọi `rails.generate_async`, kiểm tra `result.log.activated_rails` có cờ `stop`. Trả về `{allowed, blocked_reason, response}`. Tuyệt đối không bắt từ khóa từ chối heuristic.
  - `Task 10: run_adversarial_suite(adversarial_set, rails, ...)`: Pipeline 2 lớp: Presidio PII $\rightarrow$ NeMo input. Chạy qua danh sách 28 input (20 attacks + 8 clean). Giữ nguyên provenance.
  - `Task 11: check_output_rail(question, answer, rails)` (async): Kiểm tra output LLM qua NeMo output rails. Nếu an toàn giữ nguyên, nếu nhạy cảm redact/thay thế.
  - `Task 12: measure_p95_latency(test_inputs, n_runs=20, ...)`:
    - 2 warm-up runs thật bị loại bỏ.
    - Tối thiểu 20 sample đo thời gian thật (cộng dồn `presidio_ms` + `nemo_input_ms`).
    - Dùng `summarize_latency()` để tính percentile nội suy tuyến tính.
- **Quy trình nghiệm thu Phase C:**
  1. Kiểm thử unit test offline:
     ```bash
     python -m pytest tests/test_phase_c.py -v
     ```
  2. Chạy thực nghiệm live:
     ```bash
     python scripts/run_experiments.py --phase c --live --latency-samples 20
     ```
  3. Kiểm tra artifact `artifacts/guard_results.json`: Attacks đạt $\ge 15/20$ (hướng tới $\ge 18/20$ để lấy bonus), Clean controls đạt $8/8$ allowed (`false_positive_rate = 0.0`), Output results đạt $4/4$.

---

### Giai đoạn 2: Tích hợp, Báo cáo Tổng thể & Nghiệm thu Cuối (Agent 4 / Lead)
- **File phụ trách:** `doc/reports/blueprint.md`, `doc/TEAM.md`.
- **Điều kiện bắt đầu:** Cả 3 Phase A, B, C đã chạy sinh ra đủ 4 file JSON trong `artifacts/`.
- **Nhiệm vụ:**
  1. **Hoàn thiện Blueprint ([`doc/reports/blueprint.md`](file:///d:/AI_20K/Day%2024%20-%20Track%203%20-%20RAGAS%20LLM-as-Judge%20and%20Guardrails/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard/doc/reports/blueprint.md)):**
     - Điền chính xác số liệu P50, P95, P99 ms của Presidio, NeMo input và Input guard sequential từ `artifacts/guard_results.json`.
     - Giữ nguyên nhãn `"chưa đo"` cho RAG, NeMo output, Full guard (tránh bị lỗi điền số giả).
     - Đề xuất CI/CD quality gates và cơ chế monitoring.
     - Thay thế toàn bộ placeholder `{{...}}`.
  2. **Hoàn thiện TEAM ([`doc/TEAM.md`](file:///d:/AI_20K/Day%2024%20-%20Track%203%20-%20RAGAS%20LLM-as-Judge%20and%20Guardrails/K4-Track-3A-Day24-DuongDinhLong-2A202602474-Eval-Guard/doc/TEAM.md)):**
     - Cập nhật danh sách thành viên (nếu làm 1 người thì để 1 dòng, 1 mục INDIVIDUAL).
     - Điền cấu hình môi trường thực tế (OS, Python 3.11, venv).
     - Thay thế toàn bộ placeholder `{{...}}`.
  3. **Chạy Tổng Kiểm Tra (Final Quality Gate):**
     ```bash
     python scripts/check_lab.py --with-tests
     ```
     *Mục tiêu:* In ra `PASS` cho tất cả các tiêu chí (0 failed checks).
  4. **Git Verification:**
     - Đảm bảo 4 file trong `artifacts/` không bị ignore:
       ```bash
       git check-ignore artifacts/*.json
       ```
     - Kiểm tra không còn file `.env` hay API key nào lọt vào git status.

---

## IV. BẢNG CHECKLIST SẴN SÀNG CHO MULTI-AGENT

| Hạng mục | Tiêu chí đạt | Phụ trách |
|---|---|---|
| [ ] `answers_50q.json` | Đủ 50 id, 20/20/10, có `generation_mode` và `generation_model` hợp lệ | Agent 0 |
| [ ] `phase_a_ragas.py` | Sạch `# TODO`, pass `pytest tests/test_phase_a.py` | Agent 1 |
| [ ] `ragas_50q.json` | 50 rows, 4 metrics hữu hạn, bottom 10 có diagnosis & fix | Agent 1 |
| [ ] `failure_clusters.md` | Điền đủ số liệu, matrix $4 \times 3$, phân tích bẫy adversarial, hết `{{...}}` | Agent 1 |
| [ ] `phase_b_judge.py` | Sạch `# TODO`, pass `pytest tests/test_phase_b.py` | Agent 2 |
| [ ] `judge_results.json` | 5 cặp pairwise 2 passes, 10 binary judgments không leak nhãn human, $\kappa$ chuẩn | Agent 2 |
| [ ] `bias_report.md` | Điền đủ 5 cặp, 10 binary rows, position & verbosity bias, hết `{{...}}` | Agent 2 |
| [ ] `phase_c_guard.py` | Sạch `# TODO`, pass `pytest tests/test_phase_c.py` | Agent 3 |
| [ ] `guard_results.json` | Attacks $\ge 15/20$, Clean $8/8$ allowed ($FPR=0$), Output $4/4$, $\ge 20$ latency samples | Agent 3 |
| [ ] `blueprint.md` | Đúng số đo latency input layers, phần ngoài ghi "chưa đo", hết `{{...}}` | Agent 4 |
| [ ] `TEAM.md` | Đúng thông tin thành viên / 1 người, có mục INDIVIDUAL khớp MSSV, hết `{{...}}` | Agent 4 |
| [ ] `check_lab.py` | `0 failed checks` khi chạy `python scripts/check_lab.py --with-tests` | Agent 4 |
