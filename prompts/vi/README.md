# 🎯 Thư viện Prompt (Tiếng Việt)

> 🇬🇧 English version: [`../README.md`](../README.md)

> Bộ prompt đã được kiểm chứng, hiện thực hóa hai skill trong `.agents/skills/`.
> Skill nói cho AI biết **luật là gì**. Các prompt này nói cho nó biết **làm gì, theo thứ tự nào, và khi nào phải dừng lại hỏi bạn**.

---

## 📚 Mục lục

Mỗi prompt đều có một file **`.example.md`** đi kèm: chính prompt đó nhưng mọi `{{PLACEHOLDER}}` đã được điền sẵn theo một tình huống thật (**Acme Console** — dashboard quản trị team B2B). Copy là chạy được ngay.

### 🎨 Chỉ Frontend

| # | Prompt | Ví dụ | Dùng khi | Nguyên lý |
|---|--------|-------|----------|-----------|
| 01 | [Xây dựng UI Component tái sử dụng](fe/01-ui-component.md) | [📘](fe/01-ui-component.example.md) | Tạo component design-system dùng cho 3+ feature | OCP, LSP, ISP |
| 02 | [Xây dựng Feature Page Client-Side](fe/02-feature-page.md) | [📘](fe/02-feature-page.example.md) | Màn hình CSR hoàn chỉnh: list + filter + phân trang + các trạng thái | SRP, ISP, DIP |
| 03 | [Xây dựng Form](fe/03-form.md) | [📘](fe/03-form.example.md) | Mọi loại form — login, tạo/sửa, wizard | SRP, DIP, LSP |
| 04 | [Refactor component legacy theo SOLID](fe/04-refactor-solid.md) | [📘](fe/04-refactor-solid.example.md) | Component 300 dòng bạn được "thừa kế" | cả 5 |
| 05 | [Audit & tối ưu hiệu năng](fe/05-performance.md) | [📘](fe/05-performance.example.md) | Gõ phím giật, LCP chậm, bundle phình, re-render loạn | ISP |
| 06 | [Viết Test](fe/06-testing.md) | [📘](fe/06-testing.example.md) | Bổ sung bộ test (Testing Trophy) | DIP, LSP |

### 🔌 Frontend + Backend (tích hợp API)

| # | Prompt | Ví dụ | Dùng khi | Nguyên lý |
|---|--------|-------|----------|-----------|
| 07 | [Định nghĩa API Contract](fullstack/07-api-contract.md) | [📘](fullstack/07-api-contract.example.md) | **Chạy cái này đầu tiên.** Zod contract dùng chung FE + BE | ISP, DIP, LSP |
| 08 | [Tích hợp REST API (Client-Side)](fullstack/08-csr-api-integration.md) | [📘](fullstack/08-csr-api-integration.example.md) | React Query / SWR với backend REST | SRP, DIP, ISP, LSP |
| 09 | [Xây dựng RSC Page fetch dữ liệu trên server](fullstack/09-rsc-data-fetching.md) | [📘](fullstack/09-rsc-data-fetching.example.md) | Trang App Router fetch ở server, streaming | SRP, ISP, DIP |
| 10 | [Xây dựng CRUD với Server Actions](fullstack/10-server-action-crud.md) | [📘](fullstack/10-server-action-crud.example.md) | Mutation full-stack — port, adapter, phân quyền | cả 5 |
| 11 | [Luồng xác thực End-to-End](fullstack/11-auth-flow.md) | [📘](fullstack/11-auth-flow.example.md) | Login, session, refresh, logout, bảo vệ route | SRP, LSP, DIP |
| 12 | [Ship trọn một Feature Slice Full-Stack](fullstack/12-fullstack-slice.md) | [📘](fullstack/12-fullstack-slice.example.md) | **Prompt tổng.** Từ ticket → PR merge được | cả 5 |

### 🔍 Review & Debug

| # | Prompt | Ví dụ | Dùng khi | Nguyên lý |
|---|--------|-------|----------|-----------|
| 13 | [Code Review theo bộ Skill](review/13-code-review.md) | [📘](review/13-code-review.example.md) | Review một PR / branch / folder | cả 5 |
| 14 | [Debug lỗi tích hợp FE/BE](review/14-debug-integration.md) | [📘](review/14-debug-integration.example.md) | "Postman chạy ngon mà app thì không" | LSP, DIP |

---

## 🚀 Cách dùng

1. **Nạp skill trước.** Mọi prompt đều giả định AI đã có sẵn bộ luật:
   ```text
   Tuân thủ các quy tắc trong @.agents/skills/next-client-conversion-skills/SKILL.md
   (và @.agents/skills/nextjs-server-conversion-skills/SKILL.md cho phần server).
   ```
2. **Chọn prompt** từ mục lục ở trên — hoặc file `.example.md` đi kèm nếu bạn muốn bản đã điền sẵn.
3. **Thay hết các `{{PLACEHOLDER}}`** trong khối. Placeholder bỏ trống là nguyên nhân số 1 khiến AI sinh ra code sai ý.
4. **Copy nguyên khối** dán vào agent. Mỗi file chỉ có đúng khối đó — không có phần dạo đầu nào phải cắt bỏ.
5. **Trả lời chốt chặn.** Phần lớn prompt cố ý dừng lại sau giai đoạn thiết kế để chờ bạn duyệt. Đừng bỏ qua bước này — chính khoảng dừng đó chặn được 80% công sức phải làm lại.

---

## 🧭 Các luồng điển hình

**Feature full-stack mới**
```text
07 (contract) → 12 (prompt tổng, tự điều phối 09/10 + 02/03) → 13 (review)
```

**Feature CSR mới trên backend có sẵn**
```text
07 (contract) → 08 (tích hợp) → 02 (page) → 03 (form) → 06 (test) → 13 (review)
```

**Code legacy thừa kế lại**
```text
04 (refactor SOLID) → 06 (test) → 05 (hiệu năng) → 13 (review)
```

**Có thứ gì đó đang hỏng**
```text
14 (debug) → 06 (test hồi quy) → 13 (review)
```

---

## 🧱 Mọi prompt đều siết những gì?

Mỗi prompt là một hình chiếu của cùng một bộ luật cốt lõi trong skill:

| Nguyên lý | Phía Client | Phía Server |
|---|---|---|
| **SRP** | Logic nằm trong hook, UI nằm trong component. `.tsx` = chỉ có JSX. | Action mỏng: auth → validate → ủy thác → revalidate. |
| **OCP** | Compound component, `cva` variant map. Không có boolean prop. | Action wrapper (`withAuth`), strategy map. Không có `if (provider === ...)`. |
| **LSP** | `forwardRef` + `ComponentPropsWithoutRef` + `{...rest}`. | Một `ActionResult<T>` duy nhất. Repo thật và repo fake fail giống hệt nhau. |
| **ISP** | Props hẹp. Đọc store bằng slice selector. | `select` đúng cột. Chỉ primitive được đi qua RSC boundary. |
| **DIP** | Gateway inject qua Context. `axios` chỉ nằm trong đúng một file. | Port + adapter. Một composition root duy nhất. |

Ngoài ra, prompt nào cũng bắt buộc: **Zod ở mọi boundary** (`as T` BỊ CẤM), **đủ bốn trạng thái dữ liệu**, **a11y**, và **không có `any`**.

---

## ✍️ Tự viết prompt mới

Mọi file ở đây đều cùng một hình dạng tối giản — frontmatter, một dòng "khi nào dùng", và **khối prompt, hết**:

```markdown
---
title: ...
category: FE | FE + BE | Review
skills: [...]
principles: [...]
---

# Prompt — <tên>

> Khi nào dùng. Khi nào KHÔNG dùng.

---

```text
(toàn bộ prompt — ràng buộc, sản phẩm bàn giao, và chốt chặn)
```
```

Không bảng input, không checklist, không lời bình. **Prompt CHÍNH LÀ file.** Mọi thứ khác đều là context bạn phải trả giá ở mỗi lần chạy mà không làm thay đổi hành vi của agent.

Ba thứ làm nên hiệu quả của bộ prompt này — giữ lấy chúng:
- **Ràng buộc, không phải mong muốn.** "Wrap trong `forwardRef` và spread `...rest`" ăn đứt "làm cho nó tái sử dụng được".
- **Một chốt chặn.** Bắt agent trình bày thiết kế trước khi nó viết 400 dòng mà bạn sẽ vứt đi.
- **Quy tắc phải dẫn tới quyết định.** Nếu một dòng chỉ nhắc lại thứ mà linter đã bắt sẵn (`không dùng any`, quy ước đặt tên), đó là nhiễu — cắt đi, để file skill gánh phần đó.
