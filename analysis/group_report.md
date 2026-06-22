# Group Report — Lab 18: Production RAG

**Nhóm:** 139 - Nguyễn Đăng Dương  
**Ngày:** 22/06/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Nguyễn Đăng Dương | M1: Chunking | ☑ | Chưa chạy lại trong phiên này* |
| Nguyễn Đăng Dương | M2: Hybrid Search | ☑ | Chưa chạy lại trong phiên này* |
| Nguyễn Đăng Dương | M3: Reranking | ☑ | Chưa chạy lại trong phiên này* |
| Nguyễn Đăng Dương | M4: Evaluation | ☑ | Chưa chạy lại trong phiên này* |
| Nguyễn Đăng Dương | M5: Enrichment | ☑ | Chưa chạy lại trong phiên này* |

\* Ghi chú: pipeline đã chạy và sinh được `ragas_report.json`, nhưng trong môi trường `.venv310` hiện tại chưa có `pytest`, nên chưa xác nhận lại test count bằng lệnh `pytest` trong phiên viết report.

## Kết quả RAGAS

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8917 | 0.6631 | 0.2286 |
| Answer Relevancy | 0.7603 | 0.6923 | 0.0680 |
| Context Precision | 0.9250 | 0.9917 | -0.0667 |
| Context Recall | 0.9250 | 0.8667 | 0.0583 |

Ghi chú: cột Δ trong report đang tính theo công thức `Naive - Production`. Vì vậy Δ âm ở Context Precision nghĩa là Production tốt hơn baseline ở metric này.

## Key Findings

1. **Biggest improvement:** Context Precision tăng từ 0.9250 lên 0.9917. Điều này cho thấy pipeline Production với Hybrid Search + Reranking chọn context ít nhiễu hơn baseline; các đoạn được đưa vào LLM nhìn chung liên quan hơn với câu hỏi.
2. **Biggest challenge:** Faithfulness giảm mạnh từ 0.8917 xuống 0.6631. Phần lớn Bottom failures có `worst_metric = faithfulness` và diagnosis là `LLM hallucinating`, đặc biệt ở các câu có policy cũ/mới hoặc điều kiện số.
3. **Surprise finding:** Production retrieve context chính xác hơn nhưng answer quality không tăng tương ứng. Điều này cho thấy bottleneck không chỉ nằm ở retrieval, mà còn ở generation: prompt chưa đủ chặt, chưa ưu tiên policy hiện hành, và chưa xử lý tốt câu hỏi multi-hop.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):** Production đạt Context Precision 0.9917, cao hơn baseline 0.9250. Tuy nhiên Faithfulness giảm 0.2286 điểm, Answer Relevancy giảm 0.0680 điểm, Context Recall giảm 0.0583 điểm.
2. **Biggest win — module nào, tại sao:** M2 Hybrid Search kết hợp BM25 + Dense và M3 Reranking là điểm mạnh nhất, vì giúp context được chọn chính xác hơn. Evidence là Context Precision gần như đạt 1.0.
3. **Case study — 1 failure, Error Tree walkthrough:** Chọn câu “Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?”. Câu này có `worst_metric = answer_relevancy`, vì cần kết hợp hai thông tin: phép năm theo thâm niên và salary band Senior. Pipeline chưa tách câu hỏi thành sub-query nên câu trả lời dễ thiếu một phần.
4. **Next optimization nếu có thêm 1 giờ:** Thêm query decomposition cho câu hỏi multi-hop; thêm metadata filter theo version/status để ưu tiên policy hiện hành; siết prompt yêu cầu chỉ trả lời dựa trên context, nêu rõ rule được dùng và tránh dùng chính sách đã bị thay thế.
