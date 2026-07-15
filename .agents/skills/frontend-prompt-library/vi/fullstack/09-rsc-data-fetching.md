---
title: Xây dựng trang RSC với Server Data Fetching
category: FE + BE
skills: [nextjs-server-mastery, react-client-mastery]
principles: [SRP, ISP, DIP]
---

# 🖥️ Prompt — Xây dựng trang RSC với Server Data Fetching

> Dùng cho một trang Next.js App Router fetch dữ liệu trên **server** (DB hoặc API upstream) rồi stream về client.
> Hai kiểu thất bại mà prompt này sinh ra để ngăn chặn: **rò rỉ nguyên hàng DB vào payload HTML**, và **`'use client'` ở đầu trang**.

---

```text
Tuân theo `nextjs-server-mastery` (+ `react-client-mastery` cho các client island).
Xây dựng trang RSC tại `{{ROUTE}}`.

DỮ LIỆU CẦN: {{DATA}}
PHẦN TƯƠNG TÁC: {{INTERACTIVE}}
CHÍNH SÁCH CACHE: {{CACHING}}

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

1. MẶC ĐỊNH LÀ SERVER.
   - Page và layout là Server Component. KHÔNG `'use client'` ở cấp page/layout.
   - `'use client'` CHỈ đặt ở các lá thực sự cần state, effect,
     event handler, hoặc browser API: {{INTERACTIVE}}.
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
   để một truy vấn chậm không thể chặn cả trang. Theo {{CACHING}}, quyết định cho từng phần:
   - nhanh + trọng yếu → await ngay trong page (chặn, nhưng nó nhanh)
   - chậm + không trọng yếu → có `<Suspense>` boundary riêng, stream vào sau

7. CACHE theo {{CACHING}}:
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
  app/{{ROUTE}}/page.tsx          (server, compose)
  app/{{ROUTE}}/loading.tsx
  app/{{ROUTE}}/error.tsx         ('use client')
  app/{{ROUTE}}/not-found.tsx
  components/*.tsx                (server, trình bày)
  components/*.client.tsx         (island 'use client' — {{INTERACTIVE}})
  lib/server/use-cases/*.ts       ('server-only')
  lib/server/repositories/*.ts    (interface + adapter)

Trước khi code, in một bảng: | Component | Server hay Client? | Vì sao | và
liệt kê CHÍNH XÁC những field nào đi qua RSC boundary cho từng client island.
Rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
