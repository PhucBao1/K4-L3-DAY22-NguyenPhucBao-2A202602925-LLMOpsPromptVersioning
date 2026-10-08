# Evidence — Day 22: LangSmith + Prompt Versioning

**Học viên:** Nguyễn Phúc Bảo — 2A202602925
**LangSmith project:** `day22-lab` — https://smith.langchain.com/o/206ee5ae-3503-4440-a2cd-58188732c853/projects/p/2b2cf950-09b4-4f54-9e7e-9725389ded64
**Provider:** OpenAI `gpt-4o-mini` (LLM + RAGAS judge), `text-embedding-3-small` (embeddings)

## Danh sách file

| File | Nội dung |
|---|---|
| `01_langsmith_traces.png` | 50 traces `rag-query` (Bước 1) |
| `01_langsmith_total_traces.png` | Toàn bộ project `day22-lab`: 504 traces (bổ sung) |
| `02_prompt_hub.png` | 2 prompt trên Hub: `nguyen-phuc-bao-rag-prompt-v1`, `nguyen-phuc-bao-rag-prompt-v2` |
| `02_ab_routing_log.txt` | Log routing A/B, mỗi câu có nhãn `[prompt-v1]` / `[prompt-v2]` (V1 = 19 câu, V2 = 31 câu) |
| `03_ragas_scores.png` | Bảng so sánh V1 vs V2 |
| `03_ragas_report.json` | Bản sao `data/ragas_report.json` |
| `03_ragas_run_log.txt` | Log đầy đủ lần chạy RAGAS (bổ sung) |
| `04_pii_demo_log.txt` | 6 test case PII (email, phone, SSN, thẻ tín dụng, nhiều loại PII, text sạch) |
| `04_json_demo_log.txt` | 5 test case JSON (hợp lệ, markdown fences, nháy đơn, dấu phẩy thừa, không sửa được) |

## Hai phiên bản prompt

| | V1 — ngắn gọn | V2 — chuyên gia, có cấu trúc |
|---|---|---|
| Vai trò | Trợ lý thân thiện | Chuyên gia phân tích thông tin |
| Độ dài | 2–4 câu | 3–5 câu, xác định facts rồi tổng hợp |
| Grounding | Mọi ý phải có trong context | Chỉ dùng facts trong context, không suy đoán |

Cả hai đều giữ `{context}`, yêu cầu nói "không biết" khi context thiếu, và trả lời cùng ngôn ngữ với câu hỏi. Câu hỏi và knowledge base đều bằng tiếng Anh; nếu trả lời bằng tiếng Việt thì `answer_relevancy` sẽ bị kéo xuống.

## Kết quả RAGAS (50 cặp QA × 2 phiên bản, 0/400 job lỗi)

| Metric | V1 | V2 | Chênh lệch |
|---|---|---|---|
| faithfulness | 0.9557 | **0.9669** | V2 +0.011 |
| answer_relevancy | **0.9148** | 0.9004 | V1 +0.014 |
| context_recall | 1.0000 | 1.0000 | hòa |
| context_precision | 0.9383 | **0.9450** | V2 +0.007 |

Faithfulness ≥ 0.9 ở **cả hai** phiên bản, vượt mục tiêu 0.8.

## Phân tích V1 vs V2

1. **Faithfulness (V2 nhỉnh hơn khoảng 0.01):** RAGAS tách câu trả lời thành các claim rồi kiểm tra từng claim có được context hỗ trợ không. V2 yêu cầu "xác định facts trong context rồi mới viết", nên mô hình bám sát câu chữ của context. V1 ngắn gọn, thân thiện, nên đôi khi tự diễn đạt lại hoặc khái quát hóa; claim kiểu đó dễ bị judge chấm là không được hỗ trợ trực tiếp. Tuy vậy cả hai đều có ràng buộc "chỉ dùng context", nên chênh lệch rất nhỏ.
2. **Answer relevancy (V1 nhỉnh hơn khoảng 0.014):** metric này sinh ngược câu hỏi từ câu trả lời rồi so độ giống (embedding) với câu hỏi gốc. Câu trả lời 2–4 câu của V1 đi thẳng vào trọng tâm. Phần mở rộng 3–5 câu của V2 kéo thêm ý phụ nên câu hỏi sinh ngược hơi lệch, làm điểm giảm.
3. **Context recall / precision:** hai metric này đo **retriever**, mà cả hai phiên bản dùng chung FAISS với k=3. Vì vậy recall bằng nhau tuyệt đối (1.0). Precision lệch khoảng 0.007 là do LLM judge không hoàn toàn ổn định, không phải do prompt.

**Đánh đổi:** V2 bám context chặt hơn một chút, còn V1 trả lời đúng trọng tâm hơn, ngắn hơn và tốn ít token hơn.

## Độ tin cậy của so sánh

Lần chạy đầu bị lỗi kết nối tới OpenAI (33/400 job) vì RAGAS mặc định gửi 16 request song song. Kết quả lần đó là V1 thắng faithfulness (0.975 so với 0.954). Mình đã chạy lại với `RunConfig(max_workers=4, timeout=300)` và không còn job lỗi, nhưng thứ hạng faithfulness lại **đảo chiều**. Như vậy chênh lệch giữa V1 và V2 (≤ 0.02) nằm trong biên độ nhiễu của LLM judge, chưa đủ để khẳng định version nào tốt hơn.

**Kết luận:** về chất lượng, hai prompt tương đương nhau; cả hai đều đạt faithfulness > 0.95. Mình chọn **V1** cho production vì câu trả lời ngắn hơn (ít token, rẻ, nhanh) mà answer relevancy lại cao hơn. Muốn kết luận chắc chắn thì cần chạy lặp nhiều lần hoặc mở rộng bộ câu hỏi, rồi so sánh khoảng tin cậy của hai version.
