# Failure Analysis — Lab 18: Production RAG

**Nhóm:** 139
**Thành viên:** Nguyễn Đăng Dương

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness |0.8917|0.6631|0.2286|
| Answer Relevancy |0.7603|0.6923|0.068|
| Context Precision |0.9250|0.9917|-0.0667|
| Context Recall |0.9250|0.8667|0.0583|

Nhìn chung, Production cải thiện Context Precision so với baseline, nghĩa là các context được chọn ít nhiễu hơn. Tuy nhiên Faithfulness, Answer Relevancy và Context Recall giảm, cho thấy pipeline vẫn gặp vấn đề ở bước generation, xử lý policy có nhiều version, và các câu hỏi multi-hop cần kết hợp nhiều tài liệu.

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Câu trả lời không đủ faithful với policy hiện hành; có khả năng bị lẫn với quy định cũ 90 ngày hoặc suy diễn ngoài context.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? có thể đúng nhưng có nhiễu version cũ/mới → Query OK → Root cause: generation không ưu tiên policy hiện hành.
- **Root cause:** Câu hỏi có thông tin conflict giữa chính sách cũ và mới. Pipeline retrieve được context có độ chính xác cao, nhưng LLM chưa được ràng buộc đủ mạnh để chọn version mới nhất.
- **Suggested fix:** Siết prompt để chỉ trả lời theo chính sách hiện hành/latest version; giảm temperature; thêm metadata filter hoặc reranking ưu tiên `v2.0`.

### #2
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo chính sách v2024 hiện hành, nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm. Chính sách cũ v2023 yêu cầu 5 năm.
- **Got:** Câu trả lời không bám chắc vào chính sách v2024; có thể nhầm mốc 3 năm với mốc 5 năm của chính sách cũ.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? có thể chứa cả policy cũ và mới → Query OK → Root cause: model chọn sai version/chưa xử lý thông tin bị thay thế.
- **Root cause:** Retrieval có thể đưa vào cả quy định v2023 và v2024. Khi context có nhiều version, generator không có rule rõ ràng để loại bỏ chính sách đã hết hiệu lực.
- **Suggested fix:** Thêm metadata `year/version/status`; rerank ưu tiên document hiện hành; prompt yêu cầu nêu rõ chính sách cũ đã bị thay thế.

### #3
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** Câu trả lời không phủ định rõ ràng hoặc suy diễn theo quyền lợi chung của nhân viên, dẫn đến không faithful với điều kiện "thử việc".
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? có khả năng có đoạn về PVI và thử việc → Query OK → Root cause: generation không giữ điều kiện loại trừ.
- **Root cause:** Câu hỏi dạng yes/no cần trả lời dứt khoát. Model có xu hướng tổng quát hóa benefit cho "nhân viên" thay vì bám điều kiện cụ thể "nhân viên thử việc".
- **Suggested fix:** Prompt bắt buộc trả lời trước bằng "Có/Không"; yêu cầu trích điều kiện áp dụng; tăng trọng số/rerank cho keyword "thử việc" và "PVI".

### #4
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Câu trả lời không khớp đầy đủ với câu hỏi; có thể chỉ trả lời một phần về ngày phép hoặc một phần về khoảng lương.
- **Worst metric:** Answer Relevancy = 0.0
- **Error Tree:** Output sai/thiếu → Context đúng? cần nhiều context khác nhau → Query OK nhưng là multi-hop → Root cause: query chưa được tách thành các ý nhỏ.
- **Root cause:** Câu hỏi yêu cầu kết hợp hai loại thông tin: công thức ngày phép theo thâm niên và salary band Senior. Pipeline hiện tại retrieve/rerank theo một query duy nhất nên dễ thiếu một trong hai phần.
- **Suggested fix:** Thêm query decomposition: tách thành sub-query về phép năm và sub-query về lương Senior; merge context; prompt yêu cầu trả lời đủ từng ý.

### #5
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Câu trả lời không faithful với ngưỡng phê duyệt; có thể nhầm sang cấp Director của khoảng 5-50 triệu hoặc thiếu kết luận CEO.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? có thể chứa nhiều ngưỡng tiền → Query OK → Root cause: model áp sai điều kiện số.
- **Root cause:** Câu hỏi yêu cầu reasoning trên threshold: 55 triệu > 50 triệu. Nếu context có nhiều mức phê duyệt, generator dễ chọn nhầm rule gần kề.
- **Suggested fix:** Prompt yêu cầu kiểm tra điều kiện số trước khi trả lời; rerank ưu tiên chunk chứa "trên 50 triệu"; thêm rule-based post-check cho câu hỏi dạng threshold.

## Case Study (cho presentation)

**Question chọn phân tích:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?

**Error Tree walkthrough:**
1. Output đúng? → Chưa đúng/thiếu, vì câu trả lời cần đồng thời có số ngày phép và khoảng lương.
2. Context đúng? → Có khả năng context liên quan được retrieve, vì Context Precision của production rất cao (0.9917), nhưng Context Recall giảm còn 0.8667 nên vẫn có thể thiếu một phần evidence.
3. Query rewrite OK? → Chưa tốt. Đây là câu hỏi multi-hop gồm hai intent: tính ngày phép theo thâm niên và tìm salary band của Senior.
4. Fix ở bước: Query decomposition + retrieval/reranking theo từng sub-query + answer synthesis bắt buộc trả lời đủ các ý.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm bước tách câu hỏi phức hợp thành các sub-query nhỏ.
- Retrieve riêng policy nghỉ phép và policy lương, sau đó merge context trước khi generation.
- Cập nhật prompt để ưu tiên chính sách hiện hành và bắt buộc trả lời đủ từng phần của câu hỏi.
- Thêm metadata filter/reranking theo version để giảm lỗi nhầm chính sách cũ và mới.
