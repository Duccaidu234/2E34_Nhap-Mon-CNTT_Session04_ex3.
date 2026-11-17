# TỔNG HỢP BÀI TẬP THỰC HÀNH: KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

# PHẦN 1: SO SÁNH TRADE-OFF CÁC GIẢI PHÁP RA LỆNH CHO KỸ THUẬT PROMPTING NÂNG CAO

## 1. Bối cảnh bài toán ứng dụng thực tế
Để minh họa rõ nét nhất cho việc tối ưu hóa quy trình **Chain-of-Thought (Chuỗi suy nghĩ)**, **Prompt Chaining (Nối chuỗi câu lệnh)** và **Xử lý đứt gãy dữ liệu (Exception/Fallback Handling)**, bài làm chọn một bài toán nghiệp vụ cụ thể:
* **Bài toán:** Xây dựng hệ thống AI tự động xử lý phản hồi khách hàng (Customer Support Triage & Resolution).
* **Quy trình chuỗi (Chaining):**
  1. Đọc nội dung khiếu nại $\rightarrow$ Trích xuất thông tin đơn hàng (Mã đơn, Lỗi gặp phải).
  2. Phân tích mức độ nghiêm trọng (Logic CoT).
  3. Đề xuất phương án đền bù hoặc chuyển tiếp bộ phận kỹ thuật.
* **Bẫy dữ liệu / Điểm đứt gãy (Edge Cases):** Khách hàng cung cấp thiếu mã đơn hàng, nội dung khiếu nại mơ hồ/chứa ngôn từ cảm xúc cực đoan, hoặc lỗi nằm ngoài danh mục hỗ trợ tiêu chuẩn.

## 2. Đề xuất 2 Prompt mẫu

### Giải pháp 1: Phương án Đơn giản, Ngắn gọn (Zero-shot / Basic CoT)
Phương án này tập trung vào sự tự nhiên, ít từ khóa rườm rà, dựa vào khả năng suy luận mặc định của mô hình ngôn ngữ lớn (LLM).

> **Prompt Mẫu 1:**
> ```text
> Bạn là nhân viên chăm sóc khách hàng. Hãy đọc email khiếu nại dưới đây và xử lý theo từng bước:
> 1. Tóm tắt vấn đề và tìm mã đơn hàng.
> 2. Suy nghĩ từng bước (Chain-of-Thought) để đánh giá mức độ nghiêm trọng của lỗi.
> 3. Đưa ra phương án giải quyết phù hợp cho khách hàng.
> 
> Nếu thiếu thông tin thì hãy hỏi lại khách hàng.
> 
> Email khách hàng: "[Chèn nội dung email]"
> ```

### Giải pháp 2: Phương án Chi tiết, Chặt chẽ (Structured Prompt Chaining + Fallback Rules)
Phương án này áp dụng kỹ thuật phân tách luồng (Chaining), ép buộc định dạng đầu ra (JSON/XML) để chống đứt gãy khi truyền dữ liệu giữa các bước, và thiết lập các luật chặn lỗi biên (Guardrails) rõ ràng.

> **Prompt Mẫu 2:**
> ```text
> ### VAI TRÒ & MỤC TIÊU
> Bạn là Hệ thống Điều phối Khiếu nại Tự động (Triage AI). Nhiệm vụ của bạn là phân tích email, suy luận logic và xuất dữ liệu chuẩn hóa cho bước xử lý tiếp theo trong chuỗi (Pipeline).
> 
> ### QUY TRÌNH XỬ LÝ CHUỖI (PROMPT CHAINING & CoT)
> Thực hiện tuần tự 3 bước sau và hiển thị rõ quá trình suy luận trong thẻ <thinking>:
> - Bước 1 (Trích xuất): Tìm [Mã đơn hàng] (định dạng DH-XXXX) và [Loại lỗi].
> - Bước 2 (Đánh giá Logic - CoT): Dựa trên [Loại lỗi], phân loại mức độ:
>   + CAO: Lỗi thanh toán, mất hàng, hỏng hóc vật lý.
>   + TRUNG BÌNH: Giao chậm dưới 3 ngày, sai màu/kích thước.
>   + THẤP: Hỏi thông tin chung.
> - Bước 3 (Hành động): Nếu mức độ CAO -> Chuyển chuyên viên cấp 2. Nếu TRUNG BÌNH/THẤP -> Tạo mã giảm giá 10% xin lỗi.
> 
> ### QUY TẮC XỬ LÝ ĐỨT GÃY & BẪY DỮ LIỆU (FALLBACK RULES)
> 1. Bẫy thiếu dữ liệu: Nếu không tìm thấy [Mã đơn hàng] hợp lệ (DH-XXXX), DỪNG quy trình đánh giá đền bù, chuyển trạng thái sang "PENDING_INFO" và soạn câu hỏi yêu cầu bổ sung mã đơn.
> 2. Bẫy nhiễu thông tin: Bỏ qua các từ ngữ cảm xúc/chửi bới, chỉ tập trung vào sự kiện thực tế. Nếu email hoàn toàn vô nghĩa/spam, gán trạng thái "INVALID_INPUT".
> 
> ### ĐỊNH DẠNG ĐẦU RA BẮT BUỘC (JSON)
> Chỉ trả về duy nhất một khối JSON hợp lệ để hệ thống API đọc tiếp, không kèm văn bản thừa:
> {
>   "thinking_process": "Chuỗi suy luận từng bước của bạn",
>   "order_id": "String hoặc null",
>   "severity": "CAO | TRUNG BÌNH | THẤP | UNKNOWN",
>   "status": "RESOLVED | ESCALATED | PENDING_INFO | INVALID_INPUT",
>   "customer_reply": "Nội dung phản hồi khách hàng lịch sự"
> }
> 
> ### DỮ LIỆU ĐẦU VÀO
> Email: "[Chèn nội dung email]"
> ```

## 3. Bảng so sánh Trade-off (Đánh đổi)

| Tiêu chí đánh giá | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết & Ràng buộc) | Phân tích Trade-off (Sự đánh đổi) |
| :--- | :--- | :--- | :--- |
| **Độ dễ hiểu / Dễ viết** | **Cao** (Tự nhiên, dễ đọc, ai cũng có thể viết nhanh) | **Trung bình** (Cấu trúc phức tạp, đòi hỏi tư duy hệ thống và kỹ thuật lập trình) | Đổi sự đơn giản lúc thiết kế lấy tính ổn định khi vận hành thực tế. |
| **Tốn Token (Chi phí)** | **Ít** (Ngắn gọn, tiết kiệm chi phí đầu vào) | **Nhiều hơn** (Tốn token cho các luật, định dạng JSON và thẻ `<thinking>`) | Đổi chi phí token cao hơn một chút để giảm thiểu chi phí sửa lỗi thủ công. |
| **Chặn lỗi biên (Edge Cases)** | **Trung bình** (Dễ bị AI "ảo giác" hoặc tự ý đoán mò khi thiếu dữ liệu) | **Cực tốt** (Có luật Fallback rõ ràng, ép kiểu `null` và trạng thái chờ) | Giải pháp 2 loại bỏ hoàn toàn rủi ro hệ thống bị sập do dữ liệu rác. |
| **Khả năng nối chuỗi (Chaining)** | **Thấp** (Đầu ra là văn bản tự do, khó bóc tách dữ liệu tự động cho bước sau) | **Rất cao** (Đầu ra chuẩn JSON, dễ dàng truyền qua API sang các bước tiếp theo) | Giải pháp 2 là bắt buộc nếu muốn tự động hóa quy trình khép kín. |

## 4. Lựa chọn phương án tối ưu và Chạy thử nghiệm

### Lựa chọn & Giải thích
* **Phương án được chọn:** **Giải pháp 2 (Chi tiết & Chặt chẽ)**.
* **Lý do lựa chọn:** Trong môi trường sản phẩm thực tế (Production) có áp dụng *Prompt Chaining*, tính nhất quán của dữ liệu và khả năng chống đứt gãy quan trọng hơn việc tiết kiệm một lượng nhỏ token. Giải pháp 2 đảm bảo đầu ra luôn đúng định dạng JSON, giúp các bước phía sau hoạt động trơn tru mà không gặp lỗi cú pháp, đồng thời kiểm soát tốt các bẫy dữ liệu từ người dùng.

### Chạy thử nghiệm với Dữ liệu Bẫy (Thiếu mã đơn & Cảm xúc tiêu cực)
* **Dữ liệu đầu vào (Input Email):** *"Làm ăn kiểu gì thế hả?! Tôi đặt cái áo từ tuần trước mà giờ vẫn chưa thấy giao, bực hết cả mình, trả tiền lại cho tôi ngay!"*
* **Kết quả chạy thử từ Giải pháp 2:**
```json
{
  "thinking_process": "1. Trích xuất: Khách hàng khiếu nại giao hàng chậm ('đặt từ tuần trước chưa thấy giao'). Tìm kiếm mã đơn định dạng DH-XXXX -> Không tìm thấy. 2. Đánh giá & Bẫy dữ liệu: Kích hoạt quy tắc Bẫy thiếu dữ liệu do thiếu mã đơn. Bỏ qua thái độ bức xúc của khách hàng. 3. Hành động: Dừng quy trình đền bù, chuyển trạng thái PENDING_INFO và soạn phản hồi xoa dịu, xin mã đơn.",
  "order_id": null,
  "severity": "UNKNOWN",
  "status": "PENDING_INFO",
  "customer_reply": "Chào bạn, chúng tôi thành thật xin lỗi vì sự chậm trễ này đã gây bất tiện cho bạn. Để bộ phận hỗ trợ có thể kiểm tra ngay lộ trình đơn hàng và xử lý kịp thời, bạn vui lòng cung cấp giúp chúng tôi Mã đơn hàng (có định dạng DH-XXXX) nhé!"
}
```

---
---

# PHẦN 2: BÀI TẬP THỰC HÀNH TỔNG HỢP (SESSION 04)

## 1. Bối cảnh bài toán thực tế
* **Mục tiêu:** Xây dựng một hệ thống xử lý tự động gồm 3 bước liên kết khép kín (`Prompt Chaining`) nhằm tiếp nhận yêu cầu phân tích một ý tưởng phần mềm, chia nhỏ logic xử lý qua kỹ thuật `Chain-of-Thought (CoT)` và tự động bắt lỗi dữ liệu dị biệt (Anomaly Handling) nhằm đảm bảo hệ thống không bị crash hoặc sinh ra kết quả sai lệch.
* **Bài toán cụ thể:** Phân tích một ý tưởng ứng dụng web quản lý tài chính cá nhân, bóc tách thành các module kỹ thuật, ước lượng rủi ro và sinh mã nguồn mẫu kèm theo cơ chế phòng thủ dữ liệu đầu vào.

## 2. Thiết lập Chuỗi Prompt liên kết (Prompt Chaining)

Chuỗi gồm 3 bước tuần tự: **Bước 1 (Phân tích Logic & Bóc tách yêu cầu)** $\rightarrow$ **Bước 2 (Chain-of-Thought & Xử lý đứt gãy/dị biệt)** $\rightarrow$ **Bước 3 (Tổng hợp báo cáo và sinh code tối ưu)**.

### Bước 1: Tiếp nhận và Bóc tách ý tưởng (Prompt 1)
```markdown
[System Role]: Bạn là một System Analyst chuyên nghiệp.
[Input]: Hãy nhận thông tin ý tưởng: "Xây dựng ứng dụng Web Quản lý tài chính cá nhân bằng JavaScript thuần, lưu trữ LocalStorage, có biểu đồ thu chi."
[Task]: Bóc tách ý tưởng trên thành các thành phần cốt lõi:
1. Danh sách tính năng bắt buộc (Must-have features).
2. Cấu trúc dữ liệu (Data Schema) mẫu cho giao dịch.
3. Các điểm rủi ro tiềm ẩn về hiệu năng hoặc logic (ví dụ: tràn LocalStorage, sai lệch định dạng tiền tệ).
[Output Format]: Trình bày ngắn gọn dưới dạng Markdown.
```

### Bước 2: Chuỗi suy luận logic & Cơ chế chặn lỗi dị biệt (Prompt 2 - CoT & Anomaly Handling)
```markdown
[System Role]: Bạn là một Senior Software Architect kiêm chuyên gia tối ưu Prompting.
[Context]: Dựa trên kết quả phân tích từ Bước 1, hãy áp dụng kỹ thuật Chain-of-Thought (tư duy từng bước) kết hợp cơ chế chặn lỗi dữ liệu dị biệt (Anomaly & Fallback Handling).
[Task]: 
1. Suy luận từng bước (Step-by-step logic):
   - Bước A: Xử lý sự kiện người dùng nhập số tiền âm hoặc chuỗi ký tự chữ cái thay vì số.
   - Bước B: Xử lý trường hợp LocalStorage bị đầy hoặc bị hỏng cấu trúc (JSON Parse Error).
2. Quy tắc chặn lỗi (Guardrails): Nếu phát hiện dữ liệu đầu vào dị biệt, hệ thống tuyệt đối không được crash (lỗi cú pháp/văng app), mà phải kích hoạt cơ chế Fallback (trả về giá trị mặc định an toàn hoặc thông báo lỗi thân thiện cho người dùng).
[Output Format]: Mô tả chi tiết thuật toán xử lý lỗi và logic từng bước dưới dạng mã giả (Pseudocode) hoặc sơ đồ logic text.
```

### Bước 3: Tổng hợp hoàn thiện và Sinh mã nguồn/Báo cáo cuối cùng (Prompt 3)
```markdown
[System Role]: Bạn là một Lead Full-stack Developer.
[Context]: Tổng hợp toàn bộ kết quả từ Bước 1 và Bước 2.
[Task]: Viết một đoạn mã nguồn mẫu bằng JavaScript (tối ưu token, sạch sẽ, có comment) thực hiện chức năng thêm một giao dịch thu/chi vào LocalStorage, kèm theo toàn bộ logic bắt lỗi dị biệt đã thiết kế ở Bước 2.
[Constraints]: 
- Code sạch, không dư thừa, xử lý triệt để các trường hợp ngoại lệ (try-catch, validate kiểu dữ liệu).
- Trình bày kết quả rõ ràng, chuyên nghiệp.
```

## 3. Ràng buộc kỹ thuật & Xử lý dị biệt (Anomaly Handling)
* **Tối ưu Token:** Các prompt được thiết kế theo cấu trúc phân đoạn rõ ràng (`[System Role]`, `[Context]`, `[Task]`, `[Constraints]`), hạn chế từ ngữ thừa thãi giúp tiết kiệm lượng token đầu vào/đầu ra.
* **Chặn lỗi dữ liệu dị biệt:** 
  * Sử dụng cấu trúc `try...catch` khi thao tác với `JSON.parse()` và `localStorage`.
  * Kiểm tra kiểu dữ liệu (`typeof`, `isNaN`) trước khi thực hiện tính toán số học để ngăn chặn hiện tượng hỏng dữ liệu hệ thống (Data Corruption).

## 4. Kết quả đầu ra sạch lỗi của AI (Mô phỏng thực thi)

Dưới đây là kết quả thực tế thu được sau khi chạy chuỗi prompt hoàn chỉnh:

```javascript
/**
 * Module quản lý giao dịch tài chính cá nhân với cơ chế chặn lỗi dị biệt (Anomaly Handling)
 */
function addTransaction(amount, category, description) {
    try {
        // 1. Kiểm tra và chặn lỗi dữ liệu dị biệt (Guardrails)
        const parsedAmount = Number(amount);
        
        if (isNaN(parsedAmount)) {
            throw new Error("Lỗi dữ liệu: Số tiền nhập vào không phải là một con số hợp lệ.");
        }
        if (parsedAmount <= 0) {
            throw new Error("Lỗi dữ liệu: Số tiền giao dịch phải lớn hơn 0.");
        }
        if (!category || typeof category !== 'string' || category.trim() === "") {
            throw new Error("Lỗi dữ liệu: Danh mục không được để trống.");
        }

        // 2. Chuẩn bị đối tượng dữ liệu
        const transaction = {
            id: 'tx_' + Date.now(),
            amount: parsedAmount,
            category: category.trim(),
            description: description ? description.trim() : "Không có ghi chú",
            createdAt: new Date().toISOString()
        };

        // 3. Đọc và xử lý LocalStorage an toàn (Bắt lỗi JSON hỏng / Tràn bộ nhớ)
        let existingData = [];
        try {
            const rawData = localStorage.getItem('transactions');
            if (rawData) {
                existingData = JSON.parse(rawData);
                if (!Array.isArray(existingData)) {
                    // Nếu dữ liệu bị sai lệch cấu trúc, reset về mảng rỗng an toàn
                    existingData = [];
                }
            }
        } catch (storageReadError) {
            console.warn("Cảnh báo: Dữ liệu LocalStorage bị hỏng, khởi tạo mới bộ nhớ.", storageReadError);
            existingData = [];
        }

        // 4. Lưu dữ liệu
        existingData.push(transaction);
        localStorage.setItem('transactions', JSON.stringify(existingData));

        return {
            success: true,
            message: "Thêm giao dịch thành công!",
            data: transaction
        };

    } catch (error) {
        // 5. Fallback Handler: Đảm bảo hệ thống không bị crash
        console.error("[Anomaly Caught]:", error.message);
        return {
            success: false,
            message: error.message,
            data: null
        };
    }
}
```