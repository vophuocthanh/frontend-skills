---
title: Review Code Đối Chiếu Với Bộ Skills
category: Review
skills: [react-client-mastery, nextjs-server-mastery]
principles: [SRP, OCP, LSP, ISP, DIP]
---

# 🔍 Prompt — Review Code Đối Chiếu Với Bộ Skills

> Dùng cho một PR, một diff của branch, hoặc một thư mục. Agent review với tư cách **kỹ sư senior sẽ phải bảo trì đoạn code này**, không phải với tư cách một linter.
> Kết quả được sắp xếp theo mức độ ảnh hưởng (blast radius), không theo thứ tự dòng code.

---

```text
Review {{SCOPE}} đối chiếu với các skill `react-client-mastery` và `nextjs-server-mastery`.
CONTEXT: {{CONTEXT}}
DEPTH: {{DEPTH}}

REPO CONTEXT: {{EXISTING}}
  (Các convention sẵn có cần tái sử dụng: cấu trúc thư mục, http client, kiểu lỗi, UI primitives,
   thiết lập test. Ghi "greenfield" nếu chưa có gì.)

NON-GOALS: {{NON_GOALS}}
  (KHÔNG được làm những thứ này. Nếu bạn cho rằng một trong số đó thực sự cần thiết, DỪNG LẠI và hỏi tôi trước.)

0. BÁM REPO TRƯỚC, BÁM PROMPT SAU.
   Nếu REPO CONTEXT cho thấy đã có convention sẵn — cấu trúc thư mục, http client, kiểu lỗi,
   UI primitives, thiết lập test — hãy TÁI SỬ DỤNG chúng thay vì dựng một cấu trúc song song bên
   cạnh. Tạo ra cách thứ hai để làm một việc vốn đã có cách làm rồi là THẤT BẠI, kể cả khi cách mới
   tuân thủ mọi quy tắc bên dưới. Hãy đánh giá code dựa trên các convention repo này thực sự đang
   dùng, không phải dựa trên một hình mẫu greenfield lý tưởng.

Review với tư cách kỹ sư sẽ phải trực on-call cho đoạn code này. Phải cụ thể và
có thể kiểm chứng: mỗi phát hiện cần có file:line, một kịch bản lỗi cụ thể
("nếu người dùng làm X khi offline, Y sẽ xảy ra"), và một cách sửa. KHÔNG ĐƯỢC đưa lời khuyên
mơ hồ kiểu "cân nhắc cải thiện khả năng đọc" — nếu bạn không mô tả được nó hỏng như thế nào,
thì đó không phải là một phát hiện.

═══ MỨC ĐỘ NGHIÊM TRỌNG — báo cáo theo thứ tự này ═══

🔴 BLOCKER (bảo mật / mất dữ liệu / phá vỡ contract)
  - Server Action không có kiểm tra auth bên trong nó (middleware ≠ auth)
  - Authz chỉ theo role, thiếu ownership check (người dùng có thể sửa dữ liệu của người khác)
  - Input không được validate (`formData.get(x) as string`) → leo thang đặc quyền
  - Nguyên cả row DB / secret vượt qua RSC boundary → lộ trong view-source (ISP)
  - Token nằm trong localStorage
  - `dangerouslySetInnerHTML` với input chưa được sanitize
  - Response API không được validate (`as User`) có thể làm hỏng state phía sau
  - Thiếu `revalidatePath` sau một mutation → người dùng thấy dữ liệu cũ và thử lại
  - Secret bị lộ qua `NEXT_PUBLIC_*`

🟠 MAJOR (bug hoặc lỗi thiết kế sẽ gây đau trong vòng một sprint)
  - SRP: một file có từ 2 lý do thay đổi trở lên (fetch + logic + JSX, hoặc auth + SQL + email)
  - DIP: `axios`/`db`/`stripe`/`localStorage` được import trực tiếp trong UI/pages/actions
  - LSP: một wrapper nuốt mất `ref`/`disabled`/`type`/`aria-*`;
         các action có return contract không nhất quán; một fake repo hành xử khác
         với repo thật
  - OCP: chuỗi `if (variant === ...)` / `if (provider === ...)` trong code dùng chung
  - `useEffect` bị dùng để derive/đồng bộ state
  - Optimistic update không có rollback trong `onError`
  - Network waterfall: `await` tuần tự trên các lời gọi độc lập
  - Thiếu state error/empty/loading
  - `'use client'` đặt trên một page/layout, kéo cả cây component vào bundle
  - Import barrel của thư viện làm chết tree-shaking

🟡 MINOR (chất lượng — sửa ngay, chi phí thấp)
  - Bùng nổ boolean prop; props kiểu god-object; subscribe toàn bộ store (không dùng selector)
  - Thiếu `staleTime`/`gcTime`; query key viết bằng chuỗi literal
  - Thiếu debounce/throttle; scroll listener không có `{ passive: true }`
  - `console.log` thay vì structured logging
  - Vi phạm quy ước đặt tên; `any`; thiếu thuộc tính a11y
  - Chuỗi hiển thị cho người dùng bị hardcode (i18n)

═══ ĐỊNH DẠNG KẾT QUẢ BẮT BUỘC ═══

1) VERDICT: APPROVE / APPROVE WITH NITS / REQUEST CHANGES — trong một câu.

2) BẢNG PHÁT HIỆN, sắp xếp theo mức độ nghiêm trọng:
   | # | Sev | Nguyên lý | file:line | Cái gì hỏng (kịch bản cụ thể) | Cách sửa |

3) VÒNG KIỂM SOLID — trả lời từng mục một cách rõ ràng, kèm bằng chứng:
   - SRP: chỉ ra file có nhiều lý do thay đổi nhất. Liệt kê chúng.
   - OCP: chuyện gì xảy ra khi thêm variant/provider tiếp theo? File nào phải mở ra sửa lại?
   - LSP: tất cả wrapper có forward ref/rest không? Tất cả action/hook có chung một return contract không?
   - ISP: chính xác những field nào vượt qua RSC boundary / được truyền làm props? Có god-object nào không?
   - DIP: liệt kê mọi lần import trực tiếp một dependency cụ thể (axios/db/SDK) từ một
     module cấp cao.

4) ĐIỂM TỐT — nêu tên 2-3 thứ đã làm đúng. Review chỉ toàn chê bai sẽ bị bỏ qua.

5) TESTS: phát hiện nào lẽ ra đã bị bắt bởi một test hiện chưa tồn tại?
   Nêu tên test còn thiếu đó.

KHÔNG ĐƯỢC viết lại code. Chỉ báo cáo phát hiện. Tôi sẽ quyết định sửa cái gì.

TRƯỚC KHI BÁO CÁO: mở lại từng file bạn trích dẫn và xác nhận dòng đó vẫn đúng như bạn nói.
Một finding sai file:line còn tệ hơn không có finding — nó đốt sạch niềm tin của người đọc.
Nếu không xác minh được, hãy đánh dấu UNVERIFIED thay vì bỏ đi hoặc khẳng định chắc nịch.
```
