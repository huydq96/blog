+++
date = '2026-08-19T09:00:00+07:00'
draft = false
slug = 'ddd-p1'
title = 'DDD Là Gì? (Và Vì Sao Honeydue Cần Nó)'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "ddd", "fintech", "architecture", "honeydue"]
description = 'Domain-Driven Design giải thích dễ hiểu bằng ví dụ thật: layered architecture và bounded context trong project Honeydue'
+++

Đây là **Phần 1** của series DDD (Domain-Driven Design), nằm trong series lớn hơn tôi đang viết về [Honeydue](https://github.com/huydq96) — một app quản lý tài chính cho các cặp đôi mà tôi đang tự rebuild bằng Go. Nếu bạn chưa quen khái niệm DDD, đọc [Phần 0 — DDD Là Gì? Tổng Quan Cho Người Mới](/posts/ddd-p0/) trước sẽ dễ theo hơn. Từ bài này trở đi, thay vì giải thích DDD kiểu hàn lâm, tôi sẽ dùng chính code trong Honeydue làm ví dụ.

**Lưu ý**: project vẫn đang phát triển. Bài này chỉ mô tả những gì *đã* implement (domain layer + application layer + infrastructure của 3 bounded context: Transaction, Budget, Couple). Chỗ nào chưa xong tôi sẽ ghi rõ, không vẽ ra thứ chưa tồn tại.

## Vấn đề DDD giải quyết

Tưởng tượng bạn viết handler cho API tạo transaction, không theo DDD:

```go
func CreateTransactionHandler(c *gin.Context) {
    var req Request
    c.ShouldBindJSON(&req)

    if req.PrivacyLevel != "hidden" && req.PrivacyLevel != "balance_only" && req.PrivacyLevel != "full" {
        c.JSON(400, gin.H{"error": "invalid privacy level"})
        return
    }
    if req.Amount < 0 {
        c.JSON(400, gin.H{"error": "invalid amount"})
        return
    }

    db.Exec("INSERT INTO transactions (...) VALUES (...)", ...)

    // tính luôn spending tháng này, kiểm tra threshold, ngay trong handler
    var spend int64
    db.QueryRow("SELECT SUM(amount) FROM transactions WHERE couple_id = ? AND category = ?", ...).Scan(&spend)
    if spend > budgetLimit*80/100 {
        sendNotification(...)
    }

    c.JSON(200, gin.H{"ok": true})
}
```

Code này chạy được. Vấn đề là 6 tháng sau, khi cần thêm 1 cách tạo transaction khác (ví dụ import từ bank), toàn bộ validate + business rule ở trên phải copy-paste lại, vì nó bị khoá cứng vào tầng HTTP. Business rule quan trọng nhất của cả app — "ai được thấy transaction của ai" — nằm rải rác trong handler, dễ quên, dễ sai, không test riêng được.

**DDD** (Eric Evans, 2003) giải quyết đúng vấn đề này: đặt toàn bộ tri thức nghiệp vụ (domain knowledge) vào một tầng riêng — **domain layer** — độc lập với HTTP, độc lập với SQL. Handler chỉ còn việc parse request và gọi xuống. Trong Honeydue, đoạn logic tương đương ở trên nằm gọn trong 1 domain service, test được độc lập, không cần Gin, không cần MySQL:

```go
// internal/domain/transaction/service.go
func (s *PrivacyService) CanUserViewTransaction(viewer, txnOwner string, privacy PrivacyLevel) (bool, error) {
    if viewer == txnOwner {
        return true, nil
    }
    return !privacy.IsHidden(), nil
}
```

## Ubiquitous Language — nói chung một thứ tiếng

Nguyên tắc đầu tiên của DDD: dev và người hiểu nghiệp vụ (product, hoặc chính bạn khi đóng vai "chủ sản phẩm") phải dùng **chung một từ vựng**, và từ vựng đó phải xuất hiện y hệt trong code — không dịch qua dịch lại. Honeydue định nghĩa từ vựng này ngay trong `CLAUDE.md`:

| Thuật ngữ | Định nghĩa |
|---|---|
| **Couple Account** | Account liên kết 2 user để quản lý tài chính chung |
| **Transaction** | 1 khoản chi tiêu, có mức độ riêng tư |
| **Privacy Level** | `hidden` (chỉ chủ sở hữu thấy) \| `balance_only` (chỉ thấy số tiền) \| `full` (thấy hết) |
| **Budget** | Hạn mức chi tiêu theo category, theo tháng/năm |
| **Budget Threshold** | Ngưỡng cảnh báo khi chi tiêu chạm 80% hạn mức |

Không hề ngẫu nhiên mà type trong Go tên là `PrivacyLevel`, field tên là `CoupleID`, hàm tên là `CheckThreshold`. Khi đọc design doc nói "budget threshold", bạn tìm thẳng ra `BudgetThresholdReachedEvent` trong code — không phải đoán xem "cái ngưỡng cảnh báo" đó nằm ở biến nào.

## Layered Architecture — 4 tầng, 1 chiều phụ thuộc

Honeydue chia code thành 4 tầng. Luật duy nhất và quan trọng nhất: **tầng ngoài phụ thuộc vào tầng trong, không bao giờ ngược lại**. Domain là lõi — nó không được import bất cứ thứ gì từ Infrastructure hay Presentation.

<div class="zoomable-diagram" style="max-width:720px;margin:2rem auto;">
<svg viewBox="0 0 720 580" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p1a-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p1a-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="p1a-arrow-teal" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/>
</marker>
<marker id="p1a-arrow-red" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#E11D48"/>
</marker>
</defs>
<rect x="0" y="0" width="720" height="580" fill="#FFFDF7" rx="16"/>
<rect x="140" y="20" width="440" height="70" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p1a-shadow)"/>
<text x="360" y="48" text-anchor="middle" font-size="18" font-weight="700" fill="#134E4A">Presentation</text>
<text x="360" y="70" text-anchor="middle" font-size="12.5" fill="#0F766E">Gin handler · middleware · request/response DTO</text>
<line x1="360" y1="90" x2="360" y2="128" stroke="#57534E" stroke-width="2" marker-end="url(#p1a-arrow)"/>
<text x="380" y="113" font-size="11.5" fill="#57534E">gọi</text>
<rect x="140" y="130" width="440" height="70" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p1a-shadow)"/>
<text x="360" y="158" text-anchor="middle" font-size="18" font-weight="700" fill="#7C4A03">Application</text>
<text x="360" y="180" text-anchor="middle" font-size="12.5" fill="#92400E">Command · DTO · Service — chỉ orchestration, không business logic</text>
<line x1="360" y1="200" x2="360" y2="238" stroke="#57534E" stroke-width="2" marker-end="url(#p1a-arrow)"/>
<text x="380" y="223" font-size="11.5" fill="#57534E">gọi</text>
<rect x="110" y="240" width="500" height="110" rx="18" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#p1a-shadow)"/>
<text x="360" y="274" text-anchor="middle" font-size="20" font-weight="800" fill="#9F1239">Domain</text>
<text x="360" y="298" text-anchor="middle" font-size="12.5" fill="#BE123C">Aggregate · Entity · Value Object · Domain Service</text>
<text x="360" y="317" text-anchor="middle" font-size="12.5" fill="#BE123C">Repository — chỉ là interface, không biết SQL là gì</text>
<rect x="466" y="248" width="130" height="24" rx="12" fill="#FF6B6B"/>
<text x="531" y="264" text-anchor="middle" font-size="11" font-weight="700" fill="#ffffff">CORE — bất biến</text>
<rect x="140" y="480" width="440" height="70" rx="14" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#p1a-shadow)"/>
<text x="360" y="508" text-anchor="middle" font-size="18" font-weight="700" fill="#134E4A">Infrastructure</text>
<text x="360" y="530" text-anchor="middle" font-size="12.5" fill="#0F766E">MySQL repository impl · Kafka publisher · Redis cache</text>
<line x1="470" y1="480" x2="470" y2="352" stroke="#0D9488" stroke-width="2.5" marker-end="url(#p1a-arrow-teal)"/>
<text x="486" y="410" font-size="11.5" fill="#0D9488">implements</text>
<text x="486" y="425" font-size="11.5" fill="#0D9488">Repository interface</text>
<line x1="250" y1="352" x2="250" y2="478" stroke="#E11D48" stroke-width="2.5" stroke-dasharray="6 5" marker-end="url(#p1a-arrow-red)"/>
<circle cx="250" cy="415" r="14" fill="#ffffff" stroke="#E11D48" stroke-width="2"/>
<text x="250" y="420" text-anchor="middle" font-size="14" font-weight="800" fill="#E11D48">✕</text>
<text x="150" y="450" font-size="11.5" fill="#E11D48">Domain KHÔNG BAO GIỜ</text>
<text x="150" y="465" font-size="11.5" fill="#E11D48">import package infrastructure</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 1: Layered Architecture của Honeydue — mọi mũi tên chỉ vào Domain, không bao giờ ngược lại.</p>

Chú ý chiều mũi tên teal ở dưới: **Infrastructure phụ thuộc vào Domain**, chứ không phải Domain phụ thuộc Infrastructure. Repository interface được *định nghĩa* trong domain layer (`internal/domain/transaction/repository.go`), còn `MysqlTransactionRepository` ở infrastructure layer chỉ *implement* interface đó. Domain hoàn toàn không biết MySQL tồn tại. Đây là **Dependency Inversion** — lý do vì sao sau này có thể đổi MySQL sang Postgres, hoặc tách transaction context thành 1 service riêng mà domain layer không cần sửa 1 dòng nào.

## Bounded Context — biên giới của từng mảnh nghiệp vụ

Một domain lớn (toàn bộ app Honeydue) được chia nhỏ thành các **bounded context** — mỗi context có model, ngôn ngữ, và trách nhiệm riêng, chỉ giao tiếp với nhau qua interface rõ ràng (domain event, hoặc gọi trực tiếp qua domain service). Honeydue hiện có 3 context CORE đã implement:

<div class="zoomable-diagram" style="max-width:760px;margin:2rem auto;">
<svg viewBox="0 0 760 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p1b-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p1b-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#B45309"/>
</marker>
<marker id="p1b-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#9CA3AF"/>
</marker>
</defs>
<rect x="0" y="0" width="760" height="460" fill="#FFFDF7" rx="16"/>
<rect x="280" y="20" width="200" height="80" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p1b-shadow)"/>
<text x="380" y="52" text-anchor="middle" font-size="15" font-weight="700" fill="#134E4A">Couple Context</text>
<text x="380" y="72" text-anchor="middle" font-size="11" fill="#0F766E">coupleID — không gian chứa 2 user</text>
<line x1="345" y1="100" x2="200" y2="200" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#p1b-arrow-grey)"/>
<text x="230" y="150" font-size="11" fill="#9CA3AF">couple_id</text>
<line x1="415" y1="100" x2="560" y2="200" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#p1b-arrow-grey)"/>
<text x="480" y="150" font-size="11" fill="#9CA3AF">couple_id</text>
<rect x="40" y="200" width="280" height="120" rx="16" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#p1b-shadow)"/>
<text x="180" y="232" text-anchor="middle" font-size="15" font-weight="700" fill="#9F1239">Transaction Context</text>
<text x="180" y="254" text-anchor="middle" font-size="11" fill="#BE123C">TransactionAggregate</text>
<text x="180" y="270" text-anchor="middle" font-size="11" fill="#BE123C">PrivacyLevel · PrivacyService</text>
<rect x="130" y="282" width="100" height="20" rx="10" fill="#FF6B6B"/>
<text x="180" y="296" text-anchor="middle" font-size="10" font-weight="700" fill="#ffffff">phần khó nhất</text>
<rect x="440" y="200" width="280" height="120" rx="16" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p1b-shadow)"/>
<text x="580" y="232" text-anchor="middle" font-size="15" font-weight="700" fill="#7C4A03">Budget Context</text>
<text x="580" y="254" text-anchor="middle" font-size="11" fill="#92400E">BudgetAggregate</text>
<text x="580" y="270" text-anchor="middle" font-size="11" fill="#92400E">BudgetService.CheckThreshold</text>
<path d="M320,255 Q380,215 440,255" stroke="#B45309" stroke-width="2.5" fill="none" marker-end="url(#p1b-arrow)"/>
<text x="380" y="205" text-anchor="middle" font-size="10.5" fill="#B45309">gọi trực tiếp, đồng bộ</text>
<text x="380" y="218" text-anchor="middle" font-size="10.5" fill="#B45309">(chưa qua Kafka)</text>
<rect x="220" y="370" width="320" height="60" rx="12" fill="#F3F4F6" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4"/>
<text x="380" y="394" text-anchor="middle" font-size="12" font-weight="700" fill="#6B7280">Notification / Categorizer Worker</text>
<text x="380" y="412" text-anchor="middle" font-size="10.5" fill="#6B7280">Phase 2 — chưa implement</text>
<line x1="180" y1="320" x2="330" y2="372" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#p1b-arrow-grey)"/>
<text x="170" y="345" font-size="10.5" fill="#9CA3AF">publish qua Kafka</text>
<text x="170" y="358" font-size="10.5" fill="#9CA3AF">(sẵn sàng, chưa có consumer)</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 2: Bản đồ 3 bounded context hiện có trong Honeydue, và phần chưa xong (khung nét đứt).</p>

Vài điểm đáng chú ý khi đọc sơ đồ này:

- **Transaction context** là "phần khó nhất" theo đúng nhận định trong design doc — không phải vì đồng bộ dữ liệu ngân hàng khó, mà vì mô hình `PrivacyLevel` (`hidden`/`balance_only`/`full`) đụng tới ranh giới nhạy cảm nhất của app: 2 người chia sẻ tài chính nhưng không phải chia sẻ *mọi thứ*. Bài 2 sẽ đào sâu context này.
- **Transaction → Budget hiện tại là lời gọi trực tiếp, đồng bộ**, không phải qua Kafka: `CreateTransactionApplicationService` gọi thẳng `budgetService.CheckThreshold(...)` trong cùng 1 request. Đây là điều thật, không phải lý tưởng hoá — 2 context tách biệt về mặt model nhưng vẫn phối hợp trực tiếp ở tầng application, một cách hợp lý cho giai đoạn modular monolith.
- **Notification/Categorizer worker chưa tồn tại.** `TransactionCreatedEvent` đã được publish qua Kafka publisher, nhưng chưa có consumer nào đọc nó. Đây là việc của Phase 2 theo roadmap — tôi vẽ khung nét đứt để không đánh lừa rằng nó đã chạy.

## Tiếp theo

Bài này dừng ở "bức tranh lớn": vì sao cần DDD, ngôn ngữ chung, 4 tầng, và bounded context. Bài 2 [(DDD Building Blocks)](/posts/ddd-p2/) sẽ đi sâu vào từng "khối xây dựng" bên trong tầng Domain — Value Object, Entity, Aggregate, Repository, Domain Service, Domain Event — toàn bộ đều lấy code thật từ `internal/domain/transaction` làm ví dụ, kèm sơ đồ giải phẫu 1 aggregate và luồng 1 request đi từ HTTP đến khi lưu xong.

## Đọc thêm

- Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (cuốn "Blue Book" gốc)
- Vaughn Vernon — *Implementing Domain-Driven Design* (thực dụng hơn, nhiều ví dụ code)
- [martinfowler.com/bliki/DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [martinfowler.com/bliki/BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)

