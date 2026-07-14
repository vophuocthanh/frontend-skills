---
title: Ví dụ — Xây dựng trang RSC với Server Data Fetching
type: example
pairs_with: 09-rsc-data-fetching.md
---

# 📘 Ví dụ — Xây dựng trang RSC với Server Data Fetching

> Một lượt chạy đầy đủ của [`09-rsc-data-fetching.md`](09-rsc-data-fetching.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo `nextjs-server-mastery` (+ `react-client-mastery` cho các client island).
Xây dựng trang RSC tại `app/dashboard/page.tsx`.

DỮ LIỆU CẦN: hồ sơ người dùng hiện tại (nhanh), thống kê sử dụng seat (chậm, ~3s), lời mời gần đây (luôn phải tươi mới)
PHẦN TƯƠNG TÁC: bộ chọn khoảng ngày, nút "Export CSV"
CHÍNH SÁCH CACHE: thống kê sử dụng seat cache 1h; lời mời gần đây luôn tươi mới

REPO CONTEXT: `lib/server/` đã có sẵn `db.ts` (singleton của Prisma client) và
  `auth/session.ts` (`getSession()`, đã được bọc trong `React.cache()`), và mọi module server
  đều đã mang sẵn chốt chặn `import 'server-only'` — hãy theo đúng như vậy. Các convention
  `error.tsx` / `loading.tsx` đã tồn tại dưới `app/billing/` — hãy soi chiếu theo chúng
  thay vì phát minh ra vỏ bọc mới.

NON-GOALS: Không thiết kế lại layout dashboard. Không thêm chỉ số mới ngoài seat usage
  (các tile revenue/churn thuộc một ticket khác).

QUY TẮC

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. MẶC ĐỊNH LÀ SERVER.
   - Page và layout là Server Component. KHÔNG `'use client'` ở cấp page/layout.
   - `'use client'` CHỈ đặt ở các lá thực sự cần state, effect,
     event handler, hoặc browser API: bộ chọn khoảng ngày và nút "Export CSV".
   - Bài kiểm tra quyết định cho mỗi component: nó có dùng hook/handler/DOM không? Nếu không → ở lại server.

2. ISP — RSC BOUNDARY LÀ MỘT RANH GIỚI BẢO MẬT.
   - Truy vấn với `select` tường minh — KHÔNG BAO GIỜ `SELECT *`.
   - Truyền PRIMITIVE / object hẹp cho client component:
       ✅ <Chart points={stats.points} currency="USD" />
       ❌ <Chart data={dbRow} />
   - Mọi thứ bạn truyền đều bị SERIALIZE VÀO HTML gửi tới trình duyệt.
     Một hàng DB nguyên vẹn nghĩa là passwordHash và stripeCustomerId giờ đã nằm trong view-source.
   - KHÔNG BAO GIỜ truyền function, class instance, hay object `Date` qua boundary
     (không serialize được — hãy gửi chuỗi ISO).

3. DIP — page KHÔNG được import trực tiếp Prisma/SDK.
   - Page → use case / interface repository → adapter (Prisma, API upstream).
   - `import 'server-only'` ở đầu mọi module server để một import sai phía
     nổ ngay lúc BUILD, chứ không phải lúc 3 giờ sáng trên production.

4. SRP — page thì compose; nó không tính toán.
   - `page.tsx` = fetch (ủy thác) + compose layout. Không có quy tắc nghiệp vụ inline.
   - Quy tắc nghiệp vụ nằm ở `lib/server/use-cases/*`.

5. KHÔNG CÓ WATERFALL.
   - Các fetch độc lập → `Promise.all()`.
   - Khử trùng lặp các truy vấn trong cùng một request bằng `React.cache()` để gọi `getUser()` ở
     layout + page + sidebar chỉ chạm DB MỘT LẦN.
   - Đưa các thao tác đọc file tĩnh (font, config) lên cấp module — KHÔNG BAO GIỜ đặt trong handler.

6. STREAMING — bọc các phần chậm và độc lập trong `<Suspense>` với skeleton fallback
   để một truy vấn chậm không thể chặn cả trang. Theo chính sách cache, quyết định cho từng phần:
   - nhanh + trọng yếu → await ngay trong page (chặn, nhưng nó nhanh) → hồ sơ người dùng hiện tại
   - chậm + không trọng yếu → có `<Suspense>` boundary riêng, stream vào sau → thống kê sử dụng seat ~3s

7. CACHE theo chính sách (thống kê sử dụng seat cache 1h; lời mời gần đây luôn tươi mới):
   - `React.cache()` → khử trùng lặp trong một request
   - `unstable_cache` / `fetch(..., { next: { revalidate } })` → xuyên request
   - Đặt thời gian revalidate TƯỜNG MINH và biện minh cho từng giá trị.
   - Mọi mutation đụng tới dữ liệu này BẮT BUỘC phải `revalidatePath`/`revalidateTag`.

8. LỖI & TRẠNG THÁI:
   - `error.tsx` (client) với `reset()` để retry + `captureException`.
   - `not-found.tsx` cho 404; gọi `notFound()` từ page khi entity không tồn tại.
   - `loading.tsx` hoặc Suspense fallback — skeleton phản chiếu đúng layout thật.
   - Trạng thái rỗng được xử lý tường minh.

9. BẢO MẬT:
   - Xác thực ngay trong page/layout qua `getSession()` — đừng chỉ dựa vào middleware.
   - Secret CHỈ được đọc trong module server. Tiền tố `NEXT_PUBLIC_` = công khai, hãy coi
     bất cứ thứ gì mang nó như thể nó đang được in trên một tấm biển quảng cáo.

SẢN PHẨM BÀN GIAO
  app/dashboard/page.tsx                 (server, compose)
  app/dashboard/loading.tsx
  app/dashboard/error.tsx                ('use client')
  app/dashboard/not-found.tsx
  components/*.tsx                       (server, trình bày)
  components/*.client.tsx                (island 'use client' — bộ chọn khoảng ngày và nút "Export CSV")
  lib/server/use-cases/*.ts              ('server-only')
  lib/server/repositories/*.ts           (interface + adapter)

Trước khi code, in một bảng: | Component | Server hay Client? | Vì sao | và
liệt kê CHÍNH XÁC những field nào đi qua RSC boundary cho từng client island.
Rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
