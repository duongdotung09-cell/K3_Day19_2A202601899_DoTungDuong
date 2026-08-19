# Thuyết Minh Kỹ Thuật (10 câu hỏi) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tùng Dương
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

> File này tách từ `reports/lab_report.md` (bản gốc, đầy đủ nhất) để khớp đúng tên file `RUBRIC.md` yêu cầu. Nội dung giống hệt, không thêm/bớt thông tin.

---

### 1. Coreference Resolution
Dataset thật `HackerNoon/tech-company-news-data-dump` chỉ có cột `description` (mô tả ngắn 1-2 câu), không có full-article-text như giả định ban đầu của đề bài (`text/content/article/body/story`). Vì antecedent dễ bị thiếu trong 1 chunk quá ngắn, pipeline đã được sửa để ghép thêm `companyName` (nhãn tên công ty chính xác có sẵn trong dataset) vào đầu mỗi văn bản trước khi đưa qua coref — đây là biện pháp giảm thiểu chủ động, không phải sửa lỗi coref.

Kết quả đo được: trên 400 chunk đưa qua coref, có **75/400 chunk (18.75%)** được LLM đánh dấu `unresolved_mentions` không rỗng — tức gần 1/5 số chunk có đại từ/tham chiếu mơ hồ (`it`, `they`, `the company`) mà antecedent không xuất hiện rõ trong cùng chunk, và hệ thống tuân đúng conservative rule: giữ nguyên văn bản gốc thay vì đoán.

**Tình huống cụ thể:** vì `description` thường bắt đầu ngay bằng câu như `"(Nasdaq: ON) a leader in..."` (chủ ngữ đầy đủ nằm ở title/companyName, không lặp lại trong mô tả), khi đại từ xuất hiện ở câu thứ 2 thì antecedent thật có thể đã bị cắt do giới hạn `CHUNK_WORDS=220` hoặc bản thân mô tả gốc đã dùng đại từ ngay từ câu đầu.

**Hậu quả với Knowledge Graph nếu coref đoán sai** (failure mode lý thuyết, không xảy ra ở đây nhờ rule conservative): nếu hệ thống suy diễn `"the company"` = công ty được nhắc gần nhất trong khi 1 đoạn có nhiều công ty (ví dụ "X hợp tác với Y, sau đó Z mua lại"), rất dễ gán nhầm quan hệ `ACQUIRED`/`PARTNERED_WITH` cho sai chủ thể → tạo **False Edge**, nghiêm trọng hơn 1 câu trả lời sai vì cạnh sai này sẽ lan truyền qua BFS traversal tới các câu hỏi multi-hop không liên quan tới chunk gốc.

### 2. Entity Resolution Threshold & Lexical Guard
**Ngưỡng đã chọn:** cosine similarity `0.90` cho candidate search bằng FAISS ANN (`IndexFlatIP` trên embedding normalize) + `SequenceMatcher ratio ≥ 0.72` cho Lexical Guard sau khi strip suffix (`Inc/Corp/Ltd/LLC`).

**Guard nâng cao (Challenge B):** ngoài suffix-strip mặc định, đã thêm 2 rule chặn false-merge nguy hiểm: (1) `REJECT_PRODUCT_SUBBRAND` — chặn khi tên dài hơn = tên ngắn + 1 từ sub-brand (`watch, music, pay,...`), ví dụ `Apple Watch` vs `Apple`; (2) `REJECT_SAME_SURNAME_DIFF_FIRSTNAME` — chặn khi 2 tên người cùng họ khác tên, ví dụ `Sam Altman` vs `Steve Altman`.

**Kết quả thật:** `entity_resolution_audit_df` có **0 dòng** — không phải vì Guard hoạt động sai, mà vì bước extraction chỉ tạo ra 3 triple hợp lệ trên 400 chunk (xem root-cause ở câu 3), nên không đủ mention để đưa vào so sánh similarity. Vì vậy không thể trích 1 cặp thật >0.85 bị chặn từ chính lần chạy này — đây là hạn chế thật, không giấu diếm.

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

### 4. Bảng so sánh Benchmark (Flat RAG vs GraphRAG, 50 câu golden)
| Nhóm | Comprehensiveness (F/G) | Faithfulness (F/G) | Multi-hop (F/G) | Latency (F/G) | Token (F/G) |
|---|---|---|---|---|---|
| cross-doc | 2.32/2.09 | 2.68/2.46 | 2.32/2.05 | 6.9s/4.5s | 1255/987 |
| factoid | 2.60/1.80 | 2.60/1.80 | 2.60/1.80 | 5.2s/3.5s | 1106/811 |
| multi-hop | 2.13/2.09 | 2.35/2.35 | 2.13/2.04 | 5.1s/5.3s | 1236/1002 |

**Diễn giải quan trọng nhất (đọc cùng câu 3):** GraphRAG không thắng Flat RAG ở nhóm nào, thua rõ ở `factoid` — nhưng đây không phải bằng chứng "kiến trúc GraphRAG kém hơn". Do graph gần rỗng, `retrieve_graph_context()` trả về context gần trống với **100% câu hỏi ghi nhận `graph_supernode_events=0`**, nên context GraphRAG trong benchmark này thực chất ≈ Vector-fallback context (`retrieve_flat_context(k=4)` gọi bên trong `answer_graph_rag()`) — đang so sánh Flat RAG top-6 với một Flat RAG top-4 kèm vài dòng đồ thị gần trống, giải thích luôn vì sao token/latency GraphRAG thấp hơn. Kết quả benchmark hợp lệ về kỹ thuật (không sửa/gian lận) nhưng không đại diện cho khả năng thật của kiến trúc — bài học nằm ở chất lượng Module 2 (extraction), không nằm ở thiết kế retrieval Module 4.

> 2 ca lỗi điển hình (Flat RAG thành công/thất bại, GraphRAG thành công/thất bại) được phân tích chi tiết trong `reports/failure_analysis.md`.

### 5. Trade-offs & Kiểm soát AI Coding Agent
**Đánh đổi Quality vs Cost vs Latency:** trong lần chạy này GraphRAG có latency/token thấp hơn Flat RAG (~4.4s/933 token so với ~5.7s/1199 token), nhưng đó là hệ quả phụ của graph rỗng (k=4 thay vì k=6), không phải đánh đổi thật của kiến trúc. Đánh đổi thật theo thiết kế: GraphRAG luôn tốn thêm **Indexing Overhead** một-lần-duy-nhất (NER+RE — tốn nhiều lệnh gọi LLM nhất hệ thống — + Entity Resolution + bulk insert Neo4j), đổi lại context sạch/liên kết hơn cho mỗi query về sau; Flat RAG rẻ lúc index nhưng mỗi query tốn context thô hơn, không được lọc/liên kết.

**Quyết định từ chối đề xuất AI Coding Agent:** khi thiết kế Near-Dedup (Challenge A), hướng đơn giản nhất mà agent có thể đề xuất là pairwise cosine similarity $O(N^2)$ trên toàn bộ embedding — đề bài tường minh cấm vì tràn RAM/thời gian khi scale. Đã chọn **MinHash signature (xấp xỉ Jaccard trên 5-gram từ) + LSH banding (16 band × 4 row = 64 permutation)**, gần $O(N)$ vì chỉ so sánh cặp rơi vào cùng bucket, có audit table ghi `estimated_jaccard` cho mỗi cặp merge để kiểm tra false positive.

**Giải pháp scale 350MB (~100,000 bài báo) — bottleneck đầu tiên:**
1. **NER/RE Extraction qua LLM** (đã kiểm chứng thật ở quy mô nhỏ: 97/100 batch lỗi chỉ với 400 chunk trên Groq free tier) — chắc chắn là bottleneck đầu tiên ở quy mô lớn. Giải pháp: batch extraction bất đồng bộ qua worker queue với nhiều API key luân phiên, tôn trọng đúng `Retry-After` thay vì backoff cố định (đã áp dụng fix này trong lab).
2. **Entity Resolution ANN:** FAISS `IndexFlatIP` là exact search $O(N)$/query — ở quy mô hàng trăm nghìn mention cần chuyển sang HNSW/IVF (approximate).
3. **Neo4j bulk insert:** `UNWIND` batch 1000 đã đúng hướng, nhưng cần thêm partition theo thời gian để tránh transaction quá lớn, cân nhắc GDS/Community Detection để pre-cluster trước khi insert.
