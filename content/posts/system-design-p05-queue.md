+++
date = '2026-10-23T09:00:00+07:00'
draft = false
slug = 'system-design-p05-queue'
title = 'System Design P05: Hiểu Đúng Về Queue — Redis Queue Hay RabbitMQ?'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "redis", "rabbitmq", "queue", "golang"]
description = 'Queue rỗng không có nghĩa việc đã xong. Đo thật 4 cách triển khai queue: Redis List mất 20 job khi worker crash, 3 cách còn lại không mất — nhưng cả 4 đều gửi trùng.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, code và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên Redis 7.4.2 và RabbitMQ 3.13.7 chạy local.*

# Câu chuyện mở đầu

Một team cần hỗ trợ kiểm tra lỗi: **khách đặt hàng thành công nhưng chờ mãi không nhận được email xác nhận.**

Team nhìn vào queue thì **không còn job nào chờ**. Worker vẫn đang chạy. Nhìn qua chẳng thấy việc nào bị ùn lại cả.

Hướng điều tra đầu tiên là dịch vụ gửi email — có thể bên đó đang chậm? Nhưng kiểm tra thì phía email **chưa nhận được yêu cầu gửi** cho đơn hàng ấy.

Lần ngược về ứng dụng: job đã được tạo chưa? Đã. Đã giao cho worker chưa? Đã. Xem tiếp vòng đời của worker thì mới thấy — **nó đã crash trước bước gửi email**. Tiến trình team đang nhìn thấy là worker vừa được **khởi động lại**.

Vậy job cũ đi đâu?

Xem đoạn nhận job: ứng dụng dùng **Redis List**, worker `LPOP` job ra để xử lý. Thao tác `LPOP` **đã lấy job khỏi hàng chờ**. Tìm chỗ giữ thông tin job đang xử lý thì... ứng dụng chưa có.

> **Worker chết, thông tin về việc nó đang làm cũng mất theo. Worker mới lên nhìn vào một hàng đợi rỗng, chẳng có gì để nhận lại.**

Và đây là câu quan trọng nhất của cả bài:

> **"Queue rỗng" chỉ cho biết không còn job đang CHỜ. Còn email đã được gửi hay chưa thì hệ thống không theo dõi được.**

---

# Đo thật: `LPOP` mất bao nhiêu job?

Tôi dựng lại đúng tình huống đó để đo. 2.000 job gửi email, 20 worker, **mỗi worker crash khi đang giữ job thứ 31** — tức đúng 20 job bị crash giữa chừng. Sau phục hồi, phải gửi đúng 2.000 email.

| Cách triển khai | Gửi được | Kết quả |
|---|---:|---|
| **① Redis List + `LPOP`** | **1.980/2.000** | **❌ MẤT 20 job** |
| ② Redis List + `LMOVE` (có danh sách đang-xử-lý) | 2.000/2.000 | ✅ phục hồi 20 job |
| ③ Redis Streams + consumer group | 2.000/2.000 | ✅ PEL giữ 20 job |
| ④ RabbitMQ + manual ACK | 2.000/2.000 | ✅ tự requeue |

**Mất đúng 20 job — bằng đúng số worker crash.** Không có lỗi nào được ghi lại ở bất cứ đâu: Redis khoẻ, queue rỗng, worker mới đang chạy bình thường. Chỉ có 20 khách hàng không nhận được email.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="q1-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="q1-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="q1-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="460" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Job biến mất ở đâu — và vì sao monitoring không thấy</text>
<rect x="24" y="44" width="852" height="178" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#q1-sh)"/>
<text x="44" y="68" font-size="13" font-weight="800" fill="#9F1239">❌ Redis List + LPOP — không có chỗ giữ job đang xử lý</text>
<rect x="48" y="82" width="120" height="58" rx="8" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.8"/>
<text x="108" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Redis List</text>
<rect x="60" y="112" width="96" height="18" rx="4" fill="#FFE9EC"/>
<text x="108" y="125" text-anchor="middle" font-size="9" fill="#BE123C">job 1..2000</text>
<rect x="224" y="82" width="132" height="58" rx="8" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.8"/>
<text x="290" y="100" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Worker</text>
<text x="290" y="116" text-anchor="middle" font-size="10" font-family="monospace" fill="#BE123C">LPOP → job 847</text>
<text x="290" y="132" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">đang giữ job 847</text>
<rect x="412" y="82" width="120" height="58" rx="8" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<text x="472" y="104" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">💀 CRASH</text>
<text x="472" y="122" text-anchor="middle" font-size="10" fill="#BE123C">trước khi gửi email</text>
<rect x="588" y="82" width="132" height="58" rx="8" fill="#FFFFFF" stroke="#A8A29E" stroke-width="1.8" stroke-dasharray="5 4"/>
<text x="654" y="100" text-anchor="middle" font-size="11" font-weight="700" fill="#78716C">Worker MỚI</text>
<text x="654" y="116" text-anchor="middle" font-size="10" fill="#78716C">nhìn vào queue</text>
<text x="654" y="132" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">không thấy gì</text>
<rect x="760" y="82" width="96" height="58" rx="8" fill="#DC2626"/>
<text x="808" y="106" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">job 847</text>
<text x="808" y="124" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">BỐC HƠI</text>
<line x1="172" y1="111" x2="220" y2="111" stroke="#DC2626" stroke-width="2" marker-end="url(#q1-a)"/>
<line x1="360" y1="111" x2="408" y2="111" stroke="#DC2626" stroke-width="2" marker-end="url(#q1-a)"/>
<line x1="536" y1="111" x2="584" y2="111" stroke="#DC2626" stroke-width="2" marker-end="url(#q1-a)"/>
<line x1="724" y1="111" x2="756" y2="111" stroke="#DC2626" stroke-width="2" marker-end="url(#q1-a)"/>
<rect x="48" y="154" width="808" height="56" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="68" y="176" font-size="12" font-weight="800" fill="#9F1239">Vì sao mọi chỉ số đều XANH</text>
<text x="68" y="198" font-size="11" fill="#BE123C">Redis khoẻ · queue rỗng (= "không còn việc chờ") · worker mới đang chạy · không log lỗi nào. Chỉ có 20 khách không nhận email.</text>
<rect x="24" y="238" width="852" height="124" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#q1-sh)"/>
<text x="44" y="262" font-size="13" font-weight="800" fill="#134E4A">✅ Cách đúng — PHẢI có chỗ giữ job đã giao nhưng chưa xác nhận</text>
<rect x="48" y="276" width="120" height="70" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="108" y="296" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Queue CHỜ</text>
<rect x="60" y="304" width="96" height="16" rx="4" fill="#CCFBF1"/>
<rect x="60" y="324" width="96" height="16" rx="4" fill="#CCFBF1"/>
<text x="108" y="316" text-anchor="middle" font-size="9" fill="#0F766E">job 848</text>
<text x="108" y="336" text-anchor="middle" font-size="9" fill="#0F766E">job 849</text>
<rect x="224" y="276" width="156" height="70" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="302" y="296" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">ĐANG XỬ LÝ</text>
<text x="302" y="313" text-anchor="middle" font-size="10" fill="#0F766E">(processing list / PEL /</text>
<text x="302" y="328" text-anchor="middle" font-size="10" fill="#0F766E">unacked)</text>
<text x="302" y="342" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">job 847 nằm ĐÂY</text>
<rect x="436" y="276" width="120" height="70" rx="8" fill="#FFD9DE" stroke="#DC2626" stroke-width="2"/>
<text x="496" y="302" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">💀 CRASH</text>
<text x="496" y="322" text-anchor="middle" font-size="10" fill="#BE123C">job vẫn còn ở</text>
<text x="496" y="337" text-anchor="middle" font-size="10" fill="#BE123C">danh sách bên trái</text>
<rect x="612" y="276" width="244" height="70" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="734" y="298" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">PHỤC HỒI</text>
<text x="734" y="316" text-anchor="middle" font-size="10" fill="#0F766E">Worker mới thấy job 847 dang dở</text>
<text x="734" y="334" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">→ nhận lại và xử lý tiếp</text>
<line x1="172" y1="311" x2="220" y2="311" stroke="#0D9488" stroke-width="2" marker-end="url(#q1-t)"/>
<line x1="384" y1="311" x2="432" y2="311" stroke="#0D9488" stroke-width="2" marker-end="url(#q1-t)"/>
<line x1="560" y1="311" x2="608" y2="311" stroke="#0D9488" stroke-width="2" marker-end="url(#q1-t)"/>
<rect x="24" y="380" width="852" height="66" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="404" font-size="13" font-weight="800" fill="#7C4A03">Đo thật: 2.000 job · 20 worker crash giữa chừng</text>
<text x="44" y="428" font-size="11" fill="#92400E"><tspan font-family="monospace" font-weight="700" fill="#DC2626">LPOP → 1.980/2.000 (mất 20)</tspan>  ·  LMOVE → 2.000/2.000  ·  Streams → 2.000/2.000  ·  RabbitMQ → 2.000/2.000</text>
</svg>
</div>

---

# "Redis Queue" nghĩa là gì? Bốn cách rất khác nhau

Khi một team nói họ dùng "Redis Queue", câu đó **chưa nói gì về độ bền của job**. Có ít nhất bốn cách, và phần việc ứng dụng phải tự làm khác nhau rất xa.

| Cách | Giữ job đang xử lý | Tự phát hiện job kẹt | Retry / DLQ | Việc app phải tự làm |
|---|:---:|:---:|:---:|---|
| **List + `LPOP`** | ✗ | ✗ | ✗ | **Tất cả** — và nếu không làm thì mất job |
| **List + `LMOVE`** | ✓ | ✗ | ✗ | Reaper quét job kẹt, đếm retry, DLQ |
| **Streams + consumer group** | ✓ (PEL) | ✗ | ✗ | Chạy `XAUTOCLAIM` định kỳ, DLQ theo `delivery_count` |
| **BullMQ / Asynq** | ✓ | ✓ | ✓ | Gần như không — thư viện đóng gói sẵn |

## ① List + `LPOP` — cách làm mất job

```go
// ❌ Job rời khỏi Redis NGAY khi pop. Không ai biết worker nào đang giữ nó.
func (w *Worker) runNaive(ctx context.Context) {
    for {
        job, err := w.rdb.LPop(ctx, "jobs:email").Result()
        if err == redis.Nil {
            time.Sleep(200 * time.Millisecond)
            continue
        }
        // ← Crash ở đây = job biến mất vĩnh viễn.
        //   Queue rỗng, không log, không alert.
        w.sendEmail(ctx, job)
    }
}
```

## ② List + `LMOVE` — reliable queue tự ghép

`LMOVE` (hoặc `BRPOPLPUSH` ở Redis cũ) **atomic**: lấy khỏi queue chờ **và** đặt vào danh sách đang-xử-lý trong cùng một lệnh.

```go
const (
    qPending    = "jobs:email"
    qProcessing = "jobs:email:processing"
)

func (w *Worker) runReliable(ctx context.Context) {
    for {
        // Atomic: rời qPending VÀ vào qProcessing. Không có khoảng hở.
        job, err := w.rdb.LMove(ctx, qPending, qProcessing, "left", "right").Result()
        if err == redis.Nil {
            time.Sleep(200 * time.Millisecond)
            continue
        }
        if err != nil {
            w.logger.Error("lmove thất bại", zap.Error(err))
            continue
        }

        if err := w.sendEmail(ctx, job); err != nil {
            // Thất bại: để job NGUYÊN trong qProcessing cho reaper xử lý.
            // Đừng tự đẩy về qPending ở đây — sẽ retry vô hạn không đếm được.
            w.logger.Warn("gửi email lỗi", zap.Error(err))
            continue
        }
        // ACK = xoá khỏi danh sách đang-xử-lý
        w.rdb.LRem(ctx, qProcessing, 1, job)
    }
}
```

Nhưng `LMOVE` **chỉ giải quyết một nửa**. Phần còn lại ứng dụng phải tự viết: một **reaper** quét job kẹt quá lâu rồi đưa về queue chờ.

```go
// Reaper — Redis KHÔNG tự làm việc này
func (r *Reaper) Run(ctx context.Context) {
    t := time.NewTicker(30 * time.Second)
    defer t.Stop()
    for range t.C {
        jobs, err := r.rdb.LRange(ctx, qProcessing, 0, -1).Result()
        if err != nil {
            continue
        }
        for _, raw := range jobs {
            var j Job
            if json.Unmarshal([]byte(raw), &j) != nil {
                continue
            }
            if time.Since(j.StartedAt) < 5*time.Minute {
                continue   // vẫn đang chạy bình thường
            }
            j.Attempts++
            if j.Attempts > maxAttempts {
                // DLQ — cũng phải tự làm
                r.rdb.LPush(ctx, qPending+":dlq", raw)
                r.rdb.LRem(ctx, qProcessing, 1, raw)
                continue
            }
            nb, _ := json.Marshal(j)
            r.rdb.LRem(ctx, qProcessing, 1, raw)
            r.rdb.LPush(ctx, qPending, nb)
        }
    }
}
```

> ⚠️ **Cái bẫy của `LMOVE`**: `j.StartedAt` phải nằm **trong payload của job**, không phải trong RAM của worker. Nếu bạn lưu thời điểm bắt đầu ở biến cục bộ của worker, worker chết là mất luôn — và reaper không có cách nào biết job kẹt bao lâu. Nghe hiển nhiên, nhưng đây đúng là lỗi tôi hay thấy khi team tự ghép reliable queue.

## ③ Streams + consumer group — Redis theo dõi hộ

Streams có **PEL** (Pending Entries List) — Redis tự giữ danh sách "đã giao, chưa `XACK`". Đo thật: 20 worker crash → **PEL giữ đúng 20 job**.

```go
func (w *Worker) runStream(ctx context.Context) {
    for {
        res, err := w.rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
            Group: "emailers", Consumer: w.name,
            Streams: []string{"jobs:email", ">"},   // ">" = tin MỚI chưa ai nhận
            Count:   10, Block: time.Second,
        }).Result()
        if err != nil || len(res) == 0 {
            continue
        }
        for _, m := range res[0].Messages {
            if err := w.sendEmail(ctx, m.Values); err != nil {
                continue   // không XACK → vẫn nằm trong PEL
            }
            w.rdb.XAck(ctx, "jobs:email", "emailers", m.ID)
        }
    }
}
```

Nhưng — như đã nói kỹ ở [P03](/posts/system-design-p03-pubsub-vs-streams/#vấn-đề-1--worker-nhận-tin-rồi-chết-trước-khi-xử-lý-xong) — **Redis không tự giao lại job trong PEL.** Nó giữ vô thời hạn. App phải tự chạy `XAUTOCLAIM`:

```go
// Phải chạy ĐỊNH KỲ. Không có nó, job nằm trong PEL mãi mãi.
func (w *Worker) reclaim(ctx context.Context) {
    start := "0-0"
    for {
        msgs, next, err := w.rdb.XAutoClaim(ctx, &redis.XAutoClaimArgs{
            Stream: "jobs:email", Group: "emailers", Consumer: w.name,
            MinIdle: 5 * time.Minute,   // DÀI HƠN thời gian xử lý bình thường
            Start:   start, Count: 100,
        }).Result()
        if err != nil || len(msgs) == 0 {
            return
        }
        for _, m := range msgs {
            w.handleWithRetryLimit(ctx, m)
        }
        if next == "0-0" {
            return
        }
        start = next
    }
}
```

## ④ BullMQ / Asynq — thư viện đóng gói sẵn

[BullMQ](https://docs.bullmq.io/) (Node) và [Asynq](https://github.com/hibiken/asynq) (Go) dựng trên Redis nhưng đóng gói sẵn: phát hiện job kẹt (stalled job), retry có backoff, DLQ, scheduled job, rate limit theo queue.

```go
// Asynq — retry và DLQ là CẤU HÌNH, không phải code bạn viết
srv := asynq.NewServer(
    asynq.RedisClientOpt{Addr: "127.0.0.1:6379"},
    asynq.Config{
        Concurrency: 20,
        Queues:      map[string]int{"critical": 6, "default": 3, "low": 1},
        RetryDelayFunc: func(n int, err error, t *asynq.Task) time.Duration {
            return time.Duration(1<<uint(n)) * time.Second   // backoff 2^n
        },
    },
)
// Job hết lượt retry tự vào "archived set" — DLQ có sẵn
client.Enqueue(task, asynq.MaxRetry(5), asynq.Timeout(30*time.Second))
```

👉 **Nếu bài toán của bạn là background job (gửi email, resize ảnh, đồng bộ dữ liệu), hãy dùng thư viện.** Tự ghép reliable queue bằng `LMOVE` + reaper nghĩa là bạn sẽ tự viết lại — và tự debug lại — những thứ BullMQ/Asynq đã làm xong và đã được dùng ở nhiều nơi.

---

# RabbitMQ + manual ACK

RabbitMQ sinh ra cho đúng bài toán này, nên phần theo dõi job đang xử lý là **mặc định của broker**, không phải thứ app tự làm.

```go
ch.Qos(1, 0, false)   // prefetch 1: không giữ nhiều job chưa ACK cùng lúc

deliveries, err := ch.Consume(
    "jobs.email", consumerName,
    false,  // autoAck = FALSE — ĐÂY là dòng quan trọng nhất
    false, false, false, nil,
)

for d := range deliveries {
    if err := sendEmail(d.Body); err != nil {
        // requeue=false → đẩy sang DLQ đã cấu hình, KHÔNG quay lại queue cũ.
        // requeue=true sẽ tạo vòng lặp vô hạn cho job lỗi vĩnh viễn.
        d.Nack(false, false)
        continue
    }
    d.Ack(false)
}
```

Khi worker crash và RabbitMQ phát hiện **kết nối đã đóng**, mọi message chưa ACK trên kết nối đó **tự động quay lại hàng đợi**. Không cần reaper, không cần `XAUTOCLAIM`.

> 📌 Đo thật, tôi thấy một chi tiết đáng chú ý: khi 20 worker đóng kết nối, RabbitMQ requeue ngay và **phần lớn job được giao lại cho các worker còn sống trong lúc chạy** — chỉ 8 job đến tay consumer phục hồi. Tức phục hồi diễn ra **tự động và gần như tức thì**, khác hẳn Redis Streams nơi job nằm im trong PEL tới khi bạn chủ động gọi `XAUTOCLAIM`.

Nhưng **retry bao lâu, tối đa mấy lần, lỗi mãi thì đi đâu** vẫn cần cấu hình:

```go
// DLQ trong RabbitMQ là cấu hình queue, không phải code
ch.ExchangeDeclare("jobs.dlx", "direct", true, false, false, false, nil)
ch.QueueDeclare("jobs.email.dlq", true, false, false, false, nil)
ch.QueueBind("jobs.email.dlq", "email", "jobs.dlx", false, nil)

ch.QueueDeclare("jobs.email", true, false, false, false, amqp.Table{
    "x-dead-letter-exchange":    "jobs.dlx",
    "x-dead-letter-routing-key": "email",
    "x-message-ttl":             int32(300000),  // 5 phút
    "x-max-length":              int32(100000),  // tránh queue phình vô hạn
})
```

---

# Bẫy thứ hai: việc đã làm xong nhưng chết trước khi ACK

Giữ được job mới giải quyết một nửa. Trường hợp ngược lại: **email đã gửi thành công, worker chết trước lúc `ACK`.** Job được giao lại, và khách nhận **hai email**.

Tôi đo thật cùng kịch bản 20 worker crash, nhưng crash **sau** khi gửi email:

| | Gửi được | Trùng |
|---|---:|---:|
| ⑤ Streams, **không** chống trùng | 2.000/2.000 | **⚠️ 20 email trùng** |
| ⑥ Streams + `SET NX` theo job id | 2.000/2.000 | **0** |

Trùng đúng 20 — bằng đúng số worker crash. Và đây là điều quan trọng: **cả bốn phương án ở trên đều có vấn đề này.** Nó là hệ quả tất yếu của **at-least-once**, không có broker nào tránh được — kể cả RabbitMQ, kể cả Kafka.

Chỗ duy nhất sửa được là **điểm thực hiện công việc**:

```go
func (w *Worker) sendEmailIdempotent(ctx context.Context, job Job) error {
    // Khoá theo ID NGHIỆP VỤ (order_id + loại email),
    // KHÔNG theo message ID — vì producer retry sẽ sinh message ID khác
    // cho cùng một sự kiện nghiệp vụ.
    key := fmt.Sprintf("sent:order_confirm:%s", job.OrderID)

    ok, err := w.rdb.SetNX(ctx, key, "sending", 24*time.Hour).Result()
    if err != nil {
        return err   // lỗi Redis → KHÔNG ack, để được giao lại
    }
    if !ok {
        return nil   // đã gửi rồi → coi như thành công, ack và đi tiếp
    }

    if err := w.mailer.Send(ctx, job); err != nil {
        w.rdb.Del(ctx, key)   // NHẢ khoá để lần sau gửi lại được
        return err
    }
    return nil
}
```

Hai chi tiết quyết định:

1. **Khoá theo ID nghiệp vụ, không theo message ID.** Nếu service đặt hàng retry việc *tạo job*, bạn sẽ có hai message ID khác nhau cho cùng một đơn — khoá theo message ID không chặn được.
2. **Nhả khoá khi thất bại.** Quên bước này thì một lỗi SMTP tạm thời khiến đơn đó **vĩnh viễn** không bao giờ có email: khoá đã chiếm nhưng việc chưa làm.

> ⚠️ Với các tác vụ liên quan tới **tiền** hoặc **gọi dịch vụ bên ngoài**, `SET NX` là chưa đủ — Redis có thể mất khoá (restart, eviction). Lớp bảo đảm cuối phải là **unique constraint trong database**, như đã viết kỹ ở [P01.3](/posts/scalable-p01-3-rps-deep-dive/#hot-path-3--idempotency).

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="q2-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="q2-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai bẫy, hai chỗ sửa khác nhau</text>
<rect x="24" y="46" width="408" height="200" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#q2-sh)"/>
<text x="228" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">BẪY 1 — MẤT việc</text>
<rect x="46" y="84" width="364" height="34" rx="7" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.6"/>
<text x="228" y="106" text-anchor="middle" font-size="11" font-family="monospace" fill="#9F1239">pop job → 💀 CRASH → gửi email</text>
<text x="46" y="140" font-size="11" font-weight="800" fill="#374151">Hậu quả</text>
<text x="46" y="160" font-size="11" fill="#BE123C">Khách KHÔNG nhận được email.</text>
<text x="46" y="178" font-size="11" fill="#BE123C">Queue rỗng, không log, không alert.</text>
<rect x="46" y="190" width="364" height="44" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="228" y="208" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Sửa ở BROKER / cách nhận job</text>
<text x="228" y="226" text-anchor="middle" font-size="10" fill="#BE123C">LMOVE · PEL · manual ACK — chọn 1 là xong</text>
<rect x="448" y="46" width="408" height="200" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#q2-sh)"/>
<text x="652" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">BẪY 2 — LÀM TRÙNG</text>
<rect x="470" y="84" width="364" height="34" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.6"/>
<text x="652" y="106" text-anchor="middle" font-size="11" font-family="monospace" fill="#7C4A03">pop → gửi email → 💀 CRASH → ACK</text>
<text x="470" y="140" font-size="11" font-weight="800" fill="#374151">Hậu quả</text>
<text x="470" y="160" font-size="11" fill="#92400E">Khách nhận HAI email cho cùng 1 đơn.</text>
<text x="470" y="178" font-size="11" fill="#92400E">Với thanh toán: trừ tiền hai lần.</text>
<rect x="470" y="190" width="364" height="44" rx="7" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="652" y="208" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">Sửa ở ỨNG DỤNG — broker KHÔNG giúp được</text>
<text x="652" y="226" text-anchor="middle" font-size="10" fill="#92400E">SET NX theo ID nghiệp vụ + unique constraint DB</text>
<rect x="24" y="266" width="832" height="54" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="44" y="288" font-size="12" font-weight="800" fill="#9F1239">Đo thật — cùng kịch bản 20 worker crash, chỉ khác thời điểm chết</text>
<text x="44" y="310" font-size="11" fill="#BE123C">Crash TRƯỚC khi gửi + LPOP → <tspan font-family="monospace" font-weight="700">mất 20 job</tspan>.   Crash SAU khi gửi + không chống trùng → <tspan font-family="monospace" font-weight="700">20 email trùng</tspan>.</text>
<rect x="24" y="334" width="832" height="52" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="44" y="356" font-size="12" font-weight="800" fill="#134E4A">Chọn broker nào cũng chỉ giải quyết BẪY 1. Bẫy 2 là at-least-once — không broker nào tránh được.</text>
<text x="44" y="376" font-size="11" fill="#0F766E">RabbitMQ, Kafka, Redis Streams đều vậy. Phải chống trùng ở tầng nghiệp vụ, không có đường tắt.</text>
</svg>
</div>

---

# Thông lượng: con số và những gì nó KHÔNG nói

| Thao tác | Throughput |
|---|---:|
| **Enqueue** — Redis `RPUSH` (pipeline 2.000) | 827.264 job/s |
| **Enqueue** — Redis `XADD` (pipeline 2.000) | 497.416 job/s |
| **Enqueue** — RabbitMQ transient + publisher confirm | 39.799 job/s |
| **Enqueue** — RabbitMQ **persistent** + publisher confirm | 34.749 job/s |
| **Dequeue+ACK** — Redis Streams (batch 200) | 86.263 job/s |
| **Dequeue+ACK** — RabbitMQ (prefetch 200) | 69.208 job/s |

> ⚠️ **Hai cột enqueue KHÔNG so sánh công bằng được, và tôi muốn nói rõ điều đó.** Redis chạy pipeline 2.000 lệnh — gộp round-trip mạng và **không chờ xác nhận ghi đĩa**. RabbitMQ dùng *publisher confirm*, tức **chờ broker xác nhận đã nhận** từng message. Khoảng cách ~20× phản ánh **mức bảo đảm khác nhau** nhiều hơn là phản ánh tốc độ.
>
> Cột **dequeue công bằng hơn**: cả hai đều batch (200) và đều ACK. Ở đó khoảng cách chỉ còn **1,25×** (86.263 vs 69.208).

Và đây là điều quan trọng hơn mọi con số trên: **35.000 job/s cho background job là thừa sức với gần như mọi ứng dụng.** Nếu bạn gửi 35.000 email mỗi giây thì nút thắt của bạn là nhà cung cấp email, không phải broker. Throughput **không phải lý do** để chọn giữa hai thứ này.

---

# Điểm phân biệt thật: routing

Đây mới là chỗ RabbitMQ khác về bản chất.

**Tình huống**: sau khi thanh toán, **cả bên email và bên kho** đều cần nhận sự kiện, nhưng xử lý **độc lập**.

Nếu cho hai vai trò cùng lấy từ **một work queue**, mỗi lượt giao chỉ **một bên** nhận được việc — bên kia không thấy gì. Đây là lỗi thiết kế mà rất nhiều hệ thống mắc khi mới thêm consumer thứ hai.

| | **RabbitMQ** | **Redis Streams** |
|---|---|---|
| Cách làm | 1 exchange → bind nhiều queue | 1 stream → nhiều consumer group |
| Ai lo routing | **Broker** (cấu hình, không code) | Ứng dụng tự tạo group và quản lý |
| Thêm vai trò mới | Khai báo queue + bind — **không sửa producer** | Tạo consumer group — không sửa producer |
| Routing theo điều kiện | **Có sẵn**: direct, topic, headers | **Không có** — phải tự lọc ở consumer hoặc tách stream |
| Lưu trữ | **Nhân bản**: 2 queue = 2 bản | **Không nhân bản**: 1 bản, 2 con trỏ đọc |

Đo thật với 5.000 sự kiện và 2 vai trò:

| | Thời gian publish | Dữ liệu lưu |
|---|---:|---|
| RabbitMQ fanout exchange | 35 ms | **10.000 message** (5.000 × 2 queue) |
| Redis Streams, 2 consumer group | 11 ms | **5.000 entry** (2 group đọc cùng dữ liệu) |

Hai đặc tính rất khác nhau:

- **RabbitMQ nhân bản dữ liệu.** Thêm vai trò thứ ba = thêm 5.000 message nữa. Bù lại mỗi queue có vòng đời độc lập hoàn toàn: xoá queue email không ảnh hưởng kho, và retention của mỗi queue tách biệt.
- **Redis Streams không nhân bản.** 10 consumer group vẫn chỉ 5.000 entry. Bù lại chúng **chia sẻ chung một vòng đời**: `MAXLEN` cắt bớt là cắt cho *mọi* group — nếu một group đang chậm, nó có thể mất dữ liệu vì group khác đã đọc xong và stream bị trim.

> 📌 Điểm đánh đổi này ít được nói tới nhưng rất thực tế: **với Redis Streams, một consumer group chậm có thể bị mất dữ liệu do `MAXLEN`.** Với RabbitMQ, queue của nó cứ dài ra (tới `x-max-length`) nhưng không mất gì vì queue khác. Nếu bạn có một consumer hay bị tụt lại (ví dụ đồng bộ sang hệ thống cũ, chậm và hay lỗi), RabbitMQ an toàn hơn về điểm này.

## Routing có điều kiện — chỗ Redis không có tương đương

```go
// RabbitMQ topic exchange: routing theo PATTERN, broker lo hết
ch.ExchangeDeclare("ev.order", "topic", true, false, false, false, nil)

ch.QueueBind("q.email",     "order.paid.*",      "ev.order", false, nil)
ch.QueueBind("q.warehouse", "order.paid.*",      "ev.order", false, nil)
ch.QueueBind("q.fraud",     "order.paid.highval","ev.order", false, nil) // CHỈ đơn giá trị cao
ch.QueueBind("q.audit",     "order.#",           "ev.order", false, nil) // MỌI sự kiện order

// Producer chỉ cần publish 1 lần với routing key đúng
ch.Publish("ev.order", "order.paid.highval", false, false, msg)
```

Với Redis Streams, muốn có `q.fraud` chỉ nhận đơn giá trị cao thì bạn phải: **tách stream riêng** (producer phải biết logic này), hoặc **cho consumer đọc hết rồi tự lọc** (tốn băng thông và CPU cho những tin nó không cần).

Đó chính là "phần routing broker làm sẵn" — và là lý do chính đáng nhất để chọn RabbitMQ.

---

# Chọn thế nào?

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="q3-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="q3-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="480" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai câu hỏi, theo đúng thứ tự</text>
<rect x="190" y="46" width="500" height="50" rx="10" fill="#DC2626" filter="url(#q3-sh)"/>
<text x="440" y="68" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">① Cách triển khai hiện tại có GIỮ được job khi worker chết không?</text>
<text x="440" y="87" text-anchor="middle" font-size="11" fill="#FFD9DE">Đây là câu hỏi ĐẦU TIÊN — không phải "Redis hay RabbitMQ"</text>
<rect x="30" y="118" width="390" height="104" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#q3-sh)"/>
<text x="225" y="142" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">KHÔNG → sửa cái này TRƯỚC</text>
<text x="48" y="164" font-size="11" fill="#BE123C">Thêm worker vào thiết kế đang mất job chỉ tạo thêm</text>
<text x="48" y="182" font-size="11" fill="#BE123C">tiến trình nhìn vào một queue rỗng.</text>
<text x="48" y="206" font-size="11" font-weight="800" fill="#DC2626">Đổi broker chưa cần. Đổi cách NHẬN job đã.</text>
<rect x="450" y="118" width="400" height="104" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#q3-sh)"/>
<text x="650" y="142" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">CÓ → sang câu ②</text>
<text x="468" y="164" font-size="11" fill="#0F766E">Đã có LMOVE+reaper, hoặc Streams+XAUTOCLAIM,</text>
<text x="468" y="182" font-size="11" fill="#0F766E">hoặc BullMQ/Asynq, hoặc RabbitMQ manual ACK.</text>
<text x="468" y="206" font-size="11" font-weight="700" fill="#134E4A">Và đã TEST bằng cách kill worker giữa chừng.</text>
<rect x="190" y="244" width="500" height="50" rx="10" fill="#374151" filter="url(#q3-sh)"/>
<text x="440" y="266" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">② Có cần phân phối message theo nhiều RULE giữa các service?</text>
<text x="440" y="285" text-anchor="middle" font-size="11" fill="#D6D3D1">Đây mới là chỗ Redis và RabbitMQ khác về bản chất</text>
<rect x="30" y="316" width="390" height="136" rx="10" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#q3-sh)"/>
<text x="225" y="340" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">GIỮ REDIS</text>
<text x="48" y="362" font-size="11" fill="#0F766E">• Background job: email, resize ảnh, đồng bộ</text>
<text x="48" y="380" font-size="11" fill="#0F766E">• Fan-out đơn giản: vài consumer group độc lập</text>
<text x="48" y="398" font-size="11" fill="#0F766E">• Thư viện đã lo giữ job, phục hồi, retry</text>
<text x="48" y="420" font-size="11" font-weight="700" fill="#9F1239">Vẫn phải tự lo: persistence, tránh eviction xoá</text>
<text x="48" y="436" font-size="11" font-weight="700" fill="#9F1239">key queue như cache, theo dõi job lỗi</text>
<rect x="450" y="316" width="400" height="136" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#q3-sh)"/>
<text x="650" y="340" text-anchor="middle" font-size="14" font-weight="800" fill="#3730A3">CÂN NHẮC RABBITMQ</text>
<text x="468" y="362" font-size="11" fill="#4338CA">• Nhiều service nhận theo nhiều rule khác nhau</text>
<text x="468" y="380" font-size="11" fill="#4338CA">• Muốn BROKER lo routing (topic, headers)</text>
<text x="468" y="398" font-size="11" fill="#4338CA">• Có consumer hay tụt lại → queue riêng an toàn hơn</text>
<text x="468" y="420" font-size="11" font-weight="700" fill="#9F1239">Vẫn phải tự lo: cấu hình persistence, DLQ,</text>
<text x="468" y="436" font-size="11" font-weight="700" fill="#9F1239">retry policy, và công vận hành thêm 1 hệ thống</text>
<line x1="225" y1="98" x2="225" y2="114" stroke="#DC2626" stroke-width="2" marker-end="url(#q3-a)"/>
<line x1="650" y1="98" x2="650" y2="114" stroke="#0D9488" stroke-width="2" marker-end="url(#q3-a)"/>
<line x1="650" y1="224" x2="650" y2="240" stroke="#57534E" stroke-width="2" marker-end="url(#q3-a)"/>
<path d="M440,296 L440,306 L225,306 L225,312" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#q3-a)"/>
<path d="M440,296 L440,306 L650,306 L650,312" stroke="#6366F1" stroke-width="2" fill="none" marker-end="url(#q3-a)"/>
<rect x="110" y="300" width="60" height="18" rx="5" fill="#0D9488"/>
<rect x="710" y="300" width="48" height="18" rx="5" fill="#6366F1"/>
<text x="140" y="314" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="734" y="314" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">CÓ</text>
</svg>
</div>

Điểm then chốt ở sơ đồ: **thứ tự hai câu hỏi.** Team trong câu chuyện mở đầu không có vấn đề "chọn sai broker" — họ có vấn đề "không có chỗ giữ job đang xử lý". Đổi sang RabbitMQ sẽ sửa được, nhưng `LMOVE` + reaper, hay đổi sang Asynq, cũng sửa được — và rẻ hơn nhiều so với thêm một hệ thống phân tán vào stack.

## Một lưu ý riêng cho Redis làm queue

Nếu Redis của bạn **dùng chung cho cả cache và queue**, có một rủi ro cụ thể mà tôi đã đo ở [P04](/posts/system-design-p04-streams-vs-kafka/#chuyện-gì-xảy-ra-khi-stream-dùng-chung-redis-với-cache): với `maxmemory-policy = allkeys-lru`, Redis **xoá cả key chứa queue** khi hết bộ nhớ — mất 72.000 message, `XADD` vẫn báo thành công.

```bash
# Instance cache — evict thoải mái, đó là bản chất cache
redis-server --port 6379 --maxmemory 4gb --maxmemory-policy allkeys-lru

# Instance QUEUE — KHÔNG BAO GIỜ evict
redis-server --port 6380 --maxmemory 8gb --maxmemory-policy noeviction
```

Đây chính là điều "team vẫn phải lo lưu dữ liệu, tránh eviction xoá key của queue như cache".

---

# Quay lại câu chuyện đầu bài

Cách kiểm tra đúng không phải nhìn queue còn job hay không, mà là **dựng lại đúng tình huống worker chết giữa chừng**:

```
□ Kill worker ĐÚNG LÚC nó đang giữ job (không phải lúc rảnh)
□ Worker mới lên có TÌM ĐƯỢC job bỏ dở không?
□ Job đó có được xử lý tới bước gửi thành công không?
□ Có xác nhận hoàn tất (ACK) không?
□ Đếm: số job gửi vào = số email gửi ra? (khác nhau = đang mất job)
□ Kill worker SAU khi gửi, TRƯỚC khi ACK → có gửi trùng không?
```

Nhìn queue hết job chờ thôi thì vẫn chưa biết khách đã nhận được gì.

## Metric phải có

| Metric | Vì sao |
|---|---|
| **`jobs_enqueued` − `jobs_completed`** | **Phải về 0.** Dương kéo dài = đang mất job |
| Độ sâu **danh sách đang-xử-lý** (processing / PEL / unacked) | Tăng đều không giảm = có worker chết không ai thu hồi |
| Tuổi của job cũ nhất trong danh sách đó | Tín hiệu sớm nhất của job kẹt |
| `jobs_reclaimed/s` | Reaper / `XAUTOCLAIM` có đang chạy thật không |
| `jobs_duplicate_blocked` | Chống trùng có hoạt động — và bị kích hoạt bao nhiêu lần |
| Độ sâu DLQ | Job lỗi vĩnh viễn đang tích tụ |

**Metric quan trọng nhất: `jobs_enqueued − jobs_completed`.** Nó là metric **đúng đắn**, không phải hiệu năng — giống `oversell_count` ở [P01.3](/posts/scalable-p01-3-rps-deep-dive/) và `messages_acked − messages_persisted` ở [P01.4](/posts/scalable-p01-4-chat-deep-dive/). Nếu dương kéo dài, bạn đang mất việc — và "queue rỗng" sẽ không bao giờ nói cho bạn biết.

---

# Checklist

```txt
TRƯỚC KHI ĐƯA QUEUE LÊN PRODUCTION
□ Nhận job bằng cách nào? LPOP (❌) / LMOVE / Streams / manual ACK?
□ Có chỗ giữ job ĐÃ GIAO nhưng CHƯA XÁC NHẬN không?
□ Ai thu hồi job kẹt? (reaper tự viết / XAUTOCLAIM / broker tự làm)
□ Nếu tự viết reaper: StartedAt nằm TRONG payload hay trong RAM worker?
□ MinIdle / timeout thu hồi có DÀI HƠN thời gian xử lý bình thường?
□ Đếm số lần retry ở đâu? Vượt ngưỡng thì job đi đâu (DLQ)?
□ Handler đã idempotent chưa? Khoá theo ID NGHIỆP VỤ hay message ID?
□ Khi xử lý thất bại, khoá idempotency có được NHẢ không?
□ Tác vụ liên quan tiền → có unique constraint trong DB chưa?

NẾU DÙNG REDIS
□ Queue có nằm CHUNG instance với cache không? → TÁCH RA
□ maxmemory-policy của instance queue = noeviction?
□ AOF/RDB đã bật và đã TEST phục hồi thật chưa?
□ Streams có MAXLEN chưa? Consumer group chậm nhất có bị trim mất dữ liệu?
□ Đã cân nhắc dùng BullMQ/Asynq thay vì tự ghép chưa?

NẾU DÙNG RABBITMQ
□ autoAck = false chưa? (true = mất job y như LPOP)
□ Qos/prefetch đã đặt chưa? (không đặt = 1 worker hút hết queue)
□ DLX + x-dead-letter-routing-key đã cấu hình chưa?
□ Nack(requeue=false) cho lỗi vĩnh viễn, không phải requeue=true?
□ x-max-length có chưa? (tránh queue phình vô hạn)
□ Queue và message đã durable + persistent chưa?

TEST
□ Đã kill worker ĐANG GIỮ job và đếm lại chưa?
□ Đã kill worker SAU khi làm xong, TRƯỚC khi ACK chưa?
□ Có metric jobs_enqueued − jobs_completed không?
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| "Queue rỗng" nghĩa là gì? | **Chỉ là không còn job đang CHỜ.** Nó không nói gì về việc đã xong hay chưa |
| `LPOP` mất bao nhiêu job? | Đo thật: **20/2.000**, bằng đúng số worker crash. Không lỗi, không log |
| "Redis Queue" là gì? | **Bốn cách rất khác nhau**: List+`LPOP` (mất job), List+`LMOVE`, Streams+PEL, BullMQ/Asynq |
| Redis có tự thu hồi job kẹt? | **Không.** `LMOVE` cần reaper tự viết; Streams cần `XAUTOCLAIM` chạy định kỳ |
| RabbitMQ khác gì ở đây? | **Tự requeue khi kết nối đóng** — phục hồi gần như tức thì, không cần reaper |
| Chọn broker là xong chưa? | **Chưa.** Cả 4 đều gửi trùng khi crash sau khi làm xong: đo được **20 email trùng** |
| Chống trùng ở đâu? | **Tầng nghiệp vụ** — `SET NX` theo ID nghiệp vụ + unique constraint DB. Không broker nào làm hộ |
| Throughput có phải lý do chọn? | **Không.** 35.000 job/s đã thừa cho background job. Và hai cột enqueue không so sánh công bằng được |
| Điểm phân biệt thật? | **Routing.** RabbitMQ có exchange (direct/topic/headers); Redis không có tương đương |
| Đánh đổi của fan-out? | RabbitMQ **nhân bản** dữ liệu (2 queue = 2 bản) nhưng vòng đời độc lập; Redis **1 bản** nhưng `MAXLEN` cắt cho mọi group |
| Metric phải về 0? | **`jobs_enqueued` − `jobs_completed`** |

---

# Tham khảo

- [Redis — Reliable queue pattern (`LMOVE`)](https://redis.io/docs/latest/commands/lmove/) · [Streams consumer group](https://redis.io/docs/latest/develop/data-types/streams/)
- [RabbitMQ — Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms) · [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [Asynq](https://github.com/hibiken/asynq) (Go) · [BullMQ](https://docs.bullmq.io/) (Node) — thư viện queue trên Redis
- [Brandur — Transactionally Staged Job Drains](https://brandur.org/job-drain) — vì sao nên tạo job trong cùng transaction với dữ liệu
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — Redis không chỉ có GET/SET](/posts/system-design-p01-redis/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/) · [P04 — Streams vs Kafka](/posts/system-design-p04-streams-vs-kafka/) · [P06 — API Gateway vs Load Balancer](/posts/system-design-p06-lb-vs-gateway/)
