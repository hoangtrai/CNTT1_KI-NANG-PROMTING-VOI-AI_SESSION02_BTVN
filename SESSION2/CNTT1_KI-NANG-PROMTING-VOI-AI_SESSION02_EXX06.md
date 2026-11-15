


# Sửa lỗi Hệ thống ERP TechShop

## Phần 1 

### 1.1. Diễn biến lỗi

Backend gọi API AI , AI trả về chuỗi văn bản có **lời chào mở đầu** (`Dạ em xin chào anh/chị...`) và **lời chúc kết thúc** (`Hy vọng kết quả này...`) bao quanh khối JSON `JSON.parse()` đọc ký tự đầu tiên là `D` (không phải `{`)  ném lỗi `Unexpected token 'D' in position 0`  toàn bộ pipeline nạp dữ liệu tự động bị ngắt.

### 1.2. Vì sao AI có thói quen tự sinh lời chào / lời chúc?

| Yếu tố | Giải thích |
|---|---|
| **Huấn luyện trên văn bản hội thoại tự nhiên** | Mô hình LLM được tiền huấn luyện và tinh chỉnh (RLHF / SFT) trên các cuộc đối thoại người–máy, trong đó phản hồi "tốt" thường có cấu trúc  |
| **Tính chất next-token prediction** | LLM sinh văn bản theo xác suất token kế tiếp. Khi prompt không ràng buộc mạnh, token có xác suất cao nhất sau câu lệnh thường là các từ giao tiếp xã giao (`Dạ`, `Chào`, `Dưới đây là...`), chứ không phải dấu `{`. |
| **Thiếu ràng buộc output nghiêm ngặt** | Prompt gốc chỉ yêu cầu "trích xuất thành JSON" mà không cấm tuyệt đối các nội dung xung quanh. Mô hình diễn giải yêu cầu theo hướng "trả lời lịch sự + đính kèm JSON". |
| **Ảnh hưởng từ few-shot / system prompt** | Nếu system prompt hoặc ví dụ mẫu chứa văn xuôi giới thiệu, mô hình sẽ bắt chước phong cách đó. |
| **Markdown code-block wrapping** | Nhiều mô hình có xu hướng bọc JSON trong ```json ... ``` để "đẹp mắt", gây lỗi tương tự (ký tự đầu là `` ` ``). |

**Kết luận:** Đây không phải lỗi ngẫu nhiên mà là hành vi mặc định của LLM. Chỉ có thể khắc phục bằng **prompt engineering nghiêm ngặt** (Negative Constraint + định dạng đầu ra cứng) hoặc bổ sung **post-processing** ở backend làm lớp bảo vệ thứ hai.

---

## Phần 2 

### 2.1. System Prompt

```text
Bạn là một bộ trích xuất dữ liệu dạng máy (machine-only data extractor) cho hệ thống ERP của TechShop.

NGUYÊN TẮC TUYỆT ĐỐI:
1. Bạn KHÔNG phải trợ lý hội thoại. Bạn KHÔNG được chào hỏi, giới thiệu, giải thích, xin lỗi, hay chúc tụng bất kỳ điều gì.
2. Bạn CHỈ được trả về DUY NHẤT một chuỗi JSON hợp lệ.
3. Chuỗi trả về BẮT BUỘC bắt đầu bằng ký tự `{` và kết thúc bằng ký tự `}`.
4. KHÔNG được dùng markdown code block (không dùng ```json hay ```).
5. KHÔNG được thêm bất kỳ ký tự, khoảng trắng thừa, dòng trống nào TRƯỚC dấu `{` hoặc SAU dấu `}`.
6. Nếu không trích xuất được một trường nào, để giá trị rỗng `""` — TUYỆT ĐỐI không được trả về văn bản giải thích lý do.

CẤM TUYỆT ĐỐI (Negative Constraints):
- Cấm các từ mở đầu: "Dạ", "Chào", "Xin chào", "Dưới đây là", "Kết quả", "Sure", "Here is", "Certainly", v.v.
- Cấm các câu kết thúc: "Hy vọng", "Chúc", "Nếu cần", "Let me know", v.v.
- Cấm bọc JSON trong markdown.
- Cấm trả về văn bản tự do dưới ANY hình thức.

ĐỊNH DẠNG OUTPUT BẮT BUỘC:
{
  "customer_name": "<họ tên khách hàng, không có thì là chuỗi rỗng>",
  "device": "<tên thiết bị, không có thì là chuỗi rỗng>",
  "issue": "<mô tả lỗi/hư hỏng, không có thì là chuỗi rỗng>"
}
```

### 2.2. User Prompt (gọi mỗi lần trích xuất)

```text
Hãy trích xuất thông tin từ biên bản tiếp nhận bảo hành sau đây.

QUY TẮC ĐẦU RA (đọc kỹ trước khi trả lời):
- Trả về DUY NHẤT chuỗi JSON thuần túy.
- KHÔNG dùng markdown, KHÔNG giải thích, KHÔNG chào hỏi, KHÔNG chúc tụng.
- Bắt đầu bằng `{` và kết thúc bằng `}`. Không có ký tự nào khác ngoài JSON.

Biên bản:
"""
{{NOI_DUNG_BIEN_BAN}}
"""
```

### 2.3. Lớp bảo vệ bổ sung (khuyến nghị backend)

Prompt engineering giảm xác suất lỗi xuống gần 0%, nhưng để đạt **100% parse thành công**, nên bổ sung post-processing:

1. : Bọc trong try-catch + retry với prompt nhắc lại (fallback)
```

### 2.4. Kiểm chứng đầu ra mong muốn

```

  "customer_name": "Trần Văn A",
  "device": "Laptop Dell XPS 15",
  "issue": "Liệt phím Space"
```

 `JSON.parse()` trả về object hợp lệ, pipeline nạp DB chạy liên tục.

