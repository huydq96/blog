+++
date = '2026-10-22T01:00:00+07:00'
draft = false
slug = 'scalable-p01-2-ccu-deep-dive'
title = 'Scalable System Design P01.2: Thiết Kế Hệ Thống 1M CCU — Gaming'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "scalability", "websocket", "golang", "gaming"]
description = 'Chi tiết từng tầng component cho 1M connection, tradeoff của mỗi lựa chọn, 3 hot path đo thật (fan-out, slow consumer, reconnect storm) kèm code Go, và failure mode tệ nhất.'
+++

> *Bài viết thuộc series **Scalable System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, kiến trúc, code và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên máy local (Go 1.24, Redis 7.4.2).*

# Mở đầu

[Bài P01](/posts/scalable-p01-ccu-vs-rps/) dừng ở chỗ: *"1M CCU không phải một kiến trúc"*, và đưa ra các bước để xây workload model. Bài này đi tiếp — **chi tiết từng tầng component, tradeoff của mỗi lựa chọn, và ba hot path khó nhất** của một hệ thống CCU-heavy.

Bài tiếp theo — [P01.3](/posts/scalable-p01-3-rps-deep-dive/) — làm điều tương tự cho hệ thống RPS-heavy (flash sale).

## Workload model (chốt từ P01)

Hệ thống: **realtime gaming, 1M CCU**. Mọi thiết kế bên dưới phục vụ đúng bộ số này:

| Chỉ số | Giá trị | Nguồn |
|---|---:|---|
| Connection giữ liên tục | 1.000.000 | giả định |
| Message/giây VÀO | 10.000.000 | 10 input/giây/player |
| Message/giây RA | 20.000.000 | tick rate 20/giây |
| Bandwidth RA | 32 Gbps | 200 bytes/message |
| RAM chỉ cho connection | **3,4 GB** | **3.501 B/conn — đo thật ở P01** |
| Player mỗi room | 50 | giả định (battle royale: 100) |
| Số room đồng thời | 20.000 | 1M ÷ 50 |
| Mục tiêu p99 latency | **< 50 ms** | yêu cầu nghiệp vụ |

Hai con số quyết định toàn bộ kiến trúc:

- **p99 < 50 ms** — loại bỏ mọi thiết kế phải đi qua database trong hot path.
- **32 Gbps ra** — một NIC 25 Gbps không đủ; bắt buộc nhiều node, và **bandwidth là tài nguyên khan nhất**, không phải CPU.

---

# Kiến trúc tổng thể

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 620" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ar-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="ar-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<marker id="ar-r" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="ar-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="620" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">1M CCU gaming — các tầng và trách nhiệm</text>
<rect x="24" y="42" width="852" height="74" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2"/>
<text x="44" y="64" font-size="12" font-weight="800" fill="#3730A3">① EDGE — Anycast DNS + L4 load balancer</text>
<rect x="44" y="72" width="190" height="34" rx="7" fill="#FFFFFF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="246" y="72" width="190" height="34" rx="7" fill="#FFFFFF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="448" y="72" width="190" height="34" rx="7" fill="#FFFFFF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="650" y="72" width="206" height="34" rx="7" fill="#E0E7FF" stroke="#6366F1" stroke-width="1.6"/>
<text x="139" y="93" text-anchor="middle" font-size="11" fill="#4338CA">region ap-southeast</text>
<text x="341" y="93" text-anchor="middle" font-size="11" fill="#4338CA">region us-west</text>
<text x="543" y="93" text-anchor="middle" font-size="11" fill="#4338CA">region eu-central</text>
<text x="753" y="87" text-anchor="middle" font-size="11" font-weight="700" fill="#3730A3">Định tuyến theo RTT</text>
<text x="753" y="101" text-anchor="middle" font-size="10" fill="#4338CA">latency là NFR số 1</text>
<rect x="24" y="128" width="852" height="96" rx="10" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5"/>
<text x="44" y="150" font-size="12" font-weight="800" fill="#134E4A">② GATEWAY — giữ connection, KHÔNG giữ game state (stateless về nghiệp vụ)</text>
<rect x="44" y="158" width="152" height="54" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="208" y="158" width="152" height="54" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="372" y="158" width="152" height="54" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="536" y="158" width="100" height="54" rx="7" fill="#F5F5F4" stroke="#D6D3D1" stroke-width="1.5" stroke-dasharray="4 3"/>
<rect x="648" y="158" width="208" height="54" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="120" y="178" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">GW-1</text>
<text x="120" y="196" text-anchor="middle" font-size="10" fill="#0F766E">100k conn · 350 MB</text>
<text x="284" y="178" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">GW-2</text>
<text x="284" y="196" text-anchor="middle" font-size="10" fill="#0F766E">100k conn · 350 MB</text>
<text x="448" y="178" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">GW-3</text>
<text x="448" y="196" text-anchor="middle" font-size="10" fill="#0F766E">100k conn · 350 MB</text>
<text x="586" y="183" text-anchor="middle" font-size="11" fill="#78716C">... GW-10</text>
<text x="586" y="199" text-anchor="middle" font-size="10" fill="#A8A29E">tổng 1M</text>
<text x="752" y="178" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">10 node × 3,2 Gbps</text>
<text x="752" y="196" text-anchor="middle" font-size="10" fill="#0F766E">= 32 Gbps — vừa đủ NIC 25G</text>
<rect x="24" y="236" width="852" height="100" rx="10" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5"/>
<text x="44" y="258" font-size="12" font-weight="800" fill="#7C4A03">③ ROOM SERVER — authoritative game state, TOÀN BỘ trong RAM</text>
<rect x="44" y="266" width="186" height="60" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.8"/>
<rect x="242" y="266" width="186" height="60" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.8"/>
<rect x="440" y="266" width="186" height="60" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.8"/>
<rect x="638" y="266" width="218" height="60" rx="7" fill="#FFF3DB" stroke="#D97706" stroke-width="1.8"/>
<text x="137" y="286" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Room 1..2000</text>
<text x="137" y="304" text-anchor="middle" font-size="10" fill="#92400E">tick loop 20 Hz</text>
<text x="137" y="319" text-anchor="middle" font-size="10" fill="#92400E">state in-memory</text>
<text x="335" y="286" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Room 2001..4000</text>
<text x="335" y="304" text-anchor="middle" font-size="10" fill="#92400E">tick loop 20 Hz</text>
<text x="533" y="286" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Room 4001..6000</text>
<text x="533" y="304" text-anchor="middle" font-size="10" fill="#92400E">tick loop 20 Hz</text>
<text x="747" y="284" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">1 room = 1 goroutine</text>
<text x="747" y="300" text-anchor="middle" font-size="10" fill="#92400E">= 1 đơn vị nhất quán</text>
<text x="747" y="316" text-anchor="middle" font-size="10" font-weight="700" fill="#D97706">KHÔNG lock giữa room</text>
<rect x="24" y="348" width="852" height="86" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="370" font-size="12" font-weight="800" fill="#374151">④ HẠ TẦNG CHIA SẺ — chỉ phục vụ việc KHÔNG nằm trong hot path</text>
<rect x="44" y="378" width="194" height="46" rx="7" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="1.6"/>
<rect x="250" y="378" width="194" height="46" rx="7" fill="#E6FBF9" stroke="#14B8A6" stroke-width="1.6"/>
<rect x="456" y="378" width="194" height="46" rx="7" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="662" y="378" width="194" height="46" rx="7" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.6"/>
<text x="141" y="396" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Matchmaking</text>
<text x="141" y="413" text-anchor="middle" font-size="10" fill="#BE123C">xếp player vào room</text>
<text x="347" y="396" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Redis — presence</text>
<text x="347" y="413" text-anchor="middle" font-size="10" fill="#0F766E">presence:{uid} → SET</text>
<text x="553" y="396" text-anchor="middle" font-size="11" font-weight="700" fill="#3730A3">Redis Pub/Sub</text>
<text x="553" y="413" text-anchor="middle" font-size="10" fill="#4338CA">fan-out giữa GW node</text>
<text x="759" y="396" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Snapshot store</text>
<text x="759" y="413" text-anchor="middle" font-size="10" fill="#92400E">ghi định kỳ, KHÔNG đồng bộ</text>
<line x1="450" y1="118" x2="450" y2="126" stroke="#57534E" stroke-width="2" marker-end="url(#ar-a)"/>
<line x1="450" y1="226" x2="450" y2="234" stroke="#0D9488" stroke-width="2" marker-end="url(#ar-t)"/>
<line x1="450" y1="338" x2="450" y2="346" stroke="#D97706" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#ar-a)"/>
<rect x="24" y="446" width="418" height="162" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#ar-sh)"/>
<rect x="458" y="446" width="418" height="162" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#ar-sh)"/>
<text x="44" y="470" font-size="13" font-weight="800" fill="#134E4A">✅ HOT PATH (phải &lt; 50 ms)</text>
<text x="44" y="494" font-size="11" fill="#0F766E">client → GW → room server → tick → GW → client</text>
<text x="44" y="516" font-size="11" font-weight="700" fill="#134E4A">Toàn bộ nằm trong RAM. KHÔNG có:</text>
<text x="44" y="536" font-size="11" fill="#0F766E">✗ database query    ✗ HTTP call ra ngoài</text>
<text x="44" y="556" font-size="11" fill="#0F766E">✗ Redis round-trip  ✗ cross-region hop</text>
<text x="44" y="582" font-size="11" font-weight="700" fill="#0F766E">Mỗi thứ thêm vào đây đều ăn vào ngân sách 50ms</text>
<text x="44" y="600" font-size="11" fill="#0F766E">— và p99 thì luôn tệ hơn p50 nhiều lần.</text>
<text x="478" y="470" font-size="13" font-weight="800" fill="#9F1239">❌ KHÔNG được nằm trong hot path</text>
<text x="478" y="494" font-size="11" fill="#BE123C">• Ghi snapshot state xuống DB → async, theo chu kỳ</text>
<text x="478" y="514" font-size="11" fill="#BE123C">• Cập nhật leaderboard → qua queue</text>
<text x="478" y="534" font-size="11" fill="#BE123C">• Ghi analytics / telemetry → fire-and-forget</text>
<text x="478" y="554" font-size="11" fill="#BE123C">• Anti-cheat phân tích sâu → ngoài luồng</text>
<text x="478" y="582" font-size="11" font-weight="700" fill="#DC2626">Quy tắc: hot path chỉ đọc/ghi RAM cục bộ.</text>
<text x="478" y="600" font-size="11" fill="#BE123C">Mọi I/O khác đẩy sang đường async.</text>
</svg>
</div>

Điểm thiết kế quan trọng nhất ở sơ đồ trên: **tách Gateway (giữ connection) khỏi Room Server (giữ state)**. Lý do:

- Hai tầng **scale theo hai chiều khác nhau**: Gateway scale theo *số connection* và *bandwidth*; Room Server scale theo *số room* và *CPU của tick loop*.
- Gateway có thể deploy/restart mà **không làm mất game state**.
- Khi một Gateway chết, client reconnect sang Gateway khác và **room vẫn đang chạy** — đây là thứ cứu bạn khỏi failure mode tệ nhất (xem phần cuối).

---

# Lựa chọn component và tradeoff

## Tầng 1 — Edge: định tuyến connection

| Lựa chọn | Ưu | Nhược | Dùng khi |
|---|---|---|---|
| **L4 LB** (NLB, HAProxy TCP) | Thông lượng cao, latency thấp, giữ TCP xuyên suốt | Không hiểu protocol, không route theo nội dung | **Mặc định cho WebSocket/TCP** |
| **L7 LB** (ALB, Envoy, nginx) | Route theo path/header, TLS termination, observability tốt | Thêm 1 hop, tốn CPU, giới hạn connection thường thấp hơn | Cần route theo nội dung |
| **DNS/Anycast trực tiếp** | Không có LB nào ở giữa → latency thấp nhất | Client phải tự xử lý failover, khó drain node | Gaming có client tự viết |
| **Client nhận danh sách node** | Client tự chọn, tự retry, tự đo RTT | Logic phức tạp ở client, cần API discovery | **Tốt nhất cho gaming** |

> 💡 **Khuyến nghị cho gaming**: pattern **"discovery API + client-side connect"**. Client gọi một HTTP API nhẹ (`GET /gateway/assign`) để nhận `{gateway_host, token, ttl}`, rồi kết nối trực tiếp. Lợi ích: discovery API là nơi lý tưởng để làm **admission control** và **rải đều reconnect** (xem hot path #3), điều mà L4 LB không làm được.

## Tầng 1b — Protocol

| Protocol | Latency | Qua được firewall/proxy | Phù hợp |
|---|---|---|---|
| **WebSocket over TLS** | Tốt | ✅ Gần như mọi nơi (port 443) | **Mặc định an toàn** |
| **Raw TCP** | Tốt hơn chút (ít overhead frame) | ❌ Hay bị chặn | Client desktop/mobile tự viết |
| **QUIC / WebTransport** | **Tốt nhất** — không head-of-line blocking ở tầng transport, 0-RTT reconnect | Hỗ trợ còn lẻ, cần UDP mở | Khi latency là tối thượng |
| **UDP tự định nghĩa** | Thấp nhất | ❌ Phải tự làm reliability, NAT traversal | FPS, fighting game |

**Tradeoff cốt lõi**: TCP đảm bảo thứ tự nhưng **một packet mất làm nghẽn cả stream** (head-of-line blocking). Với game state, bạn thường **không cần** state cũ — nhận được state mới nhất là đủ. Đó là lý do game action-heavy dùng UDP và tự làm reliability *chỉ cho những message cần thiết*.

## Tầng 2 — Gán player vào Room

| Cách | Ưu | Nhược |
|---|---|---|
| **Consistent hashing** `hash(room_id) % ring` | Không cần service tra cứu, O(1), client tự tính được | Rebalance khi thêm/bớt node làm **di chuyển room đang chạy** |
| **Directory service** (Redis: `room:{id} → node`) | Linh hoạt tuyệt đối, đặt room ở đâu cũng được, dễ drain node | Thêm 1 lần tra cứu, và directory thành single point | 
| **Static assignment** (room range cố định) | Đơn giản nhất, dễ debug | Không tự cân bằng, node nóng node nguội |

> 💡 **Khuyến nghị**: **Directory service**, nhưng chỉ tra **một lần lúc join room**, rồi cache ở Gateway suốt vòng đời connection. Như vậy directory **không nằm trong hot path** — nó chỉ bị gọi 20.000 lần (số room) chứ không phải 10 triệu lần/giây.
>
> Consistent hashing nghe hấp dẫn nhưng với room **có state đang sống**, việc rebalance là một migration thật — khác hẳn rebalance cache (chỉ việc mất cache). Đừng chọn nó chỉ vì "nghe đúng kiểu phân tán".

## Tầng 4 — Fan-out giữa các Gateway node

Vấn đề: player A ở Gateway-1, player B cùng room ở Gateway-7. Room server phải đẩy state tới cả hai.

| Cách | Latency | Độ bền | Nhược |
|---|---|---|---|
| **Redis Pub/Sub** | ~0,2–1 ms | **Mất tin nếu GW offline** | Mất tin — nhưng [với WS thì OK](/posts/system-design-p03-pubsub-vs-streams/): client tự lấy lại state khi reconnect |
| **NATS** | ~0,1–0,5 ms | Mất tin (core NATS) | Thêm 1 hệ thống phải vận hành |
| **Direct gRPC mesh** (GW ↔ Room) | **Thấp nhất** | Không có broker | N×M connection, tự làm discovery |
| **Kafka** | ~5–20 ms | Bền | **Quá chậm cho hot path 50ms** |

> 💡 **Khuyến nghị**: **Direct gRPC stream từ Room Server tới Gateway**. Room server biết chính xác player nào ở Gateway nào (từ lúc join), nên nó gửi thẳng — không cần broker. Redis Pub/Sub dùng cho **presence và sự kiện ngoài hot path** (friend online, chat toàn server).
>
> **Tuyệt đối không dùng Kafka trong hot path** của hệ thống 50ms. Kafka sinh ra cho độ bền, không cho latency — xem [P04](/posts/system-design-p04-streams-vs-kafka/).

---

# Hot path #1 — Fan-out 1→N

Đây là chỗ tải thật nằm. Mỗi tick, room server phải đẩy state cho **mọi** player trong room. Tôi đo chi phí fan-out thật trên một Go server:

| N subscriber | Thời gian **enqueue** | Thời gian **giao xong** | Nhận được |
|---:|---:|---:|---:|
| 1.000 | 1,48 ms | 6 ms | 1.000/1.000 |
| 10.000 | 13,7 ms | 23 ms | 10.000/10.000 |
| 50.000 | **79,0 ms** | **107 ms** | 50.000/50.000 |

Đọc bảng này cùng với ngân sách **p99 < 50 ms**:

- Fan-out tới **1.000 subscriber mất 6 ms** — chiếm 12% ngân sách. Chấp nhận được.
- Fan-out tới **50.000 subscriber mất 107 ms** — **đã vượt gấp đôi ngân sách** trước khi làm bất cứ việc gì khác.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="fb-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chi phí fan-out so với ngân sách latency 50 ms</text>
<text x="440" y="48" text-anchor="middle" font-size="11" fill="#78716C">Đo thật: thời gian giao 1 message cho N subscriber trên một Go server</text>
<line x1="110" y1="310" x2="560" y2="310" stroke="#57534E" stroke-width="2"/>
<line x1="110" y1="310" x2="110" y2="76" stroke="#57534E" stroke-width="2"/>
<line x1="110" y1="194" x2="560" y2="194" stroke="#DC2626" stroke-width="2.5" stroke-dasharray="7 5"/>
<rect x="564" y="182" width="130" height="24" rx="6" fill="#DC2626"/>
<text x="629" y="199" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">ngân sách 50 ms</text>
<rect x="160" y="296" width="80" height="14" rx="3" fill="#0D9488"/>
<rect x="290" y="256" width="80" height="54" rx="3" fill="#14B8A6"/>
<rect x="420" y="60" width="80" height="250" rx="3" fill="#DC2626"/>
<text x="200" y="288" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">6 ms</text>
<text x="330" y="248" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">23 ms</text>
<text x="460" y="52" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">107 ms</text>
<text x="200" y="330" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">N = 1.000</text>
<text x="200" y="346" text-anchor="middle" font-size="10" fill="#0F766E">12% ngân sách</text>
<text x="330" y="330" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">N = 10.000</text>
<text x="330" y="346" text-anchor="middle" font-size="10" fill="#0F766E">46% ngân sách</text>
<text x="460" y="330" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">N = 50.000</text>
<text x="460" y="346" text-anchor="middle" font-size="10" font-weight="800" fill="#DC2626">214% — VỠ</text>
<text x="76" y="190" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 76 190)">thời gian giao xong</text>
<rect x="580" y="230" width="276" height="100" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#fb-sh)"/>
<text x="600" y="254" font-size="12" font-weight="800" fill="#134E4A">Hệ quả thiết kế</text>
<text x="600" y="276" font-size="11" fill="#0F766E">Giới hạn room 50 player KHÔNG</text>
<text x="600" y="294" font-size="11" fill="#0F766E">phải lựa chọn tối ưu hoá —</text>
<text x="600" y="312" font-size="11" font-weight="700" fill="#134E4A">nó là ràng buộc toán học từ</text>
<text x="600" y="326" font-size="11" font-weight="700" fill="#134E4A">ngân sách latency.</text>
<rect x="580" y="76" width="276" height="92" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<text x="600" y="98" font-size="12" font-weight="800" fill="#374151">Số đo chi tiết</text>
<text x="600" y="120" font-size="10" font-family="monospace" fill="#57534E">N=1.000  enqueue 1,48ms</text>
<text x="600" y="138" font-size="10" font-family="monospace" fill="#57534E">N=10.000 enqueue 13,7ms</text>
<text x="600" y="156" font-size="10" font-family="monospace" font-weight="700" fill="#9F1239">N=50.000 enqueue 79,0ms</text>
<text x="440" y="378" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Chi phí tăng gần như tuyến tính theo N — nên N là biến bạn PHẢI kiểm soát, không phải để nó tự lớn.</text>
</svg>
</div>

👉 **Đây chính là lý do kiến trúc phải partition theo room 50 player, không phải broadcast toàn server.** Giới hạn fan-out không phải lựa chọn tối ưu hoá — nó là ràng buộc toán học từ ngân sách latency.

Code của tầng broadcast:

```go
// internal/gateway/hub.go
type Conn struct {
    id   int64
    ws   *websocket.Conn
    out  chan []byte   // buffer riêng cho mỗi connection — KHÔNG dùng chung
}

type Room struct {
    mu      sync.RWMutex
    members map[int64]*Conn
}

// Broadcast chạy trong tick loop — PHẢI không bao giờ block.
func (r *Room) Broadcast(payload []byte) (sent, dropped int) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for _, c := range r.members {
        select {
        case c.out <- payload:
            sent++
        default:
            // Buffer đầy = client này không theo kịp.
            // DROP message của NÓ, không làm chậm 49 player còn lại.
            dropped++
            c.markSlow()
        }
    }
    return
}

// Mỗi connection có đúng MỘT goroutine ghi. Không goroutine nào khác
// được gọi ws.Write — nếu không sẽ data race trên WebSocket frame.
func (c *Conn) writeLoop() {
    defer c.ws.Close()
    for payload := range c.out {
        c.ws.SetWriteDeadline(time.Now().Add(5 * time.Second))
        if err := c.ws.WriteMessage(websocket.BinaryMessage, payload); err != nil {
            return
        }
    }
}
```

Ba chi tiết quan trọng:

1. **`c.out` là buffered channel riêng cho mỗi connection.** Dùng chung một channel cho cả room sẽ biến fan-out thành tuần tự.
2. **`select` + `default`** biến broadcast thành non-blocking. Vì sao bắt buộc — xem hot path #2.
3. **Đúng một goroutine ghi mỗi connection.** `gorilla/websocket` không an toàn khi ghi đồng thời; hai goroutine cùng `WriteMessage` sẽ làm **frame lẫn vào nhau** và client disconnect với lỗi protocol khó debug.

## Giảm chi phí fan-out

| Kỹ thuật | Hiệu quả | Đánh đổi |
|---|---|---|
| **Delta encoding** — chỉ gửi phần đổi | Giảm 60–90% bandwidth | Cần giữ last-sent state mỗi player |
| **Area of Interest** — chỉ gửi entity trong tầm nhìn | Giảm mạnh ở map lớn | Cần spatial index, tốn CPU |
| **Tick rate thích ứng** — xa thì 5 Hz, gần thì 20 Hz | Giảm 50–70% message | Logic phức tạp hơn |
| **Serialize MỘT LẦN, gửi N lần** | Giảm CPU rất nhiều | Chỉ được khi payload giống nhau cho mọi player |
| **Batch nhiều tick vào 1 frame** | Ít syscall hơn | Thêm latency bằng kích thước batch |

Kỹ thuật thứ tư rẻ nhất và hay bị bỏ qua:

```go
// ❌ SAI: serialize lại cho từng player — 50× công việc
for _, c := range r.members {
    b, _ := proto.Marshal(state)      // 50 lần marshal CÙNG một state
    c.out <- b
}

// ✅ ĐÚNG: marshal một lần, chia sẻ []byte (read-only nên an toàn)
b, err := proto.Marshal(state)
if err != nil { return }
for _, c := range r.members {
    select {
    case c.out <- b:                  // cùng một slice, không copy
    default:
    }
}
```

---

# Hot path #2 — Slow consumer: lỗi âm thầm nguy hiểm nhất

Đây là failure mode tôi thấy ít được nói tới nhất, nhưng gây sự cố production nhiều nhất trong hệ thống WebSocket.

**Tình huống**: một client mất mạng kiểu "nửa sống nửa chết" (mobile đi vào hầm) — TCP chưa đóng, nhưng client không đọc dữ liệu. Buffer gửi của server đầy dần. Nếu broadcast của bạn **chặn** ở connection đó, nó chặn luôn mọi player còn lại.

Tôi đo thật: 5.000 client, **1 client không bao giờ đọc**, payload 4 KB.

| Cách viết broadcast | Kết quả |
|---|---|
| **Chặn** — `c.out <- msg` | ⚠️ **BỊ TREO ở vòng 737** (sau ~2,9 MB) — 1 client chặn broadcast tới 4.999 client còn lại |
| **Drop** — `select` + `default` | ✅ Xong **2.000 vòng**, chậm nhất 92,9 ms, chỉ **drop 1.273 msg của đúng client chậm đó**. 4.999 client khoẻ nhận **6.729.358 msg — không bị ảnh hưởng** |

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="sc-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="sc-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="sc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">1 client không đọc — hai cách viết broadcast, hai kết cục</text>
<rect x="24" y="46" width="420" height="330" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#sc-sh)"/>
<rect x="456" y="46" width="420" height="330" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#sc-sh)"/>
<text x="234" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">❌ CHẶN — c.out &lt;- msg</text>
<text x="666" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">✅ DROP — select + default</text>
<rect x="44" y="84" width="120" height="44" rx="8" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.8"/>
<rect x="476" y="84" width="120" height="44" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="104" y="102" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Broadcaster</text>
<text x="104" y="119" text-anchor="middle" font-size="10" fill="#BE123C">tick loop</text>
<text x="536" y="102" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Broadcaster</text>
<text x="536" y="119" text-anchor="middle" font-size="10" fill="#0F766E">tick loop</text>
<rect x="200" y="84" width="104" height="44" rx="8" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<rect x="632" y="84" width="104" height="44" rx="8" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<text x="252" y="102" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">client #0</text>
<text x="252" y="119" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">KHÔNG đọc</text>
<text x="684" y="102" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">client #0</text>
<text x="684" y="119" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">KHÔNG đọc</text>
<rect x="336" y="84" width="88" height="44" rx="8" fill="#F5F5F4" stroke="#A8A29E" stroke-width="1.5" stroke-dasharray="4 3"/>
<rect x="768" y="84" width="88" height="44" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="380" y="102" text-anchor="middle" font-size="11" fill="#78716C">4.999</text>
<text x="380" y="119" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">BỊ CHẶN</text>
<text x="812" y="102" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">4.999</text>
<text x="812" y="119" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">vẫn chạy</text>
<line x1="166" y1="106" x2="196" y2="106" stroke="#DC2626" stroke-width="2" marker-end="url(#sc-a)"/>
<line x1="598" y1="106" x2="628" y2="106" stroke="#0D9488" stroke-width="2" marker-end="url(#sc-t)"/>
<line x1="306" y1="106" x2="332" y2="106" stroke="#DC2626" stroke-width="2" stroke-dasharray="4 3"/>
<text x="319" y="146" text-anchor="middle" font-size="16" font-weight="800" fill="#DC2626">✗</text>
<line x1="738" y1="106" x2="764" y2="106" stroke="#0D9488" stroke-width="2" marker-end="url(#sc-t)"/>
<rect x="44" y="156" width="380" height="80" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="234" y="180" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">BỊ TREO ở vòng 737 / 2.000</text>
<text x="234" y="202" text-anchor="middle" font-size="11" fill="#BE123C">Channel buffer (64) + socket buffer kernel (~2,5 MB)</text>
<text x="234" y="220" text-anchor="middle" font-size="11" fill="#BE123C">hấp thụ ~2,9 MB đầu rồi mới treo.</text>
<rect x="476" y="156" width="380" height="80" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="666" y="180" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">XONG đủ 2.000 vòng</text>
<text x="666" y="202" text-anchor="middle" font-size="11" fill="#0F766E">Drop 1.273 msg — của ĐÚNG client chậm đó.</text>
<text x="666" y="220" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">4.999 client khoẻ nhận 6.729.358 msg.</text>
<rect x="44" y="252" width="380" height="110" rx="9" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.8"/>
<text x="62" y="274" font-size="12" font-weight="800" fill="#9F1239">Vì sao test tải thấp KHÔNG bắt được</text>
<text x="62" y="296" font-size="11" fill="#BE123C">2,9 MB nghe nhiều, nhưng ở 20 tick/s với</text>
<text x="62" y="314" font-size="11" fill="#BE123C">payload 4 KB thì chỉ là ~37 giây.</text>
<text x="62" y="336" font-size="11" font-weight="700" fill="#DC2626">Smoke test 10 giây → PASS.</text>
<text x="62" y="354" font-size="11" font-weight="700" fill="#DC2626">Production 1 phút → TREO.</text>
<rect x="476" y="252" width="380" height="110" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="494" y="274" font-size="12" font-weight="800" fill="#134E4A">Vì sao DROP là đúng, không phải thô bạo</text>
<text x="494" y="296" font-size="11" fill="#0F766E">Client không theo kịp 20 tick/s sẽ KHÔNG</text>
<text x="494" y="314" font-size="11" fill="#0F766E">BAO GIỜ theo kịp — giữ lại chỉ tích buffer.</text>
<text x="494" y="336" font-size="11" fill="#0F766E">Reconnect nạp lại state ĐẦY ĐỦ nên</text>
<text x="494" y="354" font-size="11" font-weight="700" fill="#134E4A">không mất tính đúng đắn.</text>
</svg>
</div>

Con số **737 vòng** đáng chú ý: nó không treo ngay lập tức. Channel buffer (64 slot) cộng socket send buffer của kernel (~2,5 MB mặc định) hấp thụ được gần 3 MB đầu. Nghĩa là trên môi trường test tải thấp, **bug này không bao giờ xuất hiện** — nó chỉ nổ ra ở production khi payload lớn hơn hoặc client chậm lâu hơn.

Code xử lý đầy đủ, gồm cả phần dọn client chậm:

```go
const (
    outBufSize      = 64
    slowTolerance   = 3   // drop quá 3 lần liên tiếp → coi là chết
    writeTimeout    = 5 * time.Second
)

type Conn struct {
    id         int64
    ws         *websocket.Conn
    out        chan []byte
    slowStrike atomic.Int32
    closeOnce  sync.Once
    done       chan struct{}
}

func (c *Conn) trySend(payload []byte) bool {
    select {
    case c.out <- payload:
        c.slowStrike.Store(0)        // gửi được → reset
        return true
    default:
        // Buffer đầy. Đây KHÔNG phải lỗi tạm thời cần retry —
        // nó nghĩa là client không theo kịp nhịp của server.
        if c.slowStrike.Add(1) >= slowTolerance {
            c.kick("slow consumer")   // ngắt chủ động, giải phóng tài nguyên
        }
        return false
    }
}

func (c *Conn) kick(reason string) {
    c.closeOnce.Do(func() {
        // Đóng connection làm writeLoop thoát và client buộc phải reconnect.
        // Reconnect sẽ nạp lại state đầy đủ — KHÔNG mất tính đúng đắn.
        c.ws.SetWriteDeadline(time.Now().Add(time.Second))
        c.ws.WriteControl(websocket.CloseMessage,
            websocket.FormatCloseMessage(websocket.CloseTryAgainLater, reason),
            time.Now().Add(time.Second))
        c.ws.Close()
        close(c.done)
        metrics.ConnKicked.WithLabelValues(reason).Inc()
    })
}
```

**Vì sao ngắt client chậm là ĐÚNG, không phải thô bạo:**

Một client không theo kịp 20 tick/giây sẽ **không bao giờ** theo kịp — giữ nó lại chỉ tích tụ thêm buffer. Ngắt nó ra buộc client reconnect, và reconnect **nạp lại state đầy đủ** nên không mất tính đúng đắn. Với game, nhận state mới nhất quan trọng hơn nhận đủ mọi state cũ.

> ⚠️ Nhưng việc ngắt hàng loạt lại dẫn thẳng tới hot path #3. Nếu mạng của cả một ISP bị chập, bạn có thể ngắt 50.000 client **cùng lúc** — và tất cả sẽ reconnect cùng lúc.

---

# Hot path #3 — Reconnect storm (thundering herd)

Đây là **failure mode tệ nhất** của hệ thống CCU-heavy, và là chỗ một sự cố nhỏ biến thành sự cố toàn hệ thống.

**Kịch bản**: một Gateway node giữ 100.000 connection bị chết (hoặc deploy, hoặc OOM). 100.000 client mất kết nối **cùng một thời điểm** và tất cả reconnect ngay lập tức. Mỗi reconnect cần: TLS handshake + verify token + nạp player state.

Tôi đo với 20.000 client (chia tỉ lệ xuống cho vừa máy local):

| Chiến lược client | **ĐỈNH conn/s** | Rải trong | dial p99 |
|---|---:|---:|---:|
| **Không backoff — tất cả thử lại ngay** | **57.570** | 1,23 s | **1,01 s** |
| Jitter 5 giây | 4.610 | 5,04 s | ~0 |
| Jitter 30 giây | **880** | 30,01 s | ~0 |
| Jitter 60 giây | 520 | 60,01 s | ~0 |

**Jitter 30 giây giảm đỉnh 65 lần** (57.570 → 880). Và `dial p99` từ 1,01 giây về gần 0 — nghĩa là không chỉ server dễ thở hơn, mà **chính client cũng kết nối nhanh hơn**.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="jt-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Full jitter giảm đỉnh reconnect 65 lần</text>
<text x="440" y="48" text-anchor="middle" font-size="11" fill="#78716C">Đo thật: 20.000 client cùng reconnect · trục dọc = ĐỈNH connection/giây</text>
<line x1="120" y1="306" x2="540" y2="306" stroke="#57534E" stroke-width="2"/>
<line x1="120" y1="306" x2="120" y2="72" stroke="#57534E" stroke-width="2"/>
<rect x="150" y="76" width="66" height="230" rx="3" fill="#DC2626"/>
<rect x="246" y="288" width="66" height="18" rx="3" fill="#F5A623"/>
<rect x="342" y="302" width="66" height="4" rx="2" fill="#14B8A6"/>
<rect x="438" y="304" width="66" height="2" rx="1" fill="#0D9488"/>
<text x="183" y="66" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">57.570</text>
<text x="279" y="280" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">4.610</text>
<text x="375" y="294" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">880</text>
<text x="471" y="296" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">520</text>
<text x="183" y="326" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">không backoff</text>
<text x="183" y="342" text-anchor="middle" font-size="10" fill="#BE123C">dial p99 = 1,01s</text>
<text x="279" y="326" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">jitter 5s</text>
<text x="279" y="342" text-anchor="middle" font-size="10" fill="#0F766E">↓ 12,5×</text>
<text x="375" y="326" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">jitter 30s</text>
<text x="375" y="342" text-anchor="middle" font-size="10" font-weight="800" fill="#0F766E">↓ 65×</text>
<text x="471" y="326" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">jitter 60s</text>
<text x="471" y="342" text-anchor="middle" font-size="10" fill="#0F766E">↓ 111×</text>
<text x="86" y="190" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 86 190)">đỉnh conn/giây</text>
<rect x="564" y="72" width="292" height="104" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#jt-sh)"/>
<text x="584" y="96" font-size="12" font-weight="800" fill="#9F1239">Backoff THUẦN không đủ</text>
<text x="584" y="118" font-size="11" fill="#BE123C">1s, 2s, 4s... chỉ DỊCH cả đám sang</text>
<text x="584" y="136" font-size="11" fill="#BE123C">sau — vẫn cùng lúc, chỉ muộn hơn.</text>
<text x="584" y="160" font-size="11" font-weight="800" fill="#DC2626">Bắt buộc có rand() để RẢI ĐỀU.</text>
<rect x="564" y="190" width="292" height="116" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#jt-sh)"/>
<text x="584" y="214" font-size="12" font-weight="800" fill="#134E4A">Điều phản trực giác</text>
<text x="584" y="236" font-size="11" fill="#0F766E">Jitter không chỉ cứu server —</text>
<text x="584" y="254" font-size="11" fill="#0F766E">nó làm CLIENT kết nối NHANH hơn.</text>
<text x="584" y="278" font-size="11" font-weight="700" fill="#134E4A">dial p99: 1,01s → gần 0</text>
<text x="584" y="296" font-size="10" fill="#0F766E">Vì không còn phải xếp hàng sau 20.000 người.</text>
<text x="440" y="376" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Phải jitter NGAY TỪ LẦN THỬ ĐẦU — chính lần đầu của 100.000 client mới là cái đỉnh giết server.</text>
</svg>
</div>

Quy đổi lên quy mô thật: 100.000 client không backoff sẽ tạo đỉnh khoảng **288.000 conn/s** đổ vào các node còn sống. Nếu mỗi reconnect cần 2 round-trip Redis, đó là **576.000 Redis op/s** chỉ cho việc reconnect — đủ để làm Redis thành bottleneck và khiến các node còn lại cũng chậm, rồi chết theo.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="st-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="st-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="460" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cascading failure: 1 node chết → cả cụm chết theo</text>
<rect x="30" y="52" width="152" height="76" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#st-sh)"/>
<rect x="214" y="52" width="152" height="76" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#st-sh)"/>
<rect x="398" y="52" width="152" height="76" rx="10" fill="#FFD9DE" stroke="#DC2626" stroke-width="2" filter="url(#st-sh)"/>
<rect x="582" y="52" width="152" height="76" rx="10" fill="#FFD9DE" stroke="#DC2626" stroke-width="2" filter="url(#st-sh)"/>
<rect x="766" y="52" width="104" height="76" rx="10" fill="#DC2626" filter="url(#st-sh)"/>
<text x="106" y="76" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">① GW-3 chết</text>
<text x="106" y="96" text-anchor="middle" font-size="10" fill="#BE123C">100k connection</text>
<text x="106" y="112" text-anchor="middle" font-size="10" fill="#BE123C">mất cùng lúc</text>
<text x="290" y="76" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">② Reconnect ngay</text>
<text x="290" y="96" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">đỉnh ~288.000 conn/s</text>
<text x="290" y="112" text-anchor="middle" font-size="10" fill="#BE123C">(quy đổi từ số đo)</text>
<text x="474" y="76" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">③ Redis quá tải</text>
<text x="474" y="96" text-anchor="middle" font-size="10" fill="#BE123C">576.000 op/s cho auth</text>
<text x="474" y="112" text-anchor="middle" font-size="10" fill="#BE123C">+ nạp state</text>
<text x="658" y="76" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">④ GW còn lại chậm</text>
<text x="658" y="96" text-anchor="middle" font-size="10" fill="#BE123C">handshake p99 &gt; 1s</text>
<text x="658" y="112" text-anchor="middle" font-size="10" fill="#BE123C">→ health check fail</text>
<text x="818" y="80" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">⑤ SẬP</text>
<text x="818" y="100" text-anchor="middle" font-size="10" fill="#FFD9DE">toàn bộ</text>
<text x="818" y="114" text-anchor="middle" font-size="10" fill="#FFD9DE">1M CCU</text>
<line x1="184" y1="90" x2="210" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#st-arr)"/>
<line x1="368" y1="90" x2="394" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#st-arr)"/>
<line x1="552" y1="90" x2="578" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#st-arr)"/>
<line x1="736" y1="90" x2="762" y2="90" stroke="#DC2626" stroke-width="2" marker-end="url(#st-arr)"/>
<path d="M658,132 C600,160 380,160 290,134" stroke="#DC2626" stroke-width="2" stroke-dasharray="5 4" fill="none" marker-end="url(#st-arr)"/>
<text x="474" y="158" text-anchor="middle" font-size="11" font-weight="800" fill="#DC2626">node chết thêm → càng nhiều client reconnect → vòng lặp tự khuếch đại</text>
<text x="450" y="194" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">Bốn lớp phòng vệ — phải có ĐỦ CẢ BỐN</text>
<rect x="30" y="208" width="204" height="112" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#st-sh)"/>
<rect x="246" y="208" width="204" height="112" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#st-sh)"/>
<rect x="462" y="208" width="204" height="112" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#st-sh)"/>
<rect x="678" y="208" width="192" height="112" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2" filter="url(#st-sh)"/>
<text x="132" y="230" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">① CLIENT</text>
<text x="46" y="252" font-size="11" fill="#0F766E">Backoff + full jitter</text>
<text x="46" y="270" font-size="11" fill="#0F766E">Đo được: đỉnh giảm</text>
<text x="46" y="288" font-size="11" font-weight="800" fill="#134E4A">65× với jitter 30s</text>
<text x="46" y="310" font-size="10" fill="#0F766E">Rẻ nhất, hiệu quả nhất</text>
<text x="348" y="230" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">② ADMISSION</text>
<text x="262" y="252" font-size="11" fill="#0F766E">Discovery API cấp</text>
<text x="262" y="270" font-size="11" fill="#0F766E">token có TTL + queue</text>
<text x="262" y="288" font-size="11" fill="#0F766E">Server quyết định</text>
<text x="262" y="306" font-size="11" fill="#0F766E">AI được vào, KHI NÀO</text>
<text x="564" y="230" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">③ CACHE</text>
<text x="478" y="252" font-size="11" fill="#0F766E">Cache token verify</text>
<text x="478" y="270" font-size="11" fill="#0F766E">trong RAM gateway</text>
<text x="478" y="288" font-size="11" fill="#0F766E">+ singleflight khi</text>
<text x="478" y="306" font-size="11" fill="#0F766E">nạp state (xem P02)</text>
<text x="774" y="230" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">④ CAPACITY</text>
<text x="694" y="252" font-size="11" fill="#0F766E">N+2 node, mỗi node</text>
<text x="694" y="270" font-size="11" fill="#0F766E">chỉ dùng &lt; 60% tải</text>
<text x="694" y="288" font-size="11" fill="#0F766E">1 node chết = 1/10</text>
<text x="694" y="306" font-size="11" fill="#0F766E">connection, không 1/2</text>
<rect x="30" y="334" width="840" height="112" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="50" y="358" font-size="13" font-weight="800" fill="#7C4A03">Hai sai lầm phổ biến khi chống reconnect storm</text>
<text x="50" y="382" font-size="11" fill="#92400E"><tspan font-weight="700">1. Chỉ có backoff, không có jitter.</tspan> Backoff thuần (1s, 2s, 4s...) chỉ DỊCH cả đám sang sau — vẫn cùng lúc,</text>
<text x="50" y="400" font-size="11" fill="#92400E">chỉ là muộn hơn. Phải có <tspan font-family="monospace" font-weight="700">rand()</tspan> để RẢI ĐỀU. Đây là lỗi tôi gặp nhiều nhất trong code thật.</text>
<text x="50" y="424" font-size="11" fill="#92400E"><tspan font-weight="700">2. Chỉ dựa vào client.</tspan> Client cũ không update được, client bị hack, hoặc đơn giản là có bug —</text>
<text x="50" y="440" font-size="11" fill="#92400E">bạn <tspan font-weight="700">không kiểm soát được client</tspan>. Server BẮT BUỘC phải có admission control của riêng nó.</text>
</svg>
</div>

## Code: backoff + full jitter ở client

```go
// Full jitter — thuật toán AWS khuyến nghị. KHÔNG phải backoff thuần.
func reconnectWithJitter(ctx context.Context, dial func(context.Context) (*websocket.Conn, error)) (*websocket.Conn, error) {
    const (
        base    = 500 * time.Millisecond
        maxWait = 60 * time.Second
    )
    for attempt := 0; ; attempt++ {
        // Lần đầu tiên cũng phải chờ — vì sự cố là ĐỒNG LOẠT,
        // mọi client đều ở attempt 0 cùng một thời điểm.
        window := base << min(attempt, 7)       // 0.5s → 64s
        if window > maxWait {
            window = maxWait
        }
        // FULL JITTER: ngủ ngẫu nhiên trong [0, window), không phải đúng window.
        // Đây là điểm khác biệt giữa "đỉnh 57.570/s" và "đỉnh 880/s".
        wait := time.Duration(rand.Int63n(int64(window)))

        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(wait):
        }

        conn, err := dial(ctx)
        if err == nil {
            return conn, nil
        }
        if attempt >= 12 {
            return nil, fmt.Errorf("hết lượt thử: %w", err)
        }
    }
}
```

> ⚠️ **Chi tiết dễ sai nhất**: phải jitter **ngay từ lần thử đầu tiên**. Rất nhiều implementation viết "thử ngay, thất bại thì mới backoff" — nhưng trong reconnect storm, **chính lần thử đầu tiên của 100.000 client mới là cái đỉnh giết server**. Lần thử đầu cũng phải được rải.

## Code: admission control ở server

Client không đáng tin — phải có cửa chặn phía server:

```go
// internal/presentation/gateway/admission.go

// Token bucket toàn cục cho việc NHẬN connection mới.
// Tách riêng khỏi rate limit của request thường.
type Admission struct {
    rdb      *redis.Client
    perSec   int                  // số connection mới cho phép mỗi giây
    logger   *zap.Logger
}

var admitScript = redis.NewScript(`
  local t = redis.call('TIME')
  local nowMs = tonumber(t[1])*1000 + math.floor(tonumber(t[2])/1000)
  local d = redis.call('HMGET', KEYS[1], 'tokens','ts')
  local tokens, ts = tonumber(d[1]), tonumber(d[2])
  local rate, burst = tonumber(ARGV[1]), tonumber(ARGV[2])
  if tokens == nil then tokens = burst; ts = nowMs end
  tokens = math.min(burst, tokens + math.max(0, nowMs-ts)/1000.0*rate)
  local ok = 0
  if tokens >= 1 then tokens = tokens - 1; ok = 1 end
  redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', nowMs)
  redis.call('PEXPIRE', KEYS[1], 60000)
  return ok
`)

// Assign là HTTP endpoint client gọi TRƯỚC khi mở WebSocket.
// Nó là nơi duy nhất quyết định "được vào hay phải chờ".
func (a *Admission) Assign(c *gin.Context) {
    userID := c.GetString(auth.ContextUserID)

    ok, err := admitScript.Run(c, a.rdb, []string{"gw:admit"},
        a.perSec, a.perSec*2).Int64()
    if err != nil {
        // Redis chết → KHÔNG cho vào ồ ạt. Fail closed.
        c.JSON(503, gin.H{"retry_after_ms": 2000 + rand.Intn(3000)})
        return
    }
    if ok != 1 {
        // Hết token: trả 503 + retry_after CÓ JITTER.
        // Server tự rải client thay vì tin client sẽ tự rải.
        c.Header("Retry-After", "1")
        c.JSON(503, gin.H{
            "reason":         "gateway_saturated",
            "retry_after_ms": 1000 + rand.Intn(5000),
        })
        return
    }

    gw, err := a.pickLeastLoadedGateway(c)
    if err != nil {
        c.JSON(503, gin.H{"retry_after_ms": 2000 + rand.Intn(3000)})
        return
    }
    c.JSON(200, gin.H{
        "gateway": gw.Host,
        "token":   a.mintShortLivedToken(userID, gw.ID),  // TTL 30s
        "ttl_ms":  30000,
    })
}
```

Ba điều đáng nói trong đoạn này:

1. **`retry_after_ms` có jitter do SERVER sinh ra.** Không tin client tự rải — server chỉ định luôn thời điểm quay lại, và mỗi client nhận một số khác nhau.
2. **Token bucket dùng `redis.call('TIME')`, không dùng thời gian của client.** Tôi đo thật ở [P01.3](/posts/scalable-p01-3-rps-deep-dive/): dùng client time làm rate limiter **cho qua vượt 40%** so với giới hạn.
3. **Fail closed khi Redis lỗi.** Admission control mà "lỗi thì cho qua" là không có admission control — đúng lúc cần nhất thì nó mở cửa.

---

# Kế hoạch dung lượng

Từ workload model và số đo, tính ra cấu hình:

| Tài nguyên | Phép tính | Kết quả | Ràng buộc |
|---|---|---:|---|
| RAM cho connection | 1M × 3.501 B | **3,4 GB** | Không phải vấn đề |
| Bandwidth ra | 20M msg/s × 200 B | **32 Gbps** | **Ràng buộc chính** |
| Số Gateway node | 32 Gbps ÷ 3,2 Gbps/node | **10 node** | NIC 25G dùng ~60% |
| Connection mỗi GW | 1M ÷ 10 | 100.000 | Đã đo: khả thi |
| CPU cho fan-out | 20M msg/s, ~54k msg/s/core (đo ở P01) | ~370 core | **Phải tối ưu payload** |
| Room server | 20.000 room × 20 Hz | 400k tick/s | 1 goroutine/room |

> 📌 **Con số 370 core khiến thiết kế phải đổi.** Nó đến từ phép đo ở P01: một core xử lý ~54.000 msg/s. Với 20M msg/s thì cần ~370 core — tức khoảng 30 máy 12-core, chỉ để **ghi message ra socket**.
>
> Đây là lý do delta encoding và adaptive tick rate không phải "tối ưu hoá sau" mà là **yêu cầu thiết kế**: giảm 70% message là giảm từ 370 core xuống ~110 core, tức tiết kiệm ~20 máy.

**Vì sao mỗi node chỉ dùng 60% năng lực**: nếu 10 node đều chạy 100% và một node chết, 9 node còn lại phải gánh 111% — chúng sẽ chết tiếp. Ở mức 60%, một node chết đẩy phần còn lại lên 67% — vẫn sống. Đây là **N+2 capacity planning**, và nó là lớp phòng vệ thứ tư ở sơ đồ trên.

---

# Stress test và monitoring

## Cách tìm breaking point

Load test cho hệ thống CCU **khác hẳn** hệ thống RPS: bạn cần giữ connection, không chỉ bắn request. Trình tự nên làm:

1. **Đo chi phí một connection trước.** Mở 10k/50k/100k connection **idle**, ghi RSS và CPU. Đây là baseline — và như số đo cho thấy, CPU phải **gần 0**. Nếu idle đã tốn CPU thì có bug (thường là timer hoặc poll loop sai).
2. **Thêm tải dần ở connection count cố định.** Giữ 100k connection, tăng msg/s từng bậc tới khi p99 vỡ ngưỡng. Đây là con số capacity thật của một node.
3. **Test reconnect storm riêng.** Đây là bước hay bị bỏ qua nhất. Kill một node khi đang ở tải đỉnh và đo: đỉnh conn/s, thời gian hồi phục, có cascading không.
4. **Test slow consumer.** Mở một connection rồi cố tình không đọc. Nếu broadcast của bạn chặn, bạn sẽ thấy ngay — như số đo trên, nó treo sau ~2,9 MB.
5. **Soak test 24 giờ.** Rò rỉ goroutine và rò rỉ connection chỉ hiện ra sau nhiều giờ. Theo dõi `go_goroutines` và RSS: **tăng đều không giảm = rò rỉ**.

> ⚠️ Lưu ý về công cụ: khi test từ **một máy** bạn sẽ chạm trần **ephemeral port (~28.000 connection cho mỗi source IP)** trước khi chạm trần server. Tôi gặp đúng lỗi này — phải bind nhiều địa chỉ loopback (`127.0.0.2`–`127.0.0.9`) mới lên được 100k. Trên môi trường thật, dùng nhiều máy tạo tải.

## Metric bắt buộc phải có

| Nhóm | Metric | Vì sao quan trọng |
|---|---|---|
| **Connection** | `conn_active` theo node | Lệch giữa các node = LB không cân |
| | `conn_opened/s`, `conn_closed/s` | Đỉnh = reconnect storm đang diễn ra |
| | `handshake_duration` p50/p99 | p99 tăng = Redis/auth đang là bottleneck |
| **Fan-out** | `broadcast_duration` p99 theo room size | So trực tiếp với ngân sách 50 ms |
| | **`out_queue_depth`** p99 theo conn | **Tín hiệu sớm nhất** của slow consumer |
| | `msg_dropped/s`, `conn_kicked/s` | Tăng vọt = mạng người dùng có vấn đề |
| **Saturation** | `nic_tx_bytes/s` so với năng lực NIC | Bandwidth là ràng buộc chính, không phải CPU |
| | `go_goroutines` | Rò rỉ goroutine |
| | GC pause, `heap_inuse` | Xem [Go Interview P2](/posts/go-interview-p2/) |
| **Hot path** | `tick_duration` p99 theo room | Tick vượt 50 ms = game bị lag |
| | `tick_skipped/s` | Room server không theo kịp nhịp |

**Metric quan trọng nhất và hay bị thiếu nhất: `out_queue_depth`.** Nó là độ sâu buffer gửi của mỗi connection. Khi nó tăng, bạn còn **vài giây** trước khi bắt đầu drop. Nó cảnh báo sớm hơn mọi metric khác — sớm hơn cả latency, vì latency chỉ tăng *sau khi* buffer đã đầy.

```go
// Export histogram cho queue depth — lấy mẫu định kỳ, không phải mỗi lần gửi
func (h *Hub) sampleQueueDepth() {
    t := time.NewTicker(5 * time.Second)
    defer t.Stop()
    for range t.C {
        h.mu.RLock()
        for _, c := range h.conns {
            metrics.OutQueueDepth.Observe(float64(len(c.out)))
        }
        h.mu.RUnlock()
    }
}
```

## Checklist vận hành

```
TRƯỚC KHI LÊN PRODUCTION
□ Đã đo chi phí RAM/CPU của connection idle chưa? (phải gần 0% CPU)
□ Broadcast có dùng select+default (non-blocking) không?
□ Mỗi connection có ĐÚNG 1 goroutine ghi không?
□ Có cơ chế kick slow consumer sau N lần drop không?
□ Client có backoff + FULL JITTER ngay từ lần thử ĐẦU TIÊN không?
□ Server có admission control độc lập với client không? Fail closed chưa?
□ retry_after do SERVER sinh và có jitter chưa?
□ Capacity N+2 chưa? Mỗi node dùng dưới 60%?
□ Đã test kill 1 node ở tải đỉnh chưa? Có cascading không?
□ Hot path có còn DB/Redis call nào không? (phải không có)
□ Snapshot state là async hay đang nằm trong tick loop?

MONITORING
□ out_queue_depth p99 (tín hiệu sớm nhất)
□ conn_opened/s (phát hiện reconnect storm)
□ broadcast_duration p99 so với ngân sách latency
□ nic_tx_bytes/s so với năng lực NIC
□ go_goroutines (rò rỉ)
□ Alert khi conn_active lệch > 20% giữa các node
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| Kiến trúc cốt lõi? | **Tách Gateway (connection) khỏi Room Server (state)** — hai chiều scale khác nhau |
| Hot path gồm gì? | Chỉ RAM cục bộ. **Không DB, không Redis, không cross-region** |
| Ràng buộc thật là gì? | **Bandwidth 32 Gbps**, không phải CPU hay RAM (RAM chỉ 3,4 GB) |
| Vì sao room chỉ 50 player? | Đo thật: fan-out 50k subscriber mất **107 ms** — vượt gấp đôi ngân sách 50 ms |
| Lỗi broadcast nguy hiểm nhất? | **Blocking send.** Đo thật: 1 client không đọc **treo broadcast tới 4.999 client** sau 2,9 MB |
| Cách sửa? | `select` + `default` → drop của riêng client đó, rồi kick sau N lần |
| Failure mode tệ nhất? | **Reconnect storm.** 100k client reconnect = đỉnh ~288.000 conn/s → cascading |
| Backoff thuần đủ chưa? | **Không.** Phải có **full jitter**. Đo thật: jitter 30s giảm đỉnh **65×** |
| Chỉ dựa vào client được không? | **Không.** Client không update được, có bug, bị hack — server phải có admission control riêng |
| Metric cảnh báo sớm nhất? | **`out_queue_depth`** — tăng trước cả latency |

Bài tiếp theo: [**P01.3 — Thiết kế hệ thống 1M RPS**](/posts/scalable-p01-3-rps-deep-dive/), với flash sale và burst 50× trong 10 giây.

Và nếu bạn muốn so sánh **cùng 1M CCU nhưng cho chat thay vì gaming** — nơi fan-out không chặn trên và hot path bắt buộc có ghi bền: [**P01.4 — Chat/Notification 1M CCU**](/posts/scalable-p01-4-chat-deep-dive/).

---

# Tham khảo

- [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — nguồn của thuật toán full jitter
- [Google SRE — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) · [Handling Overload](https://sre.google/sre-book/handling-overload/)
- [gorilla/websocket — Concurrency](https://pkg.go.dev/github.com/gorilla/websocket#hdr-Concurrency) — quy tắc 1 goroutine ghi
- [Gaffer On Games — Networking for Game Programmers](https://gafferongames.com/categories/game-networking/) — vì sao game action dùng UDP
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — CCU không phải RPS](/posts/scalable-p01-ccu-vs-rps/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/) · [P04 — Streams vs Kafka](/posts/system-design-p04-streams-vs-kafka/)
