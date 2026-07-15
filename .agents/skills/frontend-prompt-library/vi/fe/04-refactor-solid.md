---
title: Refactor một Legacy Component về chuẩn SOLID
category: FE
skills: [react-client-mastery]
principles: [SRP, OCP, LSP, ISP, DIP]
---

# ♻️ Prompt — Refactor một Legacy Component về chuẩn SOLID

> Dùng cho god-component: 300+ dòng, trộn lẫn fetching + state + business logic + JSX, bùng nổ boolean props, import `axios` thẳng bên trong.
> Refactor này phải **bảo toàn hành vi**. Nếu hành vi buộc phải thay đổi, đó là một PR riêng.

---

```text
Tuân thủ skill `react-client-mastery`. Refactor `{{TARGET}}` cho tuân thủ SOLID.

RÀNG BUỘC
- {{CONSTRAINTS}}
- Test hiện có: {{HAS_TESTS}}
- HÀNH VI KHÔNG ĐƯỢC THAY ĐỔI. Đây là refactor thuần túy. Không thêm tính năng, không sửa bug
  (nếu phát hiện bug, hãy LIỆT KÊ riêng — không được âm thầm sửa).

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

GIAI ĐOẠN 1 — CHẨN ĐOÁN (chưa viết code)
Lập bảng các vi phạm tìm thấy trong file:

  | Nguyên lý | Vi phạm | Bằng chứng (dòng) | Cách sửa |
  |---|---|---|---|

Kiểm tra cụ thể:
- SRP: file này có bao nhiêu lý do để thay đổi? Gọi tên từng lý do.
        (fetching, business rules, định dạng, render, analytics, routing…)
- OCP: các chuỗi `if (variant === ...)` / `if (isPrimary)` bên trong thân render.
- LSP: các element wrapper nuốt mất `ref`, `disabled`, `type`, `aria-*`, `...rest`.
        Các hook variant trả về hình dạng khác nhau.
- ISP: props kiểu god-object (`<Child user={user} />`), subscribe toàn bộ store
        (`useStore()` không có selector), interface props phình to trộn nhiều mối quan tâm.
- DIP: import trực tiếp `axios` / `fetch` / `localStorage` / SDK bên trong tầng UI.

Đồng thời đánh dấu các vi phạm skill tuy không thuộc SOLID nhưng vẫn quan trọng:
- dùng `useEffect` để dẫn xuất/đồng bộ state
- state đáng lẽ thuộc về URL lại nằm trong `useState`
- response API không được validate (`as User`)
- thiếu trạng thái loading/error/empty
- kiểu `any`

GIAI ĐOẠN 2 — LƯỚI AN TOÀN
{{HAS_TESTS}}
Nếu chưa có test: TRƯỚC TIÊN viết characterization test (RTL, truy vấn theo role, msw tại
biên network) để ghim chặt hành vi HIỆN TẠI — bao gồm cả những phần xấu xí.
Chạy chúng xanh TRƯỚC khi động vào phần triển khai.

GIAI ĐOẠN 3 — REFACTOR (mỗi bước một commit, test phải xanh sau mỗi bước)
1. Tách tầng dữ liệu       → `api/*-api.ts` + Zod schema (DIP + validation)
2. Tách business logic     → `hooks/use-*.ts` (SRP: logic-in-hook)
3. Thay `useEffect` dẫn xuất state bằng tính toán ngay lúc render
4. Thay các chuỗi boolean props bằng `cva` variant map / composition (OCP)
5. Khôi phục khả năng thay thế: forwardRef + ComponentPropsWithoutRef + {...rest} (LSP)
6. Thu hẹp props; thêm store selector (ISP)
7. Tách phần JSX còn lại thành các presentational component < 150 dòng mỗi cái
8. Xóa code chết

GIAI ĐOẠN 4 — KIỂM CHỨNG
- Toàn bộ characterization test vẫn xanh.
- In bản tóm tắt trước/sau: số lượng file, file dài nhất, số lý do-để-thay-đổi mỗi file.
- Liệt kê mọi bug đã phát hiện nhưng KHÔNG sửa, để tôi phân loại xử lý.

Chỉ làm GIAI ĐOẠN 1, sau đó dừng lại và chờ tôi phê duyệt.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
