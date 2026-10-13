# Nhóm thực hiện K4 Track 3B

Tên nhóm: Duo

Repo chung: https://github.com/hoangtrunghieu0025-lab/K4-L3-DAY24-Duo-Eval-Guard.git

Trưởng nhóm/người nộp: Hoàng Trung Hiếu - 2A202602945

Ngày lab/deadline áp dụng: 23:59 (13/10/2026) — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: 60adcd6

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Hoàng Trung Hiếu | 2A202602945 | Trưởng nhóm, tích hợp pipeline DAY18, Phase A RAGAS và Phase C Guardrails | Commit Git, artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/guard_results.json |
| 2 | Nguyễn Việt Hoàng Hải | 2A202602967 | Thành viên, phụ trách Phase B LLM Judge, hiệu chuẩn Cohen Kappa và Blueprint | Commit Git, artifacts/judge_results.json, doc/reports/bias_report.md, doc/reports/blueprint.md |

Người tích hợp/chạy batch/nộp link: Hoàng Trung Hiếu - 2A202602945, tích hợp module từ DAY18, thực thi setup_answers.py và run_experiments.py.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11, Python 3.11/3.13, Microsoft Visual C++ Build Tools 14.0.
Virtualenv/dependencies/imports: Đã cài đặt đầy đủ Ragas, LangChain, NeMo Guardrails, Presidio; pytest unit tests offline chạy đạt 100%.
Models/Qdrant/50 answers đã chuẩn bị: Qdrant v1.9.7 (localhost:6333), embedding BAAI/bge-m3, reranker BAAI/bge-reranker-v2-m3, pipeline sinh đủ 50 câu hỏi cho artifacts/answers_50q.json.
Nếu cài full NeMo chưa thành công, ghi lỗi thực tế; không dùng unit test pass
hoặc pip resolve để khẳng định runtime đã chạy.

## INDIVIDUAL

### HoangTrungHieu-2A202602945

- Phần được giao và phần thực tế đã làm: Phụ trách tích hợp 5 module RAG từ DAY18, cài đặt Phase A (RAGAS 50q, failure clustering) và Phase C (Presidio PII scanner, NeMo input guard, P95 latency).
- Bằng chứng: Các commit thực hiện src/phase_a_ragas.py, src/phase_c_guard.py, file artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/guard_results.json.
- Cách kiểm tra: Chạy pytest tests/test_phase_a.py, tests/test_phase_c.py pass và chạy live batch thí nghiệm thành công.
- Điều đã học: Hiểu cách đánh giá 4 metric RAGAS, cấu trúc failure clusters và thiết lập guardrail tuần tự để cân bằng giữa độ an toàn và độ trễ P95.
- Giới hạn/cách xử lý: Môi trường Windows cần compiler MSVC để build Annoy; xử lý format metadata cho model evaluator chuẩn xác.
- Nguồn/AI hỗ trợ nếu có: Tham khảo tài liệu repo và sử dụng AI assistant hỗ trợ rà soát logic code, tự chạy và xác minh bằng bộ test.

### NguyenVietHoangHai-2A202602967

- Phần được giao và phần thực tế đã làm: Phụ trách Phase B (Pairwise judge, swap-and-average, tính Cohen's kappa, đo Position & Verbosity bias) và hoàn thiện Blueprint CI/CD.
- Bằng chứng: Các commit thực hiện src/phase_b_judge.py, artifacts/judge_results.json, doc/reports/bias_report.md, doc/reports/blueprint.md.
- Cách kiểm tra: Chạy pytest tests/test_phase_b.py pass, chạy batch Phase B đối chiếu chính xác với 10 nhãn human labels.
- Điều đã học: Hiểu cơ chế position bias của LLM khi đánh giá pairwise, kỹ thuật swap-and-average để khử bias và tính Cohen's kappa để đo lường độ tin cậy của judge.
- Giới hạn/cách xử lý: Với tập dữ liệu nhỏ 10 mẫu nhãn người, cần phân tích kỹ lưỡng các case không đồng thuận và giải thích trong báo cáo.
- Nguồn/AI hỗ trợ nếu có: Tham khảo chuẩn công thức Cohen Kappa từ scikit-learn.

## Cập nhật sau khi nộp

Không có cập nhật bổ sung sau deadline.
