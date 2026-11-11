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


# Hộp công cụ đồng bộ: `sync`, `sync/atomic`, `errgroup`

Ba package này giải quyết ba lớp vấn đề khác nhau. Chọn sai lớp là nguồn gốc của phần lớn code đồng bộ vừa chậm vừa khó đọc.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 430" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="tb-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="430" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chọn công cụ theo HÌNH DẠNG bài toán, không theo thói quen</text>
<rect x="20" y="48" width="276" height="196" rx="11" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#tb-sh)"/>
<text x="158" y="72" text-anchor="middle" font-size="13" font-weight="800" fill="#3730A3">sync</text>
<text x="158" y="90" text-anchor="middle" font-size="10" fill="#4338CA">Bảo vệ một VÙNG CODE</text>
<rect x="34" y="100" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="34" y="126" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="34" y="152" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="34" y="178" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="34" y="204" width="248" height="22" rx="4" fill="#FFFFFF"/>
<text x="44" y="115" font-size="10" fill="#1C1917"><tspan font-weight="700">Mutex</tspan> — một người vào mỗi lúc</text>
<text x="44" y="141" font-size="10" fill="#1C1917"><tspan font-weight="700">RWMutex</tspan> — nhiều đọc / một ghi</text>
<text x="44" y="167" font-size="10" fill="#1C1917"><tspan font-weight="700">Once</tspan> — khởi tạo đúng một lần</text>
<text x="44" y="193" font-size="10" fill="#1C1917"><tspan font-weight="700">Map</tspan> — map an toàn, không cần khoá tay</text>
<text x="44" y="219" font-size="10" fill="#1C1917"><tspan font-weight="700">Pool</tspan> — tái dùng object, giảm GC</text>
<rect x="312" y="48" width="276" height="196" rx="11" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#tb-sh)"/>
<text x="450" y="72" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">sync/atomic</text>
<text x="450" y="90" text-anchor="middle" font-size="10" fill="#0F766E">Bảo vệ một BIẾN duy nhất</text>
<rect x="326" y="100" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="326" y="126" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="326" y="152" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="326" y="178" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="326" y="204" width="248" height="22" rx="4" fill="#FFFFFF"/>
<text x="336" y="115" font-size="10" fill="#1C1917"><tspan font-weight="700">Int32/Int64</tspan> — bộ đếm</text>
<text x="336" y="141" font-size="10" fill="#1C1917"><tspan font-weight="700">Bool</tspan> — cờ bật/tắt</text>
<text x="336" y="167" font-size="10" fill="#1C1917"><tspan font-weight="700">Pointer[T]</tspan> — đổi nguyên cả snapshot</text>
<text x="336" y="193" font-size="10" fill="#1C1917"><tspan font-weight="700">Value</tspan> — như Pointer, thời trước generic</text>
<text x="336" y="219" font-size="10" fill="#1C1917"><tspan font-weight="700">CompareAndSwap</tspan> — cập nhật có điều kiện</text>
<rect x="604" y="48" width="276" height="196" rx="11" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5" filter="url(#tb-sh)"/>
<text x="742" y="72" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">errgroup</text>
<text x="742" y="90" text-anchor="middle" font-size="10" fill="#92400E">Quản lý VÒNG ĐỜI một nhóm goroutine</text>
<rect x="618" y="100" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="618" y="126" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="618" y="152" width="248" height="22" rx="4" fill="#FFFFFF"/>
<rect x="618" y="178" width="248" height="48" rx="4" fill="#FFFFFF"/>
<text x="628" y="115" font-size="10" fill="#1C1917"><tspan font-weight="700">Go</tspan> — chạy một task trong nhóm</text>
<text x="628" y="141" font-size="10" fill="#1C1917"><tspan font-weight="700">Wait</tspan> — chờ hết, trả lỗi ĐẦU TIÊN</text>
<text x="628" y="167" font-size="10" fill="#1C1917"><tspan font-weight="700">WithContext</tspan> — lỗi một cái, huỷ cả nhóm</text>
<text x="628" y="193" font-size="10" fill="#1C1917"><tspan font-weight="700">SetLimit</tspan> — chặn trần số goroutine</text>
<text x="628" y="211" font-size="10" fill="#57534E">chạy đồng thời</text>
<rect x="20" y="258" width="860" height="68" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="40" y="282" font-size="12" font-weight="800" fill="#374151">Thứ tự cân nhắc khi gặp dữ liệu dùng chung</text>
<text x="40" y="304" font-size="11" fill="#57534E">① Có thể <tspan font-weight="700">không chia sẻ</tspan> không (mỗi goroutine giữ bản riêng, gộp lại ở cuối)?  →  ② Chỉ là <tspan font-weight="700">một biến</tspan>? dùng atomic</text>
<text x="40" y="320" font-size="11" fill="#57534E">③ Cần giữ <tspan font-weight="700">nhiều biến nhất quán</tspan> với nhau? dùng Mutex  →  ④ Cần <tspan font-weight="700">truyền quyền sở hữu</tspan> dữ liệu? dùng channel</text>
<rect x="20" y="340" width="860" height="72" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="40" y="364" font-size="12" font-weight="800" fill="#9F1239">Nhầm lẫn hay gặp nhất: dùng atomic cho NHIỀU biến</text>
<text x="40" y="386" font-size="11" fill="#BE123C">Mỗi thao tác atomic chỉ nguyên tử <tspan font-style="italic">một mình nó</tspan>. Hai lệnh atomic cạnh nhau KHÔNG tạo thành một thao tác nguyên tử:</text>
<text x="40" y="404" font-size="11" fill="#BE123C">goroutine khác vẫn chen được vào giữa và thấy trạng thái dở dang. Cần hai biến nhất quán → <tspan font-weight="700">phải dùng Mutex</tspan>.</text>
</svg>
</div>

## `sync` — bảo vệ một vùng code

```go
// ── Mutex: đúng một goroutine được vào critical section ──
type Counter struct {
	mu sync.Mutex
	n  int
	// Quy ước đặt mu NGAY TRÊN các field mà nó bảo vệ,
	// để người đọc biết chính xác lock này che những gì.
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock() // defer để panic giữa chừng vẫn nhả khoá
	c.n++
}

// ── RWMutex: nhiều reader song song, writer độc quyền ──
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *Cache) Get(k string) (string, bool) {
	c.mu.RLock()         // nhiều goroutine RLock cùng lúc được
	defer c.mu.RUnlock()
	v, ok := c.m[k]
	return v, ok
}

func (c *Cache) Set(k, v string) {
	c.mu.Lock()          // Lock chặn TẤT CẢ reader lẫn writer khác
	defer c.mu.Unlock()
	c.m[k] = v
}
```

Ba cái bẫy của `RWMutex` ít người để ý:

- **`RLock` không tái nhập được.** Gọi `RLock` lồng nhau trên cùng một mutex có thể deadlock nếu có một `Lock` đang xếp hàng ở giữa — Go cố tình chặn writer bị bỏ đói.
- **Không nâng cấp được từ `RLock` lên `Lock`.** Phải `RUnlock` rồi `Lock` lại, và giữa hai lời gọi đó dữ liệu có thể đã đổi — phải kiểm tra lại.
- **`RLock` vẫn là một thao tác GHI vào bộ nhớ dùng chung** (bộ đếm reader). Đây là lý do nó không hề miễn phí, và cũng là lý do nó không scale — phần dưới có số đo cụ thể.

```go
// ── Once: khởi tạo đúng một lần, kể cả khi 100 goroutine cùng gọi ──
var (
	once sync.Once
	conn *Conn
)

func getConn() *Conn {
	once.Do(func() { conn = mustConnect() })
	return conn
}

// ── Pool: tái dùng object tạm, giảm số lần cấp phát ──
var bufPool = sync.Pool{
	New: func() any { return new([4096]byte) },
}

func handle() {
	b := bufPool.Get().(*[4096]byte)
	defer bufPool.Put(b) // trả lại pool, KHÔNG free
	_ = b
}
```

> ⚠️ `sync.Pool` **không phải** connection pool. Nội dung trong pool có thể bị GC dọn sạch bất cứ lúc nào, nên chỉ dùng cho object *tạm thời có thể tạo lại*, không dùng cho tài nguyên cần đóng/mở tường minh. Và luôn **reset** object trước khi `Put` — dữ liệu cũ sẽ theo nó sang request của người khác.

Chạy thử bốn thứ trên:

```
[sync.Once]  100 goroutine gọi getConn() → khởi tạo 1 lần, conn.id=42
[atomic.Bool] 50 goroutine cùng đóng → số goroutine thắng CAS: 1
[CAS]        200 goroutine ghi max đồng thời → max=200 (đúng kỳ vọng 200)
[sync.Pool]  10000 lần Get/Put → chỉ cấp phát mới 10 buffer
```

10.000 lần `Get`/`Put` chỉ sinh **10 buffer mới** — đúng bằng số `P` mà runtime đang dùng, vì `sync.Pool` giữ một kho cục bộ cho mỗi `P` để khỏi phải tranh khoá.

## `sync.Map` — và chuyện nó đã đổi trong Go 1.24

`sync.Map` **không phải** bản thay thế tổng quát cho `map` + `RWMutex`. Nhưng lời khuyên cũ "chỉ dùng khi read-mostly, còn lại thì chậm hơn" đã lỗi thời: Go 1.24 thay ruột `sync.Map` bằng một cấu trúc mới (HashTrieMap). Đo lại trên Go 1.24.4, 8 goroutine, mỗi kịch bản 1.5 giây:

| Kịch bản | `map` + `RWMutex` | `sync.Map` | |
|---|---:|---:|---|
| 99% đọc / 1% ghi, key dùng chung | 27.3 M op/s | **208.4 M op/s** | nhanh hơn **7.6×** |
| 50% ghi / 50% đọc, key riêng mỗi goroutine | 29.5 M op/s | **50.7 M op/s** | nhanh hơn **1.7×** |
| Ghi liên tục **key mới** | 4.7 M op/s | **5.8 M op/s** | nhanh hơn **1.2×** |

Kịch bản thứ ba vốn là điểm yếu kinh điển của `sync.Map` cũ, giờ cũng không còn thua. Vậy khi nào vẫn nên chọn `map` + `RWMutex`?

- **Cần type safety.** `sync.Map` làm việc với `any`, mỗi lần `Load` phải ép kiểu — sai kiểu là panic lúc chạy, không phải lỗi lúc compile.
- **Cần nhiều thao tác nhất quán với nhau.** Ví dụ "đọc, tính toán, rồi ghi lại, không ai chen vào giữa" — `sync.Map` không có cách giữ khoá xuyên suốt nhiều lời gọi.
- **Cần `len()`.** `sync.Map` không có; muốn đếm phải `Range` qua toàn bộ.

## `sync/atomic` — bảo vệ đúng một biến

Từ Go 1.19 nên dùng **kiểu** `atomic.Int64` thay cho **hàm** `atomic.AddInt64(&x, 1)`: kiểu tự bảo đảm căn chỉnh bộ nhớ, và quan trọng hơn, nó khiến người đọc không thể vô tình truy cập biến đó theo kiểu thường.

```go
// ❌ Kiểu cũ: dễ lỡ tay đọc thẳng `counter` ở chỗ khác mà quên atomic
var counter int64
atomic.AddInt64(&counter, 1)

// ✅ Kiểu mới: mọi truy cập BẮT BUỘC đi qua method
var counter atomic.Int64
counter.Add(1)
n := counter.Load()
```

```go
// ── atomic.Bool: cờ một chiều, ai bật được là người duy nhất ──
var closed atomic.Bool

func Close() {
	if !closed.CompareAndSwap(false, true) {
		return // đã có goroutine khác đóng rồi → thoát, không đóng hai lần
	}
	cleanup()
}

// ── atomic.Pointer[T]: thay NGUYÊN CẢ snapshot, không sửa tại chỗ ──
var cfg atomic.Pointer[Config]

func Reload(newCfg *Config) { cfg.Store(newCfg) } // writer
func Current() *Config      { return cfg.Load() } // reader, không khoá gì cả

// ── CompareAndSwap: cập nhật có điều kiện, luôn đi kèm vòng retry ──
func recordMax(m *atomic.Int64, v int64) {
	for {
		old := m.Load()
		if v <= old {
			return // đã có giá trị lớn hơn, không cần ghi
		}
		if m.CompareAndSwap(old, v) {
			return // ghi thành công
		}
		// CAS trượt = có goroutine khác vừa ghi xen vào → đọc lại và thử lại
	}
}
```

> ★ Quy tắc đáng nhớ: **atomic nguyên tử ở mức một lệnh, không phải một đoạn code.** `if counter.Load() == 0 { counter.Store(1) }` hoàn toàn **không** an toàn — hai goroutine có thể cùng đọc được `0`. Muốn "kiểm tra rồi ghi" là một thao tác không thể chia cắt thì phải dùng `CompareAndSwap`, hoặc dùng Mutex.

## `errgroup` — quản lý vòng đời cả nhóm goroutine

`WaitGroup` chỉ biết *chờ*; nó không mang lỗi về, cũng không biết dừng sớm. `errgroup` bù đúng hai chỗ đó.

```go
import "golang.org/x/sync/errgroup"

// ── 1. Wait: chờ tất cả, trả về lỗi ĐẦU TIÊN gặp phải ──
var g errgroup.Group
for _, url := range urls {
	g.Go(func() error { return fetch(url) }) // Go 1.22+ không cần copy biến vòng lặp
}
if err := g.Wait(); err != nil {
	return err
}

// ── 2. WithContext: một task lỗi → ctx bị huỷ → các task khác dừng theo ──
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error { return fetchUser(ctx) })
g.Go(func() error { return fetchOrder(ctx) })   // nếu cái này lỗi...
g.Go(func() error { return fetchReport(ctx) })  // ...cái này nhận ctx.Done() và thoát sớm
err := g.Wait()

// ── 3. SetLimit: chặn trần số goroutine chạy đồng thời ──
g := new(errgroup.Group)
g.SetLimit(10)           // 10.000 task nhưng tối đa 10 chạy một lúc
for _, item := range tenThousandItems {
	g.Go(func() error { return process(item) })
}
err := g.Wait()
```

Kết quả chạy thật — chú ý thời gian ở mục [2]:

```
[1] errgroup.Group + Wait — tất cả thành công
    payment  xong sau 50ms
    order    xong sau 50ms
    user     xong sau 50ms
  Wait() → <nil>

[2] errgroup.WithContext — 'order' lỗi ở 50ms, 'report' đang chạy 2s
    user     xong sau 20ms
    report   bị huỷ giữa chừng (context canceled)
  Wait() → order: dịch vụ trả 500  (chỉ mất 50ms, KHÔNG phải 2s)

[3] SetLimit(2) — 6 task nhưng tối đa 2 chạy cùng lúc
  số goroutine chạy đồng thời cao nhất đo được: 2

[4] Wait() chỉ trả 1 lỗi — lỗi xảy ra ĐẦU TIÊN
  Wait() → "lỗi SỚM (10ms)"
  ctx bị huỷ chưa? true → lỗi SỚM (10ms)
```

Ba điều cần nhớ:

- **`Wait()` chỉ trả về *một* lỗi** — lỗi xảy ra sớm nhất. Muốn gom hết thì tự thu lỗi vào slice có khoá, hoặc dùng `errors.Join`.
- **`WithContext` chỉ huỷ ngữ cảnh, không giết được goroutine.** Nếu task của bạn không bao giờ `select` trên `ctx.Done()` thì nó vẫn chạy tới hết — huỷ chỉ là một *tín hiệu*.
- **Không gọi `g.Go` sau khi đã `g.Wait`.** Nhóm đã đóng; hành vi không xác định.


# Thiết kế: một goroutine ghi liên tục, nhiều goroutine đọc

Đây là hình dạng rất hay gặp: một goroutine nền cập nhật config / bảng tỉ giá / routing table / snapshot giá, còn hàng chục goroutine xử lý request chỉ đọc nó. Trực giác thông thường là "read-heavy thì dùng `RWMutex`". **Số đo nói ngược lại.**

## Ba cách làm

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 470" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="rw-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#78716C"/></marker>
<marker id="rw-arg" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="rw-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="470" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng một bài toán — ba cách, ba kết cục khác hẳn nhau</text>
<rect x="20" y="44" width="276" height="250" rx="11" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#rw-sh)"/>
<text x="158" y="68" text-anchor="middle" font-size="12.5" font-weight="800" fill="#9F1239">① Mutex</text>
<text x="158" y="86" text-anchor="middle" font-size="10" fill="#BE123C">Reader xếp hàng từng người một</text>
<rect x="36" y="98" width="68" height="26" rx="5" fill="#374151"/>
<text x="70" y="115" text-anchor="middle" font-size="9.5" font-weight="700" fill="#FFFFFF">writer</text>
<rect x="36" y="144" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="36" y="172" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="36" y="200" width="68" height="22" rx="5" fill="#A8A29E"/>
<text x="70" y="159" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 1</text>
<text x="70" y="187" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 2</text>
<text x="70" y="215" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 3</text>
<rect x="188" y="130" width="84" height="100" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<text x="230" y="172" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">CỬA</text>
<text x="230" y="188" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">HẸP</text>
<text x="230" y="206" text-anchor="middle" font-size="9" fill="#BE123C">1 người/lượt</text>
<line x1="108" y1="111" x2="184" y2="140" stroke="#DC2626" stroke-width="2" marker-end="url(#rw-ar)"/>
<line x1="108" y1="155" x2="184" y2="165" stroke="#DC2626" stroke-width="1.6" marker-end="url(#rw-ar)"/>
<line x1="108" y1="183" x2="184" y2="183" stroke="#DC2626" stroke-width="1.6" marker-end="url(#rw-ar)"/>
<line x1="108" y1="211" x2="184" y2="201" stroke="#DC2626" stroke-width="1.6" marker-end="url(#rw-ar)"/>
<rect x="36" y="244" width="236" height="38" rx="6" fill="#FFE9EC"/>
<text x="154" y="260" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">Thêm reader = CHẬM đi</text>
<text x="154" y="275" text-anchor="middle" font-size="10" fill="#BE123C">8 reader chỉ còn 0.18× so với 1 reader</text>
<rect x="312" y="44" width="276" height="250" rx="11" fill="#FFFBEB" stroke="#F5A623" stroke-width="2.5" filter="url(#rw-sh)"/>
<text x="450" y="68" text-anchor="middle" font-size="12.5" font-weight="800" fill="#7C4A03">② RWMutex</text>
<text x="450" y="86" text-anchor="middle" font-size="10" fill="#92400E">Reader vào cùng lúc — nhưng vẫn phải GHI</text>
<rect x="328" y="98" width="68" height="26" rx="5" fill="#374151"/>
<text x="362" y="115" text-anchor="middle" font-size="9.5" font-weight="700" fill="#FFFFFF">writer</text>
<rect x="328" y="144" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="328" y="172" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="328" y="200" width="68" height="22" rx="5" fill="#A8A29E"/>
<text x="362" y="159" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 1</text>
<text x="362" y="187" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 2</text>
<text x="362" y="215" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 3</text>
<rect x="470" y="140" width="106" height="82" rx="7" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5"/>
<text x="523" y="166" text-anchor="middle" font-size="10.5" font-weight="800" fill="#7C4A03">bộ đếm reader</text>
<text x="523" y="184" text-anchor="middle" font-size="9.5" fill="#92400E">RLock = +1 vào một</text>
<text x="523" y="198" text-anchor="middle" font-size="9.5" fill="#92400E">ô nhớ DÙNG CHUNG</text>
<text x="523" y="214" text-anchor="middle" font-size="8.5" font-weight="700" fill="#B45309">→ cache line bật qua lại</text>
<line x1="400" y1="155" x2="466" y2="168" stroke="#F5A623" stroke-width="1.8" marker-end="url(#rw-ar)"/>
<line x1="400" y1="183" x2="466" y2="183" stroke="#F5A623" stroke-width="1.8" marker-end="url(#rw-ar)"/>
<line x1="400" y1="211" x2="466" y2="198" stroke="#F5A623" stroke-width="1.8" marker-end="url(#rw-ar)"/>
<rect x="328" y="244" width="236" height="38" rx="6" fill="#FFF3DB"/>
<text x="446" y="260" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">Thêm reader VẪN chậm đi</text>
<text x="446" y="275" text-anchor="middle" font-size="10" fill="#92400E">8 reader chỉ còn 0.22× — gần như Mutex</text>
<rect x="604" y="44" width="276" height="250" rx="11" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#rw-sh)"/>
<text x="742" y="68" text-anchor="middle" font-size="12.5" font-weight="800" fill="#134E4A">③ atomic.Pointer (copy-on-write)</text>
<text x="742" y="86" text-anchor="middle" font-size="10" fill="#0F766E">Reader chỉ ĐỌC, không ghi gì cả</text>
<rect x="620" y="98" width="68" height="26" rx="5" fill="#374151"/>
<text x="654" y="115" text-anchor="middle" font-size="9.5" font-weight="700" fill="#FFFFFF">writer</text>
<rect x="620" y="144" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="620" y="172" width="68" height="22" rx="5" fill="#A8A29E"/>
<rect x="620" y="200" width="68" height="22" rx="5" fill="#A8A29E"/>
<text x="654" y="159" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 1</text>
<text x="654" y="187" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 2</text>
<text x="654" y="215" text-anchor="middle" font-size="9" font-weight="700" fill="#FFFFFF">reader 3</text>
<rect x="762" y="140" width="106" height="82" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="815" y="162" text-anchor="middle" font-size="10.5" font-weight="800" fill="#134E4A">con trỏ snapshot</text>
<text x="815" y="180" text-anchor="middle" font-size="9.5" fill="#0F766E">writer tạo bản MỚI</text>
<text x="815" y="194" text-anchor="middle" font-size="9.5" fill="#0F766E">rồi đổi con trỏ</text>
<text x="815" y="212" text-anchor="middle" font-size="9.5" font-weight="700" fill="#0D9488">reader chỉ đọc</text>
<line x1="692" y1="111" x2="758" y2="146" stroke="#0D9488" stroke-width="2.5" marker-end="url(#rw-arg)"/>
<line x1="692" y1="155" x2="758" y2="170" stroke="#0D9488" stroke-width="1.6" marker-end="url(#rw-arg)"/>
<line x1="692" y1="183" x2="758" y2="183" stroke="#0D9488" stroke-width="1.6" marker-end="url(#rw-arg)"/>
<line x1="692" y1="211" x2="758" y2="196" stroke="#0D9488" stroke-width="1.6" marker-end="url(#rw-arg)"/>
<rect x="620" y="244" width="236" height="38" rx="6" fill="#CCFBF1"/>
<text x="738" y="260" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">Thêm reader = NHANH lên</text>
<text x="738" y="275" text-anchor="middle" font-size="10" fill="#0F766E">8 reader đạt 5.00× so với 1 reader</text>
<rect x="20" y="308" width="860" height="70" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="40" y="332" font-size="12" font-weight="800" fill="#374151">Vì sao RWMutex không scale dù chỉ đọc</text>
<text x="40" y="354" font-size="11" fill="#57534E">Vì <tspan font-weight="700">RLock bản thân nó là một lệnh GHI</tspan>: nó tăng bộ đếm reader nằm trong mutex. Tám core cùng ghi vào một ô nhớ</text>
<text x="40" y="370" font-size="11" fill="#57534E">khiến cache line chứa ô đó liên tục bị giành qua lại giữa các core — chính là hiện tượng ở phần sau của bài.</text>
<rect x="20" y="390" width="860" height="64" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="40" y="414" font-size="12" font-weight="800" fill="#134E4A">Ý tưởng của copy-on-write: biến dữ liệu dùng chung thành dữ liệu CHỈ ĐỌC</text>
<text x="40" y="436" font-size="11" fill="#0F766E">Snapshot cũ không bao giờ bị sửa — writer dựng bản mới rồi đổi con trỏ. Reader đang cầm bản cũ cứ đọc tiếp an toàn,</text>
<text x="40" y="450" font-size="11" fill="#0F766E">GC thu hồi bản cũ khi không còn ai cầm. Không khoá, không chờ, không ghi vào bộ nhớ chung.</text>
</svg>
</div>

```go
type Config struct {
	Version int
	Hosts   []string
}

// ── ① Mutex ──
type mutexStore struct {
	mu  sync.Mutex
	cfg *Config
}

func (s *mutexStore) Load() *Config   { s.mu.Lock(); c := s.cfg; s.mu.Unlock(); return c }
func (s *mutexStore) Store(c *Config) { s.mu.Lock(); s.cfg = c; s.mu.Unlock() }

// ── ② RWMutex ──
type rwStore struct {
	mu  sync.RWMutex
	cfg *Config
}

func (s *rwStore) Load() *Config   { s.mu.RLock(); c := s.cfg; s.mu.RUnlock(); return c }
func (s *rwStore) Store(c *Config) { s.mu.Lock(); s.cfg = c; s.mu.Unlock() }

// ── ③ atomic.Pointer — copy-on-write ──
type atomicStore struct {
	cfg atomic.Pointer[Config]
}

func (s *atomicStore) Load() *Config   { return s.cfg.Load() }  // KHÔNG khoá gì cả
func (s *atomicStore) Store(c *Config) { s.cfg.Store(c) }
```

## Số đo

Một goroutine ghi 1000 lần/giây, N goroutine đọc liên tục, đo 2 giây mỗi phương án, máy 12 core:

| | 1 reader | 4 readers | 8 readers | Scale 1→8 |
|---|---:|---:|---:|---|
| `sync.Mutex` | 90.8 M/s | 36.7 M/s | 16.6 M/s | **0.18×** ⬇ |
| `sync.RWMutex` | 89.8 M/s | 22.7 M/s | 19.7 M/s | **0.22×** ⬇ |
| `atomic.Pointer` | 564.9 M/s | 1684.9 M/s | 2825.5 M/s | **5.00×** ⬆ |

Hai điều đáng chú ý, và cả hai đều trái với trực giác:

1. **`RWMutex` gần như không khá hơn `Mutex`** trong bài toán này (0.22× so với 0.18×). Nó chỉ thắng rõ rệt khi mỗi lần đọc *giữ khoá đủ lâu* — đọc xong một cấu trúc lớn, duyệt một map vài nghìn phần tử. Nếu critical section chỉ là vài nanosecond thì chi phí vào/ra khoá đã lớn hơn chính công việc cần làm.
2. **Thêm reader làm cả hai loại mutex chậm đi**, chứ không phải nhanh lên. Reader không hề "đọc song song" — chúng đang tranh nhau quyền ghi vào bộ đếm bên trong mutex.

## Khi nào dùng cách nào

| Tình huống | Chọn |
|---|---|
| Snapshot bất biến, ghi hiếm (config, routing table, feature flag) | **`atomic.Pointer[T]` copy-on-write** |
| Reader cần giữ dữ liệu ổn định xuyên suốt một thao tác dài | **`RWMutex`** |
| Ghi nhiều ngang đọc, hoặc cần sửa tại chỗ nhiều field | **`Mutex`** |
| Dữ liệu dạng key-value, mỗi key độc lập nhau | **`sync.Map`** |
| Writer cần biết reader đã xử lý xong chưa | **channel** (atomic/mutex không làm được việc này) |

Hai điều kiện bắt buộc khi dùng copy-on-write — vi phạm là có race ngay:

```go
// ❌ SAI: sửa tại chỗ trên snapshot đang có reader đọc
cfg := store.Load()
cfg.Hosts = append(cfg.Hosts, "new-host") // reader khác đang đọc chính slice này!
store.Store(cfg)

// ✅ ĐÚNG: dựng bản MỚI hoàn toàn, rồi mới đổi con trỏ
old := store.Load()
newCfg := &Config{
	Version: old.Version + 1,
	Hosts:   append(append([]string{}, old.Hosts...), "new-host"), // copy slice
}
store.Store(newCfg)
```

- **Snapshot phải bất biến sau khi `Store`.** Không ai được sửa nó nữa, kể cả writer.
- **Phải copy sâu những gì dùng chung.** `append` lên slice cũ có thể ghi đè vào mảng nền mà reader đang đọc.

> ★ Còn một điểm nữa thường bị bỏ qua: với copy-on-write, hai lần `Load()` liên tiếp **có thể trả về hai snapshot khác nhau**. Nếu một thao tác cần dữ liệu nhất quán thì `Load()` **một lần** vào biến local rồi dùng xuyên suốt, đừng gọi `Load()` nhiều lần.


# Cache line và false sharing

Phần trên kết luận "RWMutex không scale vì cache line bị giành qua lại". Đây là phần giải thích câu đó, và nó là một trong số ít chỗ mà kiến thức phần cứng ảnh hưởng trực tiếp tới cách bạn khai báo một struct trong Go.

## Cache line là gì

CPU **không bao giờ** đọc bộ nhớ theo từng byte. Đơn vị nhỏ nhất nó chuyển giữa RAM và cache là một **cache line** — trên hầu hết CPU x86-64 và ARM hiện đại là **64 byte**.

```bash
# Kiểm tra trên máy Linux
getconf LEVEL1_DCACHE_LINESIZE   # → 64
```

Hệ quả: đọc 1 byte hay đọc 64 byte liền nhau đều tốn **đúng một lần** nạp. Và quan trọng hơn — **cache line cũng là đơn vị của giao thức đồng bộ cache giữa các core**. Khi một core ghi vào bất kỳ byte nào trong một line, mọi core khác đang giữ bản sao của line đó phải **vứt bỏ bản sao của mình** và nạp lại.

## False sharing

**False sharing** xảy ra khi hai core ghi vào **hai biến khác nhau nhưng nằm chung một cache line**. Về mặt logic chúng hoàn toàn độc lập — không có dữ liệu nào thật sự dùng chung. Nhưng phần cứng chỉ nhìn ở mức cache line, nên nó xử lý y như thể hai core đang tranh cùng một biến.

Gọi là *false* vì đó là tranh chấp **giả** — do cách bố trí bộ nhớ, không phải do logic chương trình.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 470" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="fs-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="fs-arg" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="fs-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="470" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai biến độc lập, một cache line — phần cứng không phân biệt được</text>
<rect x="20" y="46" width="420" height="212" rx="11" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="230" y="70" text-anchor="middle" font-size="12.5" font-weight="800" fill="#9F1239">KHÔNG padding — chung 1 cache line</text>
<rect x="40" y="84" width="76" height="30" rx="5" fill="#374151"/>
<rect x="344" y="84" width="76" height="30" rx="5" fill="#374151"/>
<text x="78" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">Core 0</text>
<text x="382" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">Core 1</text>
<rect x="110" y="150" width="240" height="40" rx="5" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<rect x="118" y="158" width="112" height="24" rx="3" fill="#FCA5A5"/>
<rect x="234" y="158" width="108" height="24" rx="3" fill="#FCA5A5"/>
<text x="174" y="174" text-anchor="middle" font-size="10" font-weight="700" fill="#7F1D1D">counters[0]</text>
<text x="288" y="174" text-anchor="middle" font-size="10" font-weight="700" fill="#7F1D1D">counters[1]</text>
<text x="230" y="206" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">CACHE LINE — 64 byte</text>
<path d="M78,118 C78,140 110,152 150,156" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#fs-ar)"/>
<path d="M382,118 C382,140 350,152 312,156" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#fs-ar)"/>
<text x="230" y="134" text-anchor="middle" font-size="10.5" font-weight="800" fill="#DC2626">mỗi lần ghi → line bị giành lại</text>
<rect x="40" y="218" width="380" height="28" rx="6" fill="#FFE9EC"/>
<text x="230" y="236" text-anchor="middle" font-size="10.5" font-weight="700" fill="#9F1239">Core 1 phải vứt bản sao và nạp lại, dù nó KHÔNG đụng counters[0]</text>
<rect x="460" y="46" width="420" height="212" rx="11" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="670" y="70" text-anchor="middle" font-size="12.5" font-weight="800" fill="#134E4A">CÓ padding — mỗi biến một cache line</text>
<rect x="480" y="84" width="76" height="30" rx="5" fill="#374151"/>
<rect x="784" y="84" width="76" height="30" rx="5" fill="#374151"/>
<text x="518" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">Core 0</text>
<text x="822" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">Core 1</text>
<rect x="476" y="150" width="184" height="40" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="680" y="150" width="184" height="40" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="484" y="158" width="76" height="24" rx="3" fill="#5EEAD4"/>
<rect x="688" y="158" width="76" height="24" rx="3" fill="#5EEAD4"/>
<rect x="566" y="158" width="86" height="24" rx="3" fill="#E7E5E4"/>
<rect x="770" y="158" width="86" height="24" rx="3" fill="#E7E5E4"/>
<text x="522" y="174" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">counters[0]</text>
<text x="726" y="174" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">counters[1]</text>
<text x="609" y="174" text-anchor="middle" font-size="9" fill="#78716C">padding 56B</text>
<text x="813" y="174" text-anchor="middle" font-size="9" fill="#78716C">padding 56B</text>
<text x="568" y="206" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">CACHE LINE 1</text>
<text x="772" y="206" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">CACHE LINE 2</text>
<path d="M518,118 C518,134 518,142 518,146" stroke="#0D9488" stroke-width="2.5" fill="none" marker-end="url(#fs-arg)"/>
<path d="M822,118 C822,134 822,142 822,146" stroke="#0D9488" stroke-width="2.5" fill="none" marker-end="url(#fs-arg)"/>
<text x="670" y="134" text-anchor="middle" font-size="10.5" font-weight="800" fill="#0D9488">hai core không đụng nhau nữa</text>
<rect x="480" y="218" width="380" height="28" rx="6" fill="#CCFBF1"/>
<text x="670" y="236" text-anchor="middle" font-size="10.5" font-weight="700" fill="#134E4A">Đổi lại: tốn 512 byte thay vì 64 byte cho cùng 8 bộ đếm</text>
<rect x="20" y="272" width="860" height="94" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="40" y="296" font-size="12" font-weight="800" fill="#374151">Số đo thật — 50 triệu lần Add, máy 12 core, cache line 64 byte</text>
<rect x="40" y="306" width="200" height="20" rx="4" fill="#F5F5F4"/>
<rect x="244" y="306" width="196" height="20" rx="4" fill="#FFE9EC"/>
<rect x="444" y="306" width="196" height="20" rx="4" fill="#CCFBF1"/>
<rect x="644" y="306" width="196" height="20" rx="4" fill="#F5F5F4"/>
<text x="140" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">số goroutine</text>
<text x="342" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">không padding</text>
<text x="542" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">có padding</text>
<text x="742" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">chậm hơn bao nhiêu</text>
<text x="140" y="342" text-anchor="middle" font-size="11" fill="#57534E">1</text>
<text x="342" y="342" text-anchor="middle" font-size="11" fill="#57534E">243 ms</text>
<text x="542" y="342" text-anchor="middle" font-size="11" fill="#57534E">250 ms</text>
<text x="742" y="342" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E">0.97× — không khác gì</text>
<text x="140" y="360" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">8</text>
<text x="342" y="360" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">861 ms</text>
<text x="542" y="360" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">62 ms</text>
<text x="742" y="360" text-anchor="middle" font-size="11" font-weight="800" fill="#DC2626">13.94× CHẬM HƠN</text>
<rect x="20" y="378" width="860" height="76" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="40" y="402" font-size="12" font-weight="800" fill="#7C4A03">Dấu hiệu nhận biết false sharing trong hệ thống thật</text>
<text x="40" y="424" font-size="11" fill="#92400E">Thêm core / thêm goroutine mà thông lượng <tspan font-weight="700">đứng yên hoặc tụt</tspan>, trong khi mỗi goroutine rõ ràng làm việc trên dữ liệu riêng.</text>
<text x="40" y="442" font-size="11" fill="#92400E">Với 1 goroutine thì không có dấu hiệu gì — đây là lớp lỗi <tspan font-weight="700">chỉ lộ ra khi chạy song song</tspan>, và profiler CPU thường chỉ nói "tốn ở lệnh Add".</text>
</svg>
</div>

## Tái hiện bằng code

```go
const iters = 50_000_000

// ❌ 8 bộ đếm × 8 byte = 64 byte = ĐÚNG một cache line
type packed struct {
	counters [8]atomic.Int64
}

// ✅ Mỗi bộ đếm chiếm trọn một cache line riêng
type paddedCounter struct {
	v atomic.Int64
	_ [64 - unsafe.Sizeof(atomic.Int64{})]byte // chèn 56 byte rác cho đủ 64
}

type padded struct {
	counters [8]paddedCounter
}

// Mỗi goroutine chỉ đụng ĐÚNG MỘT phần tử của riêng nó —
// về mặt logic không có dữ liệu nào dùng chung cả.
func run(n int) {
	var p packed
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < iters/n; j++ {
				p.counters[id].Add(1)
			}
		}(i)
	}
	wg.Wait()
}
```

Kết quả, máy 12 core:

```
sizeof(packed)=64 byte  sizeof(padded)=512 byte

goroutine   KHÔNG padding     CÓ padding   chậm hơn
1                   243ms          250ms      0.97x
2                   633ms          156ms      4.05x
4                   759ms          113ms      6.69x
8                   861ms           62ms     13.94x
```

Đọc kỹ hai cột giữa, vì chúng đi **ngược chiều nhau**:

- Bản **có padding** hành xử đúng như mong đợi: 1 → 8 goroutine, thời gian giảm từ 250ms xuống 62ms (nhanh hơn **4×**).
- Bản **không padding** thì càng thêm goroutine càng **chậm**: 243ms → 861ms. Tám core làm việc trên tám biến riêng biệt, vậy mà tổng thời gian tăng gấp 3,5 lần so với một core làm hết.
- Với **1 goroutine, hai bản như nhau** (0.97×). Đây là lý do false sharing không bao giờ lộ ra trong unit test chạy tuần tự.

## Khi nào thật sự cần quan tâm

Đừng đi padding mọi struct — bạn sẽ phình bộ nhớ lên 8 lần để đổi lấy không gì cả. Chỉ cân nhắc khi **cả ba** điều sau cùng đúng:

1. Nhiều goroutine **ghi** (không phải chỉ đọc — nhiều core cùng *đọc* một cache line hoàn toàn không sao).
2. Ghi vào các biến **nằm gần nhau trong bộ nhớ** — các phần tử của một mảng, các field liền kề của một struct.
3. Tần suất ghi **rất cao**, nằm trong hot path.

Trong Go, hình dạng hay dính nhất là **mảng số liệu per-worker**:

```go
// ❌ stats[i] của worker i nằm sát stats[i+1] của worker i+1
stats := make([]atomic.Int64, numWorkers)

// ✅ Cách 1: padding
type stat struct {
	v atomic.Int64
	_ [56]byte
}
stats := make([]stat, numWorkers)

// ✅ Cách 2 (thường tốt hơn): đừng chia sẻ ngay từ đầu.
// Mỗi worker đếm vào biến LOCAL trên stack, chỉ gộp MỘT LẦN lúc kết thúc.
func worker(id int, out chan<- int64) {
	var local int64 // không ai khác chạm tới → không cache line nào bị giành
	for range work {
		local++
	}
	out <- local
}
```

> ★ Cách 2 gần như luôn thắng cách 1: nó vừa bỏ được false sharing, vừa bỏ luôn chi phí của chính lệnh atomic. Padding là giải pháp cho trường hợp **bắt buộc** phải có bộ đếm dùng chung đọc được theo thời gian thực.

Và quay lại chuyện `RWMutex` ở phần trước — giờ thì rõ: `RLock` làm tăng một bộ đếm nằm trong struct mutex. Tám goroutine `RLock` cùng lúc là tám core cùng ghi vào **cùng một cache line**. Đó là **true sharing** (tranh chấp thật, không phải giả), nhưng cơ chế gây chậm thì giống hệt — và vì thế `atomic.Pointer`, vốn chỉ *đọc*, thắng tuyệt đối.


# Circular wait: deadlock dù mỗi tài nguyên đã có mutex riêng

Bài toán kinh điển của fintech: hai tài khoản chuyển tiền cho nhau. Mỗi account có mutex riêng — nghe đã đủ an toàn. Nhưng chính việc *mỗi cái một khoá* lại tạo ra khả năng chờ vòng tròn.

## Code gây deadlock

```go
type Account struct {
	id      int
	mu      sync.Mutex
	balance int
}

// ❌ SAI: thứ tự khoá phụ thuộc vào THỨ TỰ THAM SỐ truyền vào
func transfer(from, to *Account, amount int) {
	from.mu.Lock()
	defer from.mu.Unlock()

	time.Sleep(10 * time.Millisecond) // nới cửa sổ cho lỗi hiện ra chắc chắn

	to.mu.Lock()
	defer to.mu.Unlock()

	from.balance -= amount
	to.balance += amount
}

func main() {
	a := &Account{id: 1, balance: 1000}
	b := &Account{id: 2, balance: 1000}

	var wg sync.WaitGroup
	wg.Add(2)
	go func() { defer wg.Done(); transfer(a, b, 100) }() // khoá a, rồi chờ b
	go func() { defer wg.Done(); transfer(b, a, 200) }() // khoá b, rồi chờ a
	wg.Wait()
}
```

Code này **không có race condition** — mọi truy cập đều có khoá bảo vệ, `go run -race` hoàn toàn sạch. Nó chỉ đơn giản là treo vĩnh viễn.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="dl-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="dl-arg" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<marker id="dl-arh" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="dl-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chu trình chờ — và cách bẻ gãy nó</text>
<rect x="20" y="46" width="420" height="250" rx="11" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#dl-sh)"/>
<text x="230" y="70" text-anchor="middle" font-size="12.5" font-weight="800" fill="#9F1239">❌ Khoá theo thứ tự tham số</text>
<rect x="44" y="86" width="150" height="34" rx="6" fill="#374151"/>
<rect x="266" y="86" width="150" height="34" rx="6" fill="#374151"/>
<text x="119" y="107" text-anchor="middle" font-size="10.5" font-weight="700" fill="#FFFFFF">goroutine 1: A → B</text>
<text x="341" y="107" text-anchor="middle" font-size="10.5" font-weight="700" fill="#FFFFFF">goroutine 2: B → A</text>
<rect x="60" y="164" width="118" height="54" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<rect x="282" y="164" width="118" height="54" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<text x="119" y="186" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">mutex A</text>
<text x="119" y="206" text-anchor="middle" font-size="9.5" fill="#BE123C">g1 đang GIỮ</text>
<text x="341" y="186" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">mutex B</text>
<text x="341" y="206" text-anchor="middle" font-size="9.5" fill="#BE123C">g2 đang GIỮ</text>
<line x1="100" y1="124" x2="100" y2="158" stroke="#57534E" stroke-width="2" marker-end="url(#dl-arh)"/>
<line x1="360" y1="124" x2="360" y2="158" stroke="#57534E" stroke-width="2" marker-end="url(#dl-arh)"/>
<text x="76" y="144" text-anchor="middle" font-size="9" fill="#57534E">giữ</text>
<text x="384" y="144" text-anchor="middle" font-size="9" fill="#57534E">giữ</text>
<path d="M182,178 C224,170 244,172 278,176" stroke="#DC2626" stroke-width="2.8" fill="none" marker-end="url(#dl-ar)"/>
<path d="M278,206 C244,212 224,214 182,206" stroke="#DC2626" stroke-width="2.8" fill="none" marker-end="url(#dl-ar)"/>
<text x="230" y="164" text-anchor="middle" font-size="9.5" font-weight="800" fill="#DC2626">g1 CHỜ B</text>
<text x="230" y="232" text-anchor="middle" font-size="9.5" font-weight="800" fill="#DC2626">g2 CHỜ A</text>
<rect x="44" y="246" width="372" height="36" rx="6" fill="#FFE9EC"/>
<text x="230" y="262" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Vòng tròn khép kín → không ai nhả, không ai đi tiếp</text>
<text x="230" y="276" text-anchor="middle" font-size="10" fill="#BE123C">Treo vĩnh viễn. Không có race, go run -race hoàn toàn sạch.</text>
<rect x="460" y="46" width="420" height="250" rx="11" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#dl-sh)"/>
<text x="670" y="70" text-anchor="middle" font-size="12.5" font-weight="800" fill="#134E4A">✅ Khoá theo thứ tự id tăng dần</text>
<rect x="484" y="86" width="150" height="34" rx="6" fill="#374151"/>
<rect x="706" y="86" width="150" height="34" rx="6" fill="#374151"/>
<text x="559" y="107" text-anchor="middle" font-size="10.5" font-weight="700" fill="#FFFFFF">goroutine 1: A → B</text>
<text x="781" y="107" text-anchor="middle" font-size="10.5" font-weight="700" fill="#FFFFFF">goroutine 2: B → A</text>
<rect x="500" y="164" width="118" height="54" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="722" y="164" width="118" height="54" rx="7" fill="#F5F5F4" stroke="#A8A29E" stroke-width="2"/>
<text x="559" y="186" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">mutex A (id=1)</text>
<text x="559" y="206" text-anchor="middle" font-size="9.5" fill="#0F766E">CẢ HAI đều xin TRƯỚC</text>
<text x="781" y="186" text-anchor="middle" font-size="12" font-weight="800" fill="#57534E">mutex B (id=2)</text>
<text x="781" y="206" text-anchor="middle" font-size="9.5" fill="#78716C">chỉ xin SAU khi đã có A</text>
<path d="M540,124 C530,140 528,150 538,158" stroke="#0D9488" stroke-width="2.5" fill="none" marker-end="url(#dl-arg)"/>
<path d="M760,124 C700,136 640,150 616,166" stroke="#0D9488" stroke-width="2.5" fill="none" marker-end="url(#dl-arg)"/>
<path d="M622,196 C660,210 690,208 718,200" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#dl-arg)"/>
<text x="670" y="230" text-anchor="middle" font-size="9.5" font-weight="700" fill="#0D9488">g2 chuyển B → A nhưng VẪN xin mutex A trước</text>
<text x="670" y="244" text-anchor="middle" font-size="9.5" fill="#0F766E">ai thắng A thì đi tiếp xin B, người kia chờ rồi tới lượt</text>
<rect x="484" y="252" width="372" height="36" rx="6" fill="#CCFBF1"/>
<text x="670" y="268" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">Không thể có vòng tròn — mũi tên chờ luôn đi một chiều</text>
<text x="670" y="282" text-anchor="middle" font-size="10" fill="#0F766E">từ id nhỏ sang id lớn, mà một chiều thì không khép kín được.</text>
<rect x="20" y="310" width="860" height="78" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="40" y="334" font-size="12" font-weight="800" fill="#374151">Bốn điều kiện Coffman — deadlock cần ĐỦ CẢ BỐN, phá một cái là đủ</text>
<text x="40" y="356" font-size="11" fill="#57534E">① <tspan font-weight="700">Loại trừ lẫn nhau</tspan> (mutex vốn thế)  ② <tspan font-weight="700">Giữ và chờ</tspan> (ôm A rồi mới xin B)</text>
<text x="40" y="376" font-size="11" fill="#57534E">③ <tspan font-weight="700">Không giành được</tspan> (không ai cướp khoá của ai)  ④ <tspan font-weight="700" fill="#DC2626">Chờ vòng tròn</tspan> ← đây là cái dễ phá nhất trong code ứng dụng</text>
<rect x="20" y="400" width="860" height="64" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="40" y="424" font-size="12" font-weight="800" fill="#7C4A03">Quy tắc tổng quát: áp một THỨ TỰ TOÀN CỤC lên mọi khoá</text>
<text x="40" y="446" font-size="11" fill="#92400E">Mọi goroutine phải xin khoá theo cùng một thứ tự (id, địa chỉ bộ nhớ, tên tài nguyên — miễn là nhất quán).</text>
<text x="40" y="460" font-size="11" fill="#92400E">Khi đó đồ thị chờ trở thành một chiều, và <tspan font-weight="700">một đồ thị một chiều thì không thể có chu trình</tspan>.</text>
</svg>
</div>

## Go nói gì khi gặp deadlock

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
main.main()
	/.../main.go:31 +0x13c

goroutine 33 [sync.Mutex.Lock]:
internal/sync.(*Mutex).lockSlow(0xc000198020)      ← đang chờ mutex của B
main.transfer(0xc000198000, 0xc000198018, 0x64)    ← transfer(a, b, 100)
	/.../main.go:18 +0xa6

goroutine 34 [sync.Mutex.Lock]:
internal/sync.(*Mutex).lockSlow(0xc000198008)      ← đang chờ mutex của A
main.transfer(0xc000198018, 0xc000198000, 0xc8)    ← transfer(b, a, 200)
	/.../main.go:18 +0xa6
```

Chu trình nằm ngay trong các địa chỉ, chỉ cần đọc kỹ:

- `a` ở `0xc000198000`, mutex của nó ở offset 8 → `0xc000198008`
- `b` ở `0xc000198018`, mutex của nó → `0xc000198020`
- goroutine 33 nhận tham số `(0xc000198000, 0xc000198018)` = `(a, b)`, đang chờ `0xc000198020` = **mutex của b**
- goroutine 34 nhận `(b, a)`, đang chờ `0xc000198008` = **mutex của a**

> ⚠️ **Đừng trông cậy vào bộ phát hiện deadlock của Go.** Nó chỉ bắn khi **tuyệt đối mọi goroutine** đều đang ngủ. Trong một service thật luôn có goroutine khác đang chạy — HTTP server, ticker, health check — nên runtime **không coi đó là deadlock**. Hệ quả: hai request bị treo im lặng, không log, không panic, goroutine cứ dồn lên tới khi tràn bộ nhớ. Đây chính là lý do kiểu lỗi này rất khó truy.

## Cách sửa 1: thứ tự khoá toàn cục (khuyến nghị)

```go
// ✅ Luôn khoá account có id nhỏ trước — bất kể hướng chuyển tiền
func transfer(from, to *Account, amount int) error {
	if from.id == to.id {
		return errors.New("không thể chuyển cho chính mình") // tự khoá chính mình = deadlock
	}

	first, second := from, to
	if first.id > second.id {
		first, second = second, first
	}

	first.mu.Lock()
	defer first.mu.Unlock()
	second.mu.Lock()
	defer second.mu.Unlock()

	if from.balance < amount {
		return errors.New("số dư không đủ")
	}
	from.balance -= amount
	to.balance += amount
	return nil
}
```

Kết quả chạy hai phiên bản cạnh nhau, cùng kịch bản A→B 100đ và B→A 200đ chạy đồng thời:

```
A chuyển 100 cho B, CÙNG LÚC B chuyển 200 cho A (mỗi account 1 mutex riêng)

  transferBad    ❌ DEADLOCK — treo quá 2s, không lệnh nào hoàn tất
  transferGood   ✅ xong sau khi cả 2 lệnh chạy — A=1100 B=900
```

Số dư cuối đúng như kỳ vọng: `A = 1000 - 100 + 200 = 1100`, `B = 1000 + 100 - 200 = 900`, tổng vẫn là 2000.

> ⚠️ Nhớ chặn trường hợp `from.id == to.id`. `sync.Mutex` **không tái nhập được** — khoá hai lần cùng một mutex trong một goroutine là tự treo chính mình, và đây là lỗi rất dễ lọt vì "chuyển cho chính mình" nghe như một no-op vô hại.

Nếu không có id ổn định, dùng địa chỉ con trỏ làm thứ tự:

```go
import "unsafe"

if uintptr(unsafe.Pointer(first)) > uintptr(unsafe.Pointer(second)) {
	first, second = second, first
}
```

## Cách sửa 2: một khoá duy nhất cho cả nhóm

```go
// Đơn giản nhất, và thường là đủ: một mutex cho toàn bộ sổ cái.
// Không thể có chu trình khi chỉ có một khoá.
type Ledger struct {
	mu       sync.Mutex
	balances map[int]int
}

func (l *Ledger) Transfer(from, to, amount int) error {
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.balances[from] < amount {
		return errors.New("số dư không đủ")
	}
	l.balances[from] -= amount
	l.balances[to] += amount
	return nil
}
```

Đánh đổi: mọi giao dịch nối đuôi nhau qua một khoá. Nhưng hãy ước lượng trước khi chê — một critical section vài trăm nanosecond vẫn cho hàng triệu giao dịch mỗi giây, thừa sức cho phần lớn hệ thống. **Chỉ chia nhỏ khoá khi đã đo được rằng khoá chung là nút thắt thật.**

## Cách sửa 3: `TryLock` + rút lui và thử lại

```go
// Dùng khi KHÔNG thể áp thứ tự toàn cục (ví dụ khoá nằm ở nhiều hệ thống khác nhau).
func transferTry(from, to *Account, amount int) error {
	for attempt := 0; attempt < 50; attempt++ {
		from.mu.Lock()
		if to.mu.TryLock() { // không lấy được thì KHÔNG chờ
			from.balance -= amount
			to.balance += amount
			to.mu.Unlock()
			from.mu.Unlock()
			return nil
		}
		from.mu.Unlock() // nhả khoá đang giữ → phá điều kiện "giữ và chờ"
		// Lùi ngẫu nhiên, tránh hai goroutine cứ va nhau mãi theo cùng nhịp
		time.Sleep(time.Duration(rand.Intn(100)) * time.Microsecond)
	}
	return errors.New("không lấy được khoá sau 50 lần thử")
}
```

Cách này phá điều kiện **giữ-và-chờ** thay vì phá chu trình. Đổi lại là rủi ro **livelock** — hai goroutine liên tục nhường nhau mà không ai tiến được. Đó là lý do bắt buộc phải có **lùi ngẫu nhiên** và **giới hạn số lần thử**.

## So sánh ba cách

| Cách | Phá điều kiện nào | Ưu | Nhược |
|---|---|---|---|
| **Thứ tự toàn cục** | chờ vòng tròn | không treo, giữ được song song | cần một thứ tự ổn định cho mọi tài nguyên |
| **Một khoá chung** | chờ vòng tròn (chỉ còn 1 khoá) | đơn giản nhất, không thể sai | mọi thao tác nối đuôi nhau |
| **`TryLock` + retry** | giữ và chờ | dùng được khi không áp được thứ tự | phức tạp, có thể livelock, cần backoff |

## Checklist chống deadlock

```
□ Có chỗ nào giữ 2 khoá cùng lúc không? (nếu không — bạn an toàn)
□ Thứ tự khoá có PHỤ THUỘC THAM SỐ không? ← dấu hiệu nguy hiểm nhất
□ Đã áp một thứ tự toàn cục (id / địa chỉ / tên) cho mọi khoá chưa?
□ Đã chặn trường hợp hai tài nguyên trùng nhau chưa? (Mutex không tái nhập)
□ Có gọi hàm bên ngoài / callback trong lúc đang GIỮ khoá không?
  (hàm đó có thể xin thêm khoá khác mà bạn không biết)
□ Có gửi/nhận channel trong lúc đang giữ khoá không?
□ Đã viết test chạy hai chiều ngược nhau ĐỒNG THỜI, có timeout chưa?
□ Nhớ: go run -race KHÔNG phát hiện deadlock — nó chỉ tìm data race
```

> ★ Bài test duy nhất bắt được lớp lỗi này: chạy hai thao tác **ngược chiều nhau** thật sự đồng thời, bọc trong `select` với `time.After`. Deadlock không bao giờ xuất hiện trong test tuần tự, và cũng gần như không xuất hiện khi chỉ chạy một chiều.


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