# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Lâm Thành Bảo
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 2026-08-19

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)

> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

_Trả lời:_

- **Ví dụ từ dữ liệu:** `chunk_id = 5d1df43703396c12c3c2::c0000`
  - Câu gốc: _"That is until defense contractor Lockheed noticed it and had **his** works translated into English; Ufimtsev's work became the basis for modern-day stealth technology..."_
  - Câu sau khi resolve: _"...and had **Lockheed** works translated into English; Ufimtsev's work became the basis..."_
- **Hiện tượng:** Đại từ `his` thực chất trỏ về nhà vật lý Liên Xô **Ufimtsev** (được nhắc tên ngay sau đó trong cùng câu), nhưng module coref lại chọn `Lockheed` — danh từ riêng gần nhất phía trước — làm antecedent. Đây là lỗi kinh điển của heuristic "nearest-noun": model ưu tiên thực thể gần về mặt vị trí thay vì đúng về mặt cú pháp/ngữ nghĩa (chủ thể sở hữu "works" phải là một _Person_, không phải _Company_).
- **Hậu quả đối với Graph:** Nếu bước NER+RE ở Module 2 trích xuất quan hệ từ câu đã bị resolve sai này, quan hệ `DEVELOPED` (hoặc tương tự) về công nghệ stealth sẽ bị gán nhầm cho **Lockheed** thay vì đúng ra phải ghi nhận **Ufimtsev** là tác giả gốc, còn Lockheed chỉ là bên "phát hiện/ứng dụng" — tạo ra False Edge làm sai lệch quan hệ phát minh vs. thương mại hóa. Đây chính là lý do prompt coreference được thiết kế "conservative" (chỉ resolve khi antecedent rõ ràng), nhưng ranh giới "rõ ràng" vẫn có thể bị model diễn giải sai khi có nhiều ứng viên cùng loại từ trong câu.

---

### 2. Entity Resolution Threshold & Lexical Guard

> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

_Trả lời:_

- **Ngưỡng cosine similarity:** `threshold = 0.90` (giữ nguyên mặc định của `build_resolution_map()`).
- **Quan sát thực tế trên dữ liệu đã trích xuất (60 entity, `EXTRACTION_MAX_CHUNKS=80` do giới hạn rate-limit Groq free tier):** không có cặp nào vượt ngưỡng 0.85 trong lần chạy này — điểm similarity cao nhất quan sát được trong toàn bộ 81 cặp candidate là:

| Cặp thực thể                    | Similarity | Quyết định   |
| ------------------------------- | ---------- | ------------ |
| `Nexsyst 360` vs `Nexxiot`      | 0.582      | REJECT_GUARD |
| `ServiceNow Inc.` vs `NIO Inc.` | 0.537      | REJECT_GUARD |
| `Lockheed` vs `Mitsubishi`      | 0.494      | REJECT_GUARD |

- **Lý do chặn:** Cả 3 cặp trên có similarity cao vì embedding model bắt được sự tương đồng về _loại hình văn bản_ (đều là tên công ty viết hoa, đôi khi có hậu tố "Inc."), nhưng sau khi `strip_suffix()` chuẩn hóa, tên lõi khác nhau hoàn toàn (`nexsyst 360` ≠ `nexxiot`, `servicenow` ≠ `nio`) nên `SequenceMatcher` trả về ratio thấp hơn 0.72 → đúng là REJECT.
- **Nhận xét về quy mô mẫu:** Vì bài lab giới hạn `EXTRACTION_MAX_CHUNKS=80` (để né rate-limit Groq free tier), tập entity thu được nhỏ và đa dạng ngành nghề nên chưa xuất hiện tình huống kinh điển "False Merge nguy hiểm" như `Sam Altman` vs `Steve Altman` hay `Apple` vs `Apple Watch`. Ở quy mô đầy đủ (1500+ chunk hoặc 350MB), với mật độ lặp lại tên công ty lớn (Microsoft, Google, Apple...) cao hơn nhiều, khả năng xuất hiện cặp similarity > 0.85 bị Guard chặn đúng sẽ tăng đáng kể — đây là lý do Lexical Guard bắt buộc phải có ở production, không thể chỉ dựa vector threshold.

---

### 3. Đồ thị & Super-node Mitigation

> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

_Trả lời:_

- **Số liệu đồ thị thật (Neo4j Aura, sau khi chạy pipeline đầy đủ):** 60 node, 38 edge, **0 invalid_provenance_edges** (đạt yêu cầu bắt buộc).
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
| ---- | ------------ | -------------------- | -------------------- |
| 1    | Railergy     | Company              | 6                    |
| 2    | Max Homa     | Person               | 4                    |
| 3    | Meelo        | Company              | 3                    |

- **Nhận xét quan trọng:** Ở quy mô lab (`EXTRACTION_MAX_CHUNKS=80`), **chưa có node nào vượt ngưỡng `SUPER_NODE_DEGREE=100`** để kích hoạt cơ chế cắt tỉa — `graph_supernode_events = 0` ở toàn bộ 6 câu golden đã eval. Điều này đúng như kỳ vọng: super-node (kiểu Microsoft/Google có hàng nghìn kết nối) chỉ xuất hiện khi đồ thị đủ lớn (ví dụ chạy với `EXTRACTION_MAX_CHUNKS` ở mức hàng nghìn chunk như README gợi ý mặc định 400, hoặc scale lên toàn bộ 350MB).
- **Ưu điểm & Rủi ro của Temporal Mitigation (dựa trên thiết kế, chưa quan sát được trực tiếp do đồ thị lab chưa đủ lớn để kích hoạt):**
  - _Ưu điểm:_ Giới hạn cứng 50 cạnh/`GLOBAL_EDGE_CAP=250` giữ context trong ngân sách token hợp lý cho LLM sinh câu trả lời, tránh việc một câu hỏi chạm vào node như "Microsoft" kéo theo hàng nghìn cạnh làm nổ context và tăng latency/chi phí phi tuyến.
  - _Rủi ro:_ Ưu tiên `ORDER BY published_date DESC` thiên lệch về sự kiện gần đây — nếu câu hỏi liên quan lịch sử xa (ví dụ "Microsoft mua lại công ty nào năm 2015?"), cạnh cũ có thể bị cắt mất dù vẫn liên quan trực tiếp đến câu hỏi, dẫn tới câu trả lời thiếu hoặc sai lệch thời gian.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

> **Lưu ý minh bạch:** Do giới hạn quota Groq free tier (200.000 token/ngày) bị chạm giữa quá trình đánh giá, chỉ **6/12 câu golden** (nhóm `cross-doc`: 4 câu, `factoid`: 2 câu) hoàn thành đánh giá đầy đủ; chưa có câu nhóm `multi-hop`. Số liệu dưới đây phản ánh đúng 6 câu đã chạy thật, không suy diễn.

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge, trung bình 6 câu):

| Tiêu chí đánh giá             | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích                                                 |
| ----------------------------- | -------- | -------- | ------------------------ | ------------------------------------------------------------------ |
| **Comprehensiveness (1–5)**   | 1.00     | 1.00     | 0                        | Hai phương pháp giống hệt nhau — cả hai đều đạt điểm sàn tối thiểu |
| **Faithfulness (1–5)**        | 1.00     | 1.00     | 0                        | Giống hệt nhau                                                     |
| **Multi-hop Reasoning (1–5)** | 1.00     | 1.00     | 0                        | Giống hệt nhau                                                     |
| **Latency trung bình (s)**    | 4.17s    | 2.26s    | GraphRAG nhanh hơn ~46%  | Xem giải thích bên dưới                                            |
| **Token usage trung bình**    | 779      | 611      | GraphRAG ít hơn ~22%     | GraphRAG dùng context ngắn gọn hơn nhờ linearize subgraph          |

#### Phân tích 2 Ca lỗi Điển hình:

1. **Ca lỗi: cả Flat RAG lẫn GraphRAG cùng thất bại — `G5000-02` ("Did the first two Aeris/Ericsson reports describe a completed acquisition or a planned transfer...")**
   - _Nguyên nhân gốc rễ:_ Bộ golden dataset 50 câu do giảng viên cung cấp được xây dựng dựa trên **5.000 dòng đầu** của dataset gốc (tham chiếu `row 33`, `row 1746`...), trong khi pipeline lab (do scale guard `LAB_MAX_CHUNKS=1500` + `EXTRACTION_MAX_CHUNKS=80`) chỉ index một **subset ngẫu nhiên nhỏ hơn nhiều** và không đảm bảo trùng đúng các dòng mà golden dataset tham chiếu tới. Kết quả: cả Flat Index lẫn Knowledge Graph đều **không hề chứa** bài báo Aeris/Ericsson.
   - _Cả 2 phương pháp trả lời giống nhau:_ "The supplied excerpts do not contain any of the Aeris/Ericsson reports... [no relevant chunk found]" — đây là hành vi **đúng** của hệ thống (không bịa đặt khi thiếu context), nhưng LLM-as-a-Judge vẫn chấm 1/5 cho comprehensiveness/faithfulness vì so với `reference_answer` thì câu trả lời "không có gì" là không đầy đủ.
   - _Bài học:_ Đây không phải lỗi logic retrieval của GraphRAG hay Flat RAG, mà là **lỗi thiết kế thực nghiệm** — golden dataset và corpus đã ingest phải cùng phạm vi dữ liệu thì benchmark mới có ý nghĩa. Khi scale lên full 350MB hoặc dùng đúng golden dataset do giảng viên cung cấp, cần đảm bảo `LAB_MAX_ARTICLES`/`EXTRACTION_MAX_CHUNKS` bao phủ đúng các dòng golden dataset tham chiếu.

2. **Ca lỗi hiệu năng (không phải chất lượng) — `G5000-04` ("Which two named Ericsson IoT businesses recur across multiple reports...")**
   - _Quan sát:_ Flat RAG latency = 9.12s trong khi GraphRAG chỉ 0.85s cho cùng câu hỏi (dù cả hai cùng trả lời "không tìm thấy context" như case 1).
   - _Nguyên nhân:_ Flat RAG phải encode query + search trên toàn bộ FAISS index (1500 vector) mỗi lần, trong khi GraphRAG thất bại sớm ở bước `match_seeds()` (không tìm thấy seed entity phù hợp trong graph nhỏ) nên trả về context rỗng gần như ngay lập tức, tiết kiệm được vòng gọi LLM tốn thời gian nhất.
   - _Đề xuất khắc phục:_ Thêm bước "context sufficiency check" trước khi gọi LLM sinh câu trả lời tốn kém (tương tự cơ chế `context_sufficient()` trong Bonus Self-Correction) để cả hai pipeline đều có thể early-exit khi retrieval rỗng, tránh lãng phí token/latency không cần thiết.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent

> **Trade-offs, Agent Control & Scale 350MB:**
>
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

_Trả lời:_

- **Đánh đổi Quality vs Cost vs Latency:** Trong sample thực tế (6 câu), GraphRAG rẻ hơn (~22% ít token hơn) và nhanh hơn (~48%) Flat RAG mà chất lượng không đổi — nhưng đây là kết quả trong tình huống **cả hai cùng không tìm thấy context** (early-exit), không phản ánh chi phí thật khi graph traversal thành công (khi đó GraphRAG phải cộng thêm chi phí `match_seeds()` gọi LLM trích seed entity + BFS traversal Cypher, tốn hơn Flat RAG). Indexing overhead của GraphRAG cũng cao hơn hẳn: cần thêm NER+RE extraction (10 batch Groq call), Entity Resolution (embedding + FAISS), và bulk insert Neo4j — trong khi Flat RAG chỉ cần 1 lần encode toàn bộ chunk.
- **Quyết định từ chối đề xuất của AI Coding Agent:** Khi quá trình Golden Evaluation chạm giới hạn quota Groq (200k token/ngày, chỉ còn 5/12 câu), AI Coding Agent đề xuất tiếp tục chạy nền, tự động retry mỗi 11 phút cho tới khi quota hồi (ước tính có thể mất tới ~2.5 giờ). Tôi đã **từ chối** phương án này và chọn dừng lại với 6 câu đã có, chấp nhận rủi ro bị trừ điểm rubric vì thiếu nhóm `multi-hop`, thay vì đánh đổi thời gian thực tế lấy một benchmark đầy đủ hơn — ưu tiên có deliverable đúng hạn nộp bài hơn là benchmark hoàn chỉnh 100%.
- **Giải pháp scale 350MB:** Bottleneck đầu tiên chắc chắn là **NER+RE extraction qua LLM** — với ~100.000 bài báo, ngay cả với batch_size=8 cũng cần hàng chục nghìn API call, vượt xa quota free tier trong một ngày (thực tế đã thấy: chỉ 80 chunk × 2 loại call (coref+extraction) đã tốn gần hết 200k token/ngày). Giải pháp: (1) chuyển sang mô hình local nhỏ hơn (ví dụ Llama 3.1 8B chạy on-prem) cho bước extraction số lượng lớn, chỉ dùng Groq/OpenAI cho các bước cần độ chính xác cao (judge, coreference khó); (2) async batch queue với nhiều API key/nhiều provider xoay vòng; (3) Entity Resolution ở quy mô lớn cần chuyển từ FAISS FlatIP ($O(N^2)$-ish khi search) sang HNSW/IVF index để tránh bùng nổ thời gian tính similarity.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng          | Module tương ứng | Hàm / Khối code cụ thể                       | Quan sát thực tế & Đánh giá                                                                                                                                                                                    |
| ---------------------------------- | ---------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Conservative Coreference**       | Module 1         | `resolve_coref_batch()`                      | Hoạt động đúng thiết kế "conservative" phần lớn, nhưng vẫn có lỗi chọn sai antecedent khi nhiều ứng viên cùng loại (xem mục 1) — 13/80 chunk có `unresolved_mentions` được ghi nhận minh bạch thay vì bịa đặt. |
| **Schema & Allowlist Guard**       | Module 2         | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`    | Lọc hiệu quả — toàn bộ 25 triple thô đều nằm trong 3 node type và 7/8 relation type cho phép (không thấy `PARTNERED_WITH` bị lạm dụng quá mức dù chiếm 11/25 quan hệ).                                         |
| **Bulk Cypher Ingestion**          | Module 2         | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` batch hoạt động đúng, MERGE theo `(id)` và `(source_chunk_id)` tránh insert trùng khi rerun — xác nhận qua sanity check `invalid_provenance_edges = 0`.                                               |
| **Entity Resolution & Union-Find** | Module 3         | `build_resolution_map()`, `UF`               | 60 entity gộp từ nhiều mention thô, 0 cặp MERGE_VECTOR trong sample nhỏ này (do similarity cao nhất chỉ 0.58, dưới ngưỡng 0.90) — cho thấy threshold khá bảo thủ ở quy mô nhỏ.                                 |
| **Super-node Degree Cap**          | Module 4         | `retrieve_graph_context()`                   | Logic đúng nhưng chưa được kích hoạt thực tế (degree cao nhất chỉ 6, xa ngưỡng 100) — cần graph lớn hơn để kiểm chứng hành vi cắt tỉa thật.                                                                    |
| **LLM-as-a-Judge Evaluation**      | Module 5         | `judge_answer()`                             | Judge chấm nhất quán và hợp lý — phát hiện đúng cả 6 câu đều là "no relevant context" và chấm điểm sàn phù hợp với rationale rõ ràng.                                                                          |

---

### 2. Quá trình Debugging & Bài học

- **Lỗi kỹ thuật phức tạp nhất gặp phải:** `Segmentation fault` (exit code 139) khi chạy notebook local trên macOS, xảy ra ngay sau bước load `sentence-transformers` embedding model trong Entity Resolution. Nguyên nhân là xung đột OpenMP runtime giữa `faiss-cpu` và `torch` (cả hai đều bundle riêng thư viện OpenMP) — một lỗi native-level rất khó chẩn đoán vì không có traceback Python rõ ràng, chỉ có exit code.
- **Cách xử lý thành công:** Cô lập vấn đề bằng cách chạy lại đúng đoạn `faiss + sentence-transformers` tối thiểu trong một process test riêng để tái hiện lỗi nhanh, sau đó áp dụng workaround chuẩn cho xung đột OpenMP trên macOS: đặt biến môi trường `KMP_DUPLICATE_LIB_OK=TRUE` và `OMP_NUM_THREADS=1` trước khi chạy. Ngoài ra còn gặp một lỗi phụ đáng chú ý: `pandas 3.0` đổi hành vi mặc định của `DataFrameGroupBy.apply()` khiến cột dùng để `groupby` (`"group"`) bị loại khỏi kết quả — sửa bằng cách thay `.groupby().apply()` bằng vòng lặp `pd.concat([...])` tường minh, không phụ thuộc hành vi ngầm định của pandas.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

> ⚠️ _Phần này cần điền theo đồ án cá nhân thực tế của bạn — nội dung dưới đây là khung gợi ý, hãy thay bằng thông tin đúng với dự án của bạn trước khi nộp._

- **Tên đồ án / Dự án:** [Tên dự án của bạn]
- **Đặc thù bài toán & Lý do chọn giải pháp:** [Bài toán của bạn có cần truy vết quan hệ nhiều bước giữa các thực thể (multi-hop) hay so sánh chéo nhiều nguồn tài liệu (cross-doc) không? Nếu câu hỏi người dùng chủ yếu là tra cứu 1-hop đơn giản, Flat/Hybrid RAG có thể đã đủ và rẻ hơn nhiều so với GraphRAG.]
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `...`
  - Relations: `...`
- **Chiến lược xử lý Super-node & Entity Resolution:** [Dựa trên bài học từ lab: cần threshold + lexical guard kết hợp, và cơ chế cap cạnh theo thời gian/độ quan trọng nếu miền dữ liệu của bạn có các thực thể trung tâm (hub) bị kết nối quá nhiều.]

---

## 🎯 TỰ ĐÁNH GIÁ

> Điền điểm tự chấm (1–5) theo đánh giá cá nhân của bạn trước khi nộp.

| Tiêu chí                             | Điểm tự chấm (1–5) | Ghi chú |
| ------------------------------------ | ------------------ | ------- |
| Mức độ hiểu bài giảng GraphRAG       |                    |         |
| Khả năng kiểm soát AI Coding Agent   |                    |         |
| Chất lượng đồ thị tri thức xây dựng  |                    |         |
| Khả năng phân tích và debug hệ thống |                    |         |
