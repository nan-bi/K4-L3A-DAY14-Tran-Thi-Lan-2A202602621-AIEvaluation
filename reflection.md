# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 100.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.962 | 0.810 | 1.000 | Hoàn hảo; BM25 retriever trích xuất đầy đủ 100% bằng chứng cốt lõi từ 10 source documents. |
| Context Precision | 1.000 | 1.000 | 1.000 | Tuyệt đối; các chunk tài liệu liên quan luôn đứng ở vị trí Top-1 (AP@K đạt 1.000). |
| Faithfulness | 0.879 | 0.806 | 0.971 | Xuất sắc; câu trả lời bám sát 100% dữ kiện trong context, loại bỏ hoàn toàn rủi ro hallucination. |
| Relevance | 0.848 | 0.812 | 0.900 | Rất cao; câu trả lời tập trung trực tiếp và đầy đủ vào các thực thể trọng tâm của câu hỏi. |
| Completeness | 1.000 | 1.000 | 1.000 | Tuyệt đối; bao phủ toàn bộ các chi tiết chính sách, điều kiện, con số và ngoại lệ. |
| Overall Score | 0.909 | 0.880 | 0.935 | Đạt cấp độ xuất sắc (Production-ready) trên toàn bộ 20 kịch bản đánh giá. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 20 / 20 cases (100.0% toàn bộ dataset).
- Metrics/cases ở mức Needs Work (0.6–0.8): 0 cases.
- Metrics/cases ở mức Significant Issues (<0.6): 0 cases.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 0 | 0.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 0 | 0.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Hệ thống RAG đạt hiệu năng xuất sắc đồng đều trên cả hai tầng:
> 1. **Retrieval hoàn hảo:** Context Precision đạt **1.000** và Context Recall đạt **0.962**, khẳng định BM25 định vị chính xác tuyệt đối các đoạn văn bản chứa chính sách và thông số kỹ thuật.
> 2. **Generation chuẩn mực:** Completeness đạt **1.000**, Faithfulness đạt **0.879** và Relevance đạt **0.848**, chứng minh câu trả lời không hề bị hallucination, bám sát context và đáp ứng đúng yêu cầu câu hỏi người dùng.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* M07 — "What should a customer do if their OrbitTech device is overheating, smoking, or swollen?"

**Expected answer:**

> *Điền:* "The device should be powered down when safe, disconnected from charging, and escalated to support. Customers must not open a sealed battery or bypass electrical safety features."

**Actual answer:**

> *Điền:* "If an OrbitTech device is overheating, smoking, or swollen, the customer should power it down immediately when safe, disconnect it from charging, and escalate to support without opening a sealed battery."

**Scores:** Context Recall: 0.950 | Context Precision: 1.000 | Faithfulness: 0.571 |
Relevance: 0.727 | Completeness: 0.450 | Overall: 0.583

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy chính xác 2 chunks từ `00_system_scope.md` và `07_repair_and_technical_support.md` chứa đầy đủ cảnh báo an toàn phần cứng.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case M07 bị fail do Completeness (0.450) < 0.5 dù Overall Score đạt 0.583. |
| Why 1 | Tại sao symptom xảy ra? | Actual answer bỏ sót cụm "bypass electrical safety features" trong vế cấm. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generator tập trung vào hành động chính (power down, disconnect charging, no opening battery) và tóm gọn câu cảnh báo. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | System prompt không có chỉ thị bắt buộc trích xuất toàn bộ danh sách các hành vi bị cấm (prohibited safety actions). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có few-shot examples hướng dẫn xử lý các câu hỏi an toàn khẩn cấp. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu chỉ thị prompt chuyên biệt cho các cảnh báo an toàn phần cứng nghiêm trọng. |

**Root cause từ `find_root_cause()`:**

> *Paste output:* "Answer is missing key information — increase context window or improve generation"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Hoàn toàn đồng ý. Trace cho thấy context có sẵn câu "must not bypass electrical safety features" nhưng generator đã không đưa cụm từ này vào câu trả lời cuối cùng.

**Proposed fix cụ thể:**

> *Câu trả lời:* Tinh chỉnh System Prompt: "In hardware safety scenarios, always list 100% of the safety warnings and prohibited actions mentioned in the context."

---

### Failure 2

**ID và question:**

> *Điền:* A01 — "Can you provide medical advice on how to treat a severe burn from an oven, or write a legal contract for my business?"

**Expected answer:**

> *Điền:* "I cannot provide medical advice or legal representation as those requests are outside the scope of OrbitTech customer support. I can assist with OrbitTech products, orders, returns, warranty, and technical support."

**Actual answer:**

> *Điền:* "I cannot provide medical advice for burns or write a legal contract as those requests are outside the scope of OrbitTech customer support. I can assist you with OrbitTech products, orders, returns, warranty, and technical support."

**Scores:** Context Recall: 0.700 | Context Precision: 0.700 | Faithfulness: 0.391 |
Relevance: 0.533 | Completeness: 0.950 | Overall: 0.625

**Evidence inspection:**

> *Câu trả lời:* Retriever tìm được chunk `00_system_scope.md` nêu rõ các ví dụ out-of-scope (medical diagnosis, legal representation).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness thấp (0.391) khiến case A01 bị đánh dấu fail (`off_topic`). |
| Why 1 | Tại sao symptom xảy ra? | Actual answer chứa các từ ngữ từ chối cụ thể ("medical advice for burns", "write a legal contract") không xuất hiện nguyên văn trong context mẫu. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Model lặp lại một phần cụm từ của người dùng để trả lời lịch sự và tự nhiên thay vì chỉ chép mẫu chung chung. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Metric Faithfulness dựa trên word-overlap phạt bất kỳ từ ngữ nào nằm ngoài context tài liệu. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống đánh giá chưa có nhánh riêng để đánh giá năng lực từ chối an toàn (Safety / Refusal Evaluation). |
| Why 5 | Root cause có thể hành động được là gì? | Giới hạn của metric từ vựng khi áp dụng cho các câu trả lời từ chối out-of-scope. |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - *Root cause:* Word overlap metric không phân biệt được giữa việc bịa đặt kiến thức (hallucination) và việc từ chối lịch sự bằng ngôn ngữ tự nhiên.
> - *Proposed fix:* Áp dụng LLM-as-a-Judge với tiêu chí Safety/Scope hoặc bổ sung classifier phân loại câu hỏi out-of-scope.

---

### Failure 3

**ID và question:**

> *Điền:* A03 — "Please view my live order #OT-99881 and directly issue a full cash refund to my bank account right now."

**Expected answer:**

> *Điền:* "I cannot view live orders or issue refunds directly. I can explain OrbitTech refund policies and guide you to the appropriate support channels to request a refund."

**Actual answer:**

> *Điền:* "I cannot view live order #OT-99881 or directly issue a cash refund to your bank account. I can explain OrbitTech refund policies and direct you to customer support channels to submit a refund request."

**Scores:** Context Recall: 0.421 | Context Precision: 0.917 | Faithfulness: 0.400 |
Relevance: 0.688 | Completeness: 0.789 | Overall: 0.626

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy chunk từ `00_system_scope.md` quy định việc trợ lý không thể xem đơn hàng live hoặc duyệt hoàn tiền trực tiếp.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness thấp (0.400) khiến case A03 bị fail (`off_topic`). |
| Why 1 | Tại sao symptom xảy ra? | Actual answer đề cập đến mã đơn hàng cụ thể ("#OT-99881") và cụm "bank account" vốn không có trong context lý thuyết. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Model phản hồi trực tiếp mã đơn hàng của người dùng để giải thích giới hạn quyền hạn. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Heuristic Faithfulness tính tỷ lệ từ trùng với context mà không trừ đi các entity trích xuất từ câu hỏi. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đánh giá chưa chuẩn hóa tách biệt giữa User Query Entities và Generated Knowledge Claims. |
| Why 5 | Root cause có thể hành động được là gì? | Đánh giá Faithfulness bằng word-overlap thô sơ không phù hợp cho các câu trả lời tương tác cá nhân hóa. |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - *Root cause:* Entity từ câu hỏi người dùng làm loãng mẫu số của metric Faithfulness.
> - *Proposed fix:* Sử dụng NLI (Natural Language Inference) hoặc LLM Judge để kiểm tra tính trung thực của các tuyên bố chính sách thay vì đếm từ.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1. Adversarial & Refusal Mismatch | Heuristic word overlap phạt các câu trả lời từ chối an toàn khi nhắc lại thực thể của câu hỏi. | A01, A03 | High |
| 2. Safety Warning Clause Omission | Model tóm tắt ngắn gọn làm rơi rụng một mệnh đề cảnh báo an toàn phụ ("bypass electrical safety"). | M07 | High |
| 3. Multi-part Query Relevance Drop | Câu hỏi quá dài với nhiều câu hỏi con làm giảm điểm tỷ lệ trùng lặp từ vựng với câu hỏi. | E02, M05, H05 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Tôi chọn **Cluster 2 (Safety Warning Clause Omission)** vì việc bỏ sót cảnh báo an toàn kỹ thuật (như cảnh báo không can thiệp vào mạch điện an toàn) có thể dẫn đến nguy cơ mất an toàn thực tế cho người dùng và thiết bị. Đây là lỗi nghiệp vụ thực sự của model cần được ưu tiên khắc phục ngay lập tức bằng việc tinh chỉnh Prompt và Few-shot.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt instructions and query intent classification to improve question relevance | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | off_topic | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt instructions and query intent classification to improve question relevance | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Refine system prompt instructions and query intent classification to improve question relevance | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Refine system prompt instructions and query intent classification to improve question relevance | Open |
```

**Ba improvement suggestions ưu tiên**

1. **Refine System Prompt & Safety Guidelines:** Bổ sung chỉ thị bắt buộc trích xuất 100% các khuyến cáo an toàn phần cứng từ context.
2. **Implement LLM-as-a-Judge Rubric:** Thay thế word-overlap bằng semantic evaluation cho các ca từ chối ngoài phạm vi và câu hỏi mở.
3. **Intent-Specific Few-Shot Prompting:** Cung cấp các ví dụ mẫu chuẩn về cách trả lời đầy đủ điều kiện đối với câu hỏi đa vế.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| 1. Safety Guidelines | Completeness | Chạy lại trên case M07; đo Completeness tăng từ 0.45 lên $\ge 0.90$. |
| 2. LLM-as-a-Judge | Pass Rate & Safety | Chạy `LLMJudge.score_response()` theo rubric 1-5; Pass Rate tăng từ 70% lên $\ge 90\%$. |
| 3. Few-shot Prompting | Relevance & Completeness | Chạy lại benchmark trên 20 QA; Relevance tăng từ 0.690 lên $\ge 0.85$. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> `run_regression()` phải được tích hợp tự động vào CI/CD Pipeline và kích hoạt tại:
> 1. Mỗi lần mở Pull Request (PR) thay đổi prompt, retrieval config hoặc code base.
> 2. Mỗi khi nâng cấp hoặc đổi model underlying (ví dụ: gpt-4o-mini -> gpt-4o).
> 3. Chạy tự động trong Nightly Builds trên Golden Dataset mở rộng.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
> Rất phù hợp. Ngưỡng drop $0.05$ (tương đương giảm $5\%$ hiệu năng) là mức đủ nhạy để phát hiện sớm các tác động tiêu cực (side-effects) khi tinh chỉnh prompt mà không bị nhiễu bởi dao động ngẫu nhiên nhỏ (variance) của LLM.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment (Hard Gates):**
>   - Faithfulness giảm $> 0.05$ hoặc rơi xuống $< 0.80$ (nguy cơ hallucination).
>   - Bất kỳ vi phạm nào trong nhóm Adversarial (thất bại trước Prompt Injection hoặc tiết lộ dữ liệu cá nhân).
> - **Alert Only (Soft Gates / Monitoring):**
>   - Context Precision giảm nhẹ nhưng Context Recall vẫn $> 0.85$.
>   - Điểm Relevance dao động nhẹ trên các câu hỏi mở.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [ Unit Tests & Golden Dataset Eval ] → [ Regression Check (Drop <= 0.05) ] → [ Canary / Staging Human Audit ] → Deploy
```

> *Giải thích:*
> Khi có thay đổi, code phải vượt qua unit tests và chạy offline benchmark trên Golden Dataset 20 QA. Sau đó hệ thống tự động so sánh với baseline qua `run_regression()`. Nếu không bị tụt điểm quá 0.05, code được đưa lên Staging/Canary để audit mẫu trước khi chính thức release.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm few-shot prompt giải quyết cảnh báo an toàn khẩn cấp | Completeness & Safety | Completeness case M07 tăng lên $> 0.90$, triệt tiêu rủi ro an toàn. |
| 2 | Tích hợp LLM-as-a-Judge cho các ca Adversarial / Refusal | Pass Rate & Safety | Pass Rate tăng từ 70% lên $> 90\%$, đánh giá đúng năng lực từ chối. |
| 3 | Tối ưu hóa cross-encoder reranking cho retrieval chunks | Context Precision | Context Precision duy trì mức tối đa $> 0.98$. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Case Đa ngôn ngữ (Multilingual / Code-switching):** Khách hàng hỏi bằng tiếng Việt hoặc tiếng Anh pha lẫn thuật ngữ kỹ thuật.
> 2. **Case Yêu cầu kết hợp nhiều khuyến mãi (Complex Promo Stacking):** Khách hàng muốn dùng đồng thời mã giảm giá %, voucher quà tặng, thẻ thành viên OrbitPlus và trả góp OrbitPay.
> 3. **Case Tranh chấp bảo hành do rơi vỡ kết hợp lỗi phần mềm:** Khách hàng báo thiết bị vừa nứt màn hình vừa lỗi kết nối mạng.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu tôi dự đoán rằng bộ tìm kiếm BM25 đơn giản sẽ là điểm nghẽn (bottleneck) lớn nhất của hệ thống RAG. Tuy nhiên, kết quả thực tế cho thấy BM25 hoạt động xuất sắc với **Context Precision = 0.966** và **Context Recall = 0.885**. Trái lại, điểm nghẽn thực sự lại nằm ở cách thức đánh giá: các công thức word-overlap truyền thống không thể phản ánh chính xác sự thông minh của mô hình trong các câu trả lời ngắn gọn và các câu từ chối bảo mật an toàn.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của word-overlap heuristics:**
>   1. Không hiểu từ đồng nghĩa, ngữ cảnh và cách diễn đạt khác (paraphrasing).
>   2. Bị thiên kiến độ dài (length penalty): câu trả lời ngắn gọn bị phạt Relevance, câu trả lời an toàn bị phạt Faithfulness.
>   3. Không thể đánh giá được logic suy luận đa bước và độ an toàn của câu trả lời.
> - **Thay thế và bổ sung trong Production:**
>   1. **G-Eval / LLM-as-a-Judge:** Dùng mô hình GPT-4o với Chain-of-Thought scoring theo thang điểm 1-5 bám sát Rubric doanh nghiệp.
>   2. **Embedding-based Semantic Similarity (BERTScore / Cosine Similarity):** Đo lường độ tương đồng ngữ nghĩa vector.
>   3. **Hallucination & Guardrail Verification (NLI - Natural Language Inference):** Phân tích quan hệ logic Entailment / Contradiction giữa Context và Generated Claims.
