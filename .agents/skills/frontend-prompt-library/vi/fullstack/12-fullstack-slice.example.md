---
title: Ví dụ — Bàn giao một Feature Slice Full-Stack hoàn chỉnh
type: example
pairs_with: 12-fullstack-slice.md
---

# 📘 Ví dụ — Bàn giao một Feature Slice Full-Stack hoàn chỉnh

> Một lượt chạy đầy đủ của [`12-fullstack-slice.md`](12-fullstack-slice.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo `react-client-mastery` + `nextjs-server-mastery` (cả hai, bao gồm cả phần SOLID của chúng).
Bàn giao feature `Mời thành viên vào team` từ đầu tới cuối.

USER STORY: Là admin, tôi mời đồng đội bằng email; họ nhận được một link; tôi có thể thu hồi lời mời đang chờ.
STACK: Next.js 15 App Router (RSC + Server Actions), React 19, TypeScript strict, Prisma + Postgres, Zod, TanStack Query v5 cho các client island, react-hook-form, Resend, Upstash rate limit, Vitest + RTL + msw, Playwright
ACCEPTANCE: lời mời xuất hiện trong danh sách tức thì; thu hồi được phản ánh ngay lập tức và vẫn đúng sau khi refresh; chỉ admin mới được mời
NON-GOALS: mời hàng loạt bằng CSV (sprint sau); không làm analytics cho lời mời   ← đừng xây những thứ này. Nếu bạn nghĩ chúng là bắt buộc, hãy hỏi.

REPO CONTEXT: repo đã có SẴN TẤT CẢ các mảnh mà slice này phải khớp vào:
  phân tầng api/hooks/components của `src/features/*` (xem billing, settings);
  `src/lib/http-client.ts` (file duy nhất được phép import axios) với các lỗi có kiểu trong
  `src/shared/api/errors.ts`; pattern ports/adapters/`container.ts` dưới `lib/server/`
  (billing/Stripe); contract `ActionResult<T>` trong `src/shared/api/action-result.ts`;
  `withAuth` trong `src/lib/server/action-wrappers.ts`; msw tại `src/mocks/server.ts` với
  các handler trong `src/mocks/handlers/`. Hãy tái sử dụng từng thứ một — đừng phát minh gì mới.

Làm việc theo GIAI ĐOẠN. DỪNG sau mỗi giai đoạn và chờ tôi phê duyệt.
KHÔNG viết một dòng code hiện thực nào trước khi GIAI ĐOẠN 1 được phê duyệt.

RULE 0 — BÁM REPO TRƯỚC, BÁM PROMPT SAU.
Các đường dẫn trong phần sản phẩm bàn giao bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

──────────────────────────────────────────
GIAI ĐOẠN 1 — THIẾT KẾ (không code)
──────────────────────────────────────────
Hãy tạo ra:
a) Cây file của mọi thứ bạn sẽ tạo/sửa.
b) Với MỖI file, MỘT câu nêu lý do thay đổi duy nhất của nó (SRP).
   Nếu một câu cần dùng chữ "và", hãy tách file đó ra.
c) Contract Zod: entity, input create/update, query params, union lỗi.
d) Các interface port (repository, mailer, gateway) — những trừu tượng mà UI và
   use case sẽ phụ thuộc vào (DIP).
e) Bảng ranh giới RSC/client:
   | Component | Server hay Client | Vì sao | Chính xác các field đi qua boundary |
   Đánh dấu bất kỳ field nào KHÔNG được phép đi qua (secret, nguyên hàng DB) — ISP.
f) Kế hoạch cache & invalidation: cái gì được cache, trong bao lâu, cái gì làm nó hết hạn.
g) Ma trận phân quyền: | Thao tác | Ai được làm | Cưỡng chế ở đâu |
   ("Cưỡng chế trong UI" không phải là một câu trả lời. Nó BẮT BUỘC phải được cưỡng chế ở tầng dữ liệu.)
h) Rủi ro + câu hỏi mở dành cho tôi.

──────────────────────────────────────────
GIAI ĐOẠN 2 — CONTRACT & SERVER
──────────────────────────────────────────
- Zod schema dùng chung (src/shared/api/) — MỘT nguồn chân lý cho FE và BE.
- Port + adapter + container (DIP). `import 'server-only'` trên tất cả chúng.
- Use case: quy tắc nghiệp vụ, deps được tiêm vào, unit-test với port FAKE (không DB).
- Server Action: MỎNG — auth → validate → ủy thác → revalidate.
  Các mối quan tâm cắt ngang qua wrapper (withAuth/withValidation/withRateLimit) — OCP.
  MỘT contract `ActionResult<T>` cho mọi action — LSP.
- Phân quyền được cưỡng chế dựa trên chính TÀI NGUYÊN (quyền sở hữu), không chỉ dựa trên role.
- Log có cấu trúc. Không secret hay stack trace nào trong response.

Cổng chặn: unit test của use case xanh với các fake, không cần database. Cho tôi xem chúng.

──────────────────────────────────────────
GIAI ĐOẠN 3 — TẦNG DỮ LIỆU (client)
──────────────────────────────────────────
- Interface gateway + hiện thực; `axios`/`fetch` gói gọn trong MỘT file (DIP).
- Mọi response được `Schema.parse` tại biên. `as T` BỊ CẤM.
- Query key factory; staleTime/gcTime tường minh; invalidateQueries sau mutation.
- Optimistic update KÈM rollback ở nơi tiêu chí nghiệm thu đòi hỏi phản hồi tức thì
  (lời mời xuất hiện trong danh sách tức thì; thu hồi được phản ánh ngay lập tức).

──────────────────────────────────────────
GIAI ĐOẠN 4 — UI
──────────────────────────────────────────
- Logic trong hook, UI trong component. Các file .tsx CHỈ chứa JSX (SRP).
- State UI chia sẻ được (filter/phân trang/tab) nằm trong URL, không phải useState.
- State dẫn xuất được tính trong lúc render — không đồng bộ state bằng useEffect.
- Component dùng chung: forwardRef + {...rest} (LSP); biến thể bằng cva, không dùng props boolean (OCP).
- Props hẹp; selector slice của store (ISP).
- CẢ BỐN trạng thái: loading (skeleton), lỗi (+retry), rỗng, thành công.
- A11y: role, label, quản lý focus, đường đi bằng bàn phím, aria-live cho cập nhật bất đồng bộ.
- i18n: không có chuỗi hiển thị cho người dùng nào bị hardcode.

──────────────────────────────────────────
GIAI ĐOẠN 5 — TEST & KIỂM CHỨNG
──────────────────────────────────────────
- Unit: use case (port fake), hook (renderHook), Zod schema.
- Tích hợp (RTL + msw): toàn bộ luồng "Là admin, tôi mời đồng đội bằng email; họ
  nhận được một link; tôi có thể thu hồi lời mời đang chờ."; cả bốn trạng thái dữ liệu;
  mutation thất bại + ROLLBACK optimistic.
- Bảo mật: caller ẩn danh bị action từ chối; non-admin bị chặn;
  không thể thay đổi một tài nguyên họ không sở hữu.
- E2E (chỉ khi đây là đường đi trọng yếu).
- Sau đó đi qua các tiêu chí nghiệm thu (lời mời xuất hiện trong danh sách tức thì; thu hồi được
  phản ánh ngay lập tức và vẫn đúng sau khi refresh; chỉ admin mới được mời) từng điểm một
  và cho tôi xem bằng chứng cho từng điểm.

──────────────────────────────────────────
GIAI ĐOẠN 6 — PR
──────────────────────────────────────────
- Conventional Commits: `feat(team-invitations): ...`. Một mối quan tâm cho mỗi commit.
- PR < 400 dòng thay đổi thực. Nếu lớn hơn, hãy tách nó ra và nói cho tôi biết cách tách.
- Nội dung PR: cái gì đã thay đổi, các quyết định về boundary/phân quyền, cái gì bạn ĐÃ KHÔNG làm
  (mời hàng loạt bằng CSV — sprint sau), và cách test thủ công.
- Tự rà soát theo checklist PR của CẢ HAI skill (bao gồm cả 5 mục SOLID mỗi bên)
  và dán checklist đã điền vào.

Bắt đầu với GIAI ĐOẠN 1. Chưa được viết code hiện thực.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
