# Suy Ngẫm & Kế Hoạch Đồ Án (Reflection & Action Plan) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tùng Dương
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

> File này tách từ `reports/lab_report.md` (bản gốc, đầy đủ nhất) để khớp đúng tên file `RUBRIC.md` yêu cầu. Nội dung giống hệt, không thêm/bớt thông tin.

---

## 1. Mapping Bài giảng vào Code
| Khái niệm | Hàm/Code | Quan sát & Đánh giá |
|---|---|---|
| Conservative Coreference | `resolve_coref_batch()`, `run_coref()` | Đúng thiết kế: 75/400 chunk giữ nguyên thay vì đoán, không phát hiện False Coreference nào |
| Near-Dedup (Bonus) | `near_dedup()`, `_minhash_signature()` | Loại 45/1500 bài gần trùng (threshold=0.85, 590 candidate pairs), đúng yêu cầu không dùng $O(N^2)$ |
| Schema & Allowlist Guard | `ALLOWED_NODE_TYPES/RELATIONS` | Hoạt động đúng, nhưng "đói dữ liệu" do 97/100 batch extraction lỗi rate-limit |
| Bulk Cypher Ingestion | `bulk_insert_nodes/edges()` | Đúng kỹ thuật `UNWIND`, `invalid_provenance_edges=0`, nhưng chỉ có 6 node/3 edge để insert |
| Entity Resolution & Union-Find | `build_resolution_map()`, `merge_guard()` | Guard đã nâng cấp (product-subbrand, same-surname) nhưng chưa test được với data thật |
| Super-node Degree Cap | `retrieve_graph_context()` | Logic đúng (kiểm chứng qua `test_supernode_policy()`), chưa kích hoạt điều kiện thật |
| LLM-as-a-Judge Evaluation | `judge_answer()`, `run_evaluation()` | Chạy đủ 50/50 câu, 3 nhóm, export đúng 2 file CSV yêu cầu |

## 2. Quá trình Debugging & Bài học
**Lỗi kỹ thuật phức tạp nhất:** chuỗi lỗi liên hoàn bắt đầu từ rate-limit Groq không được xử lý đúng. `groq_chat()` bản gốc chỉ backoff cố định tối đa ~20s/4 lần thử, trong khi Groq khi rate-limit trả về message ghi rõ thời gian chờ thật (ví dụ `"try again in 1m44.112s"`) — dài hơn nhiều 20s. Hậu quả dây chuyền: 97/100 batch NER/RE thất bại âm thầm (`except: continue` nuốt lỗi) → graph chỉ còn 6 node/3 edge; notebook vẫn báo "chạy xong, 0 lỗi" vì không cell nào crash — đây là dạng lỗi nguy hiểm nhất: **silent failure**, sai mà trông như đúng.

**Cách xử lý:** (1) viết lại `groq_chat()` đọc đúng số giây Groq đề xuất bằng regex thay vì đoán mò, tăng `max_retries` lên 6; (2) thêm cảnh báo chủ động ngay tại nguồn lỗi (`if raw_triples_df.empty: print(...)`) thay vì để crash mù mờ ở cell sau; (3) khi fix xong nhưng không kịp chạy lại full pipeline trước deadline (cell extraction từng treo 1 giờ ở lần chạy local do vẫn dính rate-limit dồn dập), quyết định dùng bản Colab đã hoàn thành làm bản nộp chính thức, báo cáo trung thực toàn bộ hạn chế thay vì che giấu.

**Bài học lớn nhất:** với hệ thống phụ thuộc external LLM API, retry logic phải tôn trọng tín hiệu backoff mà chính API trả về, không tự đoán số cố định — và mọi bước "âm thầm bỏ qua lỗi" trong pipeline production bắt buộc đi kèm cảnh báo đếm lỗi tổng hợp để tránh silent failure lan tới tận báo cáo cuối cùng.

## 3. Kế hoạch Áp dụng vào Đồ án Thực tế — P-168
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
