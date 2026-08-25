+++
date = '2026-08-19T09:30:00+07:00'
draft = false
slug = 'ddd-p2'
title = 'DDD Building Blocks: Value Object, Entity, Aggregate... Qua Code Thật'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "ddd", "fintech", "architecture", "honeydue"]
description = 'Value Object, Entity, Aggregate, Repository, Domain Service, Domain Event, Application Service — giải thích từng khái niệm DDD bằng code thật trong Honeydue'
+++

Bài trước ([DDD Là Gì?](/posts/ddd-p1/)) nói về bức tranh lớn: 4 tầng, bounded context. Bài này đi vào chi tiết — từng "khối xây dựng" (building block) bên trong tầng Domain, toàn bộ ví dụ lấy từ `internal/domain/transaction` và `internal/application/transaction` — context đã implement đầy đủ nhất trong Honeydue hiện tại.

Thứ tự đi từ nhỏ đến lớn: Value Object → Entity → Aggregate → Repository → Domain Service (khác Application Service ở đâu) → Domain Event.

## Value Object — không có identity, chỉ có giá trị

Value Object (VO) là một khái niệm mô tả **một giá trị**, không phải một "thứ" có danh tính riêng. Hai VO có cùng giá trị thì coi như **là một**, không cần so ID. Ba đặc điểm bắt buộc:

1. **Immutable** — không có setter, mọi "thay đổi" thực ra là tạo VO mới.
2. **Tự validate trong constructor** — không có cách nào tạo ra một VO ở trạng thái không hợp lệ.
3. **So sánh bằng giá trị** — `Equals()`, không so con trỏ.

`Money` trong `internal/domain/shared/value_objects.go` là ví dụ kinh điển nhất — và quan trọng nhất trong 1 app fintech:

```go
type Money struct {
    amount   int64  // luôn là minor unit — cent, hào — KHÔNG BAO GIỜ float64
    currency string
}

func NewMoney(amount int64, currency string) (Money, error) {
    if amount < 0 {
        return Money{}, NewDomainErrorWithDetails(ErrCodeInvalidAmount, "...", ...)
    }
    if !validCurrencies[currency] {
        return Money{}, NewDomainErrorWithDetails(ErrCodeInvalidCurrency, "...", ...)
    }
    return Money{amount: amount, currency: currency}, nil
}

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, NewDomainErrorWithDetails(ErrCodeCurrencyMismatch, "...", ...)
    }
    // ...
}
```

Vì sao dùng `int64` thay vì `float64`? Số thực dấu phẩy động không biểu diễn chính xác được số thập phân — `0.1 + 0.2` trong hầu hết ngôn ngữ (kể cả Go) không ra đúng `0.3`. Với tiền, sai số dù nhỏ cũng không chấp nhận được. Honeydue lưu **minor unit** — cent với USD, hào với VND — dưới dạng số nguyên, quy đổi hiển thị chỉ diễn ra ở tầng presentation.

Để ý `Add` cũng không cho cộng khác currency — nghiệp vụ "không được cộng 20 USD với 500.000 VND" nằm ngay trong value object, không thể vi phạm dù gọi từ đâu.

`PrivacyLevel` (trong `internal/domain/transaction/value_objects.go`) là VO thứ hai đáng chú ý — validate ngay khi tạo, và constructor còn tự chuẩn hoá input:

```go
func NewPrivacyLevel(level string) (PrivacyLevel, error) {
    normalized := strings.ToLower(strings.TrimSpace(level))
    switch normalized {
    case shared.PrivacyHidden, shared.PrivacyBalanceOnly, shared.PrivacyFull:
        return PrivacyLevel{value: normalized}, nil
    default:
        return PrivacyLevel{}, shared.NewDomainErrorWithDetails(
            shared.ErrCodeInvalidPrivacy,
            "privacy level must be one of: hidden, balance_only, full",
            map[string]string{"privacy_level": level},
        )
    }
}
```

`"Full"` và `" full "` đều được chấp nhận và chuẩn hoá về `"full"`. Nhờ vậy, mọi nơi khác trong code cầm 1 giá trị `PrivacyLevel` đều chắc chắn nó hợp lệ — không cần validate lại lần hai.

## Entity — có identity, field thay đổi được nhưng "vẫn là nó"

Khác VO, **Entity** có một ID phân biệt nó với mọi entity khác, kể cả khi các field khác giống hệt nhau. `Transaction` (trong `entity.go`) là entity — nhưng chú ý: **không có setter nào cả**:

```go
type Transaction struct {
    id          string
    coupleID    string
    userID      string
    description string
    bookedDate  time.Time
    createdAt   time.Time
    updatedAt   time.Time
}

func (t *Transaction) GetID() string          { return t.id }
func (t *Transaction) GetCoupleID() string    { return t.coupleID }
// ... chỉ toàn Get, không có Set
```

Field đều **unexported** (chữ thường). Đây là cơ chế Go dùng để ép buộc quy tắc "chỉ sửa qua aggregate root": code ngoài package `transaction` không thể viết `txn.description = "..."` — Go sẽ báo lỗi compile, không phải lỗi runtime hay convention "đừng làm vậy". Chỉ code cùng package (tức là `TransactionAggregate`, trong `aggregate.go`) mới gán trực tiếp được vào field này.

## Aggregate — biên giới nhất quán, chỉ 1 cửa vào

Đây là khái niệm hay bị hiểu sai nhất. **Aggregate** không phải "một class to chứa nhiều class con" — nó là **ranh giới giao dịch (consistency boundary)**: mọi thay đổi bên trong ranh giới đó phải xảy ra atomically, và **chỉ có đúng 1 điểm vào** — aggregate root.

```go
type TransactionAggregate struct {
    transaction  Transaction
    money        shared.Money
    privacy      PrivacyLevel
    category     shared.Category
    domainEvents []any
}
```

<div style="max-width:680px;margin:2rem auto;">
<svg viewBox="0 0 640 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p2a-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p2a-arrow-green" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#16A34A"/>
</marker>
<marker id="p2a-arrow-red" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#E11D48"/>
</marker>
</defs>
<rect x="0" y="0" width="640" height="480" fill="#FFFDF7" rx="16"/>
<rect x="30" y="60" width="560" height="380" rx="24" fill="#FFF5F6" stroke="#FF6B6B" stroke-width="2.5" stroke-dasharray="10 6"/>
<text x="55" y="90" font-size="15" font-weight="800" fill="#9F1239">TransactionAggregate</text>
<text x="55" y="108" font-size="11.5" fill="#BE123C">Consistency Boundary — chỉ có 1 cửa vào</text>
<rect x="60" y="130" width="250" height="90" rx="12" fill="#EEF2FF" stroke="#4F46E5" stroke-width="2" filter="url(#p2a-shadow)"/>
<text x="185" y="155" text-anchor="middle" font-size="14" font-weight="700" fill="#3730A3">Transaction (Entity)</text>
<text x="185" y="175" text-anchor="middle" font-size="10.5" fill="#4338CA">id, coupleID, userID,</text>
<text x="185" y="190" text-anchor="middle" font-size="10.5" fill="#4338CA">description, bookedDate</text>
<text x="185" y="206" text-anchor="middle" font-size="10.5" font-weight="700" fill="#4F46E5">→ CÓ identity</text>
<rect x="330" y="130" width="230" height="50" rx="25" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p2a-shadow)"/>
<text x="445" y="160" text-anchor="middle" font-size="12.5" font-weight="700" fill="#92400E">Money{amount, currency}</text>
<rect x="330" y="190" width="230" height="50" rx="25" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p2a-shadow)"/>
<text x="445" y="220" text-anchor="middle" font-size="12.5" font-weight="700" fill="#92400E">PrivacyLevel{value}</text>
<rect x="330" y="250" width="230" height="50" rx="25" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p2a-shadow)"/>
<text x="445" y="280" text-anchor="middle" font-size="12.5" font-weight="700" fill="#92400E">Category{name}</text>
<text x="445" y="318" text-anchor="middle" font-size="10.5" fill="#92400E">Value Object — immutable, không id</text>
<rect x="60" y="330" width="500" height="70" rx="12" fill="#F3F4F6" stroke="#6B7280" stroke-width="2"/>
<text x="75" y="352" font-size="12.5" font-weight="700" fill="#374151">domainEvents []any</text>
<text x="75" y="372" font-size="10.5" fill="#4B5563">TransactionCreatedEvent</text>
<text x="75" y="388" font-size="10.5" fill="#4B5563">TransactionPrivacyChangedEvent</text>
<line x1="150" y1="15" x2="150" y2="58" stroke="#16A34A" stroke-width="2.5" marker-end="url(#p2a-arrow-green)"/>
<text x="160" y="28" font-size="10.5" fill="#16A34A" font-weight="700">agg.UpdatePrivacy(ctx, newLevel)</text>
<text x="160" y="42" font-size="10.5" fill="#16A34A">✓ duy nhất cách hợp lệ</text>
<line x1="615" y1="20" x2="555" y2="155" stroke="#E11D48" stroke-width="2.5" stroke-dasharray="6 5" marker-end="url(#p2a-arrow-red)"/>
<circle cx="588" cy="85" r="13" fill="#ffffff" stroke="#E11D48" stroke-width="2"/>
<text x="588" y="90" text-anchor="middle" font-size="13" font-weight="800" fill="#E11D48">✕</text>
<text x="460" y="112" text-anchor="middle" font-size="10.5" fill="#E11D48">txn.money = ... ❌</text>
<text x="460" y="126" text-anchor="middle" font-size="10.5" fill="#E11D48">field unexported, ngoài package</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 3: Bên trong TransactionAggregate — mọi thứ trong boundary chỉ đổi được qua aggregate root.</p>

Cách duy nhất để đổi privacy của 1 transaction là gọi `agg.UpdatePrivacy(...)`, không phải sửa thẳng field:

```go
func (a *TransactionAggregate) UpdatePrivacy(ctx context.Context, newLevel PrivacyLevel) error {
    if newLevel.Equals(a.privacy) {
        return nil // không đổi gì thì không publish event — tránh làm phiền Notification vô ích
    }
    oldLevel := a.privacy
    a.privacy = newLevel
    a.transaction.updatedAt = time.Now()

    a.publishEvent(TransactionPrivacyChangedEvent{
        TransactionID: a.transaction.id,
        OldPrivacy:    oldLevel.String(),
        NewPrivacy:    newLevel.String(),
        ChangedAt:     time.Now(),
    })
    return nil
}
```

Có một chi tiết dễ bị bỏ qua: bên cạnh `NewTransactionAggregate` (constructor cho transaction **mới**, publish `TransactionCreatedEvent`), Honeydue còn có `ReconstructTransactionAggregate` — dùng riêng cho repository khi **load lại** transaction đã tồn tại từ DB. Load lại không phải là 1 sự kiện nghiệp vụ, nên hàm này giữ nguyên `id`/`createdAt` cũ và **không** publish event nào — nếu lỡ dùng nhầm `NewTransactionAggregate` để load, mỗi lần đọc 1 transaction cũ sẽ vô tình tạo ID mới và bắn ra event "vừa tạo transaction" — sai hoàn toàn.

## Repository — interface nằm trong domain, implementation nằm ngoài

```go
// internal/domain/transaction/repository.go
type Repository interface {
    Save(ctx context.Context, agg *TransactionAggregate) error
    GetByID(ctx context.Context, id string) (*TransactionAggregate, error)
    GetByCoupleID(ctx context.Context, coupleID string) ([]*TransactionAggregate, error)
    Delete(ctx context.Context, id string) error
}
```

Chú ý tên hàm: `Save`, `GetByID` — ngôn ngữ domain (giống thao tác trên 1 collection trong bộ nhớ), không phải `INSERT`, `SELECT WHERE`. Interface này khai báo ở domain layer; implementation thật (`MysqlTransactionRepository`, dùng `? placeholder`, `ON DUPLICATE KEY UPDATE`) nằm ở `internal/infrastructure/persistence` — đây chính là mũi tên "implements" màu teal ở Hình 1 bài trước. Domain chỉ cần biết "có thể Save và GetByID", không quan tâm dưới đó là MySQL hay bất cứ thứ gì khác.

## Domain Service khác Application Service ở đâu?

Đây là chỗ dễ nhầm nhất khi mới học DDD. Cả hai đều là "service", nhưng vai trò hoàn toàn khác nhau — bảng so sánh dựa trên 2 ví dụ thật trong Honeydue:

| | Domain Service | Application Service |
|---|---|---|
| Ví dụ | `PrivacyService`, `BudgetService` | `CreateTransactionApplicationService` |
| Chứa gì | Business logic thật — quy tắc nghiệp vụ | Không business logic — chỉ điều phối (orchestration) |
| Khi nào cần | Logic không thuộc về 1 aggregate/VO nào (cần dữ liệu từ nhiều nguồn) | Luôn cần — là "nhạc trưởng" gọi domain theo đúng thứ tự |
| Biết gì về hạ tầng | Được phép đọc qua repository (query-only) | Biết cả repository, cả event publisher |
| Có setter/thay đổi state của DB? | Không — chỉ đọc, hoặc coordinate save qua aggregate | Không — chỉ gọi `repo.Save`, không tự viết SQL |

`PrivacyService` là domain service "thuần" — hoàn toàn không đụng repository, chỉ là 1 hàm logic:

```go
func (s *PrivacyService) GetVisibleFields(viewer, txnOwner string, privacy PrivacyLevel) (VisibleFields, error) {
    if viewer == txnOwner {
        return VisibleFields{Amount: true, Category: true, Description: true}, nil
    }
    switch {
    case privacy.IsFull():
        return VisibleFields{Amount: true, Category: true, Description: true}, nil
    case privacy.IsBalanceOnly():
        return VisibleFields{Amount: true}, nil
    default: // IsHidden hoặc giá trị lạ — fail closed, không lộ gì cả
        return VisibleFields{}, nil
    }
}
```

`BudgetService` là domain service "lai" — vẫn thuần business logic, nhưng cần đọc từ **2 repository khác nhau** (`budget.Repository` và `transaction.Repository`) để tính được. Đây chính là lý do nó phải là domain service chứ không thể là method của riêng `BudgetAggregate` hay `TransactionAggregate` — logic "kiểm tra ngưỡng" thuộc về *quan hệ giữa 2 aggregate*, không thuộc về aggregate nào một mình:

```go
func (s *BudgetService) CheckThreshold(ctx context.Context, coupleID string, category shared.Category, newAmount shared.Money) (*BudgetThresholdReachedEvent, error) {
    budgetAgg, err := s.budgetRepo.GetByCoupleIDAndCategory(ctx, coupleID, category)
    // ... không có budget cho category này -> không phải lỗi, chỉ là (nil, nil)

    transactions, _ := s.transactionRepo.GetByCoupleID(ctx, coupleID)
    // cộng dồn spend trong kỳ hiện tại, so với limit ở 3 mốc 50/80/100%
    // ...
}
```

**Lưu ý về trạng thái hiện tại**: `CheckThreshold` tính bằng cách load toàn bộ transaction của couple rồi `SUM` trong Go — chưa dùng Redis counter. Đây là cách đúng và đơn giản cho MVP, nhưng sẽ đổi khi có Kafka event-driven ở Phase 2.

`CreateTransactionApplicationService` thì hoàn toàn khác — nó không hề tự tính privacy hay tự tính threshold, chỉ **gọi đúng thứ tự** rồi giao lại cho domain:

```go
func (s *CreateTransactionApplicationService) Execute(ctx context.Context, cmd CreateTransactionCommand) (CreateTransactionResponse, error) {
    agg, err := transaction.NewTransactionAggregate(cmd.CoupleID, cmd.UserID, cmd.Amount, ...)
    if err != nil {
        return CreateTransactionResponse{}, err
    }

    newAmount, _ := shared.NewMoney(agg.GetAmount(), agg.GetCurrency())
    thresholdEvent, _ := s.budgetService.CheckThreshold(ctx, agg.GetCoupleID(), agg.GetCategory(), newAmount)

    if err := s.repo.Save(ctx, agg); err != nil {
        return CreateTransactionResponse{}, fmt.Errorf("save transaction: %w", err)
    }

    events := agg.GetDomainEvents()
    if thresholdEvent != nil {
        events = append(events, *thresholdEvent)
    }
    for _, event := range events {
        s.eventPublisher.Publish(ctx, event)
    }
    agg.ClearDomainEvents()

    return CreateTransactionResponse{TransactionID: agg.GetID(), ...}, nil
}
```

## Domain Event — publish sau khi lưu, không phải trước

`TransactionAggregate` không tự đẩy event lên Kafka — nó chỉ gom event vào 1 slice nội bộ (`domainEvents`) khi có thay đổi:

```go
func (a *TransactionAggregate) publishEvent(event any) {
    a.domainEvents = append(a.domainEvents, event)
}
```

Application service mới là nơi thật sự publish, và **chỉ publish sau khi `repo.Save` thành công** — nhìn lại đoạn code `Execute` ở trên: `Save` luôn chạy trước, publish luôn chạy sau. Đây không phải ngẫu nhiên. Nếu publish trước khi save (hoặc publish dù save thất bại), một consumer khác (ví dụ Budget worker sau này) có thể xử lý 1 transaction... không hề tồn tại trong DB. Comment ngay trong code cũng thẳng thắn nhìn nhận giới hạn còn lại của cách làm này:

> *"the transaction is already persisted at this point. A publish failure here surfaces as an error to the caller but does not roll back the save — closing that dual-write gap would need an outbox pattern, which is out of scope for this service."*

Nói cách khác: đã tránh được lỗi "publish event cho thứ chưa lưu", nhưng chưa xử lý trường hợp lưu xong mà publish lỗi (dual-write problem kinh điển). Outbox pattern là hướng giải quyết chuẩn — chưa nằm trong scope hiện tại, ghi nhận làm việc cho sau.

## Ghép lại: 1 request đi qua hết các building block

<div style="max-width:640px;margin:2rem auto;">
<svg viewBox="0 0 640 640" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="p2b-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="p2b-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
</defs>
<rect x="0" y="0" width="640" height="640" fill="#FFFDF7" rx="16"/>
<rect x="80" y="20" width="480" height="60" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="45" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">POST /transactions → Handler.CreateTransaction</text>
<text x="320" y="64" text-anchor="middle" font-size="10.5" fill="#0F766E">Presentation</text>
<line x1="320" y1="80" x2="320" y2="108" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="110" width="480" height="60" rx="14" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="135" text-anchor="middle" font-size="13" font-weight="700" fill="#9F1239">① NewTransactionAggregate(...) — validate &amp; build</text>
<text x="320" y="154" text-anchor="middle" font-size="10.5" fill="#BE123C">Domain</text>
<line x1="320" y1="170" x2="320" y2="198" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="200" width="480" height="60" rx="14" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="225" text-anchor="middle" font-size="13" font-weight="700" fill="#9F1239">② budgetService.CheckThreshold(...) — advisory</text>
<text x="320" y="244" text-anchor="middle" font-size="10.5" fill="#BE123C">Domain</text>
<line x1="320" y1="260" x2="320" y2="288" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="290" width="480" height="60" rx="14" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="315" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">③ repo.Save(ctx, agg) — persist trước</text>
<text x="320" y="334" text-anchor="middle" font-size="10.5" fill="#0F766E">Infrastructure</text>
<line x1="320" y1="350" x2="320" y2="378" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="380" width="480" height="60" rx="14" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="405" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">④ eventPublisher.Publish(...) — publish sau khi lưu</text>
<text x="320" y="424" text-anchor="middle" font-size="10.5" fill="#0F766E">Infrastructure</text>
<line x1="320" y1="440" x2="320" y2="468" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="470" width="480" height="60" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="495" text-anchor="middle" font-size="13" font-weight="700" fill="#7C4A03">⑤ trả CreateTransactionResponse{...}</text>
<text x="320" y="514" text-anchor="middle" font-size="10.5" fill="#92400E">Application</text>
<line x1="320" y1="530" x2="320" y2="558" stroke="#57534E" stroke-width="2" marker-end="url(#p2b-arrow)"/>
<rect x="80" y="560" width="480" height="60" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#p2b-shadow)"/>
<text x="320" y="585" text-anchor="middle" font-size="13" font-weight="700" fill="#134E4A">HTTP 200 OK — SuccessResponse{...}</text>
<text x="320" y="604" text-anchor="middle" font-size="10.5" fill="#0F766E">Presentation</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Hình 4: Một request POST /transactions đi qua đủ 4 tầng — đúng theo 5 bước trong Execute().</p>

Mỗi khối ở Hình 4 map thẳng 1-1 với 1 khái niệm vừa học: bước ① là Aggregate factory, bước ② là Domain Service, bước ③–④ là Repository + Event publish (qua Infrastructure), bước ⑤ là Application Service gói kết quả thành DTO. Không có ô nào chứa "if privacy == ..." hay raw SQL nằm sai chỗ.

> **Trạng thái hiện tại & sẽ mở rộng**: bài này dừng ở context Transaction vì nó minh hoạ đủ mọi building block. Budget context và Couple context dùng đúng các pattern giống hệt (đã có domain layer, repository, application service, HTTP handler — xem lại Hình 2 ở bài 1). Những phần chưa có, sẽ quay lại trong các bài sau của series Honeydue: Kafka consumer thật (Categorizer, Budget worker, Notification), Redis counter thay cho `SUM` on-read, WebSocket realtime, mock bank provider, và tách gRPC service ở Phase 4.

## Đọc thêm

- Vaughn Vernon — *Implementing Domain-Driven Design*, chương 5 (Entities) và 10 (Aggregates)
- [martinfowler.com/bliki/DDD_Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [martinfowler.com/bliki/ValueObject](https://martinfowler.com/bliki/ValueObject.html)
- [martinfowler.com/eaaCatalog/repository.html](https://martinfowler.com/eaaCatalog/repository.html)
