+++
date = '2026-10-19T09:00:00+07:00'
draft = false
slug = 'system-design-p04-streams-vs-kafka'
title = 'System Design P04: Redis Streams vs Kafka — Khi Nào Dùng Cái Nào'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "redis", "kafka", "event-driven", "golang"]
description = 'Benchmark đối đầu thật: throughput gần như bằng nhau, khác biệt nằm ở RAM vs đĩa. Và chuyện gì xảy ra khi stream dùng chung Redis với cache.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, ví dụ, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên Redis 7.4.2 và Kafka (Confluent cp-kafka) chạy local.*

# Tổng quan

Cả Redis Streams lẫn Kafka đều **lưu được sự kiện**, cho **nhiều bên cùng đọc**, và **đọc lại lịch sử** khi cần. Nhìn vào danh sách tính năng thì khá giống nhau — vậy chọn cái nào?

Bài [P03](/posts/system-design-p03-pubsub-vs-streams/) đã dừng ở câu "khi nào thì cần Kafka". Bài này trả lời câu đó bằng số đo thật. Và kết quả đầu tiên có thể làm nhiều người bất ngờ: **về throughput, hai cái gần như bằng nhau.** Khác biệt thật nằm ở chỗ khác.

---

# Bài toán tiến hoá theo thời gian

Giả sử app bán hàng phát một sự kiện mỗi khi đơn hàng được thanh toán. Hãy đi theo vòng đời thật của nó:

**Giai đoạn 1 — vài tháng đầu.** Sự kiện đó phục vụ hai việc: gửi email xác nhận và cập nhật doanh số tạm tính trong ngày. Lượng lịch sử cần giữ nhỏ, số lượt đọc ít. Team đã chạy sẵn Redis trong hạ tầng. **Redis Streams đáp ứng tốt, không cần dựng thêm hệ thống mới.**

**Giai đoạn 2 — sau một năm.** Nhiều bộ phận khác bắt đầu cần đọc tin đơn hàng: bên kho chuẩn bị xuất hàng, bên kế toán cần đối soát, nhóm dữ liệu muốn gom về phân tích. Mỗi bên một consumer group — Streams vẫn làm được.

**Giai đoạn 3 — một ngày đẹp trời.** Logic tính doanh thu cũ bị phát hiện sai lệch. Ứng dụng buộc phải **đọc lại toàn bộ sự kiện của 30 ngày vừa qua** để tính lại từ đầu.

Redis Streams cũng đọc lại 30 ngày được — **miễn là dữ liệu vẫn còn được giữ**. Và đó chính là chỗ bài toán đổi bản chất. Câu hỏi không còn là "cái nào chạy nhanh hơn", mà là **"giữ 30 ngày sự kiện tốn bao nhiêu, và ai trả tiền cho chỗ đó"**.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ev-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ev-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Yêu cầu không đứng yên — và điểm chuyển thường đến muộn</text>
<line x1="60" y1="88" x2="840" y2="88" stroke="#D6D3D1" stroke-width="3"/>
<circle cx="150" cy="88" r="9" fill="#0D9488"/>
<circle cx="420" cy="88" r="9" fill="#F5A623"/>
<circle cx="700" cy="88" r="11" fill="#DC2626"/>
<rect x="44" y="110" width="212" height="120" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#ev-sh)"/>
<rect x="314" y="110" width="212" height="120" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#ev-sh)"/>
<rect x="594" y="110" width="246" height="120" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#ev-sh)"/>
<text x="150" y="134" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">① Vài tháng đầu</text>
<text x="150" y="156" text-anchor="middle" font-size="11" fill="#0F766E">2 consumer: gửi email,</text>
<text x="150" y="173" text-anchor="middle" font-size="11" fill="#0F766E">cập nhật doanh số ngày</text>
<text x="150" y="195" text-anchor="middle" font-size="11" fill="#0F766E">Giữ lịch sử: vài giờ</text>
<text x="150" y="219" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">→ Redis Streams</text>
<text x="420" y="134" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">② Sau một năm</text>
<text x="420" y="156" text-anchor="middle" font-size="11" fill="#92400E">Thêm kho, kế toán,</text>
<text x="420" y="173" text-anchor="middle" font-size="11" fill="#92400E">nhóm dữ liệu — 5 group</text>
<text x="420" y="195" text-anchor="middle" font-size="11" fill="#92400E">Giữ lịch sử: vài ngày</text>
<text x="420" y="219" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">→ vẫn ổn, nhưng để ý RAM</text>
<text x="717" y="134" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">③ Logic doanh thu sai</text>
<text x="717" y="156" text-anchor="middle" font-size="11" fill="#BE123C">Phải replay 30 NGÀY</text>
<text x="717" y="173" text-anchor="middle" font-size="11" fill="#BE123C">để tính lại từ đầu</text>
<text x="717" y="195" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">Giữ lịch sử: 30 ngày</text>
<text x="717" y="219" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">→ 42 GB RAM?! hay 38 GB đĩa?</text>
<line x1="150" y1="104" x2="150" y2="88" stroke="#0D9488" stroke-width="2"/>
<line x1="420" y1="104" x2="420" y2="88" stroke="#F5A623" stroke-width="2"/>
<line x1="700" y1="106" x2="700" y2="88" stroke="#DC2626" stroke-width="2"/>
<text x="150" y="72" text-anchor="middle" font-size="11" fill="#78716C">tháng 1</text>
<text x="420" y="72" text-anchor="middle" font-size="11" fill="#78716C">tháng 12</text>
<text x="700" y="72" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">tháng 14 — sự cố</text>
<rect x="44" y="254" width="796" height="60" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="64" y="278" font-size="12" font-weight="800" fill="#7C4A03">Vấn đề không phải "chọn sai ở giai đoạn ①"</text>
<text x="64" y="300" font-size="11" fill="#92400E">Redis Streams là lựa chọn ĐÚNG ở giai đoạn ①. Vấn đề là điểm chuyển ở ③ thường đến lúc đang có sự cố — lúc tệ nhất để đổi hạ tầng.</text>
<rect x="44" y="330" width="796" height="52" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="64" y="352" font-size="12" font-weight="800" fill="#134E4A">Câu hỏi nên hỏi từ giai đoạn ①:</text>
<text x="64" y="372" font-size="11" fill="#0F766E">"Nếu một ngày phải đọc lại 30 ngày sự kiện, chúng ta có giữ được không — và tốn bao nhiêu?"</text>
</svg>
</div>

---

# Benchmark đối đầu

Trước khi bàn kiến trúc, hãy xem số. Cùng một máy, cùng một payload (**132 bytes/event**, JSON của sự kiện `order.paid`), 50.000 sự kiện.

## Thông lượng ghi

| | Thời gian | Throughput |
|---|---:|---:|
| Redis `XADD` (pipeline 2.000) | 146 ms | **341.966 events/s** |
| Kafka produce (batch 2.000, `acks=1`) | 148 ms | **338.134 events/s** |

**Gần như bằng nhau.** Chênh lệch 1% nằm trong sai số. Đây là điều đáng nói, vì nhiều người mặc định "Kafka nhanh hơn" hoặc "Redis nhanh hơn vì nằm trên RAM" — ở quy mô một máy, cả hai đều bị giới hạn bởi cùng những thứ: network round-trip, serialize, và hiệu quả của batching.

## Đọc lại toàn bộ lịch sử

| | Thời gian | Throughput |
|---|---:|---:|
| Redis `XRANGE` từ `0-0` | 40 ms | **1.264.967 events/s** |
| Kafka đọc từ `offset 0` | 65 ms | **764.177 events/s** |

Redis **nhanh hơn 1,65 lần** khi replay. Hợp lý: dữ liệu đã nằm sẵn trên RAM, không phải đọc đĩa.

> 📌 **Vậy thì Redis thắng?** Không. Benchmark này chỉ nói lên một điều: **throughput không phải lý do để chọn Kafka.** Nếu bạn đang cân nhắc đổi sang Kafka "cho nhanh hơn", số liệu trên nói rằng bạn sẽ thất vọng. Lý do thật nằm ở phần sau.

## Chi phí lưu trữ — đây mới là chỗ khác nhau

Đo trên cùng 50.000 sự kiện:

| | Dung lượng | Bytes/event |
|---|---:|---:|
| Redis Stream (`MEMORY USAGE`) | 8.169.928 bytes **RAM** | **163** |
| Kafka log (`.log` file trên đĩa) | 7.378.825 bytes **đĩa** | **148** |

Số byte gần như bằng nhau — overhead của Redis Stream chỉ khoảng 24% so với payload gốc, Kafka còn thấp hơn chút. **Nhưng một bên là RAM, một bên là đĩa.** Và đó là toàn bộ câu chuyện.

---

# Bảng dung lượng: từ CCU ra GB

Để quyết định được, phải đổi "số người dùng" thành "số sự kiện". Giả định đơn giản: **mỗi người đang hoạt động tạo trung bình một sự kiện mỗi 10 giây**.

- 1.000 CCU → ~**100 sự kiện/giây**
- 10.000 CCU → ~**1.000 sự kiện/giây**

Áp con số đo được (163 B/event RAM, 148 B/event đĩa):

| Tải | Retention | Số sự kiện | **Redis RAM** | **Kafka đĩa** |
|---|---|---:|---:|---:|
| 1.000 CCU | 1 ngày | 8,6 triệu | 1,4 GB | 1,3 GB |
| 1.000 CCU | 7 ngày | 60,5 triệu | 9,9 GB | 9,0 GB |
| **1.000 CCU** | **30 ngày** | **259 triệu** | **42,2 GB** | **38,4 GB** |
| 10.000 CCU | 1 ngày | 86,4 triệu | 14,1 GB | 12,8 GB |
| 10.000 CCU | 7 ngày | 605 triệu | 98,6 GB | 89,5 GB |
| **10.000 CCU** | **30 ngày** | **2,59 tỉ** | **422,5 GB** | **383,6 GB** |

Và nhân thêm hệ số dự phòng thực tế:

| Tình huống | Redis (+1 replica) | Kafka (RF=3) |
|---|---:|---:|
| 100 ev/s · 30 ngày | **84 GB RAM** | 115 GB đĩa |
| 1.000 ev/s · 30 ngày | **845 GB RAM** | 1,15 TB đĩa |

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="cap-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="420" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng số byte — nhưng một bên là RAM, một bên là đĩa</text>
<text x="440" y="48" text-anchor="middle" font-size="11" fill="#78716C">Tải 1.000 CCU (~100 sự kiện/giây), payload 132 bytes</text>
<line x1="90" y1="330" x2="470" y2="330" stroke="#57534E" stroke-width="2"/>
<line x1="90" y1="80" x2="90" y2="330" stroke="#57534E" stroke-width="2"/>
<rect x="124" y="322" width="36" height="8" rx="2" fill="#DC2626"/>
<rect x="166" y="323" width="36" height="7" rx="2" fill="#6366F1"/>
<rect x="236" y="272" width="36" height="58" rx="2" fill="#DC2626"/>
<rect x="278" y="277" width="36" height="53" rx="2" fill="#6366F1"/>
<rect x="348" y="82" width="36" height="248" rx="2" fill="#DC2626"/>
<rect x="390" y="104" width="36" height="226" rx="2" fill="#6366F1"/>
<text x="142" y="314" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">1,4</text>
<text x="184" y="316" text-anchor="middle" font-size="10" font-weight="700" fill="#4338CA">1,3</text>
<text x="254" y="264" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">9,9</text>
<text x="296" y="269" text-anchor="middle" font-size="10" font-weight="700" fill="#4338CA">9,0</text>
<text x="366" y="74" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">42,2</text>
<text x="408" y="96" text-anchor="middle" font-size="12" font-weight="800" fill="#4338CA">38,4</text>
<text x="163" y="350" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">1 ngày</text>
<text x="275" y="350" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">7 ngày</text>
<text x="387" y="350" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">30 ngày</text>
<text x="280" y="376" text-anchor="middle" font-size="11" fill="#78716C">retention (GB)</text>
<rect x="110" y="88" width="14" height="12" rx="3" fill="#DC2626"/>
<rect x="110" y="108" width="14" height="12" rx="3" fill="#6366F1"/>
<text x="132" y="98" font-size="11" font-weight="700" fill="#9F1239">Redis — RAM</text>
<text x="132" y="118" font-size="11" font-weight="700" fill="#4338CA">Kafka — đĩa</text>
<rect x="500" y="70" width="356" height="124" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#cap-sh)"/>
<text x="520" y="96" font-size="13" font-weight="800" fill="#9F1239">42 GB RAM nghĩa là gì?</text>
<text x="520" y="120" font-size="11" fill="#BE123C">• Cần máy ≥ 64 GB RAM chỉ cho một stream</text>
<text x="520" y="140" font-size="11" fill="#BE123C">• Thêm replica → nhân đôi: 84 GB</text>
<text x="520" y="160" font-size="11" fill="#BE123C">• RAM đắt hơn SSD khoảng 10–30 lần mỗi GB</text>
<text x="520" y="182" font-size="11" font-weight="700" fill="#DC2626">• Và vẫn chỉ nằm trên MỘT node</text>
<rect x="500" y="206" width="356" height="124" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#cap-sh)"/>
<text x="520" y="232" font-size="13" font-weight="800" fill="#3730A3">38 GB đĩa nghĩa là gì?</text>
<text x="520" y="256" font-size="11" fill="#4338CA">• Một ổ SSD phổ thông, không có gì đặc biệt</text>
<text x="520" y="276" font-size="11" fill="#4338CA">• RF=3 → 115 GB, vẫn là chuyện nhỏ</text>
<text x="520" y="296" font-size="11" fill="#4338CA">• Chia được qua nhiều broker bằng partition</text>
<text x="520" y="318" font-size="11" font-weight="700" fill="#3730A3">• Đổi lại: thêm một hệ thống phải vận hành</text>
<rect x="90" y="388" width="766" height="22" rx="6" fill="#FFF3DB"/>
<text x="473" y="404" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Cùng ~150 bytes/sự kiện. Khác biệt không nằm ở SỐ byte, mà ở LOẠI bộ nhớ phải trả tiền cho chúng.</text>
</svg>
</div>

Bảng này là lý do thật để chọn Kafka — không phải tốc độ. **Giữ 30 ngày sự kiện trên RAM là một quyết định tài chính**, còn giữ trên đĩa thì gần như miễn phí.

---

# Chuyện gì xảy ra khi stream dùng chung Redis với cache

Đây là phần tôi muốn đào sâu nhất, vì bài gốc nêu đúng rủi ro — *"nếu đặt stream chung với cache, cả hai sẽ dùng chung RAM"* — nhưng ít ai hình dung được hậu quả cụ thể. Tôi dựng một Redis riêng với `maxmemory 48MB`, nạp 20.000 key cache (~22 MB), rồi ghi stream vào **cùng instance** và theo dõi từng lô 8.000 entry.

## Với `maxmemory-policy = allkeys-lru`

```
nền: 20000 key cache, 22.2 MB / 48 MB
lô  1 (+8000) → XLEN   8000 · cache còn 20000 · 25.6 MB · evicted=0
lô  5 (+8000) → XLEN  40000 · cache còn 20000 · 39.6 MB · evicted=0
lô  7 (+8000) → XLEN  56000 · cache còn 20000 · 46.6 MB · evicted=0
lô  8 (+8000) → XLEN  64000 · cache còn 17997 · 48.0 MB · evicted=2003
lô  9 (+8000) → XLEN  72000 · cache còn 14632 · 48.0 MB · evicted=5368
lô 10 (+8000) → XLEN   5848 · cache còn 13725 · 18.2 MB · evicted=6276   ⚠️
```

Đọc kỹ ba dòng cuối. Ở lô 8–9, Redis bắt đầu **xoá key cache** để lấy chỗ cho stream — 5.368 key cache biến mất, cache hit rate của ứng dụng tụt mà không ai biết vì sao.

Rồi ở lô 10, `XLEN` **tụt từ 72.000 xuống 5.848**. Không phải stream bị cắt bớt — mà **cả key stream bị xoá sạch**, rồi những lệnh `XADD` còn lại trong lô tạo nó lại từ đầu.

**Bạn vừa mất 72.000 sự kiện. Không có lỗi nào. `XADD` trả về thành công.**

Lý do: `allkeys-lru` nghĩa là *mọi* key đều có thể bị evict — và key stream cũng là một key như mọi key khác. Redis không biết đâu là "cache có thể mất" và đâu là "event log không được mất".

## Với `maxmemory-policy = noeviction`

```
lô  7 (+8000) → XLEN  56000 · cache còn 20000 · 46.6 MB · evicted=0
lô  8 (+8000) → XLEN  59248 · cache còn 20000 · 48.0 MB · evicted=0
   ✋ XADD bị TỪ CHỐI: OOM command not allowed when used memory > 'maxmemory'
```

Không mất gì cả, nhưng **`XADD` bắt đầu thất bại**. Producer của bạn nhận lỗi và phải tự xử lý. Ít nhất đây là **hỏng ồn ào** thay vì **mất dữ liệu âm thầm**.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="pol-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cache và stream chung một Redis — cả hai lựa chọn đều có giá</text>
<rect x="24" y="48" width="270" height="290" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#pol-sh)"/>
<rect x="306" y="48" width="270" height="290" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#pol-sh)"/>
<rect x="588" y="48" width="268" height="290" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#pol-sh)"/>
<rect x="24" y="48" width="270" height="34" rx="12" fill="#DC2626"/>
<rect x="24" y="68" width="270" height="14" fill="#DC2626"/>
<rect x="306" y="48" width="270" height="34" rx="12" fill="#D97706"/>
<rect x="306" y="68" width="270" height="14" fill="#D97706"/>
<rect x="588" y="48" width="268" height="34" rx="12" fill="#0D9488"/>
<rect x="588" y="68" width="268" height="14" fill="#0D9488"/>
<text x="159" y="71" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">allkeys-lru</text>
<text x="441" y="71" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">noeviction</text>
<text x="722" y="71" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">TÁCH RIÊNG 2 INSTANCE</text>
<text x="42" y="108" font-size="11" font-weight="800" fill="#374151">Chuyện gì xảy ra</text>
<text x="42" y="130" font-size="11" fill="#BE123C">Redis xoá key cache trước,</text>
<text x="42" y="148" font-size="11" fill="#BE123C">rồi xoá luôn CẢ KEY STREAM.</text>
<rect x="42" y="162" width="234" height="48" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="159" y="182" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Mất 72.000 sự kiện</text>
<text x="159" y="200" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">XADD vẫn báo THÀNH CÔNG</text>
<text x="42" y="232" font-size="11" font-weight="800" fill="#374151">Phát hiện được không?</text>
<text x="42" y="252" font-size="11" fill="#BE123C">Rất khó. Không lỗi, không log.</text>
<text x="42" y="270" font-size="11" fill="#BE123C">Chỉ thấy consumer "thiếu việc"</text>
<text x="42" y="288" font-size="11" fill="#BE123C">và cache hit rate tụt.</text>
<text x="42" y="316" font-size="11" font-weight="800" fill="#DC2626">🚫 Tuyệt đối tránh nếu Redis</text>
<text x="42" y="330" font-size="11" font-weight="800" fill="#DC2626">     có chứa stream</text>
<text x="324" y="108" font-size="11" font-weight="800" fill="#374151">Chuyện gì xảy ra</text>
<text x="324" y="130" font-size="11" fill="#92400E">Không xoá gì cả.</text>
<text x="324" y="148" font-size="11" fill="#92400E">XADD bị từ chối khi đầy.</text>
<rect x="324" y="162" width="234" height="48" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="441" y="182" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">OOM command not allowed</text>
<text x="441" y="200" text-anchor="middle" font-size="11" font-weight="700" fill="#D97706">Hỏng ỒN ÀO — tốt hơn</text>
<text x="324" y="232" font-size="11" font-weight="800" fill="#374151">Phát hiện được không?</text>
<text x="324" y="252" font-size="11" fill="#92400E">Ngay lập tức — producer</text>
<text x="324" y="270" font-size="11" fill="#92400E">nhận lỗi và alert nổ.</text>
<text x="324" y="288" font-size="11" fill="#92400E">Nhưng cache cũng ngừng</text>
<text x="324" y="306" font-size="11" fill="#92400E">nhận key mới cùng lúc.</text>
<text x="324" y="330" font-size="11" font-weight="800" fill="#7C4A03">⚠ Chấp nhận được, chưa tối ưu</text>
<text x="606" y="108" font-size="11" font-weight="800" fill="#374151">Chuyện gì xảy ra</text>
<text x="606" y="130" font-size="11" fill="#0F766E">Cache: allkeys-lru, evict</text>
<text x="606" y="148" font-size="11" fill="#0F766E">thoải mái (đúng bản chất).</text>
<rect x="606" y="162" width="232" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="722" y="182" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">Stream: noeviction</text>
<text x="722" y="200" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">+ MAXLEN tự cắt bớt</text>
<text x="606" y="232" font-size="11" font-weight="800" fill="#374151">Vì sao đúng</text>
<text x="606" y="252" font-size="11" fill="#0F766E">Hai loại dữ liệu có yêu cầu</text>
<text x="606" y="270" font-size="11" fill="#0F766E">ĐỘ BỀN khác hẳn nhau —</text>
<text x="606" y="288" font-size="11" fill="#0F766E">không thể chung một policy.</text>
<text x="606" y="316" font-size="11" font-weight="800" fill="#0F766E">✅ Nếu buộc phải dùng Redis</text>
<text x="606" y="330" font-size="11" font-weight="800" fill="#0F766E">     cho cả hai, làm cách này</text>
<rect x="24" y="356" width="832" height="34" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="440" y="378" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Kafka không có bài toán này: event log nằm trên đĩa riêng, không tranh RAM với bất kỳ thứ gì khác.</text>
</svg>
</div>

**Kết luận thực tế**: nếu dùng Redis Streams cho sự kiện quan trọng, **đừng để nó chung instance với cache**. Tách ra hai instance, đặt policy phù hợp cho từng loại:

```bash
# Instance cache — evict thoải mái, đó là bản chất của cache
redis-server --port 6379 --maxmemory 4gb --maxmemory-policy allkeys-lru

# Instance stream — KHÔNG BAO GIỜ evict, kiểm soát dung lượng bằng MAXLEN
redis-server --port 6380 --maxmemory 8gb --maxmemory-policy noeviction
```

Và luôn đặt `MAXLEN`/`MINID` khi `XADD` — xem lại [P03](/posts/system-design-p03-pubsub-vs-streams/#streams-không-phải-kho-lịch-sử-vô-hạn).

---

# Chia tải: chỗ Kafka thật sự khác

Đây là khác biệt kiến trúc, không phải khác biệt hiệu năng.

**Redis Streams**: một stream là **một key, nằm trên một node**. Thêm máy vào cụm Redis **không tự chia** một stream đang có. Muốn chia tải qua nhiều máy, **ứng dụng phải tự thiết kế** cách tách thành nhiều stream — ví dụ `order.paid:{shard}` với `shard = hash(order_id) % N` — và tự lo việc đọc từ N stream, tự cân bằng, tự xử lý khi đổi N.

**Kafka**: topic được chia sẵn thành **partition** đặt trên các broker khác nhau. Thêm partition và thêm broker là thao tác vận hành, **không phải thay đổi code ứng dụng**. Consumer group tự phân chia partition cho các consumer.

> ⚖️ **Về mặt trung thực của benchmark**: tôi *không* đưa số đo so sánh thông lượng consumer ở đây. Lý do: tôi chỉ có **một broker Kafka trên một máy**, nên mọi phép đo "chia tải qua nhiều máy" đều vô nghĩa — chúng chỉ đo được chi phí rebalance và độ trễ của chính cái test harness, không đo được thứ cần đo. Lợi thế mở rộng ngang của Kafka là **thuộc tính kiến trúc**, chỉ hiện ra khi có nhiều broker thật. Đưa ra một con số từ môi trường một máy sẽ gây hiểu nhầm hơn là giúp ích.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 380" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="sc-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="sc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="380" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Muốn chia tải qua nhiều máy thì làm thế nào?</text>
<rect x="24" y="48" width="412" height="306" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2" filter="url(#sc-sh)"/>
<rect x="448" y="48" width="408" height="306" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2" filter="url(#sc-sh)"/>
<text x="230" y="74" text-anchor="middle" font-size="14" font-weight="800" fill="#7C4A03">Redis Streams — ỨNG DỤNG tự lo</text>
<text x="652" y="74" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">Kafka — HẠ TẦNG lo</text>
<rect x="150" y="90" width="160" height="34" rx="8" fill="#D97706"/>
<text x="230" y="112" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">Producer</text>
<rect x="44" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="174" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="304" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<text x="99" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#92400E">order:shard0</text>
<text x="99" y="200" text-anchor="middle" font-size="10" fill="#92400E">node A</text>
<text x="229" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#92400E">order:shard1</text>
<text x="229" y="200" text-anchor="middle" font-size="10" fill="#92400E">node B</text>
<text x="359" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#92400E">order:shard2</text>
<text x="359" y="200" text-anchor="middle" font-size="10" fill="#92400E">node C</text>
<path d="M200,126 L110,156" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<path d="M230,126 L230,156" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<path d="M262,126 L350,156" stroke="#D97706" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<text x="230" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">hash(order_id) % 3 — CODE của bạn</text>
<rect x="44" y="230" width="370" height="112" rx="9" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="62" y="252" font-size="11" font-weight="800" fill="#7C4A03">Bạn phải tự viết và tự bảo trì:</text>
<text x="62" y="272" font-size="11" fill="#92400E">• Logic băm và định tuyến ở producer</text>
<text x="62" y="290" font-size="11" fill="#92400E">• Consumer đọc song song từ N stream</text>
<text x="62" y="308" font-size="11" fill="#92400E">• Cân bằng lại khi một node quá tải</text>
<text x="62" y="328" font-size="11" font-weight="700" fill="#DC2626">• Đổi N = viết migration, thứ tự có thể vỡ</text>
<rect x="572" y="90" width="160" height="34" rx="8" fill="#0D9488"/>
<text x="652" y="112" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">Producer</text>
<rect x="468" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="598" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="728" y="160" width="110" height="52" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<text x="523" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#0F766E">partition 0</text>
<text x="523" y="200" text-anchor="middle" font-size="10" fill="#0F766E">broker 1</text>
<text x="653" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#0F766E">partition 1</text>
<text x="653" y="200" text-anchor="middle" font-size="10" fill="#0F766E">broker 2</text>
<text x="783" y="180" text-anchor="middle" font-size="10" font-family="monospace" fill="#0F766E">partition 2</text>
<text x="783" y="200" text-anchor="middle" font-size="10" fill="#0F766E">broker 3</text>
<path d="M622,126 L534,156" stroke="#0D9488" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<path d="M652,126 L652,156" stroke="#0D9488" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<path d="M684,126 L774,156" stroke="#0D9488" stroke-width="1.6" fill="none" marker-end="url(#sc-arr)"/>
<text x="652" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">partitioner — CLIENT LIBRARY lo</text>
<rect x="468" y="230" width="370" height="112" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="486" y="252" font-size="11" font-weight="800" fill="#134E4A">Hạ tầng lo sẵn:</text>
<text x="486" y="272" font-size="11" fill="#0F766E">• Partitioner nằm trong client library</text>
<text x="486" y="290" font-size="11" fill="#0F766E">• Consumer group tự chia partition</text>
<text x="486" y="308" font-size="11" fill="#0F766E">• Thêm broker = thao tác vận hành</text>
<text x="486" y="328" font-size="11" font-weight="700" fill="#9F1239">• Đổi lại: thêm 1 hệ thống phải vận hành</text>
</svg>
</div>

---

# Hệ sinh thái và chi phí vận hành

Hai yếu tố cuối, thường bị bỏ quên lúc so sánh nhưng quyết định rất nhiều ở thực tế.

**Hệ sinh thái kết nối.** Quanh Kafka có sẵn Kafka Connect với hàng trăm connector sang database, data warehouse, S3, Elasticsearch... Nếu ứng dụng liên tục phải đẩy dữ liệu sang các nền tảng xử lý dữ liệu khác, những công cụ sẵn có này giảm đáng kể phần phải tự viết. Redis Streams không có thứ tương đương — bạn tự viết consumer cho từng đích đến.

**Chi phí vận hành.** Đây là phía ngược lại:

| | Redis Streams | Kafka tự vận hành | Kafka quản lý (managed) |
|---|---|---|---|
| Thêm hệ thống mới? | **Không, nếu đã có Redis** | Có — broker + KRaft/ZooKeeper | Có, nhưng không phải tự chạy |
| Việc phải làm | Theo dõi RAM, đặt `MAXLEN`, backup RDB/AOF | Theo dõi dung lượng, **consumer lag**, nâng cấp rolling, rebalance, xử lý sự cố | Thiết kế luồng dữ liệu, theo dõi lag |
| Kiến thức cần có | Đã có sẵn trong team | **Đáng kể** — cần người hiểu Kafka | Trung bình |
| Chi phí tiền | Chỉ RAM | Máy chủ + thời gian người | **Phí dịch vụ** + thời gian người |

Dùng Kafka quản lý sẵn giúp giảm phần bảo trì máy chủ, nhưng **vẫn có phí dịch vụ và phần thiết kế luồng dữ liệu vẫn phải tự lo**. Không có lựa chọn nào miễn phí.

---

# Chọn thế nào?

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ch4-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="460" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Năm câu hỏi — trả lời "có" từ 2 câu trở lên thì cân nhắc Kafka</text>
<rect x="30" y="50" width="800" height="48" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="106" width="800" height="48" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="162" width="800" height="48" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="218" width="800" height="48" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="274" width="800" height="48" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="50" width="10" height="48" rx="4" fill="#6366F1"/>
<rect x="30" y="106" width="10" height="48" rx="4" fill="#6366F1"/>
<rect x="30" y="162" width="10" height="48" rx="4" fill="#6366F1"/>
<rect x="30" y="218" width="10" height="48" rx="4" fill="#6366F1"/>
<rect x="30" y="274" width="10" height="48" rx="4" fill="#6366F1"/>
<text x="58" y="72" font-size="12" font-weight="800" fill="#374151">1. Cần giữ lịch sử bao lâu?</text>
<text x="58" y="90" font-size="11" fill="#57534E">Vài giờ – vài ngày → Streams.  Nhiều tuần – vĩnh viễn → Kafka (RAM không gánh nổi).</text>
<text x="58" y="128" font-size="12" font-weight="800" fill="#374151">2. Dung lượng lịch sử có vượt RAM của một node không?</text>
<text x="58" y="146" font-size="11" fill="#57534E">Tính thật: (sự kiện/giây) × (giây retention) × (~163 bytes) × (số replica).</text>
<text x="58" y="184" font-size="12" font-weight="800" fill="#374151">3. Có cần chia tải qua nhiều máy không?</text>
<text x="58" y="202" font-size="11" fill="#57534E">Streams: bạn tự shard bằng code.  Kafka: partition là chuyện vận hành.</text>
<text x="58" y="240" font-size="12" font-weight="800" fill="#374151">4. Có phải đẩy dữ liệu sang nhiều nền tảng khác không?</text>
<text x="58" y="258" font-size="11" fill="#57534E">Nhiều đích đến → Kafka Connect tiết kiệm rất nhiều code tự viết.</text>
<text x="58" y="296" font-size="12" font-weight="800" fill="#374151">5. Có được phép mất BẤT KỲ sự kiện nào không?</text>
<text x="58" y="314" font-size="11" fill="#57534E">Không được mất → Kafka acks=all + replication.  Mất 1 giây là chấp nhận được → Streams.</text>
<rect x="30" y="338" width="392" height="104" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#ch4-sh)"/>
<rect x="438" y="338" width="392" height="104" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#ch4-sh)"/>
<text x="226" y="362" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">REDIS STREAMS</text>
<text x="226" y="384" text-anchor="middle" font-size="11" fill="#0F766E">Đã có Redis · lịch sử vừa phải · xử lý tiếp</text>
<text x="226" y="402" text-anchor="middle" font-size="11" fill="#0F766E">các công việc · team quen vận hành Redis</text>
<text x="226" y="426" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Vẫn phải lo: MAXLEN, backup, phục hồi</text>
<text x="634" y="362" text-anchor="middle" font-size="14" font-weight="800" fill="#3730A3">KAFKA</text>
<text x="634" y="384" text-anchor="middle" font-size="11" fill="#4338CA">Luồng sự kiện DÙNG CHUNG · lịch sử lớn</text>
<text x="634" y="402" text-anchor="middle" font-size="11" fill="#4338CA">nhiều tích hợp · chia tải qua nhiều máy</text>
<text x="634" y="426" text-anchor="middle" font-size="11" font-weight="700" fill="#3730A3">Chọn vì NHU CẦU, và tính cả công vận hành</text>
</svg>
</div>

Tóm lại thành hai câu:

- **Redis Streams** phù hợp khi ứng dụng cần **xử lý tiếp các công việc**, lượng lịch sử và lượt đọc lại vẫn nằm trong tài nguyên đã dành cho nó. Team đã quen vận hành Redis thì có thêm lợi thế — nhưng vẫn phải lo phần lưu dữ liệu và phục hồi.
- **Kafka** đáng cân nhắc khi cần một **luồng sự kiện dùng chung**, lượng lịch sử lớn, nhiều tích hợp, và cách chia tải qua nhiều máy. Chọn vì **những nhu cầu đó**, đồng thời tính cả công vận hành — **chứ không đợi Redis gặp sự cố mới xem xét**.

> 📌 Trong [Honeydue](https://github.com/huydq96) mình dùng **Kafka** cho domain event (`transaction.created`, `budget.limit_changed`) ngay từ đầu, vì đó là sự kiện nghiệp vụ tài chính cần replay và audit — câu 1 và câu 5 ở trên đều "có". Nhưng Redis vẫn ở đó cho cache, presence, spending counter và Pub/Sub fan-out WebSocket. **Hai hệ thống, mỗi cái làm việc nó giỏi nhất** — không phải chọn một rồi dùng cho tất cả.

---

# Checklist trước khi quyết định

```
TÍNH TOÁN TRƯỚC (đừng ước lượng cảm tính)
□ Đổi CCU → sự kiện/giây (giả định: 1 sự kiện / 10 giây / người hoạt động)
□ Đo kích thước THẬT của một sự kiện sau khi serialize
□ Tính: sự kiện/giây × giây retention × bytes/event = dung lượng cần
□ Nhân hệ số replica (Redis ×2, Kafka RF=3)
□ So con số đó với RAM một node Redis — vượt thì câu trả lời đã rõ

NẾU CHỌN REDIS STREAMS
□ Stream có nằm CHUNG instance với cache không? → TÁCH RA
□ maxmemory-policy của instance chứa stream = noeviction (KHÔNG allkeys-lru)
□ Mọi XADD đều có MAXLEN hoặc MINID chưa?
□ Retention có đủ dài cho tình huống "phải replay để sửa số liệu" không?
□ RDB/AOF đã bật và đã TEST phục hồi thật chưa?
□ Nếu cần chia tải: đã thiết kế sharding ở tầng ứng dụng chưa?

NẾU CHỌN KAFKA
□ Ai trong team vận hành được? Đã tính thời gian người chưa?
□ Tự chạy hay dùng managed? Đã so chi phí thật chưa?
□ Số partition ban đầu — tăng được, nhưng GIẢM thì không
□ Có theo dõi consumer lag không?
□ acks và replication factor đã khớp với yêu cầu "được phép mất gì" chưa?
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| Cái nào ghi nhanh hơn? | **Gần như bằng nhau** — Redis 342k/s, Kafka 338k/s (đo thật, 1 máy) |
| Cái nào replay nhanh hơn? | Redis, **1,65×** (1,26M/s vs 764k/s) — vì dữ liệu sẵn trên RAM |
| Vậy chọn Kafka vì tốc độ? | **Không.** Tốc độ không phải lý do |
| Khác biệt thật là gì? | **163 bytes/event trên RAM** vs **148 bytes/event trên đĩa** |
| Giữ 30 ngày ở 1.000 CCU tốn gì? | Redis **42 GB RAM** (84 GB có replica) vs Kafka **38 GB đĩa** (115 GB RF=3) |
| Stream chung Redis với cache? | **Nguy hiểm.** `allkeys-lru` xoá cả key stream — mất 72.000 sự kiện, không báo lỗi |
| Policy nào đúng cho stream? | `noeviction` + `MAXLEN`. Và tốt nhất là **tách instance riêng** |
| Chia tải qua nhiều máy? | Streams: **bạn tự shard bằng code**. Kafka: partition là chuyện vận hành |
| Khi nào nên đổi sang Kafka? | Khi trả lời "có" từ 2/5 câu hỏi — và quyết định **trước** khi có sự cố |

---

# Tham khảo

- [Redis — Streams](https://redis.io/docs/latest/develop/data-types/streams/) · [Memory optimization](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/)
- [Redis — Key eviction policies](https://redis.io/docs/latest/develop/reference/eviction/) — đọc kỹ phần `allkeys-*` vs `volatile-*`
- [Kafka — Design: Persistence](https://kafka.apache.org/documentation/#persistence) · [Kafka Connect](https://kafka.apache.org/documentation/#connect)
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc của series
- Series: [P01 — Redis không chỉ có GET/SET](/posts/system-design-p01-redis/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/) · [P05 — Redis Queue hay RabbitMQ?](/posts/system-design-p05-queue/)
- Trên blog: [Apache Kafka P1](/posts/apache-kafka-p1/) · [Apache Kafka P2](/posts/apache-kafka-p2/)
