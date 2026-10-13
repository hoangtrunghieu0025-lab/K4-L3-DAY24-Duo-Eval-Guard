# Phân tích lỗi Phase A

Nhóm: Duo

Người phụ trách: Hoàng Trung Hiếu - 2A202602945

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T18:59:52.909878+00:00.
Generation mode/fallback và điều kiện chạy: Chế độ llm hoàn toàn cho 50/50 câu hỏi (không dùng extractive fallback); điều kiện chạy thực tế trên Qdrant v1.9.7, embedding BAAI/bge-m3, BM25 với underthesea và reranker BAAI/bge-reranker-v2-m3.
Models generation/evaluator: Generation model gpt-4o-mini-2024-07-18; Evaluator models gpt-4o-mini và text-embedding-ada-002 từ m4.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.8250 | 0.6146 | 0.6000 |
| answer_relevancy | 0.6674 | 0.7417 | 0.5016 |
| context_precision | 0.9667 | 0.9500 | 0.8667 |
| context_recall | 0.9333 | 0.6583 | 0.6083 |
| avg_score | 0.8481 | 0.7412 | 0.6442 |

## Bottom 10 và nguyên nhân

| Rank | ID | Câu hỏi | Distribution | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 39 | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | multi_hop | 0.0000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 33 | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | multi_hop | 0.2500 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 42 | Thâm niên bao nhiêu năm thì được cộng thêm ngày phép? | adversarial | 0.3125 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 50 | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | adversarial | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 5 | 44 | Bao lâu phải đổi mật khẩu một lần? | adversarial | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 6 | 48 | Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không? | adversarial | 0.4375 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 7 | 5 | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | factual | 0.5000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 7 | Nghỉ phép không lương 20 ngày cần ai phê duyệt? | factual | 0.5000 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 9 | 9 | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | factual | 0.5000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 10 | 24 | Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu? | multi_hop | 0.6034 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

Đối chiếu answer, contexts và reference của các câu đại diện:
- Câu 39 (multi_hop, avg=0.0): Câu hỏi yêu cầu tổng hợp so sánh cả 2 phiên bản tài liệu mật khẩu v1.0 và v2.0. Pipeline lấy được context nhưng LLM gặp khó khăn trong việc đối chiếu chi tiết từng trường hợp song song và bị hallucinate do xung đột dữ liệu phiên bản, dẫn tới faithfulness bằng 0.
- Câu 42, 44 (adversarial): Đây là các bẫy câu hỏi không nói rõ phiên bản áp dụng (ví dụ mật khẩu v1 quy định 90 ngày, v2 quy định 60 ngày). Khi retrieval mang về cả 2 chunk chính sách cũ và mới, LLM chọn một con số mà không giải thích điều kiện phiên bản, hoặc nhầm lẫn giữa mốc thâm niên 5 năm và quy định cụ thể, dẫn đến mất điểm faithfulness nghiêm trọng.
- Câu 50 (adversarial): Câu hỏi bẫy về việc dùng VPN cá nhân bên ngoài. Mặc dù tài liệu ghi rõ chỉ được dùng VPN do công ty cấp, mô hình diễn đạt chưa bám sát phủ định của reference, khiến câu trả lời bị đánh giá thiếu chính xác so với ground truth.

## Matrix và nhóm yếu nhất

| Worst Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 4 | 12 | 4 | 20 |
| context_recall | 1 | 5 | 5 | 11 |
| context_precision | 1 | 0 | 1 | 2 |
| answer_relevancy | 14 | 3 | 0 | 17 |

Nhóm yếu theo mean thấp nhất: adversarial; mean: 0.6442.
Worst metric xuất hiện nhiều nhất: faithfulness; số câu: 20.

Giải thích: Nhóm adversarial có điểm trung bình thấp nhất (0.6442) so với factual (0.8481) và multi_hop (0.7412). Điều này bắt nguồn từ việc các câu hỏi adversarial cố tình khai thác các xung đột phiên bản chính sách (policy versioning) và điều kiện loại trừ. Về phân bố lỗi, `faithfulness` là metric yếu nhất với 20 câu chiếm worst_metric (đặc biệt chiếm tới 12/20 câu ở multi_hop), cho thấy pipeline gặp khó khăn trong việc giữ câu trả lời bám sát 100% ngữ cảnh khi phải tổng hợp thông tin từ nhiều nguồn hoặc giải quyết mâu thuẫn tài liệu.
Đề xuất khắc phục: Bổ sung metadata filtering theo phiên bản tài liệu hiệu lực mới nhất (active_version), đồng thời tinh chỉnh system prompt yêu cầu LLM nêu rõ phiên bản nếu phát hiện có nhiều văn bản xung đột.

## Bẫy adversarial và giới hạn

So sánh với factual (0.8481) và multi_hop (0.7412), nhóm adversarial có điểm thấp hơn rõ rệt (0.6442). Điểm số thấp này phản ánh chính xác các điểm yếu của hệ thống RAG cơ bản:
1. Version Conflict: Khi corpus chứa cả văn bản cũ (mat_khau_v1, nghi_phep_nam_v2023) và văn bản mới (mat_khau_v2, nghi_phep_nam_v2024), Hybrid Search trả về cả hai và LLM không có cơ chế tự động phân biệt hiệu lực thời gian nếu không có metadata filter chuyên biệt.
2. Negation Trap: Các câu hỏi dạng phủ định hoặc câu hỏi gài bẫy (nhân viên thử việc có được bảo hiểm không, có được dùng VPN ngoài không) đòi hỏi LLM phải nhận diện ranh giới nghiêm ngặt trong tài liệu thay vì suy diễn lỏng lẻo.
Giới hạn: Tập dữ liệu adversarial có kích thước 10 câu; mô hình generation là gpt-4o-mini-2024-07-18. Việc adversarial mean thấp hơn chứng minh bộ test đã phát hiện được lỗ hổng nhưng chưa khẳng định pipeline có thể tự sửa nếu không bổ sung tầng Guardrails và lọc phiên bản.
