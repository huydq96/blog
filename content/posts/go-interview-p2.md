+++
date = '2026-09-04T10:00:00+07:00'
draft = false
title = 'Go Interview P2'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "interview"]
description = 'Một số câu hỏi phỏng vấn về Golang P2 — các loại bộ nhớ, Garbage Collector, memory leak, error wrapping'
+++

# Các loại bộ nhớ trong một chương trình Go là gì?

## Tổng quan các vùng nhớ

| Vùng nhớ | Chứa gì | Ai quản lý | Vòng đời |
|---|---|---|---|
| **Stack** | Local variable, tham số hàm, địa chỉ return — mỗi goroutine có 1 stack riêng | Go runtime, tự động | Theo từng lời gọi hàm — cấp khi gọi, thu khi return |
| **Heap** | Object "escape" khỏi function scope | Garbage Collector | Không xác định trước — sống tới khi unreachable |
| **Data / BSS segment** | Biến global đã khởi tạo (data) và chưa khởi tạo/zero value (BSS) | OS loader lúc khởi động | Suốt vòng đời chương trình |
| **Text / Code segment** | Machine code đã compile, read-only | OS | Suốt vòng đời chương trình, không đổi |

## Stack

- Mỗi goroutine có **1 stack riêng**, khác hẳn OS thread: kích thước ban đầu chỉ ~2KB (so với 1-8MB cố định của OS thread stack) — đây là lý do Go tạo hàng trăm nghìn goroutine vẫn nhẹ.
- Stack **grow động**: khi gần đầy, runtime cấp 1 vùng lớn hơn, copy toàn bộ nội dung sang và cập nhật lại mọi pointer trỏ vào stack cũ ("stack copying"). Mặc định tối đa 1GB (`debug.SetMaxStack`).
- Cấp phát/giải phóng cực rẻ — chỉ là di chuyển stack pointer, GC không cần quan tâm tới object nằm ở đây.

## Heap

- Bộ cấp phát heap của Go lấy cảm hứng từ **TCMalloc** (Thread-Caching Malloc): object được nhóm theo **size class** cố định để giảm phân mảnh. Mỗi P (processor) có cache riêng (`mcache`) cấp phát nhanh không cần lock; hết cache mới xin thêm từ `mcentral`/`mheap` (dùng chung, cần lock).
- Object lớn (> 32KB) cấp phát thẳng từ `mheap`, không qua size class.
- Đây là vùng duy nhất GC phải quét và thu hồi.

## Global / package-level (Data & BSS)

- `var x = 5` (có giá trị khởi tạo) nằm ở data segment; `var x int` (chưa gán, zero value) thuộc BSS về mặt khái niệm — Go không tách bạch rạch ròi như C, trình biên dịch/linker tự quyết định chỗ đặt.
- Cấp phát **một lần** khi chương trình khởi động (trước cả `main()`, qua `init()`), tồn tại suốt đời chương trình — đây cũng là lý do biến global luôn là GC root.

> ★ Hiểu đúng "biến này nằm ở đâu" là nền cho gần như mọi câu hỏi tối ưu hiệu năng phía sau: escape analysis quyết định stack hay heap, heap là thứ GC phải lo, global luôn là root.

# Garbage Collector (GC) trong Go là gì, hoạt động thế nào, và cần lưu ý gì để tối ưu?

## GC là gì?

Garbage Collector là cơ chế **tự động quản lý bộ nhớ** — tự tìm và giải phóng những vùng nhớ đã cấp phát (heap) mà chương trình không còn tham chiếu tới (unreachable) nữa, để lập trình viên không phải gọi `free()` thủ công như C/C++.

Go dùng thuật toán **concurrent, tri-color, mark-and-sweep, non-generational, non-compacting**:
- **Concurrent**: GC chạy song song với chương trình (mutator), không dừng hẳn toàn bộ ứng dụng để dọn rác.
- **Tri-color mark-and-sweep**: đánh dấu (mark) object còn sống bằng 3 màu, rồi quét (sweep) thu hồi phần không được đánh dấu.
- **Non-generational**: không chia heap theo "thế hệ" (young/old) như JVM — mọi object được quét như nhau mỗi chu kỳ.
- **Non-compacting**: không di chuyển object để dồn bộ nhớ liền khối — vùng nhớ trống bị phân mảnh, quản lý qua free list.

## Cách hoạt động — Tri-color mark-and-sweep

1. **Root**: điểm bắt đầu quét — biến global, biến trên stack của từng goroutine đang chạy. Mọi root được tô **grey**.
2. **Mark**: lấy 1 object grey, quét mọi con trỏ nó tham chiếu tới → tô các object đó thành grey, chính object vừa quét chuyển sang **black** (đã xử lý xong). Lặp lại tới khi hết object grey.
3. Object nào vẫn còn **white** sau khi hết grey = không có đường nào từ root tới được nó = rác.
4. **Sweep**: thu hồi toàn bộ vùng nhớ của các object white.

```
white (chưa xét) → grey (đã thấy, chưa quét con) → black (đã quét xong, chắc chắn sống)
Sau khi hết grey: white còn lại = rác → sweep thu hồi
```

**Write barrier**: vì mark chạy *đồng thời* với chương trình, nếu mutator đổi con trỏ giữa lúc GC đang quét (ví dụ gán 1 object black trỏ tới 1 object white), GC có thể bỏ sót và thu hồi nhầm object đang sống. Go dùng **hybrid write barrier** (từ Go 1.8) để chặn đúng tình huống này mà không cần dừng hẳn chương trình.

**Stop-The-World (STW)**: GC của Go chỉ dừng chương trình ở 2 thời điểm rất ngắn (thường dưới 1ms): lúc bắt đầu (bật write barrier) và lúc kết thúc mark phase (mark termination). Phần lớn thời gian mark + sweep chạy concurrent.

**GOGC** (mặc định `100`): GC chu kỳ tiếp theo được kích hoạt khi heap tăng thêm 100% so với heap còn sống sau lần GC trước (tức tăng gấp đôi). Giảm `GOGC` → GC chạy thường xuyên hơn, tốn CPU hơn nhưng RAM đỉnh thấp hơn. Từ Go 1.19 có thêm **`GOMEMLIMIT`** — đặt giới hạn cứng cho tổng bộ nhớ, hữu ích khi chạy trong container có memory limit (tránh bị OOM-killed).

## Stack vs Heap — Escape Analysis

Go tự quyết định 1 biến nằm ở **stack** hay **heap** ngay lúc compile, qua **escape analysis** — không phải cứ khai báo bằng `new`/`&` là chắc chắn lên heap.

- Biến ở **stack**: tự dọn khi function return, **không tốn chi phí GC gì cả** — rẻ nhất có thể.
- Biến **escape lên heap**: GC phải track và thu hồi khi không còn reachable.

```go
func noEscape() int {
    x := 42
    return x // x ở stack — value được copy ra ngoài, x gốc mất khi hàm return
}

func escape() *int {
    x := 42
    return &x // x escape lên heap — con trỏ sống lâu hơn function, stack không đủ an toàn
}
```

Kiểm tra thật escape analysis quyết định gì (không đoán):

```bash
go build -gcflags="-m" ./...
```

Nguyên nhân phổ biến khiến biến escape: trả về pointer trỏ tới biến local, closure giữ biến rồi bản thân closure "thoát" ra khỏi hàm (return closure, hoặc truyền vào goroutine), gán biến vào `interface{}`, hoặc kích thước không xác định lúc compile (slice tăng trưởng động).

## Lưu ý khi dùng biến để tối ưu GC

### Biến global (package scope)

Biến global sống **suốt vòng đời chương trình** → luôn là 1 GC root → GC luôn phải quét qua nó ở **mọi** chu kỳ, không có ngoại lệ.

Nguy hiểm hơn: nếu biến global trỏ tới dữ liệu lớn (map/slice cache) mà không có cơ chế dọn, GC sẽ **không bao giờ thu hồi được** — vì object đó luôn reachable từ root, đúng theo định nghĩa. Đây không phải "memory leak" kiểu C (không có con trỏ mồ côi), mà là leak kiểu Go: dữ liệu sống mãi vì vẫn có người tham chiếu, dù logic không cần nữa (xem thêm câu hỏi Memory Leak phía dưới).

### Biến local (function scope)

Biến local **không escape** → nằm ở stack → gần như miễn phí với GC. Nguyên tắc tối ưu: giữ vòng đời biến càng ngắn, càng cục bộ càng tốt — đừng "nâng" biến lên phạm vi rộng hơn mức cần thiết (đừng biến local thành field của struct, đừng đưa lên global) nếu không thực sự cần chia sẻ.

## Lưu ý với Goroutine để tối ưu GC

Đây là chỗ hay bị bỏ qua nhất: **mỗi goroutine đang tồn tại (kể cả đang block) là 1 GC root.**

- Goroutine leak (channel không ai nhận, context không cancel...) không chỉ là leak logic. Goroutine đó vẫn "sống" trong runtime, nên **stack của nó và mọi object nó đang giữ tham chiếu đều là reachable** → GC nhìn vào thấy hoàn toàn hợp lệ, không thu hồi gì cả. GC "làm đúng việc của nó", chỉ là chương trình đang giữ một thứ lẽ ra phải chết.
- Đây là lý do goroutine leak thường **khó phát hiện qua GC/heap profile thông thường** — object không "mất tích", nó vẫn nằm trong tay 1 goroutine đang treo vô ích.
- Số lượng goroutine sống càng lớn (worker pool không giới hạn, spawn vô tội vạ) → càng nhiều GC root cần quét mỗi chu kỳ mark → tốn CPU hơn dù GC vẫn chạy concurrent.

Khuyến nghị:
- Luôn có đường thoát rõ ràng cho goroutine (context/channel signal).
- Giới hạn số goroutine đồng thời bằng semaphore/worker pool kích thước cố định, không spawn không giới hạn theo số request.
- Theo dõi bằng `pprof`: `/debug/pprof/goroutine` (số lượng & stack trace từng goroutine) và `/debug/pprof/heap` (ai đang giữ bộ nhớ) — số goroutine tăng đều không giảm gần như luôn là dấu hiệu leak.

## Giảm áp lực lên GC (GC pressure) trong hot path

| Kỹ thuật | Vì sao giúp GC |
|---|---|
| `sync.Pool` tái dùng object tạm (buffer, struct lớn) | Giảm số lần allocate mới → giảm số object GC phải track |
| Preallocate `make([]T, 0, n)` khi biết trước kích thước | Tránh nhiều lần realloc + copy khi `append` vượt cap |
| Giữ biến ở scope hẹp nhất có thể | Tăng khả năng biến ở stack thay vì escape lên heap |
| Batch xử lý thay vì tạo object nhỏ lẻ trong vòng lặp lớn | Giảm tổng số allocation trong hot path |
| Đặt `GOGC`/`GOMEMLIMIT` phù hợp môi trường chạy | Cân bằng CPU dùng cho GC và RAM đỉnh, tránh OOM trong container |

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf) // trả lại pool để tái sử dụng, không cấp phát mới mỗi lần gọi
    }()
    buf.Write(data)
    // ...
}
```

## Cách quan sát GC thực tế

- `GODEBUG=gctrace=1 ./myapp` — in log mỗi lần GC chạy: thời gian, heap size trước/sau, % CPU dành cho GC.
- `runtime.ReadMemStats(&m)` — đọc số liệu heap, số lần GC, pause time ngay trong code.
- `pprof` heap profile (`go tool pprof http://localhost:6060/debug/pprof/heap`) — tìm chính xác dòng code nào allocate nhiều nhất.

> ★ Tóm tắt 1 câu: GC của Go không thu hồi thứ "không dùng nữa" theo ý định của bạn — nó thu hồi thứ **không còn reachable từ root**. Biến global và goroutine đang sống đều là root, nên kiểm soát vòng đời của 2 thứ đó chính là cách tối ưu GC hiệu quả nhất, hơn hẳn việc cố "giúp" GC bằng micro-optimization.

# Memory Leak trong Go — vì sao vẫn xảy ra dù có GC, và những dạng thường gặp?

## Vì sao có GC mà vẫn leak được?

GC chỉ thu hồi những gì **không còn reachable từ root** (xem câu hỏi GC phía trên). Nếu code vẫn giữ 1 tham chiếu tới object không cần dùng nữa — dù vô tình — GC coi nó đang sống và không bao giờ đụng tới. "Leak" trong Go, khác C, không phải quên `free()`, mà là **quên "buông" một tham chiếu**.

## Các dạng leak thường gặp

### 1. Goroutine leak

Channel không ai nhận/gửi, context không cancel khiến goroutine treo vĩnh viễn. Nguy hiểm gấp đôi: vừa leak chính goroutine, vừa leak mọi object nó đang giữ tham chiếu (stack + heap object) vì goroutine đang sống luôn là GC root.

### 2. Global cache không có eviction

```go
var userCache = map[string]*User{}

func GetUser(id string) *User {
    if u, ok := userCache[id]; ok { return u }
    u := fetchFromDB(id)
    userCache[id] = u // không bao giờ xoá → phình vô hạn
    return u
}
```

Fix: TTL/LRU eviction, hoặc giới hạn kích thước map cố định.

### 3. `time.Ticker` / `time.Timer` không `Stop()`

```go
func leak() {
    ticker := time.NewTicker(time.Second)
    go func() {
        for range ticker.C { doWork() }
    }() // không Stop() → ticker + goroutine sống mãi, dù hàm leak() đã return
}
```

`time.NewTicker` giữ 1 goroutine nội bộ trong runtime để bắn tick — quên `Stop()` là leak chắc chắn, không phụ thuộc GC.

### 4. Slice giữ nguyên backing array lớn

```go
func extractHeader(data []byte) []byte {
    return data[:10] // slice mới NHƯNG dùng chung backing array với data
    // nếu data là 50MB, cả 50MB đó không được GC thu hồi dù chỉ cần 10 byte
}
```

Fix: copy ra slice độc lập nếu phần giữ lại nhỏ hơn nhiều so với phần gốc.

```go
func extractHeaderFixed(data []byte) []byte {
    header := make([]byte, 10)
    copy(header, data[:10])
    return header // backing array gốc của data giờ có thể được GC thu hồi
}
```

### 5. `context.WithCancel` quên gọi `cancel()`

Mỗi context con giữ 1 tham chiếu ngược tới cha (để propagate cancel); quên gọi `cancel()` khiến context không được gỡ khỏi cây, giữ luôn goroutine/resource liên quan. Đây là lý do quy tắc "luôn `defer cancel()`" quan trọng hơn nhiều người nghĩ.

### 6. Closure giữ biến lớn không cần thiết

```go
func handler(bigData []byte) func() {
    result := process(bigData) // chỉ cần result
    return func() {
        fmt.Println(result)
        // closure này vẫn "nhìn thấy" bigData trong scope —
        // compiler có thể giữ cả bigData sống nếu không cẩn thận
    }
}
```

## Cách phát hiện leak trong thực tế

- So sánh heap profile theo thời gian: `go tool pprof -diff_base=heap1.prof heap2.prof`.
- Theo dõi `runtime.NumGoroutine()` và RSS theo thời gian — tăng đều không giảm là dấu hiệu rõ nhất.
- `net/http/pprof` bật sẵn trong service production để lấy profile bất cứ lúc nào không cần restart.

> ★ Câu trả lời phỏng vấn tốt: không nói "Go có GC nên không leak" — nói đúng: "GC giải quyết đúng 1 bài toán là thu hồi bộ nhớ unreachable; leak trong Go luôn là do code vô tình giữ 1 reference sống lâu hơn cần thiết — goroutine, cache, ticker, hoặc slice là 4 chỗ hay gặp nhất."

# Error wrapping trong Go — `%w`, `errors.Is`, `errors.As` hoạt động thế nào?

## Trước Go 1.13

Error chỉ là 1 giá trị implement interface `error` (có method `Error() string`). Muốn giữ "nguyên nhân gốc" khi bọc thêm ngữ cảnh, phải tự làm thủ công (custom struct chứa field lỗi gốc) — không có chuẩn chung, mỗi codebase tự nghĩ 1 kiểu.

## Go 1.13: wrapping chuẩn hoá

`fmt.Errorf` thêm verb `%w`: tạo ra error mới, nhưng vẫn giữ liên kết tới error gốc qua method ngầm `Unwrap() error`.

```go
var ErrNotFound = errors.New("not found")

func findUser(id string) error {
    return fmt.Errorf("findUser %s: %w", id, ErrNotFound)
}
```

## 3 hàm cốt lõi trong package `errors`

| Hàm | Làm gì | Dùng khi |
|---|---|---|
| `errors.Unwrap(err)` | Lấy error bị bọc bên trong (1 lớp) | Ít dùng trực tiếp — `Is`/`As` dùng ngầm bên dưới |
| `errors.Is(err, target)` | Duyệt cả **chuỗi wrap** (theo Unwrap liên tiếp), true nếu gặp `target` | So sánh với **sentinel error** đã biết trước (`sql.ErrNoRows`, `io.EOF`, `ErrNotFound`...) |
| `errors.As(err, &target)` | Duyệt chuỗi wrap, tìm error nào cùng **type** với `target`, gán vào nếu thấy | Cần lấy field/method riêng của 1 **custom error type** cụ thể |

```go
err := findUser("123")

// Is: so sánh GIÁ TRỊ, xuyên qua toàn bộ chuỗi wrap
fmt.Println(errors.Is(err, ErrNotFound)) // true — dù message đã đổi khác hẳn

// As: tìm đúng TYPE, lấy được field riêng
type ValidationError struct {
    Field string
    Err   error
}
func (e *ValidationError) Error() string { return fmt.Sprintf("field %s: %v", e.Field, e.Err) }
func (e *ValidationError) Unwrap() error  { return e.Err } // bắt buộc để Is/As đi xuyên qua được

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field)
}
```

## `%v` vs `%w` — khác biệt sống còn

- `%v`: format error thành **string phẳng**, mất hoàn toàn liên kết — `errors.Is`/`errors.As` phía sau sẽ **không** xuyên qua được.
- `%w`: giữ liên kết qua `Unwrap()` — `errors.Is`/`errors.As` xuyên qua bình thường.

Dùng nhầm `%v` khi lẽ ra cần `%w` là lỗi rất phổ biến — kiểm tra lỗi bằng `errors.Is` phía trên cứ trả `false` mà không hiểu tại sao, vì chuỗi wrap đã bị cắt đứt ngay từ chỗ dùng `%v`.

## Go 1.20+: wrap nhiều error cùng lúc

Từ Go 1.20, `fmt.Errorf` cho phép nhiều `%w` trong cùng 1 lời gọi (tạo thành cây thay vì chuỗi), và có thêm `errors.Join(err1, err2, ...)` để gộp nhiều error độc lập thành 1 error duy nhất — `errors.Is`/`errors.As` vẫn duyệt được qua tất cả nhánh.

```go
err := errors.Join(errValidation, errTimeout)
fmt.Println(errors.Is(err, errTimeout)) // true
```

## Best practice

- Sentinel error (`var ErrX = errors.New(...)`) → check bằng `errors.Is`.
- Custom error type cần lấy thêm dữ liệu → check bằng `errors.As`.
- **Đừng bao giờ** so sánh lỗi bằng `err.Error() == "..."` hoặc `strings.Contains` — message có thể đổi bất cứ lúc nào, cực kỳ dễ vỡ.
- Luôn wrap kèm ngữ cảnh khi truyền lỗi lên tầng trên: `fmt.Errorf("tên hàm: %w", err)` — giúp trace được lỗi xảy ra ở đâu mà không mất nguyên nhân gốc.

# Xem thêm

<div style="margin:1.5rem 0;text-align:left;">
<a href="/pages/golang-interview-prep.html" target="_blank" rel="noopener" style="">📋 Senior Golang Interview Prepare<thêm/a>
</div>
