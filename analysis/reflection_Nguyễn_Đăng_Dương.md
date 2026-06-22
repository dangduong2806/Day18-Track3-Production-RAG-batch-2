# Individual Reflection — Lab 18: Production RAG

**Tên:** Nguyễn Đăng Dương  
**Nhóm:** 139  
**Module phụ trách:** M1, M2, M3, M4, M5 và pipeline end-to-end

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** toàn bộ pipeline Production RAG gồm Chunking, Hybrid Search, Reranking, RAGAS Evaluation và Enrichment.
- **Các hàm/class chính đã viết hoặc hoàn thiện:**
  - M1 Chunking: `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`, `compare_strategies()`.
  - M2 Hybrid Search: `segment_vietnamese()`, `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()`, `HybridSearch`.
  - M3 Reranking: `CrossEncoderReranker`, `FlashrankReranker`, `benchmark_reranker()`.
  - M4 Evaluation: `evaluate_ragas()`, `failure_analysis()`, `save_report()`.
  - M5 Enrichment: `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `extract_metadata()`, `_enrich_single_call()`, `enrich_chunks()`.
- **Kết quả chạy pipeline:** sinh được `ragas_report.json` với 20 câu hỏi đánh giá.
- **Số tests pass:** chưa xác nhận lại bằng `pytest` trong phiên viết report vì `.venv310` hiện chưa cài `pytest`. Tuy nhiên pipeline đã chạy được và tạo report RAGAS.

## 2. Mapping bài giảng vào code

| Lecture Concept | Module | Hàm/class cụ thể | Observation |
|----------------|--------|------------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Chunk theo mức tương đồng giữa câu giúp giảm việc cắt rời ý nghĩa, nhưng cần threshold hợp lý để không tạo chunk quá vụn. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent-child chunking phù hợp với RAG vì có thể retrieve child nhỏ nhưng vẫn giữ context rộng qua parent. |
| Structure-aware chunking | M1 | `chunk_structure_aware()` | Với tài liệu policy có heading rõ ràng, giữ metadata section giúp truy xuất và phân tích lỗi dễ hơn. |
| BM25 + Dense fusion | M2 | `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 tốt với keyword chính xác, Dense tốt với ngữ nghĩa; RRF giúp kết hợp hai nguồn ranking mà không cần scale score giống nhau. |
| Vietnamese preprocessing | M2 | `segment_vietnamese()` | Với tiếng Việt, tách từ giúp BM25 ổn định hơn, đặc biệt với các cụm như “nghỉ phép”, “bảo hiểm sức khỏe”. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Reranker giúp tăng Context Precision vì đánh giá lại từng cặp query-document thay vì chỉ dựa vào vector/BM25 score ban đầu. |
| RAGAS metrics | M4 | `evaluate_ragas()` | RAGAS cho thấy retrieval tốt chưa đủ: Context Precision cao nhưng Faithfulness vẫn thấp nếu generation không bám context. |
| Diagnostic Tree | M4 | `failure_analysis()` | Mapping metric thấp nhất sang diagnosis giúp biết lỗi nằm ở retrieval, reranking hay generation. |
| Contextual enrichment | M5 | `contextual_prepend()`, `generate_hypothesis_questions()` | Enrichment có thể làm chunk dễ match hơn với query, nhưng cũng cần kiểm soát để tránh thêm nhiễu hoặc làm model bám vào phần summary sai. |

## 3. Kiến thức học được

- **Khái niệm quan trọng nhất:** Production RAG không chỉ là “embed rồi search”. Một pipeline tốt cần phối hợp chunking, hybrid retrieval, reranking, generation prompt và evaluation.
- **Điều bất ngờ nhất:** Production pipeline có Context Precision cao hơn baseline (0.9917 so với 0.9250), nhưng Faithfulness lại thấp hơn (0.6631 so với 0.8917). Điều này cho thấy lấy đúng context chưa đảm bảo câu trả lời đúng nếu prompt/generation chưa đủ chặt.
- **Kết nối với bài giảng:** Các failure trong `ragas_report.json` phản ánh đúng các nhóm lỗi trong lecture: retrieval miss, context conflict, hallucination, và multi-hop question chưa được decomposition.

## 4. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Môi trường chạy có nhiều lỗi hạ tầng trước khi pipeline ổn định, đặc biệt là Qdrant và Python environment.
- **Lỗi đã gặp:**
  - Qdrant không start do storage cũ không tương thích version: `unknown variant on_disk, expected mmap or in_ram_mmap`.
  - Python không kết nối được Qdrant: `WinError 10061`.
  - Khi chạy pipeline có lúc gặp `Segmentation fault` ở bước dense indexing.
  - Kích hoạt venv nhầm shell: chạy lệnh PowerShell trong bash dẫn đến `command not found`.
- **Cách giải quyết:**
  - Kiểm tra `docker compose ps`, `docker compose logs qdrant`, và test `curl http://localhost:6333` để xác nhận Qdrant thật sự chạy.
  - Reset hoặc nâng version Qdrant khi storage không tương thích.
  - Tạo lại virtual environment `.venv310` và phân biệt lệnh activate giữa PowerShell, CMD và bash.
  - Dựa trên `ragas_report.json` và `test_set.json` để viết failure analysis thay vì suy đoán thủ công.
- **Thời gian debug:** phần lớn thời gian debug nằm ở môi trường Docker/Qdrant và dependency Python; sau khi môi trường ổn định thì việc đọc RAGAS report và phân tích lỗi rõ ràng hơn.

## 5. Kết quả & Nhận xét RAGAS

| Metric | Naive Baseline | Production | Nhận xét |
|--------|---------------|------------|----------|
| Faithfulness | 0.8917 | 0.6631 | Giảm mạnh; lỗi chính nằm ở hallucination hoặc chọn nhầm policy cũ/mới. |
| Answer Relevancy | 0.7603 | 0.6923 | Giảm nhẹ; câu hỏi multi-hop dễ bị trả lời thiếu ý. |
| Context Precision | 0.9250 | 0.9917 | Cải thiện rõ; Hybrid Search + Reranking chọn context ít nhiễu hơn. |
| Context Recall | 0.9250 | 0.8667 | Giảm; một số câu cần nhiều evidence nhưng retrieval chưa lấy đủ. |

Nhận xét chính: pipeline Production cải thiện chất lượng context, nhưng phần generation cần được siết lại. Các lỗi nghiêm trọng nhất đều liên quan đến câu hỏi có version policy, điều kiện số, hoặc cần tổng hợp nhiều tài liệu.

## 6. Nếu làm lại

- **Sẽ làm khác điều gì:**
  - Lưu lại toàn bộ per-question answer và contexts vào report, không chỉ lưu failure summary. Điều này sẽ giúp điền phần `Got` trong failure analysis chính xác hơn.
  - Thêm metadata `version`, `year`, `status=current/deprecated` cho tài liệu policy để tránh nhầm chính sách cũ và mới.
  - Thêm query decomposition cho câu hỏi nhiều ý, ví dụ câu hỏi Senior cần vừa tính ngày phép vừa tìm khoảng lương.
  - Thêm prompt format bắt buộc: trả lời ngắn, nêu rule dùng, và nói rõ nếu context không đủ.
- **Module muốn thử tiếp:** M2 và M4. M2 vì retrieval quyết định chất lượng context, M4 vì evaluation giúp chỉ ra bottleneck thật thay vì chỉ cảm tính.

## 7. Action Plan cho project tiếp theo

### Project: Production RAG cho tài liệu nội bộ/policy

### Hiện tại
- Pipeline đã có chunking, hybrid search, reranking, enrichment và RAGAS evaluation.
- Known issues: hallucination ở policy có nhiều version, câu hỏi threshold số, và câu hỏi multi-hop.

### Plan áp dụng
1. [ ] **Chunking strategy:** dùng hierarchical + structure-aware chunking để vừa giữ section metadata vừa tránh mất ngữ cảnh.
2. [ ] **Search:** tiếp tục dùng BM25 + Dense + RRF, nhưng thêm metadata filter theo document version/status.
3. [ ] **Reranking:** dùng cross-encoder reranker cho top-k retrieved docs, đồng thời kiểm tra latency.
4. [ ] **Evaluation:** lưu per-question answer/context đầy đủ, chạy RAGAS và thêm manual error taxonomy.
5. [ ] **Enrichment:** dùng contextual prepend có kiểm soát, tránh summary quá dài hoặc thêm thông tin không có trong tài liệu gốc.
6. [ ] **Generation:** siết prompt theo dạng “chỉ dựa trên context”, “ưu tiên policy hiện hành”, “nếu có nhiều version thì chọn version mới nhất”.

### Timeline
- **Tuần 1:** chuẩn hóa metadata tài liệu và lưu version/status.
- **Tuần 2:** cải thiện retrieval + reranking, đo Context Precision/Recall.
- **Tuần 3:** thêm query decomposition và prompt mới cho generation.
- **Tuần 4:** chạy RAGAS, phân tích bottom failures, lặp lại tối ưu.

## 8. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) | Lý do |
|----------|---------------|-------|
| Hiểu bài giảng | 5 | Nắm được vai trò từng module trong Production RAG và đọc được ý nghĩa từng RAGAS metric. |
| Code quality | 4 | Pipeline chạy được và có module rõ ràng, nhưng cần lưu thêm per-question output để phân tích tốt hơn. |
| Debugging | 4 | Đã xử lý được nhiều lỗi môi trường như Qdrant, venv, dependency; vẫn cần setup reproducible hơn. |
| Problem solving | 5 | Biết dùng RAGAS/failure report để xác định bottleneck thay vì chỉ nhìn điểm tổng. |
| Reflection | 5 | Rút ra được action plan cụ thể cho lần cải thiện tiếp theo. |
