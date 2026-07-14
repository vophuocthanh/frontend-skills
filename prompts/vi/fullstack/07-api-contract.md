---
title: Định nghĩa API Contract (Contract-First)
category: FE + BE
skills: [react-client-mastery, nextjs-server-mastery]
principles: [ISP, DIP, LSP]
---

# 📜 Prompt — Định nghĩa API Contract

> **Chạy prompt này TRƯỚC mọi prompt tích hợp.** Contract là đường ranh giới giữa FE và BE.
> Mọi thứ phía sau (client hook, Server Action, test, mock) đều được sinh ra từ nó.
> Một Zod schema, hai bên tiêu thụ. KHÔNG BAO GIỜ viết type bằng tay.

---

```text
Tuân theo `react-client-mastery` + `nextjs-server-mastery`. Định nghĩa API contract cho {{RESOURCE}}.

NGUỒN CHÂN LÝ: {{SPEC_SOURCE}}
ENDPOINTS: {{ENDPOINTS}}
AUTH: {{AUTH}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

QUY TẮC

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. ZOD CHÍNH LÀ CONTRACT. Mọi type đều được SUY RA (inferred), KHÔNG BAO GIỜ viết tay:
     export const UserSchema = z.object({...});
     export type User = z.infer<typeof UserSchema>;
   Một `interface User` viết tay trùng lặp với schema là một bug — nó SẼ lệch pha (drift).

2. Tách schema theo chiều dữ liệu và theo use case (ISP — đừng đẩy ra một type béo phì):
     - {{RESOURCE}}Schema          → entity đầy đủ đúng như API trả về
     - {{RESOURCE}}ListItemSchema  → hình dạng rút gọn mà endpoint list trả về
     - Create{{RESOURCE}}Schema    → request body cho POST (không id, không timestamp)
     - Update{{RESOURCE}}Schema    → thường là Create...Schema.partial()
     - {{RESOURCE}}QuerySchema     → query params (page, search, sort, filters)
   Dẫn xuất bằng `.pick()` / `.omit()` / `.partial()` / `.extend()` — KHÔNG copy-paste
   các object literal giữa các schema.

3. KHÔNG BAO GIỜ lộ secret trong response schema. Hãy `.omit()` tường minh các field như
   passwordHash, internal id, stripeCustomerId. Cái gì nằm trong schema là cái đó
   đi qua dây — và, trong RSC, là cái được serialize vào payload HTML.

4. Envelope + lỗi (LSP — mọi endpoint đều thất bại theo CÙNG một cách):
     - Định nghĩa MỘT response envelope và MỘT hình dạng lỗi dùng chung cho mọi endpoint.
     - Định nghĩa envelope phân trang MỘT lần:
         PaginatedSchema(item) → { data: item[], meta: { page, pageSize, total } }
     - Định nghĩa union lỗi của app: 400 validation / 401 / 403 / 404 / 409 conflict / 5xx.
     - Map HTTP status → một lớp domain error có kiểu. Bên gọi KHÔNG BAO GIỜ được tự soi
       các mã status thô rải rác khắp codebase.

5. Validate ở biên là BẮT BUỘC ở CẢ HAI phía:
     - FE: `Schema.parse(response)` ở mọi lần đọc. `as User` BỊ CẤM.
     - BE (Server Action / route handler): parse INPUT. KHÔNG BAO GIỜ tin client.
   Nếu contract backend chưa ổn định, dùng `.safeParse()` ở biên và
   báo cáo lỗi parse như một trạng thái lỗi hạng nhất — đừng lặng lẽ đi tiếp.

6. Ngày tháng/tiền tệ/enum:
     - Ngày: `z.string().datetime()` trên đường truyền; chỉ chuyển sang `Date` ở tầng
       ứng dụng. KHÔNG BAO GIỜ gửi object `Date` qua RSC boundary (không serialize được).
     - Tiền: số nguyên đơn vị nhỏ nhất + mã tiền tệ. KHÔNG BAO GIỜ dùng float.
     - Enum: `z.enum([...])` — KHÔNG BAO GIỜ dùng `z.string()` trần.

7. Một vị trí duy nhất, dùng chung CẢ HAI phía:
     src/shared/api/{{RESOURCE}}/
     ├── {{RESOURCE}}-schema.ts     (schema + type được suy ra)
     ├── {{RESOURCE}}-errors.ts     (union lỗi có kiểu + mapping HTTP)
     └── {{RESOURCE}}-contract.ts   (bản đồ endpoint: method, path, input, output schema)

SẢN PHẨM BÀN GIAO
- Các file schema ở trên.
- Một bảng markdown mô tả contract:
  | Endpoint | Method | Input schema | Output schema | Errors | Auth |
- Danh sách MỌI GIẢ ĐỊNH bạn buộc phải đưa ra vì {{SPEC_SOURCE}} mơ hồ.
  Tôi sẽ xác nhận chúng với backend trước khi chúng ta xây dựng tiếp lên trên.

Nếu {{SPEC_SOURCE}} là một OpenAPI spec, hãy dẫn xuất schema từ nó một cách trung thành và
đánh dấu mọi chỗ mà spec lỏng lẻo hơn nhu cầu thực tế của UI
(ví dụ `nullable: true` trên một field mà UI coi là bắt buộc).

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
