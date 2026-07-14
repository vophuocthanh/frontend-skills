---
title: Audit & Tối ưu hiệu năng
category: FE
skills: [react-client-mastery, nextjs-server-mastery]
principles: [ISP]
---

# ⚡ Prompt — Audit & Tối ưu hiệu năng

> Dùng khi một trang bị chậm: gõ phím giật, TTI lâu, bundle khổng lồ, bão re-render, network waterfall.
> Quy tắc của nhà: **đo trước khi tối ưu**. Không rải `memo()` theo cảm tính.

---

```text
Tuân thủ skill `react-client-mastery` (và `nextjs-server-mastery` nếu route này
có server component). Audit và tối ưu `{{TARGET}}`.

TRIỆU CHỨNG: {{SYMPTOM}}
NGÂN SÁCH:  {{BUDGET}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

RULE 0 — BÁM REPO TRƯỚC, BÁM PROMPT SAU.
Nếu REPO CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi,
UI primitives, thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh.
Tạo ra cách thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ
mọi quy tắc bên dưới. Hãy đánh giá code dựa trên các convention repo này thực sự đang dùng, không
phải dựa trên một hình mẫu greenfield lý tưởng.

GIAI ĐOẠN 1 — ĐO TRƯỚC (chưa sửa gì)
Xác định nút thắt THỰC SỰ và phân loại nó. Báo cáo kết quả theo dạng:

  | # | Nhóm | Bằng chứng (file:dòng) | Chi phí ước tính | Cách sửa |
  |---|------|------------------------|------------------|----------|

Audit các nhóm sau THEO THỨ TỰ — nhóm đứng trước thường chiếm phần lớn tác động:

A. NETWORK WATERFALL (thường mang lại lợi ích lớn nhất)
   - `await` tuần tự trên các lời gọi độc lập → BẮT BUỘC chuyển sang `Promise.all()`.
   - Fetch phía client chỉ chạy sau khi fetch của component cha resolve (chuỗi request).
   - Server: thiếu dedup bằng `React.cache()`; cùng một query chạy ở layout + page + sidebar.

B. KÍCH THƯỚC BUNDLE
   - Import barrel của thư viện giết chết tree-shaking:
     `import { Button } from '@mui/material'` → `import Button from '@mui/material/Button'`.
     (Barrel `index.ts` ở mức feature thì ổn — KHÔNG được xóa chúng.)
   - Component nặng (chart, editor, map, date picker) không nằm sau `next/dynamic`.
   - Import cả package kiểu Moment/lodash.
   - `'use client'` đặt ở mức page/layout, kéo cả cây component vào bundle.
     Đẩy nó xuống các lá.

C. BÃO RE-RENDER
   - Input controlled re-render cả một cây lớn mỗi lần gõ phím → chuyển sang uncontrolled (RHF)
     hoặc cô lập input thành component lá riêng.
   - Thiếu debounce (input, resize) / throttle (scroll, mousemove).
   - Listener scroll/wheel/touch không có `{ passive: true }`.
   - Dùng Context cho state TẦN SUẤT CAO → chuyển sang Zustand với slice selector.
     ISP: `useStore(s => s.x)`, không bao giờ `useStore()`.
   - Giá trị tần suất cao giữ trong `useState` dù chúng không bao giờ tới DOM → `useRef`.
   - Object/function inline truyền vào các con đã `memo()`, phá vỡ memoization.
   - `useEffect` đồng bộ derived state → chắc chắn gây double render.

D. CHI PHÍ RENDER
   - JSX tĩnh bị dựng lại mỗi lần render → nâng ra ngoài thân component.
   - Tính toán tốn kém mà không có `useMemo` (chỉ thêm sau khi chứng minh được là tốn kém).
   - Danh sách dài không virtualization.
   - Đọc/ghi DOM xen kẽ nhau → layout thrashing.

E. ASSETS
   - Dùng `<img>` thô thay vì `next/image`; thiếu width/height (CLS);
     thiếu `priority` trên ảnh LCP; dùng PNG ở nơi đáng lẽ phải là WebP/AVIF.

GIAI ĐOẠN 2 — SỬA
- Sửa theo thứ tự tác động (A → E). Mỗi commit một mối quan tâm.
- Với mỗi lần sửa, nêu rõ cơ chế kỳ vọng ("loại bỏ một vòng round-trip tuần tự 300ms",
  "giảm 180KB khỏi chunk ban đầu") — không chỉ nói "nhanh hơn".
- KHÔNG thêm `memo`/`useMemo`/`useCallback` theo cảm tính. Chỉ thêm ở nơi có re-render
  đã đo được hoặc tính toán tốn kém đã chứng minh, và phải nói rõ là cái nào.

GIAI ĐOẠN 3 — KIỂM CHỨNG
- Đo lại theo {{BUDGET}}. Đưa ra số liệu trước/sau.
- Nếu một bản sửa KHÔNG giúp ích, hãy revert nó và nói rõ. Tối ưu vô dụng là nợ kỹ thuật.

Chỉ làm GIAI ĐOẠN 1, sau đó dừng lại.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
