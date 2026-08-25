+++
date = '2026-08-19T08:00:00+07:00'
draft = false
slug = 'ddd-p0'
title = 'DDD Là Gì? Tổng Quan Dễ Hiểu Cho Người Mới Bắt Đầu'
author = 'Huy Dang Quang'
categories = ["Architecture"]
tags = ["ddd", "architecture"]
description = 'Domain-Driven Design là gì, ra đời để làm gì, tư tưởng cốt lõi, lợi ích, các thành phần và quan hệ giữa chúng — tổng quan dễ hiểu kèm mindmap cho người mới bắt đầu'
+++

Đây là **Phần 0** của series DDD (Domain-Driven Design) — viết trước 2 bài [DDD Là Gì? (Và Vì Sao Honeydue Cần Nó)](/posts/ddd-p1/) và [DDD Building Blocks](/posts/ddd-p2/), vốn đi thẳng vào code thật của [Honeydue](https://github.com/huydq96). Nếu bạn chưa biết DDD là gì, đọc bài này trước — nó trả lời một vài câu hỏi nền tảng nhất mà ai mới tiếp cận DDD cũng có thể sẽ thắc mắc.

<div class="ddd-blueprint-toggle" style="margin:2rem 0;text-align:center;">
<style>
.ddd-blueprint-toggle summary{list-style:none;cursor:pointer;display:inline-block;padding:14px 30px;background:#123a5e;color:#eaf4ff;border:2px solid #ffb454;border-radius:10px;font-weight:700;font-size:15px;}
.ddd-blueprint-toggle summary::-webkit-details-marker{display:none;}
.ddd-blueprint-toggle summary:hover{background:#153f68;}
.ddd-blueprint-toggle details[open] summary{margin-bottom:1.5rem;}
</style>
<details>
<summary>📐 Xem Bản Vẽ DDD — không cần biết code cũng hiểu</summary>
<div style="max-width:960px;margin:0 auto;">
<iframe src="/pages/ddd-blueprint.html" title="Bản Vẽ DDD" loading="lazy" style="width:100%;height:3800px;border:none;border-radius:12px;box-shadow:0 4px 24px rgba(0,0,0,0.18);"></iframe>
<p style="margin-top:0.75rem;"><a href="/pages/ddd-blueprint.html" target="_blank" rel="noopener">Mở toàn màn hình ↗</a></p>
</div>
</details>
</div>

## 1. DDD là gì? (Tư duy thiết kế, hay một tiêu chuẩn?)

DDD **không phải** một chuẩn (standard) — không có tổ chức nào ban hành "spec DDD" để bạn build xong rồi validate đúng/sai. Nó cũng **không phải** framework hay thư viện — không có gì để `import` hay `go get`.

DDD là **một tư duy thiết kế phần mềm** (a design approach), do Eric Evans đúc kết trong cuốn sách *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — dân lập trình hay gọi tắt là "Blue Book". Nó là tập hợp nguyên tắc + pattern trả lời câu hỏi: *tổ chức code thế nào khi nghiệp vụ (business) đủ phức tạp để tự nó trở thành thứ khó nhất trong dự án — khó hơn cả bài toán kỹ thuật?*

So sánh cho dễ hình dung: DDD giống SOLID hay Clean Architecture — một **tư duy** bạn áp dụng vào cách chia module, đặt tên, tổ chức tầng — chứ không phải một công cụ bạn cài vào rồi có ngay.

<div style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 680" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p0m1-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
</defs>
<rect x="0" y="0" width="900" height="680" fill="#FFFDF7" rx="16"/>
<path d="M395,285 Q300,220 220,180" stroke="#0D9488" stroke-width="3" fill="none" stroke-linecap="round"/>
<path d="M505,285 Q600,220 680,180" stroke="#0D9488" stroke-width="3" fill="none" stroke-linecap="round"/>
<path d="M395,395 Q300,460 220,500" stroke="#D97706" stroke-width="3" fill="none" stroke-linecap="round"/>
<path d="M505,395 Q600,460 680,500" stroke="#D97706" stroke-width="3" fill="none" stroke-linecap="round"/>
<circle cx="450" cy="340" r="85" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#p0m1-shadow)"/>
<text x="450" y="332" text-anchor="middle" font-size="26" font-weight="800" fill="#9F1239">DDD</text>
<text x="450" y="356" text-anchor="middle" font-size="11" fill="#BE123C">Domain-Driven Design</text>
<rect x="40" y="40" width="340" height="140" rx="20" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p0m1-shadow)"/>
<text x="210" y="68" text-anchor="middle" font-size="15" font-weight="700" fill="#134E4A">① Vấn đề DDD giải quyết</text>
<text x="60" y="96" font-size="11.5" fill="#0F766E">• Business logic rải rác nhiều nơi</text>
<text x="60" y="118" font-size="11.5" fill="#0F766E">• Anemic Domain Model — chỉ get/set</text>
<text x="60" y="140" font-size="11.5" fill="#0F766E">• Dev &amp; Business lệch ngôn ngữ</text>
<rect x="520" y="40" width="340" height="140" rx="20" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p0m1-shadow)"/>
<text x="690" y="68" text-anchor="middle" font-size="15" font-weight="700" fill="#134E4A">② Tư tưởng cốt lõi</text>
<text x="540" y="96" font-size="11.5" fill="#0F766E">• Domain là trung tâm thiết kế</text>
<text x="540" y="118" font-size="11.5" fill="#0F766E">• Ubiquitous Language — ngôn ngữ chung</text>
<text x="540" y="140" font-size="11.5" fill="#0F766E">• Bounded Context — chia để trị</text>
<rect x="40" y="500" width="340" height="140" rx="20" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p0m1-shadow)"/>
<text x="210" y="528" text-anchor="middle" font-size="15" font-weight="700" fill="#7C4A03">③ Strategic — bức tranh lớn</text>
<text x="60" y="556" font-size="11.5" fill="#92400E">• Ubiquitous Language</text>
<text x="60" y="578" font-size="11.5" fill="#92400E">• Bounded Context · Context Map</text>
<text x="60" y="600" font-size="11.5" fill="#92400E">• Subdomain: Core / Supporting / Generic</text>
<rect x="520" y="500" width="340" height="140" rx="20" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p0m1-shadow)"/>
<text x="690" y="528" text-anchor="middle" font-size="15" font-weight="700" fill="#7C4A03">④ Tactical — chi tiết trong code</text>
<text x="540" y="556" font-size="11.5" fill="#92400E">• Entity · Value Object · Aggregate</text>
<text x="540" y="578" font-size="11.5" fill="#92400E">• Repository · Domain Service</text>
<text x="540" y="600" font-size="11.5" fill="#92400E">• Domain Event · Factory</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 1: Bản đồ tư duy tổng quan — 4 mảnh ghép sẽ được giải thích lần lượt trong bài này.</p>

## 2. DDD ra đời để giải quyết vấn đề gì?

Trước DDD, cách làm phổ biến là thiết kế bắt đầu từ **database** (vẽ ERD trước, map field ra object) hoặc từ **framework** (theo khuôn MVC, business logic nhét vào đâu tiện thì nhét). Với hệ thống nhỏ, cách này ổn. Với hệ thống phức tạp, nó sinh ra một anti-pattern rất phổ biến gọi là **Anemic Domain Model** — object chỉ là cái túi đựng field (`get`/`set`), còn logic thật sự nằm hết ở các class `Service` bên ngoài, tự ý thao túng field của object khác. Hệ quả: mất luôn lợi ích của lập trình hướng đối tượng, nhưng vẫn phải gánh toàn bộ độ phức tạp mà lẽ ra OOP giúp giảm bớt.

DDD ra đời để đảo ngược việc đó: đưa logic **về đúng chỗ của nó** — bên trong chính domain object, dưới dạng behavior thật sự chứ không phải field trần trụi.

<div style="max-width:760px;margin:2rem auto;">
<svg viewBox="0 0 760 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p0m2-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p0m2-arrow-red" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#E11D48"/>
</marker>
<marker id="p0m2-arrow-green" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#16A34A"/>
</marker>
</defs>
<rect x="0" y="0" width="760" height="400" fill="#FFFDF7" rx="16"/>
<text x="190" y="35" text-anchor="middle" font-size="16" font-weight="800" fill="#DC2626">❌ Anemic Model</text>
<rect x="70" y="55" width="240" height="85" rx="12" fill="#F3F4F6" stroke="#9CA3AF" stroke-width="2" filter="url(#p0m2-shadow)"/>
<text x="190" y="80" text-anchor="middle" font-size="13" font-weight="700" fill="#374151">Order (Entity)</text>
<text x="190" y="99" text-anchor="middle" font-size="11" fill="#4B5563">id, amount, status</text>
<text x="190" y="117" text-anchor="middle" font-size="10.5" fill="#6B7280">chỉ get/set — không có logic</text>
<line x1="190" y1="195" x2="190" y2="142" stroke="#E11D48" stroke-width="2.5" stroke-dasharray="6 5" marker-end="url(#p0m2-arrow-red)"/>
<rect x="70" y="200" width="240" height="110" rx="12" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p0m2-shadow)"/>
<text x="190" y="224" text-anchor="middle" font-size="13" font-weight="700" fill="#92400E">OrderService</text>
<text x="190" y="244" text-anchor="middle" font-size="10.5" fill="#92400E">if (order.getStatus()==X)</text>
<text x="190" y="262" text-anchor="middle" font-size="10.5" fill="#92400E">order.setStatus(Y)</text>
<text x="190" y="282" text-anchor="middle" font-size="10.5" fill="#DC2626">→ logic nằm NGOÀI entity</text>
<text x="570" y="35" text-anchor="middle" font-size="16" font-weight="800" fill="#16A34A">✅ Rich Domain Model</text>
<rect x="450" y="55" width="240" height="200" rx="16" fill="#EEF2FF" stroke="#4F46E5" stroke-width="2.5" filter="url(#p0m2-shadow)"/>
<text x="570" y="82" text-anchor="middle" font-size="14" font-weight="700" fill="#3730A3">Order (Aggregate Root)</text>
<text x="570" y="104" text-anchor="middle" font-size="10.5" fill="#4338CA">- id, amount, status (private)</text>
<text x="570" y="128" text-anchor="middle" font-size="11.5" font-weight="700" fill="#4F46E5">+ ship()</text>
<text x="570" y="150" text-anchor="middle" font-size="11.5" font-weight="700" fill="#4F46E5">+ cancel()</text>
<text x="570" y="176" text-anchor="middle" font-size="10.5" fill="#3730A3">→ tự bảo vệ invariant</text>
<text x="570" y="194" text-anchor="middle" font-size="10.5" fill="#3730A3">của chính nó</text>
<line x1="570" y1="310" x2="570" y2="260" stroke="#16A34A" stroke-width="2.5" marker-end="url(#p0m2-arrow-green)"/>
<rect x="450" y="315" width="240" height="60" rx="12" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p0m2-shadow)"/>
<text x="570" y="340" text-anchor="middle" font-size="12.5" font-weight="700" fill="#134E4A">Application Service</text>
<text x="570" y="360" text-anchor="middle" font-size="10.5" fill="#0F766E">order.ship()</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 2: Anemic Model — Service ở ngoài tự ý đọc/ghi field. Rich Domain Model — bên ngoài chỉ được gọi method, entity tự bảo vệ chính nó.</p>

Viết lại thành pseudocode cho dễ so sánh:

```
// Anemic — logic nằm ngoài, entity chỉ là data
if order.getStatus() == "paid" {
    order.setStatus("shipped")
}

// Rich Domain Model — logic nằm trong entity
order.Ship()   // tự kiểm tra status hợp lệ, tự đổi state, tự phát event
```

## 3. Tư tưởng cốt lõi của DDD

4 ý chính, xếp theo mức độ nền tảng:

1. **Domain là trung tâm** — mọi quyết định thiết kế xoay quanh nghiệp vụ trước, công nghệ tính sau. Ngược hẳn tư duy "database-first" hay "framework-first" ở mục 2.
2. **Ubiquitous Language** — dev và người hiểu nghiệp vụ dùng chung 1 từ vựng, từ đó xuất hiện y hệt trong code. Business nói "ngưỡng cảnh báo ngân sách" thì code phải có hẳn 1 khái niệm tên gần như vậy, không phải biến `flag2` hay `limitPct`.
3. **Chia để trị (Bounded Context)** — domain lớn không model được trong 1 khối duy nhất, nên chia thành nhiều context nhỏ, mỗi context có model và ngôn ngữ riêng, chỉ giao tiếp qua ranh giới rõ ràng.
4. **Bảo vệ tính nhất quán nghiệp vụ (invariant)** — mọi thay đổi state phải đi qua đúng "cửa" hợp lệ, để dữ liệu không bao giờ rơi vào trạng thái vi phạm business rule, dù chỉ trong khoảnh khắc.

## 4. Lợi ích khi áp dụng DDD

| Lợi ích | Vì sao |
|---|---|
| Business logic tập trung | Sửa 1 rule chỉ sửa đúng 1 chỗ, không lùng khắp handler/SQL |
| Test nhanh, không cần DB/HTTP | Domain logic không phụ thuộc infra → unit test chạy trong mili-giây |
| Dễ đổi công nghệ | Đổi database, đổi framework — domain layer không cần sửa |
| Giảm hiểu lầm dev ↔ business | Ubiquitous Language làm cầu nối chung |
| Scale được đội ngũ | Mỗi bounded context 1 team làm riêng, ít giẫm chân nhau |

**Mặt trái cần biết**: DDD tốn công hơn hẳn so với CRUD thuần — tạo Value Object, Aggregate, Repository interface cho 1 app "thêm/sửa/xoá" đơn giản là over-engineering. DDD đáng giá khi nghiệp vụ **thật sự phức tạp**: nhiều rule, nhiều ràng buộc, nhiều actor. Một todo-list app không cần DDD.

## 5. Các thành phần trong DDD và quan hệ giữa chúng

Nhìn lại nhánh ③④ ở Hình 1 — Strategic (bức tranh lớn) và Tactical (chi tiết trong code). Phần Tactical là nơi hầu hết người mới bị rối vì có nhiều khái niệm cùng lúc, nên tôi minh hoạ bằng vòng đời thay vì liệt kê khô khan:

<div style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 320" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p0m3-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p0m3-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="p0m3-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#9CA3AF"/>
</marker>
</defs>
<rect x="0" y="0" width="900" height="320" fill="#FFFDF7" rx="16"/>
<rect x="40" y="190" width="200" height="100" rx="16" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p0m3-shadow)"/>
<text x="140" y="222" text-anchor="middle" font-size="14" font-weight="700" fill="#92400E">Factory</text>
<text x="140" y="242" text-anchor="middle" font-size="10.5" fill="#7C4A03">tạo Aggregate hợp lệ</text>
<text x="140" y="260" text-anchor="middle" font-size="9.5" fill="#92400E">NewXAggregate(...)</text>
<line x1="240" y1="240" x2="338" y2="240" stroke="#57534E" stroke-width="2" marker-end="url(#p0m3-arrow)"/>
<text x="289" y="232" text-anchor="middle" font-size="10" fill="#57534E">tạo ra</text>
<rect x="340" y="150" width="220" height="180" rx="20" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#p0m3-shadow)"/>
<text x="450" y="185" text-anchor="middle" font-size="15" font-weight="800" fill="#9F1239">Aggregate</text>
<text x="450" y="205" text-anchor="middle" font-size="10.5" fill="#BE123C">(Entity + Value Object)</text>
<text x="450" y="230" text-anchor="middle" font-size="10.5" fill="#BE123C">vận hành qua method</text>
<text x="450" y="250" text-anchor="middle" font-size="10.5" fill="#BE123C">bảo vệ invariant</text>
<text x="450" y="275" text-anchor="middle" font-size="10.5" font-weight="700" fill="#9F1239">chỉ 1 cửa vào: Root</text>
<line x1="562" y1="240" x2="658" y2="240" stroke="#57534E" stroke-width="2" marker-end="url(#p0m3-arrow)"/>
<text x="610" y="232" text-anchor="middle" font-size="10" fill="#57534E">lưu / lấy lại</text>
<rect x="660" y="190" width="200" height="100" rx="16" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#p0m3-shadow)"/>
<text x="760" y="222" text-anchor="middle" font-size="14" font-weight="700" fill="#134E4A">Repository</text>
<text x="760" y="242" text-anchor="middle" font-size="10.5" fill="#0F766E">Save / GetByID</text>
<text x="760" y="260" text-anchor="middle" font-size="9.5" fill="#0F766E">che giấu SQL, DB...</text>
<line x1="450" y1="150" x2="450" y2="102" stroke="#57534E" stroke-width="2" marker-end="url(#p0m3-arrow)"/>
<text x="465" y="128" font-size="10" fill="#57534E">phát Event</text>
<rect x="370" y="20" width="160" height="80" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#p0m3-shadow)"/>
<text x="450" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">Domain Event</text>
<text x="450" y="72" text-anchor="middle" font-size="10" fill="#4B5563">XCreatedEvent</text>
<text x="450" y="88" text-anchor="middle" font-size="9.5" fill="#6B7280">state vừa đổi</text>
<line x1="530" y1="60" x2="648" y2="60" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#p0m3-arrow-grey)"/>
<text x="590" y="48" text-anchor="middle" font-size="9.5" fill="#9CA3AF">loose coupling</text>
<rect x="650" y="20" width="220" height="80" rx="14" fill="#F9FAFB" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4"/>
<text x="760" y="52" text-anchor="middle" font-size="12" font-weight="700" fill="#6B7280">Bounded Context khác</text>
<text x="760" y="72" text-anchor="middle" font-size="10" fill="#6B7280">(vd: Notification)</text>
<text x="760" y="88" text-anchor="middle" font-size="9.5" fill="#9CA3AF">subscribe qua message queue</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 3: Vòng đời 1 domain object — Factory tạo, Aggregate vận hành, Repository lưu trữ, Domain Event thông báo ra ngoài mà không cần gọi trực tiếp.</p>

Vài điểm bổ sung, lấy từ loạt bài DDD của softwaredesign.vn (liệt kê đầy đủ ở mục Đọc thêm) — vì mỗi khối trong Hình 3 còn có những quyết định thiết kế nhỏ hơn đáng biết:

- **Identity**: nên sinh ID *trước khi lưu*, ngay trong Factory (không chờ database tự tăng) — nhờ vậy Domain Event có thể tham chiếu đến entity ngay lập tức, và ID có thể được model như 1 Value Object riêng (`UserId`) thay vì string trần.
- **Factory**: không phải object nào cũng cần Factory riêng — quyết định dựa trên độ phức tạp của việc khởi tạo là chính, tính đóng gói là phụ. Object đơn giản thì constructor bình thường là đủ.
- **Aggregate**: khi 2 entity có vòng đời khác nhau, nên tách thành 2 aggregate riêng, chỉ tham chiếu nhau qua ID — **không** giữ tham chiếu object trực tiếp. Đây là lý do vì sao 2 aggregate độc lập chỉ nên đồng bộ với nhau qua Domain Event (eventual consistency), thay vì gọi thẳng vào nhau.
- **Domain Event**: chỉ nên được tạo ra bên trong Aggregate Root, Factory, hoặc Domain Service — không bao giờ ở Application layer hay Value Object. Và một cạm bẫy thường gặp: khi *load lại* một aggregate đã tồn tại từ database, đừng dùng lại constructor tạo-mới — nó sẽ vô tình phát lại sự kiện "vừa được tạo" cho một thứ đã tồn tại từ lâu.

## 6. Một ứng dụng "đáp ứng được DDD" khi nào?

Không có bài kiểm tra pass/fail duy nhất — DDD là một dải phổ, áp dụng nhiều hay ít tuỳ độ phức tạp nghiệp vụ. Vài tín hiệu để tự kiểm tra:

- ✅ Đọc code domain layer, người **không biết ngôn ngữ lập trình** vẫn hiểu được nghiệp vụ đang làm gì.
- ✅ Domain layer **test được mà không cần chạy database, không cần start web server**.
- ✅ Đổi hẳn database hoặc đổi hẳn framework HTTP — business rule (domain layer) **không cần sửa**.
- ✅ Mọi thay đổi state đi qua đúng 1 "cửa" hợp lệ — không có chỗ nào gán thẳng field bỏ qua validate.
- ✅ Ranh giới bounded context rõ ràng — context này không tự ý đọc/ghi model nội bộ của context khác.

Thiếu 1-2 tín hiệu không có nghĩa là "sai" — chỉ là đang áp dụng ở mức vừa phải, hợp lý cho quy mô hiện tại. Không có app nào đạt DDD "100%" tuyệt đối; quan trọng là áp dụng đúng liều lượng cho đúng chỗ phức tạp.

## Đi tiếp

Từ đây, 2 bài tiếp theo trong series áp dụng toàn bộ những khái niệm trên vào code Go thật của Honeydue:

- [Phần 1 — DDD Là Gì? (Và Vì Sao Honeydue Cần Nó)](/posts/ddd-p1/): layered architecture, bounded context map thật của Honeydue.
- [Phần 2 — DDD Building Blocks](/posts/ddd-p2/): Value Object, Entity, Aggregate, Repository, Domain Service, Domain Event — từng dòng code thật, kèm sơ đồ giải phẫu aggregate và luồng request end-to-end.

## Đọc thêm

- Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (cuốn "Blue Book" gốc)
- Vaughn Vernon — *Implementing Domain-Driven Design*
- [martinfowler.com/bliki/DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)

Loạt bài DDD tiếng Việt rất chi tiết trên **softwaredesign.vn** — đáng đọc để đào sâu từng khối:

- [DDD Part 1 — Entity](https://softwaredesign.vn/ddd-part-1-entity)
- [Entity & Anemic Domain Model](https://softwaredesign.vn/entity-anemic-domain-model)
- [DDD Part 2 — Identity](https://softwaredesign.vn/ddd-part-2-identity)
- [DDD Part 3 — Factory](https://softwaredesign.vn/ddd-part-3-factory)
- [DDD Part 4 — Aggregate (1)](https://softwaredesign.vn/ddd-part-4-aggregate-1)
- [DDD Part 5 — Aggregate (2)](https://softwaredesign.vn/ddd-part-5-aggregate-2)
- [DDD Part 6 — Domain Event (1)](https://softwaredesign.vn/ddd-part-6-domain-event)
- [DDD Part 7 — Domain Event (2)](https://softwaredesign.vn/ddd-part-7-domain-event-2)
- [DDD Part 8 — Domain Event (3)](https://softwaredesign.vn/ddd-part-8-domain-event-3)
