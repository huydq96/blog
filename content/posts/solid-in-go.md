+++
date = '2025-05-04T22:00:00+07:00'
draft = false
title = 'Solid in Go'
author = 'Huy Dang Quang'
categories = ["Design Pattern"]
tags = ["golang", "coding", "design pattern"]
description = 'Nguyên tắc SOLID với các ví dụ trong Go'
+++


# SOLID
Là tập hợp 5 nguyên lý thiết kế phần mềm giúp lập trình viên viết mã nguồn dễ bảo trì, mở rộng và ổn định hơn.

## **S** – Single Responsibility Principle (SRP)

> "A module should be responsible to one, and only one, actor." [(wikipedia)](https://en.wikipedia.org/wiki/Single-responsibility_principle)

- Một class nên có trách nhiệm cho một và chỉ một tác nhân. Hay nói cách khác, một class nên có một và chỉ một lý do thay đổi.

Ví dụ một đối tượng Trade phải chịu trách nhiệm xử lý dữ liệu và lưu trữ. Ta nên tách ra thành 2 type khác nhau (Repository và Validator, tương ứng với 2 nhiệm vụ). Bằng cách tách các mối quan tâm này, mỗi đối tượng có thể được kiểm tra và duy trì dễ dàng hơn.

```go
type Trade struct {
    TradeID int
    Symbol string
    Quantity float64
    Price float64
}

type TradeRepository struct {
    db *sql.DB
}

func (tr *TradeRepository) Save(trade *Trade) error {
    _, err := tr.db.Exec("INSERT INTO trades (trade_id, symbol, quantity, price) VALUES (?, ?, ?, ?)", trade.TradeID, trade.Symbol, trade.Quantity, trade.Price)
    if err != nil {
        return err
    }
    return nil
}

type TradeValidator struct {}

func (tv *TradeValidator) Validate(trade *Trade) error {
    if trade.Quantity <= 0 {
        return errors.New("Trade quantity must be greater than zero")
    }
    if trade.Price <= 0 {
        return errors.New("Trade price must be greater than zero")
    }
    return nil
}
```

### Đọc thêm
- [When to avoid DRY in Go](https://threedots.tech/post/things-to-know-about-dry/)
- [SRP - "S" trong SOLID](https://softwaredesign.vn/srp-s-trong-solid)


## **O** – Open-closed Principle

> "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification." [(wikipedia)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)

- Các entity nên mở để mở rộng nhưng đóng để sửa đổi. Có thể hiểu đơn giản rằng chúng ta cần hạn chế tối đa sửa code cũ, mà ưu tiên mở rộng, và “thay thế” nếu có thể.

Ví dụ ta có thể định nghĩa 1 interface TradeProcessor với method Process. Nếu một loại Processor mới được thêm vào, TradeProcessor có thể xử lý mà không cần thay đổi code. 

```go
type TradeProcessor interface {
    Process(trade *Trade) error
}

type FutureTradeProcessor struct {}

func (ftp *FutureTradeProcessor) Process(trade *Trade) error {
    // process future trade
    return nil
}

type OptionTradeProcessor struct {}

func (otp *OptionTradeProcessor) Process(trade *Trade) error {
    // process option trade
    return nil
}
```


## **L** – Liskov Substitution Principle (LSP)

> "An object (such as a class) may be replaced by a sub-object (such as a class that extends the first class) without breaking the program." [(wikipedia)](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

- Hiểu một cách đơn giản, class cha có hành vi gì, thì class con phải có đầy đủ có hành vi đó. Nghĩa là ta có thể thay thế class cha bằng class con mà hoàn toàn không phá vỡ ứng dụng của mình.

Ví dụ FutureTrade là một subtype của Trade. Điều đó có nghĩa là FutureTrade sẽ có thể được sử dụng thay cho Trade mà không gây ra bất kỳ vấn đề nào.

```go
type Trade interface {
    Process() error
}

type FutureTrade struct {
    Trade
}

func (ft *FutureTrade) Process() error {
    // process future trade
    return nil
}
```


## **I** – Interface Segregation Principle (ISP)

> "Clients should not be forced to depend upon interfaces that they do not use." [(wikipedia)](https://en.wikipedia.org/wiki/Interface_segregation_principle)

- Client chỉ nên phụ thuộc vào những gì mà nó cần. Khi triển khai các interface, đừng làm nó trở nên quá “rộng” và cover mọi trường hợp. Hãy thu hẹp nó làm sao để các class con không cần triển khai các hành vi mà nó không cần.

### Đọc thêm
- [Common Reuse Principle - CRP](https://softwaredesign.vn/common-reuse-principle-crp)


## **D** – Dependency Inversion Principle (DIP)

> "High-level modules should not import anything from low-level modules. Both should depend on abstractions (e.g., interfaces)."
>
> "Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions." [(wikipedia)](https://en.wikipedia.org/wiki/Dependency_inversion_principle)

- Module cấp cao không nên phụ thuộc vào bất cứ thứ gì trong các module cấp thấp. Cả hai nên phụ thuộc vào các trừu tượng.

- Trừu tượng không nên phụ thuộc vào chi tiết. Chi tiết nên phụ thuộc vào trừu tượng.

Ví dụ TradeProcessor phụ thuộc vào interface TradeService, thay vì thực hiện cụ thể SqlServerTradeRepository. Bằng cách này, các triển khai khác nhau của TradeService interface có thể được hoán đổi mà không làm ảnh hưởng đến TradeProcessor. Ví dụ có thể sử dụng MongoDbTradeRepository mà không cần sửa đổi TradeProcessor.

```go
type TradeService interface {
    Save(trade *Trade) error
}

type TradeProcessor struct {
    tradeService TradeService
}

func (tp *TradeProcessor) Process(trade *Trade) error {
    err := tp.tradeService.Save(trade)
    if err != nil {
        return err
    }
    // process trade
    return nil
}

type SqlServerTradeRepository struct {
    db *sql.DB
}

func (str *SqlServerTradeRepository) Save(trade *Trade) error {
    _, err := str.db.Exec("INSERT INTO trades (trade_id, symbol, quantity, price) VALUES (?, ?, ?, ?)", trade.TradeID, trade.Symbol, trade.Quantity, trade.Price)
    if err != nil {
        return err
    }
    return nil
}

type MongoDbTradeRepository struct {
    session *mgo.Session
}

func (mdtr *MongoDbTradeRepository) Save(trade *Trade) error {
    collection := mdtr.session.DB("trades").C("trade")
    err := collection.Insert(trade)
    if err != nil {
        return err
    }
    return nil
}
```

### Đọc thêm
- [DIP - Chữ "D" trong SOLID (part 1)](https://softwaredesign.vn/dependency-inversion-principle-dip)
- [DIP - Chữ "D" trong SOLID (part 2)](https://softwaredesign.vn/dip-chu-d-trong-solid-part-2)


Các ví dụ được lấy từ bài viết [SOLID Principles: Explained with Golang Examples](https://dev.to/ansu/solid-principles-explained-with-golang-examples-5eh).