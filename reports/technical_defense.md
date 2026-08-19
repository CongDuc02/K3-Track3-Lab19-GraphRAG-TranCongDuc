# Thuyết Minh Kỹ Thuật — Lab 19: Production-Grade GraphRAG vs Flat RAG

**Học viên:** Trần Công Đức
**Khoá:** K3 · Track 3: GraphRAG
**Ngày:** 19/08/2026
**Notebook:** `Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb`

---

## Thông số hệ thống đã xây dựng

| Hạng mục | Giá trị |
|---|---|
| Corpus gốc | 514.417 bài (HackerNoon tech-company-news-data-dump, 300 MB) |
| Sau lọc miền công nghệ | 22.947 bài (4,5%) → dedup còn 19.378 |
| Đưa vào pipeline | 1.500 bài → **1.506 chunk** |
| Chunk qua NER+RE | **400** (`EXTRACTION_MAX_CHUNKS`) |
| Triple trích xuất | **182** |
| **Node / Edge trong Neo4j** | **215 / 181** |
| Phân bố node | Company 118 · Technology 75 · Person 22 |
| **Cạnh thiếu provenance** | **0** ✅ |
| Bảng audit Entity Resolution | **21 dòng** (4 nhóm quyết định) |
| Community (bonus) | 53 cụm / 215 node |
| Model | Coref `gpt-oss-safeguard-20b` · NER-RE/Gen/Judge `gpt-oss-20b` |

---

## Câu 1 — Coreference Resolution phân giải sai

> *Nêu ít nhất 1 tình huống cụ thể trong dữ liệu mà cơ chế Coreference Resolution phân giải sai. Hậu quả với Knowledge Graph là gì?*

**Định lượng trước:** trong 394 chunk đối chiếu được với văn bản gốc, **153 chunk (38,8%) bị bước coref sửa đổi**.

### Ca sai điển hình — `f9f06b416e5553944d89::c0000`

```
GỐC : Adobe Express Gets Generative AI for Flashy Fliers Social Videos.
      The company's all-purpose creation app gets new video editing abilities…

SAU : Adobe Express Gets Generative AI for Flashy Fliers Social Videos.
      Adobe Express's all-purpose creation app gets new video editing abilities…
```

**Sai ở đâu:** `The company` trỏ tới **Adobe** — *công ty*. Model lại phân giải thành **Adobe Express** — *sản phẩm* của công ty đó. Tiền ngữ gần nhất về mặt vị trí bị chọn thay vì tiền ngữ đúng về mặt ngữ nghĩa.

**Hậu quả với Knowledge Graph:** bước NER+RE phía sau đọc văn bản đã sửa và sinh ra quan hệ gán cho **sai chủ thể**: `Adobe Express -DEVELOPED-> ...` thay vì `Adobe -DEVELOPED-> Adobe Express`. Ba thiệt hại cùng lúc:
1. **Node giả:** `Adobe Express` bị nâng cấp từ *sản phẩm* thành *chủ thể hành động*.
2. **Phân mảnh:** cạnh lẽ ra thuộc về node `Adobe` (degree 6) bị tách sang node khác → đường multi-hop đi qua Adobe bị đứt.
3. **Không thể phát hiện bằng provenance:** cạnh vẫn có đủ `source_chunk_id` và `evidence`, chỉ là evidence trích từ **văn bản đã bị sửa sai**. Provenance chứng minh được *nguồn gốc*, không chứng minh được *tính đúng*.

### Ca sai thứ hai — thay đại từ bằng cụm mô tả, không phải tên thực thể

```
GỐC : This new platform offers enhanced document collaboration…
SAU : The powerful alternative to Apple's iWork and Microsoft Office offers…
```
Model lấy cụm mô tả trong tiêu đề làm "tiền ngữ". NER+RE sau đó sẽ cố trích một thực thể tên `The powerful alternative to Apple's iWork and Microsoft Office` — rác thuần tuý.

### Ca nghiêm trọng nhất — mất nội dung âm thầm

| Mức độ | Số chunk |
|---|---|
| Bị sửa đổi | 153/394 (38,8%) |
| Ngắn đi >5% | 11/394 |
| **Ngắn đi >20%** | **7/394** |

Tệ nhất là `282123176a4882dbf10f::c0000` — **chỉ còn 31,5%** văn bản gốc. Model tự ý xoá nguyên mệnh đề `"ties with the government of Syrian President Bashar al-Assad. Last mon…"`. Đây là bước lẽ ra **chỉ được thay đại từ**, tuyệt đối không được xoá nội dung. Mọi quan hệ nằm trong phần bị xoá đều biến mất khỏi đồ thị mà không để lại dấu vết.

### Mặt được của thiết kế conservative

28/400 chunk giữ nguyên `unresolved_mentions` (`us`, `you`, `it's`) thay vì đoán bừa — đúng như quy tắc an toàn yêu cầu. Nhưng các ca trên cho thấy **prompt conservative là chưa đủ**: cần thêm ràng buộc cứng ở tầng code, ví dụ từ chối kết quả nếu `len(resolved) < 0.9 × len(original)`, và chỉ chấp nhận thay thế bằng chuỗi đã xuất hiện nguyên văn trong chunk.

---

## Câu 2 — Ngưỡng Entity Resolution & Lexical Guard

> *Ngưỡng cosine similarity là bao nhiêu? Trích dẫn 1 cặp có tương đồng vector cao (>0.85) nhưng bị Lexical Guard chặn, và giải thích lý do.*

**Ngưỡng đang dùng:**
- Vector ANN (FAISS `IndexFlatIP`, cosine): **≥ 0.90** để gộp
- Lexical Guard (`SequenceMatcher` sau khi bỏ hậu tố pháp nhân): **≥ 0.72**
- Ngưỡng **ghi audit**: 0.80 — thấp hơn ngưỡng gộp để nhìn được vùng xám

### Trả lời thẳng: **không có cặp nào bị Lexical Guard chặn trong corpus này**

Bảng audit 21 dòng có 4 nhóm, **0 dòng `REJECT_GUARD`**:

| Quyết định | Số dòng |
|---|---|
| `REJECT_THRESHOLD` | 8 |
| `MERGE_LEXICAL` | 6 |
| `MERGE_MANUAL` | 4 |
| `MERGE_VECTOR` | 3 |
| **`REJECT_GUARD`** | **0** |

Lý do: mọi cặp đạt cosine ≥ 0.90 đều thực sự là biến thể của cùng một thực thể, nên guard không phải phủ quyết lần nào. Tôi không bịa ra một ca để lấp chỗ trống.

### Nhưng dữ liệu thật cho một kết luận đáng giá hơn

Đo **riêng từng tầng** trên các cặp nguy hiểm:

| Cặp | cosine | lexical ratio | Qua vector? | Qua guard? | Kết cục |
|---|---|---|---|---|---|
| `Sam Altman` / `Steve Altman` | 0,824 | **0,727** | ❌ | ✅ | Thoát nhờ **tầng vector** |
| `RTX 4090` / `RTX 4070` | 0,803 | **0,875** | ❌ | ✅ | Thoát nhờ **tầng vector** |
| `ChatGPT` / `ChatGPT API` | 0,836 | **0,778** | ❌ | ✅ | Thoát nhờ **tầng vector** |
| `Apple` / `Apple Watch` | 0,609 | 0,625 | ❌ | ❌ | Cả hai tầng cùng chặn |
| **`Tim Cook` / `Tim Cooke`** | **0,902** | ~0,94 | ✅ | ✅ | 🔴 **SẼ GỘP NHẦM** |

**Kết luận 1 — ngưỡng 0.72 trượt chính ví dụ mà đề bài nêu đích danh.** `Sam Altman` vs `Steve Altman` đạt ratio **0,727**, vượt ngưỡng đúng **0,007**. Nếu chỉ dựa vào Lexical Guard thì đã gộp nhầm hai người khác nhau. Cặp này sống sót **hoàn toàn nhờ tầng vector** (0,824 < 0,90) — tức guard đã thất bại đúng ở ca nó sinh ra để chặn.

**Kết luận 2 — vẫn còn lỗ thủng.** `Tim Cook` / `Tim Cooke` vượt **cả hai** tầng. Với tên người sai một ký tự, embedding vẫn cho ~0,90 và ratio ký tự thì gần như tuyệt đối. Corpus hiện tại chưa chứa cặp này nên chưa gây hại, nhưng ở quy mô lớn hơn chắc chắn sẽ gặp.

**Đề xuất:** với `type == Person`, bỏ ratio ký tự, yêu cầu **khớp token-level tuyệt đối cho họ và tên** — sai một chữ cái trong tên người thường là *người khác*, không phải biến thể viết. Với `type == Technology`, thêm luật **khác chữ số ⇒ không gộp**.

### Một khiếm khuyết kiến trúc đã phát hiện và sửa

Hai tầng nối bằng **AND** nên Lexical Guard chỉ có quyền *từ chối*, không có quyền *cứu*:

| Cặp | cosine | Lẽ ra | Thực tế ban đầu |
|---|---|---|---|
| `Adobe Inc.` / `Adobe` | 0,814 | gộp | ❌ bị loại ở tầng vector |
| `Intel Corp` / `Intel` | 0,815 | gộp | ❌ bị loại ở tầng vector |

`all-MiniLM-L6-v2` xử lý hậu tố pháp nhân kém — thêm "Inc." làm cosine tụt xuống ~0,81. Trong khi `strip_suffix()` đã có sẵn logic đúng nhưng không bao giờ được gọi tới.

**Đã sửa:** thêm nhánh `MERGE_LEXICAL` chạy song song — nếu sau khi bỏ hậu tố pháp nhân mà tên **trùng khớp tuyệt đối** thì gộp, không phụ thuộc điểm vector. Bảng audit từ 6 → 21 dòng, thu hồi 6 ca gộp bị bỏ sót.

**Một lỗi một ký tự:** `norm_entity()` giữ lại dấu chấm nên `"Microsoft Corp."` → `"microsoft corp."`, trong khi `MANUAL_ALIASES` khai báo khoá `"microsoft corp"`. Lookup trượt → đồ thị có **đồng thời** node `Microsoft` (degree 19) và `Microsoft Corp.` tách rời. Đã sửa bằng cách thử nhiều biến thể khi tra alias.

---

## Câu 3 — Super-node: Top 3 và chính sách cắt tỉa theo thời gian

> *Top 3 thực thể có degree cao nhất? Ưu điểm và rủi ro của việc ưu tiên 50 cạnh mới nhất?*

### Top thực thể theo degree

| Hạng | Thực thể | Loại | Degree |
|---|---|---|---|
| 1 | **Microsoft** | Company | **19** |
| 2 | **Apple** | Company | **18** |
| 3 | **Activision Blizzard** | Company | **15** |
| 4 | Nvidia | Company | 13 |
| 5 | EU content rules | Technology | 7 |
| 6 | Meta | Company | 7 |

### Sự thật phải nói rõ: ở quy mô này **không có super-node nào**

Degree cao nhất là **19**, ngưỡng `SUPER_NODE_DEGREE` là **100**. Nhánh cắt tỉa **không bao giờ chạy** trong toàn bộ lần đánh giá — cột `graph_supernode_events` bằng **0 ở cả 6 câu hỏi**. Hàm `test_supernode_policy()` của đề chạy qua nhưng không chứng minh được gì ngoài việc đồ thị còn nhỏ.

Nên tôi bổ sung `test_supernode_cap_enforced()` kiểm chứng **từng thành phần** của chính sách trên dữ liệu thật:

1. **`recent_edges` tôn trọng LIMIT:** gọi với cap = 3/5/10/50 → số cạnh trả về luôn ≤ cap.
2. **Cắt đúng cạnh mới nhất:** 5 cạnh giữ lại khớp chính xác 5 `published_date` lớn nhất trong toàn bộ cạnh của node.
3. **Công thức chọn limit:** `degree=19 → limit 50` (không cắt); `degree=101 hoặc 5000 → limit 50 = SUPER_NODE_EDGE_CAP` (cắt).

### Ưu điểm của temporal mitigation

- **Chặn bùng nổ context:** một node degree 5.000 mà không cắt sẽ sinh ~5.000 dòng linearize ≈ 500K ký tự, vượt xa `MAX_GRAPH_CONTEXT_CHARS = 14000` và vượt cửa sổ ngữ cảnh.
- **Chi phí dự đoán được:** trần cứng `GLOBAL_EDGE_CAP = 250` giữ token của GraphRAG ở mức so sánh được với Flat RAG.
- **Hợp với miền tin tức:** với tin công nghệ, quan hệ mới thường là quan hệ đang có hiệu lực (thương vụ đã hoàn tất, đối tác hiện hành).

### Rủi ro — và rủi ro này là thật, không lý thuyết

- **Mù lịch sử:** câu hỏi *"Microsoft đã mua ai năm 2014?"* sẽ trượt nếu 50 cạnh mới nhất đều thuộc 2023. Ngay corpus này đã lộ dấu hiệu: **10/19 cạnh của Microsoft đều là `ACQUIRED → Activision Blizzard`** lặp lại từ 10 bài báo khác nhau. Ở quy mô lớn, một sự kiện được đưa tin dày đặc sẽ **chiếm trọn hạn ngạch 50 cạnh** và đẩy các quan hệ khác ra ngoài — cắt theo thời gian biến thành *cắt theo độ nóng của tin tức*.
- **Thiên lệch recency:** thực thể mới được nhắc nhiều sẽ lấn át thực thể quan trọng nhưng ít tin.
- **Không đối xứng:** `ORDER BY published_date DESC` áp cho mọi loại quan hệ, trong khi `FOUNDED` (sự kiện một lần, giá trị vĩnh viễn) không nên bị đối xử như `PARTNERED_WITH` (trạng thái thay đổi).

**Đề xuất:** phân bổ hạn ngạch **theo từng loại quan hệ** (mỗi `relation` được N cạnh mới nhất) thay vì lấy 50 cạnh mới nhất trên toàn node; và **khử trùng lặp theo cặp (source, relation, target)** trước khi cắt — riêng việc này đã giảm 10 cạnh Microsoft→Activision Blizzard xuống còn 1, giải phóng chỗ cho 9 quan hệ khác.

---

## Câu 4 — Bảng so sánh Benchmark

> *Điền bảng so sánh Quality vs Latency vs Token usage.*

**Trạng thái:** 4/6 câu đã chấm xong (G01–G04). Hai câu `cross-doc` (G05, G06) đang chờ hạn mức token/ngày của Groq hồi lại — xem Câu 10. Bảng dưới là số liệu **thật của 4 câu đã hoàn tất**, không ngoại suy.

### Điểm LLM-as-a-Judge theo từng câu (sau khi sửa lỗi)

| ID | Nhóm | Flat: Compr./Faith./Multi-hop | Graph: Compr./Faith./Multi-hop | Người thắng |
|---|---|---|---|---|
| G01 | factoid | 5 / 5 / 5 | 5 / 5 / 5 | Hoà |
| G02 | factoid | 1 / 1 / 1 | **5 / 5 / 5** | **GraphRAG** |
| G03 | multi-hop | 1 / 1 / 1 | **5 / 5 / 5** | **GraphRAG** |
| G04 | multi-hop | 5 / 5 / 5 | 5 / 5 / 5 | Hoà |
| **TB** | | **3,0 / 3,0 / 3,0** | **5,0 / 5,0 / 5,0** | **+2,0 điểm** |

### Latency & Token (đo từ lần chạy đầy đủ 6 câu, trước khi sửa)

| Nhóm | Metric | Flat RAG | GraphRAG | Nhận xét |
|---|---|---|---|---|
| factoid | Latency (s) | 0,67 | 0,66 | Tương đương |
| factoid | Token | 820 | 668 | GraphRAG **rẻ hơn** — context graph cô đọng hơn 6 chunk thô |
| multi-hop | Latency (s) | 28,0 | 24,9 | GraphRAG nhanh hơn |
| multi-hop | Token | 1.099 | 2.178 | GraphRAG đắt gấp **2×** |
| cross-doc | Latency (s) | 1,69 | 24,3 | GraphRAG chậm gấp **14×** |
| cross-doc | Token | 1.718 | 4.399 | GraphRAG đắt gấp **2,6×** |

### Đọc bảng này thế nào

- **Chất lượng:** GraphRAG hơn **+2,0/5 điểm** trên cả ba tiêu chí ở 4 câu đã chấm. Đáng chú ý là mức tăng đến từ **2 câu Flat RAG trượt hoàn toàn** (1/5), chứ không phải nhích đều — đúng bản chất: GraphRAG không làm câu dễ tốt hơn, nó **cứu những câu Flat RAG không thể trả lời**.
- **Token:** GraphRAG đắt hơn 2–2,6× ở multi-hop và cross-doc, nhưng **rẻ hơn** ở factoid. Chi phí tỷ lệ với kích thước subgraph chứ không cố định.
- **Latency:** khoảng chênh lớn nhất (14×) nằm ở cross-doc, và **không phải do LLM** mà do BFS thực hiện nhiều lượt round-trip Cypher riêng lẻ tới Neo4j AuraDB (mỗi node là 2 truy vấn: `node_degree` + `recent_edges`). Đây là vấn đề triển khai, không phải giới hạn bản chất của GraphRAG — gộp thành một truy vấn `apoc.path` hoặc một `UNWIND` cho cả frontier sẽ giảm mạnh.

---

## Câu 5 — Ca lỗi Flat RAG thất bại, GraphRAG thành công

> Chi tiết đầy đủ: [`failure_analysis.md`](failure_analysis.md) — Ca lỗi 1.

**G02 (factoid):** *"which company did Apple invest in?"* → đáp án `SoftBank's UK chip unit`

| | Điểm | Câu trả lời |
|---|---|---|
| Flat RAG | 1/5 | *"I couldn't find any mention… The available chunks focus on Apple's market-cap milestones"* |
| GraphRAG | **5/5** | *"Apple invested in SoftBank's UK chip unit【chunk_id=2a8d4bb2…】"* |

**Vì sao Flat RAG hỏng:** vector search xếp hạng theo độ giống ngữ nghĩa toàn chunk. Các chunk về vốn hoá thị trường Apple (`market cap`, `$3 trillion`, `investors`) trùng cả *chủ đề tài chính* lẫn *thực thể Apple* nên chiếm hết top-6, đẩy chunk chứa sự kiện đầu tư thật ra ngoài. Đây là **semantic drift**: đúng chủ đề, trượt sự kiện. Tăng `k` chỉ làm loãng context chứ không đảm bảo trúng.

**Vì sao GraphRAG được:** quan hệ đã được **vật chất hoá thành cạnh** `Apple -INVESTED_IN-> SoftBank's UK chip unit` kèm provenance. Truy hồi chuyển từ *"chunk nào trông giống câu hỏi"* sang *"Apple có những cạnh nào"* — không còn phụ thuộc thứ hạng tương đồng.

---

## Câu 6 — Ca lỗi GraphRAG thất bại

> Chi tiết đầy đủ: [`failure_analysis.md`](failure_analysis.md) — Ca lỗi 2.

**G01 (factoid):** *"which company did Microsoft acquire?"* — GraphRAG chấm **1/5** dù đồ thị có sẵn **10 cạnh** `Microsoft -ACQUIRED-> Activision Blizzard`.

**Nguyên nhân gốc rễ:** `extract_seeds()` trả về seed = **`GitHub`**, không phải `Microsoft`. LLM đã **dùng kiến thức tham số để đoán ĐÁP ÁN** (Microsoft từng mua GitHub ngoài đời thật) thay vì trích thực thể có trong câu hỏi — bất chấp system prompt ghi rõ `Do not answer the question.` Câu G02 tương tự: trả về `Mojang`.

`GitHub` không có trong đồ thị → `NO_SEED` → context graph rỗng → **GraphRAG âm thầm thoái hoá thành Flat RAG kém hơn** (k=4 thay vì k=6) mà không báo lỗi.

**Khắc phục 2 lớp:**
1. *Lexical guard:* chỉ giữ seed có tên thực sự xuất hiện trong câu hỏi.
2. *Lớp không dùng LLM:* `graph_seeds_from_text()` quét thẳng `name_norm` của node trong Neo4j — biến seed matching từ **tác vụ sinh** (có thể bịa) thành **tác vụ tra cứu** (không thể bịa).

**Kết quả:** G01 và G02 đều từ **1/5 → 5/5**.

---

## Câu 7 — Đánh đổi Quality vs Cost vs Latency

| Chiều | Flat RAG | GraphRAG | Ghi chú |
|---|---|---|---|
| Chi phí index ban đầu | Chỉ embedding 1.506 chunk (~1 phút CPU) | **+ 400 lượt gọi LLM** cho coref + NER/RE | GraphRAG đắt hơn hàng trăm lần ở khâu dựng |
| Thời gian dựng thực đo | ~1 phút | **~75 phút** (coref 11' + extraction 20' + phần còn lại) | Ở hạn mức 8.000 token/phút |
| Chi phí truy vấn | 1 lần embed + 1 lần sinh | **+1 lần trích seed + N lượt Cypher** | |
| Token/câu hỏi | 820–1.718 | 668–4.399 | GraphRAG rẻ hơn ở factoid, đắt hơn ở cross-doc |
| Latency | 0,7–28 s | 0,7–24 s | Chênh lớn nhất do round-trip Cypher, không do LLM |
| Chất lượng | 3,0/5 | **5,0/5** | Trên 4 câu đã chấm |
| Khả năng truy vết | Chỉ tới mức chunk | **Tới mức từng cạnh** (`source_chunk_id`, `published_date`, `evidence`, `confidence`) | Khác biệt về chất |

**Kết luận đánh đổi:** GraphRAG dịch chuyển chi phí từ **lúc truy vấn** sang **lúc dựng index**. Nếu corpus tĩnh và số truy vấn lớn thì đây là đánh đổi tốt. Nếu corpus thay đổi liên tục mà truy vấn thưa thì chi phí trích xuất lại không bao giờ hoàn vốn — khi đó Flat RAG hoặc Hybrid nhẹ là lựa chọn đúng.

**Điều quan trọng nhất về chất lượng:** GraphRAG **không cải thiện đều**. Ở 2/4 câu nó hoà với Flat RAG; toàn bộ khoảng cách +2,0 điểm đến từ 2 câu mà Flat RAG **trượt hoàn toàn** (1/5). Giá trị của GraphRAG nằm ở việc **giảm tỷ lệ thất bại thảm hoạ**, không phải nâng điểm trung bình.

---

## Câu 8 — Đề xuất của AI Coding Agent đã bị từ chối

Trong quá trình làm, các đề xuất sau đã bị bác:

**1. Đổi sang `groq/compound-mini` để chạy nhanh gấp 8,75 lần.** Model này có 70.000 token/phút thay vì 8.000, đủ để rút thời gian pipeline từ ~75 phút xuống ~10 phút.
**Từ chối vì:** compound tự bật **web search**. Nó sẽ kéo thông tin ngoài context vào câu trả lời và **làm hỏng chính thứ benchmark này đo — Faithfulness**. Điểm số sẽ đẹp hơn nhưng vô nghĩa. (Kiểm chứng sau đó còn cho thấy nó route ngầm về `gpt-oss-120b` nên dùng chung bucket quota đã cạn — nhưng lý do từ chối ban đầu vẫn là lý do đúng.)

**2. Giảm `EXTRACTION_MAX_CHUNKS` từ 400 xuống 150 để lách trần quota.**
**Từ chối vì:** đây là con số Scale Guard đề bài quy định. Lách trần bằng cách thu nhỏ bài toán là né vấn đề, không giải quyết. Thay vào đó tôi giải đúng gốc: phân bổ model theo bucket TPD riêng, thêm checkpoint, và bỏ coref cho chunk không có tham chiếu.

**3. Bọc `try/except` rộng quanh cả batch extraction cho "an toàn".**
**Từ chối vì:** chính phạm vi rộng đó đã làm **mất 34% chunk** — một `AttributeError` do phần tử rác trong mảng JSON giết cả 4 chunk dù 3 chunk có dữ liệu tốt. Phạm vi bắt lỗi phải bằng đơn vị nhỏ nhất có thể mất.

**4. Bỏ qua 2 câu cross-doc trả về `NaN` và báo cáo trung bình 4 câu còn lại.**
**Từ chối vì:** `NaN` ở đây không phải "không có dữ liệu" mà là **triệu chứng của một lỗi thật** (completion rỗng do reasoning token ăn hết ngân sách). Che đi thì mất luôn một ca lỗi đáng giá và bảng benchmark sẽ nói dối.

**5. Tự ước lượng số token để pace rate limit.** Đây là đề xuất tôi **đã áp dụng và đã sai** — nó gây ra 48% batch coref hỏng vì bucket cục bộ tự khoá chính nó. Bài học: không suy đoán trạng thái mà server đã công bố qua header.

---

## Câu 9 — Kiến trúc khi scale lên 350 MB (~100.000 bài)

### Bottleneck đầu tiên **không phải** Neo4j hay FAISS

Số liệu thực đo từ lab này:

| Thành phần | Thời gian cho 400 chunk | Ngoại suy cho 100.000 bài |
|---|---|---|
| **LLM extraction (coref + NER/RE)** | **~31 phút** | **~130 giờ** 🔴 |
| Embedding 1.506 chunk | ~60 giây | ~1,1 giờ |
| Neo4j bulk insert (UNWIND, batch 1000) | ~2 giây | ~10 phút |
| Đọc + lọc CSV 300 MB | 13 giây | ~15 giây |

**Bottleneck là hạn mức token của LLM extraction, lệch hai bậc độ lớn so với mọi thứ khác.** Và trần thật không phải token/phút mà là **200.000 token/NGÀY cho mỗi model** — ở hạn mức đó, 100.000 bài cần **hàng tháng**, bất kể hạ tầng mạnh đến đâu.

### Giải pháp kiến trúc, theo thứ tự ưu tiên

**1. Tầng lọc trước khi gọi LLM (rẻ nhất, hiệu quả nhất).**
Lab này đã chứng minh: lọc miền công nghệ giảm corpus từ 514.417 → 22.947 bài (**giảm 95,5%**) mà không mất bài liên quan nào. Bước bỏ coref cho chunk không chứa đại từ cũng cùng nguyên lý: **đừng gọi LLM cho việc mà regex làm được.**

**2. Song song hoá theo shard, không theo dòng.**
Chia corpus thành N shard độc lập, mỗi shard một worker + một API key/model riêng (mỗi model có bucket TPD riêng). Vì extraction là **embarrassingly parallel** ở mức chunk, throughput tăng gần tuyến tính theo số bucket. Với 5 bucket, 130 giờ → ~26 giờ.

**3. Checkpoint bắt buộc, không phải tuỳ chọn.**
Đã chứng minh giá trị: sau khi có checkpoint JSONL theo từng batch, chạy lại toàn bộ Phase 1 mất **70 giây** thay vì ~2 giờ. Ở quy mô 130 giờ, chạy không checkpoint là bất khả thi.

**4. Trích xuất phân tầng theo giá trị.**
Không phải chunk nào cũng đáng gọi model mạnh. Dùng NER rẻ (spaCy/regex) quét trước để chấm điểm mật độ thực thể, chỉ đẩy chunk giàu thực thể qua LLM. Lab này cho 182 triple/400 chunk = **0,46 triple/chunk** — hơn nửa số lượt gọi LLM không sinh ra cạnh nào.

**5. Entity Resolution phải chuyển sang blocking.**
Hiện tại `IndexFlatIP` là brute-force O(N²). Với 215 node thì không sao; với ~500.000 mention thì phải chuyển sang **HNSW + blocking key** (ví dụ ký tự đầu + độ dài sau khi bỏ hậu tố) để chỉ so sánh trong khối, và Union-Find chạy trên tập cạnh ứng viên đã thu hẹp.

**6. Neo4j: đã sẵn sàng, chỉ cần giữ kỷ luật.**
`UNWIND $rows` batch 1.000 + unique constraint trên `Entity.id` + index trên `name_norm` là đúng chuẩn. Ở quy mô lớn cần thêm: index trên `published_date` để phục vụ cắt tỉa temporal, và cân nhắc `apoc.periodic.iterate` cho batch cực lớn.

**7. Sửa mô hình truy vấn traversal.**
BFS hiện tại gọi 2 truy vấn Cypher **cho mỗi node** (`node_degree` + `recent_edges`) — chính là nguyên nhân latency 24s ở cross-doc. Ở đồ thị lớn phải gộp cả frontier vào **một** truy vấn `UNWIND $node_ids` và tính degree ngay trong truy vấn đó.

---

## Câu 10 — Giới hạn và tính trung thực của kết quả

Những điều sau ảnh hưởng tới cách đọc kết quả, tôi nêu rõ thay vì để người chấm tự phát hiện:

**1. Benchmark chưa đủ 6/6 câu.** Hai câu `cross-doc` (G05, G06) chưa chấm xong vì cạn hạn mức 200.000 token/ngày trên **cả hai** model dùng được (`gpt-oss-120b` và `gpt-oss-20b`). Hạn mức hồi lại khoảng 10.000 token/giờ. Cơ chế resume đã sẵn sàng: `run_evaluation()` đọc lại `CHECKPOINT` nên chỉ chạy 2 câu còn thiếu. Tôi **không** điền số ước lượng vào bảng.

**2. Generator và Judge dùng chung `gpt-oss-20b`** — có nguy cơ thiên vị tự chấm. Đây là hệ quả bắt buộc của việc `gpt-oss-120b` đã cạn quota và `qwen3.6-27b` không dùng được (output `<think>` phá JSON mode). Cách khắc phục đúng là dùng judge từ nhà cung cấp khác (OpenAI), nhưng `OPENAI_API_KEY` trong `.env` chỉ là placeholder.

**3. Cỡ mẫu nhỏ.** 6 câu hỏi trên đồ thị 215 node/181 cạnh. Chênh lệch +2,0 điểm là tín hiệu rõ nhưng **không có ý nghĩa thống kê**.

**4. Golden Dataset do tôi sinh từ chính đồ thị.** Bộ câu hỏi starter của đề hỏi về sự kiện không tồn tại trong 400 chunk được trích xuất (CEO Hugging Face, đầu tư của Meta/Apple) nên cả hai hệ thống đều trả lời "không đủ bằng chứng" và benchmark mất ý nghĩa. Tôi sinh câu hỏi từ chính các cạnh đã nạp, với ràng buộc quan trọng: **câu multi-hop bắt buộc có 2 cạnh nằm ở 2 `source_chunk_id` khác nhau** — đảm bảo Flat RAG không thể lấy đủ bằng chứng từ một chunk đơn lẻ. Điểm yếu cần thừa nhận: câu hỏi sinh từ đồ thị **thiên vị GraphRAG theo thiết kế**, vì mọi câu đều đảm bảo có đường đi trong đồ thị. Một bộ golden do người ra đề độc lập sẽ khắt khe hơn.

**5. Không có super-node thật.** Degree cao nhất là 19 so với ngưỡng 100, nên chính sách cắt tỉa chưa từng kích hoạt trong thực tế — đã bù bằng kiểm thử từng thành phần (Câu 3).

**6. Bộ lọc corpus có false positive.** Bài *"Adobe student receives national Information and Technology award"* lọt vào vì trường trung học tên **Adobe** Middle School. Tầng NER+RE phía sau đã loại (không sinh triple nào).

---

## Phụ lục — Tệp bằng chứng

| Tệp | Nội dung |
|---|---|
| `outputs/graphrag_eval_results.csv` | Kết quả chấm chi tiết từng câu |
| `outputs/graphrag_vs_flatrag_summary.csv` | Bảng so sánh tổng hợp theo nhóm |
| `outputs/golden_dataset.csv` | 6 câu hỏi + đáp án chuẩn + provenance |
| `outputs/entity_resolution_audit.csv` | 21 dòng audit, 4 nhóm quyết định |
| `outputs/cache/coref_checkpoint.jsonl` | 400 chunk sau phân giải đại từ |
| `outputs/cache/extraction_checkpoint.jsonl` | 400 chunk, 182 triple |
