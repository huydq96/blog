+++
date = '2025-05-26T09:40:49+07:00'
draft = false
title = 'Go Interview P1'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "interview"]
description = 'Một số câu hỏi phỏng vấn về Golang P1'
+++

# Giải thích khái niệm về channels trong Go. Khi nào và tại sao bạn sẽ sử dụng chúng ?

## Khái niệm và ứng dụng của Channels trong Go

Channels là một trong những cơ chế concurrency mạnh mẽ nhất trong Go, được thiết kế để quản lý giao tiếp và đồng bộ hóa giữa các goroutine. Chúng đóng vai trò trung tâm trong việc xây dựng các hệ thống phân tán, pipeline xử lý dữ liệu, và các mô hình đồng thời phức tạp.

Channel được biểu diễn bằng struct hchan trong runtime Go:

```go
type hchan struct {
    qcount   uint           // Số phần tử hiện có
    dataqsiz uint           // Kích thước buffer
    buf      unsafe.Pointer // Vùng nhớ buffer
    sendx    uint           // Vị trí gửi tiếp theo
    recvx    uint           // Vị trí nhận tiếp theo
    lock     mutex          // Mutex đảm bảo thread-safe
}
```

Cấu trúc này sử dụng hàng đợi vòng (circular queue) để quản lý dữ liệu và mutex để đồng bộ truy cập

## Hoạt động của Channels

### Gửi/nhận dữ liệu

- **Gửi dữ liệu**: **ch <- value** block goroutine nếu channel đầy (buffered) hoặc chưa có receiver (unbuffered)
- **Nhận dữ liệu**: **value := <-ch** block goroutine nếu channel rỗng
- **Đóng channel**: **close(ch)** đánh dấu không còn dữ liệu gửi đi. Một kênh đã đóng vẫn cho phép đọc dữ liệu từ nó nhưng không thể truyền dữ liệu đến nó (gây ra panic).

### Select statement

Cơ chế `select` cho phép chờ nhiều channel cùng lúc, xử lý trường hợp đầu tiên sẵn sàng

```go
select {
case msg := <-ch1:
    fmt.Println("Nhận từ ch1:", msg)
case ch2 <- "data":
    fmt.Println("Gửi thành công vào ch2")
default:
    fmt.Println("Không có hoạt động nào sẵn sàng")
}
```

`select` thường được dùng để triển khai timeout hoặc non-blocking operations.

- Xem thêm: [Go Channels](https://blog.huydang.dev/posts/go-concurrency/#channels)

## Khi nào sử dụng Channels ?

Channels hợp cho các bài toán yêu cầu giao tiếp phức tạp giữa các goroutine, phân tải công việc, hoặc xử lý luồng dữ liệu theo pipeline.

- Ưu điểm:
    - An toàn đồng thời: Loại bỏ race conditions thông qua cơ chế truyền thông điệp thay vì chia sẻ bộ nhớ.
    - Linh hoạt: Hỗ trợ đa dạng mô hình (pipeline, pub/sub, worker pools).
    - Tích hợp sâu với runtime: Tối ưu hiệu năng nhờ quản lý trực tiếp bởi Go scheduler.
- Hạn chế:
    - Deadlock tiềm ẩn: Nếu không đóng channel hoặc xử lý block không đúng cách.
    - Overhead cho simple tasks: Đôi khi mutex đơn giản hơn cho các tác vụ đơn lẻ.

- So Sánh Với Mutex

| Tiêu Chí | Channels | Mutex |
| -------- | ------- | ------- |
| Phạm Vi | Giao tiếp giữa goroutines | Bảo vệ truy cập tài nguyên |
| Độ Phức Tạp	| 	Cao (cần quản lý vòng đời) | Thấp |
| Phù Hợp |	Pipeline, phân tác vụ phức tạp | Critical section đơn giản |

### Các mô hình ứng dụng

1. Đồng bộ hóa Goroutines

Channels giải quyết vấn đề race conditions bằng cách thay thế mutex truyền thống. Ví dụ, đảm bảo một goroutine chỉ chạy sau khi goroutine khác hoàn thành

```go
done := make(chan bool)
go func() {
    // Xử lý tác vụ
    done <- true
}()
<-done // Block cho đến khi nhận tín hiệu
```

2. Pipeline xử lý dữ liệu

Kết hợp nhiều giai đoạn xử lý, mỗi giai đoạn là một goroutine độc lập

```go
// Giai đoạn 1: Tạo số nguyên
generator := func() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)
    }()
    return ch
}

// Giai đoạn 2: Bình phương
squares := func(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Kết hợp pipeline
for n := range squares(generator()) {
    fmt.Println(n) // 0, 1, 4, 9, 16
}
```

3. Mô hình Fan-Out/Fan-In

```go
jobs := make(chan int, 100)
results := make(chan int)

// Khởi động 3 worker
for w := 1; w <= 3; w++ {
    go func(id int) {
        for j := range jobs {
            results <- j * 2
        }
    }(w)
}

// Gửi 5 jobs
for j := 1; j <= 5; j++ {
    jobs <- j
}
close(jobs)

// Thu thập kết quả
for a := 1; a <= 5; a++ {
    fmt.Println(<-results)
}

// 2
// 4
// 6
// 8
// 10
```

4. Semaphore và Rate limiting

```go
sem := make(chan struct{}, 3) // Giới hạn 3 goroutines
for i := 0; i < 10; i++ {
    sem <- struct{}{}
    go func(id int) {
        defer func() { <-sem }()
        // Xử lý tác vụ
    }(i)
}
```


# Làm thế nào để quản lý goroutines và tránh memory leaks ?

## Nguyên nhân gây rò rỉ Goroutine

1. Channel blocking vĩnh viễn

- Unbuffered channel yêu cầu cả sender và receiver sẵn sàng. Nếu một bên không tồn tại, goroutine sẽ bị chặn mãi mãi.
- Buffered channel đầy: Khi buffer đã đầy, sender bị chặn cho đến khi có receiver.

2. Vòng lặp vô hạn thiếu điều kiện thoát

```go
go func() {
    for { // Không có điều kiện thoát
        fmt.Println("Running...")
    }
}()
```

3. Sử dụng Context không đúng cách

Không hủy context khi goroutine không còn cần thiết:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // Nếu quên gọi cancel()
go func(ctx context.Context) {
    <-ctx.Done() // Không bao giờ nhận tín hiệu
}(ctx)
```

## Phòng tránh rò rỉ Goroutines

1. Sử Dụng Context hủy tác vụ

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

go func(ctx context.Context) {
    select {
    case <-ctx.Done():
        return // Thoát khi hết thời gian
    case data := <-ch:
        process(data)
    }
}(ctx)
```

2. Đảm bảo đóng Channel đúng cách

- Sender đóng channel sau khi gửi xong:
```go
ch := make(chan int)
go func() {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}()
```

3. Sử dụng `select` với `default` để tránh block

```go
select {
case msg := <-ch:
    handle(msg)
default:
    // Không block nếu không có dữ liệu
}
```

4. Giới hạn số lượng Goroutine với Semaphore

- Sử dụng buffered channel làm semaphore:

```go
sem := make(chan struct{}, 10) // Tối đa 10 goroutines
for i := 0; i < 100; i++ {
    sem <- struct{}{}
    go func(id int) {
        defer func() { <-sem }()
        processTask(id)
    }(i)
}
```

5. Tránh Capture biến vòng lặp

```go
for i := 0; i < 5; i++ {
    i := i // Tạo bản sao
    go func() {
        fmt.Println(i)
    }()
}
```

# Làm thế nào để dừng một goroutine đang chạy ?

## Sử dụng Channel tín hiệu dừng

Tạo một channel để gửi tín hiệu dừng đến goroutine. Goroutine sẽ kiểm tra channel này định kỳ và thoát khi nhận được tín hiệu:

```go
quit := make(chan bool)
go func() {
    for {
        select {
        case <-quit:
            return
        default:
            // Thực hiện công việc
        }
    }
}()
// Gửi tín hiệu dừng
quit <- true
```

## Sử dụng `Context.WithCancel`

Context cung cấp cơ chế hủy bỏ lan truyền, cho phép hủy nhiều goroutine cùng lúc:

```go
ctx, cancel := context.WithCancel(context.Background())
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // Thực hiện công việc
        }
    }
}(ctx)

// Hủy tất cả goroutine sử dụng context này
cancel()
```

## Đóng Channel đầu vào

Khi goroutine đang đọc từ một channel, việc đóng channel sẽ khiến vòng lặp range tự động thoát:

```go
jobs := make(chan int)
go func() {
    for job := range jobs {
        // Xử lý job
    }
}()

// Đóng channel để dừng goroutine
close(jobs)
```

## Sử dụng Timeout

Kết hợp `select` với `time.After` để tự động hủy sau khoảng thời gian nhất định:

```go
go func() {
    for {
        select {
        case <-time.After(5 * time.Second):
            return
        default:
            // Thực hiện công việc
        }
    }
}()
```

## So sánh các phương pháp

| Phương pháp | Ưu điểm | Nhược điểm |
| -------- | ------- | ------- |
| Channel tín hiệu dừng | Đơn giản, dễ triển khai | Khó quản lý với nhiều goroutine |
| Context | Hủy lan truyền, tích hợp sẵn | Cần truyền context qua nhiều tầng |
| Đóng Channel | Tự động thoát vòng lặp range | Chỉ áp dụng khi đọc từ channel |
| Timeout | Tự động dừng sau thời gian cố định | Không linh hoạt cho các tác vụ dài |

## Ví dụ kết hợp `context` và `WaitGroup`

```go
func worker(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Nhận tín hiệu dừng")
            return
        default:
            fmt.Println("Đang làm việc...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background())
    
    wg.Add(1)
    go worker(ctx, &wg)
    
    time.Sleep(3 * time.Second)
    cancel() // Gửi tín hiệu dừng
    
    wg.Wait()
    fmt.Println("Tất cả goroutine đã dừng")
}
```


# Giải thích về interfaces trong Go và cách chúng khác với interfaces trong các ngôn ngữ OOP khác

**Interfaces** là một trong những tính năng cốt lõi của Go, thiết kế để xử lý tính đa hình (**polymorphism**) theo cách tối giản và linh hoạt.

## Định nghĩa và cơ chế hoạt động

Một interface trong Go định nghĩa tập hợp các **method signatures** (chữ ký phương thức) mà một kiểu dữ liệu phải triển khai. Ví dụ:

```go
type Writer interface {
    Write([]byte) (int, error)
}
```

- **Implicit Implementation**: Bất kỳ kiểu nào (struct, kiểu tự định nghĩa, v.v.) có phương thức **Write** sẽ tự động triển khai interface **Writer** mà không cần khai báo tường minh.
- **Structural Typing**: Interface xác định hành vi thông qua cấu trúc phương thức, không qua tên kiểu.

### Empty Interface (interface{})

- Đại diện cho mọi kiểu dữ liệu vì không yêu cầu phương thức nào:

```go
var any interface{} = "hello"  // Hợp lệ
any = 42                       // Hợp lệ
```

- Thường dùng cho xử lý dữ liệu động (JSON decoding, generic containers).

## So sánh với Interface trong ngôn ngữ OOP

1. Implicit vs. Explicit Implementation


| Đặc Điểm | Go | Java |
| -------- | ------- | ------- |
| Khai báo triển khai | Không cần (implicit) | Bắt buộc (implements) |
| Liên kết | Tách biệt interface và triển khai | Interface và class phụ thuộc trực tiếp |
| Ví dụ | type T struct{}; func (T) F() {} | class T implements I { ... } |

- Ưu điểm Go: Giảm coupling, cho phép thêm interface cho kiểu có sẵn.
- Nhược điểm: Khó theo dõi triển khai interface trong mã nguồn lớn.

2. Phương thức mặc định (Default Methods)

- **Java**: Từ Java 8, interface có thể chứa phương thức mặc định:

```java
interface Animal {
    default void sound() { System.out.println("Default sound"); }
}
```

- **Go**: Không hỗ trợ. Mọi phương thức trong interface đều trừu tượng.

3. Kế Thừa và Composition

- Go: Cho phép **embedding interfaces** để kết hợp phương thức:

```go
type ReadWriter interface {
    Reader  // Nhúng interface Reader
    Writer  // Nhúng interface Writer
}
```

- Java: Kế thừa interface qua `extends`

## Ưu và nhược điểm

- Ưu điểm của Go:
    - **Linh hoạt**: Thêm interface mới không cần sửa code hiện có.
    - **Giảm boilerplate**: Không cần từ khóa implements.
    - **Duck typing an toàn**: Kiểm tra phương thức lúc biên dịch.

- Nhược điểm:
    - **Khó debug**: Khó xác định kiểu nào triển khai interface trong codebase lớn.
    - **Thiếu phương thức mặc định**: Không thể chia sẻ logic chung giữa các triển khai.

## Kết luận

Interfaces trong Go tập trung vào **hành vi** thay vì **kế thừa**. Khác biệt chính so với các ngôn ngữ OOP truyền thống nằm ở:
- **Implicit implementation** giúp code linh hoạt, ít phụ thuộc.
- **Structural typing** cho phép đa hình mà không cần hệ thống phân cấp phức tạp.
- **Đơn giản** với việc chỉ định nghĩa method signatures.