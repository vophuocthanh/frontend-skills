---
title: Ví dụ — Tích hợp REST API (Client-Side / React Query)
type: example
pairs_with: 08-csr-api-integration.md
---

# 📘 Ví dụ — Tích hợp REST API (Client-Side)

> Một lượt chạy đầy đủ của [`08-csr-api-integration.md`](08-csr-api-integration.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân theo skill `react-client-mastery`. Tích hợp API Users + Invitations ở phía client.

CONTRACT: src/shared/api/user/ và src/shared/api/invitation/ (đã được định nghĩa — import nó, đừng định nghĩa lại)
ENDPOINTS: users: list / detail — invitations: list / create / revoke
AUTH: httpOnly cookie + refresh khi gặp 401
BẮT BUỘC OPTIMISTIC UI CHO: thu hồi lời mời (revoke invitation)

REPO CONTEXT: `src/lib/http-client.ts` ĐÃ tồn tại (axios + interceptor + mapping status→lỗi
  có kiểu + refresh-on-401) — hãy tái sử dụng nó. KHÔNG tạo client thứ hai hay kiểu lỗi
  thứ hai. Pattern query key factory đã được thiết lập sẵn bởi
  `src/features/billing/api/billing-keys.ts` — sao chép y hệt hình dạng của nó.
  Provider TanStack Query v5 đã được gắn sẵn; các msw handler nằm trong `src/mocks/handlers/`.

NON-GOALS: Không hỗ trợ offline / cache lưu bền. Không websocket hay live update —
  polling và invalidation thủ công là đủ cho lúc này.

CÁC TẦNG — mũi tên phụ thuộc hướng VÀO TRONG. Mỗi tầng chỉ được import tầng ngay dưới nó.

  components/*.tsx           → chỉ JSX. Không biết gì về HTTP.
        ↓
  hooks/use-*.ts             → quy tắc nghiệp vụ, đấu nối React Query
        ↓
  api/invitations-api.ts     → hiện thực interface InvitationGateway
  api/users-api.ts           → hiện thực interface UserGateway
        ↓
  lib/http-client.ts         → file DUY NHẤT trong codebase được import axios/fetch

QUY TẮC

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. DIP — một HTTP client tập trung, có kiểu:
   - `lib/http-client.ts` sở hữu base URL, credentials, timeout, retry và interceptor.
   - Nó chuyển HTTP status → các domain error có kiểu lấy từ contract
     (401 → UnauthorizedError, 409 → ConflictError, 5xx → ServerError).
   - Nó gắn auth và xử lý refresh-on-401 MỘT LẦN, tập trung.
   - KHÔNG có gì khác trong app được import axios/fetch. Nếu một component hay hook làm vậy,
     phân tầng đã bị phá vỡ.
   - Định nghĩa interface `InvitationGateway` (và một interface `UserGateway`);
     `invitations-api.ts` và `users-api.ts` hiện thực chúng.
     Hook phụ thuộc vào INTERFACE. Test tiêm vào một hiện thực giả (fake).

2. SRP — `invitations-api.ts` / `users-api.ts` chỉ nói chuyện HTTP: request vào, dữ liệu đã parse ra.
   Không React, không quyết định cache, không toast, không routing.

3. VALIDATE MỌI RESPONSE tại biên:
   `return InvitationSchema.parse(res.data)` — `as Invitation` BỊ CẤM.
   Một backend đổi hình dạng dữ liệu phải nổ ầm ĩ ngay tại đây, chứ không phải làm hỏng UI
   ở ba màn hình sau đó.

4. Query key factory — tập trung, KHÔNG BAO GIỜ dùng string literal:
     export const invitationsKeys = {
       all: ['invitations'] as const,
       lists: () => [...invitationsKeys.all, 'list'] as const,
       list: (filters: InvitationQuery) => [...invitationsKeys.lists(), filters] as const,
       details: () => [...invitationsKeys.all, 'detail'] as const,
       detail: (id: string) => [...invitationsKeys.details(), id] as const,
     };
   Làm một factory tương tự cho `usersKeys`.

5. Query và mutation — KHÔNG BAO GIỜ trộn lẫn:
   - `useQuery` cho đọc, `useMutation` cho ghi.
   - MỌI query đều đặt `staleTime` và `gcTime` tường minh dựa trên độ biến động của dữ liệu.
     Nêu rõ lập luận cho từng query (ví dụ "danh sách user: staleTime 30s — hiếm khi đổi,
     nhưng không được trông cũ sau khi đồng đội vừa sửa").
   - MỌI mutation thành công đều gọi `queryClient.invalidateQueries` với
     key hẹp nhất mà vẫn đúng.

6. Optimistic update cho thu hồi lời mời — đủ ba bước, KHÔNG đi tắt:
   - `onMutate`: `cancelQueries` → chụp snapshot trạng thái trước → `setQueryData`
   - `onError`:  khôi phục snapshot (ROLLBACK LÀ BẮT BUỘC)
   - `onSettled`: `invalidateQueries`
   Làm ít hơn thế là đẩy ra một UI nói dối người dùng khi mạng chập chờn.

7. LSP — mọi hook trong feature này trả về cùng một họ hình dạng:
   `{ data, isLoading, error }` cho query; `{ mutate, isPending, error }` cho mutation.
   Đừng để hook này trả về mảng còn hook kia trả về object.

8. ISP — truyền props hẹp cho component; select slice của store.
   KHÔNG BAO GIỜ đưa nguyên một DTO thô từ API cho một component trình bày chỉ cần hai field.

9. CẢ BỐN trạng thái đều được đấu nối trong UI: loading (skeleton), lỗi (kèm retry),
   rỗng, thành công. Lỗi được render từ domain error CÓ KIỂU, không phải
   `error.response.data.message` bới ra tại nơi gọi.

10. Auth: token nằm trong httpOnly cookie. KHÔNG BAO GIỜ localStorage/sessionStorage.

SẢN PHẨM BÀN GIAO
  src/lib/http-client.ts
  src/features/invitations/
  ├── api/invitations-gateway.ts   (interface — phần trừu tượng)
  ├── api/invitations-api.ts       (hiện thực)
  ├── api/invitations-keys.ts
  ├── hooks/use-invitations-list.ts
  ├── hooks/use-invitations-detail.ts
  ├── hooks/use-create-invitation.ts
  ├── hooks/use-revoke-invitation.ts
  └── (không có hook delete — revoke là con đường xóa duy nhất trong API này)

  src/features/users/
  ├── api/users-gateway.ts         (interface — phần trừu tượng)
  ├── api/users-api.ts             (hiện thực)
  ├── api/users-keys.ts
  ├── hooks/use-users-list.ts
  └── hooks/use-users-detail.ts

  + msw handler dẫn xuất từ contract
  + test: thành công / lỗi 500 / rỗng / ROLLBACK optimistic khi revoke

In interface gateway và key factory trước, rồi chờ tôi nói "go".

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
