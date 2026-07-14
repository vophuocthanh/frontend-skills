---
title: Xây dựng UI Component tái sử dụng
category: FE
skills: [react-client-mastery]
principles: [OCP, LSP, ISP]
---

# 🧩 Prompt — Xây dựng UI Component tái sử dụng

> Dùng khi tạo một component **dùng chung, ở tầng design-system** (`Button`, `Input`, `Modal`, `DataTable`, `Card`) sẽ được 3+ feature sử dụng.
> **Không** dùng cho component dùng một lần trong một feature — loại đó thuộc về thư mục của feature (xem `02-feature-page.md`).

---

```text
Tuân thủ skill `react-client-mastery`. Xây dựng component `{{COMPONENT}}` tái sử dụng được.

BỐI CẢNH
- Element HTML gốc: `<{{BASE_ELEMENT}}>`
- Các variant cần có: {{VARIANTS}}
- Các slot để composition: {{SLOTS}}
- Nguồn thiết kế: {{DESIGN_SOURCE}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

RÀNG BUỘC KHÔNG THỎA HIỆP (theo skill)

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. LSP — component BẮT BUỘC phải thay thế được cho `<{{BASE_ELEMENT}}>`:
   - Khai báo kiểu props là `React.ComponentPropsWithoutRef<'{{BASE_ELEMENT}}'> & VariantProps<typeof ...>`.
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
- `src/components/{{COMPONENT}}/{{COMPONENT}}.tsx`  (file PascalCase, chỉ render)
- `src/components/{{COMPONENT}}/{{COMPONENT}}.variants.ts` (kebab-case, chứa `cva` map)
- `src/components/{{COMPONENT}}/index.ts` (public API — barrel ở mức feature được phép)
- `src/components/{{COMPONENT}}/{{COMPONENT}}.test.tsx` (xem bên dưới)

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
