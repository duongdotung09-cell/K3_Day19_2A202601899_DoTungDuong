# Phân Tích Ca Lỗi (Failure Analysis) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tùng Dương
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

> File này tách từ `reports/lab_report.md` (bản gốc, đầy đủ nhất) để khớp đúng tên file `RUBRIC.md` yêu cầu. Nội dung giống hệt, không thêm/bớt thông tin.

---

## Bối cảnh chung (đọc trước khi vào 2 ca lỗi)

Graph thật thu được trong lần chạy chỉ có **6 node / 3 edge** (root cause: NER+RE extraction chỉ thành công 3/100 batch do rate-limit Groq — xem chi tiết trong `reports/technical_defense.md` câu 3). Vì graph gần rỗng, **100% câu hỏi ghi nhận `graph_supernode_events=0`**, khiến context GraphRAG trong benchmark này thực chất ≈ Vector-fallback context. Đây là điều kiện nền cần biết để hiểu đúng 2 ca lỗi dưới đây — cả hai đều là hệ quả trực tiếp của tình trạng graph thưa, không phải lỗi thiết kế retrieval.

## Ca lỗi 1 — Cả hai phương pháp đều thành công (Root-cause: thông tin nằm gọn trong 1 nguồn)

- **Question ID:** `G5000-01` (multi-hop) — *"Reconstruct the Aeris–Ericsson IoT transaction across the available reports: which Ericsson businesses moved to Aeris, and what scale of IoT connectivity was attributed to the resulting Aeris footprint?"*
- **Kết quả:** cả Flat RAG và GraphRAG đạt điểm tuyệt đối **5/5/5** (comprehensiveness/faithfulness/multi-hop reasoning). Cả hai cùng trích đúng chunk `b0cee438e01518c15f62` chứa đầy đủ thông tin: Ericsson IoT Accelerator + Connected Vehicle Cloud business chuyển sang Aeris, hỗ trợ >100 triệu thiết bị IoT cho 9,000 doanh nghiệp tại 190 quốc gia.
- **Root-cause vì sao 2 phương pháp không khác biệt:** GraphRAG context ở đây thực chất ≈ Vector context (do graph rỗng), nên 2 phương pháp gần như dùng cùng 1 cơ chế truy xuất trong trường hợp này.
- **Kết luận:** đây **không phải** ví dụ tốt để chứng minh ưu thế multi-hop của GraphRAG — nó chỉ chứng minh Flat RAG (top-k vector) đã đủ tốt khi câu trả lời nằm trọn trong 1 chunk liên tục, không cần truy vết qua nhiều bước quan hệ.

## Ca lỗi 2 — Cả hai phương pháp đều thất bại (Root-cause: giới hạn Scale Guard)

- **Question ID:** `G5000-13` (factoid) — *"Which three companies launched AI Lighthouse in the selected 5,000-row scope?"*
- **Kết quả:** cả Flat RAG và GraphRAG đều trả lời **"context không chứa thông tin"** (điểm 1/1/1 cả hai), trong khi `reference_answer` yêu cầu 3 công ty cụ thể (ServiceNow, NVIDIA, Accenture theo rationale của Judge).
- **Root-cause (truy vết theo quy trình root-cause analysis):**
  1. *Triệu chứng:* cả 2 phương pháp cùng báo thiếu context — loại trừ khả năng lỗi riêng của 1 kiến trúc.
  2. *Giả thuyết 1 — lỗi extraction:* loại trừ, vì Flat RAG (không phụ thuộc extraction) cũng thất bại y hệt.
  3. *Giả thuyết 2 — dữ liệu liên quan nằm ngoài phạm vi được xử lý:* xác nhận đúng — `LAB_MAX_CHUNKS=3000` chỉ lấy từ 1,500 bài được **sample ngẫu nhiên** trong tổng 4,466 bài đã dedup (Scale Guard của đề bài), và `EXTRACTION_MAX_CHUNKS=400` càng hẹp hơn nữa cho GraphRAG.
  4. *Nguyên nhân gốc:* bộ 50 câu golden được xây dựng từ phân tích **toàn bộ** 5,000 dòng dữ liệu gốc, trong khi pipeline chỉ thực sự xử lý một tập con lấy mẫu — đây là giới hạn Scale Guard **thật**, không phải bug logic.
- **Đề xuất khắc phục:**
  - (a) Tăng `LAB_MAX_CHUNKS`/`EXTRACTION_MAX_CHUNKS` nếu ngân sách thời gian/token cho phép, để tăng recall phạm vi dữ liệu được xử lý.
  - (b) Hoặc chấp nhận đây là giới hạn đã biết trước của Scale Guard trong khung 2 giờ lab và ghi rõ trong tài liệu bàn giao (như đang làm ở đây) thay vì âm thầm bỏ qua — tránh để người dùng cuối hiểu lầm là hệ thống "không tìm thấy vì không tồn tại" trong khi thực chất là "chưa từng được đưa vào xử lý".
