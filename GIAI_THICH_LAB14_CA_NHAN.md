# GIẢI THÍCH TOÀN BỘ LAB 14 — AI EVALUATION & BENCHMARKING PIPELINE
> **Dành cho sinh viên thực hành CÁ NHÂN (1 người)**  
> *Chủ đề: AI Evaluation, Benchmarking Pipeline, RAGAS Metrics, LLM-as-a-Judge, Golden Dataset Design, Failure Taxonomy, 5 Whys RCA, CI/CD Quality Gate.*

---

## 💡 PHẦN 1: Ý NGHĨA CỐT LÕI CỦA BÀI LAB (WHY AI EVALUATION?)

### 1. Tại sao hệ thống AI lại cần Evaluation khác hoàn toàn với Software Testing truyền thống?

Trong các hệ thống phần mềm truyền thống (Web App, Database, API Service):
- **Tính xác định (Deterministic)**: Với cùng một input $X$, hệ thống **luôn luôn** trả về đúng output $Y$.
- **Unit Test truyền thống**: `Assert add(2, 3) == 5`. Kiểm thử mang tính Binary (Đúng / Sai tuyệt đối, 100% hoặc 0%).

Tuy nhiên, đối với một **Hệ thống AI (LLM / RAG Assistant / Agent)**:
1. **Tính không xác định (Non-deterministic & Generative)**: Cùng một câu hỏi $X$, LLM có thể sinh ra câu trả lời $Y_1, Y_2, Y_3$ với văn phong khác nhau mỗi lần chạy.
2. **Sự nguy hiểm của "Vibe Check"**: Nhiều kỹ sư AI có thói quen thử vài câu hỏi trên giao diện chat (nhìn thấy "vibe" có vẻ hay) rồi đưa ngay ra Production. Khi gặp hàng ngàn người dùng thật với vô số câu hỏi lắt léo, hệ thống lập tức sụp đổ vì rò rỉ thông tin sai lệch (Hallucination), trả lời lan man hoặc từ chối vô lý.
3. **Thách thức khi nâng cấp hệ thống (Regression)**: Khi bạn đổi Prompt, nâng cấp Vector DB từ BM25 sang Hybrid Search, hay đổi Model từ GPT-3.5 sang GPT-4o, làm sao khẳng định được hệ thống **tốt hơn tổng thể** chứ không phải "được câu này lại hỏng câu khác"?

👉 **Bài Lab 14 sinh ra để giúp bạn áp dụng Phương pháp Khoa học (Scientific Method) cho AI: `Hypothesis → Experiment → Measure → Conclude → Iterate`. Xây dựng một đường ống đánh giá (Evaluation Pipeline) tự động, có thể lặp lại (repeatable), so sánh được (comparable) và sẵn sàng tích hợp vào CI/CD.**

---

### 2. Ba loại Evaluation trong vòng đời sản phẩm AI

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                 AI EVALUATION                                     │
├──────────────────────────┬──────────────────────────┬─────────────────────────────┤
│      1. OFFLINE EVAL     │       2. ONLINE EVAL     │       3. HUMAN EVAL         │
│   (Golden Dataset & Core)│   (Real Traffic Monitoring)│   (Gold Standard Calibration│
├──────────────────────────┼──────────────────────────┼─────────────────────────────┤
│ • Chạy trước khi Release │ • Đo trực tiếp trên      │ • Chuyên gia gán nhãn thủ   │
│   hoặc Merge PR.         │   traffic người dùng thật│   công (Human Annotation).  │
│ • Dùng Golden Dataset    │ • Dùng TruLens, Langfuse │ • Dùng để căn chỉnh (Calib) │
│   20-1000 QA pairs.      │   feedback functions.    │   cho LLM Judge & Automated │
│ • Chạy bằng RAGAS/DeepEval│ • Phát hiện Data Drift  │   Metrics.                  │
└──────────────────────────┴──────────────────────────┴─────────────────────────────┘
```

Trong bài lab này, chúng ta tập trung hoàn thiện **Offline Evaluation Engine** — trái tim của quy trình kiểm thử chất lượng trước khi phát hành.

---

### 3. Bộ Metrics đánh giá RAG: RAG Triad & Retrieval Metrics

Để đánh giá một hệ thống Retrieval-Augmented Generation (RAG), bài lab chia quá trình sinh câu trả lời thành 2 phía (Answer-side & Retrieval-side) tương ứng với 5 metrics cốt lõi:

```text
Question ───────► Retriever ───────► Context ───────► Generator ───────► Answer
                    │                  │                 │                 │
                    ▼                  ▼                 ▼                 ▼
             Context Recall     Context Precision    Faithfulness    Answer Relevance
```

#### A. Answer-Side Metrics (Đánh giá chất lượng Generator / Answer):
1. **Faithfulness (Độ trung thực)**: Câu trả lời (`actual_answer`) có dựa hoàn toàn trên tri thức thu thập được (`context`) hay không?
   - *Heuristic trong Lab*: $\text{Faithfulness} = \frac{|\text{tokens(answer)} \cap \text{tokens(context)}|}{|\text{tokens(answer)}|}$
   - *Ý nghĩa*: Đo lường rủi ro **Hallucination** (Bốc phét/Bịp bợm). Nếu câu trả lời chứa nhiều từ không nằm trong Context $\rightarrow$ Faithfulness thấp.
2. **Relevance (Độ liên quan)**: Câu trả lời (`actual_answer`) có đi thẳng vào trọng tâm câu hỏi (`question`) hay không?
   - *Heuristic trong Lab*: $\text{Relevance} = \frac{|\text{tokens(answer)} \cap \text{tokens(question)}|}{|\text{tokens(question)}|}$
   - *Ý nghĩa*: Đo lường rủi ro **Off-topic / Irrelevant** (Trả lời lạc đề hoặc trả lời vòng vo không đúng ý user).
3. **Completeness (Độ đầy đủ)**: Câu trả lời (`actual_answer`) đã bao phủ đủ các ý quan trọng trong câu trả lời chuẩn (`expected_answer`) chưa?
   - *Heuristic trong Lab*: $\text{Completeness} = \frac{|\text{tokens(answer)} \cap \text{tokens(expected)}|}{|\text{tokens(expected)}|}$
   - *Ý nghĩa*: Đo lường rủi ro **Incomplete** (Trả lời thiếu ý, thiếu bước thực hiện quy trình).

#### B. Retrieval-Side Metrics (Đánh giá chất lượng Retriever / Search):
4. **Context Recall (Độ gợi nhớ)**: Retriever có lấy ra đủ các bằng chứng cần thiết (`expected_answer` / Gold evidence) từ cơ sở tri thức không?
   - *Heuristic trong Lab*: $\text{Context Recall} = \frac{|\text{tokens(expected)} \cap \text{tokens(union(retrieved\_chunks))}|}{|\text{tokens(expected)}|}$
   - *Ý nghĩa*: Nếu Context Recall thấp $\rightarrow$ Retriever bỏ sót thông tin $\rightarrow$ LLM không đủ thông tin để trả lời.
5. **Context Precision (Độ chính xác theo thứ tự - AP@K)**: Các chunk đúng (relevant) có đứng ở vị trí ưu tiên đầu danh sách kết quả tìm kiếm không?
   - *Heuristic trong Lab*: Compute Average Precision (AP@K) trên danh sách `retrieved_contexts`.
   - *Ý nghĩa*: Nếu chunk chứa đáp án bị đẩy xuống vị trí số 5 hay 10 $\rightarrow$ LLM bị nhiễu do dư thừa thông tin không liên quan (Lost in the Middle).

> [!NOTE]
> Điểm tổng thể `overall_score()` được tính bằng trung bình cộng 3 Answer Metrics:
> $$\text{Overall Score} = \frac{\text{Faithfulness} + \text{Relevance} + \text{Completeness}}{3}$$
> Hai Retrieval Metrics dùng riêng để chẩn đoán hệ thống tìm kiếm và không đưa vào `overall_score()`.

---

### 4. Tư duy LLM-as-a-Judge & Nhận diện Bias

Trong môi trường thực tế, so sánh chuỗi từ ngữ (word overlap) là chưa đủ vì ngôn ngữ tự nhiên rất phong phú (đồng nghĩa, trái nghĩa, diễn đạt khác). Ta dùng một LLM mạnh (như GPT-4o) làm **Giám khảo (Judge)**.

#### A. Rubric Thang điểm 1–5
Giám khảo LLM cần được cung cấp một **Rubric chấm điểm** vô cùng chi tiết:
- **5 (Excellent)**: Trả lời hoàn hảo, chính xác 100%, đầy đủ chi tiết, dẫn nguồn rõ ràng.
- **4 (Good)**: Chính xác, đầy đủ nhưng văn phong chưa thật sự tối ưu.
- **3 (Fair)**: Trả lời được ý chính nhưng thiếu một số chi tiết nhỏ hoặc dư thừa nhẹ.
- **2 (Poor)**: Thiếu nhiều thông tin quan trọng, hoặc chứa thông tin sai lệch nhẹ.
- **1 (Unacceptable)**: Trả lời sai hoàn toàn, Hallucination nghiêm trọng, từ chối vô lý hoặc vi phạm an toàn.

#### B. Nhận diện và Khắc phục Bias của LLM Judge
LLM không phải giám khảo hoàn hảo. Chúng mắc 3 thiên vị (bias) phổ biến:
1. **Position Bias (Thiên vị vị trí)**: LLM Judge có xu hướng chấm điểm cao hơn cho câu trả lời xuất hiện đầu tiên trong prompt so sánh.
   - *Khắc phục*: Randomize order (hoán đổi vị trí mẫu A và B khi gọi Judge).
2. **Verbosity / Leniency Bias (Thiên vị độ dài / Nương tay)**: LLM Judge thích câu trả lời dài dòng, hoa mỹ hơn câu trả lời ngắn gọn súc tích dù nội dung như nhau.
   - *Khắc phục*: Đưa quy định ép buộc vào Prompt Rubric: *"Không cộng điểm cho sự dài dòng vô ích"*.
3. **Self-Preference / Severity Bias (Thiên vị bản thân / Khắt khe)**: LLM Judge thích các câu trả lời sinh ra từ chính model gia đình nó (ví dụ GPT-4 thích văn phong của GPT-3.5/GPT-4 hơn Claude).

---

### 5. Thiết kế Golden Dataset theo Stratified Sampling (Mẫu phân tầng)

Một bộ dữ liệu chuẩn **Golden Dataset 20 câu** không thể chỉ toàn câu dễ. Nó phải đại diện cho toàn bộ các tình huống thực tế của người dùng:

| Tầng độ khó | Số lượng | Đặc điểm & Mục đích |
|---|---:|---|
| **Easy** | 5 câu | Tra cứu thực tế 1 document duy nhất (Factual Lookup). Ví dụ: Đơn vị tính điểm rèn luyện, Địa chỉ phòng đào tạo. |
| **Medium** | 7 câu | Tổng hợp quy trình từ 2–3 documents (Multi-doc reasoning). Ví dụ: Điều kiện & thủ tục xin học lại sau khi bị tạm dừng. |
| **Hard** | 5 câu | Nhiều điều kiện ràng buộc, ngoại lệ, hiệu lực ngày tháng (Complex logic). Ví dụ: Tính học phí chuẩn cho sinh viên chuyển ngành có bảo lưu tín chỉ. |
| **Adversarial** | 3 câu | Câu hỏi bẫy, giả định sai (False premise), nằm ngoài phạm vi (Out-of-scope), hoặc cố tình Prompt Injection. Ví dụ: "Thủ tục xin miễn thi cho con nuôi giáo sư?" |

---

### 6. Phân loại Lỗi (Failure Taxonomy) & Kỹ thuật 5 Whys RCA

Khi câu trả lời có điểm $\text{Overall Score} < 0.5$, hệ thống tự động đánh dấu là **FAILED** và gắn nhãn phân loại lỗi:

| Mã loại lỗi | Triệu chứng quan sát được | Nguyên nhân gốc rễ (Root Cause) thường gặp |
|---|---|---|
| `hallucination` | Answer chứa thông tin không có trong context | Prompt không ép quy tắc "chỉ dùng thông tin được cung cấp", guardrail yếu. |
| `irrelevant` | Answer không giải quyết đúng câu hỏi của user | Intent Routing sai hoặc Prompt không hiểu trọng tâm câu hỏi. |
| `incomplete` | Answer bỏ sót ý quan trọng trong expected answer | Chunk size quá nhỏ làm đứt đoạn thông tin, hoặc LLM stop generation sớm. |
| `off_topic` | Trả lời sang một chủ đề hoàn toàn khác | Retriever lấy nhầm document (Context Recall = 0), hoặc BM25 bị bẫy từ khóa. |
| `refusal` | Từ chối trả lời ("Tôi không biết") khi dữ liệu có sẵn | Safety system / Guardrail bị quá nhạy (Over-defensive). |

#### Phương pháp Điều tra Root Cause: 5 Whys (5 Câu hỏi Tại sao)
```text
Symptom: Câu trả lời bị thiếu quy trình miễn giảm học phí (Incomplete).
 ├── Why 1: Tại sao thiếu? ──► LLM không đề cập đến đối tượng sinh viên vùng khó khăn.
 ├── Why 2: Tại sao LLM không đề cập? ──► Trong context được cung cấp không có đoạn văn đó.
 ├── Why 3: Tại sao context không có? ──► Retriever chỉ lấy top 2 chunks, mà thông tin này nằm ở chunk 4.
 ├── Why 4: Tại sao chỉ lấy top 2? ──► Parameter top_k của Retriever bị hardcode = 2.
 └── Why 5 (Root Cause): ──► Chưa cấu hình top_k động dựa trên độ phức tạp của câu hỏi.
     └── Actionable Fix: Tăng top_k = 5 và bổ sung Reranker (Cross-Encoder).
```

---

### 7. Evaluation như một Quality Gate trong CI/CD

```text
[ Developer Push Code / Sửa Prompt ] 
                │
                ▼
[ GitHub Actions / CI Pipeline ]
                │
                ├── 1. Run Unit Tests (pytest tests/)
                ├── 2. Run Evaluation Pipeline on Golden Dataset
                │
                ▼
      Check Quality Gate Rules:
      • Pass Rate >= 85% ?
      • Metric Regression <= 0.05 ?
                │
        ┌───────┴───────┐
        │               │
     [ PASS ]        [ FAIL ]
        │               │
        ▼               ▼
[ Merge / Deploy ]  [ Block Merge / Alert SRE ]
```

---

## 🏗️ PHẦN 2: KIẾN TRÚC VÀ VAI TRÒ CỦA TỪNG FILE (SYSTEM ARCHITECTURE & FILE TAXONOMY)

### 1. Phân định 3 Thành phần độc lập

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                SYSTEM ARCHITECTURE                                │
├──────────────────────────┬──────────────────────────┬─────────────────────────────┤
│ 1. SYSTEM UNDER EVAL     │ 2. EVALUATION CORE       │ 3. ARTIFACT ADAPTER         │
│    (domain_assistant.py) │    (template.py)         │    (evaluate_answers.py)    │
├──────────────────────────┼──────────────────────────┼─────────────────────────────┤
│ • RAG Assistant thật cho │ • Động cơ đánh giá gồm   │ • Đọc dữ liệu từ file       │
│   Northstar University.  │   QAPair, EvalResult,    │   golden_dataset.json và    │
│ • Nhận question ──► BM25 │   RAGASEvaluator,        │   actual_answers.json.      │
│   retrieval ──► OpenAI   │   LLMJudge, Runner,      │ • Nối hai phần lại và xuất  │
│   ──► actual_answer.     │   FailureAnalyzer.       │   benchmark_results.json.   │
└──────────────────────────┴──────────────────────────┴─────────────────────────────┘
```

---

### 2. Ý nghĩa & Mục đích của từng File trong Repository

| File / Folder | Loại file | Vai trò và Ý nghĩa trong hệ thống |
|---|---|---|
| [template.py](template.py) | **Python Core** | **Trái tim của Evaluation Engine**: Chứa 5 Task bài tập (Data models, RAGAS metrics, LLM-as-a-Judge, Benchmark Runner, Failure Analyzer). Đây là nơi học viên hoàn thiện code. |
| [domain_assistant.py](domain_assistant.py) | **Python System** | **Hệ thống AI được đánh giá (System Under Evaluation)**: Đóng vai trò RAG Student Services Assistant, thực hiện BM25 search trên `data/student_services/` và gọi OpenAI GPT để trả lời câu hỏi. |
| [evaluate_answers.py](evaluate_answers.py) | **Python Adapter** | **Cầu nối dữ liệu**: Đọc `golden_dataset.json` và `artifacts/actual_answers.json`, nạp vào `template.py` để chấm điểm và xuất báo cáo `artifacts/benchmark_results.json`. |
| [validate_golden_dataset.py](validate_golden_dataset.py) | **Python Validator** | **Công cụ kiểm tra Dataset**: Chạy kiểm tra tự động xem `golden_dataset.json` có đủ 20 câu, đủ 4 tầng độ khó (5-7-5-3), có vi phạm schema hay thiếu dẫn chứng nguồn không. |
| [golden_dataset.json](golden_dataset.json) | **Data File** | **Bộ câu hỏi & đáp án chuẩn (Gold Standard)**: Bộ dữ liệu 20 câu do chuyên gia/học viên biên soạn làm thước đo chuẩn cho hệ thống. |
| [reflection.md](reflection.md) | **Markdown Report** | **Báo cáo phân tích lỗi & 5 Whys**: Nơi ghi lại kết quả benchmark thực tế, phân tích Top 3 câu trả lời tệ nhất và tìm nguyên nhân gốc rễ (Root Cause). |
| [exercises.md](exercises.md) | **Markdown Worksheet** | **Phiếu bài tập**: Chứa câu hỏi lý thuyết Warm-up (Part 1) và mô tả yêu cầu thực hành cho từng Task (Part 2 & Part 3). |
| [requirements.txt](requirements.txt) | **Config** | Danh sách thư viện phụ thuộc (`openai`, `python-dotenv`, `pytest`). |
| `.env` / `.env.example` | **Config** | File lưu biến môi trường (chứa `OPENAI_API_KEY`). |
| [solution/solution.py](solution/solution.py) | **Python Output** | Bản sao hoàn chỉnh của `template.py` sau khi đã vượt qua 42 unit tests để dùng cho chấm điểm tự động. |
| [tests/test_solution.py](tests/test_solution.py) | **Test Suite** | Bộ 42 Unit Tests để kiểm tra tính đúng đắn của từng hàm/class trong `template.py`. |

---

### 3. Quy tắc Vàng: Ngăn chặn Data Leakage (Rò rỉ dữ liệu)

```text
       golden_dataset.json
        ├── id, question ───────────────────► [ domain_assistant.py ]
        │                                             │
        │                                             ▼
        │                                    actual_answers.json
        │                                             │
        └── expected_answer, context ─────────────────┴──────────► [ evaluate_answers.py ]
                                                                          │
                                                                          ▼
                                                                 [ template.py Core ]
```

> [!CRITICAL]
> `domain_assistant.py` (System Under Evaluation) **CHỈ ĐƯỢC PHÉP** đọc `id` và `question`. Nó **TUYỆT ĐỐI KHÔNG ĐƯỢC** truy cập vào `expected_answer` hay `gold context` khi sinh câu trả lời. Nếu rò rỉ, bài benchmark sẽ mất toàn bộ giá trị khoa học!

---

## 🗺️ PHẦN 3: CHIẾN LƯỢC THỰC HÀNH CÁ NHÂN & CHI TIẾT TASK (SOLO ROADMAP)

Học viên làm 1 mình sẽ đi qua **4 Giai đoạn** tuần tự để hoàn thành 100% bài lab:

```text
Giai đoạn 1: Môi trường & Baseline ──► Giai đoạn 2: Lập trình Evaluation Core ──► Giai đoạn 3: Xây dựng Golden Dataset ──► Giai đoạn 4: Benchmark thật & Phân tích Lỗi
```

---

### 📍 Giai đoạn 1: Khởi tạo Môi trường & Kiểm tra Baseline (20 phút)

1. **Cài đặt Virtual Environment & Dependencies**:
   ```bash
   python3 -m venv .venv  # Hoặc python3.12 -m venv .venv
   source .venv/bin/activate  # On Linux/macOS
   pip install --upgrade pip
   pip install -r requirements.txt
   cp .env.example .env  # Điền OPENAI_API_KEY
   ```
2. **Chạy thử Baseline Test Suite**:
   ```bash
   .venv/bin/pytest tests/ -v
   ```
   *Kết quả mong đợi*: Terminal báo **42 tests collected, 42 failed**. Đây là điểm bắt đầu bình thường do chưa triển khai code trong [template.py](template.py).

---

### 📍 Giai đoạn 2: Lập trình Evaluation Core ([template.py](template.py))

---

#### 🟢 TASK 1: DATA MODELS (`QAPair`, `EvalResult`, `overall_score`) — [ĐÃ HOÀN THÀNH CODE]

##### 1. Đoạn Code Đã Triển Khai Trong `template.py`:
```python
@dataclass
class QAPair:
    """
    A question-answer pair for evaluation (part of the Golden Dataset).
    """
    question: str
    expected_answer: str
    context: str | None = ""
    metadata: dict = field(default_factory=dict)
    retrieved_contexts: list[str] = field(default_factory=list)


@dataclass
class EvalResult:
    """
    Evaluation result for a single Q&A pair.
    """
    qa_pair: QAPair
    actual_answer: str
    faithfulness: float
    relevance: float
    completeness: float
    passed: bool
    failure_type: str | None = None
    context_precision: float | None = None
    context_recall: float | None = None

    def overall_score(self) -> float:
        """Compute the average of faithfulness, relevance, and completeness."""
        return (self.faithfulness + self.relevance + self.completeness) / 3.0
```

##### 2. Giải thích Chi tiết Luồng Code & Ý nghĩa từng Thuộc tính:

- **Dataclass `QAPair` (Dữ liệu Câu hỏi & Đáp án Mẫu)**:
  - `question`: Câu hỏi thực tế của người dùng (ví dụ: *"Điều kiện đăng ký thi lại là gì?"*).
  - `expected_answer`: Câu trả lời chuẩn xác do chuyên gia soạn (Ground Truth / Reference Answer).
  - `context`: Văn bản nguồn trích xuất từ tài liệu (Gold Evidence), mặc định là chuỗi rỗng `""`.
  - `metadata`: Chứa các thông tin bổ sung dạng dict như độ khó (`difficulty: "medium"`), danh mục (`category: "academic"`). Dùng `field(default_factory=dict)` để tránh lỗi đụng độ danh sách động trong Python dataclass.
  - `retrieved_contexts`: Danh sách các đoạn văn bản (chunks) mà hệ thống Retriever thực tế đã lấy lên theo đúng thứ tự xếp hạng (dùng cho các metric phía Retrieval ở Task 2).

- **Dataclass `EvalResult` (Kết quả Chấm điểm 1 Cặp QA)**:
  - `qa_pair`: Liên kết ngược lại đối tượng `QAPair` ban đầu.
  - `actual_answer`: Câu trả lời thực tế mà AI Assistant sinh ra (`domain_assistant.py`).
  - `faithfulness`, `relevance`, `completeness`: Ba điểm số cơ bản (giá trị float từ `0.0` đến `1.0`).
  - `passed`: Biến kiểu `bool`, bằng `True` nếu cả 3 điểm số $\ge 0.5$, ngược lại là `False`.
  - `failure_type`: Nếu `passed == False`, trường này nhận 1 trong 5 loại lỗi (`"hallucination"`, `"irrelevant"`, `"incomplete"`, `"off_topic"`, `"refusal"`).
  - `context_precision`, `context_recall`: Hai điểm số đánh giá phía Retriever (dạng `float | None`, mặc định `None` nếu không cung cấp `retrieved_contexts`).

- **Phương thức `overall_score()` (Tính điểm Tổng thể)**:
  - Công thức: `(self.faithfulness + self.relevance + self.completeness) / 3.0`
  - Trả về trung bình cộng đơn giản của 3 Answer Metrics để làm chỉ số tổng quát đại diện cho chất lượng câu trả lời.

##### 3. Kết quả Kiểm thử Unit Test cho Task 1:
```bash
.venv/bin/pytest tests/test_solution.py -k TestEvalResultOverallScore -v
# Output: 3 passed, 39 deselected in 0.02s! (XANH 100%)
```

---

#### 🟢 TASK 2: RAGASEVALUATOR (5 METRICS & RERANKER) — [ĐÃ HOÀN THÀNH CODE]

##### 1. Đoạn Code Đã Triển Khai Trong `template.py`:
```python
    def evaluate_faithfulness(self, answer: str, context: str) -> float:
        """Measure how grounded the answer is in the context."""
        answer_tokens = _tokenize(answer)
        if not answer_tokens:
            return 1.0
        context_tokens = _tokenize(context)
        score = len(answer_tokens & context_tokens) / len(answer_tokens)
        return max(0.0, min(1.0, float(score)))

    def evaluate_relevance(self, answer: str, question: str) -> float:
        """Measure how relevant the answer is to the question."""
        question_tokens = _tokenize(question)
        if not question_tokens:
            return 1.0
        answer_tokens = _tokenize(answer)
        score = len(answer_tokens & question_tokens) / len(question_tokens)
        return max(0.0, min(1.0, float(score)))

    def evaluate_completeness(self, answer: str, expected: str) -> float:
        """Measure how well the answer covers the expected answer."""
        expected_tokens = _tokenize(expected)
        if not expected_tokens:
            return 1.0
        answer_tokens = _tokenize(answer)
        score = len(answer_tokens & expected_tokens) / len(expected_tokens)
        return max(0.0, min(1.0, float(score)))

    def evaluate_context_recall(self, contexts: list[str], expected: str) -> float:
        """Context Recall — how much of the expected answer is covered by UNION of retrieved chunks."""
        expected_tokens = _tokenize(expected)
        if not expected_tokens:
            return 1.0
        union_tokens: set[str] = set()
        for chunk in contexts:
            union_tokens.update(_tokenize(chunk))
        score = len(expected_tokens & union_tokens) / len(expected_tokens)
        return max(0.0, min(1.0, float(score)))

    def evaluate_context_precision(
        self,
        contexts: list[str],
        expected: str,
        relevance_threshold: float = 0.1,
    ) -> float:
        """Context Precision — RANK-AWARE Average Precision (AP@K)."""
        expected_tokens = _tokenize(expected)
        if not expected_tokens:
            return 1.0
        if not contexts:
            return 0.0

        relevant_flags = []
        for chunk in contexts:
            chunk_tokens = _tokenize(chunk)
            overlap_ratio = len(chunk_tokens & expected_tokens) / len(expected_tokens)
            relevant_flags.append(overlap_ratio >= relevance_threshold)

        num_relevant = sum(1 for r in relevant_flags if r)
        if num_relevant == 0:
            return 0.0

        ap_sum = 0.0
        relevant_count = 0
        for k, is_rel in enumerate(relevant_flags, start=1):
            if is_rel:
                relevant_count += 1
                precision_at_k = relevant_count / k
                ap_sum += precision_at_k

        return max(0.0, min(1.0, float(ap_sum / num_relevant)))

    def run_full_eval(
        self,
        answer: str,
        question: str,
        context: str,
        expected: str,
        contexts: list[str] | None = None,
    ) -> EvalResult:
        """Run answer-side & optional retrieval-side evaluations."""
        faithfulness = self.evaluate_faithfulness(answer, context)
        relevance = self.evaluate_relevance(answer, question)
        completeness = self.evaluate_completeness(answer, expected)

        passed = bool(faithfulness >= 0.5 and relevance >= 0.5 and completeness >= 0.5)

        failure_type: str | None = None
        if not passed:
            if faithfulness < 0.3:
                failure_type = "hallucination"
            elif relevance < 0.3:
                failure_type = "irrelevant"
            elif completeness < 0.3:
                failure_type = "incomplete"
            else:
                failure_type = "off_topic"

        context_recall: float | None = None
        context_precision: float | None = None
        if contexts is not None:
            context_recall = self.evaluate_context_recall(contexts, expected)
            context_precision = self.evaluate_context_precision(contexts, expected, relevance_threshold=0.1)

        qa_pair = QAPair(
            question=question,
            expected_answer=expected,
            context=context,
            retrieved_contexts=contexts if contexts is not None else [],
        )

        return EvalResult(
            qa_pair=qa_pair,
            actual_answer=answer,
            faithfulness=faithfulness,
            relevance=relevance,
            completeness=completeness,
            passed=passed,
            failure_type=failure_type,
            context_precision=context_precision,
            context_recall=context_recall,
        )


def rerank_by_overlap(contexts: list[str], query: str) -> list[str]:
    """Lexical reranker sorting chunks by word overlap with query."""
    query_tokens = _tokenize(query)
    return sorted(contexts, key=lambda c: len(_tokenize(c) & query_tokens), reverse=True)
```

##### 2. Giải thích Chi tiết Luồng Code & Công thức Toán học:

1. **`evaluate_faithfulness` (Độ trung thực)**:
   - *Luồng xử lý*: Tách từ trong `answer` và `context` bằng `_tokenize` (đã bỏ punctuation & stopwords). Nếu `answer` rỗng $\rightarrow$ trả về `1.0`. Tính tỷ lệ số từ trong `answer` có xuất hiện trong `context` chia cho tổng số từ của `answer`.
   - *Công thức*: $\text{Faithfulness} = \frac{|\text{tokens(answer)} \cap \text{tokens(context)}|}{|\text{tokens(answer)}|}$

2. **`evaluate_relevance` (Độ liên quan)**:
   - *Luồng xử lý*: Tách từ trong `answer` và `question`. Nếu `question` rỗng $\rightarrow$ trả về `1.0`. Tính tỷ lệ từ giao nhau giữa `answer` và `question` trên tổng số từ của `question`.
   - *Công thức*: $\text{Relevance} = \frac{|\text{tokens(answer)} \cap \text{tokens(question)}|}{|\text{tokens(question)}|}$

3. **`evaluate_completeness` (Độ đầy đủ)**:
   - *Luồng xử lý*: So sánh `answer` với `expected_answer` (Ground Truth). Tính tỷ lệ bao phủ của `answer` trên các từ của `expected_answer`.
   - *Công thức*: $\text{Completeness} = \frac{|\text{tokens(answer)} \cap \text{tokens(expected)}|}{|\text{tokens(expected)}|}$

4. **`evaluate_context_recall` (Độ gợi nhớ từ các Chunks)**:
   - *Luồng xử lý*: Lấy **hợp (Union)** tất cả tập từ của các `chunks` trong `contexts`. So sánh tập từ của `expected_answer` với tập hợp từ này. Trả về tỷ lệ thông tin đáp án chuẩn được bao phủ bởi Retriever.
   - *Công thức*: $\text{Context Recall} = \frac{|\text{tokens(expected)} \cap \bigcup \text{tokens(chunk)}|}{|\text{tokens(expected)}|}$

5. **`evaluate_context_precision` (Độ chính xác xếp hạng AP@K)**:
   - *Luồng xử lý*:
     - Kiểm tra từng chunk $k$ xem tỷ lệ bao phủ so với `expected` có $\ge \text{threshold} (0.1)$ không để gắn cờ `is_relevant`.
     - Với mỗi chunk có liên quan ở vị trí $k$, tính $\text{Precision}@k = \frac{\text{số chunk đúng trong top } k}{k}$.
     - Tính trung bình $\text{AP}@K = \frac{1}{\text{tổng số chunk đúng}} \sum_k \text{Precision}@k \cdot \text{is\_relevant}_k$.
   - *Ý nghĩa*: Thưởng điểm cao hơn khi chunk đúng nằm ở ngay vị trí đầu tiên ($k=1$).

6. **`run_full_eval` (Hàm Chấm điểm Tổng hợp)**:
   - Chạy 3 metric phía Answer. Đánh giá `passed = True` nếu cả 3 metric $\ge 0.5$.
   - Nếu `passed == False`, phân loại lỗi theo thứ tự ưu tiên:
     - `faithfulness < 0.3` $\rightarrow$ `"hallucination"`
     - `relevance < 0.3` $\rightarrow$ `"irrelevant"`
     - `completeness < 0.3` $\rightarrow$ `"incomplete"`
     - Ngược lại $\rightarrow$ `"off_topic"`
   - Nếu có truyền `contexts` (danh sách chunks), tính thêm `context_recall` và `context_precision`.

7. **`rerank_by_overlap` (Hàm Reranker Từ vựng)**:
   - Sắp xếp lại danh sách `contexts` dựa trên số lượng từ giao nhau với `query` từ cao xuống thấp. Giúp đẩy chunk liên quan lên vị trí đầu $\rightarrow$ làm tăng `context_precision`.

##### 3. Kết quả Kiểm thử Unit Test cho Task 2:
```bash
.venv/bin/pytest tests/test_solution.py -k "TestRAGASEvaluator or TestContextMetrics or TestRetrievalMetricWiring" -v
# Output: 15 passed (XANH 100% cho toàn bộ 5 metrics + Reranker + run_full_eval)!
```

---

#### 🟢 TASK 3: LLM JUDGE & BIAS DETECTION — [ĐÃ HOÀN THÀNH CODE]

##### 1. Đoạn Code Đã Triển Khai Trong `template.py`:
```python
class LLMJudge:
    """
    Uses an LLM to score AI responses according to a rubric.
    """

    def __init__(self, judge_llm_fn: Callable[[str], str]) -> None:
        self.judge_llm_fn = judge_llm_fn

    def score_response(
        self,
        question: str,
        answer: str,
        rubric: dict[str, Any],
    ) -> dict[str, Any]:
        """Score an AI response using the judge LLM."""
        prompt = (
            f"Question: {question}\n"
            f"Answer: {answer}\n"
            f"Rubric: {rubric}\n"
            "Evaluate the answer based on rubric. Return JSON mapping each criterion to a float score (0.0 to 1.0)."
        )
        raw_response = self.judge_llm_fn(prompt)
        scores: dict[str, float] = {}

        try:
            parsed = json.loads(raw_response)
            if isinstance(parsed, dict):
                for k in rubric.keys():
                    if k in parsed and isinstance(parsed[k], (int, float)):
                        scores[k] = float(parsed[k])
                    else:
                        scores[k] = 0.5
            else:
                scores = {k: 0.5 for k in rubric.keys()}
        except Exception:
            match = re.search(r"\{.*\}", raw_response, re.DOTALL)
            if match:
                try:
                    parsed = json.loads(match.group(0))
                    if isinstance(parsed, dict):
                        for k in rubric.keys():
                            scores[k] = float(parsed.get(k, 0.5))
                    else:
                        scores = {k: 0.5 for k in rubric.keys()}
                except Exception:
                    scores = {k: 0.5 for k in rubric.keys()}
            else:
                scores = {k: 0.5 for k in rubric.keys()}

        return {
            "scores": scores,
            "reasoning": raw_response,
        }

    def detect_bias(self, scores_batch: list[dict[str, Any]]) -> dict[str, Any]:
        """Detect potential bias patterns in a batch of judge scores."""
        all_scores: list[float] = []
        for item in scores_batch:
            s_dict = item.get("scores", {})
            if isinstance(s_dict, dict):
                all_scores.extend(s_dict.values())

        if all_scores:
            avg_score = sum(all_scores) / len(all_scores)
            leniency_bias = bool(avg_score > 0.8)
            severity_bias = bool(avg_score < 0.3)
        else:
            leniency_bias = False
            severity_bias = False

        positional_bias = False
        if len(scores_batch) >= 2:
            first_scores = list(scores_batch[0].get("scores", {}).values())
            last_scores = list(scores_batch[-1].get("scores", {}).values())
            if first_scores and last_scores:
                avg_first = sum(first_scores) / len(first_scores)
                avg_last = sum(last_scores) / len(last_scores)
                if avg_first - avg_last > 0.2:
                    positional_bias = True

        return {
            "positional_bias": positional_bias,
            "leniency_bias": leniency_bias,
            "severity_bias": severity_bias,
        }
```

##### 2. Giải thích Chi tiết Luồng Code & Cơ chế Nhận diện Bias:

1. **`__init__` (Lưu hàm Giám khảo LLM)**:
   - Lưu trữ `judge_llm_fn` (hàm nhận vào `prompt: str` và trả về `raw_response: str`). Thiết kế này cho phép dễ dàng swap giữa LLM thật (OpenAI) và Mock LLM khi chạy unit test.

2. **`score_response` (Tạo Prompt, Gọi Judge & Parse JSON Fallback)**:
   - *Tạo Prompt*: Kết hợp `question`, `answer` và `rubric` thành một câu lệnh yêu cầu Giám khảo LLM chấm điểm theo định dạng JSON.
   - *Cơ chế Fallback An toàn (Robust Parsing)*:
     - Thử parse trực tiếp `json.loads(raw_response)`.
     - Nếu LLM trả về văn bản có chứa block JSON (dạng Markdown ```json ... ```), sử dụng Regex `re.search(r"\{.*\}", ...)` để trích xuất JSON.
     - Nếu hoàn toàn không parse được JSON, tự động gán điểm mặc định `0.5` cho từng tiêu chí trong Rubric để không làm hỏng pipeline đánh giá.
   - *Output*: Trả về `{"scores": dict, "reasoning": str}`.

3. **`detect_bias` (Phát hiện 3 Loại Bias Cốt lõi của Giám khảo LLM)**:
   - **`leniency_bias` (Thiên vị Nương tay / Chấm nương)**: Kích hoạt (`True`) nếu điểm số trung bình trung bình của toàn bộ các tiêu chí trong batch $> 0.8$.
   - **`severity_bias` (Thiên vị Khắt khe / Chấm quá ép)**: Kích hoạt (`True`) nếu điểm số trung bình của toàn bộ các tiêu chí trong batch $< 0.3$.
   - **`positional_bias` (Thiên vị Vị trí)**: Kích hoạt (`True`) khi so sánh mẫu đầu và mẫu cuối trong batch có sự sụt giảm điểm số đáng kể ($> 0.2$), thể hiện việc LLM ưu ái các vị trí xuất hiện đầu tiên.

##### 3. Kết quả Kiểm thử Unit Test cho Task 3:
```bash
.venv/bin/pytest tests/test_solution.py -k TestLLMJudge -v
# Output: 4 passed in 0.02s (XANH 100% toàn bộ 4 unit tests của Task 3)!
```

---

#### 🟢 TASK 4: BENCHMARK RUNNER — [ĐÃ HOÀN THÀNH CODE]

##### 1. Đoạn Code Đã Triển Khai Trong `template.py`:
```python
class BenchmarkRunner:
    """
    Runs a full evaluation benchmark.
    """

    def run(
        self,
        qa_pairs: list[QAPair],
        agent_fn: Callable[[str], str],
        evaluator: RAGASEvaluator,
    ) -> list[EvalResult]:
        """Run all QA pairs through the agent and evaluate each result."""
        results: list[EvalResult] = []
        for pair in qa_pairs:
            answer = agent_fn(pair.question)
            res = evaluator.run_full_eval(
                answer=answer,
                question=pair.question,
                context=pair.context or "",
                expected=pair.expected_answer,
                contexts=pair.retrieved_contexts if pair.retrieved_contexts else None,
            )
            res.qa_pair = pair
            results.append(res)
        return results

    def generate_report(self, results: list[EvalResult]) -> dict[str, Any]:
        """Generate an aggregate report from evaluation results."""
        total = len(results)
        if total == 0:
            return {
                "total": 0,
                "passed": 0,
                "pass_rate": 0.0,
                "avg_faithfulness": 0.0,
                "avg_relevance": 0.0,
                "avg_completeness": 0.0,
                "avg_context_recall": None,
                "avg_context_precision": None,
                "failure_types": {},
            }

        passed_count = sum(1 for r in results if r.passed)
        avg_f = sum(r.faithfulness for r in results) / total
        avg_r = sum(r.relevance for r in results) / total
        avg_c = sum(r.completeness for r in results) / total

        recalls = [r.context_recall for r in results if r.context_recall is not None]
        precisions = [r.context_precision for r in results if r.context_precision is not None]

        avg_recall = (sum(recalls) / len(recalls)) if recalls else None
        avg_precision = (sum(precisions) / len(precisions)) if precisions else None

        failure_counts: dict[str, int] = {}
        for r in results:
            if r.failure_type:
                failure_counts[r.failure_type] = failure_counts.get(r.failure_type, 0) + 1

        return {
            "total": total,
            "passed": passed_count,
            "pass_rate": passed_count / total,
            "avg_faithfulness": avg_f,
            "avg_relevance": avg_r,
            "avg_completeness": avg_c,
            "avg_context_recall": avg_recall,
            "avg_context_precision": avg_precision,
            "failure_types": failure_counts,
        }

    def run_regression(self, new_results: list[EvalResult], baseline_results: list[EvalResult]) -> dict[str, Any]:
        """Compare new evaluation results against a baseline."""
        new_report = self.generate_report(new_results)
        base_report = self.generate_report(baseline_results)

        new_f = new_report["avg_faithfulness"]
        new_r = new_report["avg_relevance"]
        new_c = new_report["avg_completeness"]

        base_f = base_report["avg_faithfulness"]
        base_r = base_report["avg_relevance"]
        base_c = base_report["avg_completeness"]

        regressions = []
        if base_f - new_f > 0.05:
            regressions.append("faithfulness")
        if base_r - new_r > 0.05:
            regressions.append("relevance")
        if base_c - new_c > 0.05:
            regressions.append("completeness")

        return {
            "new_avg_faithfulness": new_f,
            "new_avg_relevance": new_r,
            "new_avg_completeness": new_c,
            "baseline_avg_faithfulness": base_f,
            "baseline_avg_relevance": base_r,
            "baseline_avg_completeness": base_c,
            "regressions": regressions,
            "passed": len(regressions) == 0,
        }

    def identify_failures(
        self,
        results: list[EvalResult],
        threshold: float = 0.5,
    ) -> list[EvalResult]:
        """Return EvalResults where any score is below threshold."""
        failing = []
        for r in results:
            if (
                r.faithfulness < threshold
                or r.relevance < threshold
                or r.completeness < threshold
                or not r.passed
            ):
                failing.append(r)
        return failing
```

##### 2. Giải thích Chi tiết Luồng Code & Logic Kiểm soát Chất lượng CI/CD:

1. **`run` (Duyệt toàn bộ Golden Dataset qua AI Agent & Evaluator)**:
   - Lần lượt gọi `agent_fn(pair.question)` cho từng cặp câu hỏi trong `qa_pairs`.
   - Nạp kết quả vào `evaluator.run_full_eval` cùng các thuộc tính `question`, `context`, `expected_answer` và `retrieved_contexts`.
   - Đảm bảo giữ nguyên `qa_pair` gốc trên đối tượng `EvalResult` trả về.

2. **`generate_report` (Tổng hợp Báo cáo Thống kê Aggregate)**:
   - Tính tổng số câu (`total`), số câu đạt (`passed`), tỷ lệ đỗ (`pass_rate = passed / total`).
   - Tính trung bình 3 metric chính: `avg_faithfulness`, `avg_relevance`, `avg_completeness`.
   - Tính trung bình 2 metric phía Retrieval (`avg_context_recall`, `avg_context_precision`) chỉ trên những kết quả có chứa điểm (bỏ qua những mẫu `None`). Nếu không có mẫu nào chứa điểm Retrieval $\rightarrow$ trả về `None`.
   - Đếm tần suất xuất hiện của các loại lỗi trong `failure_types` (`{"hallucination": count, ...}`).

3. **`run_regression` (Kiểm định Suy giảm Chất lượng - Regression Testing cho CI/CD)**:
   - Tạo báo cáo tổng hợp cho lượt chạy mới (`new_results`) và lượt chạy chuẩn (`baseline_results`).
   - Kiểm tra sụt giảm cho 3 metric Answer. Nếu điểm trung bình của bất kỳ metric nào ở bản mới bị **giảm quá $0.05$** so với baseline $\rightarrow$ ghi nhận tên metric đó vào danh sách `regressions`.
   - Nếu `len(regressions) == 0` $\rightarrow$ `passed = True` (Cho phép Merge/Deploy). Ngược lại $\rightarrow$ `passed = False` (Block Deployment).

4. **`identify_failures` (Trích xuất các Mẫu Lỗi phục vụ Phân tích)**:
   - Lọc ra toàn bộ các đối tượng `EvalResult` có bất kỳ metric nào $< \text{threshold}$ (mặc định $0.5$) hoặc có trạng thái `passed == False`.

##### 3. Kết quả Kiểm thử Unit Test cho Task 4:
```bash
.venv/bin/pytest tests/test_solution.py -k "TestBenchmarkRunner or TestRunRegression or TestRetrievalMetricWiring" -v
# Output: 12 passed in 0.02s (XANH 100% toàn bộ 12 unit tests liên quan đến Task 4)!
```

---

#### 🟢 TASK 5: FAILURE ANALYZER — [ĐÃ HOÀN THÀNH CODE]

##### 1. Đoạn Code Đã Triển Khai Trong `template.py`:
```python
class FailureAnalyzer:
    """
    Analyzes failed evaluation results to identify patterns and suggest fixes.
    """

    def categorize_failures(
        self, failures: list[EvalResult]
    ) -> dict[str, int]:
        """Count failures by failure_type."""
        counts: dict[str, int] = {}
        for f in failures:
            if f.failure_type:
                counts[f.failure_type] = counts.get(f.failure_type, 0) + 1
        return counts

    def find_root_cause(self, failure: EvalResult) -> str:
        """Suggest a root cause for a single failure based on its scores."""
        f, r, c = failure.faithfulness, failure.relevance, failure.completeness
        min_score = min(f, r, c)

        if min_score >= 0.5:
            return "Multiple issues detected — review full pipeline"

        low_count = sum(1 for s in (f, r, c) if s == min_score)
        if low_count > 1:
            return "Multiple issues detected — review full pipeline"

        if min_score == f:
            return "Context is missing or irrelevant — improve retrieval"
        elif min_score == r:
            return "Answer does not address the question — improve prompt clarity"
        else:
            return "Answer is missing key information — increase context window or improve generation"

    def generate_improvement_log(self, failures: list[EvalResult], suggestions: list[str]) -> str:
        """Generate a Markdown table logging failures and improvement actions."""
        lines = [
            "| Failure ID | Type | Root Cause | Suggested Fix | Status |",
            "|------------|------|------------|---------------|--------|",
        ]
        for idx, failure in enumerate(failures, start=1):
            fid = f"F{idx:03d}"
            ftype = failure.failure_type or "Unknown"
            root_cause = self.find_root_cause(failure)
            fix = suggestions[idx - 1] if idx - 1 < len(suggestions) else (suggestions[0] if suggestions else "Review pipeline")
            lines.append(f"| {fid} | {ftype} | {root_cause} | {fix} | Open |")
        return "\n".join(lines)

    def generate_improvement_suggestions(
        self, failures: list[EvalResult]
    ) -> list[str]:
        """Generate a prioritized list of improvement suggestions based on failure patterns."""
        if not failures:
            return []

        counts = self.categorize_failures(failures)
        suggestions: list[str] = []

        if counts.get("hallucination", 0) > 0 or any(f.faithfulness < 0.5 for f in failures):
            suggestions.append("Implement hallucination checker to filter unsupported claims")

        if counts.get("irrelevant", 0) > 0 or any(f.relevance < 0.5 for f in failures):
            suggestions.append("Improve prompt clarity and intent detection to address the question directly")

        if counts.get("incomplete", 0) > 0 or any(f.completeness < 0.5 for f in failures):
            suggestions.append("Increase chunk size in RAG pipeline to reduce context fragmentation")

        default_suggestions = [
            "Implement hallucination checker to filter unsupported claims",
            "Increase chunk size in RAG pipeline to reduce context fragmentation",
            "Add few-shot examples showing complete answers to improve completeness",
            "Optimize vector retrieval parameters (top_k and similarity threshold)",
        ]

        for s in default_suggestions:
            if len(suggestions) >= 3:
                break
            if s not in suggestions:
                suggestions.append(s)

        return suggestions
```

##### 2. Giải thích Chi tiết Luồng Code & Phương pháp Phân tích Nguyên nhân Gốc rễ (RCA):

1. **`categorize_failures` (Phân nhóm Lỗi theo Failure Taxonomy)**:
   - Thống kê danh sách lỗi theo 5 loại chuẩn: `hallucination`, `irrelevant`, `incomplete`, `off_topic`, `refusal`. Trả về một dict đếm tần suất (`{"hallucination": 3, ...}`).

2. **`find_root_cause` (Xác định Nguyên nhân Gốc rễ)**:
   - Tìm điểm số thấp nhất trong 3 Answer metrics (`faithfulness`, `relevance`, `completeness`).
   - Nếu `faithfulness` thấp nhất $\rightarrow$ *"Context is missing or irrelevant — improve retrieval"*.
   - Nếu `relevance` thấp nhất $\rightarrow$ *"Answer does not address the question — improve prompt clarity"*.
   - Nếu `completeness` thấp nhất $\rightarrow$ *"Answer is missing key information — increase context window or improve generation"*.
   - Nếu có nhiều hơn 1 điểm đồng vị trí thấp nhất $\rightarrow$ *"Multiple issues detected — review full pipeline"*.

3. **`generate_improvement_suggestions` (Đề xuất Giải pháp Hành động Được - Actionable Fixes)**:
   - Dựa trên các loại lỗi xuất hiện để sinh danh sách khuyến nghị ưu tiên (ví dụ: bổ sung hallucination checker, tối ưu chunk size, làm rõ prompt intent).
   - Đảm bảo danh sách luôn chứa ít nhất 3 gợi ý cụ thể.

4. **`generate_improvement_log` (Tạo Bảng Log Phân tích Lỗi dạng Markdown)**:
   - Sinh ra một bảng Markdown gồm 5 cột (`Failure ID`, `Type`, `Root Cause`, `Suggested Fix`, `Status`).
   - Tự động gán mã `F001`, `F002`, ... và mặc định trạng thái `Status = Open` để đưa vào báo cáo [reflection.md](reflection.md).

##### 3. Kết quả Kiểm thử Unit Test cho Task 5 & Toàn bộ Hệ thống:
```bash
.venv/bin/pytest tests/test_solution.py -v
# Output: 42 passed in 0.06s (XANH 100% HOÀN TOÀN 42/42 UNIT TESTS)!
```

> [!SUCCESS]
> **ĐÃ SAO CHÉP CODE HOÀN CHỈNH SANG [`solution/solution.py`](solution/solution.py)**  
> Tất cả 5 Task trong `template.py` đã hoàn thành xuất sắc và sẵn sàng cho chấm điểm tự động.

---

---

### 📍 Giai đoạn 3: Xây dựng & Validate Golden Dataset — [ĐÃ HOÀN THÀNH 100%]

#### 1. Đã hoàn thiện 20 QA Pairs trong [`golden_dataset.json`](golden_dataset.json):
- **5 Easy (`E01` $\rightarrow$ `E05`)**: Tra cứu thông tin thực tế 1 tài liệu duy nhất (Factual Lookup - ngày census date, mức học phí USD 420, mức học bổng 50%, tỷ lệ điểm danh 80%, số giờ thực tập 240h).
- **7 Medium (`M01` $\rightarrow$ `M07`)**: Tổng hợp quy trình và điều kiện liên kết từ 2 tài liệu (Multi-doc reasoning - thủ tục late add v2.0, ảnh hưởng census date tới học bổng, quy trình phúc khảo điểm, điểm W khi rút môn, holds tài chính, miễn giảm cho sinh viên đặc biệt, nâng cấp chính sách v1.0 $\rightarrow$ v2.0).
- **5 Hard (`H01` $\rightarrow$ `H05`)**: Xử lý logic nhiều điều kiện ràng buộc, mốc ngày tháng áp dụng hiệu lực chính sách, điều kiện nghỉ y tế vs nghỉ tự nguyện, quy định điểm I incomplete, điều kiện tốt nghiệp và hoàn tiền học phí.
- **3 Adversarial (`A01` $\rightarrow$ `A03`)**: Thử nghiệm các tình huống bẫy (Out-of-scope hỏi Harvard, Prompt Injection bảo reveal secret keys, False premise hỏi hoàn tiền 100% khi bỏ học tuần cuối). Cả 3 câu đều bắt buộc liên kết chứng minh từ tài liệu `00_system_scope.md`.

#### 2. Đảm bảo Quy tắc Evidence Provenance & Document Coverage:
- **100% Evidence Exact Match**: Mỗi đoạn văn trong trường `text` thuộc `contexts` đều là trích dẫn chính xác nguyên văn từng ký tự từ các file `.md` trong `data/student_services/`.
- **10/10 Document Coverage**: Bộ dữ liệu đã phủ toàn bộ 10 file tài liệu trong corpus nguồn từ `00_system_scope.md` đến `09_privacy_security_and_policy_updates.md`.

#### 3. Kết quả Kiểm tra Tự động (Validation Output):
```bash
.venv/bin/python validate_golden_dataset.py
```
*Kết quả Terminal*:
```text
Dataset: golden_dataset.json
Corpus:  data/student_services
QA pairs: 20
Difficulty: easy=5, medium=7, hard=5, adversarial=3
Document coverage: 10/10
PASS: dataset structure and evidence provenance are valid.
```

---

### 📍 Giai đoạn 4: Benchmark Hệ thống RAG thật & Báo cáo Phân tích Lỗi — [ĐÃ HOÀN THÀNH 100%]

#### 1. Thực thi Sinh câu trả lời RAG thật & Đánh giá Benchmark:
1. **Sinh câu trả lời từ RAG Agent thực tế**:
   ```bash
   .venv/bin/python domain_assistant.py
   ```
   *Kết quả*: Hệ thống RAG tự động đọc 20 câu hỏi từ `golden_dataset.json`, truy vấn corpus `data/student_services` và dùng OpenAI LLM (`gpt-4o-mini`) để tạo câu trả lời thực tế lưu tại `artifacts/actual_answers.json`.

2. **Đánh giá Benchmark tự động**:
   ```bash
   .venv/bin/python evaluate_answers.py
   ```
   *Kết quả*: Gọi Evaluation Core trong `template.py` để chấm 5 metrics và xuất báo cáo `artifacts/benchmark_results.json`.

#### 2. Thống kê Kết quả Benchmark Thực tế (Real Empirical Results):
- **Overall Pass Rate**: **70.0%** (14 / 20 test cases đạt điểm đỗ $\ge 0.5$).
- **Điểm trung bình 5 Metrics**:
  - `Context Recall`: **0.858** (Khả năng tìm đúng tài liệu liên quan rất cao)
  - `Context Precision`: **0.950** (Xếp vị trí tài liệu liên quan ở top đầu xuất sắc)
  - `Faithfulness`: **0.634** (Điểm yếu cốt lõi do LLM trả về từ ngữ nằm ngoài retrieved context)
  - `Relevance`: **0.760** (Bám sát câu hỏi nhưng bị phạt khi trả lời quá ngắn)
  - `Completeness`: **0.732** (Đáp ứng tốt hầu hết điều kiện)
- **Phân bố loại lỗi (Failure Types)**: 3 lỗi `off_topic` (15%) và 3 lỗi `hallucination` (15%).

#### 3. Phân tích Top 3 Worst Failures bằng phương pháp 5 Whys ([`reflection.md`](reflection.md)):
1. **`A01` (Score: 0.394 | `hallucination`)**: Câu hỏi ngoài phạm vi về đại học Harvard. LLM trả lời khuyên lên website Harvard thay vì dùng mẫu từ chối quy định trong `00_system_scope.md`.
   - *Root Cause*: Thiếu Out-of-Scope Intent Filter / Guardrail ở tiền xử lý query.
   - *Fix Proposal*: Tích hợp Intent Classification Guardrail và siết chặt System Prompt cho các câu hỏi ngoại vi.
2. **`A02` (Score: 0.398 | `off_topic`)**: Câu hỏi tấn công Prompt Injection yêu cầu lộ secret keys. LLM từ chối an toàn nhưng trả lời quá ngắn.
   - *Root Cause*: Thiếu mẫu từ chối chuẩn (Few-shot Prompt Alignment) cho Prompt Injection Defence.
   - *Fix Proposal*: Bổ sung Few-shot prompt ví dụ mẫu câu từ chối chuẩn trong System Prompt.
3. **`M04` (Score: 0.475 | `off_topic`)**: Câu hỏi về điểm W khi rút môn sau census date. Retriever bị kéo theo chunk nhiễu về việc bỏ học không xin phép.
   - *Root Cause*: `top_k=5` lấy thừa chunks nhiễu và LLM bị over-generation.
   - *Fix Proposal*: Áp dụng thuật toán Reranking `rerank_by_overlap` để lọc còn top 2-3 chunks sát nhất.

#### 4. Đã hoàn thiện toàn bộ file Báo cáo & Worksheet (Bao gồm cả 2 Bài tập Bonus +15đ):
- [`reflection.md`](reflection.md): Cập nhật 100% số liệu thực tế, bảng phân loại lỗi, phân tích 5 Whys, Failure Clustering, chiến lược CI/CD Regression Testing và Continuous Improvement Loop.
- [`exercises.md`](exercises.md): Hoàn thành 100% tất cả các câu hỏi lý thuyết Warm-up Part 1, Rubric Design Part 3.3, Checklist và **2 bài tập Bonus**:
  - **Exercise 3.4 (+10đ)**: So sánh chi tiết hai framework RAGAS vs DeepEval (về độ phức tạp setup, số lượng metrics, khả năng tích hợp CI/CD và kết quả thực nghiệm).
  - **Exercise 3.5 (+5đ)**: Thực nghiệm Reranking bằng `rerank_by_overlap` trên 5 test cases thực tế, chứng minh `Context Recall` giữ nguyên ($\Delta = 0.000$) do giữ nguyên tập chunks, trong khi `Context Precision` tăng ở các case có chunk bị đảo thứ tự.

---

---

## 🛠️ PHẦN 4: BẢNG LỆNH TRA CỨU NHANH (CHEAT SHEET)

| Thao tác | Câu lệnh Terminal | Kết quả mong đợi |
|---|---|---|
| **Cài môi trường** | `python3.12 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt` | Môi trường Python 3.11+ / 3.12 sẵn sàng |
| **Chạy toàn bộ Test Suite** | `.venv/bin/pytest tests/ -v` | Starter: 42 failed. Task 1 done: 3 passed. Full done: 42 passed |
| **Test riêng từng Task** | `.venv/bin/pytest tests/test_solution.py -k TestEvalResultOverallScore -v`<br>`.venv/bin/pytest tests/test_solution.py -k TestRAGASEvaluator -v`<br>`.venv/bin/pytest tests/test_solution.py -k TestLLMJudge -v`<br>`.venv/bin/pytest tests/test_solution.py -k TestBenchmarkRunner -v`<br>`.venv/bin/pytest tests/test_solution.py -k TestFailureAnalyzer -v` | Xanh từng phần code tương ứng |
| **Validate Golden Dataset** | `python validate_golden_dataset.py` | Golden dataset VALID (20 QA, 4 tiers) |
| **Sinh RAG Answers** | `python domain_assistant.py` | Tạo file `artifacts/actual_answers.json` |
| **Chạy Benchmark Engine** | `python evaluate_answers.py` | Tạo file `artifacts/benchmark_results.json` |
| **Copy Solution** | `cp template.py solution/solution.py` | Sẵn sàng nộp bài |

---
*Tài liệu hướng dẫn cá nhân chuẩn hóa cho Day 14 AI Evaluation Lab.*
