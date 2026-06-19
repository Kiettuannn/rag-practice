- Đây là các note lại được ghi ra từ tài liệu của AIO VIETNAM
- Mục đích để học và ghi ra thì nhớ hơn thui

# 1. Kiến trúc RAG hiện đại

![image.png](image.png)

## 1.1 Phase 1: Indexing

- Giống như quy trình ETL
- Mục tiêu: raw data → 1 dạng data thống nhất
- Các bước cơ bản:
  1. **Document Loading:**

     ![image.png](image%201.png)
     - **Trích xuất nội dung**
     - **Thu thập siêu dữ liệu:** trích xuất thôg tin metadata
       - vai trò của metadata ví dụ: Trong tính năng pre- filtering người dùng hỏi về “Doanh thu năm 2024”, hệ thống sẽ dùng metadata để lọc đúng các tài liệu trong năm 2024 thay vì tìm kiếm toàn bộ dữ liệu.Documents
     1. **Text Splitting (Chunking):**
        1. Tại sao cần nó?
           - Giới hạn context window: Các LLM và embedding model có thể bị giới hạn token đầu vào. Ko thể đưa toàn bộ nội dung vào cùng 1 lúc
           - Độ chính xác: Vector của 1 đoạn ngắn → tập trung ý cụ thể hơn là vector 1 đoạn dài
        2. **Fixed-size chunking:**
           - Chia VB dựa trên kí tự hoặc token cố định
           - Đơn giản
           - Dễ làm mất ngữ nghĩa nếu cắt vào giữa câu hoặc 1 ý đang trình bày
        3. **Recursize chunking**
           - Cắt Dựa trên cấu trúc tự nhiên của VB
           - Thứ tự ưu tiên: dấu ngắt đoạn (\n\n) → xuống dòng (\n) → dấu chấm → khoảng trắng
        4. **Chunk overlap**
           - Để đảm bảo ngữ nghĩa ko bị mất tại điểm cắt giữa 2 chunk liền kề
           - thiết lập tham số chunk_overlap = 10 → 20% độ dài chunk
           - Ví dụ:
             ![image.png](image%202.png)

        **3. Chiến lược indexing nâng cao**

        ![image.png](image%203.png)

## 1.2 Phase 2: Retrieval

### 1. Query Processing:

- Nhận câu hỏi từ user → đưa qua embedding model → tạo ra query vector q
- Tuy nhiên, trong thực tế thì câu hỏi của user thường ngắn, thiếu ngữ cảnh.
- Cách khắc phục:
  - **Multi-Query:**
    - Dùng LLM sinh ra 3 - 5 biến thể câu hỏi
    - Dùng các câu đó tìm kiếm → gộp kết quả lại
    ![image.png](image%204.png)
  - **HyDE (Hypothetical Document Embeddings)**
    - Yêu cầu LLM viết ra câu trả lời giả định cho câu hỏi
    - Sau đó dùng vector embedding của câu trả lời giả định này để tìm kiếm
    ![image.png](image%205.png)

### 2. Similarity search

- Dense Retrieval: Hệ thống tính toán độ tương đồng giữa q (query) và các vector tài liệu d trong CSDL
  ![image.png](image%206.png)
- Algorithm:
  - Đảm bảo tốc độ khi dữ liệu lớn
  - Thay vì so khớp từng đôi một: Brute-force
  - Ta dùng thuật toán: ANN (Approximate Nearest Neighbor) như HNSW để tìm lân cận gần đúng với độ trễ thấp
- **Vấn đề:**
  - Tìm kiếm vector rất tốt về ngữ nghĩa
  - Nhưng yếu về từ khoá chính xác
  - Vd: iphone 14 và iphone 15 gần sematic nhưng user chỉ muốn iphone 14

### 3. Hybird search

- Cơ chế:
  - Dense retrieval (vector search): tìm kiếm dựa trên độ tương đồng
  - Sparse Retrieval (keyword): Sử dụng thuật toán như BM25, TF-IDF
    - Các thuật toán này hoạt động = cách đếm tần số xuất hiện của chữ chính xác của keyword
- Reciprocal Rank Fusion (RRF):
  - Là thuật toán hậu xử lí dùng để gộp và xếp hạng lại kết quả từ 2 luồng trên
  ![image.png](image%207.png)

### 4. Re-ranking

- Sau khi có dc tập ứng viên (vd top 50 tài liệu) từ bước trên, thứ tự của chúng có thể chưa thể hoàn toàn chính xác do vector chỉ là dạng nén thông tin
- **Cross encoder**
  - Sử dụng mô hình DL chuyên biệt để chấm điểm lại mức độ liên quan giữa câu hỏi và tài liệu trong tập ứng viên ban đầu
  - Tại sao cần nó???
    - **Bi-encoder:**
      - Dùng ở bước indexing/retrieval
      - Mã hoá câu hỏi và tài liệu thành 2 vector riêng, độc lập
      - Ưu điểm: nhanh
      - Nhược: mất mối quan hệ ngữ pháp và ngữ nghĩa phức tạp giữa câu hỏi và văn bản
    - **Cross-Encoder:**
      - Dùng ở bước re-ranking
      - Đưa câu hỏi + vb vào mô hình cùng lúc (như con người đọc song song)
      - Nhận biết đc sắc thái phủ định, quan hệ nguyên nhân - kết quả phức tạp
      - Chính xác hơn → chậm hơn
- **Chiến lược “Hình phễu”:**
  - Retrieve many: lấy nhanh 50 tài liệu bằng bi-encoder → re-rank few : lấy 5 doc tốt nhất bằng cross-encoder → Đưa vào LLM
    ![image.png](image%208.png)
    ![image.png](image%209.png)

## 1.3 Phase 3: Generation

### 1. Context Preparation

- Context Stuffing:
  - Đơn giản nhất
  - gộp toàn bộ docs tìm dc(top-k) ⇒ thành 1 đoạn văn dài ⇒ đưa vô prompt
- Vấn đề:
  - Chi phí cao
  - độ trễ cao
  - nhiễu thông tin
- **Context Selection và Compression**
  - Context reordering:
    - dựa trên hiện tượng “Lost in the middle”
    - LLM thường chú ý đầu và cuối, quên giữa
    ![image.png](image%2010.png)
  - Context Compressing:
    - Sử dụng LLM nhỏ hoặc thuật toán NLP ⇒ tóm tắt ý chính ⇒ Đưa cho LLM
    ![image.png](image%2011.png)

### 2. Prompt Engineering

- **Zero-short Prompting:**
  - Sử dụng 1 template cố định để hướng dẫn mô hình trả lời dựa trên ngữ cảnh mà ko cần ví dụ mẫu
  ![image.png](image%2012.png)
- **Few-shot Learning**
  - Cung cấp thêm 1 - 2 ví dụ mẫu
  - gồm bộ 3: context, question, answer chuẩn
  - Đưa vào prompt để llm học và trả lời theo mẫu đó
    ![image.png](image%2013.png)
- Chain-of Thought (CoT - Chuỗi suy luận)
  - Yêu cầu mô hình suy nghĩ từng bước dựa trên context trước khi đưa ra kết quả
    ![image.png](image%2014.png)

## 3. Generation & Attribution

- Cơ bản:
  - LLM sinh ra câu trả lời dưới dạng văn bản thông thường
  - Mục tiêu: tính trôi chảy và đúng ngữ pháp
- Citation: 1 trong những lợi thế lớn nhất của rag so với chatbot truyền thống
  - Cơ chế: Trong prompt, ta yêu cầu: “Mọi thông tin đưa ra phải kèm them ID của nguồn tài liệu”
  - Lợi ích: Giúp người dùng dễ dàng kiểm chugws, xây dựng niềm tin, giảm thiểu rủi ro khi mô hình bịa đặt thông tin

# 2. Thư viện LangChain

![image.png](image%2015.png)

![image.png](image%2016.png)

![image.png](image%2017.png)

![image.png](image%2018.png)

![image.png](image%2019.png)

![image.png](image%2020.png)

# 3. Thực hành

## 1. System overview

![image.png](image%2021.png)
