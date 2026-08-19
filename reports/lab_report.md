# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trần Công Đức
**Khóa học:** K3 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

> Báo cáo này là bản tổng hợp. Chi tiết đầy đủ nằm ở:
>
> - [`technical_defense.md`](technical_defense.md) — 10 câu thuyết minh kỹ thuật
> - [`failure_analysis.md`](failure_analysis.md) — phân tích 11 ca lỗi theo truy vết nguyên nhân gốc rễ
> - [`reflection_TranCongDuc.md`](reflection_TranCongDuc.md) — mapping bài giảng + action plan

---

## Thông số hệ thống

| Hạng mục                                               | Giá trị                                                                              |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| Corpus gốc → sau lọc miền công nghệ                    | 514.417 → 22.947 bài (4,5%)                                                          |
| Bài đưa vào pipeline / chunk                           | 1.500 / **1.506**                                                                    |
| Chunk qua NER+RE                                       | **400**                                                                              |
| Triple trích xuất                                      | **182**                                                                              |
| **Node / Edge trong Neo4j**                            | **215 / 181**                                                                        |
| Phân bố node                                           | Company 118 · Technology 75 · Person 22                                              |
| **Cạnh thiếu `source_chunk_id` hoặc `published_date`** | **0** ✅                                                                             |
| Bảng audit Entity Resolution                           | **21 dòng** (MERGE_LEXICAL 6 · MERGE_MANUAL 4 · MERGE_VECTOR 3 · REJECT_THRESHOLD 8) |
| Community detection (bonus)                            | 53 cụm / 215 node                                                                    |

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)

_Trả lời:_

- **Ví dụ từ dữ liệu:** chunk `f9f06b416e5553944d89::c0000`
  ```
  GỐC : Adobe Express Gets Generative AI… The company's all-purpose creation app…
  SAU : Adobe Express Gets Generative AI… Adobe Express's all-purpose creation app…
  ```
- **Hiện tượng:** `The company` trỏ tới **Adobe** (_công ty_), nhưng model phân giải thành **Adobe Express** (_sản phẩm_). Chọn tiền ngữ gần nhất về vị trí thay vì đúng về ngữ nghĩa.
- **Hậu quả đối với Graph:** NER+RE đọc văn bản đã sửa và gán quan hệ cho **sai chủ thể** — sinh ra `Adobe Express -DEVELOPED-> …` thay vì `Adobe -DEVELOPED-> Adobe Express`. Ba thiệt hại: (a) node giả, sản phẩm bị nâng thành chủ thể hành động; (b) cạnh lẽ ra thuộc node `Adobe` bị tách sang node khác, làm đứt đường multi-hop; (c) **provenance không cứu được** — cạnh vẫn đủ `source_chunk_id` và `evidence`, chỉ là evidence trích từ văn bản đã sai.

- **Định lượng:** 153/394 chunk (38,8%) bị coref sửa đổi. Nghiêm trọng hơn: **7 chunk mất >20% nội dung** do model tự ý xoá mệnh đề — tệ nhất chỉ còn **31,5%** văn bản gốc. Đây là bước lẽ ra chỉ được thay đại từ, tuyệt đối không được xoá nội dung.
- **Mặt được:** 28/400 chunk giữ nguyên `unresolved_mentions` (`us`, `you`, `it's`) thay vì đoán bừa — đúng quy tắc conservative. Nhưng prompt conservative là **chưa đủ**; cần ràng buộc cứng ở tầng code (từ chối nếu `len(resolved) < 0.9 × len(original)`).

---

### 2. Entity Resolution Threshold & Lexical Guard

_Trả lời:_

- **Ngưỡng cosine similarity:** `threshold = 0.90` để gộp; Lexical Guard `SequenceMatcher ≥ 0.72`; ngưỡng **ghi audit** hạ xuống `0.80` để nhìn được vùng xám.

- **Cặp bị Guard chặn:** **không có cặp nào.** Bảng audit 21 dòng có 0 dòng `REJECT_GUARD` — mọi cặp đạt cosine ≥ 0,90 đều thực sự là biến thể của cùng một thực thể. Tôi không bịa ca để lấp chỗ trống.

- **Nhưng dữ liệu thật cho kết luận đáng giá hơn** — đo riêng từng tầng:

  | Cặp                           | cosine    | lexical ratio | Qua vector? | Qua guard? | Kết cục                   |
  | ----------------------------- | --------- | ------------- | ----------- | ---------- | ------------------------- |
  | `Sam Altman` / `Steve Altman` | 0,824     | **0,727**     | ❌          | ✅         | Thoát nhờ **tầng vector** |
  | `RTX 4090` / `RTX 4070`       | 0,803     | 0,875         | ❌          | ✅         | Thoát nhờ **tầng vector** |
  | `ChatGPT` / `ChatGPT API`     | 0,836     | 0,778         | ❌          | ✅         | Thoát nhờ **tầng vector** |
  | **`Tim Cook` / `Tim Cooke`**  | **0,902** | ~0,94         | ✅          | ✅         | 🔴 **SẼ GỘP NHẦM**        |

- **Lý do & kết luận:** ngưỡng 0,72 **trượt chính ví dụ đề bài nêu đích danh** — `Sam Altman` vs `Steve Altman` đạt 0,727, vượt ngưỡng đúng 0,007. Cặp này sống sót hoàn toàn nhờ tầng vector. Và `Tim Cook` / `Tim Cooke` vượt **cả hai tầng** → lỗ thủng còn tồn tại. Đề xuất: với `Person` bỏ ratio ký tự, yêu cầu khớp token-level tuyệt đối; với `Technology` thêm luật _khác chữ số ⇒ không gộp_.

- **Khiếm khuyết kiến trúc đã sửa:** hai tầng nối bằng **AND** nên Guard chỉ được _từ chối_, không được _cứu_. `Adobe Inc.`/`Adobe` (0,814) và `Intel Corp`/`Intel` (0,815) bị loại oan ở tầng vector vì MiniLM xử lý hậu tố pháp nhân kém. Đã thêm nhánh `MERGE_LEXICAL` chạy song song → audit từ 6 lên 21 dòng, thu hồi 6 ca gộp bị bỏ sót.

---

### 3. Đồ thị & Super-node Mitigation

_Trả lời:_

- **Top 3 Super-nodes:**

| Hạng | Tên thực thể        | Loại    | Bậc kết nối |
| ---- | ------------------- | ------- | ----------- |
| 1    | Microsoft           | Company | **19**      |
| 2    | Apple               | Company | **18**      |
| 3    | Activision Blizzard | Company | **15**      |

- **Sự thật phải nói rõ:** degree cao nhất là **19**, ngưỡng `SUPER_NODE_DEGREE` là **100** → **nhánh cắt tỉa chưa từng chạy**, `graph_supernode_events = 0` ở cả 6 câu hỏi. Đã bổ sung `test_supernode_cap_enforced()` kiểm chứng từng thành phần trên dữ liệu thật: (a) `recent_edges` tôn trọng LIMIT với cap 3/5/10/50; (b) 5 cạnh giữ lại khớp chính xác 5 `published_date` lớn nhất; (c) công thức chọn limit đúng ở `degree=19 → 50` và `degree=101/5000 → 50`.

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - _Ưu điểm:_ chặn bùng nổ context (node degree 5.000 không cắt sẽ sinh ~500K ký tự, vượt xa `MAX_GRAPH_CONTEXT_CHARS = 14000`); chi phí dự đoán được nhờ `GLOBAL_EDGE_CAP = 250`; hợp với miền tin tức vì quan hệ mới thường là quan hệ đang có hiệu lực.
  - _Rủi ro (thật, không lý thuyết):_ **10/19 cạnh của Microsoft đều là cùng một quan hệ** `ACQUIRED → Activision Blizzard` lặp lại từ 10 bài báo. Ở quy mô lớn, một sự kiện được đưa tin dày đặc sẽ **chiếm trọn hạn ngạch 50 cạnh**, đẩy mọi quan hệ khác ra ngoài — "cắt theo thời gian" biến thành **"cắt theo độ nóng của tin tức"**. Ngoài ra câu hỏi lịch sử xa sẽ bị cắt mất, và `FOUNDED` (sự kiện một lần) không nên bị đối xử như `PARTNERED_WITH` (trạng thái đổi theo thời gian).
  - _Đề xuất:_ khử trùng lặp `(source, relation, target)` **trước** khi cắt, và phân bổ hạn ngạch **theo từng loại quan hệ**.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

> **Trạng thái:** 4/6 câu đã chấm xong (G01–G04). Hai câu `cross-doc` chờ hạn mức token/ngày của Groq hồi lại. Số liệu dưới là **thật của 4 câu đã hoàn tất**, không ngoại suy.

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá             | Flat RAG    | GraphRAG    | Δ    | Nhận xét phân tích                                        |
| ----------------------------- | ----------- | ----------- | ---- | --------------------------------------------------------- |
| **Comprehensiveness (1–5)**   | 3,0         | **5,0**     | +2,0 | Toàn bộ chênh lệch đến từ 2 câu Flat RAG trượt hoàn toàn  |
| **Faithfulness (1–5)**        | 3,0         | **5,0**     | +2,0 | GraphRAG trích dẫn tới **từng cạnh**, không chỉ tới chunk |
| **Multi-hop Reasoning (1–5)** | 3,0         | **5,0**     | +2,0 | Đúng loại câu hỏi GraphRAG sinh ra để giải                |
| **Latency trung bình (s)**    | 0,67 – 28,0 | 0,66 – 24,9 | ~0   | Chênh do round-trip Cypher, không do LLM                  |
| **Token usage trung bình**    | 820 – 1.718 | 668 – 4.399 | +2×  | GraphRAG **rẻ hơn** ở factoid, đắt hơn ở cross-doc        |

**Điểm quan trọng nhất khi đọc bảng:** GraphRAG **hoà** với Flat RAG ở 2/4 câu; toàn bộ khoảng cách +2,0 điểm đến từ 2 câu Flat RAG được **1/5**. Giá trị của GraphRAG là **giảm tỷ lệ thất bại thảm hoạ**, không phải nâng điểm trung bình.

#### Phân tích 2 Ca lỗi Điển hình:

**1. Ca lỗi Flat RAG thất bại (GraphRAG thành công):**

- _Question ID & Câu hỏi:_ **G02** — _"According to the news corpus, which company did Apple invest in?"_ (đáp án: `SoftBank's UK chip unit`)
- _Tại sao Flat RAG thất bại?_ Vector search xếp hạng theo độ giống ngữ nghĩa toàn chunk. Các chunk về vốn hoá thị trường Apple (`market cap`, `$3 trillion`, `investors`) trùng cả **chủ đề tài chính** lẫn **thực thể Apple** nên chiếm hết top-6, đẩy chunk chứa sự kiện đầu tư thật ra ngoài. Đây là **semantic drift**: đúng chủ đề, trượt sự kiện. Tăng `k` chỉ làm loãng context. Flat RAG trả lời _"I couldn't find any mention…"_ → 1/5.
- _GraphRAG đã giải quyết như thế nào?_ Quan hệ đã được **vật chất hoá thành cạnh** `Apple -INVESTED_IN-> SoftBank's UK chip unit` kèm provenance. Truy hồi chuyển từ _"chunk nào trông giống câu hỏi"_ sang _"Apple có những cạnh nào"_ → **5/5** với trích dẫn `【chunk_id=2a8d4bb2…】`.

**2. Ca lỗi GraphRAG thất bại:**

- _Question ID & Câu hỏi:_ **G01** — _"which company did Microsoft acquire?"_ → GraphRAG **1/5** dù đồ thị có sẵn **10 cạnh** `Microsoft -ACQUIRED-> Activision Blizzard`.
- _Nguyên nhân:_ `extract_seeds()` trả về seed = **`GitHub`**, không phải `Microsoft`. LLM đã **dùng kiến thức tham số để đoán ĐÁP ÁN** (Microsoft từng mua GitHub ngoài đời) thay vì trích thực thể có trong câu hỏi — bất chấp prompt ghi rõ `Do not answer the question`. G02 tương tự, trả về `Mojang`. `GitHub` không có trong đồ thị → `NO_SEED` → context graph rỗng → **GraphRAG âm thầm thoái hoá thành Flat RAG kém hơn** (k=4 thay vì k=6) mà không báo lỗi.
- _Đề xuất khắc phục (đã triển khai):_ hai lớp — (a) _lexical guard_ chỉ giữ seed thực sự xuất hiện trong câu hỏi; (b) _lớp không dùng LLM_ `graph_seeds_from_text()` quét thẳng `name_norm` trong Neo4j, biến seed matching từ **tác vụ sinh** (có thể bịa) thành **tác vụ tra cứu** (không thể bịa). **Kết quả: G01 và G02 đều từ 1/5 → 5/5.**

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent

_Trả lời:_

- **Đánh đổi Quality vs Cost vs Latency:** GraphRAG dịch chuyển chi phí từ **lúc truy vấn** sang **lúc dựng index** — Flat RAG index xong trong ~1 phút, GraphRAG cần **~75 phút** (400 lượt gọi LLM cho coref + NER/RE ở hạn mức 8.000 token/phút). Đổi lại: chất lượng +2,0/5 và khả năng truy vết tới **từng cạnh** thay vì tới chunk. Nếu corpus tĩnh và truy vấn nhiều thì đáng; nếu corpus đổi liên tục mà truy vấn thưa thì chi phí trích xuất không bao giờ hoàn vốn.

- **Quyết định từ chối AI Coding Agent:**
  1. **Từ chối đổi sang `groq/compound-mini`** dù nhanh gấp 8,75 lần (70.000 vs 8.000 token/phút) — model này tự bật **web search**, sẽ kéo thông tin ngoài context vào câu trả lời và **làm hỏng chính thứ benchmark đo: Faithfulness**. Điểm số sẽ đẹp hơn nhưng vô nghĩa.
  2. **Từ chối giảm `EXTRACTION_MAX_CHUNKS` từ 400 xuống 150** để lách trần quota — đây là con số Scale Guard đề bài quy định; lách bằng cách thu nhỏ bài toán là né vấn đề.
  3. **Từ chối bọc `try/except` rộng quanh cả batch** — chính phạm vi rộng đó đã làm **mất 34% chunk**.
  4. **Từ chối bỏ qua 2 câu trả về `NaN`** — `NaN` ở đây là **triệu chứng của một lỗi thật**, che đi thì bảng benchmark nói dối.

- **Giải pháp scale 350MB (~100.000 bài):** bottleneck **không phải Neo4j hay FAISS**. Số đo thật: LLM extraction ~31 phút cho 400 chunk → ngoại suy **~130 giờ**; embedding ~1,1 giờ; Neo4j bulk insert ~10 phút. Bottleneck lệch **hai bậc độ lớn**, và trần thật là **200.000 token/NGÀY/model**. Giải pháp theo thứ tự: (1) **tầng lọc trước khi gọi LLM** — lab này đã giảm corpus 95,5% mà không mất bài liên quan; (2) **song song hoá theo shard**, mỗi shard một bucket quota riêng; (3) **checkpoint bắt buộc** — đã chứng minh: chạy lại Phase 1 mất 70 giây thay vì 2 giờ; (4) **trích xuất phân tầng** — 0,46 triple/chunk nghĩa là hơn nửa số lượt gọi LLM không sinh cạnh nào; (5) **Entity Resolution chuyển sang HNSW + blocking** thay cho `IndexFlatIP` O(N²); (6) **gộp round-trip Cypher** — BFS hiện gọi 2 truy vấn cho mỗi node, chính là nguyên nhân latency 24s ở cross-doc.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng          | Module | Hàm / Khối code cụ thể                                             | Quan sát thực tế & Đánh giá                                                                                                                                                              |
| ---------------------------------- | ------ | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Conservative Coreference**       | M1     | `resolve_coref_batch()`, `COREF_TRIGGER`                           | Prompt conservative **chưa đủ**: 153/394 chunk bị sửa, 7 chunk mất >20% nội dung. Cần ràng buộc cứng ở tầng code.                                                                        |
| **Schema & Allowlist Guard**       | M2     | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `_validate_relations()` | Chặn được quan hệ lạ nhưng **không chặn được quan hệ đúng schema mà sai ngữ nghĩa** (`Microsoft -LEADS-> cloud computing`). Cần ràng buộc theo cặp (source_type, relation, target_type). |
| **Bulk Cypher Ingestion**          | M2     | `bulk_insert_nodes()`, `bulk_insert_edges()`                       | `UNWIND` batch 1.000 nạp 215 node + 181 cạnh trong **~2 giây** — phần nhanh nhất pipeline, hoàn toàn không phải bottleneck.                                                              |
| **Entity Resolution & Union-Find** | M3     | `build_resolution_map()`, `UF`, `merge_guard()`, `alias_lookup()`  | Kiến trúc AND khiến Guard chỉ được từ chối, không được cứu. Phải thêm nhánh `MERGE_LEXICAL`.                                                                                             |
| **Super-node Degree Cap**          | M4     | `retrieve_graph_context()`, `test_supernode_cap_enforced()`        | **Chưa từng kích hoạt** (max degree 19 < 100). Phải viết kiểm thử từng thành phần thay vì báo cáo suông.                                                                                 |
| **LLM-as-a-Judge Evaluation**      | M5     | `judge_answer()`, `judge_json()`                                   | Judge chấm **chuỗi rỗng là 1/5** y như câu trả lời sai — che mất một lỗi kỹ thuật thật.                                                                                                  |

---

### 2. Quá trình Debugging & Bài học

- **Lỗi kỹ thuật phức tạp nhất gặp phải:** 48% batch coref hỏng (`COREF_BATCH_FAILED`), mỗi batch phình từ 4s lên **487s**. Phải đi qua **3 giả thuyết sai** (mạng chậm → prompt truncation → rate limit thường) mới tới nguyên nhân thật. Và nguyên nhân trực tiếp lại là **bản vá do chính tôi viết**: token-bucket tự ước lượng token, trong nhánh `except` vẫn cộng token ước lượng vào bucket → sau 5 lần retry bucket tự cộng ~8.000 token = đúng bằng hạn mức → **cơ chế bảo vệ tự khoá chính nó**.

- **Cách xử lý thành công:** bỏ hẳn việc đoán, đọc trực tiếp header `x-ratelimit-remaining-tokens` từ server qua `with_raw_response`. Nhưng vẫn 429 dù header báo còn đủ quota — chỉ khi đọc **body** của lỗi mới thấy `on tokens per day (TPD): Limit 200000, Used 199874`. **Trần thật là 200.000 token/NGÀY, toàn bộ header `x-ratelimit-*` chỉ nói về cửa sổ PHÚT.** Kết quả sau khi sửa: coref 400/400 chunk, **0 batch lỗi**.

- **Ba bài học:**
  1. **Đừng suy đoán trạng thái mà đối tác đã công bố.**
  2. **Một cơ chế bảo vệ viết sai còn tệ hơn không có** — trước khi có rate limiter hệ thống chỉ _chậm_, sau khi có nó _hỏng_.
  3. **Giới hạn thật thường không nằm ở chỗ ta đang giám sát.**

- **Bài học phụ:** (a) không checkpoint = mất trắng 58 phút coref; sau khi có checkpoint, chạy lại Phase 1 chỉ mất **70 giây**; (b) **JSON mode đảm bảo cú pháp, không đảm bảo schema** — `gpt-oss-20b` chèn chuỗi rỗng `""` xen giữa các object, làm mất 34% chunk vì `try/except` bọc cả batch; (c) **lỗi im lặng nguy hiểm hơn lỗi ồn ào** — nếu chỉ nhìn bảng điểm mà không mở từng câu trả lời ra đọc, tôi đã kết luận sai rằng "GraphRAG kém hơn ở nhóm factoid".

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

> **Đã hoàn thiện theo định hướng áp dụng vào hệ thống RAG tra cứu và quản lý văn bản pháp luật.** Các quyết định về GraphRAG/Hybrid RAG, provenance, Entity Resolution và super-node được giữ nhất quán với các phát hiện thực nghiệm ở Phần 1.

- **Tên đồ án / Dự án:** **Hệ thống RAG hỗ trợ tra cứu và quản lý văn bản pháp luật**
- **Đặc thù bài toán & Lý do chọn giải pháp:** **Hệ thống cần truy hồi văn bản pháp luật theo nội dung, thời gian và quan hệ giữa các văn bản; đặc biệt có các trường hợp văn bản bổ sung, sửa đổi, thay thế hoặc liên quan đến một văn bản khác. Vì vậy, Flat/Hybrid RAG phù hợp cho các câu hỏi factoid và truy hồi đoạn văn, còn GraphRAG có giá trị khi câu hỏi cần nối nhiều thực thể và quan hệ giữa các văn bản. Giải pháp ưu tiên là kiến trúc Hybrid/GraphRAG theo loại câu hỏi thay vì mặc định dùng GraphRAG cho mọi truy vấn.** Bằng chứng từ lab: GraphRAG hoà ở 2/4 câu; toàn bộ +2,0 điểm đến từ 2 câu Flat RAG trượt hoàn toàn.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: **Document/LegalDocument, Article/Clause, Organization/Authority, Topic/LegalDomain, Person/IssuingAuthority**
  - Relations: **AMENDS, SUPPLEMENTS, REPLACES, REFERENCES, ISSUED_BY, APPLIES_TO, CONTAINS, RELATED_TO**
  - _Nguyên tắc:_ allowlist chặt từ đầu; ràng buộc theo cặp `(source_type, relation, target_type)`; provenance bắt buộc trên mọi cạnh.
- **Chiến lược xử lý Super-node & Entity Resolution:** **Khử trùng lặp theo bộ `(source, relation, target)` trước khi áp dụng giới hạn degree; với các node có degree cao, ưu tiên phân bổ quota theo loại quan hệ và giữ provenance để không làm mất khả năng truy vết. Entity Resolution dùng **ba nhánh song song**: lexical exact/normalized, vector similarity và manual/semantic review; không nối các tầng bằng AND thuần túy để một tầng lexical có thể cứu trường hợp embedding bỏ sót. Với thực thể pháp luật, áp dụng luật riêng cho số hiệu văn bản, cơ quan ban hành và tên văn bản; khác số hiệu hoặc khác định danh chính thức thì không tự động gộp. Mọi cạnh phải có `source_chunk_id`, `published_date` và evidence/provenance.** — lưu ý: khử trùng lặp `(source, relation, target)` **trước** khi cắt tỉa (lab này có 10/19 cạnh Microsoft trùng nhau).

---

## 🎯 TỰ ĐÁNH GIÁ

| Tiêu chí                             | Điểm tự chấm (1–5) | Ghi chú                                                                                                                                              |
| ------------------------------------ | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mức độ hiểu bài giảng GraphRAG       | 4                  | Nắm được vì sao GraphRAG thắng và quan trọng hơn là **khi nào nó không thắng** (hoà ở 2/4 câu).                                                      |
| Khả năng kiểm soát AI Coding Agent   | 4                  | Từ chối 4 đề xuất có hại; trừ điểm vì có 1 đề xuất **đã áp dụng và đã sai** (tự ước lượng token).                                                    |
| Chất lượng đồ thị tri thức xây dựng  | 3                  | 215 node/181 cạnh, **0 cạnh thiếu provenance**; trừ điểm vì đồ thị thưa (0,46 triple/chunk), chưa có super-node thật, còn lọt quan hệ sai ngữ nghĩa. |
| Khả năng phân tích và debug hệ thống | 4                  | Truy vết 11 ca lỗi tới nguyên nhân gốc rễ, đều tái lập được; điểm mạnh là **không dừng ở triệu chứng đầu tiên**.                                     |
