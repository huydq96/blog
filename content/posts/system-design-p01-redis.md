+++
date = '2026-10-16T09:00:00+07:00'
draft = false
slug = 'system-design-p01-redis'
title = 'System Design P01: Redis Không Chỉ Có GET/SET'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "redis", "caching", "data-structures", "performance"]
description = 'Dùng Redis như memcached là bỏ qua phần giá trị nhất của nó. Đo thật sự khác biệt giữa nhét JSON vào String và dùng cấu trúc dữ liệu native.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, ví dụ, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên Redis 7.4.2 local.*

# Tổng quan

Có một thói quen rất phổ biến khi mới làm quen với Redis: cứ có dữ liệu gì từ database hoặc API trả về, ta đem serialize thành chuỗi JSON rồi `SET` vào một key. Khi cần đọc thì `GET` nguyên chuỗi đó ra, parse ngược lại thành object trong code.

Cách làm này tiện và dễ hiểu. Object trong backend đang có cấu trúc thế nào thì nhét thẳng vào Redis như thế, gần như không phải suy nghĩ xem phải mô hình hoá lại dữ liệu ra sao. Nếu mục tiêu chỉ là dựng một tầng cache cơ bản cho các response đọc nguyên khối, `GET` và `SET` đáp ứng hoàn toàn ổn.

Nhưng nếu dừng lại ở đó, ta rất dễ bỏ qua phần giá trị nhất của Redis.

Vấn đề bắt đầu từ **cái tên**. Việc gắn chặt chữ "cache" với Redis làm nhiều người vô tình coi nó như một cái memcached mở rộng, hoặc một ổ cứng chạy trên RAM — nơi dữ liệu chỉ được ném vào rồi rút ra nguyên vẹn từng cục.

Bản chất của Redis là một **remote data structure store**. Điểm mạnh của nó không chỉ nằm ở việc dữ liệu nằm trên RAM để đọc nhanh, mà nằm ở chỗ nó **cung cấp sẵn các cấu trúc dữ liệu kèm theo những thao tác tính toán thực hiện ngay tại server**.

Đó là một khác biệt về bản chất, không phải về hiệu năng:

| | **Memcached** | **Redis** |
|---|---|---|
| Mô hình dữ liệu | key → blob | key → **cấu trúc dữ liệu** |
| Thao tác | `get` / `set` / `delete` | ~250 lệnh, nhiều lệnh **tính toán tại server** |
| Việc tính toán xảy ra ở | **App** | **Redis** (hoặc app, tuỳ bạn chọn) |
| Dữ liệu truyền qua network | Toàn bộ giá trị | Đúng phần bạn cần |

Cột cuối là chỗ tạo ra khác biệt lớn nhất, và cũng là chỗ dễ bị bỏ qua nhất. Phần còn lại của bài là đo xem "đúng phần bạn cần" đáng giá bao nhiêu.

---

# Ví dụ 1 — Bảng xếp hạng điểm tuần

Hãy thử hình dung một bài toán quen thuộc: tính năng **bảng xếp hạng điểm tuần** trong một ứng dụng học tập. Giao diện chỉ cần hiển thị **top 10**.

## Cách làm theo thói quen: JSON array trong String

Gom toàn bộ danh sách gồm user ID và điểm thành một JSON array, rồi `SET` vào một Redis String.

```go
// ĐỌC: muốn 10 dòng, nhưng phải kéo cả 10.000 dòng về
raw, _ := rdb.Get(ctx, "leaderboard:week").Result()

var all []Entry
json.Unmarshal([]byte(raw), &all)           // parse 10.000 phần tử

sort.Slice(all, func(a, b int) bool {       // sort 10.000 phần tử
    return all[a].Score > all[b].Score
})
top10 := all[:10]                           // ...để dùng đúng 10
```

Ghi còn tệ hơn. Mỗi khi có ai đó hoàn thành bài học và được cộng điểm:

```go
// GHI: read-modify-write toàn bộ mảng, chỉ để sửa 1 con số
raw, _ := rdb.Get(ctx, "leaderboard:week").Result()
var all []Entry
json.Unmarshal([]byte(raw), &all)

for i := range all {
    if all[i].UserID == userID {
        all[i].Score += 10
        break
    }
}

newBlob, _ := json.Marshal(all)
rdb.Set(ctx, "leaderboard:week", newBlob, 0)
```

Nhìn qua thì hệ thống vẫn "có dùng Redis". Nhưng áp lực xử lý thực tế dồn hết về phía backend:

```
GET cả mảng qua network → parse JSON → sort trên CPU của app → cắt lấy kết quả
```

## Cách dùng đúng cấu trúc: Sorted Set

Mỗi user là một **member** gắn liền với một **score**. Redis giữ cấu trúc này **luôn ở trạng thái đã sắp xếp**.

```go
// GHI: 1 lệnh, 1 round-trip, Redis tự đặt lại vị trí trong cấu trúc
rdb.ZIncrBy(ctx, "leaderboard:week", 10, userID)

// ĐỌC: yêu cầu đúng khoảng dữ liệu cần, Redis trả về đúng 10 bản ghi
top10, _ := rdb.ZRevRangeWithScores(ctx, "leaderboard:week", 0, 9).Result()
```

Hết. Không `Unmarshal`, không `sort.Slice`, không read-modify-write.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 540" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="rd-arrR" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#FF6B6B"/></marker>
<marker id="rd-arrT" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="rd-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="540" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Lấy top 10 từ bảng xếp hạng 10.000 user — việc tính toán nằm ở đâu?</text>
<rect x="20" y="44" width="860" height="230" rx="14" fill="#FFF7F7" stroke="#FFC9CC" stroke-width="1.5"/>
<rect x="20" y="290" width="860" height="234" rx="14" fill="#F2FCFB" stroke="#9BE5DC" stroke-width="1.5"/>
<text x="42" y="68" font-size="14" font-weight="800" fill="#9F1239">❌ JSON array trong String — app làm hết</text>
<text x="42" y="314" font-size="14" font-weight="800" fill="#134E4A">✅ Sorted Set — Redis làm hết</text>
<rect x="640" y="86" width="210" height="130" rx="10" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2" filter="url(#rd-sh)"/>
<rect x="640" y="86" width="210" height="26" rx="10" fill="#DC2626"/>
<rect x="640" y="102" width="210" height="10" fill="#DC2626"/>
<text x="745" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">REDIS — String</text>
<rect x="654" y="122" width="182" height="82" rx="6" fill="#FFE9EC"/>
<text x="745" y="142" text-anchor="middle" font-size="10" font-family="monospace" fill="#9F1239">"[{user_00001,842},</text>
<text x="745" y="158" text-anchor="middle" font-size="10" font-family="monospace" fill="#9F1239">{user_00002,13}, ...</text>
<text x="745" y="174" text-anchor="middle" font-size="10" font-family="monospace" fill="#9F1239">...10.000 phần tử]"</text>
<text x="745" y="195" text-anchor="middle" font-size="10" font-weight="700" fill="#BE123C">1 blob — không truy vấn được</text>
<rect x="50" y="86" width="230" height="130" rx="10" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2" filter="url(#rd-sh)"/>
<rect x="50" y="86" width="230" height="26" rx="10" fill="#57534E"/>
<rect x="50" y="102" width="230" height="10" fill="#57534E"/>
<text x="165" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">BACKEND (app)</text>
<rect x="64" y="120" width="202" height="20" rx="5" fill="#FFE9EC"/>
<rect x="64" y="144" width="202" height="20" rx="5" fill="#FFE9EC"/>
<rect x="64" y="168" width="202" height="20" rx="5" fill="#FFE9EC"/>
<rect x="64" y="192" width="202" height="20" rx="5" fill="#FFD9DE"/>
<text x="165" y="134" text-anchor="middle" font-size="10" fill="#9F1239">1. parse JSON 10.000 phần tử</text>
<text x="165" y="158" text-anchor="middle" font-size="10" fill="#9F1239">2. sort 10.000 phần tử trên CPU</text>
<text x="165" y="182" text-anchor="middle" font-size="10" fill="#9F1239">3. cắt lấy 10 phần tử đầu</text>
<text x="165" y="206" text-anchor="middle" font-size="10" font-weight="700" fill="#BE123C">9.990 phần tử bị bỏ đi</text>
<line x1="636" y1="130" x2="286" y2="130" stroke="#FF6B6B" stroke-width="2.5" marker-end="url(#rd-arrR)"/>
<rect x="356" y="104" width="210" height="22" rx="6" fill="#FFE9EC"/>
<text x="461" y="120" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">388.897 bytes qua network</text>
<line x1="286" y1="178" x2="636" y2="178" stroke="#D6D3D1" stroke-width="1.6" stroke-dasharray="4 3"/>
<text x="461" y="172" text-anchor="middle" font-size="10" fill="#78716C">GET leaderboard:week</text>
<rect x="300" y="228" width="300" height="34" rx="8" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="450" y="250" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">p50 = 7,49 ms   ·   p99 = 11,84 ms</text>
<rect x="640" y="332" width="210" height="130" rx="10" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#rd-sh)"/>
<rect x="640" y="332" width="210" height="26" rx="10" fill="#0D9488"/>
<rect x="640" y="348" width="210" height="10" fill="#0D9488"/>
<text x="745" y="350" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">REDIS — Sorted Set</text>
<rect x="654" y="366" width="182" height="16" rx="4" fill="#CCFBF1"/>
<rect x="654" y="386" width="182" height="16" rx="4" fill="#CCFBF1"/>
<rect x="654" y="406" width="182" height="16" rx="4" fill="#CCFBF1"/>
<rect x="654" y="426" width="182" height="16" rx="4" fill="#F5F5F4"/>
<text x="745" y="378" text-anchor="middle" font-size="9" font-family="monospace" fill="#0F766E">user_04218 → 99.912</text>
<text x="745" y="398" text-anchor="middle" font-size="9" font-family="monospace" fill="#0F766E">user_00877 → 99.840</text>
<text x="745" y="418" text-anchor="middle" font-size="9" font-family="monospace" fill="#0F766E">user_07031 → 99.755</text>
<text x="745" y="438" text-anchor="middle" font-size="9" fill="#A8A29E">... luôn đã sắp sẵn</text>
<text x="745" y="455" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">truy vấn được theo rank</text>
<rect x="50" y="332" width="230" height="130" rx="10" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#rd-sh)"/>
<rect x="50" y="332" width="230" height="26" rx="10" fill="#57534E"/>
<rect x="50" y="348" width="230" height="10" fill="#57534E"/>
<text x="165" y="350" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">BACKEND (app)</text>
<rect x="64" y="372" width="202" height="24" rx="5" fill="#CCFBF1"/>
<text x="165" y="388" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">1. nhận đúng 10 bản ghi</text>
<text x="165" y="416" text-anchor="middle" font-size="10" fill="#0F766E">Không parse. Không sort.</text>
<text x="165" y="434" text-anchor="middle" font-size="10" fill="#0F766E">Không read-modify-write.</text>
<text x="165" y="452" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">CPU app gần như rảnh</text>
<line x1="286" y1="376" x2="636" y2="376" stroke="#0D9488" stroke-width="2" marker-end="url(#rd-arrT)"/>
<text x="461" y="370" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">ZREVRANGE leaderboard:week 0 9</text>
<line x1="636" y1="424" x2="286" y2="424" stroke="#0D9488" stroke-width="2.5" marker-end="url(#rd-arrT)"/>
<rect x="386" y="398" width="150" height="22" rx="6" fill="#CCFBF1"/>
<text x="461" y="414" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">180 bytes</text>
<rect x="300" y="478" width="300" height="34" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="450" y="500" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">p50 = 53 µs   ·   p99 = 225 µs</text>
<rect x="628" y="478" width="252" height="34" rx="8" fill="#0D9488"/>
<text x="754" y="500" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">nhanh hơn 140× · nhẹ hơn 2.160×</text>
</svg>
</div>

## Số đo thật

Đo trên Redis 7.4.2 local, 10.000 user, go-redis v9, 200 lần mỗi phép, lấy p50/p99:

| Thao tác | JSON trong String | Sorted Set | Chênh lệch |
|---|---:|---:|---:|
| **Đọc top 10** — p50 | 7,49 ms | **53 µs** | **141×** |
| **Đọc top 10** — p99 | 11,84 ms | **225 µs** | 53× |
| Bytes qua network | 388.897 | **180** | **2.160×** |
| **Cộng 10 điểm cho 1 user** — p50 | 8,25 ms | **80 µs** | **104×** |
| **Cộng 10 điểm cho 1 user** — p99 | 13,07 ms | **365 µs** | 36× |

Lúc dữ liệu còn vài chục phần tử, độ trễ chưa lộ rõ — và đó chính là lý do thói quen này sống sót qua giai đoạn đầu của dự án. Nhưng khi danh sách dài ra, lượng dữ liệu trung chuyển qua network tăng lên vô ích, backend tốn thêm chu kỳ CPU chỉ để làm những việc lặp đi lặp lại.

> ⚠️ Còn một vấn đề nghiêm trọng hơn cả hiệu năng mà bảng trên không thể hiện: **read-modify-write không atomic**. Hai user cùng được cộng điểm đồng thời, cả hai cùng `GET` mảng cũ, cùng sửa, cùng `SET` đè — **một trong hai lần cộng điểm biến mất**. Không có lỗi, không có log, chỉ là điểm bị mất. `ZINCRBY` là một lệnh đơn, atomic theo thiết kế, nên bài toán này không tồn tại. Đây là lý do mạnh nhất để đổi sang cấu trúc native, mạnh hơn cả 140×.

---

# Ví dụ 2 — Danh sách bài học đã lưu

Bài toán: người dùng lưu các bài học yêu thích. Mỗi lần mở một bài, giao diện cần biết **bài này đã được lưu chưa** để tô đậm icon trái tim.

```go
// ❌ JSON array: kéo cả danh sách về rồi duyệt tìm
raw, _ := rdb.Get(ctx, "saved:user_42").Result()
var ids []string
json.Unmarshal([]byte(raw), &ids)
found := slices.Contains(ids, lessonID)

// Muốn thêm mới còn phải TỰ kiểm tra trùng lặp:
if !found {
    ids = append(ids, lessonID)
    nb, _ := json.Marshal(ids)
    rdb.Set(ctx, "saved:user_42", nb, 0)   // lại read-modify-write, lại không atomic
}
```

```go
// ✅ Set: thêm, xoá, kiểm tra tồn tại — mỗi thứ 1 lệnh, atomic
saved, _ := rdb.SIsMember(ctx, "saved:user_42", lessonID).Result()
rdb.SAdd(ctx, "saved:user_42", lessonID)       // Set tự khử trùng, không cần check trước
rdb.SRem(ctx, "saved:user_42", lessonID)
count, _ := rdb.SCard(ctx, "saved:user_42").Result()   // đếm không cần kéo dữ liệu về
```

Đo với danh sách 500 ID:

| Thao tác | JSON array | Set | Chênh lệch |
|---|---:|---:|---:|
| Kiểm tra 1 ID có trong danh sách — p50 | 251 µs | **89 µs** | 2,8× |
| Bytes qua network | 7.001 | **~1** | 7.000× |

Chênh lệch độ trễ ở đây khiêm tốn hơn ví dụ leaderboard (2,8× chứ không phải 140×) vì 500 phần tử còn nhỏ. Nhưng hãy để ý **cột bytes**: `SISMEMBER` trả về đúng một bit thông tin — `0` hoặc `1`. Đó là thứ mở rộng theo kích thước dữ liệu tốt nhất: danh sách có 500 hay 500.000 phần tử, câu trả lời vẫn là 1 byte.

Và Set còn cho những thứ JSON array không cho được — các **phép toán tập hợp chạy tại server**:

```go
// "Những bài học mà CẢ HAI chúng tôi đều đã lưu" — 1 lệnh, không kéo gì về app
common, _ := rdb.SInter(ctx, "saved:user_42", "saved:user_99").Result()

// "Bài học tôi lưu mà bạn chưa lưu" — gợi ý cho partner
diff, _ := rdb.SDiff(ctx, "saved:user_42", "saved:user_99").Result()
```

Làm việc này với JSON array nghĩa là kéo **cả hai** danh sách về app rồi tự viết vòng lặp giao/hiệu. Với Set, nó là một lệnh.

---

# Chọn cấu trúc nào?

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 580" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="cat-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="580" fill="#FFFDF7" rx="16"/>
<text x="450" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Bộ cấu trúc dữ liệu của Redis — và việc nó sinh ra để làm</text>
<rect x="24" y="48" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="311" y="48" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="598" y="48" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="24" y="216" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#F5A623" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="311" y="216" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#F5A623" stroke-width="2.5" filter="url(#cat-sh)"/>
<rect x="598" y="216" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#F5A623" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="24" y="384" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#6366F1" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="311" y="384" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#6366F1" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="598" y="384" width="278" height="158" rx="12" fill="#FFFFFF" stroke="#6366F1" stroke-width="2" filter="url(#cat-sh)"/>
<rect x="24" y="48" width="278" height="30" rx="12" fill="#0D9488"/>
<rect x="24" y="66" width="278" height="12" fill="#0D9488"/>
<rect x="311" y="48" width="278" height="30" rx="12" fill="#0D9488"/>
<rect x="311" y="66" width="278" height="12" fill="#0D9488"/>
<rect x="598" y="48" width="278" height="30" rx="12" fill="#0D9488"/>
<rect x="598" y="66" width="278" height="12" fill="#0D9488"/>
<rect x="24" y="216" width="278" height="30" rx="12" fill="#D97706"/>
<rect x="24" y="234" width="278" height="12" fill="#D97706"/>
<rect x="311" y="216" width="278" height="30" rx="12" fill="#D97706"/>
<rect x="311" y="234" width="278" height="12" fill="#D97706"/>
<rect x="598" y="216" width="278" height="30" rx="12" fill="#D97706"/>
<rect x="598" y="234" width="278" height="12" fill="#D97706"/>
<rect x="24" y="384" width="278" height="30" rx="12" fill="#6366F1"/>
<rect x="24" y="402" width="278" height="12" fill="#6366F1"/>
<rect x="311" y="384" width="278" height="30" rx="12" fill="#6366F1"/>
<rect x="311" y="402" width="278" height="12" fill="#6366F1"/>
<rect x="598" y="384" width="278" height="30" rx="12" fill="#6366F1"/>
<rect x="598" y="402" width="278" height="12" fill="#6366F1"/>
<text x="163" y="69" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">STRING</text>
<text x="450" y="69" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">HASH</text>
<text x="737" y="69" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">LIST</text>
<text x="163" y="237" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">SET</text>
<text x="450" y="237" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">SORTED SET</text>
<text x="737" y="237" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">STREAM</text>
<text x="163" y="405" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">BITMAP</text>
<text x="450" y="405" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">HYPERLOGLOG</text>
<text x="737" y="405" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">GEO</text>
<text x="42" y="100" font-size="11" font-weight="700" fill="#134E4A">Cache response, session, token</text>
<text x="42" y="120" font-size="11" fill="#57534E">Blob đọc nguyên khối — vẫn hợp lệ!</text>
<text x="42" y="144" font-size="11" fill="#57534E">Số nguyên có sẵn INCR/DECR —</text>
<text x="42" y="162" font-size="11" fill="#57534E">không cần GET rồi SET lại.</text>
<text x="42" y="190" font-size="10" font-family="monospace" fill="#0F766E">SET · GET · INCR · INCRBY · SETEX</text>
<text x="329" y="100" font-size="11" font-weight="700" fill="#134E4A">Object dạng phẳng</text>
<text x="329" y="120" font-size="11" fill="#57534E">Profile user, giỏ hàng theo mã SP.</text>
<text x="329" y="144" font-size="11" fill="#57534E">Đọc/ghi TỪNG FIELD mà không</text>
<text x="329" y="162" font-size="11" fill="#57534E">phải đụng tới cả object.</text>
<text x="329" y="190" font-size="10" font-family="monospace" fill="#0F766E">HSET · HGET · HINCRBY · HGETALL</text>
<text x="616" y="100" font-size="11" font-weight="700" fill="#134E4A">Giữ THỨ TỰ CHÈN</text>
<text x="616" y="120" font-size="11" fill="#57534E">Lịch sử truy cập gần nhất</text>
<text x="616" y="138" font-size="11" fill="#57534E">(LPUSH + LTRIM giới hạn độ dài),</text>
<text x="616" y="156" font-size="11" fill="#57534E">hàng đợi đơn giản vào đầu ra đuôi.</text>
<text x="616" y="190" font-size="10" font-family="monospace" fill="#0F766E">LPUSH · RPOP · LRANGE · LTRIM</text>
<text x="42" y="268" font-size="11" font-weight="700" fill="#7C4A03">Tập hợp KHÔNG TRÙNG LẶP</text>
<text x="42" y="288" font-size="11" fill="#57534E">User đã like, bài viết đã lưu, tag.</text>
<text x="42" y="312" font-size="11" fill="#57534E">Phép giao/hợp/hiệu chạy tại server</text>
<text x="42" y="330" font-size="11" fill="#57534E">— không kéo 2 danh sách về app.</text>
<text x="42" y="358" font-size="10" font-family="monospace" fill="#92400E">SADD · SISMEMBER · SINTER · SCARD</text>
<text x="329" y="268" font-size="11" font-weight="700" fill="#7C4A03">Xếp hạng theo ĐIỂM SỐ</text>
<text x="329" y="288" font-size="11" fill="#57534E">Leaderboard, hàng đợi ưu tiên,</text>
<text x="329" y="306" font-size="11" fill="#57534E">chuỗi thời gian (score = timestamp),</text>
<text x="329" y="324" font-size="11" fill="#57534E">rate limit sliding window.</text>
<text x="329" y="358" font-size="10" font-family="monospace" fill="#92400E">ZADD · ZINCRBY · ZREVRANGE · ZRANK</text>
<text x="616" y="268" font-size="11" font-weight="700" fill="#7C4A03">Luồng sự kiện có ACK</text>
<text x="616" y="288" font-size="11" fill="#57534E">Nhiều worker cùng xử lý: có</text>
<text x="616" y="306" font-size="11" fill="#57534E">consumer group, danh sách pending,</text>
<text x="616" y="324" font-size="11" fill="#57534E">cơ chế xác nhận (ACK).</text>
<text x="616" y="358" font-size="10" font-family="monospace" fill="#92400E">XADD · XREADGROUP · XACK · XPENDING</text>
<text x="42" y="436" font-size="11" font-weight="700" fill="#3730A3">Cờ nhị phân theo chỉ số</text>
<text x="42" y="456" font-size="11" fill="#57534E">Điểm danh theo ngày, feature flag</text>
<text x="42" y="474" font-size="11" fill="#57534E">theo user_id. Cực kỳ tiết kiệm:</text>
<text x="42" y="492" font-size="11" fill="#57534E">1 triệu user = 125 KB.</text>
<text x="42" y="524" font-size="10" font-family="monospace" fill="#4338CA">SETBIT · GETBIT · BITCOUNT · BITOP</text>
<text x="329" y="436" font-size="11" font-weight="700" fill="#3730A3">ĐẾM số phần tử duy nhất</text>
<text x="329" y="456" font-size="11" fill="#57534E">Unique visitor theo ngày/tháng.</text>
<text x="329" y="480" font-size="11" fill="#57534E">Dung lượng CỐ ĐỊNH ~12 KB bất kể</text>
<text x="329" y="498" font-size="11" fill="#57534E">bao nhiêu phần tử — sai số ~0,81%.</text>
<text x="329" y="524" font-size="10" font-family="monospace" fill="#4338CA">PFADD · PFCOUNT · PFMERGE</text>
<text x="616" y="436" font-size="11" font-weight="700" fill="#3730A3">Toạ độ địa lý</text>
<text x="616" y="456" font-size="11" fill="#57534E">"Cửa hàng trong bán kính 5km",</text>
<text x="616" y="474" font-size="11" fill="#57534E">tài xế gần nhất. Thực chất là</text>
<text x="616" y="492" font-size="11" fill="#57534E">Sorted Set với score = geohash.</text>
<text x="616" y="524" font-size="10" font-family="monospace" fill="#4338CA">GEOADD · GEOSEARCH · GEODIST</text>
<rect x="24" y="552" width="852" height="20" rx="6" fill="#CCFBF1"/>
<text x="450" y="566" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Hàng xanh: đọc/ghi nguyên khối hoặc theo phần · Hàng cam: tính toán tại server · Hàng tím: cấu trúc chuyên dụng, tiết kiệm bộ nhớ</text>
</svg>
</div>

Quy tắc chọn, gọn lại thành một câu hỏi: **"tôi cần *làm gì* với dữ liệu này?"**

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 644" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="rf-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="rf-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="644" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cây quyết định: nhét JSON vào String, hay dùng cấu trúc native?</text>
<rect x="268" y="50" width="324" height="46" rx="10" fill="#374151" filter="url(#rf-sh)"/>
<text x="430" y="79" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">App có cần TÍNH TOÁN trên dữ liệu này không?</text>
<rect x="30" y="128" width="300" height="86" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#rf-sh)"/>
<text x="180" y="154" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">STRING (JSON blob)</text>
<text x="180" y="176" text-anchor="middle" font-size="11" fill="#0F766E">Đọc nguyên khối rồi trả thẳng cho client</text>
<text x="180" y="196" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">👉 Hoàn toàn hợp lệ. Đừng bẻ nhỏ vô ích.</text>
<rect x="400" y="128" width="430" height="52" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#rf-sh)"/>
<text x="615" y="151" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Tính toán đó là loại gì?</text>
<text x="615" y="169" text-anchor="middle" font-size="11" fill="#92400E">(sort / tìm / đếm / cộng dồn / lọc một phần)</text>
<rect x="400" y="206" width="430" height="42" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="1.8"/>
<rect x="400" y="256" width="430" height="42" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="1.8"/>
<rect x="400" y="306" width="430" height="42" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="1.8"/>
<rect x="400" y="356" width="430" height="42" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="1.8"/>
<rect x="400" y="406" width="430" height="42" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="1.8"/>
<text x="418" y="223" font-size="11" font-weight="700" fill="#7C4A03">Xếp hạng, lấy top-N, khoảng điểm</text>
<text x="418" y="240" font-size="11" fill="#92400E">leaderboard · priority queue · sliding window</text>
<text x="418" y="273" font-size="11" font-weight="700" fill="#7C4A03">Có tồn tại không? Giao/hợp/hiệu?</text>
<text x="418" y="290" font-size="11" fill="#92400E">đã like · đã lưu · tag · bạn chung</text>
<text x="418" y="323" font-size="11" font-weight="700" fill="#7C4A03">Đọc/ghi từng field của object</text>
<text x="418" y="340" font-size="11" fill="#92400E">profile · giỏ hàng · counter theo field</text>
<text x="418" y="373" font-size="11" font-weight="700" fill="#7C4A03">Giữ thứ tự chèn, giới hạn độ dài</text>
<text x="418" y="390" font-size="11" fill="#92400E">lịch sử gần nhất · queue đơn giản</text>
<text x="418" y="423" font-size="11" font-weight="700" fill="#7C4A03">Nhiều worker + cần ACK, retry</text>
<text x="418" y="440" font-size="11" fill="#92400E">event stream · job queue có bảo đảm</text>
<rect x="266" y="206" width="118" height="42" rx="9" fill="#D97706"/>
<rect x="266" y="256" width="118" height="42" rx="9" fill="#D97706"/>
<rect x="266" y="306" width="118" height="42" rx="9" fill="#D97706"/>
<rect x="266" y="356" width="118" height="42" rx="9" fill="#D97706"/>
<rect x="266" y="406" width="118" height="42" rx="9" fill="#D97706"/>
<text x="325" y="232" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">SORTED SET</text>
<text x="325" y="282" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">SET</text>
<text x="325" y="332" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">HASH</text>
<text x="325" y="382" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">LIST</text>
<text x="325" y="432" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">STREAM</text>
<line x1="430" y1="98" x2="430" y2="124" stroke="#57534E" stroke-width="2" marker-end="url(#rf-arr)"/>
<path d="M268,73 L180,73 L180,124" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#rf-arr)"/>
<path d="M592,73 L700,73 L700,124" stroke="#57534E" stroke-width="2" fill="none" marker-end="url(#rf-arr)"/>
<line x1="396" y1="227" x2="388" y2="227" stroke="#D97706" stroke-width="2" marker-end="url(#rf-arr)"/>
<line x1="396" y1="277" x2="388" y2="277" stroke="#D97706" stroke-width="2" marker-end="url(#rf-arr)"/>
<line x1="396" y1="327" x2="388" y2="327" stroke="#D97706" stroke-width="2" marker-end="url(#rf-arr)"/>
<line x1="396" y1="377" x2="388" y2="377" stroke="#D97706" stroke-width="2" marker-end="url(#rf-arr)"/>
<line x1="396" y1="427" x2="388" y2="427" stroke="#D97706" stroke-width="2" marker-end="url(#rf-arr)"/>
<rect x="196" y="63" width="62" height="20" rx="5" fill="#0D9488"/>
<rect x="600" y="63" width="48" height="20" rx="5" fill="#57534E"/>
<text x="227" y="78" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="624" y="78" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<rect x="24" y="466" width="812" height="124" rx="10" fill="#FFF7F7" stroke="#FFC9CC" stroke-width="1.5"/>
<text x="430" y="490" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Dấu hiệu bạn đang chọn sai cấu trúc</text>
<text x="44" y="514" font-size="11" fill="#BE123C">🚩 Sau một lệnh GET là một vòng for để tìm phần tử</text>
<text x="44" y="534" font-size="11" fill="#BE123C">🚩 Sau một lệnh GET là một lệnh sort</text>
<text x="44" y="554" font-size="11" fill="#BE123C">🚩 GET → sửa → SET đè lên key cũ (không atomic!)</text>
<text x="444" y="514" font-size="11" fill="#BE123C">🚩 Tự viết code kiểm tra trùng lặp trước khi thêm</text>
<text x="444" y="534" font-size="11" fill="#BE123C">🚩 Kéo cả danh sách về chỉ để đếm số lượng</text>
<text x="444" y="554" font-size="11" fill="#BE123C">🚩 Kéo hai danh sách về để tự tính phần giao nhau</text>
<text x="430" y="578" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">→ Cả 6 thứ trên, Redis đều làm được bằng 1 lệnh: nhanh hơn, ít dữ liệu hơn, và atomic.</text>
<text x="430" y="614" text-anchor="middle" font-size="12" font-weight="700" fill="#57534E">Câu hỏi đúng không phải "dữ liệu này hình dạng gì", mà là "tôi sẽ TRUY VẤN nó thế nào".</text>
<text x="430" y="634" text-anchor="middle" font-size="11" fill="#78716C">Giống như chọn index cho database — bạn thiết kế theo access pattern, không theo cấu trúc object trong code.</text>
</svg>
</div>

---

# Cái giá phải trả: bộ nhớ

Đây là phần bài gốc không nhắc tới, nhưng cần biết trước khi đi đổi hết code: **cấu trúc native của Redis thường tốn NHIỀU bộ nhớ hơn một chuỗi JSON.** Chúng đánh đổi RAM để lấy khả năng tính toán.

Đo bằng `MEMORY USAGE` trên cùng dữ liệu của hai ví dụ phía trên:

| Dữ liệu | Lưu dạng String | Lưu dạng native | Chênh |
|---|---:|---:|---:|
| Leaderboard 10.000 user | 393.272 bytes | Sorted Set: **963.872** | **2,45× tốn hơn** |
| 500 ID bài học đã lưu | 7.232 bytes | Set: **24.216** | **3,35× tốn hơn** |

Với leaderboard, đánh đổi này rõ ràng là lời: trả thêm 2,45× RAM để nhận 140× độ trễ và 2.160× network — chưa kể tính atomic. Nhưng cần biết là có đánh đổi, nhất là khi bạn có **hàng triệu key nhỏ**.

## Cái bẫy thật: ngưỡng encoding

Redis có hai cách lưu cho mỗi cấu trúc: khi tập dữ liệu còn **nhỏ**, nó dùng **listpack** — một mảng đóng gói liền mạch, cực kỳ tiết kiệm. Khi vượt ngưỡng, nó chuyển sang cấu trúc "thật" (skiplist / hashtable) — nhanh hơn về mặt lý thuyết nhưng tốn bộ nhớ hơn rất nhiều.

Và việc chuyển đổi đó xảy ra **chỉ vì thêm đúng một phần tử**:

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="mc-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="mc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="420" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Thêm 1 phần tử, bộ nhớ nhảy 12,7 lần</text>
<text x="430" y="50" text-anchor="middle" font-size="11" fill="#78716C">Sorted Set, đo bằng MEMORY USAGE trên Redis 7.4.2</text>
<rect x="24" y="66" width="400" height="330" rx="12" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<line x1="80" y1="350" x2="400" y2="350" stroke="#57534E" stroke-width="2"/>
<line x1="80" y1="90" x2="80" y2="350" stroke="#57534E" stroke-width="2"/>
<rect x="130" y="336" width="90" height="14" rx="3" fill="#0D9488"/>
<rect x="270" y="170" width="90" height="180" rx="3" fill="#DC2626"/>
<text x="175" y="326" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">1.072 B</text>
<text x="315" y="160" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">13.572 B</text>
<text x="175" y="370" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">n = 128</text>
<text x="175" y="387" text-anchor="middle" font-size="11" font-family="monospace" fill="#0F766E">listpack</text>
<text x="315" y="370" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">n = 129</text>
<text x="315" y="387" text-anchor="middle" font-size="11" font-family="monospace" fill="#9F1239">skiplist</text>
<path d="M226,300 C244,300 252,210 266,190" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#mc-arr)"/>
<rect x="196" y="100" width="196" height="46" rx="8" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="294" y="120" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">+1 phần tử</text>
<text x="294" y="138" text-anchor="middle" font-size="12" font-weight="800" fill="#BE123C">= +12.500 bytes</text>
<text x="52" y="220" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 52 220)">bộ nhớ (bytes)</text>
<rect x="444" y="66" width="392" height="330" rx="12" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="464" y="92" font-size="13" font-weight="800" fill="#374151">Ngưỡng chuyển encoding (mặc định)</text>
<rect x="464" y="104" width="352" height="26" rx="6" fill="#F5F5F4"/>
<text x="476" y="122" font-size="11" font-weight="700" fill="#374151">Cấu trúc</text>
<text x="640" y="122" font-size="11" font-weight="700" fill="#374151">Nhỏ</text>
<text x="724" y="122" font-size="11" font-weight="700" fill="#374151">Ngưỡng</text>
<rect x="464" y="134" width="352" height="26" rx="4" fill="#FFFFFF"/>
<rect x="464" y="164" width="352" height="26" rx="4" fill="#FAFAF9"/>
<rect x="464" y="194" width="352" height="26" rx="4" fill="#FFFFFF"/>
<text x="476" y="152" font-size="11" fill="#57534E">Sorted Set</text>
<text x="640" y="152" font-size="11" font-family="monospace" fill="#0F766E">listpack</text>
<text x="724" y="152" font-size="11" font-weight="700" fill="#9F1239">128 phần tử</text>
<text x="476" y="182" font-size="11" fill="#57534E">Set (chuỗi)</text>
<text x="640" y="182" font-size="11" font-family="monospace" fill="#0F766E">listpack</text>
<text x="724" y="182" font-size="11" font-weight="700" fill="#9F1239">128 phần tử</text>
<text x="476" y="212" font-size="11" fill="#57534E">Hash</text>
<text x="640" y="212" font-size="11" font-family="monospace" fill="#0F766E">listpack</text>
<text x="724" y="212" font-size="11" font-weight="700" fill="#9F1239">512 field</text>
<text x="464" y="248" font-size="12" font-weight="800" fill="#374151">Đo thật ở các kích thước</text>
<rect x="464" y="258" width="352" height="24" rx="4" fill="#F5F5F4"/>
<text x="476" y="274" font-size="11" font-weight="700" fill="#374151">n</text>
<text x="556" y="274" font-size="11" font-weight="700" fill="#374151">Sorted Set</text>
<text x="664" y="274" font-size="11" font-weight="700" fill="#374151">Hash</text>
<text x="752" y="274" font-size="11" font-weight="700" fill="#374151">Set</text>
<text x="476" y="300" font-size="11" font-family="monospace" fill="#57534E">50</text>
<text x="556" y="300" font-size="11" font-family="monospace" fill="#0F766E">432</text>
<text x="664" y="300" font-size="11" font-family="monospace" fill="#0F766E">560</text>
<text x="752" y="300" font-size="11" font-family="monospace" fill="#0F766E">304</text>
<text x="476" y="322" font-size="11" font-family="monospace" fill="#57534E">128</text>
<text x="556" y="322" font-size="11" font-family="monospace" fill="#0F766E">1.072</text>
<text x="664" y="322" font-size="11" font-family="monospace" fill="#0F766E">1.584</text>
<text x="752" y="322" font-size="11" font-family="monospace" fill="#0F766E">816</text>
<text x="476" y="344" font-size="11" font-family="monospace" font-weight="700" fill="#9F1239">129</text>
<text x="556" y="344" font-size="11" font-family="monospace" font-weight="700" fill="#9F1239">13.572</text>
<text x="664" y="344" font-size="11" font-family="monospace" fill="#0F766E">1.584</text>
<text x="752" y="344" font-size="11" font-family="monospace" font-weight="700" fill="#9F1239">6.280</text>
<text x="476" y="366" font-size="11" font-family="monospace" fill="#57534E">1.000</text>
<text x="556" y="366" font-size="11" font-family="monospace" fill="#9F1239">80.984</text>
<text x="664" y="366" font-size="11" font-family="monospace" fill="#9F1239">48.296</text>
<text x="752" y="366" font-size="11" font-family="monospace" fill="#9F1239">40.296</text>
<text x="464" y="388" font-size="11" fill="#78716C">Hash vẫn là listpack ở n=129 — ngưỡng của nó là 512.</text>
</svg>
</div>

Vì sao điều này quan trọng: nếu bạn có **1 triệu** Sorted Set nhỏ (ví dụ `leaderboard:{couple_id}` cho mỗi cặp đôi), chênh lệch giữa "mỗi set có 128 phần tử" và "mỗi set có 129 phần tử" là **1 GB vs 13 GB RAM**. Cùng một lượng dữ liệu nghiệp vụ.

Hai lệnh cần biết để kiểm tra:

```bash
redis-cli OBJECT ENCODING leaderboard:week   # listpack | skiplist | hashtable | intset
redis-cli MEMORY USAGE leaderboard:week      # số bytes thật, kể cả overhead
```

Có thể nâng ngưỡng để giữ encoding tiết kiệm lâu hơn:

```bash
redis-cli CONFIG SET zset-max-listpack-entries 512
```

Nhưng đây là đánh đổi ngược: listpack là mảng liền mạch, nên các thao tác trên nó là **`O(n)` quét tuyến tính**, không phải `O(log n)`. Nâng ngưỡng quá cao sẽ tiết kiệm RAM nhưng làm chậm từng phép truy cập. Mặc định 128 là con số đã được cân khá tốt — chỉ đổi khi bạn đã đo.

---

# Khi nào `GET`/`SET` với JSON vẫn là lựa chọn đúng

Nói vậy không có nghĩa là mọi thứ đều phải chuyển sang cấu trúc native. Có những trường hợp nhét JSON vào String là **lựa chọn đúng**, không phải lựa chọn lười:

| Tình huống | Vì sao String vẫn tốt hơn |
|---|---|
| Payload **lồng nhiều tầng** | Bẻ object lồng nhau thành nhiều key chỉ làm tăng gánh nặng đồng bộ và dọn dẹp |
| **Luôn đọc nguyên khối** rồi trả thẳng cho client | Không có phép tính nào để đẩy sang Redis — `GET` là tối ưu rồi |
| Block **HTML/JSON đã render sẵn** | Đúng mục đích của cache response |
| Cần **TTL cho cả khối** | `SETEX` hết hạn cả object một lần; với Hash/Set thì TTL chỉ đặt được ở mức key, không đặt được cho từng field |
| Dữ liệu **ghi một lần, đọc nhiều lần**, không bao giờ sửa từng phần | Không có read-modify-write nên không có vấn đề atomic |

Cố bẻ một object lồng nhau phức tạp thành nhiều key riêng lẻ đôi khi chỉ làm tăng gánh nặng đồng bộ, dọn dẹp key và quản lý vòng đời dữ liệu.

> 📌 **Phân biệt rõ**: serialize JSON rồi nhét vào một **String thường** khác hoàn toàn với dùng module **RedisJSON** (`JSON.SET` / `JSON.GET` với JSONPath). RedisJSON cho phép đọc/ghi **một đường dẫn bên trong** document mà không kéo cả khối về — đúng tinh thần "tính toán tại server", nhưng là một module riêng (có trong Redis Stack / Redis Open Source 8), không phải Redis core. Nếu dữ liệu của bạn thật sự là document lồng nhiều tầng mà vẫn cần truy cập từng phần, đó là lựa chọn đáng xem thay vì bẻ thủ công thành nhiều key.

---

# Vài thao tác server-side ít được để ý

Phần này không có trong bài gốc nhưng cùng một tinh thần — đẩy việc về phía Redis:

**`LPUSH` + `LTRIM` — danh sách "N phần tử gần nhất" tự giới hạn độ dài:**
```go
pipe := rdb.Pipeline()
pipe.LPush(ctx, "recent:user_42", lessonID)
pipe.LTrim(ctx, "recent:user_42", 0, 19)   // chỉ giữ 20 phần tử mới nhất
pipe.Exec(ctx)
```
Không cần đọc về để kiểm tra độ dài rồi cắt — Redis tự lo.

**`ZRANGEBYSCORE` + `ZREMRANGEBYSCORE` — rate limit sliding window chính xác:**
```go
now := time.Now().UnixMilli()
pipe := rdb.Pipeline()
pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(now-60_000, 10))  // bỏ request cũ hơn 60s
pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: requestID})
pipe.ZCard(ctx, key)                                                      // đếm còn lại
pipe.Expire(ctx, key, 2*time.Minute)
res, _ := pipe.Exec(ctx)
```
Cửa sổ trượt thật, không phải fixed window có bug "gấp đôi giới hạn ở ranh giới".

**`HINCRBY` — tăng một field trong object, không đụng các field khác:**
```go
rdb.HIncrBy(ctx, "stats:user_42", "lessons_completed", 1)
```

**`SCAN` thay cho `KEYS`** — `KEYS pattern` quét toàn bộ keyspace và **block Redis** (single-thread). Trên production đó là một sự cố tự gây ra. `SCAN` trả về theo lô, có cursor, không block.

**Lua script khi cần nhiều lệnh atomic:** pipeline chỉ gộp round-trip, **không** đảm bảo atomic. Nếu logic cần "đọc rồi quyết định rồi ghi" trong một khối không ai chen vào được, dùng `EVAL` — Redis chạy toàn bộ script như một lệnh đơn.

---

# Áp vào một dự án thật

Trong dự án [Honeydue](https://github.com/huydq96) (quản lý tài chính cho các cặp đôi) mình đang làm, các key Redis được chọn cấu trúc theo đúng access pattern:

| Key | Cấu trúc | Vì sao |
|---|---|---|
| `spending:{couple_id}:{category_id}:{month_year}` | **String** + `INCRBY` | Chỉ là một con số cộng dồn. `INCRBY` atomic — nhiều worker Kafka cùng cộng vào không mất dữ liệu |
| `presence:{user_id}` | **Set** of connection_id | Một user nhiều thiết bị/tab. Cần `SADD`/`SREM`/`SCARD`, tự khử trùng lặp |
| `couple:{couple_id}` | **String** (JSON) | Đọc nguyên khối, gần như không đổi, luôn dùng cả object → JSON blob là đúng |

Key đầu tiên là ví dụ rõ nhất cho luận điểm của bài. Bộ đếm chi tiêu được cập nhật bởi các worker tiêu thụ event `transaction.categorized`. Nếu lưu dạng JSON blob `{"groceries": 50000, "entertainment": 12000}` và làm read-modify-write, thì hai worker xử lý hai transaction cùng lúc sẽ **ghi đè lẫn nhau và mất tiền** — đúng kiểu bug tệ nhất với một ứng dụng tài chính: không crash, không log, chỉ là con số sai. Với `INCRBY` thì vấn đề không tồn tại.

Key thứ ba thì ngược lại: `couple:{couple_id}` được đọc nguyên khối, rất ít khi đổi, và không có phép tính nào cần đẩy sang Redis. Bẻ nó thành Hash chỉ thêm việc chứ không thêm giá trị. **Hai quyết định trái ngược nhau, cùng một nguyên tắc.**

---

# Dấu hiệu nhận biết và checklist

```
🚩 DẤU HIỆU ĐANG DÙNG REDIS NHƯ MEMCACHED
□ Sau một lệnh GET là một vòng for để tìm phần tử
□ Sau một lệnh GET là một lệnh sort
□ Mẫu GET → sửa → SET đè lên cùng key (read-modify-write)
□ Tự viết code kiểm tra trùng lặp trước khi thêm vào danh sách
□ Kéo cả danh sách về chỉ để đếm số lượng
□ Kéo hai danh sách về để tự tính phần giao nhau

✅ CHECKLIST KHI THIẾT KẾ KEY MỚI
□ Tôi sẽ TRUY VẤN dữ liệu này thế nào? (không phải: nó có hình dạng gì)
□ Có phép tính nào Redis làm hộ được không? (sort, đếm, tìm, cộng dồn, giao/hợp)
□ Có nhiều tiến trình cùng ghi vào key này không? → cần lệnh atomic, không read-modify-write
□ Tôi có cần đọc/ghi từng phần không? → Hash/Set/ZSet. Chỉ đọc nguyên khối? → String là đủ
□ Tập dữ liệu có vượt ngưỡng encoding (128 / 512) không? Nếu có nhiều key nhỏ, đã đo RAM chưa?
□ Đã đặt TTL chưa? (key không TTL là nguồn gốc của Redis hết RAM lúc 3 giờ sáng)
□ Có dùng KEYS ở đâu không? → đổi sang SCAN
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| Redis là gì? | **Remote data structure store**, không phải "cache nhanh" hay memcached mở rộng |
| Giá trị lớn nhất của Redis? | Các **thao tác tính toán chạy ngay tại server** — không phải việc dữ liệu nằm trên RAM |
| Nhét JSON vào String có sai không? | **Không sai** — nếu bạn luôn đọc nguyên khối và không tính toán gì trên đó |
| Khi nào nó thành sai? | Khi code xuất hiện `GET` → `sort`/`for`/`SET đè`. Đó là việc Redis làm được, nhanh hơn 100× |
| Chênh lệch thật là bao nhiêu? | Leaderboard 10.000 user: **141× latency, 2.160× network** (đo thật) |
| Lý do mạnh nhất để đổi? | Không phải tốc độ, mà là **tính atomic** — read-modify-write làm mất dữ liệu khi có ghi đồng thời |
| Cái giá phải trả? | Bộ nhớ: cấu trúc native tốn 2–3× so với JSON String. Và có ngưỡng encoding nhảy 12,7× |
| Câu hỏi đúng khi thiết kế key? | "Tôi sẽ **truy vấn** nó thế nào", giống như khi chọn index cho database |

Hồi mới học môn cấu trúc dữ liệu, có thể nhiều người từng tự hỏi mấy khái niệm như bảng băm, tập hợp hay cây sắp thứ tự sau này ra ngoài làm việc sẽ dùng ở đâu. Redis chính là một ví dụ trực quan: **chỉ cần đặt dữ liệu vào đúng cấu trúc của nó, ta đã bớt được cả một đoạn xử lý cồng kềnh trong code ứng dụng** — và bớt luôn một class bug về đồng thời mà có thể bạn còn chưa biết là mình đang có.

---

# Tham khảo

- [Redis — Data types](https://redis.io/docs/latest/develop/data-types/) — tài liệu chính thức từng cấu trúc
- [Redis — Memory optimization](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/) — ngưỡng encoding và cách đo
- [Redis — Command reference](https://redis.io/docs/latest/commands/) — tra độ phức tạp `O()` của từng lệnh, ghi ngay trong doc
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc của series này
- Bài liên quan trên blog: [Cẩm Nang Phỏng Vấn Backend](/posts/backend-interview-cheatsheet/) · [MySQL Indexing](/posts/mysql-indexing/)
