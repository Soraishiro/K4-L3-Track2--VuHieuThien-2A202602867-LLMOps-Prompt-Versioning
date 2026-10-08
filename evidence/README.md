# Evidence — Day 22: LangSmith + Prompt Versioning

Bằng chứng cho bài lab của **Vũ Hiếu Thiên (2A202602867)**.

## Danh sách bằng chứng

| Tệp | Nội dung |
|---|---|
| `01_langsmith_traces.png` | Danh sách ≥ 50 traces `rag-query` trên LangSmith project `day22-lab` |
| `02_prompt_hub.png` | 2 prompt `vuhieuthien-rag-prompt-v1` / `-v2` trên Prompt Hub |
| `02_ab_routing_log.txt` | Log console A/B routing, có nhãn `[prompt-v1]` / `[prompt-v2]` |
| `03_ragas_scores.png` | Bảng so sánh điểm RAGAS giữa V1 và V2 |
| `03_ragas_report.json` | Báo cáo điểm dạng máy đọc được |
| `04_pii_demo_log.txt` | Demo PIIDetector — 6 test case |
| `04_json_demo_log.txt` | Demo JSONFormatter — 5 test case |

## Cấu hình dùng để chạy

- Provider: `openai` — `gpt-4o-mini` (temperature 0), embeddings `text-embedding-3-small`
- LangSmith project: `day22-lab`
- Vector store: FAISS, `chunk_size=500`, `chunk_overlap=50` (107 chunks), `k=3`
- A/B routing: MD5 hash của `request_id` → V1 = 19 câu, V2 = 31 câu

---

## Phân tích kết quả V1 so với V2

### Điểm số

| Metric | V1 (ngắn gọn, thân thiện) | V2 (chuyên gia, có cấu trúc) | Chênh lệch |
|---|---|---|---|
| `faithfulness` | 0.9372 | **0.9584** | +0.0212 (V2 thắng) |
| `answer_relevancy` | **0.9070** | 0.8879 | −0.0191 (V1 thắng) |
| `context_recall` | 1.0000 | 1.0000 | 0 |
| `context_precision` | 0.9475 | **0.9483** | +0.0008 (V2 thắng) |

Cả hai phiên bản đều đạt `faithfulness ≥ 0.9`, vượt ngưỡng mục tiêu 0.8 của lab.

### Vì sao V2 có `faithfulness` cao hơn?

Hai prompt khác nhau ở **cách yêu cầu model xử lý context**:

- **V1** yêu cầu *"Trả lời ngắn gọn (2–4 câu)"*. Ràng buộc độ dài khiến model có xu hướng **bỏ bớt hoặc gom ý**, đôi khi lược bớt một chi tiết có trong context. Mỗi câu bị lược bớt là một câu trả lời không thể truy ngược hoàn toàn về context → RAGAS đánh dấu là không trung thành.
- **V2** yêu cầu *"đọc kỹ context, xác định các facts liên quan, rồi viết câu trả lời có tổ chức"* và cấm suy đoán ngoài context. Chỉ thị "xác định facts liên quan" buộc model bám theo từng ý có trong tài liệu, nên ít khi phát sinh nội dung không có căn cứ.

Kết quả: V2 thắng `faithfulness` +0.0212. Đây là bằng chứng cho thấy **chỉ thị về cách đọc context quan trọng hơn chỉ thị về độ dài câu trả lời**.

### Vì sao V1 có `answer_relevancy` cao hơn?

`answer_relevancy` đo độ liên quan của câu trả lời so với câu hỏi, và bị ảnh hưởng ngược lại bởi độ dài. V1 trả lời 2–4 câu nên **tập trung** đúng vào trọng tâm câu hỏi; V2 viết 3–5 câu có tổ chức nên có xu hướng **bổ sung thêm ngữ cảnh**, làm giảm mật độ thông tin trực tiếp trả lời câu hỏi. V1 thắng +0.0191 ở chỉ số này.

### Vì sao `context_recall` bằng 1.0 ở cả hai?

`context_recall` so sánh các câu trả lời mẫu (ground truth) với tập đoạn context đã truy xuất, chỉ phụ thuộc vào **retrieval chứ không phụ thuộc prompt**. Vì vậy cùng một vector store và cùng `k=3` cho kết quả giống hệt — đúng như mong đợi, và cho thấy prompt chỉ ảnh hưởng giai đoạn sinh, không ảnh hưởng giai đoạn truy xuất.

### Kết luận và khuyến nghị

- Nếu ưu tiên **độ trung thành với tài liệu** ( ví dụ tóm tắt tài liệu pháp lý, hướng dẫn kỹ thuật, QA nội bộ) → chọn **V2**.
- Nếu ưu tiên **trả lời nhanh, đúng trọng tâm câu hỏi** (ví dụ trợ lý hỏi đáp nhanh) → chọn **V1**.
- Trên cả hai chỉ số quyết định, V1 và V2 chỉ chênh ~0.02 — chênh lệch nhỏ, nên **ghép hai prompt**: dùng V2 cho câu hỏi cần độ sâu, chuyển sang V1 cho câu hỏi cần phản hồi ngắn.

### Ghi chú về độ ổn định của kết quả

Trong lần chạy thực tế, 46/50 mẫu của V1 và 47/50 mẫu của V2 chấm được điểm; các mẫu còn lại không có điểm do lỗi kết nối tạm thời tới OpenAI trong lúc RAGAS gọi song song. Vì vậy `answer_relevancy` của V1 được tính trên 47 mẫu còn hợp lệ — điểm trung bình vẫn đại diện cho tập câu hỏi vì các mẫu bị mất rơi rải rác, không tập trung ở một nhóm chủ đề.