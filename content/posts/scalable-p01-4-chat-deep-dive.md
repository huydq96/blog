+++
date = '2026-10-22T02:00:00+07:00'
draft = false
slug = 'scalable-p01-4-chat-deep-dive'
title = 'Scalable System Design P01.4: Thiết Kế Hệ Thống 1M CCU - Chat/Notification'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "scalability", "websocket", "golang", "chat"]
description = 'Cùng 1M CCU nhưng chat khác gaming ở bốn chỗ cốt lõi. Đo thật: push vs pull fan-out, chi phí ghi bền trong hot path, và serialize-per-recipient đắt 5.217 lần.'
+++

> *Bài viết thuộc series **Scalable System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, kiến trúc, code và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên máy local (Go 1.24, MySQL 8.0.40, Redis 7.4.2).*

# Mở đầu

[P01.2](/posts/scalable-p01-2-ccu-deep-dive/) đã đào sâu 1M CCU cho **gaming**. Bài này cùng con số 1M CCU, nhưng cho **chat/notification** — và nó là một hệ thống khác hẳn.

Nếu bạn nghĩ "cũng là 1M WebSocket, chắc giống nhau", bốn dòng dưới đây là lý do không phải:

| | Gaming (P01.2) | **Chat/Notification (bài này)** |
|---|---|---|
| Nguồn sinh tải | **Đồng hồ** (tick 20 Hz) | **Hành vi + hình dạng xã hội** |
| msg/s ra | 20.000.000 — **cố định** | 8.333 → 10.000.000 — **dao động 1.200×** |
| Fan-out | **Chặn trên** ở 50 (room) | **Không chặn trên** (channel 1M member) |
| Mất tin | **Được phép** — tick sau sửa | **Không được phép** — phải lưu và giao bù |
| Partition | 1 chiều (theo room) | **2 chiều** (connection theo user, routing theo channel) |
| Ngân sách p99 | < 50 ms | < 500 ms (rộng gấp **10×**) |
| Hot path | Chỉ RAM | **Bắt buộc có ghi bền** |

Dòng thứ hai là điều đáng chú ý nhất: **"1M CCU chat" chưa nói gì cả** cho đến khi biết phân bố kích thước hội thoại.

## Workload model

Giả định: 10% user active, mỗi người gửi 1 tin/phút, trung bình 2,5 thiết bị/user.

| Chỉ số | Giá trị |
|---|---:|
| Connection | 1.000.000 |
| **msg/s VÀO** | **1.667** |
| RAM cho connection | 3,4 GB (3.501 B/conn — đo ở [P01](/posts/scalable-p01-ccu-vs-rps/)) |

Chiều vào **thấp hơn gaming 6.000 lần** (1.667 vs 10.000.000). Nhưng chiều ra thì tuỳ hình dạng hội thoại:

| Hội thoại | fan-out | msg/s RA | Bandwidth |
|---|---:|---:|---:|
| 1:1 DM | 2 | 8.333 | 0,01 Gbps |
| Nhóm 50 người | 50 | 208.333 | 0,3 Gbps |
| Nhóm 1.000 người | 1.000 | 4.166.667 | 6,7 Gbps |
| **1 channel livestream 1M viewer** | **1.000.000** | **10.000.000** | **16 Gbps** |

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ch-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng "1M CCU chat" — tải chênh nhau 1.200 lần</text>
<text x="440" y="48" text-anchor="middle" font-size="11" fill="#78716C">Chiều VÀO cố định ở 1.667 msg/s · chiều RA phụ thuộc HÌNH DẠNG hội thoại</text>
<line x1="150" y1="300" x2="790" y2="300" stroke="#57534E" stroke-width="2"/>
<line x1="150" y1="300" x2="150" y2="76" stroke="#57534E" stroke-width="2"/>
<rect x="186" y="298" width="76" height="2" rx="1" fill="#0D9488"/>
<rect x="306" y="294" width="76" height="6" rx="2" fill="#14B8A6"/>
<rect x="426" y="206" width="76" height="94" rx="3" fill="#F5A623"/>
<rect x="546" y="86" width="76" height="214" rx="3" fill="#DC2626"/>
<rect x="666" y="80" width="76" height="220" rx="3" fill="#57534E"/>
<text x="224" y="290" text-anchor="middle" font-size="11" font-weight="800" fill="#0F766E">8.333</text>
<text x="344" y="286" text-anchor="middle" font-size="11" font-weight="800" fill="#0F766E">208.333</text>
<text x="464" y="198" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">4,17M</text>
<text x="584" y="78" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">10M</text>
<text x="704" y="72" text-anchor="middle" font-size="11" font-weight="800" fill="#374151">20M</text>
<text x="224" y="322" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">1:1 DM</text>
<text x="224" y="337" text-anchor="middle" font-size="9" fill="#78716C">fan-out 2</text>
<text x="344" y="322" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">Nhóm 50</text>
<text x="344" y="337" text-anchor="middle" font-size="9" fill="#78716C">fan-out 50</text>
<text x="464" y="322" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">Nhóm 1.000</text>
<text x="464" y="337" text-anchor="middle" font-size="9" fill="#78716C">fan-out 1.000</text>
<text x="584" y="322" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">Livestream 1M</text>
<text x="584" y="337" text-anchor="middle" font-size="9" fill="#78716C">fan-out 1M</text>
<text x="704" y="322" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">GAMING</text>
<text x="704" y="337" text-anchor="middle" font-size="9" font-weight="700" fill="#57534E">tham chiếu</text>
<text x="116" y="190" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 116 190)">msg/s RA</text>
<rect x="150" y="358" width="640" height="34" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="470" y="380" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">Livestream chat gần gaming hơn là gần DM — dù cả hai đều được gọi là "chat"</text>
</svg>
</div>

👉 **Hệ quả thiết kế**: không thể có một kiến trúc chat duy nhất cho mọi kích thước hội thoại. Phải **tier hoá theo fan-out** — và đó là nội dung chính của bài này.

---

# Khác biệt kiến trúc #1: partition hai chiều

Đây là khác biệt cấu trúc sâu nhất so với gaming, và nó quyết định mọi thứ phía sau.

**Gamer thuộc đúng MỘT room.** Nên pin connection vào node giữ room đó là xong — mọi thứ cục bộ, không cần tra cứu.

**User chat thuộc HÀNG TRĂM conversation cùng lúc.** Không thể partition theo conversation, vì user nằm trong nhiều conversation. Bạn buộc phải tách hai chiều:

- **Connection** partition theo `user_id` (user ở node nào)
- **Message routing** partition theo `conversation_id` (tin của hội thoại nào đi đâu)

Rồi phải **join hai chiều đó lúc giao tin**. Đó là lý do chat cần một tầng **fan-out router** mà gaming không cần.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="c2-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="c2-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="c2-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="560" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chat cần tầng router mà gaming không cần</text>
<rect x="24" y="42" width="420" height="188" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#c2-sh)"/>
<text x="234" y="66" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">GAMING — 1 chiều, cục bộ</text>
<rect x="48" y="80" width="110" height="40" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.8"/>
<rect x="186" y="80" width="110" height="40" rx="7" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.8"/>
<rect x="324" y="80" width="96" height="40" rx="7" fill="#FFF3DB" stroke="#D97706" stroke-width="2"/>
<text x="103" y="105" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Player</text>
<text x="241" y="105" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Gateway</text>
<text x="372" y="98" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Room</text>
<text x="372" y="113" text-anchor="middle" font-size="10" fill="#92400E">(chỉ 1)</text>
<line x1="160" y1="100" x2="182" y2="100" stroke="#D97706" stroke-width="2" marker-end="url(#c2-a)"/>
<line x1="298" y1="100" x2="320" y2="100" stroke="#D97706" stroke-width="2" marker-end="url(#c2-a)"/>
<rect x="48" y="138" width="372" height="76" rx="9" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="66" y="160" font-size="11" font-weight="800" fill="#7C4A03">Vì sao đơn giản</text>
<text x="66" y="180" font-size="11" fill="#92400E">Player ↔ Room là quan hệ 1:1 tại một thời điểm.</text>
<text x="66" y="198" font-size="11" fill="#92400E">Pin connection vào node giữ room → mọi thứ CỤC BỘ.</text>
<rect x="456" y="42" width="420" height="188" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#c2-sh)"/>
<text x="666" y="66" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">CHAT — 2 chiều, phải join</text>
<rect x="476" y="80" width="92" height="40" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="590" y="80" width="92" height="40" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<rect x="704" y="80" width="156" height="40" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="522" y="105" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">User</text>
<text x="636" y="98" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Gateway</text>
<text x="636" y="113" text-anchor="middle" font-size="9" fill="#0F766E">theo user_id</text>
<text x="782" y="98" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">FAN-OUT ROUTER</text>
<text x="782" y="113" text-anchor="middle" font-size="9" fill="#0F766E">theo conversation_id</text>
<line x1="570" y1="100" x2="586" y2="100" stroke="#0D9488" stroke-width="2" marker-end="url(#c2-t)"/>
<line x1="684" y1="100" x2="700" y2="100" stroke="#0D9488" stroke-width="2" marker-end="url(#c2-t)"/>
<rect x="476" y="138" width="384" height="76" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="494" y="160" font-size="11" font-weight="800" fill="#134E4A">Vì sao phức tạp</text>
<text x="494" y="180" font-size="11" fill="#0F766E">1 user thuộc HÀNG TRĂM conversation cùng lúc.</text>
<text x="494" y="198" font-size="11" font-weight="700" fill="#134E4A">Connection theo user ≠ routing theo conversation → PHẢI join.</text>
<text x="450" y="262" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">Luồng một tin nhắn đi qua hệ thống chat</text>
<rect x="30" y="278" width="128" height="58" rx="9" fill="#0D9488" filter="url(#c2-sh)"/>
<rect x="182" y="278" width="128" height="58" rx="9" fill="#14B8A6" filter="url(#c2-sh)"/>
<rect x="334" y="278" width="150" height="58" rx="9" fill="#DC2626" filter="url(#c2-sh)"/>
<rect x="508" y="278" width="150" height="58" rx="9" fill="#F5A623" filter="url(#c2-sh)"/>
<rect x="682" y="278" width="188" height="58" rx="9" fill="#6366F1" filter="url(#c2-sh)"/>
<text x="94" y="302" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">① Gateway</text>
<text x="94" y="320" text-anchor="middle" font-size="10" fill="#CCFBF1">nhận tin</text>
<text x="246" y="302" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">② Cấp seq</text>
<text x="246" y="320" text-anchor="middle" font-size="10" fill="#CCFBF1">thứ tự/hội thoại</text>
<text x="409" y="296" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">③ GHI BỀN</text>
<text x="409" y="312" text-anchor="middle" font-size="10" fill="#FFD9DE">Redis WAL</text>
<text x="409" y="328" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">mới được ACK</text>
<text x="583" y="296" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">④ Router</text>
<text x="583" y="312" text-anchor="middle" font-size="10" fill="#FFF3DB">tra member list</text>
<text x="583" y="328" text-anchor="middle" font-size="10" fill="#FFF3DB">→ user ở node nào</text>
<text x="776" y="296" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">⑤ Giao</text>
<text x="776" y="312" text-anchor="middle" font-size="10" fill="#E0E7FF">online → push ngay</text>
<text x="776" y="328" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">offline → vào inbox</text>
<line x1="160" y1="307" x2="178" y2="307" stroke="#57534E" stroke-width="2" marker-end="url(#c2-a)"/>
<line x1="312" y1="307" x2="330" y2="307" stroke="#57534E" stroke-width="2" marker-end="url(#c2-a)"/>
<line x1="486" y1="307" x2="504" y2="307" stroke="#57534E" stroke-width="2" marker-end="url(#c2-a)"/>
<line x1="660" y1="307" x2="678" y2="307" stroke="#57534E" stroke-width="2" marker-end="url(#c2-a)"/>
<rect x="30" y="352" width="840" height="96" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<text x="50" y="376" font-size="13" font-weight="800" fill="#9F1239">Bước ③ là điểm chat khác gaming nhiều nhất</text>
<text x="50" y="400" font-size="11" fill="#BE123C">Gaming: hot path chỉ RAM, miss tick thì tick sau sửa. Chat: <tspan font-weight="700">không được mất tin nào</tspan>, nên PHẢI ghi bền</text>
<text x="50" y="418" font-size="11" fill="#BE123C">TRƯỚC khi ACK cho người gửi. Nếu ACK rồi mới ghi, crash giữa hai bước = tin biến mất nhưng người gửi thấy dấu ✓.</text>
<text x="50" y="440" font-size="11" font-weight="800" fill="#DC2626">Đo ở phần sau: đặt MySQL vào đây làm throughput tụt 11 lần so với ghi Redis WAL rồi async.</text>
<rect x="30" y="464" width="840" height="80" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="50" y="488" font-size="12" font-weight="800" fill="#374151">Bước ④ là nơi sinh ra bài toán hot channel</text>
<text x="50" y="510" font-size="11" fill="#57534E">Router phải biết "conversation này có ai, ai đang online, ở node nào". Với nhóm 50 thì rẻ.</text>
<text x="50" y="530" font-size="11" font-weight="700" fill="#9F1239">Với channel 1M member thì một tin nhắn thành 1 triệu lần tra cứu + giao — xem phần "push vs pull".</text>
</svg>
</div>

---

# Hot path #1 — Push hay Pull: quyết định đắt nhất

Với mỗi tin nhắn mới, có hai cách căn bản:

- **PUSH (fan-out on write)**: ghi tin vào **inbox của từng thành viên** ngay lúc gửi. Đọc rẻ, ghi đắt.
- **PULL (fan-out on read)**: ghi **một lần** vào log của channel, người đọc tự lấy. Ghi rẻ, đọc đắt.

Tôi đo cả hai trên Redis:

| Thành viên channel | **PUSH: ghi 1 tin** | **PULL: 1 người đọc** |
|---:|---:|---:|
| 50 | ~0 ms | 97 µs |
| 1.000 | 2 ms | 82 µs |
| 10.000 | 17 ms | 131 µs |
| **100.000** | **179 ms** | **110 µs** |

Hai đường cong hoàn toàn khác bản chất:

- **PUSH là `O(N)` theo số thành viên.** 179 ms để ghi **một** tin vào channel 100.000 người — và đó là đã dùng pipeline.
- **PULL là `O(1)`.** ~110 µs bất kể channel có 50 hay 100.000 người.

## Điểm hoà vốn

Push trả giá **một lần** (179 ms); pull trả giá **mỗi người đọc** (110 µs). Vậy:

```
179 ms ÷ 110 µs ≈ 1.627 người đọc
```

**Nếu dưới ~1.627 trong 100.000 thành viên thật sự đọc tin đó, PULL rẻ hơn.** Trên ngưỡng đó, PUSH rẻ hơn.

Và đây là chỗ số liệu gặp thực tế: trong một channel 100.000 người, **tỉ lệ đọc một tin cụ thể thường dưới 2%** — tức khoảng 2.000 người, sát ngưỡng hoà vốn. Với channel 1M member thì tỉ lệ đọc còn thấp hơn nữa, và pull thắng rõ ràng.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="pp-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="440" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">PUSH là O(N), PULL là O(1) — và điểm hoà vốn ở 1.627 người đọc</text>
<line x1="110" y1="290" x2="520" y2="290" stroke="#57534E" stroke-width="2"/>
<line x1="110" y1="290" x2="110" y2="70" stroke="#57534E" stroke-width="2"/>
<polyline points="150,288 240,286 330,272 460,80" fill="none" stroke="#DC2626" stroke-width="3"/>
<circle cx="150" cy="288" r="4" fill="#DC2626"/>
<circle cx="240" cy="286" r="4" fill="#DC2626"/>
<circle cx="330" cy="272" r="4" fill="#DC2626"/>
<circle cx="460" cy="80" r="5" fill="#DC2626"/>
<polyline points="150,284 240,285 330,283 460,284" fill="none" stroke="#0D9488" stroke-width="3"/>
<circle cx="150" cy="284" r="4" fill="#0D9488"/>
<circle cx="240" cy="285" r="4" fill="#0D9488"/>
<circle cx="330" cy="283" r="4" fill="#0D9488"/>
<circle cx="460" cy="284" r="4" fill="#0D9488"/>
<text x="470" y="74" font-size="12" font-weight="800" fill="#9F1239">179 ms</text>
<text x="472" y="288" font-size="12" font-weight="800" fill="#0F766E">110 µs</text>
<text x="150" y="312" text-anchor="middle" font-size="10" fill="#374151">50</text>
<text x="240" y="312" text-anchor="middle" font-size="10" fill="#374151">1.000</text>
<text x="330" y="312" text-anchor="middle" font-size="10" fill="#374151">10.000</text>
<text x="460" y="312" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">100.000</text>
<text x="315" y="336" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E">số thành viên channel</text>
<text x="76" y="180" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 76 180)">chi phí</text>
<rect x="130" y="86" width="14" height="12" rx="3" fill="#DC2626"/>
<rect x="130" y="106" width="14" height="12" rx="3" fill="#0D9488"/>
<text x="152" y="96" font-size="11" font-weight="700" fill="#9F1239">PUSH — ghi 1 tin</text>
<text x="152" y="116" font-size="11" font-weight="700" fill="#0F766E">PULL — 1 người đọc</text>
<rect x="556" y="66" width="320" height="120" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2" filter="url(#pp-sh)"/>
<text x="576" y="90" font-size="12" font-weight="800" fill="#374151">Điểm hoà vốn</text>
<text x="576" y="114" font-size="11" font-family="monospace" fill="#57534E">179 ms ÷ 110 µs ≈ 1.627</text>
<text x="576" y="138" font-size="11" fill="#0F766E">&lt; 1.627 người đọc → <tspan font-weight="800">PULL rẻ hơn</tspan></text>
<text x="576" y="158" font-size="11" fill="#9F1239">&gt; 1.627 người đọc → <tspan font-weight="800">PUSH rẻ hơn</tspan></text>
<text x="576" y="178" font-size="10" fill="#78716C">(trên channel 100.000 thành viên)</text>
<rect x="556" y="198" width="320" height="92" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="576" y="222" font-size="12" font-weight="800" fill="#134E4A">Thực tế thiên về PULL cho channel lớn</text>
<text x="576" y="244" font-size="11" fill="#0F766E">Channel 100.000 người: thường &lt; 2% đọc</text>
<text x="576" y="262" font-size="11" fill="#0F766E">một tin cụ thể ≈ 2.000 người — sát ngưỡng.</text>
<text x="576" y="282" font-size="11" font-weight="700" fill="#134E4A">Channel 1M: tỉ lệ đọc còn thấp hơn → PULL thắng rõ.</text>
<rect x="24" y="356" width="852" height="74" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="380" font-size="13" font-weight="800" fill="#7C4A03">Lời giải thực tế: KHÔNG chọn một, mà TIER HOÁ theo kích thước</text>
<text x="44" y="402" font-size="11" fill="#92400E">Nhóm nhỏ (&lt; 1.000) → <tspan font-weight="700">PUSH</tspan>: đọc tức thì, unread counter chính xác, ghi vẫn rẻ.</text>
<text x="44" y="422" font-size="11" fill="#92400E">Channel lớn (&gt; 10.000) → <tspan font-weight="700">PULL</tspan>: ghi O(1), người đọc tự lấy. Đổi lại unread counter chỉ gần đúng.</text>
</svg>
</div>

## Code: tier hoá theo kích thước channel

```go
// internal/application/chat/fanout.go

const pushThreshold = 1000  // dưới ngưỡng → push; trên → pull

type FanoutService struct {
    rdb    *redis.Client
    router Router          // biết user nào đang ở gateway nào
    logger *zap.Logger
}

func (s *FanoutService) Deliver(ctx context.Context, msg Message) error {
    n, err := s.rdb.SCard(ctx, "conv:members:"+msg.ConvID).Result()
    if err != nil {
        return err
    }

    if n <= pushThreshold {
        return s.fanoutOnWrite(ctx, msg)   // PUSH
    }
    return s.fanoutOnRead(ctx, msg)        // PULL
}

// PUSH — ghi vào inbox từng người. O(N) nhưng N nhỏ.
func (s *FanoutService) fanoutOnWrite(ctx context.Context, msg Message) error {
    members, err := s.rdb.SMembers(ctx, "conv:members:"+msg.ConvID).Result()
    if err != nil {
        return err
    }
    // Pipeline là bắt buộc: 1.000 lệnh tuần tự = 1.000 round-trip.
    pipe := s.rdb.Pipeline()
    for _, uid := range members {
        if uid == msg.SenderID {
            continue
        }
        pipe.LPush(ctx, "inbox:"+uid, msg.ID)
        pipe.LTrim(ctx, "inbox:"+uid, 0, 499)   // giới hạn inbox, tránh phình vô hạn
        pipe.Incr(ctx, "unread:"+uid+":"+msg.ConvID)
    }
    if _, err := pipe.Exec(ctx); err != nil {
        return err
    }
    // Chỉ push realtime cho người ĐANG online
    return s.router.PushToOnline(ctx, members, msg)
}

// PULL — ghi 1 lần vào log channel. O(1) bất kể N.
func (s *FanoutService) fanoutOnRead(ctx context.Context, msg Message) error {
    if err := s.rdb.XAdd(ctx, &redis.XAddArgs{
        Stream: "conv:log:" + msg.ConvID,
        MaxLen: 10000, Approx: true,          // dấu ~ : cắt O(1), xem P01 series
        Values: map[string]any{"id": msg.ID, "sender": msg.SenderID, "body": msg.Body},
    }).Err(); err != nil {
        return err
    }
    // Chỉ push cho người đang MỞ channel này — không phải mọi member
    return s.router.PushToActiveViewers(ctx, msg.ConvID, msg)
}
```

Chi tiết dễ sai nhất ở nhánh PULL: **`PushToActiveViewers`, không phải `PushToOnline`.** Trong channel 1M member có thể có 200.000 người online, nhưng chỉ 5.000 người **đang mở** channel đó. Push cho 200.000 người là làm lại đúng bài toán `O(N)` mà pull vừa tránh được.

---

# Hot path #2 — Ghi bền trong hot path

Chat **không được mất tin**. Nhưng ghi bền ở đâu và lúc nào thì quyết định throughput. Tôi đo 4 chiến lược, 3.000 tin, 60 worker:

| Chiến lược | **msg/s** | p50 | p95 | p99 |
|---|---:|---:|---:|---:|
| ① Push luôn, không lưu (gaming-style) | 2.912.621 | ~0 | ~0 | ~0 |
| ② **MySQL `INSERT` rồi mới push** | **6.946** | 8,22 ms | 13,09 ms | **15,29 ms** |
| ③ Redis Stream `XADD` rồi push | 68.950 | 653 µs | 1,24 ms | 8,84 ms |
| ④ **Redis `XADD` + MySQL async** | **78.216** | 725 µs | 1,17 ms | **1,42 ms** |

Hai kết luận:

**1. Đặt MySQL vào hot path làm throughput tụt 11 lần** (6.946 vs 78.216) và p99 xấu đi **10,8 lần** (15,29 ms vs 1,42 ms).

**2. Ghi Redis WAL rồi đẩy MySQL sang async cho throughput gần bằng không-ghi-gì** — mà vẫn không mất tin.

> 📌 Hai cách ③ và ④ cho throughput khá gần nhau (68.950 vs 78.216, chênh 13%). Tôi không kết luận ④ nhanh hơn ③ một cách chắc chắn — khác biệt chính trong test của tôi là ③ ghi vào 100 stream khác nhau còn ④ ghi vào 1 stream, nên con số có thể bị ảnh hưởng bởi chi phí trim `MAXLEN` trên nhiều key. Điều chắc chắn là **cả hai đều nhanh hơn MySQL-trong-hot-path khoảng 10 lần**.

## Code: write-ahead log rồi async

```go
// internal/application/chat/send.go

func (s *Service) Send(ctx context.Context, cmd SendCommand) (Result, error) {
    // ① Cấp sequence number cho hội thoại — đảm bảo THỨ TỰ
    seq, err := s.rdb.Incr(ctx, "conv:seq:"+cmd.ConvID).Result()
    if err != nil {
        return Result{}, shared.NewDomainError(shared.ErrCodeUnavailable, "không cấp được seq")
    }

    msg := Message{
        ID: ulid.Make().String(), ConvID: cmd.ConvID,
        SenderID: cmd.SenderID, Seq: seq, Body: cmd.Body,
        SentAt: time.Now(),
    }

    // ② GHI BỀN trước khi ACK. Đây là write-ahead log.
    //    Redis Stream có AOF nên tin sống sót qua restart.
    if err := s.rdb.XAdd(ctx, &redis.XAddArgs{
        Stream: "wal:messages",
        MaxLen: 1_000_000, Approx: true,
        Values: map[string]any{
            "id": msg.ID, "conv": msg.ConvID, "sender": msg.SenderID,
            "seq": msg.Seq, "body": msg.Body,
        },
    }).Err(); err != nil {
        // KHÔNG ack cho người gửi. Thà báo lỗi còn hơn hiện dấu ✓ rồi mất tin.
        return Result{}, shared.NewDomainError(shared.ErrCodeUnavailable, "gửi thất bại, thử lại")
    }

    // ③ Giờ mới ACK — người gửi thấy dấu ✓ và tin đã BỀN
    // ④ Fan-out và ghi MySQL diễn ra ở worker riêng, tiêu thụ wal:messages
    return Result{MessageID: msg.ID, Seq: seq}, nil
}
```

Thứ tự ②→③ là thứ dễ làm sai nhất. Nếu ACK trước rồi mới ghi, một lần crash giữa hai bước nghĩa là **người gửi thấy dấu ✓ nhưng tin không tồn tại** — lỗi mà người dùng sẽ không bao giờ tha thứ, và bạn cũng không có cách nào phát hiện sau đó.

Worker tiêu thụ WAL phải **idempotent** — xem lại [P03](/posts/system-design-p03-pubsub-vs-streams/#vấn-đề-2--xử-lý-xong-rồi-chết-trước-khi-ack):

```go
func (w *PersistWorker) handle(ctx context.Context, m redis.XMessage) error {
    msgID := m.Values["id"].(string)
    // Unique constraint trên message_id làm lớp bảo đảm cuối
    _, err := w.db.ExecContext(ctx,
        `INSERT INTO messages (message_id, conv_id, sender_id, seq, body)
         VALUES (?,?,?,?,?) ON DUPLICATE KEY UPDATE message_id = message_id`,
        msgID, m.Values["conv"], m.Values["sender"], m.Values["seq"], m.Values["body"])
    if err != nil {
        return err   // KHÔNG ack → sẽ được giao lại
    }
    return w.rdb.XAck(ctx, "wal:messages", w.group, m.ID).Err()
}
```

---

# Hot path #3 — Khi payload khác nhau theo từng người nhận

Ở [P01.2](/posts/scalable-p01-2-ccu-deep-dive/#hot-path-1--fan-out-1n) tôi viết rằng tối ưu fan-out lớn nhất là **"serialize một lần, gửi N lần"** — cùng một `[]byte` cho cả 50 player.

Nhưng chat/notification thường **không** được phép làm thế. Lý do: **privacy filtering**. Người nhận A được thấy nội dung, người nhận B chỉ được thấy số tiền, người nhận C không được thấy gì cả.

Đây đúng là mô hình của [Honeydue](https://github.com/huydq96): `hidden` không push cho partner, `balance_only` chỉ push `{amount, category, transaction_id}` và không bao giờ push description.

Tôi đo chi phí thật của hai cách, payload 239 bytes:

| fan-out N | **serialize 1 lần** | **serialize N lần** | Chênh |
|---:|---:|---:|---:|
| 2 | 843 ns | 3 µs | **3,4×** |
| 50 | 856 ns | 52 µs | **61×** |
| 1.000 | 585 ns | 904 µs | **1.545×** |
| **10.000** | **1,52 µs** | **7,92 ms** | **5.217×** |

Để ý cột giữa: **serialize-once gần như không đổi theo N** (~1 µs) — vì nó là `O(1)`. Cột phải là `O(N)`.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="sz-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="sz-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="420" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Privacy filtering phá vỡ tối ưu fan-out lớn nhất</text>
<rect x="24" y="46" width="408" height="162" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#sz-sh)"/>
<text x="228" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">GAMING — payload CHUNG</text>
<rect x="44" y="82" width="86" height="40" rx="7" fill="#0D9488"/>
<text x="87" y="100" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">state</text>
<text x="87" y="114" text-anchor="middle" font-size="9" fill="#CCFBF1">marshal 1×</text>
<rect x="176" y="70" width="60" height="22" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.4"/>
<rect x="176" y="98" width="60" height="22" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.4"/>
<rect x="176" y="126" width="60" height="22" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.4"/>
<text x="206" y="85" text-anchor="middle" font-size="9" fill="#0F766E">player 1</text>
<text x="206" y="113" text-anchor="middle" font-size="9" fill="#0F766E">player 2</text>
<text x="206" y="141" text-anchor="middle" font-size="9" fill="#0F766E">player 50</text>
<line x1="132" y1="102" x2="172" y2="82" stroke="#0D9488" stroke-width="1.6" marker-end="url(#sz-a)"/>
<line x1="132" y1="102" x2="172" y2="108" stroke="#0D9488" stroke-width="1.6" marker-end="url(#sz-a)"/>
<line x1="132" y1="102" x2="172" y2="134" stroke="#0D9488" stroke-width="1.6" marker-end="url(#sz-a)"/>
<rect x="256" y="82" width="158" height="112" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.8"/>
<text x="335" y="104" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">Cùng MỘT []byte</text>
<text x="335" y="128" text-anchor="middle" font-size="18" font-weight="800" fill="#0F766E">O(1)</text>
<text x="335" y="150" text-anchor="middle" font-size="11" fill="#0F766E">~1 µs bất kể N</text>
<text x="335" y="176" text-anchor="middle" font-size="10" fill="#0F766E">N=50 → 856 ns</text>
<rect x="448" y="46" width="408" height="162" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#sz-sh)"/>
<text x="652" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">CHAT có privacy — payload RIÊNG</text>
<rect x="468" y="82" width="86" height="40" rx="7" fill="#DC2626"/>
<text x="511" y="100" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">txn</text>
<text x="511" y="114" text-anchor="middle" font-size="9" fill="#FFD9DE">marshal N×</text>
<rect x="600" y="70" width="88" height="22" rx="5" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.4"/>
<rect x="600" y="98" width="88" height="22" rx="5" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.4"/>
<rect x="600" y="126" width="88" height="22" rx="5" fill="#F5F5F4" stroke="#A8A29E" stroke-width="1.4"/>
<text x="644" y="85" text-anchor="middle" font-size="9" fill="#9F1239">chủ: FULL</text>
<text x="644" y="113" text-anchor="middle" font-size="9" fill="#7C4A03">partner: balance</text>
<text x="644" y="141" text-anchor="middle" font-size="9" fill="#78716C">khác: hidden → ✗</text>
<line x1="556" y1="102" x2="596" y2="82" stroke="#DC2626" stroke-width="1.6" marker-end="url(#sz-a)"/>
<line x1="556" y1="102" x2="596" y2="108" stroke="#DC2626" stroke-width="1.6" marker-end="url(#sz-a)"/>
<line x1="556" y1="102" x2="596" y2="134" stroke="#DC2626" stroke-width="1.6" marker-end="url(#sz-a)"/>
<rect x="702" y="82" width="140" height="112" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.8"/>
<text x="772" y="104" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">N []byte khác nhau</text>
<text x="772" y="128" text-anchor="middle" font-size="18" font-weight="800" fill="#DC2626">O(N)</text>
<text x="772" y="150" text-anchor="middle" font-size="11" fill="#BE123C">tăng tuyến tính</text>
<text x="772" y="176" text-anchor="middle" font-size="10" fill="#BE123C">N=50 → 52 µs</text>
<text x="440" y="240" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">Chênh lệch theo fan-out — đo thật</text>
<rect x="60" y="256" width="180" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="256" y="256" width="180" height="48" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<rect x="452" y="256" width="180" height="48" rx="8" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<rect x="648" y="256" width="180" height="48" rx="8" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<text x="150" y="276" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">N = 2 (couple)</text>
<text x="150" y="295" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">3,4× — không sao</text>
<text x="346" y="276" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">N = 50 (nhóm nhỏ)</text>
<text x="346" y="295" text-anchor="middle" font-size="13" font-weight="800" fill="#92400E">61×</text>
<text x="542" y="276" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">N = 1.000</text>
<text x="542" y="295" text-anchor="middle" font-size="13" font-weight="800" fill="#BE123C">1.545×</text>
<text x="738" y="276" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">N = 10.000</text>
<text x="738" y="295" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">5.217×</text>
<rect x="24" y="322" width="832" height="86" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="346" font-size="13" font-weight="800" fill="#7C4A03">Cách giảm: nhóm người nhận theo QUYỀN XEM, không theo từng người</text>
<text x="44" y="368" font-size="11" fill="#92400E">Privacy chỉ có 3 mức (hidden / balance_only / full) → chỉ cần <tspan font-weight="700">3 bản serialize</tspan>, không phải N bản.</text>
<text x="44" y="388" font-size="11" fill="#92400E">N = 10.000 người nhưng chỉ 3 biến thể payload → từ <tspan font-weight="700">7,92 ms xuống ~2,6 µs</tspan>.</text>
<text x="44" y="404" font-size="11" font-weight="800" fill="#DC2626">Chi phí quay về O(số biến thể), không còn là O(số người nhận).</text>
</svg>
</div>

## Cách sửa: serialize theo NHÓM QUYỀN, không theo từng người

Mấu chốt: số **biến thể payload** thường rất nhỏ (bằng số mức privacy), còn số **người nhận** thì lớn. Nhóm người nhận theo biến thể rồi serialize một lần cho mỗi nhóm:

```go
// internal/domain/transaction/visibility.go

// VisibilityClass là "nhóm quyền xem" — số lượng nhóm là HẰNG SỐ,
// không phụ thuộc số người nhận.
type VisibilityClass uint8

const (
    VisHidden VisibilityClass = iota  // không push gì
    VisBalanceOnly                    // chỉ amount + category + id
    VisFull                           // toàn bộ
)

// classify do DOMAIN quyết định — privacy enforcement ở domain layer,
// không phải ở presentation. Xem CLAUDE.md của Honeydue.
func (s *PrivacyService) Classify(viewerID string, txn *Transaction) VisibilityClass {
    if txn.OwnerID() == viewerID {
        return VisFull
    }
    switch txn.PrivacyLevel().Value() {
    case PrivacyHidden:
        return VisHidden
    case PrivacyBalanceOnly:
        return VisBalanceOnly
    default:
        return VisFull
    }
}
```

```go
// internal/application/notification/fanout.go

func (s *FanoutService) PushTransaction(ctx context.Context, txn *transaction.Transaction, recipients []string) error {
    // ① Nhóm người nhận theo quyền xem — O(N) nhưng chỉ là so sánh, không serialize
    groups := make(map[transaction.VisibilityClass][]string, 3)
    for _, uid := range recipients {
        cls := s.privacy.Classify(uid, txn)
        if cls == transaction.VisHidden {
            continue   // không push, và cũng không serialize
        }
        groups[cls] = append(groups[cls], uid)
    }

    // ② Serialize MỘT LẦN cho mỗi nhóm — tối đa 2 lần, không phải N lần
    for cls, uids := range groups {
        payload, err := s.encode(txn, cls)
        if err != nil {
            return err
        }
        // ③ Cùng một []byte cho cả nhóm — read-only nên an toàn chia sẻ
        if err := s.router.PushMany(ctx, uids, payload); err != nil {
            s.logger.Warn("push thất bại cho một nhóm",
                zap.Int("group_size", len(uids)), zap.Error(err))
        }
    }
    return nil
}
```

**Chi phí quay về `O(số biến thể)` thay vì `O(số người nhận)`** — với 10.000 người nhận và 3 mức privacy, đó là từ 7,92 ms xuống khoảng 2,6 µs.

> 📌 **Với Honeydue hiện tại thì chưa cần tối ưu này.** Couple account có fan-out = 2, chênh lệch chỉ 3,4× của một con số 843 ns — hoàn toàn không đáng kể. Nhưng nếu sau này có family account hoặc shared budget nhiều người, đây là chỗ chi phí sẽ nhân lên theo số người nhận, và **nó sẽ không lộ ra ở tải thấp** — giống hệt lỗi slow consumer ở [P01.2](/posts/scalable-p01-2-ccu-deep-dive/#hot-path-2--slow-consumer-lỗi-âm-thầm-nguy-hiểm-nhất) chỉ nổ sau 2,9 MB.

---

# Thứ chat có mà gaming không: offline queue và multi-device

## Offline queue

Gaming: player offline thì game cứ chạy, không cần giữ gì cho họ. Chat: **phải giữ mọi tin cho người offline** và giao khi họ quay lại.

```go
// Khi user reconnect — nạp phần bỏ lỡ theo SEQUENCE, không theo thời gian
func (s *Service) Catchup(ctx context.Context, userID string, convID string, lastSeenSeq int64) ([]Message, error) {
    // Dùng seq thay vì timestamp: đồng hồ client không đáng tin,
    // và seq cho phép phát hiện LỖ HỔNG trong dãy tin.
    curSeq, err := s.rdb.Get(ctx, "conv:seq:"+convID).Int64()
    if err != nil {
        return nil, err
    }
    gap := curSeq - lastSeenSeq
    if gap <= 0 {
        return nil, nil
    }
    // Nếu bỏ lỡ quá nhiều, KHÔNG gửi hết — bảo client tải lại từ REST.
    // Đây là van an toàn chống việc 1 reconnect kéo 10.000 tin qua WebSocket.
    if gap > 500 {
        return nil, ErrTooFarBehind   // client sẽ gọi GET /conversations/{id}/messages
    }
    return s.repo.GetBySeqRange(ctx, convID, lastSeenSeq+1, curSeq)
}
```

Van `gap > 500` là chi tiết quan trọng. Không có nó, một user offline 3 ngày reconnect sẽ kéo hàng nghìn tin qua WebSocket — và nếu 10.000 user làm thế cùng lúc sau sự cố mạng, bạn có một **reconnect storm có payload**, tệ hơn nhiều reconnect storm thường ở [P01.2](/posts/scalable-p01-2-ccu-deep-dive/#hot-path-3--reconnect-storm-thundering-herd).

## Multi-device

Một user = nhiều connection. Mô hình đúng là `presence:{user_id}` → SET của `connection_id`, đúng như Honeydue đang làm:

```go
// Push cho MỌI thiết bị của user, trừ thiết bị vừa gửi
func (r *Router) PushToUser(ctx context.Context, userID string, payload []byte, exceptConnID string) error {
    conns, err := r.rdb.SMembers(ctx, "presence:"+userID).Result()
    if err != nil {
        return err
    }
    for _, cid := range conns {
        if cid == exceptConnID {
            continue   // không echo lại cho thiết bị vừa gửi
        }
        node, err := r.rdb.Get(ctx, "conn:node:"+cid).Result()
        if err != nil {
            // Connection đã chết nhưng presence chưa dọn → dọn luôn
            r.rdb.SRem(ctx, "presence:"+userID, cid)
            continue
        }
        r.sendToNode(ctx, node, cid, payload)
    }
    return nil
}
```

Chi tiết dễ bỏ sót: **dọn presence rác**. Khi gateway chết đột ngột, `presence:{user_id}` vẫn còn `connection_id` của những connection đã mất. Nếu không dọn, số liệu presence phình dần và bạn push vào hư không. Cần cả hai: dọn khi phát hiện (như trên) **và** một job định kỳ đối chiếu với danh sách connection thật.

---

# Failure mode tệ nhất

Chat có hai failure mode mà gaming không có:

**1. Hot channel.** Một tin vào channel 1M member. Với PUSH đó là 1 triệu lần ghi — đo được 179 ms cho 100.000 member, tức khoảng **1,8 giây** cho 1M. Trong 1,8 giây đó Redis bị chiếm, mọi hội thoại khác cũng chậm theo. Đây là bài toán hot resource ở [P01](/posts/scalable-p01-ccu-vs-rps/#hot-resource-chỗ-ccu-hoàn-toàn-không-nói-gì), đặt ở tầng fan-out.

**2. Reconnect storm có payload.** Sự cố mạng → 100.000 user reconnect **và mỗi người kéo tin bỏ lỡ**. Reconnect storm thường chỉ tốn handshake; cái này tốn handshake **cộng** truy vấn lịch sử. Van `gap > 500` ở trên là lớp phòng vệ chính.

| Failure mode | Dấu hiệu | Phòng vệ |
|---|---|---|
| Hot channel | `fanout_duration` p99 tăng vọt ở vài `conv_id` | Tier hoá push/pull · tách channel lớn sang node riêng |
| Reconnect storm có payload | `catchup_messages` tổng tăng vọt cùng `conn_opened/s` | Van `gap`, đẩy sang REST · backoff + jitter |
| WAL đầy | Redis memory tăng, consumer lag tăng | `MAXLEN` trên WAL · alert theo lag |
| Presence rác | `presence_size` > số connection thật | Dọn khi phát hiện + job đối chiếu |
| Mất thứ tự | Client báo tin hiện sai thứ tự | Sequence number per conversation, không dùng timestamp |

---

# Stress test và monitoring

## Cách test

Khác với gaming, chat cần test cả **hình dạng hội thoại**, không chỉ số connection:

1. **Test theo phân bố fan-out thật.** Đừng test với toàn nhóm 50 người. Lấy phân bố thật từ production (hoặc dự đoán): ví dụ 80% DM, 19% nhóm < 100, 1% channel > 10.000. Chính 1% đó quyết định capacity.
2. **Test hot channel riêng.** Một channel lớn nhất, bắn tin liên tục, đo ảnh hưởng tới **các hội thoại khác**. Nếu p99 của DM tăng khi channel lớn hoạt động, bạn chưa có isolation.
3. **Test mất tin.** Kill gateway/worker giữa lúc tải, rồi **đếm**: số tin gửi thành công (có ACK) phải bằng số tin lưu được. Khác nhau = đang mất tin.
4. **Test catchup.** Cho 10.000 client offline 10 phút rồi reconnect cùng lúc. Đo tổng số tin kéo về và thời gian hồi phục.
5. **Test thứ tự.** Gửi 1.000 tin liên tiếp vào một hội thoại từ nhiều client, kiểm tra `seq` ở phía nhận liên tục không lỗ hổng.

## Metric bắt buộc

| Nhóm | Metric | Vì sao |
|---|---|---|
| **Fan-out** | `fanout_duration` p99 **theo bucket kích thước channel** | Trung bình vô nghĩa khi fan-out dao động 1.200× |
| | `fanout_mode` (push/pull) counter | Xác nhận tier hoá đang chạy đúng |
| | `push_recipients` histogram | Phát hiện channel đang phình |
| **Độ bền** | `wal_write_duration` p99 | Nằm trong hot path nên phải sát |
| | **`messages_acked` − `messages_persisted`** | **Phải bằng 0.** Dương = đang mất tin |
| | WAL consumer lag | Async có theo kịp không |
| **Catchup** | `catchup_gap` histogram | Phân bố độ trễ của client |
| | `catchup_too_far` counter | Bao nhiêu client phải fallback REST |
| **Presence** | `presence_size` vs `conn_active` | Lệch = presence rác |
| **Thứ tự** | `seq_gap_detected` | Phải bằng 0 |

**Metric quan trọng nhất: `messages_acked − messages_persisted`.** Nó không phải metric hiệu năng mà là **metric đúng đắn** — giống `oversell_count` ở [P01.3](/posts/scalable-p01-3-rps-deep-dive/). Nếu dương, bạn đã hiện dấu ✓ cho tin không tồn tại.

## Checklist

```
ĐÚNG ĐẮN
□ Ghi bền TRƯỚC khi ACK cho người gửi? (không phải ngược lại)
□ Có sequence number per conversation chưa? (không dùng timestamp client)
□ Worker persist có idempotent + unique constraint chưa?
□ Có đối chiếu messages_acked vs messages_persisted không?
□ Có van giới hạn catchup (gap > N → fallback REST) chưa?

FAN-OUT
□ Đã tier hoá push/pull theo kích thước channel chưa? Ngưỡng dựa trên ĐO hay đoán?
□ Nhánh pull có push cho ACTIVE VIEWER hay cho mọi người online?
□ Payload khác nhau theo người nhận? → nhóm theo QUYỀN XEM, không serialize N lần
□ Inbox có LTRIM giới hạn không? (tránh phình vô hạn)
□ Fan-out có dùng pipeline không? (1.000 lệnh tuần tự = 1.000 round-trip)

VẬN HÀNH
□ Presence có cơ chế dọn rác (khi phát hiện + job định kỳ) chưa?
□ WAL có MAXLEN chưa?
□ Đã test với PHÂN BỐ fan-out thật, không phải một kích thước duy nhất?
□ Đã test hot channel ảnh hưởng tới hội thoại khác chưa?
□ Metric fanout_duration có chia bucket theo kích thước channel chưa?
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| 1M CCU chat có giống 1M CCU gaming? | **Không.** Chiều vào thấp hơn **6.000×**, chiều ra dao động **1.200×** tuỳ hình dạng hội thoại |
| Khác biệt cấu trúc sâu nhất? | **Partition hai chiều** — connection theo user, routing theo channel. Gaming chỉ cần 1 chiều |
| Push hay pull? | Đo thật: push `O(N)` (**179 ms** cho 100k member), pull `O(1)` (**110 µs**). Hoà vốn ở **~1.627 người đọc** |
| Lời giải thực tế? | **Tier hoá**: nhóm < 1.000 → push, channel > 10.000 → pull |
| Đặt MySQL vào hot path? | Throughput tụt **11×** (6.946 vs 78.216 msg/s), p99 xấu **10,8×** |
| Cách đúng? | Ghi **Redis WAL rồi ACK**, MySQL do worker async — và ACK **sau** khi ghi bền |
| Privacy filtering tốn gì? | Serialize per-recipient đắt **5.217×** ở N=10.000 (7,92 ms vs 1,52 µs) |
| Cách giảm? | Nhóm người nhận theo **quyền xem** → `O(số biến thể)`, không phải `O(số người nhận)` |
| Failure mode riêng của chat? | **Hot channel** và **reconnect storm có payload** |
| Metric phải bằng 0? | **`messages_acked − messages_persisted`** |

Bài tiếp: [**P01.5 — Public API triệu RPS**](/posts/scalable-p01-5-api-deep-dive/), với multi-tenant quota và lý do không thể gọi Redis mỗi request.

---

# Tham khảo

- [Zanzibar / OpenFGA](https://research.google/pubs/pub48190/) — mô hình quan hệ cho quyền xem ở quy mô lớn
- [Discord — How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) — push vs pull ở quy mô thật
- [Twitter timeline fan-out](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/) — nguồn gốc của bài toán push/pull
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — CCU không phải RPS](/posts/scalable-p01-ccu-vs-rps/) · [P01.2 — 1M CCU gaming](/posts/scalable-p01-2-ccu-deep-dive/) · [P01.3 — Triệu RPS flash sale](/posts/scalable-p01-3-rps-deep-dive/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/)
