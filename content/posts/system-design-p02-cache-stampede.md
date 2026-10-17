+++
date = '2026-10-17T09:00:00+07:00'
draft = false
slug = 'system-design-p02-cache-stampede'
title = 'System Design P02: Đã Có Redis Cache, Sao Hệ Thống Vẫn Chậm Rồi Treo?'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "redis", "caching", "performance", "golang"]
description = 'Cache stampede: khi hàng trăm request cùng miss một key và cùng dồn xuống database. Đo thật 5 chiến lược chống, từ singleflight tới probabilistic early expiration.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, ví dụ, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên MySQL 8.0.40 + Redis 7.4.2 local.*

# Tổng quan

Có một tình huống làm sập hệ thống mà hầu như ai làm tải cao cũng gặp ít nhất một lần: **các request cùng gặp cache miss, rồi cùng truy vấn database để lấy lại đúng một kết quả.**

Điều khó chịu nhất là lúc đó **Redis vẫn hoàn toàn khoẻ**. Dashboard Redis xanh lè: latency thấp, CPU thấp, không có lệnh nào chậm. Nhưng database phía sau thì quá tải vì phải xử lý dồn dập cùng một việc, và toàn bộ ứng dụng chậm dần rồi treo cứng.

Đa số engineer mới làm hệ thống tải cao rất tự tin vào Redis. Nhưng lại quên mất một câu hỏi: **khi cache invalidate thì hệ thống sẽ ra sao?**

Bài này trả lời câu hỏi đó bằng số đo thật, rồi đi qua 5 cách sửa — mỗi cách kèm code và kết quả benchmark.

---

# Bài toán

Ứng dụng bán hàng có mục **"10 sản phẩm bán chạy trong tuần"**. Để lấy danh sách này, backend phải tổng hợp dữ liệu đơn hàng từ database:

```sql
SELECT product_id, SUM(qty) AS total
FROM   order_items
WHERE  created_at >= NOW() - INTERVAL 7 DAY
GROUP  BY product_id
ORDER  BY total DESC
LIMIT  10;
```

Trên bảng 300.000 dòng, query này mất **77 ms**. Chậm, nên ta cache vào Redis với TTL 60 giây. Tất cả request mở trang đều đọc chung một key.

Code trông hoàn toàn bình thường — đây là mẫu **cache-aside** mà mọi tài liệu đều dạy:

```go
func GetBestSellers(ctx context.Context) ([]Row, error) {
    // 1. Thử đọc cache
    if raw, err := rdb.Get(ctx, "bestsellers:week").Result(); err == nil {
        var out []Row
        return out, json.Unmarshal([]byte(raw), &out)
    }
    // 2. Miss → truy vấn DB
    out, err := loadFromDB(ctx)
    if err != nil {
        return nil, err
    }
    // 3. Ghi lại cache
    b, _ := json.Marshal(out)
    rdb.Set(ctx, "bestsellers:week", b, 60*time.Second)
    return out, nil
}
```

Không có gì sai về mặt logic. Vấn đề nằm ở chỗ **nó đúng với một request, và sai với hai trăm request cùng lúc**.

## Chuyện gì xảy ra sau giây thứ 60

Request A đến trước, đọc Redis thấy miss, gửi truy vấn xuống database. **Trong lúc database đang xử lý và A chưa kịp ghi kết quả vào Redis**, request B và C tiếp tục đến. Cả hai cùng hỏi đúng key đó, cùng gặp miss, và cũng gửi tiếp câu truy vấn y hệt xuống database.

Mấu chốt nằm ở một câu:

> **Lệnh `GET` báo key không tồn tại. Nó không báo cho backend biết rằng một request khác đang lấy dữ liệu từ DB.**

Nếu có 20 request rơi vào khoảng thời gian ngắn trước khi lượt nạp đầu tiên hoàn tất, đoạn code trên tạo ra **20 truy vấn nặng chạy đồng thời cho cùng một kết quả**. Tình huống này gọi là **cache stampede** (hay thundering herd, dogpile).

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="cs-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="cs-arrG" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="cs-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cửa sổ chết: từ lúc key hết hạn tới lúc có ai đó ghi lại cache</text>
<line x1="90" y1="78" x2="840" y2="78" stroke="#57534E" stroke-width="2"/>
<rect x="90" y="56" width="140" height="44" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="230" y="56" width="330" height="44" rx="0" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<rect x="560" y="56" width="280" height="44" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="160" y="83" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">cache HIT</text>
<text x="395" y="76" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">CỬA SỔ CHẾT — mọi request đều MISS</text>
<text x="395" y="92" text-anchor="middle" font-size="11" fill="#BE123C">dài bằng thời gian chạy query (77 ms), không phải 0</text>
<text x="700" y="83" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">cache HIT trở lại</text>
<text x="230" y="120" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">t = 60s</text>
<text x="230" y="136" text-anchor="middle" font-size="10" fill="#78716C">TTL hết</text>
<text x="560" y="120" text-anchor="middle" font-size="11" font-weight="700" fill="#0D9488">t = 60,077s</text>
<text x="560" y="136" text-anchor="middle" font-size="10" fill="#78716C">A ghi xong cache</text>
<rect x="40" y="158" width="130" height="30" rx="7" fill="#57534E"/>
<text x="105" y="178" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Request A</text>
<rect x="40" y="196" width="130" height="30" rx="7" fill="#57534E"/>
<text x="105" y="216" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Request B</text>
<rect x="40" y="234" width="130" height="30" rx="7" fill="#57534E"/>
<text x="105" y="254" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Request C</text>
<rect x="40" y="272" width="130" height="30" rx="7" fill="#A8A29E"/>
<text x="105" y="292" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">... 197 request</text>
<rect x="240" y="160" width="300" height="26" rx="5" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="1.5"/>
<rect x="270" y="198" width="285" height="26" rx="5" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="1.5"/>
<rect x="300" y="236" width="265" height="26" rx="5" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="1.5"/>
<rect x="330" y="274" width="245" height="26" rx="5" fill="#F5F5F4" stroke="#D6D3D1" stroke-width="1.5" stroke-dasharray="4 3"/>
<text x="390" y="178" text-anchor="middle" font-size="11" fill="#9F1239">GET → miss → query DB...</text>
<text x="412" y="216" text-anchor="middle" font-size="11" fill="#9F1239">GET → miss → query DB...</text>
<text x="432" y="254" text-anchor="middle" font-size="11" fill="#9F1239">GET → miss → query DB...</text>
<text x="452" y="292" text-anchor="middle" font-size="11" fill="#78716C">...cùng một câu query, cùng một kết quả</text>
<line x1="230" y1="150" x2="230" y2="314" stroke="#DC2626" stroke-width="2" stroke-dasharray="5 4"/>
<line x1="560" y1="150" x2="560" y2="314" stroke="#0D9488" stroke-width="2" stroke-dasharray="5 4"/>
<rect x="600" y="158" width="256" height="144" rx="10" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2"/>
<text x="618" y="182" font-size="12" font-weight="800" fill="#9F1239">Vì sao không ai chặn được?</text>
<text x="618" y="208" font-size="11" fill="#57534E">GET trả về đúng 2 trạng thái:</text>
<text x="618" y="228" font-size="11" font-family="monospace" fill="#0F766E">• có giá trị → dùng</text>
<text x="618" y="248" font-size="11" font-family="monospace" fill="#9F1239">• redis.Nil → tự đi lấy</text>
<text x="618" y="272" font-size="11" font-weight="700" fill="#BE123C">Không có trạng thái thứ ba:</text>
<text x="618" y="292" font-size="11" font-weight="700" fill="#BE123C">"đang có người đi lấy rồi".</text>
<rect x="40" y="332" width="816" height="54" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5"/>
<text x="448" y="356" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">Đo thật: 200 request đồng thời → 200 truy vấn DB giống hệt nhau</text>
<text x="448" y="376" text-anchor="middle" font-size="12" fill="#BE123C">Query chạy một mình mất 77 ms. Dưới stampede, p99 của request là 3.437 ms — chậm gấp 44 lần.</text>
<rect x="40" y="400" width="816" height="56" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="60" y="422" font-size="12" font-weight="800" fill="#7C4A03">Điểm phản trực giác quan trọng nhất:</text>
<text x="60" y="443" font-size="12" fill="#92400E">Cửa sổ chết càng dài khi query càng chậm — mà query càng chậm thì càng nhiều request dồn vào. Đây là vòng lặp tự khuếch đại.</text>
</svg>
</div>

---

# Cơ chế sụp đổ: không phải DB chậm, mà là connection pool

Đây là phần cần hiểu rõ, vì nó giải thích vì sao **cả ứng dụng** chết chứ không chỉ riêng cái endpoint kia.

Khi nhiều truy vấn trùng lặp dồn xuống cùng lúc, database phải chia tài nguyên để xử lý. Nếu vượt quá khả năng xử lý của DB, backend phải chờ lâu hơn mới có kết quả để ghi vào Redis. Trong lúc đó, các request đến sau vẫn thấy key chưa có dữ liệu và **tiếp tục chạy lại cùng truy vấn**.

Đến khi **pool kết nối của backend bị các truy vấn này giữ chặt**, những request khác dùng chung pool — kể cả những API hoàn toàn không liên quan — cũng phải chờ và có thể timeout.

## Số đo thật

Benchmark: **200 goroutine xuất phát cùng một lúc**, cùng gọi một key vừa hết hạn. MySQL 8.0.40, pool tối đa 25 connection, query tổng hợp 77 ms trên 300.000 dòng.

| | Giá trị |
|---|---:|
| Query chạy một mình | 77 ms |
| Số truy vấn DB thực tế | **200** |
| p50 của request | 1.780 ms |
| p95 | 3.394 ms |
| **p99** | **3.437 ms** |

Làm phép tính để thấy vấn đề: 200 query ÷ 25 slot pool = **8 lượt**. Nếu mỗi query vẫn mất 77 ms thì tổng phải là ~620 ms. Thực tế là **3.437 ms** — gấp 5,5 lần. Phần chênh đó chính là **DB tự làm chậm mình**: 25 câu aggregation chạy song song giành nhau CPU, buffer pool và I/O, nên mỗi câu chậm hơn nhiều so với khi chạy một mình.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="cas-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="cas-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="420" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Vì sao CẢ ứng dụng chết, không chỉ một endpoint</text>
<rect x="30" y="52" width="150" height="76" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#cas-sh)"/>
<rect x="212" y="52" width="150" height="76" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#cas-sh)"/>
<rect x="394" y="52" width="150" height="76" rx="10" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5" filter="url(#cas-sh)"/>
<rect x="576" y="52" width="150" height="76" rx="10" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5" filter="url(#cas-sh)"/>
<rect x="30" y="178" width="150" height="76" rx="10" fill="#DC2626" filter="url(#cas-sh)"/>
<text x="105" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">① Key hết hạn</text>
<text x="105" y="98" text-anchor="middle" font-size="10" fill="#BE123C">200 request cùng</text>
<text x="105" y="114" text-anchor="middle" font-size="10" fill="#BE123C">gặp miss</text>
<text x="287" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">② 200 query</text>
<text x="287" y="98" text-anchor="middle" font-size="10" fill="#BE123C">giống hệt nhau</text>
<text x="287" y="114" text-anchor="middle" font-size="10" fill="#BE123C">dồn xuống DB</text>
<text x="469" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">③ DB tự chậm lại</text>
<text x="469" y="98" text-anchor="middle" font-size="10" fill="#BE123C">25 aggregation giành</text>
<text x="469" y="114" text-anchor="middle" font-size="10" fill="#BE123C">CPU/IO → 77ms thành 430ms</text>
<text x="651" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">④ Pool cạn</text>
<text x="651" y="98" text-anchor="middle" font-size="10" fill="#BE123C">25/25 connection bị</text>
<text x="651" y="114" text-anchor="middle" font-size="10" fill="#BE123C">giữ bởi cùng 1 query</text>
<text x="105" y="204" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">⑥ Ứng dụng treo</text>
<text x="105" y="224" text-anchor="middle" font-size="10" fill="#FFD9DE">Redis vẫn xanh lè.</text>
<text x="105" y="240" text-anchor="middle" font-size="10" fill="#FFD9DE">Dashboard không báo gì.</text>
<rect x="212" y="178" width="514" height="76" rx="10" fill="#FFFFFF" stroke="#DC2626" stroke-width="2.5" filter="url(#cas-sh)"/>
<text x="469" y="202" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">⑤ MỌI API KHÁC dùng chung pool cũng phải xếp hàng</text>
<text x="469" y="222" text-anchor="middle" font-size="11" fill="#BE123C">/login, /cart, /checkout — không liên quan gì tới "sản phẩm bán chạy" — bắt đầu timeout.</text>
<text x="469" y="240" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">Đây là chỗ một sự cố ở 1 endpoint biến thành sự cố toàn hệ thống.</text>
<line x1="182" y1="90" x2="208" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#cas-arr)"/>
<line x1="364" y1="90" x2="390" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#cas-arr)"/>
<line x1="546" y1="90" x2="572" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#cas-arr)"/>
<path d="M651,130 L651,156 L469,156 L469,174" stroke="#DC2626" stroke-width="2" fill="none" marker-end="url(#cas-arr)"/>
<path d="M212,216 L186,216" stroke="#DC2626" stroke-width="2" fill="none" marker-end="url(#cas-arr)"/>
<path d="M394,130 C360,146 330,120 300,100" stroke="#DC2626" stroke-width="2" stroke-dasharray="5 4" fill="none" marker-end="url(#cas-arr)"/>
<text x="352" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">vòng lặp tự khuếch đại</text>
<rect x="30" y="282" width="820" height="118" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="50" y="306" font-size="13" font-weight="800" fill="#7C4A03">Vì sao monitoring thường không bắt được sớm</text>
<text x="50" y="330" font-size="11" fill="#92400E">• Redis: latency thấp, CPU thấp, không slow log — <tspan font-weight="700">mọi chỉ số đều xanh</tspan>, vì Redis đang làm đúng việc của nó.</text>
<text x="50" y="352" font-size="11" fill="#92400E">• DB: CPU cao, nhưng <tspan font-weight="700">không có query nào "lạ"</tspan> — chỉ là một query quen thuộc chạy 200 lần.</text>
<text x="50" y="374" font-size="11" fill="#92400E">• App: timeout rải rác ở <tspan font-weight="700">những endpoint không liên quan</tspan> → rất dễ điều tra nhầm hướng.</text>
<text x="50" y="394" font-size="11" font-weight="700" fill="#7C4A03">👉 Chỉ số đáng theo dõi: số truy vấn/giây của từng câu query, và thời gian CHỜ lấy connection từ pool.</text>
</svg>
</div>

---

# Ba bài toán hay bị gộp làm một

Trước khi sửa, cần phân biệt ba thất bại khác nhau của tầng cache — chúng hay bị gọi chung là "cache bị sập" nhưng **cách sửa hoàn toàn khác nhau**:

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="t3-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Stampede · Avalanche · Penetration — ba bệnh khác nhau</text>
<rect x="24" y="48" width="278" height="334" rx="12" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2.5" filter="url(#t3-sh)"/>
<rect x="311" y="48" width="278" height="334" rx="12" fill="#FFFFFF" stroke="#F5A623" stroke-width="2.5" filter="url(#t3-sh)"/>
<rect x="598" y="48" width="278" height="334" rx="12" fill="#FFFFFF" stroke="#6366F1" stroke-width="2.5" filter="url(#t3-sh)"/>
<rect x="24" y="48" width="278" height="36" rx="12" fill="#DC2626"/>
<rect x="24" y="70" width="278" height="14" fill="#DC2626"/>
<rect x="311" y="48" width="278" height="36" rx="12" fill="#D97706"/>
<rect x="311" y="70" width="278" height="14" fill="#D97706"/>
<rect x="598" y="48" width="278" height="36" rx="12" fill="#6366F1"/>
<rect x="598" y="70" width="278" height="14" fill="#6366F1"/>
<text x="163" y="72" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">CACHE STAMPEDE</text>
<text x="450" y="72" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">CACHE AVALANCHE</text>
<text x="737" y="72" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">CACHE PENETRATION</text>
<text x="42" y="110" font-size="11" font-weight="800" fill="#374151">Triệu chứng</text>
<text x="42" y="130" font-size="11" fill="#57534E">MỘT key nóng hết hạn,</text>
<text x="42" y="147" font-size="11" fill="#57534E">hàng trăm request cùng</text>
<text x="42" y="164" font-size="11" fill="#57534E">nạp lại nó.</text>
<text x="42" y="192" font-size="11" font-weight="800" fill="#374151">Nguyên nhân</text>
<text x="42" y="212" font-size="11" fill="#57534E">GET không cho biết</text>
<text x="42" y="229" font-size="11" fill="#57534E">"đang có người đi lấy".</text>
<text x="42" y="258" font-size="11" font-weight="800" fill="#0F766E">Cách sửa</text>
<text x="42" y="278" font-size="11" fill="#0F766E">✓ singleflight / Promise map</text>
<text x="42" y="295" font-size="11" fill="#0F766E">✓ Distributed lock (SET NX)</text>
<text x="42" y="312" font-size="11" fill="#0F766E">✓ Stale-while-revalidate</text>
<text x="42" y="329" font-size="11" fill="#0F766E">✓ Probabilistic early expiry</text>
<text x="42" y="358" font-size="11" font-weight="700" fill="#9F1239">Đây là chủ đề chính của bài</text>
<text x="329" y="110" font-size="11" font-weight="800" fill="#374151">Triệu chứng</text>
<text x="329" y="130" font-size="11" fill="#57534E">HÀNG NGHÌN key khác</text>
<text x="329" y="147" font-size="11" fill="#57534E">nhau cùng hết hạn trong</text>
<text x="329" y="164" font-size="11" fill="#57534E">một khoảnh khắc.</text>
<text x="329" y="192" font-size="11" font-weight="800" fill="#374151">Nguyên nhân</text>
<text x="329" y="212" font-size="11" fill="#57534E">Nạp hàng loạt cùng lúc</text>
<text x="329" y="229" font-size="11" fill="#57534E">với TTL giống hệt nhau</text>
<text x="329" y="246" font-size="11" fill="#57534E">(deploy, warm-up, restart).</text>
<text x="329" y="274" font-size="11" font-weight="800" fill="#0F766E">Cách sửa</text>
<text x="329" y="294" font-size="11" fill="#0F766E">✓ TTL + jitter ngẫu nhiên</text>
<text x="329" y="311" font-size="11" fill="#0F766E">✓ Warm-up rải đều</text>
<text x="329" y="328" font-size="11" fill="#0F766E">✓ Phân tầng TTL theo độ nóng</text>
<text x="329" y="358" font-size="11" font-weight="700" fill="#7C4A03">Đo ở cuối bài — jitter 10%</text>
<text x="616" y="110" font-size="11" font-weight="800" fill="#374151">Triệu chứng</text>
<text x="616" y="130" font-size="11" fill="#57534E">Request hỏi key KHÔNG</text>
<text x="616" y="147" font-size="11" fill="#57534E">BAO GIỜ tồn tại → luôn</text>
<text x="616" y="164" font-size="11" fill="#57534E">miss, luôn xuống DB.</text>
<text x="616" y="192" font-size="11" font-weight="800" fill="#374151">Nguyên nhân</text>
<text x="616" y="212" font-size="11" fill="#57534E">Cache chỉ lưu kết quả CÓ,</text>
<text x="616" y="229" font-size="11" fill="#57534E">không lưu kết quả RỖNG.</text>
<text x="616" y="246" font-size="11" fill="#57534E">Hay bị khai thác để DDoS.</text>
<text x="616" y="274" font-size="11" font-weight="800" fill="#0F766E">Cách sửa</text>
<text x="616" y="294" font-size="11" fill="#0F766E">✓ Negative caching (TTL ngắn)</text>
<text x="616" y="311" font-size="11" fill="#0F766E">✓ Bloom filter chặn từ đầu</text>
<text x="616" y="328" font-size="11" fill="#0F766E">✓ Validate format ID trước</text>
<text x="616" y="358" font-size="11" font-weight="700" fill="#4338CA">singleflight KHÔNG cứu được</text>
</svg>
</div>

Lưu ý dòng cuối cột ba: **`singleflight` không giải quyết được cache penetration**, vì mỗi request hỏi một key *khác nhau* nên không có gì để gộp. Đây là lý do cần phân biệt rõ ba bệnh trước khi chọn thuốc.

---

# Cách sửa 1 — Gộp các lượt nạp trong cùng tiến trình

Ý tưởng: trong cùng một tiến trình backend, nếu một request đang lấy lại dữ liệu cho key này, các request đến sau **chờ và dùng chung kết quả** thay vì tự chạy thêm truy vấn.

Với Go, giải pháp chuẩn là [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight):

```go
var sf singleflight.Group

func GetBestSellers(ctx context.Context) ([]Row, error) {
    if raw, err := rdb.Get(ctx, cacheKey).Result(); err == nil {
        var out []Row
        return out, json.Unmarshal([]byte(raw), &out)
    }

    // Nhiều goroutine cùng gọi Do với CÙNG key → chỉ MỘT hàm được chạy,
    // số còn lại chờ và nhận chung kết quả.
    v, err, _ := sf.Do(cacheKey, func() (interface{}, error) {
        // Kiểm tra lại cache: lượt nạp ngay trước đó có thể vừa ghi xong
        if raw, err := rdb.Get(ctx, cacheKey).Result(); err == nil {
            var out []Row
            return out, json.Unmarshal([]byte(raw), &out)
        }
        out, err := loadFromDB(ctx)
        if err != nil {
            return nil, err
        }
        b, _ := json.Marshal(out)
        rdb.Set(ctx, cacheKey, b, 60*time.Second)
        return out, nil
    })
    if err != nil {
        return nil, err
    }
    return v.([]Row), nil
}
```

Kết quả đo, cùng điều kiện 200 request đồng thời:

| | Naive | singleflight | Cải thiện |
|---|---:|---:|---:|
| Truy vấn DB | 200 | **1** | **200×** |
| p50 | 1.780 ms | **65,7 ms** | 27× |
| p99 | 3.437 ms | **65,9 ms** | **52×** |

Để ý con số 65,9 ms: nó gần bằng đúng thời gian chạy query một mình (77 ms, thậm chí nhanh hơn chút vì buffer pool đã nóng). Nghĩa là **request chậm nhất cũng chỉ chậm bằng đúng một lần truy vấn** — không hơn.

## Bốn cái bẫy của `singleflight`

**1. Lỗi được chia sẻ cho tất cả.** Nếu lượt nạp của "người dẫn đầu" thất bại, **mọi** request đang chờ đều nhận cùng lỗi đó. Một lỗi timeout tạm thời biến thành 200 response lỗi. Nếu không muốn vậy, hãy phân biệt lỗi tạm thời và lỗi thật, hoặc gọi `Forget` để lượt sau bắt đầu lại:

```go
v, err, _ := sf.Do(key, fn)
if err != nil {
    sf.Forget(key)   // không để lượt hỏng "dính" vào các request kế tiếp
}
```

**2. Một request chậm giữ chân tất cả.** `Do` không nhận `context`. Nếu hàm nạp treo 30 giây, cả 200 request cùng treo 30 giây. Dùng `DoChan` để mỗi request tự áp timeout riêng:

```go
ch := sf.DoChan(cacheKey, fn)
select {
case res := <-ch:
    return res.Val.([]Row), res.Err
case <-ctx.Done():
    return nil, ctx.Err()   // request này bỏ cuộc...
}
```

⚠️ Nhưng cẩn thận: `DoChan` **không huỷ hàm đang chạy** khi context hết hạn — hàm vẫn chạy tiếp tới khi xong. Context chỉ giúp *request này* thoát ra, không dừng được lượt nạp.

**3. `sync.Once` KHÔNG thay thế được.** `sync.Once` chỉ thực thi hàm **một lần duy nhất cho mỗi đối tượng** — hợp với khởi tạo một lần, không hợp với cache cần nạp lại theo TTL. Lần hết hạn tiếp theo, `singleflight` vẫn nạp lại bình thường; `sync.Once` thì không bao giờ chạy lần hai.

**4. Phạm vi chỉ trong MỘT tiến trình.** Đây là giới hạn quan trọng nhất — xem phần ngay dưới.

## Với Node.js / TypeScript

Không có thư viện chuẩn, nhưng pattern tương đương chỉ vài dòng: giữ một `Map` lưu `Promise` theo cache key **ngay khi bắt đầu** lượt đọc DB.

```js
const inflight = new Map<string, Promise<Row[]>>();

async function getBestSellers(): Promise<Row[]> {
  const cached = await redis.get(CACHE_KEY);
  if (cached) return JSON.parse(cached);

  const existing = inflight.get(CACHE_KEY);
  if (existing) return existing;           // đã có người đi lấy → await chung

  const p = (async () => {
    const rows = await loadFromDB();
    await redis.set(CACHE_KEY, JSON.stringify(rows), { EX: 60 });
    return rows;
  })().finally(() => {
    inflight.delete(CACHE_KEY);            // xoá dù THÀNH CÔNG hay LỖI
  });

  inflight.set(CACHE_KEY, p);
  return p;
}
```

Hai chi tiết dễ sai: phải `set` vào Map **trước** khi `await`, và phải xoá entry trong `finally` — quên `finally` thì một lần lỗi sẽ khiến key đó hỏng vĩnh viễn (mọi request sau đều `await` một Promise đã reject).

`Map` này nằm trong bộ nhớ của **từng tiến trình**, không tự dùng chung giữa các instance — giống hệt giới hạn của `singleflight`.

---

# Cách sửa 2 — Khoá phân tán cho nhiều instance

Production hiếm khi chạy một instance. Và đây là điều benchmark cho thấy rõ:

| Chiến lược | Truy vấn DB | p99 |
|---|---:|---:|
| Naive | 200 | 3.437 ms |
| singleflight — **1 instance** | **1** | 65,9 ms |
| singleflight — **4 instance** | **4** | 102,7 ms |
| singleflight + Redis lock — 4 instance | **1** | 88,7 ms |

Với 4 instance, mỗi instance có `singleflight.Group` riêng nên vẫn có **4 truy vấn**. Đã giảm 50× so với 200, nhưng chưa về 1.

Câu hỏi quan trọng: **4 có đủ tốt không?** Thường là **có**. Với 20 instance thì 20 query đồng thời có thể vẫn chấp nhận được, trong khi distributed lock thêm độ phức tạp thật (thêm round-trip, thêm trường hợp lỗi, thêm thứ để debug lúc 3 giờ sáng). Chỉ leo lên mức khoá phân tán khi **query thật sự nặng** hoặc **số instance thật sự lớn**.

Nếu cần, cách làm với Redis `SET NX`:

```go
func loadWithLock(ctx context.Context) ([]Row, error) {
    lockKey := cacheKey + ":lock"
    token := uuid.NewString()               // token riêng để không xoá nhầm lock của người khác

    for i := 0; i < 50; i++ {
        // Có thể người giữ lock vừa ghi xong cache
        if raw, err := rdb.Get(ctx, cacheKey).Result(); err == nil {
            var out []Row
            return out, json.Unmarshal([]byte(raw), &out)
        }

        ok, _ := rdb.SetNX(ctx, lockKey, token, 5*time.Second).Result()
        if ok {
            defer releaseLock(ctx, lockKey, token)
            out, err := loadFromDB(ctx)
            if err != nil {
                return nil, err
            }
            b, _ := json.Marshal(out)
            rdb.Set(ctx, cacheKey, b, 60*time.Second)
            return out, nil
        }
        time.Sleep(20 * time.Millisecond)   // không lấy được lock → chờ rồi thử lại
    }
    return nil, errors.New("timeout chờ lock")
}

// Chỉ xoá lock nếu NÓ VẪN LÀ CỦA MÌNH — compare-and-delete phải atomic
var releaseScript = redis.NewScript(`
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    end
    return 0
`)

func releaseLock(ctx context.Context, key, token string) {
    releaseScript.Run(ctx, rdb, []string{key}, token)
}
```

**Ba bẫy của distributed lock:**

1. **TTL của lock phải dài hơn thời gian query.** Nếu query mất 7 giây mà lock TTL 5 giây, lock tự hết hạn giữa chừng và instance thứ hai cũng vào — thành hai query. Tệ hơn: instance đầu xong việc rồi `DEL` **lock của instance thứ hai**. Đó là lý do phải compare-and-delete bằng Lua như trên, không `DEL` mù quáng.
2. **Phải có đường thoát.** Vòng lặp trên giới hạn 50 lần × 20ms = 1 giây. Không có giới hạn thì một lock kẹt sẽ treo toàn bộ request.
3. **Không cần Redlock ở đây.** Thuật toán Redlock sinh ra cho các bài toán cần *tính đúng đắn*. Với cache stampede, lock chỉ là **tối ưu hoá** — trường hợp xấu nhất khi lock hỏng là chạy thừa một query, hoàn toàn vô hại. Đừng trả giá phức tạp cho mức bảo đảm mà bài toán không cần.

> 💡 **Kết hợp cả hai là tốt nhất**: `singleflight` gộp trong tiến trình (rẻ, không round-trip), rồi mới tới Redis lock cho phần liên tiến trình. Benchmark cột cuối chính là cách này — 100 request/instance gộp xuống 1 lời gọi lock mỗi instance, rồi lock gộp 4 instance xuống 1 query.

---

# Cách sửa 3 — Giữ kết quả cũ trong lúc làm mới (SWR)

Với danh sách sản phẩm bán chạy, người xem **trễ vài giây thường không phải vấn đề lớn**. Vậy tại sao phải bắt họ chờ?

Ý tưởng: thay vì để Redis tự xoá sạch dữ liệu, ứng dụng lưu một **mốc thời gian "cần làm mới"** ngay trong giá trị cache, và đặt TTL lưu trữ trên Redis **dài hơn** mốc đó.

```
t=0 ────────── t=10s ──────────────────── t=60s
     còn tươi    │   CÒN DÙNG ĐƯỢC         │  Redis xoá
     trả ngay    │   trả ngay + refresh ngầm │
              refresh_at                  TTL
```

```go
type envelope struct {
    Data        []Row `json:"data"`
    RefreshAtMs int64 `json:"refresh_at_ms"`   // mốc LOGIC, nằm trong giá trị
}

func GetBestSellersSWR(ctx context.Context) ([]Row, error) {
    raw, err := rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        var e envelope
        if json.Unmarshal([]byte(raw), &e) == nil {
            if time.Now().UnixMilli() < e.RefreshAtMs {
                return e.Data, nil                      // còn tươi
            }
            // Quá hạn LOGIC nhưng còn trong TTL:
            // trả dữ liệu cũ NGAY, đồng thời refresh ngầm.
            // singleflight ở đây để không sinh ra 200 tác vụ nền.
            go swrSF.Do(cacheKey, func() (interface{}, error) {
                bg, cancel := context.WithTimeout(context.Background(), 5*time.Second)
                defer cancel()
                fresh, err := loadFromDB(bg)
                if err == nil {
                    writeEnvelope(bg, fresh, 10*time.Second, 60*time.Second)
                }
                return nil, err
            })
            return e.Data, nil
        }
    }
    // Cache rỗng hoàn toàn (cold start) → vẫn phải chặn stampede
    return loadWithSingleflight(ctx)
}
```

Kết quả đo — đây là con số ấn tượng nhất của cả bài:

| Chiến lược | Truy vấn DB | p50 | p99 |
|---|---:|---:|---:|
| Naive | 200 | 1.780 ms | 3.437 ms |
| singleflight | 1 | 65,7 ms | 65,9 ms |
| **Stale-while-revalidate** | **1** (chạy ngầm) | **1,6 ms** | **2,1 ms** |

SWR nhanh hơn `singleflight` **40 lần** nữa, vì **không ai phải chờ database cả** — kể cả request đầu tiên. Người dùng nhận dữ liệu cũ vài giây, việc nạp mới diễn ra ở nền.

**Ba điều kiện bắt buộc khi dùng SWR:**

1. **Giới hạn tuổi tối đa của dữ liệu cũ** — chính là TTL cứng trên Redis. Nếu backend nạp lỗi liên tục, TTL đảm bảo dữ liệu không cũ vô hạn.
2. **Giới hạn số tác vụ làm mới** — nếu không, mỗi request quá hạn lại sinh một goroutine, và bạn đổi stampede ở DB lấy stampede ở goroutine. Đây là lý do có `singleflight` bọc quanh tác vụ nền trong code trên.
3. **Dùng `context.Background()` cho tác vụ nền**, không dùng context của request. Request kết thúc là context bị huỷ, tác vụ refresh chết giữa chừng, và lần sau lại refresh — vĩnh viễn không bao giờ nạp xong.

**Không dùng SWR khi** dữ liệu cũ gây hậu quả thật: số dư tài khoản, tồn kho lúc checkout, giá đang khuyến mãi. Với "top 10 bán chạy" thì cũ 10 giây là vô hại; với "còn mấy vé" thì không.

---

# Cách sửa 4 — Làm mới sớm theo xác suất (XFetch)

Có một cách thanh lịch hơn SWR, ít được biết tới: thay vì đợi tới mốc làm mới rồi mới hành động, **mỗi request tự quyết định theo xác suất** xem có nên nạp lại sớm hay không. Càng gần hạn, xác suất càng cao.

Công thức đến từ bài báo [*Optimal Probabilistic Cache Stampede Prevention*](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf) (Vattani, Chierichetti, Lowenstein — VLDB 2015):

```
nếu   now − delta × beta × ln(rand())   ≥   expiry   →   nạp lại ngay
```

- `delta` = thời gian lượt nạp **trước** mất (query càng chậm thì càng nên nạp sớm)
- `beta` = hệ số điều chỉnh, mặc định `1.0`
- `rand()` = số ngẫu nhiên trong `(0, 1]`

```go
type xfEntry struct {
    Data     []Row `json:"data"`
    DeltaMs  int64 `json:"delta_ms"`    // lượt nạp trước mất bao lâu
    ExpiryMs int64 `json:"expiry_ms"`
}

func shouldRefreshEarly(e xfEntry, beta float64) bool {
    delta := float64(e.DeltaMs)
    early := delta * beta * -math.Log(rand.Float64())
    return float64(time.Now().UnixMilli())+early >= float64(e.ExpiryMs)
}
```

Mô phỏng: 1 key, TTL 2 giây, chi phí nạp lại 100 ms, 200 request/giây trong 20 giây (~4.000 request):

| Chiến lược | Số lần tính lại | Request **phải chờ** DB |
|---|---:|---:|
| TTL thường (đợi hết hạn mới nạp) | 9 | **184** |
| XFetch `beta=0.5` | 10 | 3 |
| **XFetch `beta=1.0`** | **11** | **0** |
| XFetch `beta=2.0` | 16 | 0 |

Đọc bảng này: trả thêm **2 lượt tính lại** (9 → 11, tức +22%) để đổi lấy **184 → 0 request phải chờ**. Và `beta=2.0` cho thấy đánh đổi theo hướng ngược lại: nạp lại nhiều hơn 78% mà không mua thêm được gì, vì đã về 0 rồi.

**So với SWR**, XFetch có hai ưu điểm: không bao giờ trả dữ liệu đã quá hạn (nó nạp lại *trước* khi hết hạn), và tự điều chỉnh theo chi phí thật của query. Nhược điểm: khó giải thích cho người mới đọc code hơn, và cần lưu thêm `delta` vào cache.

---

# Cache avalanche: TTL phải có jitter

Bài toán khác, cách sửa khác. Khi bạn nạp hàng loạt key cùng một lúc — sau deploy, sau restart, hoặc trong một job warm-up — và tất cả dùng **cùng một TTL**, thì chúng cũng sẽ **hết hạn cùng một lúc**.

Đo thật với 5.000 key:

| Cách đặt TTL | Số giây khác nhau mà key hết hạn | Giây đông nhất |
|---|---:|---:|
| `TTL = 300s` cố định | **1** | **5.000 key** |
| `TTL = 300s ± jitter 10%` | **61** | **105 key** |

Cùng một lượng dữ liệu, nhưng đỉnh tải giảm **47 lần**. Sửa chỉ một dòng:

```go
func ttlWithJitter(base time.Duration, pct float64) time.Duration {
    // pct = 0.1 → TTL nằm trong khoảng [90%, 110%] của base
    delta := (rand.Float64()*2 - 1) * pct * float64(base)
    return base + time.Duration(delta)
}

rdb.Set(ctx, key, val, ttlWithJitter(5*time.Minute, 0.1))
```

Đây là loại sửa đổi rẻ nhất trong cả bài: một hàm bốn dòng, không thêm phụ thuộc, không thêm trạng thái. Nếu hôm nay bạn chỉ làm được một việc sau khi đọc bài này, hãy làm việc này.

---

# Chọn cách nào?

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ch-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ch-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="560" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Thang leo: chỉ leo lên bậc tiếp theo khi bậc hiện tại không đủ</text>
<rect x="40" y="48" width="800" height="58" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#ch-sh)"/>
<text x="62" y="72" font-size="13" font-weight="800" fill="#134E4A">Bậc 0 — TTL + jitter</text>
<text x="62" y="92" font-size="11" fill="#0F766E">4 dòng code, không phụ thuộc, không trạng thái. Chống avalanche. LÀM CHO MỌI KEY, không có ngoại lệ.</text>
<rect x="700" y="60" width="124" height="34" rx="8" fill="#0D9488"/>
<text x="762" y="82" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">luôn làm</text>
<rect x="40" y="122" width="800" height="76" rx="10" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2.5" filter="url(#ch-sh)"/>
<text x="62" y="146" font-size="13" font-weight="800" fill="#134E4A">Bậc 1 — singleflight (hoặc Map&lt;key, Promise&gt;)</text>
<text x="62" y="166" font-size="11" fill="#0F766E">Gộp các lượt nạp trùng trong cùng tiến trình. ~10 dòng, không thêm hạ tầng, không thêm round-trip.</text>
<text x="62" y="185" font-size="11" font-weight="700" fill="#0F766E">Đo được: 200 query → 1. p99 từ 3.437 ms xuống 65,9 ms.</text>
<rect x="700" y="142" width="124" height="34" rx="8" fill="#14B8A6"/>
<text x="762" y="164" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">mặc định nên có</text>
<rect x="40" y="214" width="800" height="76" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5" filter="url(#ch-sh)"/>
<text x="62" y="238" font-size="13" font-weight="800" fill="#7C4A03">Bậc 2 — Stale-while-revalidate</text>
<text x="62" y="258" font-size="11" fill="#92400E">Trả dữ liệu cũ ngay, nạp mới ở nền. Chỉ dùng khi dữ liệu cũ vài giây là VÔ HẠI.</text>
<text x="62" y="277" font-size="11" font-weight="700" fill="#92400E">Đo được: p99 còn 2,1 ms — không ai phải chờ DB, kể cả request đầu tiên.</text>
<rect x="700" y="234" width="124" height="34" rx="8" fill="#F5A623"/>
<text x="762" y="256" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">khi chịu được cũ</text>
<rect x="40" y="306" width="800" height="76" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#ch-sh)"/>
<text x="62" y="330" font-size="13" font-weight="800" fill="#3730A3">Bậc 3 — Probabilistic early expiration (XFetch)</text>
<text x="62" y="350" font-size="11" fill="#4338CA">Nạp lại sớm theo xác suất, trước khi hết hạn. Không bao giờ trả dữ liệu quá hạn.</text>
<text x="62" y="369" font-size="11" font-weight="700" fill="#4338CA">Đo được: +22% số lần tính lại, đổi lấy 184 → 0 request phải chờ.</text>
<rect x="700" y="326" width="124" height="34" rx="8" fill="#6366F1"/>
<text x="762" y="348" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">khi cần tươi</text>
<rect x="40" y="398" width="800" height="76" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#ch-sh)"/>
<text x="62" y="422" font-size="13" font-weight="800" fill="#9F1239">Bậc 4 — Distributed lock (Redis SET NX + Lua release)</text>
<text x="62" y="442" font-size="11" fill="#BE123C">Gộp qua NHIỀU instance. Thêm round-trip, thêm trường hợp lỗi, thêm thứ phải debug lúc 3 giờ sáng.</text>
<text x="62" y="461" font-size="11" font-weight="700" fill="#BE123C">Chỉ đáng khi query THẬT SỰ nặng — 4 instance = 4 query, thường đã đủ tốt rồi.</text>
<rect x="700" y="418" width="124" height="34" rx="8" fill="#FF6B6B"/>
<text x="762" y="440" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">cân nhắc kỹ</text>
<rect x="40" y="490" width="800" height="56" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="62" y="514" font-size="12" font-weight="800" fill="#374151">Và dù chọn bậc nào, vẫn cần hai thứ này ở tầng dưới:</text>
<text x="62" y="536" font-size="11" fill="#57534E">⏱ Timeout cho mọi lời gọi DB (không có timeout = chờ vô hạn)   ·   🔌 Circuit breaker để ngắt khi DB đã quá tải</text>
</svg>
</div>

Bảng tổng hợp toàn bộ số đo — 200 request đồng thời, cùng một key vừa hết hạn:

| Chiến lược | Truy vấn DB | p50 | p95 | p99 |
|---|---:|---:|---:|---:|
| Naive cache-aside | **200** | 1.780 ms | 3.394 ms | **3.437 ms** |
| singleflight (1 instance) | 1 | 65,7 ms | 65,9 ms | 65,9 ms |
| singleflight × 4 instance | 4 | 97,4 ms | 102,7 ms | 102,7 ms |
| singleflight + Redis lock (4 instance) | 1 | 88,1 ms | 88,6 ms | 88,7 ms |
| **Stale-while-revalidate** | 1 (nền) | **1,6 ms** | **2,0 ms** | **2,1 ms** |

---

# Hai thứ phải có ở tầng dưới

Mọi chiến lược ở trên đều giả định database **cuối cùng cũng trả lời**. Khi DB đã quá tải thật, cần thêm hai lớp bảo vệ:

**1. Timeout cho mọi lời gọi.** Không có timeout nghĩa là một request có thể giữ một connection pool vô hạn.

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
rows, err := db.QueryContext(ctx, query)
```

**2. Circuit breaker.** Khi DB đã có dấu hiệu quá tải, đừng cho tất cả request cùng dồn thẳng xuống — ngắt mạch và trả lỗi nhanh (hoặc trả dữ liệu cũ) sẽ giúp DB hồi phục nhanh hơn là tiếp tục dồn tải.

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "bestsellers-db",
    MaxRequests: 3,
    Timeout:     10 * time.Second,
    ReadyToTrip: func(c gobreaker.Counts) bool {
        return c.ConsecutiveFailures > 5
    },
})

res, err := cb.Execute(func() (interface{}, error) { return loadFromDB(ctx) })
```

Kết hợp với SWR thì rất đẹp: breaker mở → trả dữ liệu cũ thay vì trả lỗi. Người dùng thấy dữ liệu hơi cũ, còn hơn thấy trang trắng.

---

# Còn một bước nữa: fine-grained caching

Trong ví dụ này, cả A, B và C đều chạy đúng đoạn code `GET` → miss → đọc DB → `SET` lại. Khi thêm cache vào app, cần xử lý cả trường hợp chúng **cùng miss một key**.

Nhưng thực tế còn cần cân nhắc **tách cache theo tần suất đọc và thay đổi**. Ví dụ với "top 10 bán chạy":

```
❌ Một key chứa tất cả:
   bestsellers:week → [{id, tên, giá, ảnh, tồn kho}, ...]
   → Chỉ cần giá của 1 sản phẩm đổi là phải nạp lại CẢ KHỐI (query tổng hợp 77ms)

✅ Tách theo nhịp thay đổi:
   bestsellers:week:ids → [102, 47, 891, ...]   TTL 60s   (đổi chậm, query nặng)
   product:102          → {tên, giá, ảnh}        TTL 300s  (đổi nhanh, query nhẹ)
```

Giá sản phẩm đổi → chỉ invalidate `product:102`, không đụng tới danh sách ID. Danh sách bán chạy đổi → chỉ nạp lại query tổng hợp, không nạp lại chi tiết 10 sản phẩm.

Lưu ý: **những key được đọc nhiều vẫn cần gộp refill** — tách nhỏ không tự động giải quyết stampede, nó chỉ làm cửa sổ chết hẹp lại và rẻ hơn. Phần chia cache chi tiết sẽ nằm ở một bài khác trong series.

---

# Checklist

```
TRƯỚC KHI ĐƯA MỘT CACHE KEY MỚI LÊN PRODUCTION
□ TTL có jitter chưa? (chống avalanche — rẻ nhất, làm cho MỌI key)
□ Nếu 200 request cùng miss key này, chuyện gì xảy ra? Đã thử chưa?
□ Có gộp lượt nạp chưa? (singleflight / Map<key, Promise>)
□ Query nạp lại mất bao lâu? → đó chính là độ dài cửa sổ chết
□ Dữ liệu cũ vài giây có gây hậu quả không? Nếu không → dùng SWR
□ Lời gọi DB đã có timeout chưa?
□ Tác vụ refresh nền có dùng context.Background() không? (không dùng ctx của request)
□ Tác vụ refresh nền có bị giới hạn số lượng không?
□ Key không tồn tại có được cache âm không? (chống penetration)

KHI CÓ SỰ CỐ "REDIS XANH MÀ APP CHẬM"
□ Đếm số lần chạy của TỪNG câu query, không chỉ tổng QPS
□ Đo thời gian CHỜ lấy connection từ pool (Go: db.Stats().WaitDuration)
□ Xem có nhiều query giống hệt nhau chạy đồng thời không
□ Kiểm tra có key nóng nào vừa hết hạn đúng lúc sự cố bắt đầu không
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| Cache stampede là gì? | Nhiều request cùng miss **một** key, cùng chạy lại **một** truy vấn nặng |
| Vì sao code cache-aside chuẩn vẫn dính? | `GET` chỉ nói "có / không có", **không nói "đang có người đi lấy"** |
| Vì sao Redis vẫn xanh mà app chết? | Nút thắt là **connection pool của backend**, không phải Redis |
| Vì sao cả app chết chứ không chỉ 1 endpoint? | Mọi API dùng chung pool đều phải xếp hàng sau các query trùng lặp |
| Sửa rẻ nhất? | **TTL + jitter** (4 dòng) rồi **singleflight** (~10 dòng) |
| singleflight đủ chưa? | Chỉ trong **một tiến trình**. 4 instance = 4 query — thường vẫn chấp nhận được |
| `sync.Once` thay được không? | **Không** — chỉ chạy một lần duy nhất, không nạp lại theo TTL |
| Nhanh nhất? | **SWR** — p99 2,1 ms, không ai chờ DB. Nhưng phải chịu được dữ liệu cũ |
| Không chịu được dữ liệu cũ? | **XFetch** — nạp lại sớm theo xác suất, +22% query đổi lấy 0 request phải chờ |
| Có cần Redlock không? | **Không.** Lock ở đây là tối ưu hoá, hỏng thì cùng lắm chạy thừa 1 query |

---

# Tham khảo

- [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) — tài liệu chính thức, đọc kỹ phần `DoChan` và `Forget`
- [Optimal Probabilistic Cache Stampede Prevention](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf) — Vattani et al., VLDB 2015 (nguồn của XFetch)
- [Redis — Distributed locks with Redis](https://redis.io/docs/latest/develop/use-cases/patterns/distributed-locks/) — kèm phần bàn về khi nào **không** cần Redlock
- [`sony/gobreaker`](https://github.com/sony/gobreaker) — circuit breaker cho Go
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc của series
- Bài trước trong series: [System Design P01: Redis Không Chỉ Có GET/SET](/posts/system-design-p01-redis/)
