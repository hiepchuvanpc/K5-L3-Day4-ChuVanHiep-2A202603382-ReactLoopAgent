# Ghi chú phân tích · Day 04

Họ và tên: Chử Văn Hiệp
MSSV: 2A202603382

## 1. Case được chọn

- ID case: H6.
- Lý do chọn: H6 kiểm tra điều tra nhiều bước (multi-step investigation), khả năng lọc qua 40 bản ghi nhiễu giả lập, truy xuất incident liên quan qua liên kết, kiểm tra trích dẫn nguồn xác thực và nêu rõ giới hạn khi dữ liệu bị cắt (`truncated: true`).
- `eval_before.json`: H6 được chấm 98/100 (Factuality: 40, Completeness: 25, Evidence: 18, Boundaries: 15).

## 2. Bằng chứng trong trace

Trong H6 của `eval_before.json`:
- Step 1: Agent gọi `get_status` cho GitHub Actions và `get_incident_details` cho incident `h02n9572mx9y`. Kết quả incident thành công có URL `https://stspg.io/7pwv28hvkmz8`.
- Step 2: Agent gọi tiếp `get_incident_details` cho incident liên quan `xr3myzj7ms51`. Kết quả thành công có URL `https://stspg.io/pjgptg3cfc6n`.
- Update của incident liên quan trả về có `truncated: true`; nội dung dừng lại tại cụm từ “The code change”. Do đó câu trả lời chỉ được phép kết luận từ phần dữ liệu đã nhận, không được suy diễn phần nội dung bị cắt.
- Step 3: Model đưa ra câu trả lời nhưng chưa tích hợp đầy đủ URL nguồn của các incident đã truy xuất. Cơ chế validation phát hiện và ghi event `validation_missing_source` với 2 URL trên vào trace, đồng thời gửi phản hồi yêu cầu model sửa đổi và bổ sung nguồn.
- Step 4: Model bổ sung URL và hoàn tất câu trả lời. Tuy nhiên, việc thiếu trích dẫn nguồn ở lượt đầu khiến tiêu chí Evidence bị trừ 2 điểm (18/20), điểm tổng H6 là 98/100 và tốn 4 lượt gọi model.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Trong `agent.py` (vòng lặp `run_agent`):
     - Xây dựng cơ chế **Source Validation** (chốt chặn kiểm tra nguồn): Theo dõi và lưu danh sách URL incident được lấy thành công từ `get_incident_details` vào danh sách `incident_urls`.
     - Khi model trả về phản hồi cuối cùng (`finish == 'stop'`), hàm `extract_urls` sẽ trích xuất và chuẩn hóa tất cả URL trong câu trả lời để so sánh với `incident_urls`.
     - Nếu phát hiện thiếu URL (`missing_urls`), agent ghi nhận event `validation_missing_source` vào trace và gửi tin nhắn phản hồi nhắc model bổ sung đầy đủ, chính xác các URL này vào sát nhận định tương ứng trước khi hết ngân sách `max_steps`.
     - Khi toàn bộ URL đã xuất hiện đầy đủ, ghi event `validation_passed` vào trace và trả về `status: "completed"`.
  2. Trong `agent.py` (`SYSTEM_PROMPT`):
     - Bổ sung quy tắc bắt buộc về trích dẫn nguồn: Hướng dẫn model đặt URL của incident ngay sau nhận định mà nội dung đó hỗ trợ, không dồn tất cả nguồn về cuối bài.
     - Củng cố quy tắc giới hạn dữ liệu: Khi gặp `truncated: true`, chỉ được kết luận dựa trên phần văn bản thực tế nhận được và nêu rõ đoạn bị cắt; phân biệt rõ ngày tạo bản ghi (`created_at`) với thời điểm bắt đầu sự cố; giữ nguyên các thông số định lượng quan trọng.
- **Vì sao chọn thay đổi này:**
  - Ở lần chạy đầu tiên, dù chốt chặn validation bằng code đã giúp agent tự sửa lỗi thiếu nguồn qua một lượt gọi bổ sung, nhưng việc model ban đầu không chủ động dẫn nguồn khiến điểm tiêu chí Evidence của LLM judge bị giảm (18/20 điểm ở H6) và gây lãng phí thêm 1 lượt gọi model (tốn thêm token và tăng độ trễ).
  - Việc kết hợp hướng dẫn chi tiết trong `SYSTEM_PROMPT` cùng chốt chặn code validation giúp model trích dẫn nguồn chuẩn xác ngay từ lượt đầu, tiết kiệm 1 model call ở H6, nâng điểm Evidence lên 20/20 (tổng điểm H6 đạt 100/100) và tăng tính nhất quán trên toàn bộ tập đánh giá.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (22/22 case) | 99.73 / 100 | 99.95 / 100 |
| Điểm case đã chọn (H6) | 98 / 100 | 100 / 100 |
| Số model calls (H6 / Toàn bộ eval) | 4 calls (H6) / 61 calls (tổng) | 3 calls (H6) / 56 calls (tổng) |
| Token (H6 / Toàn bộ eval) | 10,468 tokens (H6) / 69,709 tokens (tổng) | 9,834 tokens (H6) / 122,415 tokens (tổng) |
| Chi phí (H6 / Toàn bộ eval) | $0.00202 (H6) / $0.02084 (tổng) | $0.00169 (H6) / $0.02007 (tổng) |

- **Lỗi đã giải quyết:**
  - Case H6: Khắc phục triệt để lỗi thiếu trích dẫn URL nguồn của incident liên quan ở lượt trả lời đầu; model dẫn nguồn chính xác cạnh từng nhận định; không còn kích hoạt `validation_missing_source` mà đạt `validation_passed` ngay ở Step 3. Điểm H6 tăng từ 98 lên 100/100.
  - Các case khác (E5, A5, H8) cũng được cải thiện từ 98–99 lên 100/100 nhờ tuân thủ chặt chẽ các chỉ dẫn về giới hạn dữ liệu và trích dẫn nguồn.
  - Toàn bộ eval giảm được 5 lượt gọi model không cần thiết (từ 61 xuống 56 calls), chi phí tổng thể giảm từ $0.0208 xuống $0.0200.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Case H1 có sự biến động nhẹ từ 100 xuống 99 điểm (do tính ngẫu nhiên nhẹ trong cách diễn đạt và chấm điểm của LLM judge).
  - Giới hạn dữ liệu không thể vượt qua: Khi tool trả về `truncated: true`, agent buộc phải dừng kết luận tại điểm bị cắt và không thể có thêm thông tin chi tiết nếu API không hỗ trợ phân trang hoặc dữ liệu snapshot bị giới hạn.

## 5. Giới hạn

- **Giới hạn của điểm LLM Judge:** LLM Judge (sử dụng `openai/gpt-6-luna`) là một phương pháp đánh giá xấp xỉ. Đánh giá dựa trên văn bản tự nhiên có thể tồn tại sai số ngẫu nhiên (chênh lệch 1-2 điểm giữa các lần chạy như trường hợp H1 và H6) và có thể mang thiên lệch tự đánh giá (self-preference bias) do cả agent và judge đều dùng chung một họ mô hình.
- **Giới hạn của Snapshot:** Dữ liệu snapshot được đóng băng tại ngày 09/11/2026. Agent chỉ hoạt động chính xác dựa trên dữ liệu tĩnh này, không thể phản ánh tình trạng sự cố theo thời gian thực (live) nếu không có kết nối internet và dữ liệu mới nhất từ API.
- **Giới hạn của việc chạy một lần mỗi phiên bản:** Một lần chạy duy nhất không thể hiện hết độ ổn định (variance) của mô hình ngôn ngữ lớn do tính bất định (non-deterministic). Để có kết luận mang tính thống kê vững chắc hơn, cần thực hiện lặp lại nhiều lần (repeated runs) để tính toán độ lệch chuẩn.