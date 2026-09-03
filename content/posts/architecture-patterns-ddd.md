+++
date = '2026-08-27T10:00:00+07:00'
draft = false
slug = 'architecture-patterns-ddd'
title = 'Layered, Clean, Hexagonal Architecture: So Sánh Dễ Hiểu (Và DDD Nằm Ở Đâu Trong Đó)'
author = 'Huy Dang Quang'
categories = ["Architecture"]
tags = ["architecture", "ddd", "golang", "clean-architecture", "hexagonal-architecture"]
description = 'Layered, Clean, Hexagonal Architecture khác nhau ở đâu, giống nhau ở đâu, và kết hợp với DDD như thế nào — kèm sơ đồ trực quan và code structure mẫu bằng Go'
+++

Ba cái tên này hay bị nghĩ là ba lựa chọn tách biệt, phải chọn đúng 1 trong 3. Sự thật gần với "3 cách vẽ cùng một ý tưởng" hơn là "3 kiến trúc khác nhau". Bài này so sánh kỹ từng cái, có sơ đồ, có code structure mẫu bằng Go, và trả lời câu hỏi hay bị lẫn lộn nhất: **DDD nằm ở đâu trong bức tranh này?**

Bài này dùng thẳng thuật ngữ Aggregate, Entity, Repository, Domain Service mà không giải thích lại — nếu chưa quen, đọc [series DDD]({{< ref "ddd-p0.md" >}}) trước sẽ dễ theo hơn.

## DDD không phải là 1 trong 3 cái này

Nói trước để khỏi lẫn lộn, vì đây là chỗ tôi thấy nhiều người nhầm nhất.

**DDD trả lời câu hỏi: bên trong lõi nghiệp vụ có những gì, và chúng quan hệ với nhau ra sao** — Aggregate, Entity, Value Object, Domain Service, Domain Event, Bounded Context, Ubiquitous Language.

**Layered / Clean / Hexagonal trả lời một câu hỏi khác: code được chia thành mấy khối, và khối nào được phép phụ thuộc vào khối nào.**

Hai câu hỏi này độc lập với nhau. Bạn có thể áp dụng DDD trong một file `main.go` duy nhất (dở, nhưng làm được), hoặc tổ chức code theo Clean Architecture cho một app không có tí DDD nào (chỉ có CRUD, không Aggregate, không Bounded Context). DDD ra đời năm 2003 (Eric Evans), Clean Architecture là 2012 (Robert C. Martin), Hexagonal là 2005 (Alistair Cockburn) — ba nguồn gốc, ba mối quan tâm khác nhau, nhưng vì chúng bổ sung cho nhau cực kỳ tốt nên thường bị dạy chung, rồi bị hiểu lẫn thành một thứ.

Trong bài này, khi nói "Domain" hay "Core", đó chính là chỗ các building block của DDD (Aggregate, Entity...) sống — bất kể bạn vẽ ranh giới quanh nó theo kiểu nào trong 3 kiểu dưới đây.

## Layered Architecture

Kiểu truyền thống nhất, dạy trong hầu hết giáo trình: chia code thành các tầng xếp chồng, tầng trên gọi xuống tầng dưới.

```
Presentation  (HTTP handler, CLI, gRPC...)
      ↓
Application   (orchestration)
      ↓
Domain        (business logic)
      ↓
Infrastructure (database, message queue...)
```

Cấu trúc thư mục Go — đây chính xác là cách Honeydue đang tổ chức:

```
internal/
├── domain/
│   └── transaction/
│       ├── aggregate.go
│       ├── entity.go
│       ├── repository.go      // interface — Save, GetByID...
│       └── service.go
├── application/
│   └── transaction/
│       └── service.go         // orchestration, gọi domain + repo
├── infrastructure/
│   └── persistence/
│       └── mysql_transaction_repository.go  // implements domain/transaction.Repository
└── presentation/
    └── transaction/
        └── handler.go          // Gin handler
```

**Điểm yếu thật của Layered "thuần"**: nó không *bắt buộc* Dependency Inversion. Trong nhiều sách giáo khoa cũ (kiểu 3-tier ASP.NET: UI / BLL / DAL), tầng Business Logic gọi *thẳng* xuống class ở Data Access, tức Domain phụ thuộc trực tiếp vào Infrastructure — không qua interface nào cả. Domain lúc đó biết cả ORM, biết cả connection string, test domain logic bắt buộc phải có database thật chạy cùng.

<div class="zoomable-diagram" style="max-width:1240px;margin:2rem auto;">
<svg viewBox="0 0 1240 610" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ap-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="ap-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="ap-arrow-amber" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#D97706"/>
</marker>
<marker id="ap-arrow-teal" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/>
</marker>
<marker id="ap-arrow-tealdark" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/>
</marker>
</defs>
<rect x="0" y="0" width="1240" height="610" fill="#FFFDF7" rx="16"/>
<text x="210" y="25" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">LAYERED (truyền thống)</text>
<rect x="50" y="50" width="320" height="55" rx="10" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="210" y="72" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Presentation</text>
<text x="210" y="90" text-anchor="middle" font-size="10" fill="#0F766E">Gin handler</text>
<line x1="210" y1="105" x2="210" y2="128" stroke="#57534E" stroke-width="2" marker-end="url(#ap-arrow-grey)"/>
<rect x="50" y="130" width="320" height="55" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="210" y="152" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Application</text>
<text x="210" y="170" text-anchor="middle" font-size="10" fill="#92400E">orchestration</text>
<line x1="210" y1="185" x2="210" y2="208" stroke="#57534E" stroke-width="2" marker-end="url(#ap-arrow-grey)"/>
<rect x="50" y="210" width="320" height="55" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#ap-shadow)"/>
<text x="210" y="232" text-anchor="middle" font-size="12" font-weight="700" fill="#9F1239">Domain</text>
<text x="210" y="250" text-anchor="middle" font-size="10" fill="#BE123C">Aggregate · Entity · VO</text>
<rect x="50" y="290" width="320" height="55" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="210" y="312" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Infrastructure</text>
<text x="210" y="330" text-anchor="middle" font-size="10" fill="#0F766E">MySQL, Kafka, Redis...</text>
<line x1="210" y1="265" x2="210" y2="288" stroke="#D97706" stroke-width="2.5" stroke-dasharray="5 4" marker-end="url(#ap-arrow-amber)"/>
<rect x="216" y="264" width="180" height="30" rx="4" fill="#FFFDF7"/>
<text x="220" y="277" font-size="9.5" fill="#D97706">⚠ domain có thể lỡ import</text>
<text x="220" y="290" font-size="9.5" fill="#D97706">thẳng infra nếu không kỷ luật</text>
<text x="210" y="368" text-anchor="middle" font-size="10" fill="#D97706">KHÔNG bắt buộc dependency inversion —</text>
<text x="210" y="384" text-anchor="middle" font-size="10" fill="#D97706">domain dễ rò rỉ phụ thuộc xuống dưới</text>
<text x="610" y="25" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">CLEAN ARCHITECTURE</text>
<circle cx="610" cy="250" r="150" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ap-shadow)"/>
<circle cx="610" cy="250" r="112" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<circle cx="610" cy="250" r="75" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<circle cx="610" cy="250" r="38" fill="#FF6B6B" stroke="#BE123C" stroke-width="2"/>
<text x="610" y="254" text-anchor="middle" font-size="12" font-weight="800" fill="#ffffff">Entities</text>
<rect x="460" y="418" width="16" height="16" rx="3" fill="#E6FBF9" stroke="#14B8A6"/>
<text x="482" y="430" font-size="10" fill="#374151">Frameworks &amp; Drivers (Web, DB, UI)</text>
<rect x="460" y="440" width="16" height="16" rx="3" fill="#FFF3DB" stroke="#F5A623"/>
<text x="482" y="452" font-size="10" fill="#374151">Interface Adapters (Controller, Gateway)</text>
<rect x="460" y="462" width="16" height="16" rx="3" fill="#FFE9EC" stroke="#FF6B6B"/>
<text x="482" y="474" font-size="10" fill="#374151">Use Cases</text>
<rect x="460" y="484" width="16" height="16" rx="3" fill="#FF6B6B" stroke="#BE123C"/>
<text x="482" y="496" font-size="10" fill="#374151">Entities (business rule thuần)</text>
<line x1="700" y1="516" x2="670" y2="516" stroke="#0D9488" stroke-width="2.5" marker-end="url(#ap-arrow-teal)"/>
<text x="460" y="520" font-size="10.5" font-weight="700" fill="#0F766E">phụ thuộc LUÔN chỉ vào trong</text>
<text x="1010" y="25" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">HEXAGONAL (Ports &amp; Adapters)</text>
<polygon points="1092,298 1010,345 928,298 928,203 1010,155 1092,203" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#ap-shadow)"/>
<text x="1010" y="245" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">CORE</text>
<text x="1010" y="262" text-anchor="middle" font-size="9.5" fill="#BE123C">(Domain + Use Case)</text>
<rect x="780" y="175" width="110" height="50" rx="8" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="835" y="196" text-anchor="middle" font-size="10.5" font-weight="700" fill="#134E4A">HTTP</text>
<text x="835" y="212" text-anchor="middle" font-size="9.5" fill="#0F766E">Handler</text>
<rect x="780" y="278" width="110" height="50" rx="8" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="835" y="299" text-anchor="middle" font-size="10.5" font-weight="700" fill="#134E4A">CLI /</text>
<text x="835" y="315" text-anchor="middle" font-size="9.5" fill="#0F766E">Test</text>
<rect x="1108" y="175" width="92" height="50" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="1154" y="196" text-anchor="middle" font-size="10.5" font-weight="700" fill="#134E4A">MySQL</text>
<text x="1154" y="212" text-anchor="middle" font-size="9.5" fill="#0F766E">Repo</text>
<rect x="1108" y="278" width="92" height="50" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#ap-shadow)"/>
<text x="1154" y="299" text-anchor="middle" font-size="10.5" font-weight="700" fill="#134E4A">Kafka</text>
<text x="1154" y="315" text-anchor="middle" font-size="9.5" fill="#0F766E">Publisher</text>
<line x1="890" y1="200" x2="924" y2="203" stroke="#0D9488" stroke-width="2" marker-end="url(#ap-arrow-teal)"/>
<line x1="890" y1="303" x2="924" y2="298" stroke="#0D9488" stroke-width="2" marker-end="url(#ap-arrow-teal)"/>
<line x1="1092" y1="203" x2="1106" y2="200" stroke="#0D9488" stroke-width="2" marker-end="url(#ap-arrow-tealdark)"/>
<line x1="1092" y1="298" x2="1106" y2="303" stroke="#0D9488" stroke-width="2" marker-end="url(#ap-arrow-tealdark)"/>
<circle cx="928" cy="203" r="5" fill="#14B8A6" stroke="#ffffff" stroke-width="1.5"/>
<circle cx="928" cy="298" r="5" fill="#14B8A6" stroke="#ffffff" stroke-width="1.5"/>
<circle cx="1092" cy="203" r="5" fill="#0D9488" stroke="#ffffff" stroke-width="1.5"/>
<circle cx="1092" cy="298" r="5" fill="#0D9488" stroke="#ffffff" stroke-width="1.5"/>
<circle cx="838" cy="422" r="5" fill="#14B8A6"/>
<circle cx="838" cy="446" r="5" fill="#14B8A6"/>
<text x="852" y="426" font-size="10" fill="#374151">Port — interface do core định nghĩa</text>
<text x="852" y="450" font-size="10" fill="#374151">Adapter — implementation cụ thể</text>
<text x="838" y="466" font-size="10" fill="#374151">(driving bên trái, driven bên phải)</text>
<text x="620" y="580" text-anchor="middle" font-size="13" font-weight="800" fill="#374151">Domain/Core không bao giờ phụ thuộc ra ngoài — 3 cách vẽ khác nhau, cùng 1 luật phụ thuộc</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 1: Cùng một ý tưởng — "logic nghiệp vụ không phụ thuộc chi tiết kỹ thuật" — vẽ theo 3 kiểu khác nhau.</p>

Layered vẫn dùng tốt, kể cả cho hệ nghiêm túc — miễn là bạn *tự* áp thêm kỷ luật Dependency Inversion (Domain khai báo interface, Infrastructure implement). Đó chính xác là cách Honeydue làm, và tôi sẽ quay lại điểm này ở cuối bài.

## Clean Architecture

Robert C. Martin ("Uncle Bob") công bố năm 2012, vẽ thành 4 vòng tròn đồng tâm. Luật duy nhất và tuyệt đối, ông gọi là **The Dependency Rule**:

> Source code dependency chỉ được phép trỏ **vào trong**. Không có gì ở vòng trong biết bất cứ điều gì về vòng ngoài.

4 vòng, từ trong ra ngoài:

- **Entities** — business rule chung nhất, ít thay đổi nhất. Đây chính là chỗ Aggregate/Entity/VO của DDD sống.
- **Use Cases** — business rule riêng của ứng dụng này (orchestration, giống tầng Application ở Layered).
- **Interface Adapters** — controller, presenter, gateway: chuyển đổi dữ liệu qua lại giữa Use Cases và thế giới bên ngoài.
- **Frameworks & Drivers** — vòng ngoài cùng: web framework, database, UI. Uncle Bob gọi thẳng đây là "chi tiết" (details) — quan trọng về mặt kỹ thuật nhưng không quan trọng về mặt kiến trúc.

Cấu trúc thư mục Go — theo convention phổ biến nhất trong cộng đồng Go (kiểu `bxcodec/go-clean-arch`, project tham khảo Clean Architecture bằng Go được biết đến nhiều nhất):

```
transaction/
├── domain.go                 // Entity + Repository interface + UseCase interface
├── usecase/
│   └── transaction_usecase.go     // implements UseCase, phụ thuộc domain.Repository (interface)
├── repository/
│   └── mysql/
│       └── transaction_repository.go  // implements domain.Repository
└── delivery/
    └── http/
        └── transaction_handler.go     // gọi domain.UseCase (interface)
```

Điểm khác biệt quan trọng nhất so với Layered "thuần": **interface được khai báo ở tầng trong cùng, không phải ở tầng dùng nó**.

```go
// transaction/domain.go — tầng trong cùng, KHÔNG import bất kỳ package nào khác trong app
package transaction

type Transaction struct {
    ID       string
    Amount   int64
    Category string
}

// Repository interface do chính domain khai báo — usecase và delivery chỉ biết đúng interface này
type Repository interface {
    Save(ctx context.Context, t *Transaction) error
    GetByID(ctx context.Context, id string) (*Transaction, error)
}

// UseCase interface — tầng delivery (HTTP handler) chỉ gọi qua interface này
type UseCase interface {
    Create(ctx context.Context, amount int64, category string) (*Transaction, error)
}
```

```go
// transaction/usecase/transaction_usecase.go
package usecase

type transactionUseCase struct {
    repo transaction.Repository // phụ thuộc INTERFACE, chưa từng biết chữ "mysql"
}

func (uc *transactionUseCase) Create(ctx context.Context, amount int64, category string) (*transaction.Transaction, error) {
    t := &transaction.Transaction{ID: uuid.NewString(), Amount: amount, Category: category}
    if err := uc.repo.Save(ctx, t); err != nil {
        return nil, err
    }
    return t, nil
}

var _ transaction.UseCase = (*transactionUseCase)(nil) // compile-time check
```

So với Layered ở phần trước: nhìn thoáng qua giống hệt (vẫn 4 khối, vẫn top-down). Khác biệt nằm ở **ai sở hữu interface**. Ở Layered thuần, Infrastructure có thể tự định nghĩa struct riêng rồi Domain import nó. Ở Clean, interface *luôn* nằm ở tầng trong, tầng ngoài phải code theo đúng khuôn đó — đảo ngược hoàn toàn hướng "biết về nhau".

## Hexagonal Architecture (Ports & Adapters)

Alistair Cockburn công bố năm 2005 — trước cả Clean Architecture 7 năm. Ý tưởng cốt lõi giống hệt The Dependency Rule, nhưng vẽ và gọi tên khác: thay vì vòng tròn, hình lục giác (chỉ mang tính tượng trưng "có nhiều cạnh", không phải đúng 6).

Hai khái niệm cần nhớ:

- **Port** — interface do phần lõi (core) định nghĩa. Có 2 loại:
  - **Primary/Driving port** — cổng mà thế giới bên ngoài dùng để *gọi vào* core (vd: interface `TransactionService`).
  - **Secondary/Driven port** — cổng mà core dùng để *gọi ra* thế giới bên ngoài (vd: interface `TransactionRepository`).
- **Adapter** — implementation cụ thể cắm vào một port. **Driving adapter** (HTTP handler, CLI, gRPC, test) gọi vào qua primary port. **Driven adapter** (MySQL, Kafka, SMTP) bị gọi qua secondary port.

Điểm hay nhất của cách đặt tên này: nó đối xứng. HTTP handler và MySQL repository, dưới góc nhìn của core, **là cùng một thứ** — một adapter cắm vào một port. Core không quan tâm ai đang gọi nó (HTTP, CLI, hay test) hay nó đang nói chuyện với ai (MySQL, Postgres, hay một fake trong bộ nhớ).

Cấu trúc thư mục Go:

```
internal/
├── core/
│   ├── domain/
│   │   └── transaction.go          // Entity + business rule thuần
│   ├── ports/
│   │   ├── driving/
│   │   │   └── transaction_service.go   // primary port
│   │   └── driven/
│   │       └── transaction_repository.go // secondary port
│   └── service/
│       └── transaction_service.go        // implements driving port, dùng driven port
└── adapters/
    ├── driving/
    │   └── http/
    │       └── transaction_handler.go     // gọi driving port
    └── driven/
        └── mysql/
            └── transaction_repository.go  // implements driven port
```

```go
// core/ports/driven/transaction_repository.go — secondary port
package driven

type TransactionRepository interface {
    Save(ctx context.Context, t *domain.Transaction) error
}
```

```go
// core/service/transaction_service.go — implement primary port, dùng secondary port
package service

type TransactionService struct {
    repo driven.TransactionRepository // secondary port — core "cần" cái này, không biết nó là MySQL hay gì
}

func (s *TransactionService) CreateTransaction(ctx context.Context, amount int64) (*domain.Transaction, error) {
    t := domain.NewTransaction(amount)
    return t, s.repo.Save(ctx, t)
}
```

```go
// adapters/driven/mysql/transaction_repository.go — driven adapter
package mysql

type TransactionRepository struct{ db *sql.DB }

func (r *TransactionRepository) Save(ctx context.Context, t *domain.Transaction) error {
    _, err := r.db.ExecContext(ctx, `INSERT INTO transactions (id, amount) VALUES (?, ?)`, t.ID, t.Amount)
    return err
}

var _ driven.TransactionRepository = (*TransactionRepository)(nil) // compile-time: đúng port chưa
```

Đổi `mysql.TransactionRepository` này sang `postgres.TransactionRepository` hay `inmemory.TransactionRepository` (dùng cho test) — `TransactionService` không cần sửa một dòng nào, vì nó chỉ biết `driven.TransactionRepository` là một interface.

## So sánh trực tiếp

| Tiêu chí | Layered (thuần) | Clean | Hexagonal |
|---|---|---|---|
| Tác giả / năm | Truyền thống, không có tác giả cụ thể | Robert C. Martin, 2012 | Alistair Cockburn, 2005 |
| Hình vẽ | Ngăn xếp (stack) | Vòng tròn đồng tâm | Lục giác + cổng |
| Số tầng | Thường cố định 3-4 | Đúng 4 (Entities/Use Cases/Adapters/Frameworks) | Không cố định — bao nhiêu port cũng được |
| Dependency Inversion | Không bắt buộc | Bắt buộc, ở mọi ranh giới | Bắt buộc, đối xứng cả 2 phía |
| Interface được gọi là | "Repository interface" | "Boundary" | "Port" |
| Implementation được gọi là | "Repository implementation" | "Interface Adapter" | "Adapter" |
| Phân biệt chiều gọi vào/ra | Không rõ | Không nhấn mạnh | Rõ ràng — driving vs driven |
| Điểm mạnh riêng | Đơn giản, dễ dạy, dễ onboard | Tư duy 4 vòng rõ ràng, nhiều tài liệu | Vocabulary tách bạch driving/driven — hợp hệ nhiều loại client |

Sự thật ít ai nói thẳng: **trong Go, code sinh ra từ Clean Architecture và từ Hexagonal Architecture gần như giống hệt nhau.** Cả hai đều: interface khai báo ở tầng trong, implementation ở tầng ngoài, dependency luôn trỏ vào trong. Khác biệt chủ yếu là tên thư mục (`usecase/` vs `core/service/`, `repository/` vs `adapters/driven/`) và việc Hexagonal tách rõ driving/driven còn Clean thì gộp chung vào "Interface Adapters". Nếu bạn thấy 2 codebase, một khai là "Clean Architecture", một khai là "Hexagonal Architecture", và cấu trúc dependency giống hệt nhau — cả hai đều đúng, chỉ đang dùng từ điển khác nhau. (Có một biến thể thứ 3 cùng họ, ít được nhắc trong bài này: **Onion Architecture** của Jeffrey Palermo, 2008 — cũng vòng tròn đồng tâm, cũng cùng luật, khác vài chi tiết nhỏ về cách chia ring.)

Điểm phân biệt thật sự chỉ có một: **Layered thuần không bắt buộc Dependency Inversion, hai cái còn lại bắt buộc.** Ngay khi bạn thêm kỷ luật đó vào Layered, ranh giới giữa 3 cái gần như biến mất — chỉ còn khác tên gọi.

## Vậy Honeydue đang dùng gì?

Nhìn lại `CLAUDE.md` và code thật:

```go
// internal/domain/transaction/repository.go
// Interface khai báo Ở TRONG domain — infrastructure implement nó, không phải ngược lại
type Repository interface {
    Save(ctx context.Context, agg *TransactionAggregate) error
    GetByID(ctx context.Context, id string) (*TransactionAggregate, error)
    GetByCoupleID(ctx context.Context, coupleID string) ([]*TransactionAggregate, error)
    Delete(ctx context.Context, id string) error
}
```

```go
// internal/infrastructure/persistence/mysql_transaction_repository.go
// implement interface trên — package này import domain, domain KHÔNG import ngược lại
type MysqlTransactionRepository struct{ db *sql.DB }
```

<div class="zoomable-diagram" style="max-width:820px;margin:2rem auto;">
<svg viewBox="0 0 820 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ap2-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="ap2-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="ap2-arrow-green" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#16A34A"/>
</marker>
</defs>
<rect x="0" y="0" width="820" height="460" fill="#FFFDF7" rx="16"/>
<rect x="60" y="20" width="600" height="65" rx="12" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ap2-shadow)"/>
<text x="360" y="47" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">Presentation</text>
<text x="360" y="65" text-anchor="middle" font-size="10.5" fill="#0F766E" font-family="monospace">internal/presentation/transaction</text>
<line x1="360" y1="85" x2="360" y2="108" stroke="#57534E" stroke-width="2" marker-end="url(#ap2-arrow-grey)"/>
<rect x="60" y="110" width="600" height="65" rx="12" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#ap2-shadow)"/>
<text x="360" y="137" text-anchor="middle" font-size="13" font-weight="700" fill="#7C4A03">Application</text>
<text x="360" y="155" text-anchor="middle" font-size="10.5" fill="#92400E" font-family="monospace">internal/application/transaction</text>
<line x1="360" y1="175" x2="360" y2="198" stroke="#57534E" stroke-width="2" marker-end="url(#ap2-arrow-grey)"/>
<rect x="60" y="200" width="600" height="85" rx="14" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#ap2-shadow)"/>
<text x="360" y="227" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">Domain (core)</text>
<text x="360" y="245" text-anchor="middle" font-size="10.5" fill="#BE123C" font-family="monospace">internal/domain/transaction</text>
<text x="360" y="263" text-anchor="middle" font-size="10.5" font-weight="700" fill="#9F1239">→ Repository INTERFACE định nghĩa ở đây</text>
<rect x="60" y="360" width="600" height="65" rx="12" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#ap2-shadow)"/>
<text x="360" y="387" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">Infrastructure</text>
<text x="360" y="405" text-anchor="middle" font-size="10.5" fill="#0F766E" font-family="monospace">internal/infrastructure/persistence</text>
<line x1="560" y1="360" x2="560" y2="287" stroke="#16A34A" stroke-width="2.5" marker-end="url(#ap2-arrow-green)"/>
<rect x="566" y="300" width="220" height="46" rx="4" fill="#FFFDF7"/>
<text x="570" y="315" font-size="10" font-weight="700" fill="#16A34A">implements Repository ✓</text>
<text x="570" y="330" font-size="9.5" fill="#166534">= Dependency Inversion — giống</text>
<text x="570" y="343" font-size="9.5" fill="#166534">hệt nguyên lý Clean/Hexagonal</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 2: Honeydue gọi tên là "Layered Architecture", nhưng vì có Dependency Inversion ở ranh giới domain-infrastructure, nó đã thực chất áp dụng đúng The Dependency Rule của Clean/Hexagonal.</p>

Nói cách khác: cái tên "Layered Architecture" trong `CLAUDE.md` không sai, nhưng hơi khiêm tốn. Honeydue không phải Layered "thuần" (kiểu 3-tier cũ, domain phụ thuộc thẳng infra) — nó là Layered **có kỷ luật Dependency Inversion**, tức về bản chất dependency graph, nó đã đứng chung hàng với Clean và Hexagonal. Khác biệt còn lại chỉ là tên gọi: Honeydue gọi là "Repository interface", không gọi là "driven port" — nhưng ý nghĩa thì y hệt.

## Chọn cái nào?

- **App nhỏ, CRUD chủ yếu, hạ tầng gần như không đổi**: Layered đơn giản là đủ. Đừng vẽ hexagon cho một app quản lý ghi chú cá nhân.
- **Domain phức tạp** (nhiều business rule, nhiều invariant, giá trị nghiệp vụ cao — đúng kiểu DDD nhắm tới), **cần test domain logic hoàn toàn độc lập**, **cần đổi hạ tầng mà không đụng business logic**: Clean hoặc Hexagonal, không phân biệt cái nào hơn cái nào — chọn theo cái team đã quen thuật ngữ.
- **Hệ có nhiều kiểu client gọi vào** (HTTP + gRPC + CLI + consumer Kafka) **và nhiều kiểu hạ tầng gọi ra** (nhiều DB, nhiều message broker): Hexagonal có lợi thế vì vocabulary driving/driven làm rõ ngay hệ thống có bao nhiêu "cửa" mỗi loại — nhìn vào thư mục `adapters/driving/` và `adapters/driven/` là đếm được ngay.
- **Layered đã có sẵn kỷ luật Dependency Inversion (như Honeydue)**: không cần đổi tên hay tổ chức lại thư mục — bạn đã đứng đúng chỗ Clean/Hexagonal đứng, chỉ khác tên gọi.

## Kết

Câu hỏi "nên chọn Layered, Clean, hay Hexagonal" thường là câu hỏi sai. Câu hỏi đúng là: **domain của bạn có bị ép biết về infrastructure hay không?** Nếu có (domain import ORM, domain gọi thẳng HTTP client, business logic test không nổi vì thiếu database) — bất kể bạn đang gọi kiến trúc của mình là gì, bạn đang có vấn đề. Nếu không — domain hoàn toàn độc lập, chỉ giao tiếp qua interface do chính nó định nghĩa — bạn đã đúng luật, dù bạn gọi cái interface đó là "repository", "boundary", hay "port".

DDD cho bạn biết nên đặt gì vào giữa (Aggregate, Entity, Domain Service, Bounded Context). Layered/Clean/Hexagonal cho bạn biết cách vẽ và bảo vệ ranh giới quanh cái giữa đó. Hai việc khác nhau, nhưng làm tốt cả hai thì mới ra một hệ thống vừa đúng nghiệp vụ, vừa dễ bảo trì lâu dài.

## Đọc thêm

- Robert C. Martin — [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) (bài blog gốc, 2012)
- Alistair Cockburn — [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) (bài viết gốc, 2005)
- Jeffrey Palermo — [The Onion Architecture](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
- [bxcodec/go-clean-arch](https://github.com/bxcodec/go-clean-arch) — reference implementation Clean Architecture bằng Go được dùng nhiều nhất
- Series DDD trên blog này: [Phần 0]({{< ref "ddd-p0.md" >}}) · [Phần 1]({{< ref "ddd-p1.md" >}}) · [Phần 2]({{< ref "ddd-p2.md" >}})
