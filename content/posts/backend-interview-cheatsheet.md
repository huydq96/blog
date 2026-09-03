+++
date = '2026-08-29T09:00:00+07:00'
draft = false
slug = 'backend-interview-cheatsheet'
title = 'Cẩm Nang Phỏng Vấn Backend: Những Khái Niệm Phải Biết'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["interview", "system-design", "backend", "golang", "database", "caching"]
description = 'Tổng hợp các khái niệm hay gặp khi phỏng vấn backend — REST, JWT, database, caching, message queue, khả năng mở rộng,...'
+++

Bài này tổng hợp từ một cẩm nang phỏng vấn backend dạng cheatsheet ([bài viết gốc](https://www.instagram.com/p/DcBi2nOE1hv)), có bổ sung thêm (đánh dấu **📌 Bổ sung**).

Vài khái niệm ở đây đã có bài riêng đào sâu trên blog này, tôi sẽ để link trực tiếp.

## I. Nền Tảng Backend (1–7)

**1. Kiến Trúc Client – Server**  
Client gửi request, server xử lý và trả response.

```
CLIENT (Trình duyệt / App) 
  ↓
INTERNET 
  ↓
SERVER (Logic + Dữ liệu)
```

**2. REST API**  
Kiểu kiến trúc xây dựng API trên HTTP, theo 6 ràng buộc của Roy Fielding: Stateless, Client–Server, Cacheable, Layered System, Uniform Interface, Code on Demand (tùy chọn).

📌 *Bổ sung*: "Uniform Interface" thật ra gồm 4 ràng buộc con, trong đó có HATEOAS (response trả kèm link tới hành động tiếp theo) — gần như không ai implement đủ. Nếu bị hỏi "API của bạn có thật sự RESTful không", câu trả lời trung thực thường là "REST-ish" — phần lớn API gọi là REST chỉ dùng đúng HTTP method + JSON, bỏ qua HATEOAS.

**3. Các Phương Thức HTTP**  
| Phương thức | Mục đích | Idempotent? |
|---|---|---|
| GET | Đọc dữ liệu | ✓ |
| POST | Tạo dữ liệu | ✗ |
| PUT | Thay thế toàn bộ dữ liệu | ✓ |
| PATCH | Sửa đổi một phần | ✗ (thường) |
| DELETE | Xóa dữ liệu | ✓ |

📌 *Bổ sung*: cột Idempotent — gọi 100 lần giống 100 lần gọi 1 lần hay không — theo đúng chuẩn HTTP, không phải quy ước tự đặt. Đây là lý do POST cần `Idempotency-Key` (mục 10) còn PUT/DELETE thì không.

**4. Mã Trạng Thái HTTP**  
1xx Thông tin · 2xx Thành công · 3xx Chuyển hướng · 4xx Lỗi client · 5xx Lỗi server.

Thường gặp: `200 OK`, `201 Created`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `500 Internal Server Error`.

**5. Stateless vs Stateful**  
| Stateless | Stateful |
|---|---|
| Server không lưu ngữ cảnh client | Server lưu session/ngữ cảnh client |
| Mỗi request tự đủ dữ liệu | Cần lấy dữ liệu từ server |
| Vd: JWT | Vd: Session ID, giỏ hàng lưu server |
| Dễ scale | Khó mở rộng hơn |

> ★ Hầu hết REST API đều Stateless.

**6. Authentication vs Authorization**
- **Authentication** — bạn là ai (đăng nhập).
- **Authorization** — bạn được làm gì (quyền hạn).

**7. JWT (JSON Web Token)**  
Token gọn nhẹ để truyền thông tin an toàn giữa các bên.

📌 Bài [JWT: Những Điều Cần Biết Trước Khi Chọn Nó](/posts/jwt/) đi sâu về khi nào nên/không nên dùng JWT thay vì session — câu trả lời không đơn giản như "JWT là stateless nên luôn tốt hơn".

## II. API & Thiết Kế Backend (8–14)

**8. API Gateway**  
Điểm vào duy nhất, định tuyến request, xử lý auth/rate limit/logging, ẩn kiến trúc nội bộ.

> ★ Dùng khi có nhiều service hoặc microservices.

**9. Rate Limiting**  
Giới hạn số request/client trong 1 khoảng thời gian để chống lạm dụng.

📌 *Bổ sung — 4 thuật toán hay bị hỏi trong phỏng vấn*:

| Thuật toán | Ý tưởng | Nhược điểm |
|---|---|---|
| Fixed Window | Đếm request trong khung cố định (vd mỗi phút) | Burst ở ranh giới 2 khung (2x limit trong khoảnh khắc) |
| Sliding Window Log | Lưu timestamp từng request, đếm trong cửa sổ trượt | Chính xác nhưng tốn bộ nhớ |
| Sliding Window Counter | Nội suy giữa 2 fixed window | Cân bằng độ chính xác/chi phí |
| **Token Bucket** | Bucket chứa token, mỗi request tiêu 1 token, refill theo thời gian | Cho phép burst có kiểm soát — phổ biến nhất |

```go
type TokenBucket struct {
    mu         sync.Mutex
    tokens     float64
    capacity   float64
    refillRate float64 // token/giây
    lastRefill time.Time
}

func (b *TokenBucket) Allow() bool {
    b.mu.Lock()
    defer b.mu.Unlock()
    now := time.Now()
    elapsed := now.Sub(b.lastRefill).Seconds()
    b.tokens = math.Min(b.capacity, b.tokens+elapsed*b.refillRate)
    b.lastRefill = now
    if b.tokens < 1 {
        return false
    }
    b.tokens--
    return true
}
```

**10. Idempotency**  
Nhiều request giống hệt chỉ tạo một hiệu ứng duy nhất. Quan trọng với POST/PATCH. Dùng header `Idempotency-Key`.

**Dùng cho:** thanh toán, tạo đơn hàng, cơ chế retry.

**11. Pagination**  
Chia kết quả lớn thành trang nhỏ: Page-based, Cursor-based, Offset-based.

```
GET /api/users?page=2&limit=10
{"data":[...],"page":2,"limit":10,"totalPages":25,"totalItems":248}
```

📌 *Bổ sung*: offset-based (`OFFSET 100000 LIMIT 10`) chậm dần khi offset lớn (DB vẫn phải quét qua hết số dòng bị bỏ), và bị "page drift" — nếu có insert/delete giữa 2 lần gọi, trang sau có thể lặp hoặc bỏ sót dòng. Cursor-based (`WHERE id > last_id LIMIT 10`) tránh cả 2 vấn đề, đánh đổi là không nhảy thẳng tới trang N được. Feed mạng xã hội luôn dùng cursor-based vì lý do này.

**12. API Versioning**  
Quản lý thay đổi không phá vỡ client cũ. Đặt version ở URL/Header/Query. Vd: `/api/v1/users`, `/api/v2/users`.

**13. Webhooks**  
Callback server-to-server, báo sự kiện real-time. Dùng trong thanh toán, CI/CD, CRM.

📌 *Bổ sung*: webhook nhận ở endpoint public — bất kỳ ai biết URL cũng POST giả mạo được. Luôn ký payload bằng HMAC (bên gửi ký với secret chung, header vd `X-Signature`), bên nhận verify chữ ký trước khi tin nội dung. Stripe, GitHub webhook đều làm vậy.

**14. CORS (Cross-Origin Resource Sharing)**  
Cơ chế cho phép request cross-origin an toàn, kiểm soát qua header: `Access-Control-Allow-Origin`, `-Methods`, `-Headers`.

📌 *Bổ sung*: với request "không đơn giản" (có custom header, method khác GET/POST/HEAD, hoặc `Content-Type: application/json`), browser tự gửi 1 request `OPTIONS` **preflight** trước để hỏi server "tôi được phép gửi request thật không" — server phải trả lời đúng header CORS cho cả preflight lẫn request thật. Quên xử lý `OPTIONS` là lỗi CORS hay gặp nhất khi mới làm API.

### Tóm tắt nhanh (8–14)  
API Gateway = cửa vào chung · Rate Limiting = chống lạm dụng · Idempotency = gọi lặp không sao · Versioning = đổi API không vỡ client cũ · Webhooks = báo tin real-time, nhớ ký HMAC · CORS = nhớ preflight

## III. Database (15–21)

**15. SQL vs NoSQL**

| SQL | NoSQL |
|---|---|
| Bảng, schema cố định | Key-Value/Document/Column/Graph, schema linh hoạt |
| ACID | BASE (eventually consistent) |
| Join tốt | Join hạn chế |
| Scale dọc | Scale ngang |
| MySQL, PostgreSQL | MongoDB, Cassandra, Redis |

**16. Indexing**  
Cấu trúc dữ liệu giúp DB tìm hàng nhanh hơn (thường là B-Tree). Loại: Primary, Unique, Composite, Full Text.

> ★ Index giúp READ nhanh nhưng làm WRITE chậm hơn (phải cập nhật cả index).

Chi tiết composite index, thứ tự cột ảnh hưởng ra sao: [MySQL Index: Composite vs Single](/posts/mysql-index-composite-vs-single/), [MySQL Indexing](/posts/mysql-indexing/).

**17. Transactions & ACID**
- **A**tomicity — tất cả hoặc không gì cả.
- **C**onsistency — DB luôn ở trạng thái hợp lệ.
- **I**solation — transaction đồng thời không ảnh hưởng nhau.
- **D**urability — commit xong thì không mất, dù có sự cố.

**18. Isolation Levels**

| Mức | Dirty Read | Non-Repeatable | Phantom |
|---|---|---|---|
| Read Uncommitted | ✓ | ✓ | ✓ |
| Read Committed | ✗ | ✓ | ✓ |
| Repeatable Read | ✗ | ✗ | ✓ |
| Serializable | ✗ | ✗ | ✗ |

📌 *Bổ sung*: bảng trên là định nghĩa theo chuẩn SQL. MySQL InnoDB lại là ngoại lệ nổi tiếng hay bị hỏi xoáy: `REPEATABLE READ` **mặc định** của InnoDB chặn được phần lớn phantom read nhờ cơ chế gap lock + MVCC snapshot — khác với "Repeatable Read" lý thuyết trong bảng. Đừng trả lời theo đúng bảng sách giáo khoa nếu được hỏi cụ thể về MySQL.

**19. Database Normalization**  
Giảm dư thừa, tăng toàn vẹn dữ liệu. 1NF (giá trị nguyên tử) → 2NF (+không phụ thuộc bộ phận) → 3NF (+không phụ thuộc bắc cầu) → BCNF (mạnh hơn 3NF).

**20. Replication**  
Sao chép dữ liệu sang server khác, tăng availability & read scaling.

```
PRIMARY (Master) → Replication → REPLICA 1 ↔ REPLICA 2
```

📌 *Bổ sung*: **Sync replication** — primary chờ replica xác nhận mới coi là ghi xong (an toàn hơn, chậm hơn, và nếu replica chết thì primary cũng kẹt theo tuỳ cấu hình). **Async replication** — primary ghi xong trả về ngay, replica đuổi theo sau (nhanh hơn, nhưng nếu primary chết trước khi replica kịp đồng bộ thì mất dữ liệu vừa ghi). Hầu hết hệ production dùng async hoặc semi-sync (chờ ít nhất 1 replica), full sync hiếm vì quá chậm.

**21. Sharding**  
Chia dữ liệu ra nhiều DB server theo shard key. Horizontal (theo hàng) / Vertical (theo cột).

```
Bảng User → Shard 1 (ID 1-1M) → Shard 2 (1M-2M) → Shard 3 (2M-3M)...
```

📌 *Bổ sung*: chọn sai shard key gây **hotspot** — vd shard theo `created_at` thì mọi write mới dồn vào đúng 1 shard "nóng nhất". Resharding (thêm shard mới, phân phối lại dữ liệu) là bài toán khó và tốn kém — cần nghĩ shard key kỹ từ đầu, không phải việc sửa sau dễ dàng.

> ★ SQL cho quan hệ dữ liệu phức tạp · NoSQL khi cần linh hoạt & scale lớn · Schema tốt + Index đúng = hiệu năng cao.

## IV. Caching & Hiệu Năng (22–28)

**22. Cache**  
Lưu dữ liệu hay dùng trong bộ nhớ để truy cập nhanh, giảm tải DB. Loại: In-Memory (Redis, Memcached), Browser Cache, CDN Cache.

> ★ Cache cải thiện READ. Luôn đặt TTL hợp lý. Đừng cache mọi thứ.

**23. Cache-Aside Pattern (Lazy Loading)**  
App tìm cache → có thì trả về → không thì lấy DB → lưu vào cache → trả về.

**24. Write-Through vs Write-Back (và Write-Around)**

| Write-Through | Write-Back | 📌 Write-Around |
|---|---|---|
| Ghi cache + DB cùng lúc | Ghi cache trước, DB sau (async) | Ghi thẳng DB, bỏ qua cache |
| Nhất quán hơn, chậm hơn | Nhanh hơn, rủi ro mất dữ liệu nếu cache chết trước khi flush | Tránh cache đầy dữ liệu ít đọc lại |
| Banking, Orders | Logs, Analytics | Dữ liệu ghi nhiều đọc ít (vd log) |

**25. Cache Eviction Policies**  
LRU (Least Recently Used) · LFU (Least Frequently Used) · FIFO · TTL · Random.

Ví dụ LRU: capacity=3, `[1,2,3]` → truy cập `4` → `[2,3,4]` (1 bị loại).

**26. Redis**  
In-memory data store: Strings, Hashes, Lists, Sets, Sorted Sets. Dùng cho caching, session, rate limiting, leaderboard. Có Persistence, Pub/Sub, Replication, TTL.

```
SET key value
GET key
DEL key
EXPIRE key 60
HSET user:1 name abhi
LPUSH list 1
LRANGE list 0 -1
```

**27. CDN (Content Delivery Network)**  
Phân phối nội dung qua edge server gần người dùng, giảm tải origin.

```
User → CDN EDGE SERVER → (có cache thì trả luôn, không thì lấy) → ORIGIN SERVER
```

**28. Connection Pooling**  
Tái sử dụng kết nối DB đã mở, tránh chi phí tạo kết nối mới liên tục, xử lý concurrency tốt hơn.

### Tóm tắt nhanh (22–28)
Cache-Aside = lazy load · Write-Through = an toàn, chậm · Write-Back = nhanh, rủi ro · Write-Around = tránh rác cache · Eviction: LRU/LFU/FIFO/TTL · Redis = kho in-memory đa năng · CDN = đưa nội dung gần user · Connection Pool = tái dùng kết nối.

## V. Bất Đồng Bộ & Hệ Thống Phân Tán (29–35)

**29. Message Queue**  
Giao tiếp bất đồng bộ, tách producer/consumer. Vd: RabbitMQ, SQS, Kafka.

```
PRODUCER → MESSAGE QUEUE → CONSUMER
```

Lợi ích: Async Processing, Load Leveling, Fault Tolerance, Scalability.

**30–34. Kafka, Producer/Consumer, Consumer Group, Pub/Sub, Event-Driven**

Nhóm này blog đã có 2 bài đào sâu, tôi không lặp lại ở đây: [Apache Kafka P1](/posts/apache-kafka-p1/) (khái niệm cốt lõi: Topic, Partition, Broker, Producer, Consumer) và [Apache Kafka P2](/posts/apache-kafka-p2/).

Tóm rất gọn cho phỏng vấn:
- **Consumer Group**: mỗi partition chỉ được đúng 1 consumer *trong cùng 1 group* xử lý tại một thời điểm — nhưng 2 group khác nhau đọc độc lập, mỗi group đều thấy toàn bộ message (đây là chỗ hay bị hiểu nhầm thành "chỉ 1 consumer duy nhất đọc được message").
- **Pub/Sub**: publisher → topic → nhiều subscriber, mô hình one-to-many, hợp cho notification/chat/live update.
- **Event-Driven Architecture**: các service giao tiếp qua event thay vì gọi trực tiếp — loosely coupled hơn, nhưng đổi lại khó trace luồng xử lý hơn khi debug (trade-off ít cheatsheet nào nhắc).

**35. Synchronous vs Asynchronous**

| Sync | Async |
|---|---|
| Request → chờ → response | Request → tiếp tục → response sau |
| Blocking | Non-blocking |
| Throughput thấp hơn | Throughput cao hơn |
| Tight coupling | Loose coupling |
| Gọi REST API | Message Queue, Kafka |

> ★ Hiểu rõ đánh đổi consistency vs availability · Biết khi nào Queue vs Kafka · Tập trung vào fault tolerance và idempotency trong hệ async (message có thể đến 2 lần — consumer phải tự dedupe hoặc xử lý idempotent).

## VI. Khả Năng Mở Rộng & Độ Tin Cậy (36–43)

**36. Load Balancer**  
Phân phối traffic qua nhiều server. L4 (Transport, nhanh, ít thông minh) vs L7 (Application, đọc được HTTP header/path để route thông minh hơn).

```
Người dùng → LOAD BALANCER → SERVER 1/2/3 (kèm Health Check)
```

**37. Horizontal vs Vertical Scaling**

| Horizontal | Vertical |
|---|---|
| Thêm máy (scale out) | Nâng cấu hình máy hiện có (scale up) |
| Xử lý nhiều traffic hơn | Giới hạn bởi phần cứng |
| Tiết kiệm chi phí hơn ở quy mô lớn | Đơn giản, không cần phân tán dữ liệu |

**38. Microservices**  
Chia app thành service nhỏ độc lập, mỗi service 1 DB riêng, giao tiếp qua API.

**39. Service Discovery**  
Tự tìm và giao tiếp với service, không hard-code địa chỉ. Công cụ: Eureka, Consul, etcd, Zookeeper.

**40. Circuit Breaker**  
Ngăn lỗi lan truyền bằng cách chặn gọi tới service đang lỗi.

📌 *Sửa*: cheatsheet gốc gợi ý Hystrix — thư viện này Netflix đã ngừng phát triển tích cực từ 2018, cộng đồng Java hiện khuyến nghị **Resilience4j** thay thế. Nếu bị hỏi "dùng thư viện nào cho circuit breaker", nhắc Hystrix mà không kèm ghi chú "đã deprecated" là một điểm trừ nhỏ nhưng dễ bị soi ở phỏng vấn senior.

<div class="zoomable-diagram" style="max-width:700px;margin:2rem auto;">
<svg viewBox="0 0 700 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="cb-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="cb-arrow-red" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/>
</marker>
<marker id="cb-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="cb-arrow-green" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#16A34A"/>
</marker>
</defs>
<rect x="0" y="0" width="700" height="460" fill="#FFFDF7" rx="16"/>
<rect x="60" y="40" width="220" height="90" rx="16" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#cb-shadow)"/>
<text x="170" y="75" text-anchor="middle" font-size="15" font-weight="800" fill="#134E4A">CLOSED</text>
<text x="170" y="95" text-anchor="middle" font-size="10.5" fill="#0F766E">bình thường —</text>
<text x="170" y="110" text-anchor="middle" font-size="10.5" fill="#0F766E">request đi qua bình thường</text>
<rect x="420" y="40" width="220" height="90" rx="16" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="3" filter="url(#cb-shadow)"/>
<text x="530" y="75" text-anchor="middle" font-size="15" font-weight="800" fill="#9F1239">OPEN</text>
<text x="530" y="95" text-anchor="middle" font-size="10.5" fill="#BE123C">chặn hết request —</text>
<text x="530" y="110" text-anchor="middle" font-size="10.5" fill="#BE123C">fail fast, không gọi service lỗi</text>
<rect x="240" y="320" width="220" height="90" rx="16" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#cb-shadow)"/>
<text x="350" y="355" text-anchor="middle" font-size="15" font-weight="800" fill="#7C4A03">HALF-OPEN</text>
<text x="350" y="375" text-anchor="middle" font-size="10.5" fill="#92400E">cho vài request đi qua</text>
<text x="350" y="390" text-anchor="middle" font-size="10.5" fill="#92400E">để thử service đã hồi chưa</text>
<line x1="282" y1="85" x2="418" y2="85" stroke="#DC2626" stroke-width="2.5" marker-end="url(#cb-arrow-red)"/>
<text x="350" y="72" text-anchor="middle" font-size="10.5" fill="#DC2626">lỗi vượt ngưỡng</text>
<path d="M500,132 Q470,240 405,322" fill="none" stroke="#57534E" stroke-width="2.5" marker-end="url(#cb-arrow-grey)"/>
<text x="500" y="235" font-size="10.5" fill="#57534E">hết timeout,</text>
<text x="500" y="250" font-size="10.5" fill="#57534E">thử lại</text>
<path d="M285,322 Q225,220 178,132" fill="none" stroke="#16A34A" stroke-width="2.5" marker-end="url(#cb-arrow-green)"/>
<text x="95" y="235" font-size="10.5" fill="#16A34A">✓ request</text>
<text x="95" y="250" font-size="10.5" fill="#16A34A">test thành công</text>
<path d="M420,325 Q565,225 545,132" fill="none" stroke="#DC2626" stroke-width="2.5" marker-end="url(#cb-arrow-red)"/>
<text x="565" y="235" font-size="10.5" fill="#DC2626">✗ request</text>
<text x="565" y="250" font-size="10.5" fill="#DC2626">test thất bại</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Circuit Breaker: CLOSED (bình thường) → OPEN (fail fast khi lỗi vượt ngưỡng) → HALF-OPEN (thử lại) → về CLOSED hoặc OPEN tuỳ kết quả.</p>

```go
type State int
const (StateClosed State = iota; StateOpen; StateHalfOpen)

type CircuitBreaker struct {
    mu           sync.Mutex
    state        State
    failCount    int
    threshold    int
    openedAt     time.Time
    resetTimeout time.Duration
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    if cb.state == StateOpen {
        if time.Since(cb.openedAt) > cb.resetTimeout {
            cb.state = StateHalfOpen // hết timeout, cho thử lại
        } else {
            cb.mu.Unlock()
            return errors.New("circuit open: fail fast")
        }
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()
    if err != nil {
        cb.failCount++
        if cb.state == StateHalfOpen || cb.failCount >= cb.threshold {
            cb.state = StateOpen
            cb.openedAt = time.Now()
        }
        return err
    }
    cb.state = StateClosed
    cb.failCount = 0
    return nil
}
```

**41. Retry & Exponential Backoff**  
Retry request lỗi tạm thời, tăng dần thời gian chờ giữa các lần: 1s → 2s → 4s → 8s...

📌 *Bổ sung — jitter*: nếu 1000 client cùng retry sau đúng cùng 1 khoảng thời gian, chúng dội vào server cùng lúc — gọi là **thundering herd**, có thể khiến service vừa hồi phục sập lại ngay. Thêm **jitter** (random hoá backoff trong 1 khoảng) để rải các lần retry ra:

```go
func backoffWithJitter(attempt int) time.Duration {
    base := time.Second * time.Duration(1<<attempt) // 1,2,4,8...
    jitter := time.Duration(rand.Int63n(int64(base) / 2))
    return base/2 + jitter // full jitter trong [base/2, base]
}
```

**42. Health Checks**

| Loại | Mô tả |
|---|---|
| Liveness Probe | Service còn sống không — lỗi thì restart |
| Readiness Probe | Service sẵn sàng nhận traffic chưa — lỗi thì gỡ khỏi LB (không restart) |

**43. Observability**
3 trụ cột: **Metrics, Logs, Traces**.

```
METRICS / LOGS / TRACES → nền tảng observability → insights & alerts → kỹ sư
```

### Tóm tắt nhanh (36–43)
Load Balancer = rải traffic · Horizontal Scaling = thêm máy · Microservices = tách service nhỏ · Service Discovery = tự tìm service · Circuit Breaker = chặn lỗi lan (nhớ jitter khi retry) · Health Check = Liveness khác Readiness · Observability = Metrics+Logs+Traces.

## VII. Khái Niệm Nâng Cao (44–50)

**44. Distributed Lock**  
Đảm bảo chỉ 1 process truy cập tài nguyên dùng chung, chống race condition trong hệ phân tán. Thường dùng Redis, ZooKeeper.

📌 *Bổ sung*: khoá phân tán bằng Redis đơn giản (`SET key val NX EX ttl`) có lỗ hổng thời gian thực nổi tiếng (process A giữ khoá bị treo lâu hơn TTL, khoá tự hết hạn, process B lấy được khoá, rồi A tỉnh dậy và vẫn tưởng mình đang giữ khoá — 2 process cùng sửa tài nguyên). Martin Kleppmann (tác giả *Designing Data-Intensive Applications*) đã viết bài phân tích nổi tiếng chỉ ra thuật toán Redlock của Redis không an toàn tuyệt đối trong mọi trường hợp: [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html). Đây là câu hỏi hay của phỏng vấn senior/staff — biết cái lỗ hổng này quan trọng hơn biết cú pháp `SET NX`.

**45. CAP Theorem**  
Hệ phân tán chỉ đạt 2/3: **C**onsistency, **A**vailability, **P**artition Tolerance.

📌 *Sửa — hiểu lầm phổ biến nhất của CAP*: P không thật sự là "lựa chọn thứ 3" ngang hàng C và A. Trong hệ phân tán thật (nhiều node, qua mạng), network partition **sẽ xảy ra**, không phải chuyện có chọn dùng hay không. Lựa chọn thực sự chỉ nằm ở: **khi partition xảy ra, hệ thống chọn Consistency (từ chối trả lời nếu không chắc đúng) hay Availability (vẫn trả lời, chấp nhận có thể sai/cũ)?**

<div class="zoomable-diagram" style="max-width:700px;margin:2rem auto;">
<svg viewBox="0 0 700 580" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="cap-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
</defs>
<rect x="0" y="0" width="700" height="580" fill="#FFFDF7" rx="16"/>
<polygon points="350,50 120,380 580,380" fill="none" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="6 5"/>
<line x1="350" y1="50" x2="120" y2="380" stroke="#FF6B6B" stroke-width="4"/>
<text x="250" y="205" font-size="11" font-weight="700" fill="#9F1239">lựa chọn THẬT SỰ</text>
<text x="250" y="221" font-size="11" font-weight="700" fill="#9F1239">khi có partition</text>
<circle cx="350" cy="50" r="42" fill="#FFE9EC" stroke="#BE123C" stroke-width="2.5" filter="url(#cap-shadow)"/>
<text x="350" y="45" text-anchor="middle" font-size="16" font-weight="800" fill="#9F1239">C</text>
<text x="350" y="60" text-anchor="middle" font-size="8.5" fill="#9F1239">Consistency</text>
<circle cx="120" cy="380" r="42" fill="#E6FBF9" stroke="#0F766E" stroke-width="2.5" filter="url(#cap-shadow)"/>
<text x="120" y="375" text-anchor="middle" font-size="16" font-weight="800" fill="#134E4A">A</text>
<text x="120" y="390" text-anchor="middle" font-size="8.5" fill="#134E4A">Availability</text>
<circle cx="580" cy="380" r="42" fill="#FFF3DB" stroke="#92400E" stroke-width="2.5" filter="url(#cap-shadow)"/>
<text x="580" y="373" text-anchor="middle" font-size="16" font-weight="800" fill="#7C4A03">P</text>
<text x="580" y="386" text-anchor="middle" font-size="8" fill="#7C4A03">Partition</text>
<text x="580" y="397" text-anchor="middle" font-size="8" fill="#7C4A03">Tolerance</text>
<text x="580" y="445" text-anchor="middle" font-size="10" fill="#57534E">gần như bắt buộc trong</text>
<text x="580" y="460" text-anchor="middle" font-size="10" fill="#57534E">hệ phân tán thật —</text>
<text x="580" y="475" text-anchor="middle" font-size="10" fill="#57534E">không phải "lựa chọn thứ 3"</text>
<rect x="70" y="500" width="250" height="60" rx="10" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2"/>
<text x="195" y="524" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">CP — ưu tiên đúng</text>
<text x="195" y="542" text-anchor="middle" font-size="10" fill="#0F766E">MongoDB, HBase, ZooKeeper</text>
<rect x="360" y="500" width="250" height="60" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="485" y="524" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">AP — ưu tiên phản hồi</text>
<text x="485" y="542" text-anchor="middle" font-size="10" fill="#92400E">Cassandra, DynamoDB, Riak</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">P gần như luôn phải chấp nhận trong hệ phân tán thật — lựa chọn thật sự nằm ở cạnh C–A khi partition xảy ra.</p>

**46. Consistency Models**
- **Strong Consistency** — đọc luôn thấy ghi mới nhất.
- **Eventual Consistency** — nhất quán dần theo thời gian.
- **Causal Consistency** — giữ đúng thứ tự các thao tác liên quan nhân quả.

**47. Database Deadlocks**  
2+ transaction chờ nhau vô thời hạn. DB tự phát hiện, huỷ 1 transaction.

**Phòng tránh:** transaction ngắn gọn · truy cập bảng theo cùng thứ tự · index hợp lý · tránh giữ lock lâu.

**48. Distributed Transactions**  
Transaction trải nhiều service/DB, cần giữ ACID xuyên suốt. Giao thức: **2PC** (Two Phase Commit), **Saga Pattern**, **TCC** (Try-Confirm-Cancel).

**49. Eventual Consistency**  
Hệ thống chưa nhất quán ngay nhưng sẽ nhất quán dần. Đổi lấy availability & partition tolerance tốt hơn. Vd: DynamoDB, Cassandra, Riak.


### Mẹo Phỏng Vấn (áp dụng cho mọi khái niệm)
Luôn đề cập: định nghĩa → vì sao cần nó → cách hoạt động → ưu/nhược điểm → ví dụ thực tế. Với các khái niệm có "cạm bẫy" (Circuit Breaker, CAP, Distributed Lock, Isolation Level) — biết đúng cái bẫy thường ghi điểm hơn là thuộc lòng định nghĩa.

## 🔥 Ôn Tập Nhanh (10 Ý Quan Trọng Nhất)

| # | Khái niệm | Tóm tắt |
|---|---|---|
| 1 | **REST** | Stateless, cacheable, đúng HTTP method & status code |
| 2 | **JWT** | Chỉ encode, không mã hoá — đọc kỹ trước khi dùng thay session |
| 3 | **Rate Limiting** | Token Bucket phổ biến nhất, cho phép burst có kiểm soát |
| 4 | **Index** | Tăng tốc READ, làm chậm WRITE — đừng index thừa |
| 5 | **Isolation Level** | MySQL InnoDB Repeatable Read ≠ định nghĩa SQL chuẩn |
| 6 | **Cache** | Cache-Aside phổ biến nhất, luôn đặt TTL |
| 7 | **Kafka/MQ** | Tách producer-consumer, chịu lỗi tốt hơn gọi trực tiếp |
| 8 | **Circuit Breaker** | Closed → Open → Half-Open, dùng Resilience4j (không phải Hystrix) |
| 9 | **CAP** | P gần như bắt buộc — lựa chọn thật là C vs A khi có partition |
| 10 | **Distributed Lock** | Redlock có lỗ hổng đã biết — đọc bài Kleppmann trước khi tự tin dùng |

## Đọc thêm

- [JWT: Những Điều Cần Biết Trước Khi Chọn Nó](/posts/jwt/)
- [Apache Kafka P1](/posts/apache-kafka-p1/) · [Apache Kafka P2](/posts/apache-kafka-p2/)
- [MySQL Indexing](/posts/mysql-indexing/) · [MySQL Index: Composite vs Single](/posts/mysql-index-composite-vs-single/)
- [System Design Resources](/posts/system-design-resources/) — kho link tổng hợp sâu hơn từng khái niệm
- Martin Kleppmann — [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
