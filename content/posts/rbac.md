+++
date = '2026-09-23T09:00:00+07:00'
draft = false
slug = 'rbac'
title = 'RBAC: Phân Quyền Theo Vai Trò, Từ Mô Hình Đến Dòng Code'
author = 'Huy Dang Quang'
categories = ["Security"]
tags = ["security", "rbac", "authorization", "golang", "jwt"]
description = 'RBAC là gì, mô hình chuẩn NIST, cách implement với JWT + Go + MySQL + Redis, ưu nhược điểm và những chỗ RBAC không đủ dùng'
+++

# Tổng quan

Trong [OWASP Top 10](/posts/owasp/), **Broken Access Control** đứng ở vị trí **A01** — lỗ hổng phổ biến nhất, trên cả injection và lỗi mật mã. Không phải vì phân quyền khó về mặt kỹ thuật, mà vì nó **dễ làm sai một cách im lặng**: quên một middleware ở một route mới thêm, kiểm tra role ở frontend mà quên ở backend, hoặc kiểm tra "được xem giao dịch" nhưng quên kiểm tra "được xem giao dịch *này*".

RBAC (Role-Based Access Control) là câu trả lời phổ biến nhất cho bài toán đó. Bài này đi từ mô hình lý thuyết chuẩn, xuống tới schema MySQL và middleware Go thật, rồi nói thẳng về **những chỗ RBAC không đủ dùng** — phần mà phần lớn bài viết về RBAC bỏ qua, và cũng là phần khiến nhiều hệ thống có RBAC đầy đủ vẫn dính A01.

---

## Authentication ≠ Authorization

Hai khái niệm này bị dùng lẫn lộn liên tục, kể cả trong code review. Phân biệt rất đơn giản:

| | **Authentication (AuthN)** | **Authorization (AuthZ)** |
|---|---|---|
| Câu hỏi | **Bạn là ai?** | **Bạn được làm gì?** |
| Kiểm tra cái gì | Mật khẩu, JWT signature, OTP, khoá API | Role, permission, ownership, policy |
| Xảy ra khi nào | **Trước** | **Sau** AuthN |
| Mã lỗi HTTP | **401** Unauthorized | **403** Forbidden |
| Thất bại nghĩa là | "Tôi không biết bạn là ai" | "Tôi biết bạn là ai, và bạn không được phép" |
| Thay đổi tần suất | Hiếm (đổi mật khẩu) | Thường xuyên (đổi chức vụ, thêm tính năng) |

> ⚠️ **Bẫy mã lỗi**: HTTP 401 tên là "Unauthorized" nhưng thực chất nghĩa là **chưa xác thực**. Sai tên từ trong chuẩn RFC. Đúng ngữ nghĩa phải là: **401 = chưa đăng nhập / token hỏng**, **403 = đã đăng nhập nhưng không đủ quyền**. Trả nhầm 403 khi token hết hạn sẽ khiến client không biết đường refresh token.
>
> Một số hệ thống cố tình trả **404 thay vì 403** cho tài nguyên nhạy cảm, để không lộ ra rằng tài nguyên đó tồn tại. Đây là lựa chọn có chủ đích, không phải lỗi — nhưng phải nhất quán toàn hệ thống.

---

# RBAC là gì?

Ý tưởng cốt lõi chỉ có một câu:

> **Không gán quyền trực tiếp cho người. Gán quyền cho *vai trò*, rồi gán *vai trò* cho người.**

Nghe có vẻ chỉ là thêm một lớp gián tiếp thừa thãi. Nhưng lớp gián tiếp đó đổi độ phức tạp từ `O(người × quyền)` xuống `O(người + quyền)` — và quan trọng hơn: nó khớp với **cách tổ chức thật vận hành**. Công ty không nghĩ "cho anh Huy quyền xoá giao dịch", công ty nghĩ "anh Huy là Kế toán trưởng, mà Kế toán trưởng thì được xoá giao dịch".

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="rb-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="rb-arrT" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="rb-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Một request đi qua hai cửa khác nhau</text>
<rect x="40" y="52" width="130" height="58" rx="10" fill="#FFFFFF" stroke="#57534E" stroke-width="2" filter="url(#rb-sh)"/>
<rect x="230" y="52" width="190" height="58" rx="10" fill="#E6FBF9" stroke="#0D9488" stroke-width="2.5" filter="url(#rb-sh)"/>
<rect x="480" y="52" width="190" height="58" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5" filter="url(#rb-sh)"/>
<rect x="730" y="52" width="130" height="58" rx="10" fill="#FFFFFF" stroke="#57534E" stroke-width="2" filter="url(#rb-sh)"/>
<text x="105" y="78" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">Client</text>
<text x="105" y="96" text-anchor="middle" font-size="10" fill="#78716C">gửi kèm token</text>
<text x="325" y="76" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">① AuthN — Bạn là ai?</text>
<text x="325" y="94" text-anchor="middle" font-size="10" fill="#0F766E">verify chữ ký + hạn token</text>
<text x="575" y="76" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">② AuthZ — Được làm gì?</text>
<text x="575" y="94" text-anchor="middle" font-size="10" fill="#92400E">RBAC kiểm tra permission</text>
<text x="795" y="78" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">Handler</text>
<text x="795" y="96" text-anchor="middle" font-size="10" fill="#78716C">xử lý nghiệp vụ</text>
<line x1="172" y1="81" x2="226" y2="81" stroke="#57534E" stroke-width="2" marker-end="url(#rb-arr)"/>
<line x1="422" y1="81" x2="476" y2="81" stroke="#57534E" stroke-width="2" marker-end="url(#rb-arr)"/>
<line x1="672" y1="81" x2="726" y2="81" stroke="#57534E" stroke-width="2" marker-end="url(#rb-arr)"/>
<rect x="258" y="124" width="134" height="26" rx="6" fill="#FFE9EC"/>
<rect x="508" y="124" width="134" height="26" rx="6" fill="#FFE9EC"/>
<text x="325" y="142" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">thất bại → 401</text>
<text x="575" y="142" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">thất bại → 403</text>
<line x1="40" y1="172" x2="860" y2="172" stroke="#E7E5E4" stroke-width="2"/>
<text x="450" y="200" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Mô hình lõi của RBAC — hai quan hệ N:M nối tiếp</text>
<text x="130" y="232" text-anchor="middle" font-size="12" font-weight="800" fill="#0D9488">USERS</text>
<text x="430" y="232" text-anchor="middle" font-size="12" font-weight="800" fill="#D97706">ROLES</text>
<text x="765" y="232" text-anchor="middle" font-size="12" font-weight="800" fill="#6366F1">PERMISSIONS</text>
<rect x="245" y="218" width="58" height="20" rx="5" fill="#57534E"/>
<rect x="558" y="218" width="58" height="20" rx="5" fill="#57534E"/>
<text x="274" y="233" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">N : M</text>
<text x="587" y="233" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">N : M</text>
<rect x="60" y="250" width="140" height="44" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="60" y="310" width="140" height="44" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="60" y="370" width="140" height="44" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<text x="130" y="277" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">An</text>
<text x="130" y="337" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Binh</text>
<text x="130" y="397" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Cuong</text>
<rect x="350" y="250" width="160" height="44" rx="9" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<rect x="350" y="310" width="160" height="44" rx="9" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<rect x="350" y="370" width="160" height="44" rx="9" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="430" y="277" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Admin</text>
<text x="430" y="337" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Accountant</text>
<text x="430" y="397" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Viewer</text>
<rect x="660" y="232" width="210" height="40" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.8"/>
<rect x="660" y="284" width="210" height="40" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.8"/>
<rect x="660" y="336" width="210" height="40" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.8"/>
<rect x="660" y="388" width="210" height="40" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.8"/>
<text x="765" y="257" text-anchor="middle" font-size="11" font-family="monospace" fill="#3730A3">user:manage</text>
<text x="765" y="309" text-anchor="middle" font-size="11" font-family="monospace" fill="#3730A3">transaction:write</text>
<text x="765" y="361" text-anchor="middle" font-size="11" font-family="monospace" fill="#3730A3">transaction:read</text>
<text x="765" y="413" text-anchor="middle" font-size="11" font-family="monospace" fill="#3730A3">budget:read</text>
<line x1="202" y1="272" x2="346" y2="272" stroke="#0D9488" stroke-width="1.8" marker-end="url(#rb-arrT)"/>
<line x1="202" y1="332" x2="346" y2="332" stroke="#0D9488" stroke-width="1.8" marker-end="url(#rb-arrT)"/>
<path d="M202,338 C260,350 290,384 346,390" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#rb-arrT)"/>
<line x1="202" y1="392" x2="346" y2="392" stroke="#0D9488" stroke-width="1.8" marker-end="url(#rb-arrT)"/>
<path d="M512,266 C560,262 610,254 656,252" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<path d="M512,278 C560,286 610,300 656,304" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<path d="M512,326 C560,318 610,308 656,306" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<path d="M512,338 C560,344 610,354 656,356" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<path d="M512,386 C560,378 610,362 656,360" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<path d="M512,398 C560,402 610,408 656,410" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#rb-arr)"/>
<text x="130" y="440" text-anchor="middle" font-size="10" fill="#0F766E">1 user có thể có nhiều role</text>
<text x="430" y="440" text-anchor="middle" font-size="10" fill="#92400E">role = chức năng công việc</text>
<text x="765" y="440" text-anchor="middle" font-size="10" fill="#4338CA">permission = resource:action</text>
<rect x="150" y="452" width="600" height="20" rx="6" fill="#CCFBF1"/>
<text x="450" y="466" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Quyền luôn đi qua ROLE — không bao giờ nối thẳng USER → PERMISSION</text>
</svg>
</div>

## Bốn thành phần

| Thành phần | Là gì | Ví dụ |
|---|---|---|
| **User** (Subject) | Chủ thể thực hiện hành động | `user_42`, một service account, một API key |
| **Role** | Một **chức năng công việc**, không phải một con người | `accountant`, `support_agent`, `billing_admin` |
| **Permission** | Một cặp **(tài nguyên, hành động)** được phép | `transaction:read`, `budget:delete` |
| **Session** | Tập role **đang được kích hoạt** trong phiên hiện tại | Token chứa `roles: ["viewer"]` dù user có 3 role |

Thành phần thứ tư — **Session** — hay bị bỏ qua nhưng rất quan trọng: nó cho phép một user *có* nhiều role nhưng chỉ *kích hoạt* một phần trong mỗi phiên. Đây chính là cơ sở của cơ chế "sudo mode" / "elevated session": bạn là admin, nhưng phiên đăng nhập bình thường chỉ chạy với quyền thường; muốn dùng quyền admin phải xác thực lại. Mô hình `sudo` của Unix và "Confirm access" của GitHub đều là ý tưởng này.

## Quy ước đặt tên permission

Đây là quyết định nhỏ nhưng ảnh hưởng suốt đời dự án. Định dạng phổ biến và nên dùng:

```
<resource>:<action>
```

```
transaction:read          transaction:create      transaction:delete
budget:read               budget:update
user:manage               report:export
```

Vài nguyên tắc:

- **Danh từ số ít cho resource, động từ nguyên thể cho action.** Nhất quán tuyệt đối — `transaction:read` chứ không lẫn lộn `transactions:read` và `transaction:view`.
- **Đặt tên theo *tài nguyên nghiệp vụ*, không theo *endpoint*.** `transaction:read` chứ không phải `GET_/api/v1/transactions`. Endpoint sẽ đổi; khái niệm nghiệp vụ thì không.
- **Tách `read` và `list`** nếu xem một bản ghi và liệt kê toàn bộ là hai mức nhạy cảm khác nhau. Rất hay gặp trong fintech.
- **Cẩn thận với wildcard.** `transaction:*` tiện nhưng nguy hiểm: thêm một action mới `transaction:export` là mọi role có wildcard tự động được quyền đó — không ai review, không ai biết. Nếu dùng wildcard, hãy giới hạn ở role `super_admin` và ghi log mỗi lần nó được dùng.

---

# Bốn nguyên tắc nền tảng

## 1. Least Privilege — đặc quyền tối thiểu

Mỗi role chỉ được đúng những permission cần để làm việc, không hơn một cái nào. Nghe hiển nhiên, nhưng trong thực tế nó bị phá theo hai cách:

- **Phình dần theo thời gian**: role `support_agent` được thêm quyền mỗi khi có một ticket khó, và không bao giờ bị gỡ bớt. Sau hai năm nó gần bằng `admin`. Cần **review định kỳ** (access recertification), không chỉ cấp quyền một chiều.
- **Cấp quyền tạm mà quên thu**: kỹ sư được cấp quyền production để debug sự cố, xong việc không ai gỡ. Giải pháp: role có **hạn** (`expires_at`), tự hết hiệu lực.

## 2. Separation of Duties — tách biệt nhiệm vụ

Không một người nào được nắm trọn một quy trình nhạy cảm. Người *tạo* lệnh chuyển tiền không được là người *duyệt* lệnh đó. Có hai biến thể:

| | **Static SoD (SSD)** | **Dynamic SoD (DSD)** |
|---|---|---|
| Ràng buộc ở đâu | Lúc **gán** role | Lúc **kích hoạt** role trong phiên |
| Quy tắc | Một user **không được có** cả hai role xung đột | Được có cả hai, nhưng **không được dùng cùng lúc** |
| Ví dụ | Không ai vừa là `payment_creator` vừa là `payment_approver` | Có thể có cả hai, nhưng một phiên chỉ kích hoạt được một |
| Đánh đổi | An toàn hơn, cứng nhắc hơn | Linh hoạt hơn, cần kiểm soát phiên chặt |

SSD dễ implement hơn nhiều: chỉ cần một bảng `role_conflicts` và một check lúc gán role. Với công ty nhỏ (ít người, kiêm nhiệm nhiều), DSD thực tế hơn.

## 3. Role phản ánh chức năng, không phản ánh con người

```
❌ role_huy, role_team_ketoan_tang_3, role_nguoi_moi
✅ accountant, transaction_approver, read_only_auditor
```

Nếu tên role chứa tên người, tên phòng ban cụ thể, hoặc tên một cá nhân — bạn đã quay lại mô hình gán quyền trực tiếp, chỉ là đổi nhãn. Phép thử: **người rời công ty, role có sống tiếp không?** Nếu không, đó không phải role.

## 4. Deny by Default — mặc định từ chối

Không có permission nghĩa là **không được phép**, chứ không phải "chưa quyết định". Điều này nghe hiển nhiên nhưng trong code nó có ý nghĩa rất cụ thể:

```go
// ❌ SAI: route mới thêm vào mặc định là PUBLIC
r := gin.New()
r.Use(AuthMiddleware())              // chỉ xác thực, không phân quyền
r.GET("/transactions", h.List)       // ai đăng nhập cũng gọi được
r.DELETE("/transactions/:id", h.Del) // ...kể cả cái này

// ✅ ĐÚNG: mọi route phải khai báo permission, quên là compile/test fail
r.GET("/transactions",
    RequirePermission(auth.PermTransactionRead), h.List)
r.DELETE("/transactions/:id",
    RequirePermission(auth.PermTransactionDelete), h.Del)
```

Cách bảo vệ nguyên tắc này bằng test tự động nằm ở [phần Anti-pattern](#anti-pattern-6--route-mới-quên-gắn-middleware) phía dưới.

---

# Mô hình chuẩn NIST — RBAC0 đến RBAC3

Năm 1996, Sandhu và cộng sự đưa ra một họ bốn mô hình RBAC, sau này thành chuẩn **ANSI INCITS 359-2004**. Biết được mình đang ở cấp nào giúp bạn không over-engineer.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 450" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="nl2-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="nl2-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="450" fill="#FFFDF7" rx="16"/>
<text x="440" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Bốn cấp độ RBAC — mỗi cấp thêm đúng một ý tưởng</text>
<rect x="40" y="52" width="210" height="76" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#nl2-sh)"/>
<rect x="40" y="146" width="210" height="76" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5" filter="url(#nl2-sh)"/>
<rect x="40" y="240" width="210" height="76" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#nl2-sh)"/>
<rect x="40" y="334" width="210" height="76" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#nl2-sh)"/>
<text x="145" y="80" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">RBAC₀ — Core</text>
<text x="145" y="102" text-anchor="middle" font-size="11" fill="#0F766E">User · Role · Permission</text>
<text x="145" y="119" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">Session</text>
<text x="145" y="174" text-anchor="middle" font-size="14" font-weight="800" fill="#7C4A03">RBAC₁ — Hierarchical</text>
<text x="145" y="196" text-anchor="middle" font-size="11" fill="#92400E">RBAC₀ + role kế thừa</text>
<text x="145" y="213" text-anchor="middle" font-size="11" font-weight="700" fill="#92400E">quyền của role cha</text>
<text x="145" y="268" text-anchor="middle" font-size="14" font-weight="800" fill="#3730A3">RBAC₂ — Constrained</text>
<text x="145" y="290" text-anchor="middle" font-size="11" fill="#4338CA">RBAC₀ + ràng buộc</text>
<text x="145" y="307" text-anchor="middle" font-size="11" font-weight="700" fill="#4338CA">SoD tĩnh &amp; động</text>
<text x="145" y="362" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">RBAC₃ — Combined</text>
<text x="145" y="384" text-anchor="middle" font-size="11" fill="#BE123C">RBAC₁ + RBAC₂</text>
<text x="145" y="401" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">đầy đủ nhất</text>
<rect x="278" y="52" width="562" height="76" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<rect x="278" y="146" width="562" height="76" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<rect x="278" y="240" width="562" height="76" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<rect x="278" y="334" width="562" height="76" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="298" y="76" font-size="11" font-weight="700" fill="#374151">Mức tối thiểu để gọi là RBAC. Gán user vào role, role vào permission.</text>
<text x="298" y="96" font-size="11" fill="#57534E">Session cho phép kích hoạt một phần role trong mỗi phiên (nền của sudo mode).</text>
<text x="298" y="118" font-size="11" font-weight="700" fill="#0F766E">👉 Đủ cho đại đa số ứng dụng web / SaaS giai đoạn đầu.</text>
<text x="298" y="170" font-size="11" font-weight="700" fill="#374151">Role cha tự động có mọi quyền của role con — mô hình hoá cấp bậc tổ chức.</text>
<text x="298" y="190" font-size="11" fill="#57534E">senior_accountant ⊃ accountant ⊃ viewer. Giảm mạnh số permission phải khai báo.</text>
<text x="298" y="212" font-size="11" font-weight="700" fill="#92400E">⚠ Coi chừng vòng lặp kế thừa, và chi phí duyệt cây mỗi lần check.</text>
<text x="298" y="264" font-size="11" font-weight="700" fill="#374151">Thêm ràng buộc: một số role không được cùng tồn tại trên một người.</text>
<text x="298" y="284" font-size="11" fill="#57534E">SSD: chặn lúc gán role. DSD: cho phép gán nhưng chặn lúc kích hoạt cùng phiên.</text>
<text x="298" y="306" font-size="11" font-weight="700" fill="#4338CA">👉 Bắt buộc với fintech, y tế, kế toán — nơi có yêu cầu tuân thủ.</text>
<text x="298" y="358" font-size="11" font-weight="700" fill="#374151">Gộp cả kế thừa và ràng buộc. Mạnh nhất, nhưng cũng phức tạp nhất.</text>
<text x="298" y="378" font-size="11" fill="#57534E">Cấp bậc + SoD dễ mâu thuẫn nhau: role cha kế thừa có thể vô tình vi phạm SoD.</text>
<text x="298" y="400" font-size="11" font-weight="700" fill="#BE123C">👉 Chỉ làm khi thật sự cần — và phải có test kiểm chứng ràng buộc.</text>
<line x1="145" y1="130" x2="145" y2="142" stroke="#57534E" stroke-width="2" marker-end="url(#nl2-arr)"/>
<line x1="145" y1="224" x2="145" y2="236" stroke="#57534E" stroke-width="2" marker-end="url(#nl2-arr)"/>
<line x1="145" y1="318" x2="145" y2="330" stroke="#57534E" stroke-width="2" marker-end="url(#nl2-arr)"/>
<text x="440" y="436" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E">Bắt đầu ở RBAC₀. Chỉ leo lên cấp trên khi có nhu cầu thật, không leo vì nghe hay.</text>
</svg>
</div>

## Role hierarchy trong thực tế

Kế thừa role (`RBAC₁`) rất hấp dẫn nhưng có một bẫy: **nó nhân số lượng query lên**. Kiểm tra permission giờ không còn là một phép join phẳng mà là duyệt cây.

Hai cách cài đặt:

**Cách A — Closure table (nên dùng)**: lưu sẵn *mọi* cặp tổ tiên–hậu duệ, kể cả gián tiếp.

```sql
CREATE TABLE role_hierarchy (
  parent_role_id INT NOT NULL,   -- role cấp cao hơn
  child_role_id  INT NOT NULL,   -- role được kế thừa quyền
  depth          INT NOT NULL,   -- 0 = chính nó, 1 = con trực tiếp
  PRIMARY KEY (parent_role_id, child_role_id),
  KEY idx_rh_child (child_role_id)
);
```

Query permission vẫn là một phép join phẳng, không đệ quy — đổi lại phải cập nhật bảng này mỗi khi cấu trúc role đổi (hiếm khi xảy ra).

**Cách B — Recursive CTE**: giữ cấu trúc cây đơn giản (`roles.parent_id`) và duyệt bằng `WITH RECURSIVE`. Đỡ phải bảo trì, nhưng mỗi lần check là một query đệ quy — không nên đặt ở hot path nếu không cache.

> **Lời khuyên**: với dưới ~20 role, đừng làm hierarchy. Cứ gán permission phẳng cho từng role. Chi phí trùng lặp một chút permission rẻ hơn nhiều so với chi phí debug một cây kế thừa sai.

---

# Cách hoạt động — luồng một request

Đây là phần nối RBAC với [JWT](/posts/jwt/). Luồng đầy đủ của một request đã đăng nhập:

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="sq-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="sq-arrG" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<marker id="sq-arrR" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#FF6B6B"/></marker>
<filter id="sq-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="560" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Một request đi qua tầng phân quyền</text>
<rect x="34" y="46" width="150" height="40" rx="9" fill="#374151" filter="url(#sq-sh)"/>
<rect x="250" y="46" width="200" height="40" rx="9" fill="#0D9488" filter="url(#sq-sh)"/>
<rect x="520" y="46" width="150" height="40" rx="9" fill="#DC2626" filter="url(#sq-sh)"/>
<rect x="730" y="46" width="140" height="40" rx="9" fill="#6366F1" filter="url(#sq-sh)"/>
<text x="109" y="71" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Client</text>
<text x="350" y="71" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">API — Middleware</text>
<text x="595" y="71" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Redis (cache)</text>
<text x="800" y="71" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">MySQL</text>
<line x1="109" y1="88" x2="109" y2="500" stroke="#D6D3D1" stroke-width="2" stroke-dasharray="5 5"/>
<line x1="350" y1="88" x2="350" y2="500" stroke="#D6D3D1" stroke-width="2" stroke-dasharray="5 5"/>
<line x1="595" y1="88" x2="595" y2="440" stroke="#D6D3D1" stroke-width="2" stroke-dasharray="5 5"/>
<line x1="800" y1="88" x2="800" y2="400" stroke="#D6D3D1" stroke-width="2" stroke-dasharray="5 5"/>
<line x1="112" y1="112" x2="346" y2="112" stroke="#57534E" stroke-width="2" marker-end="url(#sq-arr)"/>
<text x="229" y="106" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">① GET /transactions + Bearer JWT</text>
<rect x="258" y="130" width="184" height="42" rx="8" fill="#E6FBF9" stroke="#0D9488" stroke-width="1.8"/>
<text x="350" y="148" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">② AuthN: verify chữ ký + exp</text>
<text x="350" y="164" text-anchor="middle" font-size="10" fill="#0F766E">không chạm DB — sub = user_42</text>
<line x1="353" y1="196" x2="591" y2="196" stroke="#57534E" stroke-width="2" marker-end="url(#sq-arr)"/>
<text x="472" y="190" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">③ GET perms:v7:user_42</text>
<line x1="591" y1="226" x2="353" y2="226" stroke="#0D9488" stroke-width="2" stroke-dasharray="4 3" marker-end="url(#sq-arrG)"/>
<text x="472" y="220" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">④a HIT → trả về ngay (~0.2ms)</text>
<rect x="150" y="248" width="700" height="132" rx="10" fill="#FFF9F0" stroke="#F8D48A" stroke-width="1.5" stroke-dasharray="6 4"/>
<text x="172" y="270" font-size="11" font-weight="800" fill="#7C4A03">④b Nếu MISS — chỉ xảy ra lần đầu hoặc sau khi cache bị xoá</text>
<line x1="353" y1="296" x2="796" y2="296" stroke="#D97706" stroke-width="2" marker-end="url(#sq-arr)"/>
<text x="574" y="290" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">⑤ JOIN user_roles → role_permissions → permissions</text>
<line x1="796" y1="324" x2="353" y2="324" stroke="#6366F1" stroke-width="2" stroke-dasharray="4 3" marker-end="url(#sq-arr)"/>
<text x="574" y="318" text-anchor="middle" font-size="10" font-weight="700" fill="#4338CA">⑥ ["transaction:read", "budget:read"]</text>
<line x1="353" y1="356" x2="591" y2="356" stroke="#D97706" stroke-width="2" marker-end="url(#sq-arr)"/>
<text x="472" y="350" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">⑦ SETEX perms:v7:user_42 300</text>
<rect x="232" y="398" width="236" height="44" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="350" y="416" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">⑧ AuthZ: có "transaction:read"?</text>
<text x="350" y="432" text-anchor="middle" font-size="10" fill="#92400E">so khớp với permission route yêu cầu</text>
<line x1="346" y1="466" x2="112" y2="466" stroke="#0D9488" stroke-width="2.2" marker-end="url(#sq-arrG)"/>
<text x="229" y="460" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">⑨ CÓ → 200 OK + dữ liệu</text>
<line x1="346" y1="492" x2="112" y2="492" stroke="#FF6B6B" stroke-width="2.2" marker-end="url(#sq-arrR)"/>
<text x="229" y="486" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">⑨' KHÔNG → 403 Forbidden</text>
<rect x="500" y="452" width="380" height="72" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="520" y="474" font-size="11" font-weight="800" fill="#374151">Vì sao có chữ "v7" trong cache key?</text>
<text x="520" y="494" font-size="10" fill="#57534E">Đổi permission của một ROLE ảnh hưởng tới MỌI user có role đó.</text>
<text x="520" y="512" font-size="10" font-weight="700" fill="#0F766E">Tăng version → toàn bộ cache cũ thành mồ côi, tự hết hạn. 1 lệnh.</text>
<text x="450" y="546" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Điểm mấu chốt: AuthN đọc từ TOKEN, AuthZ đọc từ SERVER — không tin quyền ghi trong token</text>
</svg>
</div>

Điểm quan trọng nhất của sơ đồ trên nằm ở dòng cuối: **danh tính lấy từ token, nhưng quyền thì tra ở server**. Chi tiết vì sao nằm ở phần tiếp theo.

---

# Implement với JWT, Go, MySQL và Redis

## Schema MySQL

```sql
CREATE TABLE roles (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  code        VARCHAR(50)  NOT NULL,          -- 'accountant' — dùng trong code
  name        VARCHAR(100) NOT NULL,          -- 'Kế toán viên' — hiển thị cho người
  description VARCHAR(255) NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_roles_code (code)
) ENGINE=InnoDB;

CREATE TABLE permissions (
  id       INT AUTO_INCREMENT PRIMARY KEY,
  code     VARCHAR(100) NOT NULL,             -- 'transaction:read'
  resource VARCHAR(50)  NOT NULL,             -- 'transaction'
  action   VARCHAR(50)  NOT NULL,             -- 'read'
  UNIQUE KEY uk_permissions_code (code),
  KEY idx_permissions_resource (resource)
) ENGINE=InnoDB;

CREATE TABLE user_roles (
  user_id    BIGINT   NOT NULL,
  role_id    INT      NOT NULL,
  granted_by BIGINT   NULL,                   -- ai cấp — phục vụ audit trail
  granted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expires_at DATETIME NULL,                   -- NULL = vĩnh viễn; có giá trị = quyền tạm
  PRIMARY KEY (user_id, role_id),
  KEY idx_user_roles_role (role_id)
) ENGINE=InnoDB;

CREATE TABLE role_permissions (
  role_id       INT NOT NULL,
  permission_id INT NOT NULL,
  PRIMARY KEY (role_id, permission_id),
  KEY idx_rp_permission (permission_id)
) ENGINE=InnoDB;
```

Ba chi tiết đáng chú ý trong schema này:

1. **`PRIMARY KEY (user_id, role_id)`** vừa là khoá chính vừa là index cho truy vấn "user này có role nào" (nhờ tính chất leftmost prefix — xem [bài về composite index](/posts/mysql-index-composite-vs-single/)). Nhưng truy vấn ngược — "role này đang gán cho ai" — cần index riêng `idx_user_roles_role`. Thiếu nó, mọi màn hình quản trị liệt kê thành viên của một role sẽ quét toàn bảng.
2. **`expires_at`** biến "cấp quyền tạm" từ một quy trình thủ công (và hay quên) thành một ràng buộc dữ liệu. Nhớ đưa điều kiện hết hạn vào **mọi** query resolve permission.
3. **`granted_by`** là phần của audit trail. Với hệ thống tài chính, "ai cấp quyền này cho ai, lúc nào" là câu hỏi kiểm toán viên sẽ hỏi.

Query resolve toàn bộ permission của một user:

```sql
SELECT DISTINCT p.code
FROM   user_roles       ur
JOIN   role_permissions rp ON rp.role_id       = ur.role_id
JOIN   permissions      p  ON p.id             = rp.permission_id
WHERE  ur.user_id = ?
  AND (ur.expires_at IS NULL OR ur.expires_at > NOW());
```

Đây là một `eq_ref` join chuỗi ba bảng qua khoá chính — rẻ, nhưng chạy **mỗi request** thì vẫn là gánh nặng không cần thiết. Đó là lý do có Redis ở bước sau.

## Đặt gì vào JWT? Ba chiến lược

Đây là quyết định thiết kế quan trọng nhất, và cũng là chỗ dễ sai nhất.

```json
// Chiến lược A — chứa ROLE
{ "sub": "user_42", "roles": ["accountant"], "exp": 1790000000 }

// Chiến lược B — chứa PERMISSION đã flatten
{ "sub": "user_42", "perms": ["transaction:read","transaction:write","budget:read"], "exp": ... }

// Chiến lược C — chỉ chứa danh tính
{ "sub": "user_42", "exp": 1790000000 }
```

| | **A — roles trong token** | **B — permissions trong token** | **C — chỉ user_id** |
|---|---|---|---|
| Kích thước token | Nhỏ | **Lớn** — phình theo số permission | Nhỏ nhất |
| Tra cứu mỗi request | 1 lần role→perm (cache được) | **0** | 1 lần user→perm (cache được) |
| Đổi **permission của role** | Hiệu lực **ngay** | Chờ token hết hạn | Hiệu lực **ngay** |
| Đổi **role của user** | Chờ token hết hạn | Chờ token hết hạn | Hiệu lực **ngay** |
| Thu hồi quyền tức thì | ✗ | ✗ | ✓ |
| Hoạt động khi store chết | ✓ | ✓ | ✗ |
| Phù hợp với | Hệ thống phân tán, nhiều service | Edge/gateway không có store | **Đa số ứng dụng web** |

**Khuyến nghị**: dùng **C + cache Redis**. Lý do:

- Token nhỏ và ổn định. Header HTTP bị giới hạn ~8KB ở hầu hết reverse proxy (nginx mặc định `4 × 8k`); một user có 200 permission là đủ để làm hỏng request theo cách rất khó debug.
- Thu hồi quyền có hiệu lực ngay. Đây không phải chuyện lý thuyết: khi một tài khoản bị chiếm hoặc một nhân viên bị cho nghỉ việc, "chờ 15 phút nữa token hết hạn" là câu trả lời không chấp nhận được với hệ thống tài chính.
- Chi phí thực tế gần như bằng không nhờ cache — một lần `GET` Redis khoảng 0,2ms, so với vài chục ms của chính request.

Chiến lược **A** là trung gian hợp lý nếu bạn có nhiều service và không muốn mọi service đều phụ thuộc vào một store phân quyền chung.

> 🚫 **Tuyệt đối không dùng B nếu chưa hiểu rõ hệ quả.** Nó nghĩa là: trong khoảng thời gian bằng TTL của token, **bạn không có cách nào gỡ quyền của một người**. Xem thêm phần thu hồi token trong [bài về JWT](/posts/jwt/#23-bốn-cách-vá-và-giá-của-từng-cách).

## Code Go — permission là kiểu có tên, không phải string trần

```go
// internal/domain/auth/permission.go
package auth

// Permission là kiểu riêng, không dùng string trần — để compiler bắt lỗi
// truyền nhầm tham số, và để gom mọi permission vào một chỗ duy nhất.
type Permission string

const (
    PermTransactionRead   Permission = "transaction:read"
    PermTransactionCreate Permission = "transaction:create"
    PermTransactionDelete Permission = "transaction:delete"
    PermBudgetRead        Permission = "budget:read"
    PermBudgetUpdate      Permission = "budget:update"
    PermUserManage        Permission = "user:manage"
)

// PermissionSet dùng map thay vì slice: kiểm tra O(1), và tự khử trùng lặp
// khi một user có nhiều role cùng cấp một permission.
type PermissionSet map[Permission]struct{}

func (s PermissionSet) Has(p Permission) bool {
    _, ok := s[p]
    return ok
}

// Resolver nằm ở domain layer dưới dạng interface — implementation
// (MySQL + Redis) thuộc infrastructure. Xem thêm bài về Layered Architecture.
type PermissionResolver interface {
    PermissionsOf(ctx context.Context, userID string) (PermissionSet, error)
}
```

Vì sao dùng kiểu `Permission` thay vì `string`? Vì `RequirePermission("transaction:raed")` sẽ compile bình thường và fail âm thầm lúc chạy, còn `RequirePermission(auth.PermTransactionRaed)` fail ngay lúc build. Với code phân quyền, đổi lỗi runtime thành lỗi compile là một món hời.

## Middleware Gin

```go
// internal/presentation/auth/middleware.go
package auth

const ContextUserID = "auth.user_id"

// RequirePermission trả về middleware yêu cầu user phải có ĐỦ TẤT CẢ
// các permission được liệt kê (quan hệ AND).
func RequirePermission(
    resolver domainauth.PermissionResolver,
    logger   *zap.Logger,
    required ...domainauth.Permission,
) gin.HandlerFunc {
    if len(required) == 0 {
        // Fail-fast lúc khởi động: một route "yêu cầu 0 permission" gần như
        // luôn là lỗi gõ nhầm, và nó tạo ra một lỗ hổng im lặng.
        panic("RequirePermission: phải khai báo ít nhất 1 permission")
    }

    return func(c *gin.Context) {
        // AuthMiddleware (chạy trước) đã verify JWT và set giá trị này.
        userID := c.GetString(ContextUserID)
        if userID == "" {
            errors.RespondError(c, logger,
                shared.NewDomainError(shared.ErrCodeUnauthenticated, "thiếu danh tính"))
            c.Abort()
            return
        }

        granted, err := resolver.PermissionsOf(c.Request.Context(), userID)
        if err != nil {
            // Lỗi khi tra quyền => TỪ CHỐI, không bao giờ cho qua.
            // "Fail open" ở tầng phân quyền là lỗi bảo mật, không phải
            // một sự đánh đổi về tính sẵn sàng.
            errors.RespondError(c, logger, err)
            c.Abort()
            return
        }

        for _, need := range required {
            if !granted.Has(need) {
                logger.Info("từ chối truy cập",
                    zap.String("user_id", userID),
                    zap.String("missing_permission", string(need)),
                    zap.String("path", c.FullPath()))
                errors.RespondError(c, logger,
                    shared.NewDomainError(shared.ErrCodeForbidden,
                        "thiếu quyền: "+string(need)))
                c.Abort()
                return
            }
        }
        c.Next()
    }
}
```

Hai quyết định trong đoạn code trên đáng nói rõ:

- **`panic` khi danh sách permission rỗng.** Đây là panic lúc *đăng ký route*, tức lúc khởi động app — không phải lúc phục vụ request. Nó biến một lỗi gõ nhầm im lặng thành một crash ngay khi deploy, trước khi có user nào đi qua route đó.
- **Lỗi khi tra quyền thì từ chối.** Nếu Redis và MySQL cùng chết, hệ thống trả 500 cho mọi request — khó chịu, nhưng an toàn. Phương án ngược lại ("store chết thì cho qua") biến một sự cố hạ tầng thành một sự cố bảo mật.

Đăng ký route:

```go
tx := r.Group("/transactions", AuthMiddleware(jwtVerifier))
{
    tx.GET("",        RequirePermission(res, log, auth.PermTransactionRead),   h.List)
    tx.POST("",       RequirePermission(res, log, auth.PermTransactionCreate), h.Create)
    tx.DELETE("/:id", RequirePermission(res, log, auth.PermTransactionDelete), h.Delete)
}
```

## Resolver có cache — và bài toán invalidation

```go
// internal/infrastructure/auth/cached_resolver.go
type cachedResolver struct {
    db    *persistence.LoggingDB
    rdb   *redis.Client
    ttl   time.Duration       // 5 phút là điểm cân bằng hợp lý
    log   *zap.Logger
}

func (r *cachedResolver) PermissionsOf(
    ctx context.Context, userID string,
) (auth.PermissionSet, error) {
    ver, err := r.globalVersion(ctx)
    if err != nil {
        return nil, err
    }
    key := fmt.Sprintf("perms:v%d:%s", ver, userID)

    if raw, err := r.rdb.Get(ctx, key).Result(); err == nil {
        return decodeSet(raw), nil
    } else if !errors.Is(err, redis.Nil) {
        return nil, err   // lỗi Redis thật => trả lỗi, KHÔNG âm thầm bỏ qua cache
    }

    perms, err := r.loadFromDB(ctx, userID)
    if err != nil {
        return nil, err
    }
    // Lỗi ghi cache không nên làm hỏng request — chỉ log lại.
    if err := r.rdb.Set(ctx, key, encodeSet(perms), r.ttl).Err(); err != nil {
        r.log.Warn("không ghi được cache permission", zap.Error(err))
    }
    return perms, nil
}
```

Phần khó không phải là đọc cache, mà là **xoá cache đúng lúc**. Có hai loại thay đổi, và chúng cần hai cách xử lý khác nhau:

```go
// (1) Đổi role CỦA MỘT USER  -> chỉ cần xoá cache của đúng user đó
func (r *cachedResolver) InvalidateUser(ctx context.Context, userID string) error {
    ver, err := r.globalVersion(ctx)
    if err != nil {
        return err
    }
    return r.rdb.Del(ctx, fmt.Sprintf("perms:v%d:%s", ver, userID)).Err()
}

// (2) Đổi permission CỦA MỘT ROLE -> ảnh hưởng MỌI user có role đó.
//     Không dùng KEYS/SCAN để quét và xoá (chậm, và block Redis).
//     Chỉ cần tăng version: toàn bộ key cũ lập tức thành mồ côi và
//     tự biến mất theo TTL.
func (r *cachedResolver) InvalidateAll(ctx context.Context) error {
    return r.rdb.Incr(ctx, "perms:version").Err()
}
```

Kỹ thuật **version key** này đáng nhớ: nó biến một thao tác xoá hàng loạt (có thể là hàng trăm nghìn key) thành **một lệnh `INCR` duy nhất**. Đánh đổi là bộ nhớ Redis tạm thời giữ cả key cũ lẫn mới trong khoảng một TTL — chấp nhận được với TTL 5 phút.

> ⚠️ **Đừng dùng `KEYS perms:*` rồi `DEL`.** `KEYS` quét toàn bộ keyspace và **block Redis single-thread** trong suốt thời gian quét. Trên production với vài triệu key, đó là một sự cố ngừng dịch vụ tự gây ra.

## Thu hồi quyền tức thì với JWT

Nếu bạn buộc phải đặt role vào JWT (chiến lược A), vẫn có cách rút ngắn độ trễ thu hồi: thêm claim `pv` (permission version) vào token và so với giá trị lưu ở server.

```go
// Lúc phát hành token
claims := jwt.MapClaims{
    "sub": userID,
    "pv":  currentPermVersion(ctx, userID),  // đọc từ Redis/DB
    "exp": time.Now().Add(15 * time.Minute).Unix(),
}

// Lúc verify — thêm một bước so sánh
if claims["pv"] != currentPermVersion(ctx, userID) {
    // Quyền đã đổi sau khi token được phát hành => buộc refresh
    return ErrTokenStale
}
```

Nhưng nhìn kỹ: bước này **vẫn phải tra store mỗi request**. Nếu đằng nào cũng phải tra, thì tra thẳng permission (chiến lược C) vừa đơn giản hơn vừa chính xác hơn. Đây là lý do C thường thắng trong thực tế — lợi thế "stateless" của việc nhét quyền vào JWT sẽ biến mất ngay khi bạn cần thu hồi tức thì.

---

# So sánh với các mô hình khác

RBAC không phải mô hình duy nhất, và quan trọng hơn — **nó không phải mô hình đủ cho mọi bài toán**.

| | **ACL** | **RBAC** | **ABAC** | **ReBAC** |
|---|---|---|---|---|
| Trả lời câu hỏi | User X có trong danh sách của resource Y? | Role của user có permission P? | Thuộc tính có thoả policy? | Có đường quan hệ từ user tới resource? |
| Đơn vị gán quyền | (user, resource) | (user, role) + (role, perm) | rule / policy | quan hệ (tuple) |
| Quyền theo **từng bản ghi** | ✓ | **✗** | ✓ | ✓ |
| Số bản ghi phải lưu | `O(user × resource)` 💥 | `O(user + role)` | `O(số rule)` | `O(số quan hệ)` |
| Trả lời "ai xem được X?" | Dễ | Dễ | **Rất khó** | Trung bình |
| Độ phức tạp vận hành | Thấp | **Thấp** | Cao | Cao |
| Ví dụ thật | Quyền file Unix, S3 ACL | Kubernetes RBAC, GitHub | AWS IAM `Condition`, XACML | Google Drive, Zanzibar |

**ABAC** (Attribute-Based) quyết định dựa trên thuộc tính của chủ thể, tài nguyên và môi trường: *"cho phép nếu `user.department == document.department` VÀ giờ hiện tại trong khung hành chính VÀ IP thuộc dải nội bộ"*. Cực kỳ linh hoạt, nhưng có một nhược điểm chí mạng: **không trả lời được câu hỏi ngược**. Muốn biết "ai đang xem được tài liệu này", bạn phải duyệt qua mọi user và chạy toàn bộ policy engine cho từng người.

**ReBAC** (Relationship-Based) mô hình hoá quyền thành đồ thị quan hệ: *user → thành viên của → team → editor của → folder → chứa → document*. Đây là cách Google Drive hoạt động, và đã được chuẩn hoá qua bài báo [Google Zanzibar](https://research.google/pubs/pub48190/) (2019). Các cài đặt mã nguồn mở: **OpenFGA**, **SpiceDB**, **Ory Keto**.

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 520" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="dc-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="dc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="520" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chọn mô hình phân quyền nào?</text>
<rect x="300" y="50" width="260" height="42" rx="10" fill="#374151" filter="url(#dc-sh)"/>
<text x="430" y="76" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">Cần phân quyền cho hệ thống</text>
<rect x="272" y="122" width="316" height="50" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#dc-sh)"/>
<text x="430" y="143" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Quyền có phụ thuộc vào TỪNG BẢN GHI</text>
<text x="430" y="161" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">cụ thể không? (ai sở hữu, ai được chia sẻ)</text>
<rect x="40" y="208" width="270" height="72" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#dc-sh)"/>
<text x="175" y="234" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">RBAC</text>
<text x="175" y="254" text-anchor="middle" font-size="11" fill="#0F766E">Quyền theo loại tài nguyên là đủ</text>
<text x="175" y="271" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">👉 Bắt đầu từ đây. Luôn luôn.</text>
<rect x="470" y="208" width="350" height="50" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#dc-sh)"/>
<text x="645" y="229" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Quan hệ có NHIỀU TẦNG không?</text>
<text x="645" y="247" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">(team → folder → file → chia sẻ tiếp)</text>
<rect x="470" y="296" width="350" height="76" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#dc-sh)"/>
<text x="645" y="322" text-anchor="middle" font-size="14" font-weight="800" fill="#3730A3">ReBAC</text>
<text x="645" y="342" text-anchor="middle" font-size="11" fill="#4338CA">Đồ thị quan hệ — Zanzibar / OpenFGA</text>
<text x="645" y="359" text-anchor="middle" font-size="11" font-weight="700" fill="#4338CA">Google Drive, Notion, Figma</text>
<rect x="470" y="396" width="350" height="76" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#dc-sh)"/>
<text x="645" y="422" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">RBAC + kiểm tra ownership</text>
<text x="645" y="442" text-anchor="middle" font-size="11" fill="#BE123C">RBAC gác cổng route, domain check chủ sở hữu</text>
<text x="645" y="459" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">👉 Lựa chọn của hầu hết ứng dụng thật</text>
<rect x="40" y="316" width="270" height="156" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="60" y="340" font-size="12" font-weight="800" fill="#374151">Còn ABAC thì sao?</text>
<text x="60" y="362" font-size="11" fill="#57534E">Dùng khi quyền phụ thuộc NGỮ CẢNH:</text>
<text x="60" y="380" font-size="11" fill="#57534E">giờ làm việc, dải IP, mức rủi ro,</text>
<text x="60" y="398" font-size="11" fill="#57534E">trạng thái của chính bản ghi.</text>
<text x="60" y="424" font-size="11" font-weight="700" fill="#9F1239">Nhược điểm lớn nhất:</text>
<text x="60" y="442" font-size="11" fill="#9F1239">không trả lời được câu hỏi ngược</text>
<text x="60" y="460" font-size="11" fill="#9F1239">"ai đang xem được tài nguyên X?"</text>
<line x1="430" y1="94" x2="430" y2="118" stroke="#57534E" stroke-width="2" marker-end="url(#dc-arr)"/>
<path d="M272,147 L175,147 L175,204" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#dc-arr)"/>
<path d="M588,147 L645,147 L645,204" stroke="#57534E" stroke-width="2" fill="none" marker-end="url(#dc-arr)"/>
<path d="M500,260 L500,292" stroke="#6366F1" stroke-width="2" fill="none" marker-end="url(#dc-arr)"/>
<path d="M790,260 L790,330 L790,392" stroke="#FF6B6B" stroke-width="2" fill="none" marker-end="url(#dc-arr)"/>
<rect x="192" y="136" width="52" height="20" rx="5" fill="#0D9488"/>
<rect x="596" y="136" width="62" height="20" rx="5" fill="#57534E"/>
<rect x="452" y="266" width="52" height="20" rx="5" fill="#6366F1"/>
<rect x="742" y="286" width="62" height="20" rx="5" fill="#FF6B6B"/>
<text x="218" y="151" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="627" y="151" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<text x="478" y="281" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<text x="773" y="301" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="430" y="500" text-anchor="middle" font-size="12" font-weight="700" fill="#57534E">Không có mô hình nào "tốt nhất" — chỉ có mô hình khớp với hình dạng dữ liệu của bạn.</text>
</svg>
</div>

---

# Ưu điểm và nhược điểm

## Ưu điểm

| Ưu điểm | Vì sao quan trọng |
|---|---|
| **Dễ hiểu với người không phải kỹ sư** | Trưởng phòng nhân sự hiểu được "gán role Kế toán", họ không hiểu được policy XACML |
| **Onboarding / offboarding nhanh** | Nhân viên mới: gán 1-2 role là xong. Nghỉ việc: gỡ hết role, một thao tác |
| **Audit dễ** | "Ai là admin?" = một câu query. Với ABAC, câu này gần như không trả lời được |
| **Tách policy khỏi code** | Đổi quyền của một role là đổi dữ liệu, không phải deploy lại |
| **Là chuẩn công nghiệp** | Có sẵn trong Kubernetes, Spring Security, Django, Laravel... không phải tự phát minh |
| **Chi phí kiểm tra rẻ** | Một phép tra tập hợp, `O(1)` — không có policy engine nào phải chạy |

## Nhược điểm

### 1. Role explosion — vấn đề nghiêm trọng nhất

Mỗi khi xuất hiện một **chiều** nhu cầu mới, số role không cộng thêm mà **nhân lên**.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="re-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#FF6B6B"/></marker>
<marker id="re-arrT" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="re-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="440" fill="#FFFDF7" rx="16"/>
<rect x="20" y="18" width="430" height="404" rx="14" fill="#FFF7F7" stroke="#FFC9CC" stroke-width="1.5"/>
<rect x="466" y="18" width="414" height="404" rx="14" fill="#F2FCFB" stroke="#9BE5DC" stroke-width="1.5"/>
<text x="235" y="44" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">❌ Bệnh: nhân role theo từng chiều</text>
<text x="673" y="44" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">✅ Thuốc: tách chiều ra khỏi role</text>
<rect x="44" y="60" width="118" height="34" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<rect x="180" y="60" width="118" height="34" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<rect x="316" y="60" width="110" height="34" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<text x="103" y="82" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">3 phòng ban</text>
<text x="239" y="82" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">4 cấp bậc</text>
<text x="371" y="82" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">2 vùng</text>
<text x="171" y="82" text-anchor="middle" font-size="14" font-weight="800" fill="#BE123C">×</text>
<text x="307" y="82" text-anchor="middle" font-size="14" font-weight="800" fill="#BE123C">×</text>
<rect x="100" y="112" width="270" height="44" rx="9" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5"/>
<text x="235" y="140" text-anchor="middle" font-size="16" font-weight="800" fill="#9F1239">= 24 role phải tạo và bảo trì</text>
<text x="235" y="180" text-anchor="middle" font-size="11" font-family="monospace" fill="#BE123C">sales_manager_north · sales_manager_south</text>
<text x="235" y="198" text-anchor="middle" font-size="11" font-family="monospace" fill="#BE123C">sales_staff_north  · sales_staff_south</text>
<text x="235" y="216" text-anchor="middle" font-size="11" font-family="monospace" fill="#BE123C">finance_manager_north · finance_manager_south</text>
<text x="235" y="234" text-anchor="middle" font-size="11" font-family="monospace" fill="#A8A29E">... và 18 role nữa</text>
<rect x="60" y="256" width="350" height="46" rx="9" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2"/>
<text x="235" y="277" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Thêm 1 phòng ban = thêm 8 role</text>
<text x="235" y="294" text-anchor="middle" font-size="11" fill="#BE123C">Thêm 1 vùng = thêm 12 role</text>
<rect x="60" y="318" width="350" height="86" rx="9" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="1.8"/>
<text x="80" y="340" font-size="11" font-weight="800" fill="#9F1239">Hậu quả thật:</text>
<text x="80" y="360" font-size="11" fill="#BE123C">• Không ai còn biết role nào cấp quyền gì</text>
<text x="80" y="378" font-size="11" fill="#BE123C">• Admin gán đại role gần đúng → quyền thừa</text>
<text x="80" y="396" font-size="11" fill="#BE123C">• Least Privilege sụp đổ trên thực tế</text>
<rect x="490" y="60" width="176" height="34" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="684" y="60" width="172" height="34" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="578" y="82" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">4 role (theo cấp bậc)</text>
<text x="770" y="82" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">+ scope là DỮ LIỆU</text>
<rect x="490" y="112" width="366" height="44" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="673" y="140" text-anchor="middle" font-size="16" font-weight="800" fill="#134E4A">= 4 role, không phải 24</text>
<rect x="490" y="172" width="366" height="106" rx="9" fill="#FFFFFF" stroke="#14B8A6" stroke-width="1.8"/>
<text x="510" y="194" font-size="11" font-weight="800" fill="#134E4A">user_roles giờ mang thêm phạm vi:</text>
<text x="510" y="218" font-size="11" font-family="monospace" fill="#0F766E">(user_7, role=manager, dept=sales, region=north)</text>
<text x="510" y="238" font-size="11" font-family="monospace" fill="#0F766E">(user_7, role=staff,   dept=finance, region=south)</text>
<text x="510" y="262" font-size="11" fill="#0F766E">Role trả lời "LÀM GÌ", scope trả lời "Ở ĐÂU"</text>
<rect x="490" y="294" width="366" height="110" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="510" y="316" font-size="11" font-weight="800" fill="#134E4A">Vì sao hiệu quả:</text>
<text x="510" y="336" font-size="11" fill="#0F766E">• Thêm phòng ban / vùng = thêm DỮ LIỆU, không thêm role</text>
<text x="510" y="354" font-size="11" fill="#0F766E">• Số role giữ nguyên khi tổ chức phình to</text>
<text x="510" y="372" font-size="11" fill="#0F766E">• Đây chính là RBAC lai một chút ABAC</text>
<text x="510" y="394" font-size="11" font-weight="700" fill="#0F766E">• Đánh đổi: mọi phép check phải mang theo ngữ cảnh scope</text>
<line x1="235" y1="98" x2="235" y2="108" stroke="#FF6B6B" stroke-width="2" marker-end="url(#re-arr)"/>
<line x1="673" y1="98" x2="673" y2="108" stroke="#0D9488" stroke-width="2" marker-end="url(#re-arrT)"/>
</svg>
</div>

### 2. Không biểu diễn được quyền theo từng bản ghi

Đây là nhược điểm bị hiểu lầm nhiều nhất, và là **nguyên nhân số một khiến hệ thống có RBAC đầy đủ vẫn dính lỗi A01**.

RBAC trả lời được: *"Role này có được đọc giao dịch không?"*
RBAC **không** trả lời được: *"User này có được đọc giao dịch **này** không?"*

```go
// ✅ RBAC đã làm đúng phần của nó: user có permission transaction:read
// ❌ Nhưng đây vẫn là lỗ hổng IDOR — đọc được giao dịch của người khác!
func (h *Handler) Get(c *gin.Context) {
    txn, err := h.repo.GetByID(c, c.Param("id"))
    c.JSON(200, txn)      // không hề kiểm tra txn có thuộc về người gọi không
}
```

RBAC là **cửa vào toà nhà**, không phải **khoá từng căn phòng**. Cần thêm một tầng kiểm tra quyền sở hữu ở tầng domain:

```go
func (h *Handler) Get(c *gin.Context) {
    userID := c.GetString(auth.ContextUserID)
    txn, err := h.repo.GetByID(c, c.Param("id"))
    if err != nil { /* ... */ }

    // Tầng thứ hai: RBAC cho biết ĐƯỢC ĐỌC GIAO DỊCH,
    // domain service cho biết ĐƯỢC ĐỌC GIAO DỊCH NÀY.
    if !h.privacyService.CanUserSeeTransaction(userID, txn) {
        errors.RespondError(c, h.log,
            shared.NewDomainError(shared.ErrCodeForbidden, "không có quyền với bản ghi này"))
        return
    }
    c.JSON(200, h.present(userID, txn))
}
```

Trong dự án Honeydue mình đang làm, đây chính là lý do tồn tại của `PrivacyService` ở tầng domain: RBAC (hoặc bất kỳ kiểm tra nào ở tầng route) không thể biết một giao dịch có `privacy_level = hidden` và người đang xem là bạn đời chứ không phải chủ sở hữu. **Phân quyền theo loại tài nguyên nằm ở middleware; phân quyền theo từng bản ghi nằm ở domain.** Trộn lẫn hai tầng này là công thức tạo lỗ hổng.

### 3. Các nhược điểm còn lại

| Nhược điểm | Biểu hiện | Cách giảm nhẹ |
|---|---|---|
| Không xử lý được **ngữ cảnh** | Không diễn đạt được "chỉ trong giờ hành chính", "chỉ từ IP nội bộ" | Lai thêm ABAC cho đúng những rule cần |
| Thay đổi cần **thao tác thủ công** | Quyền không tự động theo dữ liệu (thăng chức phải có người vào gán role) | Đồng bộ từ hệ thống HR, hoặc dùng SCIM |
| **Độ trễ thu hồi** khi dùng JWT | Gỡ role xong người dùng vẫn truy cập được tới khi token hết hạn | Chiến lược C + cache, hoặc TTL ngắn |
| Role **phình dần** | Role tích tụ permission qua năm tháng, không ai gỡ | Review định kỳ + `expires_at` cho quyền tạm |

---

# Use case thực tế

## 1. Kubernetes RBAC — mô hình sách giáo khoa

Kubernetes là ví dụ RBAC sạch nhất đang chạy ở quy mô lớn. Bốn đối tượng, ánh xạ gần như một-một với mô hình lý thuyết:

| Kubernetes | Tương ứng trong RBAC |
|---|---|
| `Role` (trong một namespace) | Tập permission, có phạm vi |
| `ClusterRole` (toàn cluster) | Tập permission, toàn cục |
| `RoleBinding` | Gán role cho user/group/service-account |
| `ClusterRoleBinding` | Gán role toàn cục |

```yaml
# Role: định nghĩa ĐƯỢC LÀM GÌ, trong namespace nào
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: honeydue
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]      # chính là "action" trong resource:action
---
# RoleBinding: gán role đó CHO AI
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: honeydue
  name: read-pods
subjects:
- kind: ServiceAccount
  name: monitoring
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Hai điểm đáng học từ thiết kế của Kubernetes:

- **Hoàn toàn cộng dồn (purely additive) — không có luật `deny`.** Không có quyền nghĩa là bị từ chối, chấm hết. Điều này loại bỏ hẳn một lớp bug kinh điển: thứ tự ưu tiên giữa allow và deny. Khi mọi thứ chỉ là phép hợp, việc suy luận "ai có quyền gì" trở nên đơn giản.
- **Tách `Role` (có namespace) khỏi `ClusterRole` (toàn cục).** Chính là kỹ thuật "tách scope ra khỏi role" đã nói ở trên, được đưa thẳng vào thiết kế API.

## 2. AWS IAM — không phải RBAC thuần, và biết điều đó rất quan trọng

IAM hay bị gọi là RBAC vì có chữ "role", nhưng **IAM role không phải RBAC role** — nó là một danh tính có thể *assume*, không phải một tập permission gán cho người. Bản thân cơ chế quyết định của IAM gần với **PBAC/ABAC** hơn:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "IpAddress":    { "aws:SourceIp": "10.0.0.0/8" },
    "StringEquals": { "aws:PrincipalTag/department": "finance" }
  }
}
```

Khối `Condition` chính là ABAC. Và khác với Kubernetes, IAM **có `Deny` tường minh, và `Deny` luôn thắng `Allow`** — mạnh hơn, nhưng cũng khiến việc trả lời "rốt cuộc user này có quyền gì" trở nên khó hơn nhiều (đó là lý do AWS phải làm riêng công cụ IAM Policy Simulator).

## 3. GitHub — RBAC phân cấp trong thực tế

GitHub dùng đúng `RBAC₁`: `Read ⊂ Triage ⊂ Write ⊂ Maintain ⊂ Admin`. Mỗi cấp bao trùm toàn bộ quyền của cấp dưới. Người dùng chỉ phải chọn một trong năm mức thay vì tích vào ba mươi ô permission.

Bài học: **role phân cấp là một quyết định về trải nghiệm người dùng nhiều hơn là về kỹ thuật.** Nó đánh đổi tính linh hoạt (không thể tạo ra "Write nhưng không được xoá branch") lấy sự đơn giản — và với đa số người dùng, đó là đánh đổi đúng.

## 4. SaaS đa tenant — role phải gắn với tenant

Đây là chỗ nhiều SaaS làm sai ngay từ đầu. Một user có thể là **Admin ở tổ chức A và Viewer ở tổ chức B**. Nếu `user_roles` chỉ có `(user_id, role_id)`, bạn không diễn đạt được điều này — và nếu cố diễn đạt bằng cách tạo role `admin_org_A`, bạn đã bước thẳng vào role explosion.

```sql
CREATE TABLE user_roles (
  user_id   BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,          -- chiều thứ ba, BẮT BUỘC
  role_id   INT    NOT NULL,
  PRIMARY KEY (user_id, tenant_id, role_id),
  KEY idx_ur_tenant_role (tenant_id, role_id)
) ENGINE=InnoDB;
```

Và mọi phép kiểm tra permission **bắt buộc** phải mang theo tenant:

```go
// ❌ Thiếu ngữ cảnh — sẽ cho qua nếu user là admin ở BẤT KỲ tổ chức nào
if perms.Has(auth.PermUserManage) { /* ... */ }

// ✅ Quyền luôn được tra trong phạm vi một tenant cụ thể
perms, err := resolver.PermissionsOf(ctx, userID, tenantID)
if perms.Has(auth.PermUserManage) { /* ... */ }
```

Lỗi "thiếu tenant trong phép kiểm tra quyền" là một trong những lỗ hổng nghiêm trọng nhất của SaaS đa tenant — nó cho phép rò rỉ dữ liệu **xuyên khách hàng**. Hãy đưa `tenant_id` vào *chữ ký hàm* để compiler bắt buộc mọi call site phải cung cấp nó, thay vì để nó là một tham số tuỳ chọn.

## 5. Quản trị nội bộ / back-office

Nơi RBAC toả sáng rõ nhất: tập người dùng nhỏ và ổn định, chức năng công việc rõ ràng, yêu cầu audit cao. Bộ role điển hình:

```
support_l1        → đọc thông tin khách, không đọc dữ liệu tài chính
support_l2        → + đọc giao dịch, tạo yêu cầu hoàn tiền (không duyệt)
finance_operator  → duyệt hoàn tiền dưới hạn mức
finance_manager   → duyệt không giới hạn hạn mức
auditor           → đọc TẤT CẢ, ghi KHÔNG GÌ CẢ
```

Để ý role `auditor`: chỉ đọc, không ghi bất cứ gì. Đây là ứng dụng trực tiếp của SoD — người kiểm tra hệ thống không được có khả năng thay đổi thứ mình đang kiểm tra.

---

# Anti-pattern thường gặp

## Anti-pattern 1 — Kiểm tra ROLE thay vì kiểm tra PERMISSION

```go
// ❌ Tên role bị đóng cứng trong code nghiệp vụ
if user.Role == "admin" || user.Role == "finance_manager" {
    approveRefund()
}

// ✅ Code chỉ quan tâm tới KHẢ NĂNG, không quan tâm ai có khả năng đó
if perms.Has(auth.PermRefundApprove) {
    approveRefund()
}
```

Vì sao quan trọng: với cách viết đầu, thêm một role mới được duyệt hoàn tiền đòi hỏi **sửa code và deploy lại** — và phải tìm cho hết mọi chỗ có chuỗi `"admin"` nằm rải rác trong codebase. Với cách thứ hai, đó chỉ là một dòng `INSERT` vào `role_permissions`.

Có đúng **một** ngoại lệ hợp lệ: màn hình quản trị chính các role đó (UI quản lý role), nơi tên role là dữ liệu nghiệp vụ thật.

## Anti-pattern 2 — Phân quyền chỉ ở frontend

Ẩn nút "Xoá" trong React là **trải nghiệm người dùng**, không phải bảo mật. Kẻ tấn công không dùng UI của bạn — họ gọi thẳng API bằng `curl`. Quy tắc: **mọi phép kiểm tra ở frontend phải có một phép kiểm tra tương ứng ở backend.** Frontend quyết định *hiển thị gì*; backend quyết định *cho phép gì*.

## Anti-pattern 3 — Gán permission thẳng cho user

```sql
CREATE TABLE user_permissions (   -- 🚩 dấu hiệu mô hình đang bị phá vỡ
  user_id       BIGINT,
  permission_id INT
);
```

Bảng này luôn xuất hiện với lý do "chỉ là trường hợp đặc biệt tạm thời". Sáu tháng sau nó có 4.000 dòng và không ai dám đụng vào. Khi thấy nhu cầu này, hãy tự hỏi: *có bao nhiêu người cần đúng tổ hợp quyền này?* Nếu nhiều hơn một, đó là một **role mới**, hãy đặt tên cho nó.

## Anti-pattern 4 — Không ghi log các lần từ chối

Một loạt 403 liên tiếp từ cùng một tài khoản là tín hiệu tấn công rõ ràng nhất mà bạn có được. Không log lại, bạn mất luôn tín hiệu đó. Nhớ [quy tắc không log dữ liệu nhạy cảm](/posts/owasp/) — ghi `user_id`, permission còn thiếu và route, **không** ghi số tiền hay nội dung bản ghi.

## Anti-pattern 5 — Fail open khi store phân quyền chết

```go
// ❌ Redis chết => tất cả mọi người thành admin
perms, err := resolver.PermissionsOf(ctx, userID)
if err != nil {
    c.Next()   // "tạm cho qua để hệ thống không sập"
    return
}
```

Đây là cách một sự cố hạ tầng biến thành một vụ rò rỉ dữ liệu. Tầng phân quyền phải **luôn fail closed**.

## Anti-pattern 6 — Route mới quên gắn middleware

Lỗi phổ biến nhất trong thực tế, và may mắn là dễ chặn nhất — bằng một bài test liệt kê mọi route đã đăng ký và khẳng định chúng từ chối request ẩn danh:

```go
func TestMoiRouteDeuYeuCauXacThuc(t *testing.T) {
    router := buildRouter(testDeps(t))

    // Danh sách trắng TƯỜNG MINH — thêm route public phải sửa file này,
    // tức là phải đi qua code review.
    public := map[string]bool{
        "GET /health":         true,
        "POST /auth/login":    true,
        "POST /auth/register": true,
        "POST /auth/refresh":  true,
    }

    for _, route := range router.Routes() {
        key := route.Method + " " + route.Path
        if public[key] {
            continue
        }
        t.Run(key, func(t *testing.T) {
            // Gọi KHÔNG kèm token
            req := httptest.NewRequest(route.Method, concretePath(route.Path), nil)
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            require.Equal(t, http.StatusUnauthorized, w.Code,
                "route %s cho qua khi không có token — thiếu AuthMiddleware?", key)
        })
    }
}

// concretePath thay tham số động bằng giá trị giả: "/transactions/:id" -> "/transactions/1"
func concretePath(p string) string {
    parts := strings.Split(p, "/")
    for i, seg := range parts {
        if strings.HasPrefix(seg, ":") || strings.HasPrefix(seg, "*") {
            parts[i] = "1"
        }
    }
    return strings.Join(parts, "/")
}
```

Bài test này rẻ (chạy vài chục ms) và bắt được đúng cái lỗi hay xảy ra nhất: ai đó thêm một endpoint mới lúc 6 giờ chiều thứ Sáu và quên mất middleware. Danh sách trắng tường minh là phần quan trọng nhất — nó biến "mở một route ra public" thành một thay đổi hiện rõ trong diff.

---

# Checklist triển khai

```txt
THIẾT KẾ
□ Permission đặt tên theo tài nguyên nghiệp vụ, không theo endpoint
□ Role đặt tên theo chức năng công việc, không theo tên người/phòng ban cụ thể
□ Bắt đầu ở RBAC₀ phẳng — chỉ thêm hierarchy khi đã > 20 role
□ Đa tenant: tenant_id nằm trong user_roles VÀ trong chữ ký hàm check quyền

DỮ LIỆU
□ user_roles có expires_at cho quyền cấp tạm
□ user_roles có granted_by phục vụ audit
□ Có index cho cả hai chiều truy vấn (user→role và role→user)
□ Mọi query resolve permission đều lọc điều kiện hết hạn

CODE
□ Permission là kiểu có tên, không phải string trần
□ Kiểm tra PERMISSION, không kiểm tra tên ROLE
□ Mọi route khai báo permission tường minh — deny by default
□ Lỗi khi tra quyền => từ chối (fail closed), không bao giờ cho qua
□ Kiểm tra quyền sở hữu từng bản ghi ở tầng domain, tách khỏi RBAC ở middleware

VẬN HÀNH
□ Cache permission có cơ chế invalidate cho CẢ hai loại thay đổi (user và role)
□ Không dùng KEYS/SCAN để xoá cache hàng loạt — dùng version key
□ Ghi log mọi lần từ chối 403 (user_id + permission thiếu + route)
□ Có test khẳng định mọi route không nằm trong whitelist đều trả 401
□ Có quy trình review quyền định kỳ (access recertification)
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| RBAC là gì? | Gán quyền cho **vai trò**, gán vai trò cho **người** — không nối thẳng người với quyền |
| 401 hay 403? | 401 = chưa xác thực · 403 = đã xác thực nhưng không đủ quyền |
| Bắt đầu từ đâu? | `RBAC₀` phẳng: user ↔ role ↔ permission. Đừng làm hierarchy sớm |
| Đặt role vào JWT được không? | Được, nhưng khi đó **không thể thu hồi quyền tức thì**. Mặc định nên: JWT chỉ chứa `sub`, permission tra ở server + cache Redis |
| Xoá cache permission thế nào? | Đổi role của user → xoá key của user. Đổi permission của role → **tăng version key**, không dùng `KEYS` |
| RBAC có chặn được IDOR không? | **Không.** RBAC là cửa toà nhà; khoá từng phòng phải làm ở tầng domain |
| Role explosion xử lý sao? | Tách các chiều (phòng ban, vùng, tenant) ra thành **dữ liệu scope**, không thành role mới |
| Khi nào RBAC không đủ? | Khi quyền phụ thuộc từng bản ghi (→ ReBAC) hoặc phụ thuộc ngữ cảnh (→ ABAC) |
| Lỗi hay gặp nhất? | Route mới quên middleware — chặn bằng test liệt kê route |

---

# Tham khảo

- [NIST — Role Based Access Control](https://csrc.nist.gov/projects/role-based-access-control) (chuẩn ANSI INCITS 359)
- Sandhu, Coyne, Feinstein, Youman — *Role-Based Access Control Models* (IEEE Computer, 1996) — nguồn gốc của RBAC₀–RBAC₃
- [Kubernetes — Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Google Zanzibar: A Global Authorization System](https://research.google/pubs/pub48190/) — nền tảng của ReBAC
- [OWASP — Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- Bài liên quan trên blog: [OWASP Top 10](/posts/owasp/) · [JWT: Những Điều Cần Biết Trước Khi Chọn Nó](/posts/jwt/) · [MySQL Index: Composite vs Single](/posts/mysql-index-composite-vs-single/)
