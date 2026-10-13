# Đánh giá LLM Judge Phase B

Nhóm: Duo

Người phụ trách: Nguyễn Việt Hoàng Hải - 2A202602967

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T19:00:57.763055+00:00.
Models generation/judge: Generation model gpt-4o-mini-2024-07-18; Judge model gpt-4o-mini.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ
fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

| ID | Candidate A | Candidate B | Winner P1 | Winner P2 (quy đổi) | Final Winner | Consistent | Điểm P1 (A/B) | Điểm P2 (A/B) |
|---|---|---|---|---|---|---|---|---|
| 1 | Nghỉ 3 ngày làm việc khi kết hôn | Nghỉ 3 ngày làm việc có lương khi kết hôn | B | B | B | True | 0.8 / 1.0 | 0.8 / 1.0 |
| 5 | Cần Tổng Giám đốc (CEO) phê duyệt | Cần Giám đốc phòng ban phê duyệt | A | A | A | True | 1.0 / 0.0 | 1.0 / 0.0 |
| 12 | Thưởng Tết tối thiểu 1 tháng lương (nêu thâm niên 6 tháng) | Thưởng Tết tối thiểu 1 tháng lương | A | A | A | True | 1.0 / 0.8 | 1.0 / 0.8 |
| 21 | 18 ngày phép, hưởng lương trong thời gian nghỉ | 18 ngày phép, lương Senior từ 20 đến 35 triệu | A | B | tie | False | 1.0 / 0.8 | 0.8 / 1.0 |
| 41 | Nhân viên chính thức được 15 ngày phép (v2024) | Nhân viên được nghỉ 12 ngày phép (v2023) | A | A | A | True | 1.0 / 0.0 | 1.0 / 0.3 |

Position bias count/rate: count = 1, rate = 0.2 (20%).
Verbosity win rate: 1.0 (100%); decisive cases: 4.

Giải thích:
- Trong 5 cặp so sánh, có 4/5 cặp đạt sự nhất quán vị trí giữa 2 lượt (case 1, 5, 12, 41). Riêng case 21 xảy ra position inconsistency (lượt 1 chọn A đứng trước, lượt 2 chọn B đứng trước), cho thấy LLM judge bị dao động khi cả hai câu trả lời đều có ưu điểm riêng (A nêu rõ việc hưởng lương, B bổ sung khoảng lương cụ thể). Kỹ thuật swap-and-average đã tự động hạ kết quả về `tie` để đảm bảo khách quan.
- Về Verbosity win rate = 1.0: Cả 4 case phân định thắng bại (1, 5, 12, 41), bên thắng đều có độ dài câu dài hơn. Tuy nhiên, khi đối chiếu nội dung, bên thắng dài hơn là do cung cấp đúng và đủ điều kiện chính sách (ví dụ case 1 làm rõ 'có lương', case 12 bổ sung điều kiện 'từ 6 tháng', case 41 phân biệt phiên bản mới 2024 thay vì phiên bản cũ 2023), chứ không đơn thuần chỉ là thiên vị độ dài hình thức.

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json;
không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference.
Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

| Question ID | Model Answer gốc | Nhãn Judge (0/1) | Reasoning | Human Label | Agree |
|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | Chính xác và đầy đủ theo reference, không thiếu ý thiết yếu. | 1 | Có |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | Sai vì cần Tổng Giám đốc phê duyệt cho đơn hàng trên 50 triệu. | 0 | Có |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | Chính xác và đầy đủ theo reference, không thiếu ý thiết yếu. | 1 | Có |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | Chính xác theo reference, nêu rõ số ngày phép và khoảng lương Senior. | 1 | Có |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | Chính xác theo reference, nêu rõ số tiền và lý do cam kết dưới 1 năm. | 1 | Có |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | Sai vì thiếu Kế toán trưởng phê duyệt và cách tính phí phạt sai. | 0 | Có |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | Cung cấp đúng số ngày phép và tổng phụ cấp hàng tháng theo chính sách. | 1 | Có |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | Sai vì dựa trên chính sách cũ v2023 (chính sách v2024 là 15 ngày). | 0 | Có |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | Chính xác theo reference, nêu rõ không có phép năm và phải nghỉ không lương. | 1 | Có |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | Sai vì chính sách cấm sử dụng VPN cá nhân, bắt buộc dùng VPN công ty. | 0 | Có |

Cohen’s κ: 1.0 (hoàn hảo, đạt mốc bonus >0.6).
Observed/chance agreement và diễn giải:
- Tỷ lệ đồng thuận quan sát: P_o = 10/10 = 1.0 (100% các nhãn trùng khớp hoàn toàn với con người).
- Tỷ lệ đồng thuận ngẫu nhiên: P_e = (6/10 * 6/10) + (4/10 * 4/10) = 0.36 + 0.16 = 0.52.
- Hệ số Cohen's kappa: κ = (1.0 - 0.52) / (1.0 - 0.52) = 1.0.
Theo thang đo Landis-Koch, giá trị κ = 1.0 biểu thị mức độ đồng thuận hoàn hảo (Almost Perfect Agreement) giữa LLM Judge và chuyên gia con người trên tập kiểm định 10 câu hỏi chuẩn.

## Giới hạn và cách dùng judge

Mô hình judge được sử dụng là gpt-4o-mini với prompt đánh giá nhị phân chặt chẽ bám sát reference ground truth. Mặc dù đạt κ = 1.0 trên tập 10 mẫu kiểm chuẩn, hệ thống vẫn tồn tại các giới hạn kỹ thuật:
1. Kích thước tập hiệu chuẩn nhỏ (10 câu), cần mở rộng tập nhãn người lên hàng trăm mẫu trong môi trường production để đánh giá độ tin cậy bao quát hơn.
2. Pairwise judge vẫn xuất hiện position inconsistency (20%) ở các câu hỏi phức tạp đa ý (như câu 21), do đó bắt buộc phải áp dụng quy trình swap-and-average trong pipeline thực tế để loại bỏ thiên kiến vị trí.
3. Không xem LLM Judge là thước đo chân lý tuyệt đối: Cần kết hợp Human-in-the-loop (HITL) để định kỳ kiểm định lại các quyết định của judge, đặc biệt là với các câu hỏi nhạy cảm liên quan đến tuân thủ pháp lý và chính sách nhân sự.
