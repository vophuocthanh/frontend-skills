---
title: Ví dụ — Xây dựng Feature Page phía client
type: example
pairs_with: 02-feature-page.md
---

# 📘 Ví dụ — Xây dựng Feature Page phía client

> Một lượt chạy đầy đủ của [`02-feature-page.md`](02-feature-page.md) trên một tình huống thật.
> Copy khối prompt bên dưới và dán thẳng vào agent.

---

```text
Tuân thủ skill `react-client-mastery`. Xây dựng feature page `users` tại `/users`.

YÊU CẦU
- Entity: User (id, name, email, role: admin|member|viewer, isActive, avatarUrl, createdAt)
- UI state: từ khóa tìm kiếm, bộ lọc theo role (admin | member | viewer), trang, sắp xếp theo createdAt
- Nguồn dữ liệu: GET /api/users?query=&role=&page=&sort=
- Hành động của người dùng: vô hiệu hóa một user

REPO CONTEXT: Các slice `src/features/*` đã tồn tại (billing, settings) và tuân theo phân tầng
  api/hooks/components — hãy sao chép y hệt chúng. `src/lib/http-client.ts` đã bọc sẵn fetch
  và map status code sang các kiểu lỗi có kiểu; dùng nó thay vì gọi fetch/axios.
  Các primitive `Button` và `EmptyState` đã tồn tại trong `src/components/`.
  TanStack Query v5 đã được cấu hình sẵn với một provider ở gốc app.

NON-GOALS: Không làm bulk action (chọn nhiều + vô hiệu hóa). Không export CSV. Không tùy biến
  cột / saved view.

KIẾN TRÚC — PHÂN TẦNG BẮT BUỘC (SRP: mỗi file chỉ có một lý do để thay đổi)

  api/users-api.ts        → thay đổi khi ENDPOINT thay đổi
  api/users-schema.ts     → Zod schemas + kiểu suy ra từ schema
  hooks/use-users-list.ts → thay đổi khi BUSINESS RULES thay đổi
  components/*.tsx        → thay đổi khi THIẾT KẾ thay đổi

  Các file .tsx CHỈ chứa JSX. Không useState, không useEffect, không fetch,
  không logic lọc/sắp xếp. Nếu một component có logic, bạn đã vi phạm quy tắc này.

QUY TẮC

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Các đường dẫn trong SẢN PHẨM BÀN GIAO bên dưới chỉ là mặc định cho một repo greenfield. Nếu REPO
   CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên cạnh. Tạo ra cách
   thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới tuân thủ mọi quy
   tắc bên dưới. Liệt kê mọi chỗ bạn lệch khỏi cây thư mục bên dưới, kèm lý do.

1. SRP — mỗi file phải mô tả được bằng một câu KHÔNG chứa từ "và".

2. URL là nguồn sự thật:
   - Từ khóa tìm kiếm, bộ lọc role, trang và sắp xếp — những thứ có thể chia sẻ/bookmark được
     BẮT BUỘC phải nằm trong `useSearchParams`, KHÔNG phải `useState`.
   - Chỉ UI state tạm thời (dropdown đang mở, hover) mới được dùng `useState`.

3. Derived state tính trong lúc render:
   - KHÔNG BAO GIỜ đồng bộ state bằng `useEffect`. Tính giá trị đã lọc/sắp xếp/dẫn xuất
     trực tiếp trong thân hook (chỉ bọc `useMemo` khi thực sự đo được là nặng).

4. State đặt cạnh nhau (colocated):
   - Mọi state phải cập nhật CÙNG NHAU thì nằm trong CÙNG một custom hook.
     Đừng tách state gắn kết chặt ra nhiều hook — sẽ bị lệch đồng bộ.

5. Tầng dữ liệu (DIP):
   - Hook phụ thuộc vào `usersApi`, không bao giờ phụ thuộc trực tiếp vào `axios`/`fetch`.
   - Validate mọi response API bằng Zod (`Schema.parse`). KHÔNG BAO GIỜ `as User`.
   - React Query: key factory tập trung (usersKeys.all / .lists() / .detail(id)),
     `useQuery` cho đọc, `useMutation` cho ghi, khai báo tường minh `staleTime` + `gcTime`,
     `invalidateQueries` sau mỗi mutation thành công.
   - Với các toggle quan trọng về UX, dùng optimistic update (onMutate + rollback trong onError).
     Vô hiệu hóa một user chính xác là một toggle như vậy.

6. ISP — truyền props hẹp. `<UserCard name={u.name} email={u.email} />`,
   không phải `<UserCard user={u} />`, trừ khi component con thực sự cần cả entity.
   Đọc Zustand BẮT BUỘC dùng slice selector: `useStore(s => s.isOpen)`, không bao giờ `useStore()`.

7. BẮT BUỘC đủ CẢ BỐN trạng thái dữ liệu: loading (Skeleton phản chiếu đúng layout,
   không phải spinner), error (<ErrorState> + retry), empty (<EmptyState>), success.

8. Hiệu năng: `Promise.all` cho các fetch độc lập (không waterfall), debounce
   ô tìm kiếm (300ms), throttle scroll, `{ passive: true }` cho scroll listener,
   `next/dynamic` cho widget nặng (chart, editor).

SẢN PHẨM BÀN GIAO
  src/features/users/
  ├── api/users-api.ts
  ├── api/users-schema.ts
  ├── api/users-keys.ts
  ├── hooks/use-users-list.ts
  ├── hooks/use-users-filters.ts   (URL state)
  ├── components/UserList.tsx
  ├── components/UserCard.tsx
  ├── components/UserFilters.tsx
  ├── components/UserListSkeleton.tsx
  └── index.ts                      (public API — phần nội bộ giữ private)

Bắt đầu bằng việc in ra cây thư mục và một câu cho mỗi file nêu rõ lý do duy nhất
khiến nó phải thay đổi. Chờ tôi nói "go" rồi mới viết phần triển khai.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
