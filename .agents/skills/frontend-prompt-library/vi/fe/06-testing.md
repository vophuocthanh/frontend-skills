---
title: Viết Test (Testing Trophy)
category: FE
skills: [react-client-mastery]
principles: [DIP, LSP]
---

# 🧪 Prompt — Viết Test

> Tuân theo **Testing Trophy**: integration test gánh phần lớn trọng lượng; unit test phủ hook/util/schema; E2E chỉ phủ các luồng tối quan trọng.
> Mock tại **biên network** (`msw`) — không bao giờ mock chính module của mình.

---

```text
Tuân thủ skill `react-client-mastery`. Viết test cho `{{TARGET}}`.

CÁC LUỒNG NGƯỜI DÙNG: {{FLOWS}}
LUỒNG TỐI QUAN TRỌNG (E2E): {{CRITICAL_PATH}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

CHIẾN LƯỢC — Testing Trophy, theo tỷ lệ sau:

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. INTEGRATION (phần lớn nhất — React Testing Library)
   Test luồng người dùng xuyên qua cây component thật với hook thật.
   - Truy vấn theo ROLE / LABEL / TEXT. `getByTestId` là phương án cuối cùng — nếu cần đến nó,
     component nhiều khả năng có bug a11y; hãy sửa component thay vì dùng nó.
   - Điều khiển bằng `userEvent`, không phải `fireEvent`.
   - CHỈ mock tại biên network bằng `msw` (`http.get('/api/users', ...)`).
     KHÔNG BAO GIỜ `vi.mock('@/features/users/api/users-api')` — mock module của chính mình
     là đang test cái mock, không phải test code.
   - Assert những gì NGƯỜI DÙNG thấy (text, role, aria-live), không phải state nội bộ.
   - Không dùng `waitFor(() => expect(mockFn).toHaveBeenCalled())` làm assertion chính —
     hãy assert kết quả đã được render.

2. UNIT (nhanh, tập trung)
   - Custom hook qua `renderHook` — hook CHÍNH LÀ business logic, nên đây là nơi
     độ phủ logic nằm ở (mục tiêu của skill: 80%+ trên các hook chứa business logic).
   - Các util và formatter thuần túy.
   - Zod schema: input hợp lệ parse được; input không hợp lệ sinh ra đúng các lỗi field kỳ vọng.

3. E2E (Playwright — chỉ {{CRITICAL_PATH}})
   Trình duyệt thật, điều hướng thật. Giữ số lượng ít; loại này chậm và dễ flaky.

ĐỘ PHỦ BẮT BUỘC CHO MỌI VIEW ĐƯỢC DẪN DẮT BỞI DỮ LIỆU
Cả bốn trạng thái đều phải có test:
   - loading  → skeleton hiển thị
   - error    → trạng thái lỗi + retry hoạt động (msw trả về 500)
   - empty    → trạng thái rỗng (msw trả về [])
   - success  → dữ liệu được render

ĐỘ PHỦ BẮT BUỘC CHO MỌI MUTATION
   - happy path → API được gọi đúng một lần với payload chính xác; cache được invalidate;
                  UI phản ánh trạng thái mới
   - thất bại   → lỗi được hiển thị cho người dùng (role="alert"); optimistic update
                  được ROLLBACK (đây là con bug mà optimistic UI luôn kéo theo)
   - pending    → submit bị vô hiệu hóa / hiện spinner

ĐỘ PHỦ BẮT BUỘC CHO CÁC COMPONENT DÙNG CHUNG (LSP)
   - `ref` được forward xuống DOM node
   - native props đi xuyên qua được (`disabled`, `type="submit"`, `aria-label`)

Lợi ích của DIP: nếu một hook nhận gateway/api dưới dạng dependency được inject, hãy unit-test nó
với một bản giả — không cần msw, không cần network. Dùng cách này ở nơi có sẵn đường nối (seam).

QUY ƯỚC
- Đặt cạnh nhau: `Component.test.tsx` nằm ngay cạnh `Component.tsx`.
- Một `describe` cho một hành vi, không phải một method.
- Tên test đọc lên như câu mô tả hướng người dùng:
  ✅ 'shows an error when credentials are invalid'
  ❌ 'test handleSubmit returns false'
- Không dùng snapshot test cho logic. Snapshot chỉ dành cho markup thuần thị giác và ổn định.
- Tính xác định: dùng fake timer cho debounce, ngày tháng cố định. Không `sleep()`.

SẢN PHẨM BÀN GIAO
Liệt kê các file test sẽ tạo và các case trong từng file, rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
