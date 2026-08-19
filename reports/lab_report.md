# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tùng Dương
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution
Dataset thật `HackerNoon/tech-company-news-data-dump` chỉ có cột `description` (mô tả ngắn 1-2 câu), không có full-article-text như giả định ban đầu của đề bài (`text/content/article/body/story`). Vì antecedent dễ bị thiếu trong 1 chunk quá ngắn, pipeline đã được sửa để ghép thêm `companyName` (nhãn tên công ty chính xác có sẵn trong dataset) vào đầu mỗi văn bản trước khi đưa qua coref — đây là biện pháp giảm thiểu chủ động, không phải sửa lỗi coref.

Kết quả đo được: trên 400 chunk đưa qua coref, có **75/400 chunk (18.75%)** được LLM đánh dấu `unresolved_mentions` không rỗng — tức gần 1/5 số chunk có đại từ/tham chiếu mơ hồ (`it`, `they`, `the company`) mà antecedent không xuất hiện rõ trong cùng chunk, và hệ thống tuân đúng conservative rule: giữ nguyên văn bản gốc thay vì đoán.

**Tình huống cụ thể:** vì `description` thường bắt đầu ngay bằng câu như `"(Nasdaq: ON) a leader in..."` (chủ ngữ đầy đủ nằm ở title/companyName, không lặp lại trong mô tả), khi đại từ xuất hiện ở câu thứ 2 thì antecedent thật có thể đã bị cắt do giới hạn `CHUNK_WORDS=220` hoặc bản thân mô tả gốc đã dùng đại từ ngay từ câu đầu.

**Hậu quả với Knowledge Graph nếu coref đoán sai** (failure mode lý thuyết, không xảy ra ở đây nhờ rule conservative): nếu hệ thống suy diễn `"the company"` = công ty được nhắc gần nhất trong khi 1 đoạn có nhiều công ty (ví dụ "X hợp tác với Y, sau đó Z mua lại"), rất dễ gán nhầm quan hệ `ACQUIRED`/`PARTNERED_WITH` cho sai chủ thể → tạo **False Edge**, nghiêm trọng hơn 1 câu trả lời sai vì cạnh sai này sẽ lan truyền qua BFS traversal tới các câu hỏi multi-hop không liên quan tới chunk gốc.

### 2. Entity Resolution Threshold & Lexical Guard
**Ngưỡng đã chọn:** cosine similarity `0.90` cho candidate search bằng FAISS ANN (`IndexFlatIP` trên embedding normalize) + `SequenceMatcher ratio ≥ 0.72` cho Lexical Guard sau khi strip suffix (`Inc/Corp/Ltd/LLC`).

**Guard nâng cao (Challenge B):** ngoài suffix-strip mặc định, đã thêm 2 rule chặn false-merge nguy hiểm: (1) `REJECT_PRODUCT_SUBBRAND` — chặn khi tên dài hơn = tên ngắn + 1 từ sub-brand (`watch, music, pay,...`), ví dụ `Apple Watch` vs `Apple`; (2) `REJECT_SAME_SURNAME_DIFF_FIRSTNAME` — chặn khi 2 tên người cùng họ khác tên, ví dụ `Sam Altman` vs `Steve Altman`.

**Kết quả thật:** `entity_resolution_audit_df` có **0 dòng** — không phải vì Guard hoạt động sai, mà vì bước extraction chỉ tạo ra 3 triple hợp lệ trên 400 chunk (xem root-cause ở mục 4), nên không đủ mention để đưa vào so sánh similarity. Vì vậy không thể trích 1 cặp thật >0.85 bị chặn từ chính lần chạy này — đây là hạn chế thật, không giấu diếm.

**Minh hoạ độc lập đúng logic Guard** (gọi trực tiếp hàm với dữ liệu mẫu, không phải từ audit graph thật): `merge_guard("Apple Watch", "Apple")` → cosine similarity giữa 2 embedding này thường >0.85, nhưng Guard vẫn trả về `REJECT_PRODUCT_SUBBRAND` vì `"watch"` nằm trong danh sách sub-brand — đúng ý đồ thiết kế: sản phẩm không được gộp làm một với công ty mẹ dù giống nhau về ngữ nghĩa bề mặt.

### 3. Đồ thị & Super-node Mitigation
| Hạng | Tên thực thể | Type | Degree |
|---|---|---|---|
| 1 | Dash Solutions | Company | 1 |
| 2 | SMS Assist | Company | 1 |
| 3 | KyckGlobal | Company | 1 |

Đồ thị thu được trong lần chạy này chỉ có **6 node / 3 edge**, bậc cao nhất chỉ = 1 — **không có super-node thật (degree>100) nào hình thành**, nên cơ chế `SUPER_NODE_DEGREE=100 → cap 50 edge` chưa được kiểm chứng bằng dữ liệu thật. Nguyên nhân gốc: NER+RE extraction chỉ thành công 3/100 batch (`Extracted triples: 3 | extraction errors: 97`), do rate-limit Groq khi gọi 100 request liên tiếp mà retry ban đầu chỉ backoff tối đa ~20s, không đủ so với thời gian chờ thật Groq yêu cầu (có lúc >1 phút) — 97 batch bị timeout và bị `except: continue` bỏ qua âm thầm.

Bằng chứng gián tiếp cơ chế vẫn đúng ở mức unit: `test_supernode_policy()` chạy trên node degree cao nhất (=1), xác nhận số cạnh fetch về = 1 ≤ limit — logic if/else đúng, chỉ chưa từng kích hoạt điều kiện `degree>100` vì graph quá thưa.

**Ưu điểm & rủi ro của Temporal Mitigation** (phân tích lý thuyết, áp dụng khi graph đủ lớn): *Ưu điểm* — với super-node thật (ví dụ Google/Microsoft nối tới hàng nghìn entity), cap 50 cạnh mới nhất giữ context nhỏ gọn, tránh bùng nổ token/latency, ưu tiên thông tin thời sự (phù hợp tin tức công nghệ vốn dễ lỗi thời). *Rủi ro* — câu hỏi về sự kiện lịch sử xa (ví dụ "Microsoft đầu tư OpenAI lần đầu năm nào") có thể bị cắt mất nếu node đã tích luỹ >50 cạnh mới hơn kể từ đó — đánh đổi Recall lịch sử lấy Relevance thời sự.

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG, 50 câu golden)
| Nhóm | Comprehensiveness (F/G) | Faithfulness (F/G) | Multi-hop (F/G) | Latency (F/G) | Token (F/G) |
|---|---|---|---|---|---|
| cross-doc | 2.32/2.09 | 2.68/2.46 | 2.32/2.05 | 6.9s/4.5s | 1255/987 |
| factoid | 2.60/1.80 | 2.60/1.80 | 2.60/1.80 | 5.2s/3.5s | 1106/811 |
| multi-hop | 2.13/2.09 | 2.35/2.35 | 2.13/2.04 | 5.1s/5.3s | 1236/1002 |

**Diễn giải quan trọng nhất (đọc cùng mục 3):** GraphRAG không thắng Flat RAG ở nhóm nào, thua rõ ở `factoid` — nhưng đây không phải bằng chứng "kiến trúc GraphRAG kém hơn". Do graph gần rỗng, `retrieve_graph_context()` trả về context gần trống với **100% câu hỏi ghi nhận `graph_supernode_events=0`**, nên context GraphRAG trong benchmark này thực chất ≈ Vector-fallback context (`retrieve_flat_context(k=4)` gọi bên trong `answer_graph_rag()`) — đang so sánh Flat RAG top-6 với một Flat RAG top-4 kèm vài dòng đồ thị gần trống, giải thích luôn vì sao token/latency GraphRAG thấp hơn. Kết quả benchmark hợp lệ về kỹ thuật (không sửa/gian lận) nhưng không đại diện cho khả năng thật của kiến trúc — bài học nằm ở chất lượng Module 2 (extraction), không nằm ở thiết kế retrieval Module 4.

**Ca lỗi 1 — cả hai đều thành công (`G5000-01`, multi-hop):** cả Flat RAG và GraphRAG đạt 5/5/5, cùng trích đúng chunk chứa đủ thông tin (Ericsson IoT Accelerator + Connected Vehicle Cloud chuyển sang Aeris, hỗ trợ >100 triệu thiết bị IoT). Vì GraphRAG context ở đây ≈ Vector context (graph rỗng), đây không phải ví dụ tốt để chứng minh ưu thế multi-hop của GraphRAG — nó chỉ chứng minh Flat RAG top-k đã đủ tốt khi thông tin nằm gọn trong 1 chunk liên tục.

**Ca lỗi 2 — cả hai đều thất bại (`G5000-13`, factoid):** cả hai trả lời "context không chứa thông tin" (1/1/1), trong khi đáp án cần 3 công ty cụ thể (ServiceNow, NVIDIA, Accenture). Root cause: dữ liệu về sự kiện "AI Lighthouse" nhiều khả năng nằm ngoài phạm vi Scale Guard — `LAB_MAX_CHUNKS=3000` chỉ lấy từ 1,500 bài được sample ngẫu nhiên trong 4,466 bài đã dedup, còn `EXTRACTION_MAX_CHUNKS=400` càng hẹp hơn — trong khi bộ 50 câu golden được xây từ phân tích toàn bộ 5,000 dòng gốc. Đây là giới hạn Scale Guard thật, không phải bug. Đề xuất khắc phục: tăng cap nếu đủ ngân sách thời gian/token, hoặc chấp nhận và ghi rõ giới hạn này thay vì âm thầm bỏ qua.

### 5. Trade-offs & Kiểm soát AI Coding Agent
**Đánh đổi Quality vs Cost vs Latency:** trong lần chạy này GraphRAG có latency/token thấp hơn Flat RAG (~4.4s/933 token so với ~5.7s/1199 token), nhưng đó là hệ quả phụ của graph rỗng (k=4 thay vì k=6), không phải đánh đổi thật của kiến trúc. Đánh đổi thật theo thiết kế: GraphRAG luôn tốn thêm **Indexing Overhead** một-lần-duy-nhất (NER+RE — tốn nhiều lệnh gọi LLM nhất hệ thống — + Entity Resolution + bulk insert Neo4j), đổi lại context sạch/liên kết hơn cho mỗi query về sau; Flat RAG rẻ lúc index nhưng mỗi query tốn context thô hơn, không được lọc/liên kết.

**Quyết định từ chối đề xuất AI Coding Agent:** khi thiết kế Near-Dedup (Challenge A), hướng đơn giản nhất mà agent có thể đề xuất là pairwise cosine similarity $O(N^2)$ trên toàn bộ embedding — đề bài tường minh cấm vì tràn RAM/thời gian khi scale. Đã chọn **MinHash signature (xấp xỉ Jaccard trên 5-gram từ) + LSH banding (16 band × 4 row = 64 permutation)**, gần $O(N)$ vì chỉ so sánh cặp rơi vào cùng bucket, có audit table ghi `estimated_jaccard` cho mỗi cặp merge để kiểm tra false positive.

**Giải pháp scale 350MB (~100,000 bài báo) — bottleneck đầu tiên:**
1. **NER/RE Extraction qua LLM** (đã kiểm chứng thật ở quy mô nhỏ: 97/100 batch lỗi chỉ với 400 chunk trên Groq free tier) — chắc chắn là bottleneck đầu tiên ở quy mô lớn. Giải pháp: batch extraction bất đồng bộ qua worker queue với nhiều API key luân phiên, tôn trọng đúng `Retry-After` thay vì backoff cố định (đã áp dụng fix này trong lab).
2. **Entity Resolution ANN:** FAISS `IndexFlatIP` là exact search $O(N)$/query — ở quy mô hàng trăm nghìn mention cần chuyển sang HNSW/IVF (approximate).
3. **Neo4j bulk insert:** `UNWIND` batch 1000 đã đúng hướng, nhưng cần thêm partition theo thời gian để tránh transaction quá lớn, cân nhắc GDS/Community Detection để pre-cluster trước khi insert.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping Bài giảng vào Code
| Khái niệm | Hàm/Code | Quan sát & Đánh giá |
|---|---|---|
| Conservative Coreference | `resolve_coref_batch()`, `run_coref()` | Đúng thiết kế: 75/400 chunk giữ nguyên thay vì đoán, không phát hiện False Coreference nào |
| Near-Dedup (Bonus) | `near_dedup()`, `_minhash_signature()` | Loại 45/1500 bài gần trùng (threshold=0.85, 590 candidate pairs), đúng yêu cầu không dùng $O(N^2)$ |
| Schema & Allowlist Guard | `ALLOWED_NODE_TYPES/RELATIONS` | Hoạt động đúng, nhưng "đói dữ liệu" do 97/100 batch extraction lỗi rate-limit |
| Bulk Cypher Ingestion | `bulk_insert_nodes/edges()` | Đúng kỹ thuật `UNWIND`, `invalid_provenance_edges=0`, nhưng chỉ có 6 node/3 edge để insert |
| Entity Resolution & Union-Find | `build_resolution_map()`, `merge_guard()` | Guard đã nâng cấp (product-subbrand, same-surname) nhưng chưa test được với data thật |
| Super-node Degree Cap | `retrieve_graph_context()` | Logic đúng (kiểm chứng qua `test_supernode_policy()`), chưa kích hoạt điều kiện thật |
| LLM-as-a-Judge Evaluation | `judge_answer()`, `run_evaluation()` | Chạy đủ 50/50 câu, 3 nhóm, export đúng 2 file CSV yêu cầu |

### 2. Quá trình Debugging & Bài học
**Lỗi kỹ thuật phức tạp nhất:** chuỗi lỗi liên hoàn bắt đầu từ rate-limit Groq không được xử lý đúng. `groq_chat()` bản gốc chỉ backoff cố định tối đa ~20s/4 lần thử, trong khi Groq khi rate-limit trả về message ghi rõ thời gian chờ thật (ví dụ `"try again in 1m44.112s"`) — dài hơn nhiều 20s. Hậu quả dây chuyền: 97/100 batch NER/RE thất bại âm thầm (`except: continue` nuốt lỗi) → graph chỉ còn 6 node/3 edge; notebook vẫn báo "chạy xong, 0 lỗi" vì không cell nào crash — đây là dạng lỗi nguy hiểm nhất: **silent failure**, sai mà trông như đúng.

**Cách xử lý:** (1) viết lại `groq_chat()` đọc đúng số giây Groq đề xuất bằng regex thay vì đoán mò, tăng `max_retries` lên 6; (2) thêm cảnh báo chủ động ngay tại nguồn lỗi (`if raw_triples_df.empty: print(...)`) thay vì để crash mù mờ ở cell sau; (3) khi fix xong nhưng không kịp chạy lại full pipeline trước deadline (cell extraction từng treo 1 giờ ở lần chạy local do vẫn dính rate-limit dồn dập), quyết định dùng bản Colab đã hoàn thành làm bản nộp chính thức, báo cáo trung thực toàn bộ hạn chế thay vì che giấu.

**Bài học lớn nhất:** với hệ thống phụ thuộc external LLM API, retry logic phải tôn trọng tín hiệu backoff mà chính API trả về, không tự đoán số cố định — và mọi bước "âm thầm bỏ qua lỗi" trong pipeline production bắt buộc đi kèm cảnh báo đếm lỗi tổng hợp để tránh silent failure lan tới tận báo cáo cuối cùng.

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế — P-168
**Dự án:** P-168 — AI Agent tích hợp chat, phát hiện URL phishing/tin giả, chấm điểm rủi ro 0–100 và cảnh báo inline trong hội thoại (LangGraph: `extract_links → rule_scan → fetch_reputation? → reputation_scan → verify_claim? → score_and_decide → escalate?`).

**Có cần GraphRAG không — câu trả lời tách theo từng phần, không phải một chữ "có/không":**
- **MVP hiện tại** (rule_scan, reputation_scan, escalate) — **không cần GraphRAG/RAG**. Đây là bài toán rule-based + cache/lookup trên 1 URL đơn lẻ tại 1 thời điểm, không có nhu cầu multi-hop; thêm pipeline NER/RE + Neo4j chỉ tạo latency và chi phí thừa cho quyết định cần nhanh/rẻ/giải thích ngay (đúng nguyên tắc "rule/cache trước, dịch vụ ngoài tốn phí sau" đã có trong PRD).
- **`verify_claim` (Could-have, RAG fact-check qua pgvector)** — **Flat/Vector RAG là đủ**. Bài toán "claim có khớp 1 nguồn fact-check đáng tin không" về bản chất là single-hop semantic similarity, giống hệt case `G5000-01` trong lab (Flat RAG top-k đã đủ khi info nằm trong 1 nguồn liên tục).
- **Phân tích xu hướng/chiến dịch (Could-have, chưa làm)** — **đây mới là nơi GraphRAG thực sự cần**. Phát hiện 1 chiến dịch phishing đòi hỏi đúng dạng câu hỏi multi-hop/cross-doc: "các URL này có dùng chung hạ tầng/registrant không, dù bề ngoài không liên quan?" — cần truy vết qua nhiều bước quan hệ (URL→IP/Domain→Registrant→URL khác dùng chung Registrant→cùng 1 Brand bị giả mạo).

**Cấu trúc Node & Relation dự kiến** (cho module phân tích chiến dịch — mở rộng tương lai, không phải MVP):
- **Nodes:** `URL` (id = hash URL chuẩn hoá), `Domain`, `IP`, `Registrant` (WHOIS, thường ẩn qua privacy-proxy — xử lý như node "unknown" riêng, không suy diễn danh tính), `Brand` (thương hiệu bị giả mạo), `Campaign` (cụm do hệ thống tự sinh, tương tự `community_id` từ NetworkX trong phần Bonus).
- **Relations** (allowlist tường minh như `ALLOWED_RELATIONS` trong lab): `RESOLVES_TO`, `BELONGS_TO_DOMAIN`, `REGISTERED_BY`, `IMPERSONATES`, `SHARES_INFRA_WITH` (cạnh suy diễn khi cùng IP/Registrant, cần `confidence` thấp hơn cạnh trực tiếp), `SCANNED_IN` (URL→ScanEvent, bắt buộc provenance `scanned_at/risk_score/decision/rule_flags` — map 1-1 với yêu cầu edge provenance của lab), `ESCALATED_TO`.

**Super-node & Entity Resolution — bài học đảo ngược từ lab:**
- **Super-node:** node `IP`/`Domain` thuộc hạ tầng dùng chung (Cloudflare Pages, bit.ly, ngrok) sẽ có degree cao một cách giả — hàng nghìn URL không liên quan cùng trỏ về 1 IP chia sẻ. Nguy hiểm hơn case Google/Microsoft trong lab vì nó gây **liên kết sai chiến dịch** (false-link giữa các campaign không liên quan), không chỉ bùng nổ context. Chiến lược: áp dụng đúng `SUPER_NODE_DEGREE`/`SUPER_NODE_EDGE_CAP` của lab, cộng thêm whitelist "hạ tầng dùng chung đã biết" để loại các node này khỏi suy luận `SHARES_INFRA_WITH` ngay từ đầu, thay vì chỉ cắt theo thời gian.
- **Entity Resolution:** ngược hoàn toàn với lab (nơi rủi ro là bỏ sót merge, ví dụ không nhận ra MSFT=Microsoft) — ở đây rủi ro nguy hiểm nhất là **hợp nhất nhầm domain giả mạo với brand thật** vì embedding/lexical similarity cao (typosquatting, Punycode homoglyph — chính là kỹ thuật tấn công). Domain resolution phải mặc định **không merge** trừ khi khớp chính xác qua chuẩn hoá kỹ thuật (bỏ `www.`, IDNA-decode Punycode rồi so khớp byte-for-byte), tuyệt đối không dùng cosine similarity ngữ nghĩa cho domain — vì "trông giống nhau" ở đây chính là mục tiêu tấn công, không phải tín hiệu để tin tưởng.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu rõ kiến trúc/trade-off, chẩn đoán đúng root-cause của kết quả benchmark bất thường |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối giải pháp O(N²) cho near-dedup, chủ động audit toàn bộ 38 cell trước khi chạy |
| Chất lượng đồ thị tri thức xây dựng | 2 | Graph thực tế quá thưa (6 node/3 edge) do lỗi rate-limit chưa kịp khắc phục hoàn toàn trước deadline |
| Khả năng phân tích và debug hệ thống | 5 | Truy vết đúng chuỗi nguyên nhân silent-failure: rate-limit → extraction fail → graph rỗng |
