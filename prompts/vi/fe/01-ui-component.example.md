---
title: Ví dụ — Xây dựng UI Component tái sử dụng
type: example
pairs_with: 01-ui-component.md
---

# 📘 Ví dụ — Xây dựng UI Component tái sử dụng

> Một lượt chạy đầy đủ của [`01-ui-component.md`](01-ui-component.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân thủ skill `react-client-mastery`. Xây dựng component `Button` tái sử dụng được.

BỐI CẢNH
- Element HTML gốc: `<button>`
- Các variant cần có: `intent: primary | danger | ghost`, `size: sm | md | lg`
- Các slot để composition: `icon-left`, `icon-right`
- Nguồn thiết kế: Design system của Acme Console — primary = nền indigo-600 đặc / chữ trắng;
  danger = nền red-600 đặc / chữ trắng; ghost = trong suốt với chữ slate-700 và
  hover slate-100; kích thước sm = h-8 text-sm, md = h-10 text-sm, lg = h-12 text-base;
  bo góc `rounded-md`; icon 16px (sm/md) hoặc 20px (lg), nằm inline trước/sau label.
  Button này sẽ được 8 feature sử dụng (users, invitations, billing, settings,
  audit-log, integrations, roles, onboarding), kể cả bên trong `<form>` và dialog.

REPO CONTEXT: `src/components/` là nơi chứa các design-system primitive (Card, Badge,
  Spinner đã có sẵn). Helper `cn()` (clsx + tailwind-merge) đã tồn tại tại
  `src/lib/cn.ts` — hãy dùng nó, đừng thêm một util merge class thứ hai. CHƯA có Button
  primitive nào: mỗi feature đang tự chế `<button className="...">` riêng.
  Tailwind + `cva` đã được cài sẵn; test dùng Vitest + RTL.

NON-GOALS: Không làm theming hay dark-mode cho Button. Không đổi icon library (giữ lucide-react).
  KHÔNG đi viết lại đống button tự chế của các feature khác lúc này — đó là việc của một PR sau.

RÀNG BUỘC KHÔNG THỎA HIỆP (theo skill)

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. LSP — component BẮT BUỘC phải thay thế được cho `<button>`:
   - Khai báo kiểu props là `React.ComponentPropsWithoutRef<'button'> & VariantProps<typeof ...>`.
   - Bọc bằng `forwardRef` và forward `ref`.
   - Spread `{...rest}` xuống element gốc để `disabled`, `type`, `aria-*`,
     và mọi native handler vẫn hoạt động.
   - KHÔNG BAO GIỜ chỉ chọn tay vài props (`{ label, onClick }`) — điều đó làm hỏng form và dialog.

2. OCP — mở cho mở rộng, đóng cho sửa đổi:
   - Mô hình hóa variant bằng một `cva` variant map. Thêm một kiểu hiển thị mới phải nghĩa là
     thêm MỘT ENTRY VÀO MAP, không bao giờ là thêm nhánh `if` trong thân render.
   - Cho phép composition qua `children` / slots / compound components.
   - KHÔNG có boolean props kiểu `isPrimary`, `isLarge`, `showIcon`.
   - Nhận prop `className` và merge nó CUỐI CÙNG (qua `cn`/`twMerge`) để người dùng
     có thể override mà không phải fork component.

3. ISP — props phải hẹp. Không bao giờ nhận god-object (`user`, `config`) khi
   `src` + `name` là đủ. Nếu bề mặt props bao trùm những mối quan tâm không liên quan nhau,
   hãy tách thành các interface riêng rồi intersect chúng.

4. SRP — file này chỉ render. Không fetch dữ liệu, không business logic, không `useQuery`.
   Mọi hành vi nội bộ (open/close, focus trap) đưa vào một custom hook đặt cạnh (colocated).

5. A11y (WCAG 2.1 AA):
   - Ưu tiên element ngữ nghĩa trước; ARIA chỉ dùng cho widget tương tác tùy biến.
   - Vòng `:focus-visible` phải nhìn thấy được; truy cập được bằng bàn phím (Tab / Enter / Escape).
   - Variant chỉ có icon BẮT BUỘC phải có `aria-label`; icon trang trí đặt `aria-hidden="true"`.
   - Vùng chạm >= 44x44px. Độ tương phản >= 4.5:1 (3:1 cho chữ lớn).
   - Nếu là overlay (modal/drawer): trap focus, khôi phục focus khi đóng,
     đóng khi nhấn Escape, đánh dấu nền là `aria-hidden`.

6. Styling — TailwindCSS + `cva`. Không dùng inline `style={{}}` cho layout/spacing/màu sắc.

7. TypeScript — `strict`. Không `any`, không assertion không an toàn. Export kiểu props ra ngoài.

SẢN PHẨM BÀN GIAO
- `src/components/Button/Button.tsx`  (file PascalCase, chỉ render)
- `src/components/Button/Button.variants.ts` (kebab-case, chứa `cva` map)
- `src/components/Button/index.ts` (public API — barrel ở mức feature được phép)
- `src/components/Button/Button.test.tsx` (xem bên dưới)

TEST (React Testing Library, truy vấn theo ROLE chứ không theo test-id)
- render được từng variant
- forward `ref` xuống đúng DOM node bên dưới
- forward native props (`disabled`, `type="submit"`, `aria-label`)
- tương tác bàn phím hoạt động
- a11y: có accessible name

Trước khi viết code, hãy nêu trong 3 gạch đầu dòng LÀM SAO component này vẫn đóng cho
sửa đổi khi variant tiếp theo xuất hiện. Sau đó mới viết code.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
