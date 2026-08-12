# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu trả lời mang tính chào hỏi (chitchat/greeting), lời từ chối lịch sự, disclaimer hoặc kiến thức phổ quát chung không yêu cầu dẫn chứng từ context. | Câu trả lời tra cứu chính sách/học vụ/quy định bị bịa đặt (hallucination), sai lệch so với tài liệu gốc của nhà trường. | Siết chặt system prompt về grounding ("chỉ trả lời dựa trên context"), hạ temperature, kiểm tra chất lượng context retriever, thêm guardrail chặn hallucination. |
| Answer Relevance | Câu hỏi mơ hồ hoặc chứa prompt injection/trap và hệ thống chủ động hỏi lại để làm rõ (clarification) hoặc từ chối lịch sự (polite refusal). | Người dùng hỏi rõ ràng một quy trình cụ thể nhưng bot trả lời lạc đề sang chủ đề khác hoặc đưa thông tin lan man không đúng trọng tâm. | Cải thiện prompt hướng dẫn LLM tập trung vào core intent của câu hỏi, thêm bước Query Rewriting/Rephrasing, cung cấp few-shot examples về câu trả lời đúng trọng tâm. |
| Context Recall | Câu hỏi mở/suy luận tổng quan mà câu trả lời kỳ vọng dựa vào kiến thức nền chung, hoặc expected answer chỉ cần tóm tắt sơ bộ. | Câu hỏi yêu cầu đầy đủ các bước/điều kiện/hồ sơ bắt buộc nhưng retriever lấy thiếu các chunk quan trọng, làm mất ý cốt lõi. | Nâng cấp retriever (chuyển sang Hybrid Search: BM25 + Vector Search), tăng `top_k`, tối ưu chunk size/overlap, áp dụng Multi-query Retrieval / Query Expansion. |
| Context Precision | Số lượng chunk lấy về ít (`top_k` nhỏ) và toàn bộ các chunk đều nằm trọn trong context window của LLM mà không gây nhiễu. | Chunk chứa thông tin chính xác bị xếp ở vị trí cuối (rank thấp) hoặc bị chìm giữa nhiều chunk rác/nhiễu, khiến LLM bị "Lost in the Middle" hoặc sinh câu trả lời sai. | Bổ sung Reranker (Cross-Encoder / Rerank Model), tinh chỉnh embedding model theo domain, tối ưu semantic chunking để nâng cao độ liên quan của chunk ở top đầu. |
| Completeness | Người dùng yêu cầu tóm tắt ngắn gọn ("trả lời trong 1 câu"), hoặc câu hỏi chấp nhận câu trả lời dạng tóm lược cấp cao (high-level summary). | Câu hỏi hỏi quy trình nhiều bước/danh sách thủ tục bắt buộc nhưng câu trả lời bỏ sót các bước/ngoại lệ/thời hạn quan trọng dẫn tới người dùng thực hiện sai. | Cải thiện generation prompt yêu cầu trả lời đầy đủ theo checklist/step-by-step, kiểm tra Context Recall để đảm bảo nguồn cấp đủ thông tin, tăng `max_tokens` nếu câu trả lời bị cắt cụt. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Mục tiêu:** Kiểm tra xem LLM Judge có xu hướng ưu tiên câu trả lời xuất hiện ở vị trí đầu (hoặc thứ hai) trong bài toán so sánh cặp (Pairwise Comparison) hay không.
> - **Thiết kế 2 conditions:**
>   - **Condition 1 (Original Order):** Cung cấp Question cùng Pair (Answer A, Answer B) với thứ tự: Candidate 1 = Answer A, Candidate 2 = Answer B. Yêu cầu LLM Judge chấm điểm/chọn câu tốt hơn.
>   - **Condition 2 (Swapped Order):** Giữ nguyên Question và nội dung, nhưng đảo ngược vị trí trong prompt: Candidate 1 = Answer B, Candidate 2 = Answer A.
> - **Đo lường & Đánh giá:**
>   - Chạy trên tập dataset $N \ge 50 - 100$ cặp QA.
>   - Tính **Consistency Rate (Tỷ lệ nhất quán):** Tỷ lệ các trường hợp mà kết quả đánh giá không đổi dù hoán đổi vị trí (A thắng ở Cond 1 thì A vẫn phải thắng ở Cond 2).
>   - Tính **Positional Win Rate:** Tỷ lệ Candidate 1 (vị trí đầu) được chọn ở cả 2 conditions. Nếu tỷ lệ này chênh lệch đáng kể so với 50% (ví dụ > 60–70%), chứng tỏ tồn tại Position Bias nghiêm trọng.
> - **Biện pháp xử lý:** Áp dụng kỹ thuật Swap Evaluation (chấm 2 lần đảo vị trí rồi lấy điểm trung bình/chỉ công nhận khi thắng cả 2) hoặc chuyển sang Pointwise Evaluation (chấm độc lập từng câu kèm Rubric chi tiết).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Định nghĩa rõ tiêu chí Conciseness & Information Density:** Trong rubric quy định rõ ràng rằng điểm tối đa (5/5) yêu cầu "cung cấp đầy đủ thông tin cốt lõi nhưng ngắn gọn, súc tích, không chứa từ ngữ thừa/vòng vo". Đặt quy tắc trừ điểm rõ ràng nếu câu trả lời dài dòng, lặp ý hoặc thêm filler text.
> 2. **Chấm điểm theo Checklist ý chính (Key Factual Points):** Thiết kế rubric dạng danh sách các sự kiện/ý bắt buộc (Checklist criteria: Fact 1, Fact 2, Fact 3,...). Đạt đủ các key points là đạt điểm tối đa, không phụ thuộc vào số lượng từ hay độ dài đoạn văn.
> 3. **Quy định khung độ dài tham chiếu (Length Guidelines/Budget):** Cung cấp giới hạn độ dài kỳ vọng cho câu hỏi (ví dụ: "Độ dài tối ưu 50–150 từ; phạt 1 điểm nếu vượt quá 300 từ mà không bổ sung thêm giá trị thông tin mới").
> 4. **Tách biệt các tiêu chí đánh giá độc lập (Multi-dimensional Rubric):** Chia rubric thành các tiêu chí riêng biệt: *Accuracy*, *Completeness*, *Conciseness/Clarity* với trọng số rõ ràng, tránh để ấn tượng về độ dài làm lu mờ tính chính xác và súc tích.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Đảm bảo Alignment với tiêu chuẩn con người (Ground Truth Alignment):** LLM Judge có các bias nội tại (self-preference, verbosity bias, position bias, leniency/severity bias). Calibration giúp đảm bảo tiêu chí chấm của LLM đồng thuận với nhận định của chuyên gia con người (Domain Experts).
> 2. **Đo lường mức độ tương quan và độ tin cậy:** Cho phép tính toán các chỉ số thống kê định lượng như Cohen's Kappa (độ đồng thuận phân loại), Pearson/Spearman Correlation giữa điểm của LLM Judge và Human Annotators trên tập mẫu (50–200 samples) để chứng minh evaluator đủ tin cậy trước khi chạy trên quy mô lớn.
> 3. **Phát hiện lỗi phán đoán để cải thiện Prompt/Rubric:** Qua calibration, ta xác định được các ca bất đồng (Disagreements / False Positives / False Negatives), từ đó bổ sung Few-shot examples, làm rõ các trường hợp biên (edge cases) và tinh chỉnh thang điểm trong Rubric.
> 4. **Tối ưu chi phí và mở rộng quy mô an toàn (Safe Scalability):** Đánh giá thủ công tốn nhiều thời gian và chi phí, nhưng dùng LLM Judge mù quáng sẽ tạo rủi ro sai sót hệ thống. Calibration giúp xây dựng một công cụ đánh giá tự động vừa nhanh, rẻ vừa đảm bảo chất lượng tương đương chuyên gia.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | $\ge 0.85$ | Đây là rào chắn an toàn quan trọng nhất để ngăn chặn Hallucination. Nếu trợ lý trả lời sai lệch tài liệu nguồn (bịa đặt thông tin học vụ, học phí, ngày hạn), hậu quả đối với sinh viên và uy tín nhà trường rất nghiêm trọng. Cần chặn tuyệt đối các bản build vi phạm grounding. |
| Answer Relevance | $\ge 0.80$ | Đảm bảo trợ lý trực tiếp giải quyết đúng nhu cầu của người dùng, không trả lời lạc đề, vòng vo hay từ chối sai ngữ cảnh. Ngưỡng $\ge 0.80$ đảm bảo chất lượng trải nghiệm người dùng đạt chuẩn trước khi release. |
| Completeness | $\ge 0.75$ | Đảm bảo cung cấp đầy đủ các bước, giấy tờ và điều kiện cần thiết trong quy trình dịch vụ sinh viên. Có thể nới lỏng nhẹ so với Faithfulness vì một số câu hỏi có thể chấp nhận câu trả lời tóm lược nếu không bỏ sót thông tin cốt lõi, nhưng không được dưới 0.75 để tránh thiếu sót thông tin quan trọng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Đánh giá ngoại tuyến / Pre-deployment):**
>   - *Khi nào dùng:* Trong quá trình phát triển (development), chạy tự động trong CI/CD pipeline trước khi merge Pull Request, hoặc khi có thay đổi về Prompt, Model, RAG retrieval parameters, Chunking/Embedding.
>   - *Mục đích:* Đánh giá trên Golden Dataset cố định với chi phí thấp, tốc độ nhanh, lặp lại được (reproducible) để ngăn chặn lỗi hồi quy (regression) trước khi triển khai lên production.
> - **Online Evaluation (Đánh giá trực tuyến / Post-deployment / Production Monitoring):**
>   - *Khi nào dùng:* Khi hệ thống đang phục vụ lưu lượng người dùng thực tế (live production traffic).
>   - *Mục đích:* Giám sát chất lượng vận hành liên tục thông qua phản hồi người dùng (thumbs up/down, rating), theo dõi real-time metrics (latency, token cost, fallback/refusal rate) và chạy LLM-as-a-Judge trên tập mẫu log người dùng; giúp phát hiện data drift và các câu hỏi mới phát sinh trong thực tế.
> - **Human Review (Đánh giá thủ công bởi chuyên gia con người):**
>   - *Khi nào dùng:*
>     - Khi xây dựng, thẩm định và làm giàu Golden Dataset chuẩn ban đầu.
>     - Khi calibrate LLM Judge hoặc giải quyết các ca có sự bất đồng lớn giữa các hệ thống đánh giá.
>     - Định kỳ audit ngẫu nhiên (sampling 1–5% logs production) hoặc điều tra các ca khiếu nại / negative feedback nghiêm trọng.
>   - *Mục đích:* Đóng vai trò là chuẩn mực Ground Truth cao nhất để định chuẩn toàn bộ pipeline đánh giá tự động.

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
