---
title: Ví dụ — Xây dựng CRUD với Server Actions
type: example
pairs_with: 10-server-action-crud.md
---

# 📘 Ví dụ — Xây dựng CRUD với Server Actions

> Một lượt chạy đầy đủ của [`10-server-action-crud.md`](10-server-action-crud.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo `nextjs-server-mastery` (+ `react-client-mastery` cho form).
Hiện thực create (mời), revoke (thu hồi), resend (gửi lại) cho Invitation bằng Server Actions.

AUTHZ: chỉ admin mới được mời; một admin có thể thu hồi bất kỳ lời mời nào, một member CHỈ được thu hồi lời mời do chính họ gửi
SIDE EFFECTS: gửi email mời qua Resend khi create/resend; ghi một bản ghi audit-log khi revoke
REVALIDATE: /team, /team/invitations

REPO CONTEXT: `lib/server/ports/` + `adapters/` + `container.ts` ĐÃ tồn tại cho billing
  (một adapter Stripe đã được đấu nối ở đó) — hãy thêm port/adapter của invitation vào cùng
  container đó, đừng dựng thêm cái thứ hai. `ActionResult<T>` ĐÃ được định nghĩa trong
  `src/shared/api/action-result.ts` — hãy import nó, đừng định nghĩa lại. `withAuth` đã
  tồn tại trong `src/lib/server/action-wrappers.ts` — hãy tái sử dụng nó và chỉ thêm những gì
  còn thiếu (withRateLimit / withValidation) bên cạnh nó. Prisma + Resend đã được cấu hình sẵn.

NON-GOALS: Không mời hàng loạt qua CSV. Không cấp phát qua SSO / SCIM. Không làm cron job
  hết hạn lời mời (lời mời quá hạn hiện được xử lý lười (lazily) ngay lúc đọc).

═══ KIẾN TRÚC — mũi tên phụ thuộc luôn hướng VÀO TRONG ═══

  InvitationForm.tsx ('use client', useActionState)
        ↓ gọi
  actions/invitation-actions.ts     ← MỎNG. auth → validate → ủy thác → revalidate
        ↓ gọi
  lib/server/use-cases/*.ts         ← quy tắc nghiệp vụ. Không biết gì về HTTP/FormData.
        ↓ phụ thuộc vào
  lib/server/ports/*.ts             ← INTERFACE do bạn sở hữu (InvitationRepository, Mailer)
        ↑ được hiện thực bởi
  lib/server/adapters/*.ts          ← Prisma, Resend, Stripe (những chi tiết thay thế được)
        ↑ được đấu nối trong
  lib/server/container.ts           ← file DUY NHẤT biết các lớp cụ thể

═══ QUY TẮC ═══

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. SRP — mỗi Server Action làm ĐÚNG bốn việc, theo thứ tự này:
     a) xác thực    → `const session = await getSession()`
     b) validate    → Zod `safeParse` trên input
     c) ủy thác    → gọi use case
     d) revalidate  → `revalidatePath` / `revalidateTag`
   KHÔNG SQL, KHÔNG email, KHÔNG Stripe bên trong action. Nếu action import `db` VÀ
   `resend` VÀ `stripe`, nó là một god-function — hãy tách nó ra.

2. BẢO MẬT — một Server Action là một ENDPOINT CÔNG KHAI:
   - Auth được kiểm tra BÊN TRONG action. Chỉ middleware KHÔNG phải là biện pháp bảo vệ.
   - Phân quyền theo các quy tắc ở trên (chỉ admin mới được mời; một admin có thể thu hồi bất kỳ
     lời mời nào, một member CHỈ được thu hồi lời mời do chính họ gửi) được kiểm tra
     dựa trên chính TÀI NGUYÊN, không chỉ dựa trên role
     (một người dùng đã đăng nhập không được phép thu hồi lời mời của người khác bằng cách
     truyền một id khác — đây là bug Server Action phổ biến số 1 trong thực tế).
   - MỌI input đều được Zod validate. `formData.get('role') as string` BỊ CẤM —
     đó là một vector leo thang đặc quyền.
   - Rate-limit cho các thao tác ghi và các action auth.
   - KHÔNG BAO GIỜ trả về chi tiết lỗi nội bộ, stack trace, hay lỗi DB về client.

3. LSP — MỘT contract kết quả cho MỌI action, không ngoại lệ:
     export type ActionResult<T> =
       | { ok: true; data: T }
       | { ok: false; fieldErrors?: Record<string, string[]>; message?: string };
   Các thất bại được lường trước (validation, conflict) TRẢ VỀ `{ ok: false }`.
   Chỉ những trường hợp thực sự ngoại lệ (unauthorized, hạ tầng sập) mới THROW ra error boundary.
   Một action KHÔNG BAO GIỜ được throw ở nơi mà action khác trả về — điều đó khiến `useActionState`
   không dùng được và buộc bên gọi phải xử lý đặc biệt cho từng action.

4. OCP — các mối quan tâm cắt ngang được COMPOSE, không copy-paste:
     export const revokeInvitation = withAuth({ role: 'admin' })(
       withRateLimit({ limit: 5, window: '1m' })(
         withValidation(RevokeInvitationSchema)(async (input, ctx) => { ... })));
   Thêm audit logging về sau BẮT BUỘC chỉ là thêm MỘT wrapper, không phải sửa N action.

5. DIP — use case phụ thuộc vào PORT, KHÔNG BAO GIỜ vào Prisma/Resend/Stripe:
     export async function createInvitationUseCase(
       input: CreateInvitationInput,
       deps = { invitationRepo, mailer, audit },   // được tiêm vào → unit-test được với fake
     ) { ... }
   `import 'server-only'` trên mọi port, use case và adapter.

6. LSP cho repository — bản fake và bản thật phải thất bại Y HỆT nhau:
     findById(id): Promise<Invitation | null>   // null khi không tìm thấy — KHÔNG BAO GIỜ throw
     create(input): Promise<Invitation>         // throw ConflictError khi trùng lặp
   Một fake trả về null ở nơi Prisma throw = test xanh, 500 trên production.

7. ISP — chỉ `select` các cột cần thiết. Trả về một DTO HẸP từ action,
   không phải hàng DB. Bất cứ thứ gì action trả về đều bị serialize về client.

8. SIDE EFFECT (gửi email mời qua Resend khi create/resend; ghi một bản ghi audit-log
   khi revoke) nằm trong use case, phía sau port của riêng chúng
   (Mailer, AuditLog). Chúng KHÔNG được phép làm hỏng cả mutation một cách lặng lẽ —
   hãy quyết định và nêu tường minh: email là giao dịch (rollback khi thất bại)
   hay fire-and-forget (log lại rồi đi tiếp)?

9. FORM (client island):
   - `useActionState` gắn với action; tái sử dụng CÙNG Zod schema ở phía client
     với react-hook-form để có phản hồi tức thì (progressive enhancement:
     nó vẫn phải hoạt động khi tắt JS).
   - Map `fieldErrors` từ ActionResult ngược lại vào các field qua `setError`.
   - `useFormStatus` cho trạng thái pending; vô hiệu hóa nút submit khi đang pending.
   - Thành công/lỗi được thông báo trong một vùng `aria-live`.

10. CACHE — mọi mutation đều revalidate /team và /team/invitations. Quên điều này là bug Server Action
    phổ biến nhất: lệnh ghi thành công, UI hiển thị dữ liệu cũ,
    và người dùng bấm lại nút lần nữa.

11. LOGGING — log có cấu trúc (`logger.info({ action, userId, requestId })`),
    KHÔNG BAO GIỜ `console.log`. Bắt exception kèm ngữ cảnh.

═══ SẢN PHẨM BÀN GIAO ═══
  src/shared/api/invitation/invitation-schema.ts   (dùng chung FE+BE)
  src/lib/server/ports/invitation-repository.ts
  src/lib/server/ports/mailer.ts
  src/lib/server/adapters/prisma-invitation-repository.ts
  src/lib/server/adapters/resend-mailer.ts
  src/lib/server/container.ts
  src/lib/server/use-cases/create-invitation.ts   (+ revoke / resend)
  src/lib/server/action-wrappers.ts               (withAuth, withValidation, withRateLimit)
  src/actions/invitation-actions.ts
  src/features/invitation/components/InvitationForm.tsx
  test: use case với port FAKE (không DB); action cho authz + từ chối khi validation sai

Bắt đầu bằng việc in các interface port và contract `ActionResult<T>`,
cộng với một câu mô tả lý do thay đổi duy nhất của từng tầng.
Rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
