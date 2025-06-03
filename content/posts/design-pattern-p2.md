+++
date = '2025-06-03T15:30:43+07:00'
draft = false
title = 'Design Pattern P2 (Structural)'
author = 'Huy Dang Quang'
categories = ["Design Pattern"]
tags = ["golang", "coding", "design pattern"]
description = 'Go và một số Design Pattern phổ biến phần 2 (Structural)'
+++

# Structural Patterns

**Structural Patterns** liên quan đến cấu trúc và mối quan hệ giữa các lớp và đối tượng nhằm tạo ra cấu trúc phần mềm dễ thay đổi và bảo trì hơn.


## Facade

**Facade Pattern** cung cấp một giao diện đơn giản để tương tác với một hệ thống phức tạp, giúp che giấu sự phức tạp và các chi tiết kỹ thuật không cần thiết khỏi người dùng.

### Implementation

```go
type CPU struct{}

func (*CPU) Freeze() {
    fmt.Println("CPU Freeze")
}

func (*CPU) Jump(position int) {
    fmt.Printf("CPU Jump to %d\n", position)
}

func (*CPU) Execute() {
    fmt.Println("CPU Execute")
}

type Memory struct{}

func (*Memory) Load(position int, data string) {
    fmt.Printf("Memory Load data '%s' to position %d\n", data, position)
}

type HardDrive struct{}

func (*HardDrive) Read(position int, size int) string {
    data := fmt.Sprintf("HardDrive Read data from position %d with size %d", position, size)
    fmt.Println(data)
    return data
}

type ComputerFacade struct {
    cpu       *CPU
    memory    *Memory
    hardDrive *HardDrive
}

func NewComputerFacade() *ComputerFacade {
    return &ComputerFacade{
        cpu:       &CPU{},
        memory:    &Memory{},
        hardDrive: &HardDrive{},
    }
}

func (c *ComputerFacade) Start() {
    c.cpu.Freeze()
    c.memory.Load(0, "boot_loader")
    c.cpu.Jump(0)
    c.cpu.Execute()
}
```

### Usage

```go
func main() {
    computer := NewComputerFacade()
    computer.Start()
}
```

Trong ví dụ trên, **ComputerFacade** cung cấp một tập các thành phần con là **CPU**, **Memory**, **HardDrive**, và nó có 1 method là **Start** thực hiện một loạt các hành động để khởi động máy tính. Người dùng chỉ cần đơn giản gọi **Start** để khởi động máy tính mà không cần quan tâm đến sự phức tạp của các hệ thống con.


## Adapter

**Adapter Pattern** giúp kết nối giữa hai interface không tương thích sao cho chúng có thể làm việc cùng nhau mà không cần sửa đổi.

Adapter Pattern dựa trên một ý tưởng đơn giản nhưng mạnh mẽ: Thay vì thay đổi các đối tượng hoặc hệ thống để chúng tương thích với nhau, chúng ta sẽ tạo ra một **"bộ chuyển đổi"** (adapter) để làm cầu nối giữa chúng.

### Implementation

Giả sử một hệ thống e-commerce sử dụng một interface đơn giản:

```go
// Existing payment interface
type PaymentProcessor interface {
    ProcessPayment(amount float64) bool
}
```

Và có sẵn 1 implementation với **Paypal**

```go
// Existing payment service
type PayPalProcessor struct{}

func (p *PayPalProcessor) ProcessPayment(amount float64) bool {
    fmt.Println("Processing payment with PayPal:", amount)

    // Logic to process payment through PayPal
    return true // Assume payment is processed successfully
}
```

Bây giờ chúng ta muốn tương tác với hệ thống payment mới là **Stripe**, với 1 method khác (**ChargeCreditCard**)

```go
// New payment service we want to integrate
type StripeProcessor struct{}

func (s *StripeProcessor) ChargeCreditCard(name string, amount float64) bool {
    fmt.Println("Charging credit card through Stripe:", name, amount)
    // Logic to charge credit card through Stripe
    return true // Assume charge is successful
}
```

Rõ ràng **PayPalProcessor** và **StripeProcessor** không implement cùng một interface. Để tương tác được với **StripeProcessor** mà không làm thay đổi code hiện có, chúng ta sẽ tạo 1 adapter

```go
// Adapter for StripeProcessor to implement PaymentProcessor interface
type StripeAdapter struct {
    Stripe *StripeProcessor
}

func (sa *StripeAdapter) ProcessPayment(amount float64) bool {
    // Adapter translates the method call to the format expected by Stripe
    return sa.Stripe.ChargeCreditCard("Customer Name", amount)
}
```

### Usage

```go
func main() {
    // Existing system using PayPal
    payPal := &PayPalProcessor{}
    processPayment(payPal, 100.0)

    // New system using Stripe through the adapter
    stripe := &StripeAdapter{Stripe: &StripeProcessor{}}
    processPayment(stripe, 100.0)
}

func processPayment(p PaymentProcessor, amount float64) {
    success := p.ProcessPayment(amount)
    if success {
        fmt.Println("Payment processed successfully.")
    } else {
        fmt.Println("Payment processing failed.")
    }
}
```

Với adapter cho **Stripe**, hệ thống e-commerce có thể sử dụng phương thức **ChargeCreditCard** như một **PaymentProcessor**, cho phép tích hợp liền mạch dịch vụ thanh toán mới.

Adapter Pattern hữu ích khi bạn cần tích hợp một thư viện hoặc một hệ thống cũ mà không thể (hoặc không nên) thay đổi. Tuy nhiên vẫn có một vài lưu ý khi áp dụng:
- **Không lạm dụng**: không nên sử dụng nó chỉ vì muốn giữ lại tất cả mã nguồn cũ mà không có lý do chính đáng.
- **Hiệu năng có thể bị ảnh hưởng**: việc thêm một lớp trung gian giữa hai hệ thống có thể làm giảm hiệu năng.
- **Bảo trì**: khi thêm một lớp adapter, bạn thêm một điểm cần phải bảo trì. Nếu thư viện hoặc hệ thống cũ được cập nhật, adapter cũng cần phải được xem xét và cập nhật.
- **Kiểm tra kỹ lưỡng**: khi viết một adapter, đặc biệt quan trọng là thực hiện các bài kiểm tra kỹ lưỡng để đảm bảo rằng nó hoạt động chính xác và không gây ra các vấn đề tiềm ẩn.


## Composite

**Composite Pattern** được sử dụng để tổ chức các đối tượng vào một cấu trúc cây.

Mục đích chính của Composite Pattern là đơn giản hóa quá trình làm việc với các cấu trúc phức tạp bằng cách cho phép client tương tác với các đối tượng đơn lẻ và tổ hợp theo cùng một cách. Điều này giúp giảm thiểu sự phức tạp khi quản lý và tương tác với cấu trúc cây, làm cho mã nguồn dễ bảo trì và mở rộng hơn.

### Implementation

Khi sử Composite Pattern bạn phải chắc chắn rằng mô hình ứng dụng của bạn có thể biểu hiện bằng sơ đồ cây.

Ví dụ đơn giản nhất là việc lưu trữ trong máy tính có hai dạng chính: **Folder** và **File**. Giả sử ta cần tìm 1 file trong 1 folder, ta sẽ tạo 1 interface chung:

```go
type component interface {
	search(string)
}
```

Kiểu dữ liệu là **File**

```go
type file struct {
	name string
}

func (f *file) search(name string) {
	if f.name == name {
		println("Found file:", f.name)
	}
}

func (f *file) getName() string {
	return f.name
}
```

Kiểu dữ liệu **Folder** có thể chứa nhiều folder và file

```go
type folder struct {
	name       string
	components []component
}

func (f *folder) search(name string) {
	fmt.Printf("Searching in folder %s for file %s \n", f.name, name)
	for _, c := range f.components {
		c.search(name)
	}
}

func (f *folder) add(c component) {
	f.components = append(f.components, c)
}
```

### Usage

```go
func main() {
	root := &folder{name: "root"}
	folder1 := &folder{name: "folder1"}
	folder2 := &folder{name: "folder2"}

	file1 := &file{name: "file1.txt"}
	file2 := &file{name: "file2.txt"}
	file3 := &file{name: "file3.txt"}

	folder1.add(file1)
	folder1.add(file2)
	folder2.add(file3)

	root.add(folder1)
	root.add(folder2)

	root.search("file2.txt")
}

// Searching in folder root for file file2.txt
// Searching in folder folder1 for file file2.txt
// Found file: file2.txt
// Searching in folder folder2 for file file2.txt
```

Composite Pattern là mẫu thiết kế hữu ích để xây dựng và quản lý cấu trúc phân cấp dưới dạng cây của các đối tượng. Nó cho phép chúng ta làm việc với một nhóm đối tượng như một đối tượng đơn lẻ, mang lại khả năng tổ chức và quản lý phân cấp một cách linh hoạt và thuận tiện.

Tuy nhiên nếu không có nhu cầu xây dựng cấu trúc dạng cây hoặc quản lý các đối tượng dưới dạng phân cấp, không nên sử dụng do cấu trúc dạng cây có thể trở nên quá phức tạp và không cần thiết cho ứng dụng.


## Proxy

**Proxy Pattern** giúp điều chỉnh quyền truy cập, cung cấp các tính năng bổ sung như khởi tạo khi cần (lazy initialization), bảo mật, và ghi nhật ký mà không cần thay đổi đối tượng ban đầu.

**Ý Tưởng Cốt Lõi** là việc tạo ra một lớp trung gian, hay **"proxy"**, giúp quản lý truy cập một cách chặt chẽ đến đối tượng gốc.

Các thành phần:
- **Subject** interface: Đây là interface mà cả RealSubject và Proxy đều triển khai.
- **Real Subject**: Lớp thực sự thực hiện logic của phương thức. Đây là lớp mà Proxy sẽ đại diện hoặc "ủy quyền".
- **Proxy**: Lớp này duy trì một tham chiếu đến đối tượng **Real Subject** và cũng triển khai interface **Subject**. Nó có thể kiểm soát hoặc bổ sung hành vi trước hoặc sau khi chuyển yêu cầu đến **Real Subject**.
- **Client**: Lớp này sử dụng đối tượng **Subject**, không biết rằng nó thực sự đang tương tác với **Proxy** của **Real Subject**.

### Implementation

Định nghĩa 1 interface mà cả **Proxy** và **Real Object** cần implement

```go
type ITerminal interface {
	Execute(cmd string) (resp string, err error)
}
```

Định nghĩa **Real Object**

```go
// GopherTerminal is Real Object
type GopherTerminal struct {
	// user is a current authorized user
	User string
}


// Execute just runs known commands for current authorized user
func (gt *GopherTerminal) Execute(cmd string) (resp string, err error) {
	// Set "terminal" prefix for output
	prefix := fmt.Sprintf("%s@go_term$:", gt.User)

	// Execute some asked commands if we know them
	switch cmd {
	case "say_hi":
		resp = fmt.Sprintf("%s Hi %s", prefix, gt.User)
	case "man":
		resp = fmt.Sprintf("%s Visit 'https://golang.org/doc/' for Golang documentation", prefix)
	default:
		err = fmt.Errorf("%s Unknown command", prefix)
	}

	return
}
```

Tạo **Proxy** để cung cấp cho người dùng và lệnh cho các đối tượng cụ thể

```go
type Terminal struct {
	currentUser    string
	gopherTerminal *GopherTerminal
}


// Execute intercepts execution of command, implements authorizing user, validates it
// and poxing command to real terminal (gopherTerminal) method
func (t *Terminal) Execute(command string) (resp string, err error) {
	// If user allowed to execute send commands then, for example we can decide which terminal can be used, remote or local etc..
	// but for example we just creating new instance of terminal,
	// set current user and send user command to execution in terminal
	t.gopherTerminal = &GopherTerminal{User: t.currentUser}

	fmt.Printf("PROXY: Intercepted execution of user (%s), asked command (%s)\n", t.currentUser, command)

	// Transfer data to original object and execute command
	if resp, err = t.gopherTerminal.Execute(command); err != nil {
		err = fmt.Errorf("I know only how to execute commands: say_hi, man")
		return
	}

	return
}
```

### Usage

```go
func NewTerminal(user string) (t *Terminal, err error) {
	// Check user if given correctly
	if user == "" {
		err = fmt.Errorf("User cannot be empty")
		return
	}

	// Before we execute user commands, we validate current user
	if authErr := authorizeUser(user); authErr != nil {
		err = fmt.Errorf("You (%s) are not allowed to use terminal and execute commands", user)
		return
	}

	// Create new instance of terminal and set valid user
	t = &Terminal{currentUser: user}

	return
}

// authorize validates user right to execute commands
func authorizeUser(user string) (err error) {
	// As we use terminal like proxy, then
	// we will intercept user name to validate if it's allowed to execute commands
	if user != "gopher" {
		// Do some logs, notifications etc...
		err = fmt.Errorf("User %s in black list", user)
		return
	}

	return
}
```

**Client** sử dụng phuơng thức **Execute** của **Proxy** 

```go
func main() {
	t, err := NewTerminal("gopher")
	if err != nil {
		// panic: User cant be empty
		// Or
		// panic: Bad users are not allowed to use terminal and execute commands
		panic(err.Error())
	}

	// Execute user command
	excResp, excErr := t.Execute("say_hi")
	if excErr != nil {
		panic(excErr.Error())
	}

	// Show execution response
	fmt.Println(excResp)
}

// PROXY: Intercepted execution of user (gopher), asked command (say_hi)
// gopher@go_term$: Hi gopher
```

Khi nào nên sử dụng Proxy Pattern:
- **Kiểm Soát Truy cập**: Điều này thường thấy trong việc quản lý quyền truy cập đối với đối tượng nhạy cảm hoặc quan trọng.
- **Lazy Loading**: Đối với việc tải các đối tượng lớn hoặc tốn kém về tài nguyên, việc sử dụng Proxy Pattern giúp trì hoãn quá trình này cho đến khi thực sự cần thiết.
- **Tạo log**: Khi cần theo dõi hoặc ghi lại các hoạt động truy cập đối với một đối tượng
- **Chức năng Bổ sung hoặc Sửa đổi**: Khi muốn thêm hoặc sửa đổi chức năng của một đối tượng mà không làm thay đổi mã nguồn của đối tượng đó

Tuy nhiên Proxy Pattern không nên sử dụng khi không cần quản lý, kiểm soát hoặc bổ sung chức năng cho đối tượng, hoặc khi việc thêm một lớp trung gian làm tăng độ phức tạp không cần thiết cho ứng dụng. Trong những trường hợp này, việc sử dụng trực tiếp đối tượng gốc có thể là lựa chọn tốt hơn.


## Decorator

**Decorator Pattern** cho phép thêm các tính năng mới cho một đối tượng thông qua một lớp trang trí, mà không cần sửa đổi lớp đó.

**Ý Tưởng Cốt Lõi**: Bằng cách sử dụng thành phần (composition), Decorator Pattern thêm "vỏ bọc" cho đối tượng cơ bản, cung cấp hành vi thêm vào và có thể thay đổi tại runtime.

### Implementation

Định nghĩa đối tượng ban đầu với phương thức **getPrice**

```go
type IPizza interface {
	getPrice() int
}

type DefaultPizza struct{}

func (p *DefaultPizza) getPrice() int {
	return 15
}
```

Thêm các thành phần cho đối tượng gốc

```go
type TomatoTopping struct {
	pizza IPizza
}

func (c *TomatoTopping) getPrice() int {
	pizzaPrice := c.pizza.getPrice()
	return pizzaPrice + 7
}

type CheeseTopping struct {
	pizza IPizza
}

func (c *CheeseTopping) getPrice() int {
	pizzaPrice := c.pizza.getPrice()
	return pizzaPrice + 10
}
```

### Usage

```go
func main() {
	pizza := &DefaultPizza{}

	//Add cheese topping
	pizzaWithCheese := &CheeseTopping{
		pizza: pizza,
	}

	//Add tomato topping
	pizzaWithCheeseAndTomato := &TomatoTopping{
		pizza: pizzaWithCheese,
	}

	fmt.Printf("Price of default pizza with tomato and cheese topping is %d\n", pizzaWithCheeseAndTomato.getPrice())
}

// Price of default pizza with tomato and cheese topping is 32
```

**Decorator Pattern** là một công cụ mạnh mẽ trong việc mở rộng chức năng của các đối tượng mà không cần thay đổi lớp gốc, giúp tuân thủ nguyên tắc Open-Closed. Nó phù hợp nhất khi cần thêm tính năng vào đối tượng một cách linh hoạt, đặc biệt trong các hệ thống mà sự mở rộng liên tục là cần thiết.

Tuy nhiên, cần thận trọng để không làm dư thừa hoặc quá phức tạp hóa hệ thống bằng cách sử dụng quá nhiều decorators.