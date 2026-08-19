# Phân Tích Ca Lỗi — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trần Công Đức
**Ngày thực hiện:** 19/08/2026
**Phạm vi:** Toàn bộ lỗi dưới đây đều là lỗi **quan sát được trong lần chạy thật**, kèm log/số liệu tái lập được. Không có ca giả định.

---

## Tóm tắt

| # | Ca lỗi | Tầng | Thiệt hại đo được | Trạng thái |
|---|--------|------|-------------------|------------|
| 1 | Seed extractor tự trả lời câu hỏi | Retrieval | GraphRAG sập về vector-only ở 2/2 câu factoid | ✅ Đã sửa |
| 2 | Câu trả lời rỗng khi context lớn | Generation | 2 câu cross-doc bị chấm 1/5 oan | ✅ Đã sửa |
| 3 | LLM chèn phần tử rỗng vào mảng JSON | Extraction | Mất 34% chunk (100/292) | ✅ Đã sửa |
| 4 | Rate-limiter tự khoá chính nó | Hạ tầng | 48% batch coref hỏng | ✅ Đã sửa |
| 5 | Trần token/NGÀY chứ không phải token/phút | Hạ tầng | Pipeline chết 2 lần | ⚠️ Giảm nhẹ |
| 6 | Alias trượt vì dấu chấm → đồ thị phân mảnh | Entity Resolution | `Microsoft` tách làm 2 node | ✅ Đã sửa |
| 7 | Lexical Guard chỉ được từ chối, không được cứu | Entity Resolution | `Adobe Inc.` ≠ `Adobe` | ✅ Đã sửa |
| 8 | False merge lọt cả hai tầng | Entity Resolution | `Tim Cook` + `Tim Cooke` | ⚠️ Tồn tại |
| 9 | Mẫu ngẫu nhiên làm đồ thị vô nghĩa | Dữ liệu | 4.5% corpus liên quan | ✅ Đã sửa |
| 10 | Không có checkpoint | Hạ tầng | Mất 58 phút coref | ✅ Đã sửa |

---

## CA LỖI 1 — Flat RAG thất bại, GraphRAG thành công *(ca lỗi bắt buộc #1)*

**Question ID:** `G02` — nhóm `factoid`
**Câu hỏi:** *"According to the news corpus, which company did Apple invest in?"*
**Đáp án chuẩn:** `SoftBank's UK chip unit`

### Kết quả

| Hệ thống | Comprehensiveness | Faithfulness | Multi-hop | Câu trả lời |
|---|---|---|---|---|
| Flat RAG | 1/5 | 1/5 | 1/5 | *"I couldn't find any mention in the supplied news excerpts that Apple invested in another company. The available chunks focus on Apple's market-cap milestones…"* |
| **GraphRAG** | **5/5** | **5/5** | **5/5** | *"Apple invested in SoftBank's UK chip unit【chunk_id=2a8d4bb2bc56803b2b0a::c0000】"* |

### Truy vết nguyên nhân gốc rễ — vì sao Flat RAG hỏng

Câu hỏi chứa từ khoá `Apple` và `invest`. Vector search trên `all-MiniLM-L6-v2` xếp hạng theo **độ tương đồng ngữ nghĩa toàn chunk**, mà trong corpus có rất nhiều chunk nói về Apple với mật độ từ vựng tài chính dày đặc hơn:

- Chunk về vốn hoá thị trường Apple (`market cap`, `$3 trillion`, `investors`, `shares`) — điểm cosine cao vì trùng *chủ đề tài chính* + *thực thể Apple*
- Chunk thật chứa sự kiện đầu tư vào SoftBank chỉ nhắc thoáng qua, nằm lẫn trong một bản tin tổng hợp

Top-6 chunk mà Flat RAG lấy về **không có chunk chứa quan hệ đích**. Đây là **failure mode kinh điển của Flat RAG: lexical/semantic drift** — truy hồi đúng *chủ đề* nhưng trượt *sự kiện*. Tăng k lên cũng chỉ làm loãng context chứ không đảm bảo trúng.

### Vì sao GraphRAG giải được

Bước trích xuất đã cô đọng chunk đó thành **một cạnh tường minh**:

```
Apple -INVESTED_IN-> SoftBank's UK chip unit
   | source_chunk_id = 2a8d4bb2bc56803b2b0a::c0000
   | published_date  = 2023-...
```

Truy hồi không còn phụ thuộc vào việc chunk đó có "trông giống câu hỏi" hay không. Seed `Apple` khớp exact vào `name_norm`, BFS 1 hop lấy toàn bộ 18 cạnh của Apple, trong đó cạnh `INVESTED_IN` được linearize thành text kèm provenance. **Quan hệ đã được vật chất hoá thành cấu trúc nên không thể bị "trôi" ra khỏi top-k nữa.**

Đây chính là luận điểm trung tâm của bài lab: *Flat RAG truy hồi theo độ giống; GraphRAG truy hồi theo quan hệ.*

---

## CA LỖI 2 — GraphRAG thất bại *(ca lỗi bắt buộc #2)*

**Question ID:** `G01` — nhóm `factoid`
**Câu hỏi:** *"According to the news corpus, which company did Microsoft acquire?"*
**Đáp án chuẩn:** `Activision Blizzard`

### Kết quả **trước khi sửa**

| Hệ thống | Faithfulness | Câu trả lời |
|---|---|---|
| Flat RAG | 5/5 | *"Microsoft acquired **Activision Blizzard**"* |
| GraphRAG | **1/5** ❌ | *"I couldn't find any mention in the provided news excerpts that Microsoft acquired a company."* |

Nghịch lý: đồ thị **có sẵn 10 cạnh** `Microsoft -ACQUIRED-> Activision Blizzard` (nhiều bài báo khác nhau), vậy mà GraphRAG lại nói không tìm thấy.

### Truy vết nguyên nhân gốc rễ

Truy vết ngược từ triệu chứng, loại trừ từng tầng:

**Bước 1 — Đồ thị có dữ liệu không?** Có.
```cypher
MATCH (a:Entity)-[r]->(b:Entity) WHERE a.name_norm='microsoft'
→ 10 × ACQUIRED → Activision Blizzard, 2 × DEVELOPED → Copilot, ...
```

**Bước 2 — Truy vấn traversal có chạy đúng không?** Có. Gọi trực tiếp `recent_edges(microsoft_id, 50)` trả về đủ 19 cạnh, sắp xếp đúng theo `published_date DESC`.

**Bước 3 — Seed matching có khớp không?** Có, nếu seed là `Microsoft`:
```
name='microsoft' typ='Company' → [{'name': 'Microsoft', 'type': 'Company'}]
```

**Bước 4 — Vậy seed thực tế là gì?** Đây là chỗ hỏng. Chạy `extract_seeds()` trên đúng câu hỏi đó:

```
G01 "According to the news corpus, which company did Microsoft acquire?"
   raw seeds: [{"name": "GitHub", "type": "Company"}]     ← !!
   MISS seed='GitHub' → []
   => tổng seed khớp: 0

G02 "...which company did Apple invest in?"
   raw seeds: [{"name": "Mojang", "type": "Company"}]     ← !!
   => tổng seed khớp: 0
```

**Nguyên nhân gốc rễ:** LLM trích seed đã **dùng kiến thức tham số sẵn có để đoán ĐÁP ÁN** thay vì trích thực thể có trong câu hỏi. Microsoft từng mua GitHub và Mojang ngoài đời thật — nên model trả về đáp án nó "nhớ được", bất chấp system prompt đã ghi rõ `Do not answer the question.`

Hệ quả dây chuyền: `GitHub` không tồn tại trong đồ thị → `match_seeds` trả rỗng → `retrieve_graph_context` trả `{"reason": "NO_SEED", "context": ""}` → prompt chỉ còn phần `=== VECTOR ===` → **GraphRAG âm thầm thoái hoá thành Flat RAG kém hơn** (k=4 thay vì k=6) mà không hề báo lỗi.

Đây là loại lỗi nguy hiểm nhất trong hệ thống production: **im lặng và trông như một câu trả lời hợp lệ**.

### Khắc phục

Hai lớp phòng thủ, lớp sau không phụ thuộc LLM:

```python
# Lớp 1 — Lexical guard: seed phải thực sự xuất hiện trong câu hỏi
if not _seed_in_question(name, query_norm):
    rejected.append(name)      # GitHub, Mojang bị loại tại đây
    continue

# Lớp 2 — Quét thẳng đồ thị, không hỏi LLM
def graph_seeds_from_text(query, max_seeds=8):
    MATCH (n:Entity)
    WHERE size(n.name_norm) >= 3 AND $q CONTAINS n.name_norm
    ORDER BY size(n.name_norm) DESC, deg DESC
```

Lớp 2 là điểm mấu chốt: nó biến seed matching từ **tác vụ sinh** (có thể bịa) thành **tác vụ tra cứu** (không thể bịa).

### Kết quả sau khi sửa

| Câu | GraphRAG *trước* | GraphRAG *sau* |
|---|---|---|
| G01 factoid | 1/5 ❌ | **5/5** ✅ `Microsoft acquired Activision Blizzard【b4c9b6f7…】【a7a43bb9…】` |
| G02 factoid | 1/5 ❌ | **5/5** ✅ |
| G03 multi-hop | 5/5 | 5/5 |
| G04 multi-hop | 5/5 | 5/5 |

---

## CA LỖI 3 — Câu trả lời rỗng bị chấm điểm như câu trả lời sai

**Question ID:** `G05`, `G06` — nhóm `cross-doc`

Cả hai câu cross-doc đều có `graph_answer` = `NaN` trong `graphrag_eval_results.csv`. Judge nhận chuỗi rỗng, chấm 1/5 cả ba tiêu chí → kéo tụt toàn bộ nhóm cross-doc của GraphRAG xuống 1.0.

**Nguyên nhân gốc rễ:** với context graph lớn (tổng hợp toàn bộ quan hệ của Microsoft/Apple), `gpt-oss-20b` ở `reasoning_effort="medium"` tiêu hết ngân sách completion cho **reasoning token** và trả về `content = ""`. Chuỗi rỗng ghi ra CSV, `pd.read_csv` đọc lại thành `NaN`.

Điểm đáng chú ý: **seed matching cho G05/G06 hoạt động hoàn toàn bình thường** (`Microsoft`, `Apple` đều khớp). Nghĩa là đây là lỗi độc lập ở tầng sinh, không liên quan ca lỗi 2 — nếu chỉ sửa seed thì hai câu này vẫn hỏng.

**Khắc phục:**
```python
text, usage = groq_chat(messages, reasoning_effort="medium")
if not str(text or "").strip():
    text, usage = groq_chat(messages, reasoning_effort="low")   # nhường ngân sách cho câu trả lời
if not str(text or "").strip():
    text = "[EMPTY_COMPLETION] Model không sinh được nội dung cho context này."
```

Cờ `[EMPTY_COMPLETION]` quan trọng không kém bản sửa: nó biến một lỗi **im lặng** thành lỗi **nhìn thấy được** trong bảng kết quả.

---

## CA LỖI 4 — LLM chèn phần tử rỗng vào mảng JSON

**Thiệt hại:** 100/292 chunk (34%) bị đánh dấu lỗi, mất trắng toàn bộ triple.

Tất cả đều cùng một thông báo: `'str' object has no attribute 'get'`. Bắt output thô thì thấy:

```json
{"items": [
    {"chunk_id": "27f4e7b1…", "relations": [{"source": "Samsung Electronics Co. Ltd.", …}]},
    "",                                            ← chuỗi rỗng xen giữa
    {"chunk_id": "ba8ab7a2…", "relations": [{"source": "Krayden", …}]},
    "",
    {"chunk_id": "42088aa7…", "relations": […]}
]}
```

**Điều này xảy ra dù đã bật `response_format={"type":"json_object"}`.** JSON vẫn hợp lệ về cú pháp — chỉ là schema không như yêu cầu. Code gốc gọi thẳng `item.get("chunk_id")` nên ném `AttributeError`, và vì `try/except` bọc cả batch nên **4 chunk chết chung** dù 3 trong số đó có dữ liệu tốt.

**Khắc phục:** parser phòng thủ theo kiểu dữ liệu ở mọi tầng lồng nhau.
```python
if not isinstance(item, dict):
    malformed_items += 1
    continue
rels = item.get("relations")
if not isinstance(rels, list):
    return []
for x in rels:
    if not isinstance(x, dict):
        continue
```

**Kết quả:** chunk lỗi giảm từ **34% → 2%** (8/400), số triple tăng từ 49 → **182**.

**Bài học chung:** *JSON mode đảm bảo cú pháp, không đảm bảo schema.* Mọi truy cập vào output LLM phải kiểm tra kiểu, và phạm vi `try/except` phải bằng đơn vị nhỏ nhất có thể mất — ở đây lẽ ra là từng chunk chứ không phải từng batch.

---

## CA LỖI 5 — Rate-limiter tự khoá chính nó *(lỗi do chính mình gây ra)*

Đây là lỗi tôi tự tạo ra khi tối ưu, và nó đắt nhất về thời gian.

**Bối cảnh:** chạy thuần retry/backoff thì từ batch ~25 trở đi mọi call đều dính 429, thời gian một batch coref phình từ **4s → 487s** (ETA cả vòng coref lên gần 7 giờ). Tôi thêm token-bucket tự ước lượng số token để chủ động chờ.

**Bug:** trong nhánh `except`, tôi vẫn ghi nhận số token *ước lượng* vào bucket:
```python
except Exception as e:
    _tpm_record(est)        # ← call hỏng vẫn cộng ~1.600 token vào bucket
```
Sau 5 lần retry, bucket cục bộ tự cộng thêm ~8.000 token = **đúng bằng hạn mức**. Bucket báo "hết quota" dù server chưa hề tính. Mỗi attempt tiếp theo phải ngủ 45s, hết 5 attempt là cả batch chết.

**Thiệt hại:** 60/125 chunk coref = **48% batch hỏng**.

**Khắc phục — bỏ hẳn việc đoán, đọc từ nguồn sự thật:**
```python
raw = groq_client.chat.completions.with_raw_response.create(**kwargs)
_rl_sync(raw.headers)          # đọc x-ratelimit-remaining-tokens từ chính server
resp = raw.parse()
```

**Kết quả:** coref 400/400 chunk, **0 batch lỗi**.

**Bài học:** không suy đoán trạng thái mà phía đối tác đã công bố. Và quan trọng hơn: **một cơ chế "bảo vệ" viết sai còn tệ hơn không có cơ chế nào** — nó biến lỗi tạm thời thành lỗi vĩnh viễn.

---

## CA LỖI 6 — Trần token/NGÀY, không phải token/phút

Sau khi sửa ca lỗi 5, pipeline vẫn chết. Header nói còn đủ quota:

```
x-ratelimit-limit-tokens     = 8000
x-ratelimit-remaining-tokens = 8000     ← còn đầy
x-ratelimit-reset-tokens     = 1ms
```

Nhưng request vẫn 429. Chỉ khi đọc **body** mới ra sự thật:

```json
"Rate limit reached … on tokens per day (TPD):
 Limit 200000, Used 199874, Requested 1476. Please try again in 9m43.2s"
```

**Hạn mức thật là 200.000 token/NGÀY cho mỗi model, và toàn bộ header `x-ratelimit-*` chỉ nói về cửa sổ PHÚT.** Giám sát theo header sẽ luôn báo "ổn" cho tới lúc dừng hẳn.

**Giảm nhẹ (không phải khắc phục triệt để — đây là giới hạn gói dịch vụ):**

1. **Phân bổ model theo bucket TPD riêng:**
   | Giai đoạn | Model | Lý do |
   |---|---|---|
   | Coreference | `openai/gpt-oss-safeguard-20b` | Bucket riêng, tác vụ cơ học |
   | NER + RE, seeds, generation, judge | `openai/gpt-oss-20b` | Chất lượng JSON tốt nhất trong các model còn dùng được |

2. **Fail-fast:** phân biệt 429-phút (chờ được) với 429-ngày (chờ vô ích) bằng cách parse `"try again in X"` trong body, ném `QuotaExhausted` ngay thay vì đốt 6 lần retry.

3. **Loại `groq/compound-mini`** dù nó có 70.000 TPM (gấp 8,75 lần): thử nghiệm cho thấy nó **route ngầm về chính `gpt-oss-120b`** nên dùng chung bucket đã cạn — và quan trọng hơn, model này tự bật web search, sẽ kéo thông tin ngoài context vào câu trả lời và **làm hỏng chính thứ benchmark này đo (Faithfulness)**.

4. **Bỏ coref cho chunk không có tham chiếu:** chunk không chứa đại từ/tham chiếu chung thì kết quả coref conservative luôn bằng văn bản gốc, gọi LLM chỉ tốn quota.

---

## CA LỖI 7 — Alias trượt vì dấu chấm, đồ thị phân mảnh

Truy vấn đồ thị cho thấy **hai node riêng biệt**:
```
{'name': 'Microsoft',       'norm': 'microsoft',       'type': 'Company'}   degree 19
{'name': 'Microsoft Corp.', 'norm': 'microsoft corp.', 'type': 'Company'}   ← tách rời
```

`MANUAL_ALIASES` có khoá `"microsoft corp"` (không dấu chấm), nhưng `norm_entity()` giữ lại dấu chấm trong regex `[^\w\s\-\.]` → sinh ra `"microsoft corp."`. Lookup trượt.

Đây là lỗi **một ký tự** nhưng hậu quả là chia đôi node trung tâm nhất của đồ thị: cạnh của Microsoft bị phân tán sang hai node, mọi đường multi-hop đi qua Microsoft đều có nguy cơ đứt.

**Khắc phục:** thử nhiều biến thể khi tra alias.
```python
for key in (norm_name, norm_name.replace(".", "").strip(), strip_suffix(norm_name)):
    if key in MANUAL_ALIASES:
        return MANUAL_ALIASES[key]
```

---

## CA LỖI 8 — Lexical Guard chỉ được từ chối, không được cứu

Kiến trúc gốc nối hai tầng bằng **AND**: `cosine ≥ 0.90` **và** `lexical ratio ≥ 0.72`. Nghĩa là Lexical Guard chỉ chạy *sau khi* đã qua tầng vector — nó có quyền phủ quyết nhưng **không có quyền cứu** cặp bị tầng vector loại oan.

Đo trên dữ liệu thật:

| Cặp | cosine | Lẽ ra | Thực tế |
|---|---|---|---|
| `Adobe Inc.` / `Adobe` | 0.814 | gộp | ❌ bị loại ở tầng vector |
| `Intel Corp` / `Intel` | 0.815 | gộp | ❌ bị loại ở tầng vector |
| `Advanced Micro Devices` / `Advanced Micro Devices Inc` | 0.817 | gộp | ❌ bị loại ở tầng vector |

Embedding của `all-MiniLM-L6-v2` **không xử lý tốt hậu tố pháp nhân** — thêm "Inc." làm cosine tụt xuống ~0.81, dưới ngưỡng 0.90. Trong khi đó `strip_suffix()` đã có sẵn logic đúng để nhận ra chúng là một, chỉ là không bao giờ được gọi tới.

**Khắc phục — thêm nhánh gộp thuần từ vựng, song song với nhánh vector:**
```python
by_stripped = defaultdict(list)
for i, nm in enumerate(names):
    by_stripped[strip_suffix(nm)].append(i)
for stripped, idxs in by_stripped.items():
    for j in idxs[1:]:
        audit.append({... "decision": "MERGE_LEXICAL"})
        uf.union(idxs[0], j)
```

**Kết quả:** bảng audit từ 6 dòng → **21 dòng**, thêm 6 ca `MERGE_LEXICAL` mà trước đó bị bỏ sót.

---

## CA LỖI 9 — False merge vẫn lọt cả hai tầng *(chưa khắc phục)*

Đây là rủi ro **còn tồn tại**, tôi ghi nhận thay vì che đi.

Kiểm thử trực tiếp từng tầng trên các cặp nguy hiểm:

| Cặp | cosine | lexical ratio | Qua vector? | Qua guard? | Kết cục |
|---|---|---|---|---|---|
| `Sam Altman` / `Steve Altman` | 0.824 | **0.727** | ❌ | ✅ | Thoát nhờ tầng vector |
| `RTX 4090` / `RTX 4070` | 0.803 | **0.875** | ❌ | ✅ | Thoát nhờ tầng vector |
| `ChatGPT` / `ChatGPT API` | 0.836 | **0.778** | ❌ | ✅ | Thoát nhờ tầng vector |
| **`Tim Cook` / `Tim Cooke`** | **0.902** | ~0.94 | ✅ | ✅ | 🔴 **SẼ GỘP NHẦM** |

Hai kết luận:

1. **Ngưỡng 0.72 của Lexical Guard trượt chính ví dụ mà đề bài nêu đích danh.** `Sam Altman` vs `Steve Altman` đạt ratio 0.727 — vượt ngưỡng đúng **0.007**. Cặp này chỉ thoát nhờ tầng vector (0.824 < 0.90), tức **guard đã thất bại đúng ở ca nó được thiết kế để chặn**.

2. **Với tên người gần trùng chính tả, cả hai tầng cùng thủng.** `Tim Cook` / `Tim Cooke` đạt cosine 0.902 (vượt 0.90) và ratio ~0.94 (vượt 0.72) → sẽ bị gộp thành một node. Corpus hiện tại chưa chứa cặp này nên chưa gây hại, nhưng ở quy mô lớn hơn thì chắc chắn sẽ gặp.

**Đề xuất khắc phục (chưa triển khai):**
- Với `type == "Person"`: yêu cầu **họ và tên khớp token-level tuyệt đối**, không dùng ratio ký tự — vì sai một chữ cái trong tên người thường là *người khác*, không phải biến thể viết.
- Với `type == "Technology"`: thêm luật **khác chữ số ⇒ không gộp** (`RTX 4090` vs `RTX 4070`) — chữ số trong tên sản phẩm mang nghĩa phân biệt phiên bản.
- Nâng ngưỡng lexical cho tên người từ 0.72 lên ≥ 0.85 và bổ sung kiểm tra độ dài chênh lệch.

---

## CA LỖI 10 — Mẫu ngẫu nhiên làm đồ thị vô nghĩa

Áp thẳng Scale Guard `LAB_MAX_ARTICLES = 1500` lên toàn corpus 514.417 bài cho ra một đồ thị vô dụng: dump này gồm ~190.000 `companyName` khác nhau, phần lớn là thông cáo báo chí của doanh nghiệp nhỏ **không liên kết chéo với nhau**. Chỉ **4,5% corpus (22.947 bài)** có nhắc tới một hãng công nghệ lớn.

Mẫu ngẫu nhiên 1.500 bài → chỉ ~65 bài liên quan → đồ thị vỡ thành hàng trăm thành phần liên thông rời rạc, **mọi câu hỏi multi-hop và cross-doc đều vô nghĩa** vì không tồn tại đường đi nào dài hơn 1 cạnh.

**Khắc phục:** lọc corpus về miền tin công nghệ **trước**, rồi mới áp Scale Guard lên tập đã lọc. Số bài đưa vào pipeline vẫn đúng 1.500 theo đề.

**Đánh đổi phải ghi nhận:** bộ lọc từ khoá có false positive — ví dụ bài *"Adobe student receives national Information and Technology award"* lọt vào vì trường trung học tên **Adobe** Middle School. Đây là nhiễu chấp nhận được ở tầng lọc thô, và tầng NER+RE phía sau đã loại bỏ (không sinh triple nào từ chunk đó).

**Một quyết định audit dữ liệu khác:** cột `companyName` **cố ý không đưa vào** nội dung chunk. Kiểm tra thực tế cho thấy nó là *nguồn crawl* chứ không phải chủ thể bài viết — ví dụ `companyName = "01Synergy"` cho bài viết về onsemi và Adobe Middle School. Đưa vào sẽ bơm hàng loạt thực thể sai vào Knowledge Graph.

---

## CA LỖI 11 — Không có checkpoint cho pipeline 2,5 giờ

Tiến trình bị kill giữa chừng, **58 phút coref mất trắng** vì kết quả chỉ nằm trong bộ nhớ kernel.

**Khắc phục:** checkpoint JSONL theo **từng batch**, kèm `fsync`:
```python
for rec in df.to_dict("records"):
    done[rec["chunk_id"]] = rec
    f.write(json.dumps(rec, ensure_ascii=False) + "\n")
f.flush(); os.fsync(f.fileno())
```
Áp cho cả 3 bước tốn kém: coref, extraction, và evaluation.

**Giá trị đo được:** sau khi có checkpoint, một lần chạy lại toàn bộ Phase 1 (M1→M4) mất **70 giây** thay vì ~2 giờ. Nhờ vậy 5 lần chạy lại tiếp theo để sửa lỗi mới khả thi về mặt thời gian.

Batch lỗi cũng được ghi checkpoint kèm cờ `error` — để không lặp vô hạn, nhưng vẫn purge được có chọn lọc khi đã sửa nguyên nhân:
```
Checkpoint extraction: 292 -> giu 192 (100 ban ghi loi da xoa de chay lai)
```

---

## Ca lỗi hạ tầng bổ sung — kernel chết vì hết RAM

`DeadKernelError` tại cell 3.1 (build FAISS index). Máy có 7,4 GB RAM, còn trống 1,2 GB; `raw_df` giữ nguyên 514k dòng (~400 MB) trong khi bước embedding cần nạp thêm torch + sentence-transformers.

**Khắc phục:** giải phóng tường minh sau khi chunking xong.
```python
import gc
del raw_df, _blob, _focus
gc.collect()
```

---

## Tổng kết bài học

1. **Lỗi im lặng nguy hiểm hơn lỗi ồn ào.** Ca lỗi 2 và 3 đều trả về "câu trả lời trông hợp lệ" — nếu chỉ nhìn bảng điểm mà không mở từng câu ra đọc thì sẽ kết luận sai rằng "GraphRAG kém hơn ở nhóm factoid".
2. **JSON mode đảm bảo cú pháp, không đảm bảo schema.**
3. **Đừng để LLM làm việc mà tra cứu làm được.** Seed matching là tra cứu, không phải sinh.
4. **Phạm vi `try/except` quyết định thiệt hại.** Bọc theo batch làm mất 4 chunk mỗi lần; bọc theo chunk chỉ mất 1.
5. **Cơ chế bảo vệ viết sai còn tệ hơn không có.**
6. **Giới hạn thật thường không nằm ở chỗ ta đang giám sát** — header nói phút, trần thật nằm ở ngày.
