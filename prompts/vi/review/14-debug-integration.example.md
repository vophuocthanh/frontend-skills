---
title: Ví dụ — Debug Lỗi Tích Hợp FE/BE
type: example
pairs_with: 14-debug-integration.md
---

# 📘 Ví dụ — Debug Lỗi Tích Hợp FE/BE

> Một lượt chạy đầy đủ của [`14-debug-integration.md`](14-debug-integration.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân thủ `react-client-mastery` + `nextjs-server-mastery`. Debug bug tích hợp này.

SYMPTOM:  sau khi revoke một lời mời, dòng đó biến mất, nhưng điều hướng sang /settings rồi quay lại thì lời mời đã revoke xuất hiện trở lại với trạng thái pending
EXPECTED: lời mời đã revoke phải biến mất hẳn (hoặc hiển thị status=revoked) sau khi điều hướng đi rồi quay lại
EVIDENCE: network tab: POST /invitations/inv_123/revoke → 200 OK; không có lỗi console; React Query devtools cho thấy query ['invitations','list',{status:'pending'}] vẫn đang giữ dòng cũ
REPRO:    1. mở /team  2. revoke một lời mời đang pending  3. điều hướng sang /settings  4. quay lại /team → lời mời hiện lại, status pending

REPO CONTEXT: query key factory đã tồn tại tại
  `src/features/invitations/api/invitation-keys.ts` — việc invalidation BẮT BUỘC phải đi qua nó,
  không phải qua một mảng key viết tay. Billing đã invalidate đúng cách thông qua factory riêng
  của nó (`src/features/billing/api/billing-keys.ts`), nên invitations mới là kẻ lạc loài —
  hãy so sánh hai bên. TanStack Query v5; msw handler nằm trong `src/mocks/handlers/`.

NON-GOALS: KHÔNG refactor feature invitations trong lúc sửa bug này. Chỉ sửa nguyên nhân gốc.

CHƯA ĐƯỢC THAY ĐỔI BẤT KỲ DÒNG CODE NÀO. Chẩn đoán trước đã.

RULE 0 — BÁM REPO TRƯỚC, BÁM PROMPT SAU.
Nếu REPO CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi,
UI primitives, thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh.
Tạo ra cách thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ
mọi quy tắc bên dưới. Hãy đánh giá code dựa trên các convention repo này thực sự đang dùng, không
phải dựa trên một hình mẫu greenfield lý tưởng.

═══ GIAI ĐOẠN 1 — XÁC ĐỊNH TẦNG ═══
Đi theo request từ đầu đến cuối và cho tôi biết tầng ĐẦU TIÊN mà thực tế lệch khỏi
kỳ vọng. Nêu rõ điều bạn kỳ vọng và điều thực sự xảy ra tại mỗi chặng:

  1. UI event        → handler có chạy không? với payload gì?
  2. Hook/mutation   → chính xác cái gì được gửi đi (log input đã parse)?
  3. HTTP client     → method, URL, headers, cookies, body. Auth có được đính kèm không?
  4. Network         → status, response body. Có khớp với contract không?
  5. Boundary parse  → `Schema.parse` có thành công không? (một cú cast `as T` ngầm sẽ che giấu điều này)
  6. Cache           → đúng query key đã được invalidate chưa? dữ liệu cũ có đang được phục vụ không?
  7. Render          → component có đang đọc đúng state mà nó tưởng không?

Nêu tên tầng đó. Mọi thứ phía sau nó chỉ là triệu chứng, không phải nguyên nhân.

═══ GIAI ĐOẠN 2 — KIỂM TRA CÁC NGHI PHẠM QUEN THUỘC CỦA LỚP BUG NÀY ═══

CONTRACT DRIFT (nguyên nhân số 1 của "chạy ngon trên Postman")
  - Response có được `Schema.parse` không, hay bị cast bằng `as T`? Một cú cast nghĩa là
    app đã và đang nói dối về hình dạng dữ liệu — hãy parse nó và xem nó fail thật to.
  - BE có trả về `null` ở chỗ schema khai là bắt buộc không? camelCase hay snake_case?
    Ngày tháng trả về dạng string trong khi code kỳ vọng Date? Tiền tệ dạng float?
  - Có một lớp envelope bọc ngoài (`{ data: ... }`) mà FE quên bóc không?

DỮ LIỆU CŨ SAU MUTATION (LSP/cache)
  - Thiếu `invalidateQueries` — hoặc invalidate một key KHÁC với key mà query đang dùng
    (một lỗi gõ nhầm chuỗi literal mà key factory lẽ ra đã ngăn được).
  - Server Action thành công nhưng không có `revalidatePath`/`revalidateTag`.
  - Optimistic update đã áp dụng nhưng không bao giờ được đối chiếu lại trong `onSettled`.
  - Router cache đang phục vụ một RSC payload đã prefetch từ trước khi mutation xảy ra.

AUTH / 401
  - Cookie không được gửi: thiếu `credentials: 'include'`, sai domain, `SameSite` quá chặt.
  - Token bị đọc ở phía client (không thể được — nó là httpOnly, đúng theo thiết kế).
  - Middleware đã redirect nhưng action vẫn chạy (hoặc ngược lại).

HYDRATION MISMATCH
  - `Date.now()`, `Math.random()`, `typeof window`, hoặc format theo locale trong lúc render.
  - Một object `Date` hoặc một function được truyền qua RSC boundary (không serialize được).
  - Server và client render hai nhánh khác nhau của cùng một điều kiện.

RACE / THỨ TỰ
  - Hai mutation cùng bay, last-write-wins đè lên nhau.
  - Search có debounce trả về không đúng thứ tự (một response cũ hơn lại về sau cùng).
  - Optimistic rollback khôi phục một snapshot được chụp SAU một update khác.

═══ GIAI ĐOẠN 3 — CHỨNG MINH ═══
- Viết một TEST FAILING tái hiện "sau khi revoke một lời mời, dòng đó biến mất,
  nhưng điều hướng sang /settings rồi quay lại thì lời mời đã revoke xuất hiện trở lại với trạng thái pending"
  tại đúng tầng bạn đã xác định
  (msw cho bug contract; một mutation test cho bug cache).
- Cho tôi thấy nó fail ĐÚNG LÝ DO (dán output lỗi vào).

═══ GIAI ĐOẠN 4 — SỬA ═══
- Sửa NGUYÊN NHÂN GỐC, không sửa triệu chứng.
  Những cách sửa triệu chứng tôi sẽ từ chối: rắc `refetch()` sau mutation thay vì
  invalidation đúng cách; `as any` để làm im lỗi parse; `setTimeout` để "chờ"
  cache; `key={Math.random()}` để ép remount.
- Nếu NGUYÊN NHÂN GỐC nằm ở contract của backend, hãy nói thẳng ra và đưa tôi bằng chứng
  request/response chính xác để tôi chuyển cho team BE. KHÔNG ĐƯỢC che đậy nó ở phía
  client — nhưng nếu buộc phải ship một boundary transform tạm thời, hãy cô lập nó trong
  tầng api, comment rõ LÝ DO, và KHÔNG BAO GIỜ để hình dạng dữ liệu sai lọt qua boundary.

═══ GIAI ĐOẠN 5 — PHÒNG NGỪA ═══
- Rào chắn nào sẽ chặn đứng cả LỚP bug này? (Zod tại boundary, một key factory,
  một package contract dùng chung, một typed error union.) Đề xuất MỘT cái và cho tôi biết chi phí của nó.

Chỉ làm GIAI ĐOẠN 1 và GIAI ĐOẠN 2. Sau đó dừng lại và cho tôi biết tầng nào và giả thuyết hàng đầu của bạn.

KHI BẠN SỬA (GIAI ĐOẠN 4, sau khi tôi duyệt chẩn đoán): chạy cái test fail bạn đã viết và dán
output chứng minh nó đã pass, kèm toàn bộ suite để chứng minh bạn không làm hỏng thứ gì khác.
KHÔNG BAO GIỜ báo đã sửa xong một bug trên code bạn chưa từng chạy.
```
