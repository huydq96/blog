+++
date = '2025-06-03T10:08:25+07:00'
draft = false
title = 'Design Pattern P1 (Creational)'
author = 'Huy Dang Quang'
categories = ["Design Pattern"]
tags = ["golang", "coding", "design pattern"]
description = 'Go và một số Design Pattern phổ biến phần 1 (Creational)'
+++

# Mở đầu

[**Design Pattern**](https://en.wikipedia.org/wiki/Design_Patterns) là những mô hình thiết kế được sử dụng để giải quyết một vấn đề cụ thể trong lập trình. Nó cung cấp một giải pháp đã được kiểm nghiệm và tối ưu hóa, giúp các lập trình viên tiết kiệm thời gian và công sức khi phát triển phần mềm.

Để tìm hiểu chi tiết về 23 patterns, các bạn có thể tham khảo những đường dẫn sau:
- [Design Pattern Tiếng Việt](https://nguyenphuc22.github.io/Design-Patterns/intro.html)
- [Series Design Patterns](https://github.com/anonystick/anonystick?tab=readme-ov-file#-series-design-patterns)

Bài viết này sẽ chỉ tập trung vào một số pattern phổ biến và cách sử dụng trong Go.

# Creational Design Patterns

**Creational Design Patterns** liên quan tới việc khởi tạo đối tượng. Nhóm pattern này cung cấp các cơ chế tạo đối tượng một cách linh hoạt và phù hợp với bối cảnh sử dụng.


## Singleton

**Singleton** đảm bảo chỉ duy nhất một thể hiện của một lớp được tạo ra trong suốt chương trình.

### Implementation

```go
package singleton

type singleton map[string]string

var (
    once sync.Once

    instance singleton
)

func New() singleton {
	once.Do(func() {
		instance = make(singleton)
	})

	return instance
}
```

### Usage

```go
s := singleton.New()

s["this"] = "that"

s2 := singleton.New()

fmt.Println("This is ", s2["this"])
// This is that
```

Với cách triển khai này, chỉ có một đối tượng **singleton** duy nhất được tạo ra, và đối tượng này có thể được truy cập từ bất kỳ nơi nào trong chương trình. Việc này sẽ giúp tránh được các vấn đề như dữ liệu trùng lặp, xung đột tài nguyên, khó kiểm soát.


Tuy nhiên, cần lưu ý những điểm hạn chế của Singleton Pattern khi áp dụng:
- Nên sử dụng Singleton Pattern khi cần đảm bảo rằng chỉ có một thể hiện duy nhất của một lớp được tạo ra.
- Tránh sử dụng Singleton Pattern khi không cần thiết.
- Hạn chế sử dụng Singleton trong các hệ thống lớn hoặc phức tạp.

## Factory Method

**Factory Method** cung cấp một cách để tạo các đối tượng của lớp một cách linh hoạt mà không cần chỉ định rõ kiểu cụ thể của đối tượng nào sẽ được tạo.

**Mục đích**:
- **Dễ dàng tạo các đối tượng của lớp**: cho phép tạo các đối tượng của lớp một cách dễ dàng thông qua một phương thức trừu tượng.
- **Dễ dàng thay đổi cách tạo các đối tượng của lớp**: bằng cách triển khai lại phương thức factory ở các lớp con, bạn có thể thay đổi cách tạo đối tượng mà không ảnh hưởng đến client code.

### Implementation

Định nghĩa Interface

```go
package data

import "io"

type Store interface {
    Open(string) (io.ReadWriteCloser, error)
}
```

Các Implementations khác nhau

```go
package data

type StorageType int

const (
    DiskStorage StorageType = 1 << iota
    TempStorage
    MemoryStorage
)

func NewStore(t StorageType) Store {
    switch t {
    case MemoryStorage:
        return newMemoryStorage( /*...*/ )
    case DiskStorage:
        return newDiskStorage( /*...*/ )
    default:
        return newTempStorage( /*...*/ )
    }
}
```

### Usage

Chỉ định cụ thể kiểu storage mong muốn

```go
s, _ := data.NewStore(data.MemoryStorage)
f, _ := s.Open("file")

n, _ := f.Write([]byte("data"))
defer f.Close()
```

**Factory Pattern** là một pattern hữu ích giúp tăng tính linh hoạt và khả năng mở rộng cho hệ thống bằng cách tách biệt quá trình khởi tạo đối tượng.


## Abstract Factory

Như phần trên đã trình bày, [Factory Method](#factory-method) giúp tạo ra các đối tượng của một lớp. Tuy nhiên, trong nhiều trường hợp, các đối tượng có quan hệ với nhau và được nhóm thành các họ. Lúc này, chúng ta cần **Abstract Factory**.

**Abstract Factory** cung cấp một interface để tạo ra các họ đối tượng liên quan với nhau một cách linh hoạt. Nó bao gồm các thành phần chính sau:
- **Abstract Factory** interface: định nghĩa các phương thức để tạo ra các sản phẩm trừu tượng.
- **Concrete Factorie**: mỗi Concrete Factory tạo ra một tập sản phẩm khác biệt.
- **Abstract Product** interface: định nghĩa interface chung cho một loại sản phẩm trừu tượng.
- **Concrete Product**: implement Abstract Product. Mỗi sản phẩm thuộc về một Concrete Factory nhất định.
- **Client**: sử dụng Abstract Factory và Abstract Product để tương tác với hệ thống. Không cần quan tâm đến các lớp cụ thể.

### Implementation

Định nghĩa **Abstract Products**

```go
// SMSNotification is an interface for SMS notifications.
type SMSNotification interface {
    SendSMS() string
}

// EmailNotification is an interface for Email notifications.
type EmailNotification interface {
    SendEmail() string
}
```

Định nghĩa **Concrete Structs** (implement **Abstract Products**)

```go
// iOS-based notifications

// IOSSMS is a struct for iOS SMS notifications.
type IOSSMS struct{}

func (s IOSSMS) SendSMS() string {
    return "Sending iOS SMS Notification"
}

// IOSEmail is a struct for iOS Email notifications.
type IOSEmail struct{}

func (e IOSEmail) SendEmail() string {
    return "Sending iOS Email Notification"
}

// Android-based notifications

// AndroidSMS is a struct for Android SMS notifications.
type AndroidSMS struct{}

func (s AndroidSMS) SendSMS() string {
    return "Sending Android SMS Notification"
}

// AndroidEmail is a struct for Android Email notifications.
type AndroidEmail struct{}

func (e AndroidEmail) SendEmail() string {
    return "Sending Android Email Notification"
}
```
Định nghĩa **Abstract Factory Interface**

```go
// NotificationFactory is an abstract factory interface for creating notifications.
type NotificationFactory interface {
    CreateSMSNotification() SMSNotification
    CreateEmailNotification() EmailNotification
}
```

Định nghĩa **Concrete Factories** (implement **Abstract Factory Interface**)

```go
// IOSFactory is a concrete factory for iOS notifications.
type IOSFactory struct{}

func (f IOSFactory) CreateSMSNotification() SMSNotification {
    return IOSSMS{}
}

func (f IOSFactory) CreateEmailNotification() EmailNotification {
    return IOSEmail{}
}

// AndroidFactory is a concrete factory for Android notifications.
type AndroidFactory struct{}

func (f AndroidFactory) CreateSMSNotification() SMSNotification {
    return AndroidSMS{}
}

func (f AndroidFactory) CreateEmailNotification() EmailNotification {
    return AndroidEmail{}
}
```

### Usage

Tạo **Factory Selector**

```go
// GetNotificationFactory returns a NotificationFactory based on the input type.
func GetNotificationFactory(platform string) NotificationFactory {
    switch platform {
    case "iOS":
        return IOSFactory{}
    case "Android":
        return AndroidFactory{}
    default:
        return nil
    }
}
```

Sử dụng **Abstract Factory** ở Client

```go
func main() {
    // Get an iOS factory
    factory := GetNotificationFactory("iOS")
    if factory != nil {
        sms := factory.CreateSMSNotification()
        email := factory.CreateEmailNotification()
        fmt.Println(sms.SendSMS())     // Output: Sending iOS SMS Notification
        fmt.Println(email.SendEmail()) // Output: Sending iOS Email Notification
    }

    // Get an Android factory
    factory = GetNotificationFactory("Android")
    if factory != nil {
        sms := factory.CreateSMSNotification()
        email := factory.CreateEmailNotification()
        fmt.Println(sms.SendSMS())     // Output: Sending Android SMS Notification
        fmt.Println(email.SendEmail()) // Output: Sending Android Email Notification
    }
}
```

**Abstract Factory** là một Design Pattern hữu ích, có một số ưu điểm sau:
- Tách biệt phần triển khai với phần sử dụng code, giảm sự phụ thuộc lẫn nhau giữa các đối tượng.
- Có thể dễ dàng thay đổi, mở rộng cách tạo đối tượng mà không ảnh hưởng đến phần còn lại của code.
- Giúp tạo ra các họ đối tượng liên quan một cách thống nhất.

Tuy nhiên cũng có nhược điểm cần lưu ý là cấu trúc phức tạp, nhiều lớp trừu tượng cần phải triển khai. Vì vậy, Abstract Factory phù hợp trong trường hợp cần tạo ra các họ đối tượng liên quan, có tính mở rộng cao. Không nên sử dụng nếu chỉ cần tạo đơn giản một Object.


## Prototype

**Prototype** cho phép sao chép các đối tượng hiện có thay vì khởi tạo chúng từ đầu.

Mục đích của Pattern này là tạo ra các đối tượng mới bằng cách **clone** từ đối tượng hiện có thay vì khởi tạo, tiết kiệm chi phí tạo mới đối tượng, đặc biệt là các đối tượng phức tạp. Ngoài ra, nó che giấu logic khởi tạo và cung cấp khả năng tạo các đối tượng tương tự một cách hiệu quả.

### Implementation

```go
type Cloneable interface {
    Clone() Cloneable
}

type Person struct {
    Name string
    Age  int
}

func (p *Person) Clone() Cloneable {
    return &Person{
        Name: p.Name,
        Age:  p.Age,
    }
}
```

### Usage

```go
func main() {
    person1 := &Person{Name: "Alice", Age: 30}
    person2 := person1.Clone().(*Person)

    // Modify the clone
    person2.Name = "Jane"
    person2.Age = 25

    fmt.Println(person1)
    fmt.Println(person2)
}
```


## So sánh nhanh

- [Singleton](#singleton) phù hợp khi cần đảm bảo tính duy nhất và quản lý tài nguyên chung.
- [Factory Method](#factory-method) là lựa chọn tốt khi cần tính linh hoạt trong việc tạo đối tượng và khả năng mở rộng trong tương lai.
- [Abstract Factory](#abstract-factory) được ưu tiên khi làm việc với các họ sản phẩm phức tạp và cần đảm bảo tính tương thích.
- [Prototype](#prototype) là giải pháp hiệu quả khi việc tạo đối tượng mới tốn kém hoặc khi cần sao chép các đối tượng phức tạp.