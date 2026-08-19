# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19

**Học viên:** Trần Công Đức
**Khoá:** AICB-K34 · Track 3: GraphRAG
**Ngày:** 19/08/2026

---

## 1. Mapping bài giảng vào code

| Khái niệm trong bài giảng | Module | Hàm / khối code | Quan sát thực tế & đánh giá |
|---|---|---|---|
| **Conservative Coreference** | M1 | `resolve_coref_batch()`, `COREF_SYSTEM`, `COREF_TRIGGER` | Prompt conservative **chưa đủ**. 153/394 chunk bị sửa, trong đó 7 chunk **mất >20% nội dung** do model tự xoá mệnh đề. Cần thêm ràng buộc cứng ở tầng code (từ chối nếu độ dài giảm quá ngưỡng), không thể chỉ tin vào prompt. |
| **Schema & Allowlist Guard** | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `_validate_relations()` | Allowlist làm tốt việc chặn quan hệ lạ, nhưng **không chặn được quan hệ đúng schema mà sai ngữ nghĩa** — ví dụ `Microsoft -LEADS-> cloud computing` (LEADS lẽ ra là Person→Company). Cần thêm ràng buộc cặp (source_type, relation, target_type). |
| **Bulk Cypher Ingestion** | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()`, `batches()` | `UNWIND $rows` batch 1.000 nạp 215 node + 181 cạnh trong **~2 giây**. Đây là phần **nhanh nhất** toàn pipeline — hoàn toàn không phải bottleneck như tôi tưởng ban đầu. |
| **Entity Resolution & Union-Find** | M3 | `build_resolution_map()`, `UF`, `merge_guard()`, `alias_lookup()` | Kiến trúc AND hai tầng khiến Lexical Guard **chỉ được từ chối, không được cứu**. Phải thêm nhánh `MERGE_LEXICAL` mới thu hồi được 6 ca gộp bị bỏ sót. |
| **Super-node Degree Cap** | M4 | `retrieve_graph_context()`, `SUPER_NODE_DEGREE`, `GLOBAL_EDGE_CAP` | **Chưa từng kích hoạt** — degree cao nhất 19 so với ngưỡng 100. Phải viết `test_supernode_cap_enforced()` kiểm chứng từng thành phần thay vì báo cáo suông. |
| **Hybrid Retrieval & Linearization** | M4 | `retrieve_graph_context()`, `textualize()`, `answer_graph_rag()` | Linearize kèm provenance là điểm mạnh nhất: câu trả lời trích dẫn được tới **từng cạnh**, không chỉ tới chunk. |
| **LLM-as-a-Judge** | M5 | `judge_answer()`, `judge_json()` | Judge chấm **chuỗi rỗng là 1/5** y như câu trả lời sai — che mất một lỗi kỹ thuật thật. Judge cần phân biệt "trả lời sai" với "không sinh được output". |
| **Golden Dataset 3 nhóm** | M5 | `mine_golden()`, `validate_golden()` | Ràng buộc **2 cạnh phải ở 2 chunk khác nhau** cho câu multi-hop là thiết kế quan trọng nhất — không có nó thì Flat RAG vẫn trả lời được và benchmark mất ý nghĩa. |
| **Community Detection** (bonus) | Bonus | `build_communities()` | 53 cụm trên 215 node. Các cụm có ngữ nghĩa rõ: cụm SpaceX (Starlink, Crew Dragon, Polaris Dawn), cụm Adobe (Firefly, Illustrator, John Warnock, Figma). |

---

## 2. Quá trình debugging & bài học

### Lỗi khó nhất: 48% batch coref hỏng — và thủ phạm là chính tôi

**Triệu chứng:** sau ~25 batch, mỗi batch coref phình từ 4s lên **487s**, ETA cả vòng lên gần **7 giờ**. Rồi 60/125 chunk bị đánh dấu `COREF_BATCH_FAILED`.

**Đường đi sai lầm:**

1. **Giả thuyết đầu — mạng chậm.** Sai. Probe một call đơn lẻ chỉ mất 1,5s.
2. **Giả thuyết hai — prompt quá dài gây truncation.** Sai. Tái hiện một batch thật: `finish_reason = stop`, JSON parse OK, 1.204 token.
3. **Giả thuyết ba — rate limit.** Đúng một nửa. Tôi thêm token-bucket tự ước lượng token để chủ động chờ thay vì ăn 429.
4. **Và chính bản vá đó tạo ra lỗi tệ hơn.** Trong nhánh `except` tôi vẫn cộng số token *ước lượng* vào bucket:
   ```python
   except Exception as e:
       _tpm_record(est)     # call hỏng vẫn cộng ~1.600 token
   ```
   Sau 5 lần retry, bucket cục bộ tự cộng ~8.000 token = **đúng bằng hạn mức**, tự báo "hết quota" dù server chưa hề tính. Mỗi attempt phải ngủ 45s, hết 5 attempt là batch chết. **Cơ chế bảo vệ tự khoá chính nó.**
5. **Sửa đúng:** bỏ hẳn việc đoán, đọc trực tiếp từ server qua `with_raw_response` để lấy header `x-ratelimit-remaining-tokens`.
6. **Nhưng vẫn 429** — dù header báo còn đủ 8.000 token. Chỉ khi đọc **body** của lỗi mới thấy sự thật:
   ```
   on tokens per day (TPD): Limit 200000, Used 199874
   ```
   **Trần thật là 200.000 token/NGÀY, còn toàn bộ header `x-ratelimit-*` chỉ nói về cửa sổ PHÚT.**

**Kết quả:** coref 400/400 chunk, **0 batch lỗi**.

### Ba bài học tôi thực sự rút ra

**1. Đừng suy đoán trạng thái mà đối tác đã công bố.** Tôi tự ước lượng token trong khi server trả về con số chính xác trong mỗi response. Ước lượng sai lệch tích luỹ đủ để phá hỏng hệ thống.

**2. Một cơ chế bảo vệ viết sai còn tệ hơn không có cơ chế nào.** Trước khi có rate limiter, hệ thống chỉ *chậm*. Sau khi có, nó *hỏng*. Lỗi tạm thời bị biến thành lỗi vĩnh viễn.

**3. Giới hạn thật thường không nằm ở chỗ ta đang giám sát.** Tôi giám sát đúng thứ được hiển thị (header phút) và bỏ lỡ hoàn toàn thứ đang thực sự chặn mình (trần ngày) — cho tới khi chịu đọc nguyên văn thông báo lỗi thay vì chỉ đọc mã lỗi.

### Bài học phụ, nhưng đắt không kém

- **Không checkpoint = mất trắng.** Tiến trình bị kill giữa chừng, **58 phút coref bay sạch** vì kết quả chỉ nằm trong bộ nhớ kernel. Sau khi thêm checkpoint JSONL theo từng batch, chạy lại toàn bộ Phase 1 chỉ mất **70 giây** — chính điều này mới khiến 5 lần chạy lại tiếp theo để sửa lỗi trở nên khả thi.
- **JSON mode đảm bảo cú pháp, không đảm bảo schema.** `gpt-oss-20b` chèn chuỗi rỗng `""` xen giữa các object hợp lệ trong mảng. Vì `try/except` bọc cả batch nên **4 chunk chết chung** dù 3 chunk có dữ liệu tốt → mất 34% chunk. Thu hẹp phạm vi bắt lỗi xuống từng chunk và kiểm tra `isinstance` ở mọi tầng lồng nhau đưa tỷ lệ lỗi về **2%**.
- **Lỗi im lặng nguy hiểm hơn lỗi ồn ào.** Câu G01 trả về *"I couldn't find any mention…"* — đọc qua thì tưởng GraphRAG thành thật thừa nhận thiếu dữ liệu. Thực tế đồ thị có sẵn 10 cạnh đúng, và thủ phạm là seed extractor trả về `GitHub` thay vì `Microsoft`. Nếu chỉ nhìn bảng điểm mà không mở từng câu trả lời ra đọc, tôi đã kết luận sai rằng "GraphRAG kém hơn ở nhóm factoid".

### Về việc kiểm soát AI Coding Agent

Agent đề xuất đổi sang `groq/compound-mini` để chạy nhanh gấp 8,75 lần. Tôi từ chối vì model đó tự bật web search — sẽ kéo thông tin ngoài context vào câu trả lời và **làm hỏng chính thứ benchmark này đo (Faithfulness)**. Điểm số sẽ đẹp hơn nhưng vô nghĩa. Đánh đổi 75 phút để giữ kết quả đo trung thực là xứng đáng.

Bài học lớn hơn: **đề xuất nghe hợp lý nhất lại thường là đề xuất nguy hiểm nhất**, vì nó tối ưu đúng chỉ số mình đang nhìn (thời gian) và phá hỏng chỉ số mình không nhìn (tính trung thực của phép đo).

---

## 3. Kế hoạch áp dụng vào đồ án thực tế (Action Plan)

> ⚠️ **[CẦN ĐIỀN]** — Phần này phụ thuộc vào đồ án cụ thể của bạn. Khung dưới đây kèm sẵn các câu hỏi định hướng và tiêu chí quyết định rút ra từ chính lab này.

### 3.1 Tên đồ án / dự án

**[CẦN ĐIỀN]**

### 3.2 Bài toán của bạn có thực sự cần GraphRAG không?

**[CẦN ĐIỀN]** — dùng bộ tiêu chí sau để tự trả lời, vì lab này cho thấy GraphRAG **không** cải thiện đều:

| Dấu hiệu **CẦN** GraphRAG | Dấu hiệu **Flat/Hybrid RAG là đủ** |
|---|---|
| Câu hỏi thường phải nối ≥2 sự kiện nằm ở các tài liệu khác nhau | Câu trả lời hầu như luôn nằm gọn trong 1 đoạn văn |
| Cùng một thực thể xuất hiện dưới nhiều biến thể tên | Tên thực thể chuẩn hoá sẵn (mã sản phẩm, mã nhân viên) |
| Cần truy vết nguồn tới mức **từng khẳng định** | Trích dẫn tới mức tài liệu là đủ |
| Quan hệ thay đổi theo thời gian, cần biết "tại thời điểm nào" | Dữ liệu tĩnh |
| Corpus tương đối ổn định, số truy vấn lớn | Corpus thay đổi liên tục, truy vấn thưa |

**Bằng chứng từ lab để cân nhắc:** trên 4 câu đã chấm, GraphRAG **hoà** với Flat RAG ở 2 câu; toàn bộ khoảng cách +2,0 điểm đến từ 2 câu mà Flat RAG **trượt hoàn toàn**. Giá trị của GraphRAG là **giảm tỷ lệ thất bại thảm hoạ**, không phải nâng điểm trung bình. Nếu bài toán của bạn không có loại câu hỏi mà Flat RAG trượt sạch, chi phí dựng đồ thị (~75× thời gian index) sẽ không hoàn vốn.

### 3.3 Thiết kế Node / Relation dự kiến

**[CẦN ĐIỀN]**

```
Node types:      ...
Relation types:  ...
```

Nguyên tắc rút ra từ lab:
- **Allowlist chặt ngay từ đầu.** 8 loại quan hệ là đủ cho tin công nghệ. Schema mở sẽ sinh ra hàng trăm biến thể quan hệ không gộp được.
- **Ràng buộc theo cặp `(source_type, relation, target_type)`**, đừng chỉ liệt kê danh sách phẳng — lab này để lọt `Microsoft -LEADS-> cloud computing` vì allowlist không ràng buộc loại hai đầu.
- **Provenance là bắt buộc trên mọi cạnh**, không phải tuỳ chọn: `source_chunk_id`, `published_date`, `evidence`, `confidence`. Kiểm tra `invalid_provenance_edges == 0` nên là test tự động chạy mỗi lần nạp.

### 3.4 Chiến lược Entity Resolution

**[CẦN ĐIỀN]** — áp dụng các bài học cụ thể sau:

- **Ba nhánh gộp song song**, không nối AND: (a) alias thủ công, (b) trùng khớp từ vựng sau khi bỏ hậu tố, (c) vector ANN. Lab này ban đầu chỉ có (a) AND (c), khiến `Adobe Inc.` không gộp được với `Adobe` vì cosine chỉ 0,814.
- **Luật riêng theo loại thực thể.** Với tên người, ratio ký tự **không dùng được**: `Sam Altman` vs `Steve Altman` đạt 0,727 (vượt ngưỡng 0,72), `Tim Cook` vs `Tim Cooke` vượt **cả hai** tầng. Với tên sản phẩm, thêm luật *khác chữ số ⇒ không gộp* (`RTX 4090` ≠ `RTX 4070`).
- **Bảng audit phải ghi cả ca bị từ chối**, với ngưỡng ghi thấp hơn ngưỡng gộp. Chỉ ghi ca đã gộp thì không còn là audit — lab này ban đầu chỉ có 6 dòng toàn MERGE, không soi được gì.
- **Chuẩn hoá dấu câu cẩn thận.** Một dấu chấm sót lại (`"microsoft corp."` vs khoá `"microsoft corp"`) đã chia đôi node trung tâm nhất của đồ thị.

### 3.5 Chiến lược xử lý Super-node

**[CẦN ĐIỀN]** — lưu ý từ lab:

- **Khử trùng lặp `(source, relation, target)` TRƯỚC khi cắt tỉa.** Trong lab, 10/19 cạnh của Microsoft đều là cùng một quan hệ `ACQUIRED → Activision Blizzard` lặp lại từ 10 bài báo. Nếu cắt lấy 50 cạnh mới nhất, một sự kiện được đưa tin dày đặc sẽ **chiếm trọn hạn ngạch** và đẩy mọi quan hệ khác ra ngoài — "cắt theo thời gian" biến thành "cắt theo độ nóng của tin tức".
- **Phân bổ hạn ngạch theo từng loại quan hệ** thay vì lấy N cạnh mới nhất trên toàn node.
- **Đừng đối xử mọi quan hệ như nhau theo thời gian:** `FOUNDED` là sự kiện một lần có giá trị vĩnh viễn, `PARTNERED_WITH` là trạng thái có thể hết hiệu lực.

### 3.6 Kế hoạch xử lý bottleneck

**[CẦN ĐIỀN]** — số liệu tham chiếu từ lab (xem chi tiết ở [`technical_defense.md`](technical_defense.md) Câu 9):

| Thành phần | 400 chunk | Ngoại suy 100.000 bài |
|---|---|---|
| **LLM extraction** | **~31 phút** | **~130 giờ** 🔴 |
| Embedding | ~60 giây | ~1,1 giờ |
| Neo4j bulk insert | ~2 giây | ~10 phút |

Bottleneck lệch **hai bậc độ lớn** so với mọi thứ khác, và nó nằm ở **hạn mức LLM**, không nằm ở database.

---

## 4. Tự đánh giá

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|---|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Nắm được vì sao GraphRAG thắng (vật chất hoá quan hệ thành cạnh) và quan trọng hơn là **khi nào nó không thắng** — hoà ở 2/4 câu. |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối 4 đề xuất có hại, trong đó đề xuất nguy hiểm nhất (`compound-mini`) lại là đề xuất hợp lý nhất về mặt hiệu năng. Trừ điểm vì có 1 đề xuất tôi **đã áp dụng và đã sai** (tự ước lượng token). |
| Chất lượng đồ thị tri thức | 3 | 215 node / 181 cạnh, **0 cạnh thiếu provenance**. Trừ điểm vì đồ thị còn thưa (0,46 triple/chunk), chưa có super-node thật, và còn lọt quan hệ sai ngữ nghĩa như `Microsoft -LEADS-> cloud computing`. |
| Khả năng phân tích và debug hệ thống | 4 | Truy vết được 11 ca lỗi tới nguyên nhân gốc rễ, đều có bằng chứng tái lập. Điểm mạnh nhất là **không dừng ở triệu chứng đầu tiên** — vụ 429 phải đi qua 3 giả thuyết sai mới tới trần token/ngày. |
