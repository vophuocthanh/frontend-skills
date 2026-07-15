---
title: Xây dựng Form (React Hook Form + Zod)
category: FE
skills: [react-client-mastery]
principles: [SRP, DIP, LSP]
---

# 📝 Prompt — Xây dựng Form

> Dùng cho mọi loại form: đăng nhập, đăng ký, tạo/sửa entity, wizard nhiều bước, panel bộ lọc.
> Quy tắc cốt lõi: **mặc định uncontrolled** (RHF), một Zod schema duy nhất làm nguồn sự thật.

---

```text
Tuân thủ skill `react-client-mastery`. Xây dựng `{{FORM}}`.

ĐẶC TẢ
- Các trường: {{FIELDS}}
- Đích submit: {{SUBMIT_TARGET}}
- Khi thành công: {{ON_SUCCESS}}

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

1. Schema trước — MỘT Zod schema duy nhất là nguồn sự thật.
   - `export const {{FORM}}Schema = z.object({...})`
   - `export type {{FORM}}Values = z.infer<typeof {{FORM}}Schema>` — không bao giờ tự viết tay kiểu này.
   - Tái sử dụng đúng schema đó ở phía server khi đích submit là một Server Action.

2. Mặc định uncontrolled:
   - `useForm({ resolver: zodResolver({{FORM}}Schema), defaultValues })` + `register`.
   - CHỈ dùng `watch`/`useWatch` khi thực sự cần phản ứng tức thời
     (bộ đếm ký tự, trường phụ thuộc, xem trước trực tiếp) — giới hạn phạm vi vào subtree nhỏ nhất.
   - KHÔNG BAO GIỜ dùng `useState` cho từng field kèm onChange thủ công. Cách đó re-render mỗi lần gõ phím.

3. SRP — tách ba mối quan tâm:
   - `{{FORM}}.schema.ts`  → hợp đồng validation
   - `use-{{FORM}}.ts`     → form instance, submit handler, mutation, side effects
   - `{{FORM}}.tsx`        → chỉ JSX; gọi hook và render các field

4. DIP — hook phụ thuộc vào một hàm API/action được inject, không phải `axios`/`fetch`
   viết thẳng bên trong. Nhờ vậy test có thể chạy form với hàm submit giả, không cần network.

5. LSP — các wrapper cho field (`<TextField>`, `<Select>`) BẮT BUỘC phải forward `ref` và spread
   `...rest` để `register()` hoạt động. Một wrapper nuốt mất `ref` sẽ âm thầm làm hỏng RHF.
   Dùng `forwardRef` + `ComponentPropsWithoutRef<'input'>`.

6. Trạng thái submit phải là discriminated union hoặc dùng chính các flag của RHF — không bao giờ
   dùng các boolean `{ isLoading, error?, success? }` vốn cho phép những trạng thái bất khả thi.

7. Xử lý lỗi (ba tầng, bắt buộc đủ cả ba):
   - Cấp field: `formState.errors.<field>` render ngay cạnh input,
     liên kết qua `aria-describedby` + `aria-invalid`.
   - Lỗi field từ phía server: map ngược trở lại bằng `setError(field, ...)`.
   - Cấp form: một vùng `role="alert"` cho các lỗi không thuộc field (409, 500, lỗi mạng).

8. A11y:
   - Mỗi input phải có `<label htmlFor>` thật — placeholder KHÔNG phải là label.
   - `aria-invalid` + `aria-describedby` trên các field bị lỗi.
   - Focus vào field không hợp lệ đầu tiên khi submit thất bại.
   - Vô hiệu hóa nút submit trong lúc `isSubmitting`, và thông báo thành công một cách lịch sự
     (`aria-live="polite"`).

9. Không bao giờ lưu file object thô hay token trong global state. Không `any`.

SẢN PHẨM BÀN GIAO
  src/features/{{FEATURE}}/
  ├── schemas/{{FORM}}-schema.ts
  ├── hooks/use-{{FORM}}.ts
  └── components/{{FORM}}.tsx

TEST (RTL + userEvent, mock tại biên network bằng msw)
- submit dữ liệu hợp lệ và gọi API đúng một lần với payload đã được parse
- hiện lỗi field khi input không hợp lệ và KHÔNG gọi API
- render lỗi từ server (ví dụ 409 email đã tồn tại) trong vùng alert
- vô hiệu hóa submit khi đang pending

In Zod schema ra trước và hỏi tôi xác nhận các quy tắc validation trước khi
viết phần còn lại.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
