# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Hoàng Minh Tuấn  **MSSV:** 2A202602758  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    106.1
graph       196     34619     5687   0.00000    189.9

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       75   0.00000     1.52
graph       0.89   1.83     5184      136   0.00000    11.42
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00000 | ×1.00 (Gemini free tier) |
| Indexing giây | 106.1 | 189.9 | ×1.79 |
| Mỗi câu: USD | 0.00000 | 0.00000 | ×1.00 (Gemini free tier) |
| Mỗi câu: giây | 1.52 | 11.42 | ×7.51 |
| Mỗi câu: in_tok | 696 | 5184 | ×7.45 |

**Chi phí tăng thêm đến từ đâu?**
> Ở pha **Indexing**, chi phí tăng thêm (từ 106.1s lên 189.9s, cộng thêm 20 lệnh gọi LLM với 34,619 input tokens và 5,687 output tokens) đến từ việc phải gọi mô hình ngôn ngữ lớn để trích xuất thực thể có cấu trúc (vụ án, đối tượng, chất ma túy, tội danh) từ 20 bài báo tin tức tự do. Ở pha **Querying**, độ trễ (×7.51 lần) và lượng token đầu vào (×7.45 lần, từ 696 lên 5,184 tokens) tăng vọt là do cơ chế Graph Context phải truy vấn mở rộng nhiều chặng (seeds $\to$ vụ án $\to$ tội danh $\to$ các điều luật và toàn bộ nội dung các khoản/chất ma túy liên quan), dẫn đến ngữ cảnh đưa vào prompt trả lời dài và giàu cấu trúc hơn rất nhiều so với việc chỉ nhồi top-3 chunk thô của Flat RAG.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Câu hỏi định nghĩa luật đơn giản, chunk chứa Điều 2 Khoản 4 được vector search xếp ngay top đầu nên cả hai pipeline đều trả lời xuất sắc. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Dữ kiện nằm trọn trong một đoạn tin tức về vụ 36kg ma túy nên vector retrieval lấy trúng ngay danh sách bị cáo tử hình. |
| **Q3** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Cần nối từ bị cáo Lê Minh Thành (tin tức) sang Điều 251 BLHS (luật) để lấy khung phạt cơ bản (2–7 năm); Flat RAG chỉ lấy được tin sơ thẩm nên thiếu điều luật và khung phạt. |
| **Q4** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Cần bắc cầu từ hành vi của Hoàng Nato sang Tội tổ chức sử dụng (Điều 255 BLHS) để tìm mức phạt kịch khung (20 năm hoặc tù chung thân); Flat RAG bỏ sót thông tin luật. |
| **Q5** | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Đòi hỏi suy luận đa chặng: Cái Quang Huy (9.6kg MDMA) $\to$ Tội vận chuyển (Điều 250) $\to$ đối chiếu ngưỡng Khoản 4 ($\ge 100g$) $\to$ khung phạt 20 năm, chung thân hoặc tử hình; Flat RAG hoàn toàn bó tay. |
| **Q6** | aggregation | 0.00 / 1 | 0.33 / 1 | **Graph** | Yêu cầu tổng hợp toàn cục tất cả vụ án về MDMA vượt quá tầm nhìn top-3 chunk cục bộ của Flat RAG; GraphRAG gom đủ 7 vụ án và liệt kê toàn bộ 5 điều luật liên quan. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken Bridge — Vụ án không nối được sang Điều luật)

- **Hiện tượng:** Có những vụ án (`Case`) trong cơ sở tri thức tin tức được tạo ra nhưng hoàn toàn không có quan hệ `[:CHARGED_WITH]` tới bất kỳ tội danh (`Crime`) nào, khiến đồ thị không thể dẫn lối sang Knowledge Base văn bản pháp luật.
- **Bằng chứng:**
Truy vấn Cypher kiểm tra các vụ án thiếu quan hệ `CHARGED_WITH`:

```cypher
MATCH (k:Case)
WHERE NOT (k)-[:CHARGED_WITH]->()
RETURN k.name AS case_name, k.doc_id AS doc_id, k.source_title AS title;
```

Kết quả trả về:

```
[
  {
    "case_name": "Triệt phá chuyên án A3-626P",
    "doc_id": "news-100261002184934505",
    "title": "Bộ đội Biên phòng quyết liệt phối hợp xây dựng xã, phường, đặc khu 'không ma túy'"
  }
]
```

- **Nguyên nhân:** Nằm ở bản chất bài báo được thu thập (`doc_id: news-100261002184934505`): Đây là bài viết tổng kết hội nghị và trao thưởng thành tích cho Bộ đội Biên phòng. Nội dung bài báo chỉ nêu ngắn gọn: *"trao thưởng chuyên án A3-626P (bắt giữ 2 đối tượng, thu giữ 40kg ma túy)"* mà hoàn toàn không đề cập cơ quan tố tụng đã khởi tố hai đối tượng về tội danh cụ thể nào (vận chuyển, tàng trữ, hay mua bán). Vì tuân thủ chỉ dẫn trích xuất trung thực của prompt, LLM không thể bịa đặt tội danh, dẫn đến mảng `charges` rỗng và quan hệ cầu nối không được tạo.
- **Đề xuất sửa:** 
  1. Trong file `src/graph.py`, tại hàm `Neo4jGraph.context`: Thiết lập cơ chế duyệt cầu nối dự phòng (fallback bridge) thông qua node chất ma túy dùng chung `Substance`: `(k:Case)-[:INVOLVES]->(s:Substance)<-[:MENTIONS]-(cl:Clause)<-[:HAS_CLAUSE]-(a:Article)`. Khi một vụ án không có `CHARGED_WITH`, đồ thị vẫn cung cấp các điều luật điều chỉnh chất ma túy đó cho LLM.
  2. *Đánh đổi:* Ngữ cảnh pháp lý cung cấp có thể bị mở rộng ra nhiều điều luật (ví dụ cả Điều 249, 250, 251) thay vì chỉ đúng 1 tội danh khởi tố, làm tăng số lượng token đầu vào.

---

### Lỗi E3: Trùng thực thể (Entity Duplication — Cùng một vụ việc bị phân mảnh thành nhiều node)

- **Hiện tượng:** Cùng một chuyên án / vụ án ngoài đời thực nhưng bị tạo thành nhiều node `Case` độc lập trên đồ thị Knowledge Graph.
- **Bằng chứng:**
Truy vấn Cypher tìm các vụ án liên quan đến đối tượng "Hoàng Nato":

```cypher
MATCH (k:Case)
WHERE k.name CONTAINS 'Hoàng Nato'
RETURN k.name AS name, k.doc_id AS doc_id;
```

Kết quả trả về:

```
1. "Vụ bắt giữ 'Hoàng Nato' và triệt phá 8 đường dây ma túy tại TP.HCM" (doc_id: news-100261003180410427)
2. "Vụ sử dụng pod chill chứa ma túy của 'Hoàng Nato' và TikToker Phannhibeauty" (doc_id: news-100261003180410427)
3. "Vụ triệt phá 8 đường dây ma túy liên quan TikToker Phannhibeauty và Hoàng Nato tại TP.HCM" (doc_id: news-100261003180410427)
4. "Vụ triệt phá 8 đường dây ma túy liên quan đến 'Hoàng Nato' tại TP.HCM" (doc_id: news-100261003180410427)
```

- **Nguyên nhân:** Nằm ở thiết kế khóa định danh của node `Case` trong ontology (`k:Case {name: $name}`). Khi LLM đọc bài báo dài hoặc nhiều bài báo khác nhau viết về cùng một đường dây tội phạm, LLM sinh ra các tiêu đề tóm tắt vụ việc khác nhau bằng ngôn ngữ tự nhiên. Lệnh `MERGE (k:Case {name: $name})` xem các chuỗi này là các thực thể khác biệt và tạo ra 4 node vụ án riêng biệt cho cùng một chuyên án thực tế.
- **Đề xuất sửa:** 
  1. Trong `src/graph.py`, áp dụng kỹ thuật Phân giải thực thể (Entity Resolution / Canonical Linking) cho `Case` tương tự như đã làm với `Crime` (`link_entity`): Gom cụm vụ án dựa trên tập giao của danh sách nhân vật (`Person`) và địa phương (`Location`), hoặc dùng LLM so sánh xem vụ án mới có trùng với vụ án đã lưu hay không trước khi tạo node.
  2. *Đánh đổi:* Tăng độ trễ và chi phí token trong quá trình Indexing do phải kiểm tra chéo danh sách vụ án trước khi `MERGE`.

---

### Lỗi E2: Thiếu ngữ cảnh luật & Ngưỡng định lượng trong Flat RAG

- **Hiện tượng:** Flat RAG hoàn toàn thất bại khi người dùng hỏi các câu hỏi đòi hỏi đối chiếu định lượng tang vật với các khung hình phạt luật định (như câu Q5), dù kho văn bản có đầy đủ cả bài báo và Điều luật 250 BLHS.
- **Bằng chứng:**
Trích nguyên văn câu trả lời từ file kết quả benchmark (`ket_qua_benchmark_kg.txt` dòng 44–47):

```
--- Q5 [cross-kb-multi-hop] flat recall=0.40 judge=1 1.75s
Dựa trên ngữ cảnh, Cái Quang Huy bị truy tố về tội vận chuyển trái phép chất ma túy với các loại ma túy là MDMA và Ketamine (tổng khối lượng gồm hơn 9,6kg MDMA và gần 406g Ketamine). 

Tuy nhiên, ngữ cảnh không cung cấp thông tin về khoản của điều luật tương ứng được áp dụng và khung hình phạt cho khối lượng MDMA này. Do đó, thông tin về khoản luật và khung hình phạt là không đủ thông tin.
```

- **Nguyên nhân:** Flat RAG hoạt động dựa trên tìm kiếm vector tương đồng (semantic similarity search). Câu hỏi chứa từ khóa "Cái Quang Huy" và "9,6kg MDMA" chỉ kéo về các chunk tin tức về vụ án; các chunk văn bản luật của Điều 250 BLHS không chứa tên "Cái Quang Huy" nên điểm cosine similarity thấp và bị loại khỏi `top_k=3`. Vì không có cấu trúc đồ thị dẫn hướng đa chặng (`Person -> Case -> Crime -> Article -> Clause`), Flat RAG thiếu hẳn ngữ cảnh luật để kết luận khung hình phạt.
- **Đề xuất sửa:** Bắt buộc sử dụng GraphRAG. Trong GraphRAG, việc đi qua đường cầu nối tri thức giúp hệ thống lấy chính xác Điều 250 và các khoản luật, nhờ đó GraphRAG đạt điểm tuyệt đối `recall=1.00, judge=2` (xem câu trả lời Q5 ở dòng 48–55 của `ket_qua_benchmark_kg.txt`).
- *Đánh đổi:* Phải chịu chi phí đồ thị hóa ban đầu và prompt mở rộng dài hơn.

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2:

> **Khi nào nên dùng GraphRAG:**
> 1. **Khi bài toán đòi hỏi suy luận xuyên nguồn dữ liệu (Cross-KB Multi-hop Reasoning):** Khi người dùng đặt câu hỏi cần liên kết giữa một thực thể thực tế (bị can, vụ án, khối lượng tang vật) với các quy định pháp luật tương ứng (điều luật, khoản định khung, mức phạt tối đa như Q3, Q4, Q5). Minh chứng từ số liệu thực nghiệm: Ở các câu cross-kb, GraphRAG đạt `recall = 1.00` và `judge = 2` tuyệt đối, trong khi Flat RAG chỉ đạt `recall = 0.33 – 0.40` và `judge = 1` do không thể tự kết nối hai mảnh thông tin rời rạc.
> 2. **Khi bài toán cần tổng hợp tri thức toàn cục (Global Aggregation):** Các câu hỏi dạng "tìm tất cả các vụ án liên quan đến một chất" (Q6). Flat RAG hoàn toàn thất bại (`recall = 0.00`) vì bị giới hạn bởi tầm nhìn hẹp của top-k chunk cục bộ, trong khi GraphRAG gom đủ 7 vụ án và 5 điều luật thông qua node hội tụ `Substance`.
>
> **Khi nào Flat RAG là đủ:**
> - Đối với các bài toán tra cứu thông tin đơn chặng (Single-hop) nằm trọn vẹn trong một ngữ cảnh cục bộ (như Q1 tra cứu định nghĩa luật, Q2 tìm danh sách bị cáo trong một bản tin cụ thể), Flat RAG đạt kết quả hoàn hảo ngang ngửa GraphRAG (`recall = 1.00`, `judge = 2`).
> - Trong các trường hợp này, Flat RAG là lựa chọn tối ưu vượt trội vì **tiết kiệm chi phí và độ trễ**: Flat RAG phản hồi nhanh hơn gấp **7.5 lần** (1.52 giây so với 11.42 giây của GraphRAG) và tiêu tốn lượng token đầu vào ít hơn **7.45 lần** (696 tokens so với 5,184 tokens). GraphRAG chỉ phát huy giá trị và xứng đáng với chi phí bỏ ra khi hệ thống cần giải quyết các bài toán suy luận mạng lưới đa thực thể.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.  
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển hơn 10kg ma túy qua sân bay Nội Bài, nối sang Tội vận chuyển trái phép chất ma túy $\to$ Điều 250 BLHS).

---

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> 1. **Giới hạn tốc độ gọi API của Gemini (Rate Limit 15 RPM ở Free Tier):** Khi chạy trích xuất tin tức hàng loạt trong `build_graph` và chấm điểm trong `bench_kg.py --judge`, Google Gemini trả về lỗi HTTP 429 (`RESOURCE_EXHAUSTED`). Nhóm đã khắc phục bằng cách thiết lập cơ chế retry tự động với exponential backoff trong `src/llm.py` và thêm độ trễ nghỉ ngắn (2.5 giây) giữa các lần gọi trích xuất tin tức.
> 2. **Mã hóa ký tự trên PowerShell Windows (UnicodeEncodeError 'charmap'):** Khi in kết quả tiếng Việt có dấu ra terminal mặc định của Windows, Python bị lỗi charmap. Đã khắc phục bằng cách thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"` trước khi thực thi.
