+++
date = '2025-06-03T21:34:51+07:00'
draft = false
title = 'Design Pattern P3 (Behavioral)'
author = 'Huy Dang Quang'
categories = ["Design Pattern"]
tags = ["golang", "coding", "design pattern"]
description = 'Go và một số Design Pattern phổ biến phần 3 (Behavioral)'
+++

# Behavioral Patterns

**Behavioral Patterns** liên quan đến cách giao tiếp và trao đổi thông tin giữa các đối tượng và lớp.


## Observer

**Observer Pattern** được sử dụng để tạo một cơ chế quản lý mối quan hệ một-nhiều giữa các đối tượng, sao cho khi trạng thái của một đối tượng thay đổi, tất cả đối tượng phụ thuộc (observers) sẽ được thông báo và cập nhật tự động.

**Tổng quan**: Observer Pattern bao gồm hai loại đối tượng chính: 'Subject' (đối tượng quan sát) và 'Observer' (đối tượng được thông báo). 'Subject' duy trì một danh sách các 'Observer' quan tâm tới trạng thái của nó và thông báo cho tất cả 'Observers' khi có bất kỳ thay đổi nào về trạng thái.

**Ý Tưởng Cốt Lõi**: Trong Observer Pattern, 'Subject' đóng vai trò là nhà xuất bản (publisher), trong khi 'Observers' đóng vai trò như các người đăng ký (subscribers). Mỗi khi 'Subject' có một thay đổi trạng thái, nó sẽ thông báo cho tất cả 'Observers' của mình.

### Implementation

Định nghĩa **Observers** (subscribers)

```go
// Event defines an indication of a point-in-time occurrence.
type Event struct {
	// Data in this case is a simple int, but the actual
	// implementation would depend on the application.
	Data int64
}

// Observer defines a standard interface for instances that wish to list for
// the occurrence of a specific event.
type Observer interface {
	// OnNotify allows an event to be "published" to interface implementations.
	// In the "real world", error handling would likely be implemented.
	OnNotify(Event)
}

type eventObserver struct {
	id int
}

func (o *eventObserver) OnNotify(e Event) {
	fmt.Printf("*** Observer %d received: %d\n", o.id, e.Data)
}
```

Định nghĩa **Subject** (publisher)

```go
// Notifier is the instance being observed. Publisher is perhaps another decent name
type Notifier interface {
	// Register allows an instance to register itself to listen/observe events.
	Register(Observer)
	// Deregister allows an instance to remove itself from the collection of observers/listeners.
	Deregister(Observer)
	// Notify publishes new events to listeners. The method is not
	// absolutely necessary, as each implementation could define this itself
	// without losing functionality.
	Notify(Event)
}

type eventNotifier struct {
	// Using a map with an empty struct allows us to keep the observers
	// unique while still keeping memory usage relatively low.
	observers map[Observer]struct{}
}

func (n *eventNotifier) Register(l Observer) {
	n.observers[l] = struct{}{}
}

func (n *eventNotifier) Deregister(l Observer) {
	delete(n.observers, l)
}

func (n *eventNotifier) Notify(e Event) {
	for o := range n.observers {
		o.OnNotify(e)
	}
}
```

### Usage

```go
func main() {
	// Initialize a new Notifier.
	n := eventNotifier{
		observers: map[Observer]struct{}{},
	}

	// Register a couple of observers.
	n.Register(&eventObserver{id: 1})
	n.Register(&eventObserver{id: 2})

	// A simple loop publishing the current Unix timestamp to observers.
	stop := time.NewTimer(10 * time.Second).C
	tick := time.NewTicker(time.Second).C
	for {
		select {
		case <-stop:
			return
		case t := <-tick:
			n.Notify(Event{Data: t.UnixNano()})
		}
	}
}
```

**Observer Pattern** phù hợp trong việc xây dựng các ứng dụng hướng sự kiện, các hệ thống linh hoạt có khả năng mở rộng cao.


## Iterator

**Iterator Pattern** cung cấp một cách để truy cập tuần tự các phần tử của một bộ sưu tập (collection) mà không cần phải hiểu rõ về cấu trúc nội bộ của nó. Điều này được thực hiện thông qua việc sử dụng một đối tượng gọi là **"Iterator"** có chức năng duyệt qua và truy cập các phần tử.

### Implementation

Định nghĩa **Iterator**

```go
type User struct {
	name string
	age  int
}

type Iterator interface {
	hasNext() bool
	getNext() *User
}

type UserIterator struct {
	index int
	users []*User
}

func (u *UserIterator) hasNext() bool {
	if u.index < len(u.users) {
		return true
	}
	return false
}

func (u *UserIterator) getNext() *User {
	if u.hasNext() {
		user := u.users[u.index]
		u.index++
		return user
	}
	return nil
}
```

Định nghĩa **Collection**

```go
type Collection interface {
    createIterator() Iterator
}

type UserCollection struct {
    users []*User
}

func (u *UserCollection) createIterator() Iterator {
    return &UserIterator{
        users: u.users,
    }
}
```

### Usage

```go
func main() {
	user1 := &User{
		name: "a",
		age:  30,
	}
	user2 := &User{
		name: "b",
		age:  20,
	}

	userCollection := &UserCollection{
		users: []*User{user1, user2},
	}

	iterator := userCollection.createIterator()

	for iterator.hasNext() {
		user := iterator.getNext()
		fmt.Printf("User is %+v\n", user)
	}
}

// User is &{name:a age:30}
// User is &{name:b age:20}
```

**Iterator** phù hợp để sử dụng trong các trường hợp:
- **Cấu Trúc Dữ Liệu Phức Tạp**: Iterator giúp duyệt qua các phần tử của một bộ sưu tập (collection) với cấu trúc phức tạp mà không cần phải lo lắng về cách chúng được tổ chức bên trong.
- **Bảo Mật Thông Tin Cấu Trúc**: Iterator cung cấp một giao diện đơn giản để tương tác với dữ liệu mà không hé lộ chi tiết phức tạp.
- **Giảm Thiểu Mã Lặp**: Thay vì viết các vòng lặp phức tạp, bạn có thể sử dụng các hàm của Iterator để làm việc này một cách gọn gàng và hiệu quả hơn.


## Strategy

**Strategy Pattern** cho phép định nghĩa một nhóm các thuật toán, đóng gói từng thuật toán lại, và làm cho chúng có thể hoán đổi cho nhau. Strategy cho phép thuật toán biến đổi độc lập với các khách hàng sử dụng nó. Điều này giúp tăng cường tính mô-đun và tái sử dụng của mã, bởi vì nó tách rời việc triển khai của các thuật toán từ các lớp sử dụng chúng.

**Mục Đích**: cung cấp một cách để cấu hình một lớp với một trong nhiều hành vi, hoặc thay đổi hành vi tại thời điểm chạy.

**Ý tưởng cốt lõi** của Strategy Pattern là **"đóng gói thuật toán"**. Bằng cách sử dụng các đối tượng **"Strategy"** để đại diện cho các thuật toán khác nhau và cho phép **"Context"** thay đổi chiến lược của mình, ứng dụng có thể thay đổi hành vi một cách linh hoạt mà không ảnh hưởng đến các lớp khách hàng. Điều này giúp mã nguồn trở nên linh hoạt hơn, dễ hiểu và dễ bảo trì.

### Implementation

Định nghĩa interface

```go
type Operator interface {
	Apply(int, int) int
}

type Operation struct {
	Operator Operator
}

func (o *Operation) Operate(leftValue, rightValue int) int {
	return o.Operator.Apply(leftValue, rightValue)
}
```

### Usage

- Addition Operator

```go
type Addition struct{}

func (Addition) Apply(lval, rval int) int {
	return lval + rval
}

func main() {
	add := Operation{Addition{}}
	fmt.Println(add.Operate(3, 5)) // 8
}
```

- Multiplication Operator

```go
type Multiplication struct{}

func (Multiplication) Apply(lval, rval int) int {
	return lval * rval
}

func main() {
	mult := Operation{Multiplication{}}
	fmt.Println(mult.Operate(3, 5)) // 15
}
```

Khi nào nên sử dụng **Strategy Pattern**:
- **Khi muốn thay đổi hành vi của một đối tượng tại thời gian chạy**: Việc sử dụng các chiến lược khác nhau giúp dễ dàng chuyển đổi giữa các thuật toán hoặc công việc mà đối tượng cần thực hiện.
- **Khi có nhiều biến thể của một hành vi**: Khi có nhiều phiên bản khác nhau của một hành vi cụ thể, sử dụng Strategy Pattern giúp tách rời chúng thành các lớp riêng biệt, mỗi lớp thực thi một phiên bản của hành vi. Điều này làm cho mã nguồn dễ quản lý và mở rộng hơn.
- **Khi muốn tránh việc sử dụng nhiều điều kiện**: Thay vì sử dụng câu lệnh điều kiện để chọn hành vi, Strategy Pattern cho phép loại bỏ cấu trúc điều kiện bằng cách đóng gói hành vi trong các lớp riêng biệt.