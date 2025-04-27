+++
date = '2025-04-26T11:00:00+07:00'
draft = false
title = 'Go Context'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "coding"]
description = 'Context Package trong Go: ngữ nghĩa và cách sử dụng'
+++


# Context Package
Được giới thiệu từ năm 2014, Context package vẫn là một thành phần then chốt trong lập trình Go, cho phép quản lý hiệu quả dữ liệu theo phạm vi yêu cầu, thời hạn và tín hiệu hủy bỏ. Khi hệ sinh thái Go tiếp tục phát triển, việc hiểu rõ ngữ nghĩa của Context package là vô cùng quan trọng để xây dựng phần mềm đáng tin cậy và dễ bảo trì.

## Giới thiệu
Go cung cấp từ khóa `go` để tạo **goroutine**, nhưng lại không có từ khóa hoặc hỗ trợ trực tiếp nào để kết thúc goroutine. Trong một dịch vụ thực tế, khả năng đặt thời gian chờ và kết thúc goroutine là rất quan trọng để duy trì sự ổn định và hoạt động của dịch vụ.

**Context package** được Go team cung cấp như một giải pháp cho vấn đề này, đã được viết và giới thiệu bởi Sameer Ajmani vào năm 2014 tại hội nghị Gotham Go. Bạn có thể tham khảo thêm về chủ đề này:

- Slide Deck: https://talks.golang.org/2014/gotham-context.slide#1
- Blog Post: https://blog.golang.org/context

# Sử dụng Context

## Tạo context khi có request đến server

Thời điểm tốt nhất để tạo Context là càng sớm càng tốt trong quá trình xử lý một request hoặc task. Làm việc với Context sớm trong chu kỳ phát triển sẽ buộc bạn phải thiết kế các API nhận Context làm tham số đầu tiên. Ngay cả khi bạn không chắc chắn 100% một hàm có cần Context hay không, việc xóa Context khỏi một vài hàm dễ hơn nhiều so với việc cố gắng thêm Context sau này.

Từ phiên bản **Go 1.7**, `http.Request` đã chứa một context [(đọc thêm)](https://website-name.com).

Theo quy ước trong Go, tên biến `ctx` thường được sử dụng cho tất cả các giá trị context. Vì `Context` là một interface, không nên sử dụng con trỏ. Mọi hàm chấp nhận Context nên nhận bản sao riêng của giá trị interface.

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

## Các lệnh gọi đi server bên ngoài nên chấp nhận một Context

Ý tưởng đằng sau điều này là các lệnh gọi cấp cao hơn cần cho các lệnh gọi cấp thấp hơn biết họ sẵn sàng chờ bao lâu. Một ví dụ tuyệt vời về điều này là với gói http và những thay đổi trong phiên bản 1.7 đối với phương thức `Do` để tôn trọng timeouts trên một request.

```go
// Sample code that implements a web request with a context
// that is used to timeout the request if it takes too long.
package main

import (
	"context"
	"io"
	"log"
	"net/http"
	"os"
	"time"
)

func main() {

	// Create a new request.
	req, err := http.NewRequest("GET", "REQUEST_URL", nil)
	if err != nil {
		log.Println("ERROR:", err)
		return
	}

	// Create a context with a timeout of 30 seconds.
	ctx, cancel := context.WithTimeout(req.Context(), 30*time.Second)
	defer cancel()

	// Bind the new context into the request.
	req = req.WithContext(ctx)

	// Make the request and return any error.
    // Do will handle the context level timeout.
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		log.Println("ERROR:", err)
		return
	}

	// Close the response body on the return.
	defer resp.Body.Close()

	// Write the response to stdout.
	io.Copy(os.Stdout, resp.Body)
}
```

## Không lưu trữ Context bên trong một kiểu struct

Truyền Context một cách tường minh cho mỗi hàm cần nó.

Về cơ bản, bất kỳ hàm nào thực hiện I/O đều nên chấp nhận một giá trị Context làm tham số đầu tiên và tôn trọng mọi timeout hoặc deadline được cấu hình bởi người gọi.

```go
// ví dụ một số method của net.Resolver
LookupHost(ctx context.Context, host string) (addrs []string, err error)
LookupIPAddr(ctx context.Context, host string) ([]net.IPAddr, error)
LookupIP(ctx context.Context, network string, host string) ([]net.IP, error)
LookupNetIP(ctx context.Context, network string, host string) ([]netip.Addr, error)
```

## Chuỗi các lệnh gọi hàm phải lan truyền Context giữa chúng

Đây là một quy tắc quan trọng vì Context dựa trên request hoặc task. Bạn muốn Context và mọi thay đổi được thực hiện đối với nó trong quá trình xử lý request hoặc task được lan truyền và tôn trọng.

```go
// Handler
func (u *User) List(ctx context.Context, w http.ResponseWriter, r *http.Request, params map[string]string) error {
     ctx, span := trace.StartSpan(ctx, "handlers.User.List")
     defer span.End()

     users, err := user.List(ctx, u.db)
     if err != nil {
         return err
     }

     return web.Respond(ctx, w, users, http.StatusOK)
}

// Repository
func List(ctx context.Context, db *sqlx.DB) ([]User, error) {
	ctx, span := trace.StartSpan(ctx, "internal.user.List")
	defer span.End()

	users := []User{}
	const q = `SELECT * FROM users`

	if err := db.SelectContext(ctx, &users, q); err != nil {
		return nil, errors.Wrap(err, "selecting users")
	}

	return users, nil
}

```

Nếu một giá trị Context cấp cao nhất mới được tạo, bất kỳ thông tin Context hiện có nào từ một lệnh gọi cấp cao hơn liên quan đến request sẽ bị mất.

## Thay thế Context bằng cách sử dụng WithCancel, WithDeadline, WithTimeout hoặc WithValue

Vì mỗi hàm có thể thêm/sửa Context cho nhu cầu cụ thể của chúng, và những thay đổi đó không ảnh hưởng đến bất kỳ hàm nào đã được gọi trước đó, Context sử dụng ngữ nghĩa giá trị. Điều này có nghĩa là bất kỳ thay đổi nào đối với giá trị Context sẽ tạo ra một giá trị Context mới, sau đó được truyền đi.

```go
// Sample code to show how to use the WithTimeout function
// of the Context package.
package main

import (
	"context"
	"fmt"
	"time"
)

type data struct {
	UserID string
}

func main() {

	// Set a duration.
	duration := 150 * time.Millisecond

	// Create a context that is both manually cancellable and will signal
	// a cancel at the specified duration.
	ctx, cancel := context.WithTimeout(context.Background(), duration)
	defer cancel()

	// Create a channel to received a signal that work is done.
	ch := make(chan data, 1)

	// Ask the goroutine to do some work for us.
	go func() {

		// Simulate work.
		time.Sleep(50 * time.Millisecond)

		// Report the work is done.
		ch <- data{"123"}
	}()

	// Wait for the work to finish. If it takes too long move on.
	select {
	case d := <-ch:
		fmt.Println("work complete", d)

	case <-ctx.Done():
		fmt.Println("work cancelled")
	}
}
```

Điều cực kỳ quan trọng là bất kỳ hàm **cancel** nào được trả về từ hàm **With** phải được thực thi trước khi hàm đó trả về. Đây là lý do tại sao quy tắc là sử dụng từ khóa `defer` ngay sau lệnh gọi **With**. Không làm như vậy sẽ gây ra rò rỉ bộ nhớ trong chương trình.

## Khi một Context bị hủy, tất cả các Context được sinh ra từ nó cũng bị hủy

```go
// Sample program to show when a Context is canceled, all Contexts
// derived from it are also canceled.
package main

import (
	"context"
	"fmt"
	"sync"
)

// Need a key type.
type myKey int

// Need a key value.
const key myKey = 0

func main() {

	// Create a Context that can be cancelled.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Use the Waitgroup for orchestration.
	var wg sync.WaitGroup
	wg.Add(10)

	// Create ten goroutines that will derive a Context from
	// the one created above.
	for i := 0; i < 10; i++ {
		go func(id int) {
			defer wg.Done()

			// Derive a new Context for this goroutine from the Context
			// owned by the main function.
			ctx := context.WithValue(ctx, key, id)

			// Wait until the Context is cancelled.
			<-ctx.Done()
			fmt.Println("Cancelled:", id)
		}(i)
	}

	// Cancel the Context and any derived Context's as well.
	cancel()
	wg.Wait()
}
```

Khi hàm `cancel` được gọi, tất cả mười goroutine sẽ được bỏ chặn và in ra rằng chúng đã bị hủy.

Điều này cũng cho thấy rằng cùng một Context có thể được truyền cho các hàm chạy trong các goroutine khác nhau. Một Context an toàn để sử dụng đồng thời bởi nhiều goroutine.

## Không truyền Context là nil, ngay cả khi một hàm cho phép

Truyền một context `TODO` nếu bạn không chắc chắn về việc sử dụng Context nào.


Bạn biết mình cần một Context nhưng không chắc nó sẽ đến từ đâu. Bạn biết mình không có trách nhiệm tạo Context cấp cao nhất, vì vậy việc sử dụng hàm `Background` là không thể. Bạn cần một Context cấp cao nhất tạm thời cho đến khi tìm ra Context thực tế đến từ đâu. Đây là lúc bạn nên sử dụng hàm `TODO` thay vì hàm `Background`.

## Chỉ sử dụng context values cho dữ liệu có phạm vi request di chuyển qua các process và API, không dùng để truyền các tham số tùy chọn cho hàm

Không sử dụng giá trị Context để truyền dữ liệu vào một hàm khi dữ liệu đó là bắt buộc để hàm thực thi code của nó thành công. Nói cách khác, một hàm phải có khả năng thực thi logic của nó với một giá trị Context trống. Trong trường hợp một hàm yêu cầu thông tin phải có trong Context, nếu thông tin đó bị thiếu, chương trình sẽ bị lỗi và báo hiệu ứng dụng ngừng hoạt động.

Theo nguyên tắc chung, bạn nên tuân theo thứ tự này khi di chuyển dữ liệu trong chương trình của mình:
- Truyền dữ liệu làm tham số hàm: Đây là cách rõ ràng nhất để di chuyển dữ liệu trong chương trình mà không cần ẩn nó.
- Truyền dữ liệu thông qua receiver: Nếu hàm cần dữ liệu không thể thay đổi signature, thì hãy sử dụng một phương thức và truyền dữ liệu thông qua receiver.

## Bài viết tham khảo
- https://www.ardanlabs.com/blog/2019/09/context-package-semantics-in-go.html
- https://goatspeed.substack.com/p/putting-context-into-context