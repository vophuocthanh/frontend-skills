---
title: Xây dựng Feature Page phía client
category: FE
skills: [react-client-mastery]
principles: [SRP, ISP, DIP]
---

# 📄 Prompt — Xây dựng Feature Page phía client

> Dùng khi xây dựng một màn hình CSR hoàn chỉnh: danh sách + bộ lọc + phân trang + chi tiết, kèm các trạng thái loading/empty/error.
> Đây là prompt **Logic-in-Hook, UI-in-Component** — quy tắc cấu trúc quan trọng nhất trong skill.

---

```text
Tuân thủ skill `react-client-mastery`. Xây dựng feature page `{{FEATURE}}` tại `{{ROUTE}}`.

YÊU CẦU
- Entity: {{ENTITY}}
- UI state: {{UI_STATE}}
- Nguồn dữ liệu: {{DATA_SOURCE}}
- Hành động của người dùng: {{ACTIONS}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

KIẾN TRÚC — PHÂN TẦNG BẮT BUỘC (SRP: mỗi file chỉ có một lý do để thay đổi)

  api/{{FEATURE}}-api.ts        → thay đổi khi ENDPOINT thay đổi
  api/{{FEATURE}}-schema.ts     → Zod schemas + kiểu suy ra từ schema
  hooks/use-{{FEATURE}}-list.ts → thay đổi khi BUSINESS RULES thay đổi
  components/*.tsx              → thay đổi khi THIẾT KẾ thay đổi

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
   - {{UI_STATE}} nào có thể chia sẻ/bookmark được (bộ lọc, phân trang, sắp xếp, tab)
     BẮT BUỘC phải nằm trong `useSearchParams`, KHÔNG phải `useState`.
   - Chỉ UI state tạm thời (dropdown đang mở, hover) mới được dùng `useState`.

3. Derived state tính trong lúc render:
   - KHÔNG BAO GIỜ đồng bộ state bằng `useEffect`. Tính giá trị đã lọc/sắp xếp/dẫn xuất
     trực tiếp trong thân hook (chỉ bọc `useMemo` khi thực sự đo được là nặng).

4. State đặt cạnh nhau (colocated):
   - Mọi state phải cập nhật CÙNG NHAU thì nằm trong CÙNG một custom hook.
     Đừng tách state gắn kết chặt ra nhiều hook — sẽ bị lệch đồng bộ.

5. Tầng dữ liệu (DIP):
   - Hook phụ thuộc vào `{{FEATURE}}Api`, không bao giờ phụ thuộc trực tiếp vào `axios`/`fetch`.
   - Validate mọi response API bằng Zod (`Schema.parse`). KHÔNG BAO GIỜ `as {{ENTITY}}`.
   - React Query: key factory tập trung ({{FEATURE}}Keys.all / .lists() / .detail(id)),
     `useQuery` cho đọc, `useMutation` cho ghi, khai báo tường minh `staleTime` + `gcTime`,
     `invalidateQueries` sau mỗi mutation thành công.
   - Với các toggle quan trọng về UX, dùng optimistic update (onMutate + rollback trong onError).

6. ISP — truyền props hẹp. `<UserCard name={u.name} email={u.email} />`,
   không phải `<UserCard user={u} />`, trừ khi component con thực sự cần cả entity.
   Đọc Zustand BẮT BUỘC dùng slice selector: `useStore(s => s.isOpen)`, không bao giờ `useStore()`.

7. BẮT BUỘC đủ CẢ BỐN trạng thái dữ liệu: loading (Skeleton phản chiếu đúng layout,
   không phải spinner), error (<ErrorState> + retry), empty (<EmptyState>), success.

8. Hiệu năng: `Promise.all` cho các fetch độc lập (không waterfall), debounce
   ô tìm kiếm (300ms), throttle scroll, `{ passive: true }` cho scroll listener,
   `next/dynamic` cho widget nặng (chart, editor).

SẢN PHẨM BÀN GIAO
  src/features/{{FEATURE}}/
  ├── api/{{FEATURE}}-api.ts
  ├── api/{{FEATURE}}-schema.ts
  ├── api/{{FEATURE}}-keys.ts
  ├── hooks/use-{{FEATURE}}-list.ts
  ├── hooks/use-{{FEATURE}}-filters.ts   (URL state)
  ├── components/{{ENTITY}}List.tsx
  ├── components/{{ENTITY}}Card.tsx
  ├── components/{{ENTITY}}Filters.tsx
  ├── components/{{ENTITY}}ListSkeleton.tsx
  └── index.ts                            (public API — phần nội bộ giữ private)

Bắt đầu bằng việc in ra cây thư mục và một câu cho mỗi file nêu rõ lý do duy nhất
khiến nó phải thay đổi. Chờ tôi nói "go" rồi mới viết phần triển khai.

KHI ĐÃ HIỆN THỰC XONG (không phải trước chốt chặn ở trên):
chạy typecheck, lint và test, rồi dán output thật ra. Nếu có thứ gì fail, nói thẳng và show ra.
Nếu không chạy được, cũng phải nói rõ. KHÔNG BAO GIỜ báo "xong" trên code bạn chưa từng chạy.
```
