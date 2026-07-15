---
title: Ví dụ — Định nghĩa API Contract (Contract-First)
type: example
pairs_with: 07-api-contract.md
---

# 📘 Ví dụ — Định nghĩa API Contract (Contract-First)

> Một lượt chạy đầy đủ của [`07-api-contract.md`](07-api-contract.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo `react-client-mastery` + `nextjs-server-mastery`. Định nghĩa API contract cho User và Invitation.

NGUỒN CHÂN LÝ: OpenAPI/Swagger spec của backend Acme Console tại https://api.acme-console.dev/openapi.json (lỏng hơn nhu cầu của UI: `role` khai báo là chuỗi tự do, `email` khai báo `nullable: true`)
ENDPOINTS: GET /users, GET /users/:id, POST /invitations, GET /invitations, POST /invitations/:id/revoke
AUTH: Bearer JWT trong httpOnly cookie

REPO CONTEXT: `src/shared/api/` đã chứa contract của billing theo đúng hình dạng này
  (`billing-schema.ts` + `billing-errors.ts` + `billing-contract.ts`) — hãy làm tương tự cho
  user và invitation. Union lỗi có kiểu đã nằm sẵn ở `src/shared/api/errors.ts`
  (AppError + mapping status→error); hãy MỞ RỘNG nó, đừng định nghĩa một kiểu lỗi thứ hai.
  Zod đã là dependency sẵn có và đang được dùng ở cả client lẫn server.

NON-GOALS: KHÔNG hiện thực các endpoint hay bất kỳ UI nào — đây chỉ là phần contract.
  Không migrate sang GraphQL. Không thay đổi chính OpenAPI spec của backend.

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
     - UserSchema / InvitationSchema                     → entity đầy đủ đúng như API trả về
     - UserListItemSchema / InvitationListItemSchema     → hình dạng rút gọn mà endpoint list trả về
     - CreateUserSchema / CreateInvitationSchema         → request body cho POST (không id, không timestamp)
     - UpdateUserSchema / UpdateInvitationSchema         → thường là Create...Schema.partial()
     - UserQuerySchema / InvitationQuerySchema           → query params (page, search, sort, filters)
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
       User.role là z.enum(['admin', 'member', 'viewer']).
       Invitation.status là z.enum(['pending', 'accepted', 'revoked']).

7. Một vị trí duy nhất, dùng chung CẢ HAI phía:
     src/shared/api/user/
     ├── user-schema.ts             (schema + type được suy ra)
     ├── user-errors.ts             (union lỗi có kiểu + mapping HTTP)
     └── user-contract.ts           (bản đồ endpoint: method, path, input, output schema)
     src/shared/api/invitation/
     ├── invitation-schema.ts       (schema + type được suy ra)
     ├── invitation-errors.ts       (union lỗi có kiểu + mapping HTTP)
     └── invitation-contract.ts     (bản đồ endpoint: method, path, input, output schema)

SẢN PHẨM BÀN GIAO
- Các file schema ở trên.
- Một bảng markdown mô tả contract:
  | Endpoint | Method | Input schema | Output schema | Errors | Auth |
- Danh sách MỌI GIẢ ĐỊNH bạn buộc phải đưa ra vì OpenAPI spec của Acme Console mơ hồ.
  Tôi sẽ xác nhận chúng với backend trước khi chúng ta xây dựng tiếp lên trên.

Nguồn là một OpenAPI spec, nên hãy dẫn xuất schema từ nó một cách trung thành và
đánh dấu mọi chỗ mà spec lỏng lẻo hơn nhu cầu thực tế của UI
(ví dụ `nullable: true` trên field `email`, thứ mà UI coi là bắt buộc,
và `role` khai báo là chuỗi tự do trong khi UI chỉ hỗ trợ admin|member|viewer).

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
