---
title: Ví dụ — Luồng xác thực đầu-cuối (End-to-End)
type: example
pairs_with: 11-auth-flow.md
---

# 📘 Ví dụ — Luồng xác thực đầu-cuối

> Một lượt chạy đầy đủ của [`11-auth-flow.md`](11-auth-flow.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo `nextjs-server-mastery` + `react-client-mastery`. Hiện thực luồng auth.

MODEL: email+password với JWT access (15m) + refresh (7d), có xoay vòng refresh token
PROVIDER: backend tự xây (Acme Console API, Prisma + Postgres)
PROTECTED: /dashboard/*, /team/*; /admin/* yêu cầu role=admin
PUBLIC: /, /login, /pricing

REPO CONTEXT: session ĐÃ được đọc qua `lib/server/auth/session.ts` (`getSession()`,
  được bọc trong `React.cache()`) — hãy mở rộng module đó, đừng thêm một session helper song song.
  `middleware.ts` đã tồn tại và hiện tại chỉ làm redirect — hãy giữ nguyên như vậy.
  `withAuth` đã tồn tại trong `src/lib/server/action-wrappers.ts`. CHƯA có
  xoay vòng refresh token: đó chính là thứ chính mà task này bổ sung.

NON-GOALS: Không OAuth / đăng nhập mạng xã hội. Không 2FA. Không magic link — cả ba đều thuộc quý sau.

═══ QUY TẮC BẢO MẬT KHÔNG THỎA HIỆP ═══

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. LƯU TRỮ TOKEN — cookie httpOnly, Secure, SameSite=Lax, được đặt BỞI SERVER.
   - KHÔNG BAO GIỜ localStorage hay sessionStorage. Bất kỳ XSS trên bất kỳ trang nào cũng cuỗm được token.
   - KHÔNG BAO GIỜ đặt token vào cookie client đọc được hay phơi bày nó qua `NEXT_PUBLIC_*`.
   - Code phía client KHÔNG BAO GIỜ được có khả năng ĐỌC access token. Nó không cần đọc.

2. MIDDLEWARE KHÔNG PHẢI LÀ PHÂN QUYỀN. Nó chỉ là một tối ưu hóa redirect, không hơn.
   - `middleware.ts` có thể redirect người dùng chưa xác thực ra khỏi /dashboard/*, /team/*
     và /admin/*.
   - MỌI Server Action, route handler và trang RSC BẮT BUỘC phải TỰ MÌNH xác minh
     session bằng `getSession()`. Server Action là endpoint công khai — một middleware
     redirect không ngăn được một lệnh POST trực tiếp tới chúng.
   - Phân quyền (role/quyền sở hữu) được cưỡng chế ở tầng DỮ LIỆU, không phải trong UI.
     Ẩn một cái nút là UX. Đó không phải là bảo mật.

3. XÁC MINH, ĐỪNG GIẢI MÃ. Luôn xác minh chữ ký và hạn dùng ở phía server
   (`jwtVerify`). KHÔNG BAO GIỜ tin một payload đã giải mã từ client.

═══ KIẾN TRÚC (SRP + DIP) ═══

  lib/server/auth/session.ts    'server-only' — getSession(), createSession(), destroySession()
  lib/server/ports/auth-gateway.ts   interface AuthGateway { login, refresh, logout }
  lib/server/adapters/*.ts      phần hiện thực cho backend tự xây (Acme Console API)
  actions/auth-actions.ts       mỏng: validate → ủy thác → set cookie → redirect
  middleware.ts                 chỉ redirect. Không gọi DB, không crypto nặng.
  components/LoginForm.tsx      'use client' — useActionState

- DIP: use case đăng nhập phụ thuộc vào `AuthGateway`, không phải vào client/SDK của Acme Console API.
  Đổi provider KHÔNG được đụng tới UI hay use case.
- SRP: quản lý session, xác minh thông tin đăng nhập, và form là ba module riêng biệt.

═══ CÁC LUỒNG CẦN HIỆN THỰC ═══

A. ĐĂNG NHẬP
   - Zod validate thông tin đăng nhập (schema dùng chung, FE + BE).
   - Rate-limit theo IP VÀ theo email (chống brute-force).
   - Khi thành công: đặt httpOnly cookie ở phía server, rồi `redirect()`.
   - Khi thất bại: trả về `{ ok: false, message: 'Invalid email or password' }`.
     Dùng CÙNG một thông điệp chung chung cho sai-email và sai-password — KHÔNG BAO GIỜ tiết lộ
     cái nào sai (user enumeration).
   - Trả về `ActionResult<T>` chuẩn (LSP — cùng contract với mọi action khác).
   - Về thời gian: đường đi thất bại không được nhanh hơn đường đi thành công một cách đo đếm được.

B. ĐỌC SESSION
   - `getSession()` được bọc trong `React.cache()` → gọi ở layout, page và action
     nhưng chỉ xác minh MỘT LẦN cho mỗi request.

C. REFRESH
   - Xử lý ở phía server. Nếu HTTP client phía client phải xử lý 401,
     nó làm việc đó TẬP TRUNG: xếp hàng các 401 đồng thời → refresh MỘT LẦN → phát lại hàng đợi →
     đăng xuất cứng nếu chính lần refresh đó thất bại.
   - Xoay vòng (rotate) refresh token khi dùng; phát hiện việc tái sử dụng một token đã bị xoay vòng (dấu hiệu bị đánh cắp).

D. ĐĂNG XUẤT
   - Server Action: hủy session ở phía server (vô hiệu hóa refresh token),
     xóa cookie, `redirect('/login')`.
   - Xóa cache phía client (`queryClient.clear()`) để người dùng tiếp theo không thấy
     dữ liệu đã cache của người dùng trước.

E. BẢO VỆ ROUTE
   - middleware.ts → redirect cho /dashboard/*, /team/* và /admin/*.
   - Mọi trang được bảo vệ → `const session = await getSession(); if (!session) redirect('/login')`.
   - Mọi action → có kiểm tra auth riêng của nó (dùng wrapper `withAuth` — OCP).

F. NGỮ CẢNH USER PHÍA CLIENT
   - USER của session (id, name, role, avatar — KHÔNG BAO GIỜ là token) được truyền từ
     server layout vào một client Context provider để DI (đọc tần suất thấp).
   - ISP: chỉ truyền những field UI cần. KHÔNG BAO GIỜ truyền nguyên hàng user.

═══ SẢN PHẨM BÀN GIAO ═══
  middleware.ts
  lib/server/auth/session.ts
  lib/server/ports/auth-gateway.ts
  lib/server/adapters/acme-backend-auth-gateway.ts
  lib/server/use-cases/login.ts
  actions/auth-actions.ts
  components/LoginForm.tsx
  providers/session-provider.tsx    (client Context — DI, chỉ thông tin user)
  test: đăng nhập thành công/thất bại, rate limit, action được bảo vệ từ chối caller ẩn danh,
        non-admin bị chặn khỏi một admin action, đăng xuất xóa sạch cache

═══ CHECKLIST BẢO MẬT — trả lời tường minh từng mục trước khi code ═══
  [ ] Access token được lưu ở đâu, và có bất kỳ JS phía client nào đọc được nó không?
  [ ] Chuyện gì xảy ra nếu ai đó POST thẳng tới một Server Action, bỏ qua UI?
  [ ] Một non-admin đã đăng nhập có thể gọi một admin action bằng cách tự chế request không?
  [ ] Một lần đăng nhập thất bại có tiết lộ email đó có tồn tại hay không?
  [ ] Refresh token có được xoay vòng không? Chuyện gì xảy ra nếu một token cũ được phát lại?
  [ ] Lỗi auth có được log kèm ngữ cảnh nhưng trả về cho người dùng một cách chung chung không?

Trả lời checklist trước. Rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
