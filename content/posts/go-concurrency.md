+++
date = '2025-05-18T13:39:54+07:00'
draft = false
title = 'Go Concurrency'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "coding", "concurrency"]
description = 'Một số khái niệm về concurrency trong Go'
+++

# Tổng quan

**Concurrency** là một trong những tính năng nổi bật nhất của Golang, được thiết kế để xây dựng các hệ thống hiệu quả, có khả năng mở rộng cao. Khác với mô hình đa luồng truyền thống, Concurrency trong Go được xây dựng dựa trên ba trụ cột: **goroutine** (đơn vị xử lý), **channel** (giao tiếp an toàn), và **runtime scheduler** (tối ưu tài nguyên).

# Goroutine

`Goroutine` là một trong những tính năng đặc biệt và mạnh mẽ nhất của ngôn ngữ lập trình Go, cho phép lập trình đồng thời (concurrency) được thực hiện một cách đơn giản và hiệu quả.

Về bản chất, Goroutine là các hàm hoặc phương thức được thực thi một cách độc lập và đồng thời nhưng vẫn có thể kết nối với nhau. Đây là một **lightweight thread of execution** được quản lý bởi **Go runtime** và cho phép viết mã bất đồng bộ (asynchronous) theo cách đồng bộ (synchronous).

## Cơ chế hoạt động

Cơ chế của Goroutine khá đơn giản: một function tồn tại một cách đa luồng với các Goroutine khác trên cùng một không gian bộ nhớ. Go có bộ điều khiển quản lý các Goroutine rồi phân phối chúng vào các bộ xử lý logic và gắn mỗi bộ xử lý logic này với một thread hệ thống được tạo ra trước đó để thực thi các Goroutine này. Nói cách khác, mỗi **thread hệ thống sẽ xử lý một nhóm Goroutine được điều phối thông qua bộ xử lý logic**.

Go runtime sử dụng mô hình M:N scheduler, có nghĩa là nó **multiplexes M goroutines onto N OS threads**. Nhiệm vụ chính của Go scheduler là phân phối các Goroutine trên các thread và core có sẵn một cách hiệu quả. Với bộ điều khiển quản lý tác vụ đồng thời và cơ chế bộ xử lý logic, những khó khăn, phức tạp khi khai báo thread Go đã xử lý hết giúp lập trình viên.

## Cơ chế giao tiếp và đồng bộ

Về mặt giao tiếp, Goroutine có thể giao tiếp an toàn với nhau thông qua các `Channel`. Các channel hỗ trợ **mutex lock** vì thế tránh được các lỗi liên quan tới cùng ghi và đọc lên vùng dữ liệu chia sẻ (**data race**). Goroutine có thể được ánh xạ và hoạt động trên nhiều OS threads thay vì ánh xạ 1:1 như Thread truyền thống.

Go áp dụng triết lý **"Do not communicate by sharing memory; instead, share memory by communicating"** (Đừng giao tiếp bằng cách chia sẻ bộ nhớ; thay vào đó, hãy chia sẻ bộ nhớ bằng cách giao tiếp).

## Các vấn đề thường gặp 

### Deadlock

**Deadlock** là một vấn đề phổ biến khi sử dụng Goroutine. Deadlock xảy ra khi một nhóm Goroutine đang đợi nhau và không ai trong số đó có thể tiến hành. Các nguyên nhân phổ biến gây deadlock bao gồm:

- Quên gọi `wg.Done()` trong `WaitGroup`
- Sử dụng số lượng `wg.Add()` không chính xác
- Gửi dữ liệu vào channel mà không có Goroutine nào nhận

### CPU và Memory Optimization

Go mặc định sử dụng **1 CPU core**. Để sử dụng nhiều CPU cho xử lý song song (**parallel processing**), cần gọi `runtime.GOMAXPROCS(CPU_count)`. GOMAXPROCS chỉ định số lượng tối đa processors có thể được sử dụng bởi Go runtime.

Từ Go 1.5, **GOMAXPROCS** có default là **"số lượng logical CPUs có sẵn"**. Điều này hoạt động tốt cho single-tenant systems nhưng có thể cần điều chỉnh cho multi-tenant systems với isolation như container orchestration systems.

### Những lỗi phổ biến

- **Race Conditions**: Nhiều Goroutine truy cập cùng một vùng dữ liệu mà không có synchronization
- **Memory Leaks**: Goroutine chạy vô tận hoặc không được cleanup đúng cách
- **Channel Deadlocks**: Gửi hoặc nhận dữ liệu từ channel mà không có corresponding operation

Để tránh các vấn đề này, nên sử dụng channel khi các Goroutine cần giao tiếp với nhau và mutex lock khi chỉ một Goroutine truy cập vào phần code quan trọng.

# Channels

`Channels` là một trong những tính năng cốt lõi của Golang để xử lý đồng thời (concurrency) và giao tiếp giữa các goroutine. Với thiết kế đơn giản nhưng hiệu quả, channels giúp quản lý luồng dữ liệu an toàn giữa các tiến trình đồng thời mà không cần dùng đến locks hay mutexes.

Channel trong Go là một cơ chế giao tiếp có kiểu dữ liệu (typed conduit), cho phép truyền nhận giá trị giữa các goroutine thông qua toán tử `<-`. Có ba loại channel chính:
- **Bidirectional**: `chan T` (vừa gửi vừa nhận)
- **Send-only**: `chan<- T` (chỉ gửi)
- **Receive-only**: `<-chan T` (chỉ nhận)

Ví dụ khởi tạo channel:

```go
unbuffered := make(chan int)      // Unbuffered
buffered := make(chan string, 5)  // Buffered capacity 5
```

## Nguyên lý hoạt động

Channel hoạt động như hàng đợi **FIFO (First-In-First-Out)**. Đối với **unbuffered channel**, thao tác gửi/nhận sẽ block cho đến khi có goroutine khác sẵn sàng thực hiện thao tác ngược lại. **Buffered channel** cho phép lưu trữ nhiều giá trị đến khi đầy buffer mới block.

## buffered vs unbuffered channels

### Tổng quan

1. Unbuffered Channels

- Gửi dữ liệu (send operation) sẽ block cho đến khi có goroutine khác nhận dữ liệu
- Nhận dữ liệu (receive operation) block cho đến khi có dữ liệu được gửi
- Hoạt động theo cơ chế rendezvous - giao tiếp trực tiếp giữa sender và receiver

```go
ch := make(chan int) // Unbuffered
go func() { ch <- 42 }()
fmt.Println(<-ch) // Output: 42
```

2. Buffered Channels

- Cho phép lưu trữ capacity giá trị trước khi block sender
- Sender chỉ block khi buffer đầy
- Receiver block khi buffer rỗng
- Hoạt động như hàng đợi FIFO

```go
ch := make(chan string, 3) // Buffered capacity 3
ch <- "A"
ch <- "B"
fmt.Println(<-ch) // Output: A
```

So sánh

| Đặc điểm | Unbuffered Channel | Buffered Channel |
| -------- | ------- | ------- |
| Khởi tạo | `make(chan T)` | `make(chan T, capacity)` |
| Block sender	| Khi không có receiver | Khi buffer đầy |
| Block receiver | Khi không có dữ liệu | Khi buffer rỗng |
| Đồng bộ | Hoàn toàn đồng bộ | Bán đồng bộ |
| Hiệu suất | Thấp hơn do block thường xuyên | Cao hơn nhờ buffer |
| Sử dụng bộ nhớ | Không cần buffer | Cần cấp phát buffer |
| Phù hợp | Giao tiếp trực tiếp | Xử lý burst traffic |

### Sử dụng thực tế

1. Unbuffered Channel - Đồng bộ hóa chặt

Phối hợp thời gian giữa các goroutines

```go
var done = make(chan struct{})

func worker() {
    // Xử lý công việc
    close(done)
}

func main() {
    go worker()
    <-done // Block cho đến khi worker hoàn thành
}
```

2. Buffered Channel - Xử lý tải biến động

Hệ thống queue xử lý request

```go

const maxRequests = 100
requests := make(chan Request, maxRequests)

// Producer
go func() {
    for req := range incoming {
        requests <- req
    }
}()

// Consumer
for i := 0; i < 5; i++ {
    go func() {
        for req := range requests {
            process(req)
        }
    }()
}
```

### Các vấn đề thường gặp

#### Deadlock với Unbuffered Channel

**Nguyên nhân**: Gửi/nhận không cân bằng

```go
ch := make(chan int)
ch <- 42 // Block vĩnh viễn
fmt.Println(<-ch)
```

**Giải pháp**:

- Luôn đảm bảo có goroutine receiver
- Sử dụng select với timeout

#### Buffer Overflow với Buffered Channel

**Nguyên nhân**: Producer nhanh hơn consumer

```go
ch := make(chan int, 10)
for i := 0; i < 100; i++ {
    ch <- i // Block khi buffer đầy
}
```

**Giải pháp**:

- Giám sát len(ch) và cap(ch)
- Sử dụng backpressure pattern

## Đồng bộ hóa và quản lý luồng

### WaitGroup cho đồng bộ

Package `sync` cung cấp `WaitGroup` để đợi nhiều goroutine hoàn thành:

```go
var wg sync.WaitGroup
wg.Add(2)

go func() {
    defer wg.Done()
    // Xử lý
}()

wg.Wait()
```

### Select statement

`select` cho phép xử lý nhiều channel đồng thời, thường dùng cho:
- Timeout
- Non-blocking operations
- Xử lý nhiều nguồn dữ liệu

```go
select {
case msg := <-ch1:
    fmt.Println(msg)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout")
default:
    // Non-blocking
}
```

## Các mô hình thiết kế phổ biến

### Fan-out/Fan-in

Phân tách công việc thành các task nhỏ (fan-out) và tổng hợp kết quả (fan-in):

```go
func fanOut(in <-chan int, out []chan int) {
    for i := range in {
        for _, ch := range out {
            ch <- i
        }
    }
}

func fanIn(inputs ...<-chan int) <-chan int {
    merged := make(chan int)
    // Merge logic
    return merged
}
```

### Pipeline

Xử lý dữ liệu qua nhiều giai đoạn:

```go
func pipeline(input <-chan int) <-chan int {
    stage1 := processStage(input)
    stage2 := processStage(stage1)
    return stage2
}
```

### Worker Pool

Quản lý nhóm worker xử lý task từ job queue

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j*2
    }
}
```

## Các vấn đề thường gặp

### Phòng tránh deadlock

Các nguyên nhân gây deadlock phổ biến:
- Gửi vào unbuffered channel không có receiver
- Đọc từ channel rỗng
- Quên đóng channel khi dùng range

Giải pháp:

```go
// Sử dụng timeout
select {
case res := <-ch:
    fmt.Println(res)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout")
}

// Sử dụng channel kết hợp context
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()

select {
case <-ctx.Done():
    return ctx.Err()
case res := <-ch:
    // Xử lý res
}
```

### Quản lý tài nguyên

- Đóng channel đúng cách: Chỉ đóng từ phía **sender**, sử dụng `defer close(ch)`
- Tránh nil channels: Luôn khởi tạo channel với `make`
- Sử dụng buffer hợp lý: Ưu tiên buffer nhỏ (3-5) trừ trường hợp cần xử lý burst traffic

```go
// Đúng
ch := make(chan int)
defer close(ch)

// Sai: panic
var ch chan int
ch <- 42
```

### Channel vs Mutex

- Dùng channel khi cần giao tiếp giữa các goroutine
- Dùng mutex khi cần truy cập shared resource

```go
// Channel approach
func counter(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// Mutex approach
var mu sync.Mutex
var count int

func increment() {
    mu.Lock()
    defer mu.Unlock()
    count++
}
```

# Race Condition

**Race condition** xảy ra khi **hai hoặc nhiều goroutine cùng truy cập vào một vùng nhớ chia sẻ**, trong đó ít nhất một truy cập là ghi (write). Kết quả của chương trình phụ thuộc vào thứ tự thực thi không xác định của các goroutine.

Ví dụ:

```go
var counter int
func increment() {
    counter++ // Không an toàn khi chạy đồng thời
}
```

Khi hai goroutine gọi increment(), giá trị counter có thể tăng 1 thay vì 2 do cả hai cùng đọc giá trị ban đầu

## Nguyên nhân phổ biến

- **Capture variable trong closure**: Biến vòng lặp bị capture bởi goroutine

```go
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // In ra giá trị không xác định
    }()
}
```

- **Shared state không được bảo vệ**: Truy cập map, slice hoặc struct từ nhiều goroutine

```go
var m = make(map[int]int)
go func() { m[1] = 1 }()
go func() { m[2] = 2 }() // Data race khi truy cập đồng thời
```

- **Sử dụng biến toàn cục**: Các goroutine thay đổi biến global mà không đồng bộ.

## Giải pháp phòng tránh

### **Sử dụng Mutex**

`sync.Mutex` cung cấp cơ chế lock/unlock để bảo vệ critical section:

```go
var mu sync.Mutex
var counter int

func safeIncrement() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

Lưu ý:
- Luôn unlock bằng defer
- Sử dụng RWMutex cho trường hợp read-heavy

### **Atomic Operations**

Package `sync/atomic` cung cấp các thao tác atomic:

```go
var counter int64
atomic.AddInt64(&counter, 1) // Tăng giá trị an toàn
```

### **Channel Synchronization**

Sử dụng channel để đồng bộ hóa:

```go
ch := make(chan int)
go func() {
    // Xử lý
    ch <- result
}()
result := <-ch
```

Mô hình worker pool với channel:

```go
jobs := make(chan Job, 100)
results := make(chan Result, 100)

// Worker
go func() {
    for job := range jobs {
        results <- process(job)
    }
}()
```

## Mẫu thiết kế an toàn

### **Pipeline**: Xử lý dữ liệu qua nhiều giai đoạn

```go
func pipeline(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n*2
        }
        close(out)
    }()
    return out
}
```

### **Fan-out/Fan-in**: Phân tán và tổng hợp task

```go
func fanOut(in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outs[i] = process(in)
    }
    return outs
}
```

### **errgroup**: Quản lý nhóm goroutine với error handling

```go
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error {
    // Xử lý
    return nil
})
if err := g.Wait(); err != nil {
    // Xử lý lỗi
}
```

### **atomic.Value**: Lưu trữ giá trị an toàn

```go
var config atomic.Value
config.Store(newConfig)
current := config.Load().(Config)
```

### Công cụ hỗ trợ

**Go Race Detector**: Công cụ tích hợp sẵn trong Go toolchain:

```bash
go run -race main.go  # Chạy với race detector
go test -race ./...   # Kiểm tra test cases
```

# Go Scheduler

**Go Runtime** quản lý goroutine thông qua:
- **M (Machine)**: OS thread.
- **P (Processor)**: Logical processor, điều phối goroutine.
- **G (Goroutine)**: Đơn vị xử lý.

Scheduler phân phối G lên M, tối ưu việc sử dụng CPU cores. Khi goroutine bị block (I/O), scheduler tự động chuyển sang G khác.

Ưu điểm:
- **Work-stealing**: P không nhàn rỗi sẽ "đánh cắp" G từ P khác để cân bằng tải.
- **Không cần manual thread pool**: Tự động scale theo số CPU và workload.

Tham khảo thêm: [Scheduling in Go](https://woolly-knave-e27.notion.site/Scheduling-in-Go-1-1bbb5045f2fc8056b3a5dc7bb65a180e)