+++
date = '2026-10-21T09:00:00+07:00'
draft = false
slug = 'scalable-p01-ccu-vs-rps'
title = 'Scalable System Design P01: 1M CCU Không Phải Một Kiến Trúc'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "scalability", "performance", "interview", "golang"]
description = 'CCU không phải RPS. Đo thật: 100.000 connection idle tốn 0% CPU, và một hot row làm throughput sụp 30 lần. Cách chuyển từ con số user sang workload model.'
+++

> *Bài viết mở đầu series **Scalable System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, ví dụ, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên máy local (Go 1.24, MySQL 8.0.40, Redis 7.4.2).*

# Tổng quan

Có một câu chuyện khá quen trong các buổi phỏng vấn System Design, thường ở level Senior hoặc SA.

Người phỏng vấn đưa đề bài:

> *"Hãy thiết kế một hệ thống chịu được 1M CCU."*

Ứng viên nghe tới một triệu user thì bắt đầu vẽ Load Balancer, thêm Redis, Kafka, chia Microservice, tính chuyện sharding database, caching... Người phỏng vấn gật gù hài lòng, lâu lâu hỏi thêm về bottleneck hoặc single point of failure.

Nếu phỏng vấn kiểu này thì **10M hay 100M CCU cũng được** — vì cơ bản vẫn dùng lại đúng những pattern đó. Chán nhỉ.

Buổi phỏng vấn trên có vấn đề ở đâu?

Ở chỗ: hai bên có thể đi hết cả buổi, diagram vẽ kín bảng, nhưng **chưa chắc đã từng thống nhất với nhau "1M CCU" thật ra nghĩa là gì.**

Điểm thú vị là không chỉ ứng viên — đôi khi chính người phỏng vấn cũng đang dùng CCU như một cách nói khác của RPS. Cứ thấy CCU lớn thì mặc định request cũng lớn tương ứng, rồi áp các pattern tăng tải quen thuộc vào.

**Nhưng CCU không phải RPS.** Bài này chứng minh điều đó bằng số đo thật, rồi đi qua cách chuyển từ con số user sang một workload model dùng được.

---

# CCU không phải RPS

| | **CCU** — Concurrent Users | **RPS** — Requests Per Second |
|---|---|---|
| Đo cái gì | **Số người** đang ở trong hệ thống | **Tốc độ việc** đi vào hệ thống |
| Đơn vị | người | request/giây |
| Tăng khi nào | Nhiều người vào hơn | Mỗi người làm nhiều việc hơn |
| Ảnh hưởng chính tới | **Bộ nhớ**, connection, per-user state | **CPU**, I/O, database |
| Business dự báo được? | **Có** — dễ hình dung | Khó — phụ thuộc implementation |

Ẩn dụ dễ hình dung nhất: **CCU cho biết có bao nhiêu người đang ở trong "cửa hàng". RPS cho biết mỗi giây những người đó tạo ra bao nhiêu việc cho hệ thống xử lý.**

Một triệu người đứng trong cửa hàng nhưng chỉ nhìn ngó sẽ khác **hoàn toàn** một triệu người cùng lúc bỏ hàng vào giỏ, thanh toán và tranh nhau vài sản phẩm cuối cùng.

Và còn một lớp mơ hồ nữa: **chỉ riêng chữ "CCU" đã có ít nhất ba cách hiểu**, mỗi cách tạo ra một workload khác nhau.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 460" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="cc-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="cc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="460" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">"1M CCU" — ba cách hiểu, ba hệ thống khác nhau</text>
<rect x="24" y="48" width="278" height="196" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#cc-sh)"/>
<rect x="311" y="48" width="278" height="196" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#cc-sh)"/>
<rect x="598" y="48" width="278" height="196" rx="12" fill="#FFF7F7" stroke="#FF6B6B" stroke-width="2.5" filter="url(#cc-sh)"/>
<rect x="24" y="48" width="278" height="34" rx="12" fill="#0D9488"/>
<rect x="24" y="68" width="278" height="14" fill="#0D9488"/>
<rect x="311" y="48" width="278" height="34" rx="12" fill="#D97706"/>
<rect x="311" y="68" width="278" height="14" fill="#D97706"/>
<rect x="598" y="48" width="278" height="34" rx="12" fill="#DC2626"/>
<rect x="598" y="68" width="278" height="14" fill="#DC2626"/>
<text x="163" y="71" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">① ĐANG ONLINE</text>
<text x="450" y="71" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">② GIỮ CONNECTION</text>
<text x="737" y="71" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">③ ĐANG ACTIVE</text>
<text x="42" y="106" font-size="11" fill="#57534E">Đã đăng nhập, app còn mở,</text>
<text x="42" y="123" font-size="11" fill="#57534E">nhưng có thể đang để màn</text>
<text x="42" y="140" font-size="11" fill="#57534E">hình chờ.</text>
<rect x="42" y="152" width="242" height="34" rx="7" fill="#CCFBF1"/>
<text x="163" y="166" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">Tải thật: gần 0</text>
<text x="163" y="181" text-anchor="middle" font-size="10" fill="#0F766E">chỉ tốn session store</text>
<text x="42" y="206" font-size="11" font-weight="700" fill="#374151">Tốn gì:</text>
<text x="42" y="224" font-size="11" fill="#0F766E">session entry, presence flag</text>
<text x="329" y="106" font-size="11" fill="#57534E">1 WebSocket / long-poll /</text>
<text x="329" y="123" font-size="11" fill="#57534E">gRPC stream đang mở, dù</text>
<text x="329" y="140" font-size="11" fill="#57534E">không gửi gì.</text>
<rect x="329" y="152" width="242" height="34" rx="7" fill="#FDE9C8"/>
<text x="450" y="166" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">Tải thật: RAM, không CPU</text>
<text x="450" y="181" text-anchor="middle" font-size="10" fill="#92400E">đo được: 3.501 bytes/conn</text>
<text x="329" y="206" font-size="11" font-weight="700" fill="#374151">Tốn gì:</text>
<text x="329" y="224" font-size="11" fill="#92400E">fd, buffer, goroutine/thread</text>
<text x="616" y="106" font-size="11" fill="#57534E">Thật sự đang gửi request</text>
<text x="616" y="123" font-size="11" fill="#57534E">hoặc message trong khoảng</text>
<text x="616" y="140" font-size="11" fill="#57534E">thời gian đang xét.</text>
<rect x="616" y="152" width="242" height="34" rx="7" fill="#FFD9DE"/>
<text x="737" y="166" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Tải thật: ĐÂY mới là tải</text>
<text x="737" y="181" text-anchor="middle" font-size="10" fill="#BE123C">CPU, I/O, DB, bandwidth</text>
<text x="616" y="206" font-size="11" font-weight="700" fill="#374151">Tốn gì:</text>
<text x="616" y="224" font-size="11" fill="#BE123C">gần như MỌI tài nguyên</text>
<rect x="24" y="264" width="852" height="80" rx="12" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="46" y="288" font-size="13" font-weight="800" fill="#374151">Ẩn dụ cửa hàng</text>
<text x="46" y="312" font-size="12" fill="#57534E"><tspan font-weight="700" fill="#0D9488">CCU</tspan> = bao nhiêu người đang ở TRONG cửa hàng.   <tspan font-weight="700" fill="#DC2626">RPS</tspan> = mỗi giây những người đó tạo ra bao nhiêu VIỆC cho nhân viên.</text>
<text x="46" y="334" font-size="12" font-weight="700" fill="#7C4A03">1 triệu người đứng nhìn ngó ≠ 1 triệu người cùng thanh toán và tranh nhau vài món hàng cuối.</text>
<rect x="24" y="362" width="852" height="84" rx="12" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="46" y="386" font-size="13" font-weight="800" fill="#7C4A03">Câu hỏi đầu tiên nên hỏi trong buổi phỏng vấn</text>
<text x="46" y="410" font-size="12" fill="#92400E">"Anh/chị nói 1M CCU — ý là 1 triệu người <tspan font-weight="700">đang online</tspan>, 1 triệu <tspan font-weight="700">connection đang mở</tspan>,</text>
<text x="46" y="432" font-size="12" fill="#92400E">hay 1 triệu người <tspan font-weight="700">đang thật sự gửi request</tspan>?" — Ba câu trả lời dẫn tới ba hệ thống hoàn toàn khác nhau.</text>
</svg>
</div>

---

# Chứng minh: 1M CCU có thể bằng 0 RPS

Bài gốc có một nhận xét rất sắc: *"Bạn có thể dùng 1 server đủ mạnh, mở một triệu kết nối WebSocket vào và giữ ở đó. OK, bạn đã thành công xây dựng hệ thống chịu tải 1M CCU."*

Tôi quyết định **đo thật xem điều đó tốn bao nhiêu**. Một Go server mô hình 1 goroutine/connection, client mở connection rồi giữ nguyên, không gửi gì:

| Số connection | RSS server | **Bytes/connection** | Goroutine |
|---:|---:|---:|---:|
| 10.000 | 37,7 MB | 3.403 | 10.002 |
| 50.000 | 171,8 MB | 3.427 | 50.002 |
| **100.000** | **346,3 MB** | **3.501** | 100.002 |

Rất ổn định: **~3,5 KB cho mỗi connection đang mở**. Ngoại suy tuyến tính: **1.000.000 connection ≈ 3,4 GB RAM.** Trên một máy 37 GB RAM thì đó là chuyện nhỏ.

Giờ đến phần quan trọng. Giữ **nguyên 100.000 connection**, chỉ đổi lượng message chạy qua:

| Tải | **CPU server** | msg/s xử lý được | RSS |
|---|---:|---:|---:|
| **0 msg/s (idle hoàn toàn)** | **0,0%** | 0 | 328 MB |
| 10.000 msg/s | 29,8% | 10.010 | 378 MB |
| 100.000 msg/s (yêu cầu) | **98,2%** (bão hoà) | **54.096** | 407 MB |

Đọc kỹ dòng đầu: **100.000 connection đồng thời, CPU 0,0%.** Server hoàn toàn rảnh rỗi.

Và dòng cuối: khi yêu cầu 100.000 msg/s, server bão hoà một core và **chỉ xử lý được 54.096 msg/s** — không đạt được mức yêu cầu.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ax-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ax-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="440" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">CCU và RPS là hai trục ĐỘC LẬP — đo thật trên 100.000 connection</text>
<line x1="110" y1="330" x2="470" y2="330" stroke="#57534E" stroke-width="2" marker-end="url(#ax-arr)"/>
<line x1="110" y1="330" x2="110" y2="70" stroke="#57534E" stroke-width="2" marker-end="url(#ax-arr)"/>
<text x="300" y="358" text-anchor="middle" font-size="12" font-weight="700" fill="#DC2626">RPS / msg mỗi giây  →  quyết định CPU</text>
<text x="76" y="200" text-anchor="middle" font-size="12" font-weight="700" fill="#0D9488" transform="rotate(-90 76 200)">CCU / connection → quyết định RAM</text>
<rect x="128" y="246" width="96" height="76" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="248" y="246" width="96" height="76" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<rect x="368" y="246" width="96" height="76" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="176" y="270" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">100k conn</text>
<text x="176" y="288" text-anchor="middle" font-size="11" fill="#0F766E">0 msg/s</text>
<rect x="140" y="296" width="72" height="18" rx="4" fill="#0D9488"/>
<text x="176" y="309" text-anchor="middle" font-size="10" font-weight="800" fill="#FFFFFF">CPU 0,0%</text>
<text x="296" y="270" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">100k conn</text>
<text x="296" y="288" text-anchor="middle" font-size="11" fill="#92400E">10k msg/s</text>
<rect x="260" y="296" width="72" height="18" rx="4" fill="#D97706"/>
<text x="296" y="309" text-anchor="middle" font-size="10" font-weight="800" fill="#FFFFFF">CPU 29,8%</text>
<text x="416" y="270" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">100k conn</text>
<text x="416" y="288" text-anchor="middle" font-size="11" fill="#BE123C">bão hoà</text>
<rect x="380" y="296" width="72" height="18" rx="4" fill="#DC2626"/>
<text x="416" y="309" text-anchor="middle" font-size="10" font-weight="800" fill="#FFFFFF">CPU 98,2%</text>
<rect x="128" y="96" width="336" height="130" rx="10" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" stroke-dasharray="6 4"/>
<text x="296" y="120" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">RAM gần như KHÔNG đổi theo tải</text>
<text x="146" y="144" font-size="11" fill="#0F766E">0 msg/s      → RSS 328 MB</text>
<text x="146" y="164" font-size="11" fill="#0F766E">10k msg/s   → RSS 378 MB</text>
<text x="146" y="184" font-size="11" fill="#0F766E">bão hoà      → RSS 407 MB</text>
<text x="146" y="210" font-size="11" font-weight="700" fill="#134E4A">Cùng 100k conn → RAM quyết định bởi CCU</text>
<rect x="500" y="70" width="356" height="156" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#ax-sh)"/>
<text x="520" y="96" font-size="13" font-weight="800" fill="#134E4A">Chi phí MỘT connection idle</text>
<text x="520" y="124" font-size="22" font-weight="800" fill="#0F766E">3.501 bytes</text>
<text x="520" y="148" font-size="11" fill="#0F766E">Đo tại mốc 100.000 connection,</text>
<text x="520" y="166" font-size="11" fill="#0F766E">gồm cả goroutine + buffer + fd.</text>
<text x="520" y="192" font-size="12" font-weight="800" fill="#134E4A">→ 1.000.000 conn ≈ 3,4 GB RAM</text>
<text x="520" y="214" font-size="11" fill="#0F766E">(ngoại suy tuyến tính từ 3 mốc đo)</text>
<rect x="500" y="246" width="356" height="76" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#ax-sh)"/>
<text x="520" y="270" font-size="13" font-weight="800" fill="#9F1239">Nhưng khi có tải thật</text>
<text x="520" y="292" font-size="11" fill="#BE123C">Yêu cầu 100k msg/s → chỉ đạt 54.096 msg/s</text>
<text x="520" y="312" font-size="11" font-weight="700" fill="#DC2626">Nút thắt là CPU, không phải số connection</text>
<rect x="110" y="376" width="746" height="52" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="483" y="398" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">"Chịu được 1M CCU" mà không nói tải thì có thể chỉ nghĩa là: giữ 1 triệu socket và không làm gì cả.</text>
<text x="483" y="418" text-anchor="middle" font-size="12" fill="#92400E">Tốn 3,4 GB RAM, 0% CPU. Đúng đề bài, và vô nghĩa.</text>
</svg>
</div>

Kết luận từ hai bảng trên: **số connection quyết định RAM, số message quyết định CPU.** Hai trục độc lập. Một con số CCU không nói gì về trục còn lại.

---

# Công thức map CCU sang RPS

Muốn chuyển CCU thành RPS thì tối thiểu phải biết ba thừa số:

```
RPS ≈ Active Users × (Action mỗi giây) × (Request mỗi Action)
```

Nhưng đó **mới chỉ là traffic ở lớp ngoài**. Một request còn có thể:

- **Fan-out** thành nhiều internal request (microservice gọi nhau)
- Sinh nhiều **query database**
- **Emit event** sang message broker
- Bị **retry** khi lỗi — và retry là thứ nhân tải lên đúng lúc hệ thống đang yếu nhất
- **Cache hit hay miss** làm tải xuống database khác nhau hàng chục lần

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="am-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="am-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="420" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Chuỗi khuếch đại: từ 1M CCU xuống tới DB QPS</text>
<text x="450" y="48" text-anchor="middle" font-size="11" fill="#78716C">Ví dụ e-commerce — mọi giả định đều ghi rõ, thay số của bạn vào để tự tính</text>
<rect x="30" y="68" width="132" height="64" rx="10" fill="#0D9488" filter="url(#am-sh)"/>
<rect x="194" y="68" width="132" height="64" rx="10" fill="#14B8A6" filter="url(#am-sh)"/>
<rect x="358" y="68" width="132" height="64" rx="10" fill="#F5A623" filter="url(#am-sh)"/>
<rect x="522" y="68" width="132" height="64" rx="10" fill="#D97706" filter="url(#am-sh)"/>
<rect x="686" y="68" width="132" height="64" rx="10" fill="#DC2626" filter="url(#am-sh)"/>
<text x="96" y="92" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CCU</text>
<text x="96" y="114" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">1.000.000</text>
<text x="260" y="92" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">Action/giây</text>
<text x="260" y="114" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">50.000</text>
<text x="424" y="92" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">RPS tại edge</text>
<text x="424" y="114" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">150.000</text>
<text x="588" y="92" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">RPS tới origin</text>
<text x="588" y="114" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">30.000</text>
<text x="752" y="92" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">DB QPS</text>
<text x="752" y="114" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">15.000</text>
<line x1="164" y1="100" x2="190" y2="100" stroke="#57534E" stroke-width="2" marker-end="url(#am-arr)"/>
<line x1="328" y1="100" x2="354" y2="100" stroke="#57534E" stroke-width="2" marker-end="url(#am-arr)"/>
<line x1="492" y1="100" x2="518" y2="100" stroke="#57534E" stroke-width="2" marker-end="url(#am-arr)"/>
<line x1="656" y1="100" x2="682" y2="100" stroke="#57534E" stroke-width="2" marker-end="url(#am-arr)"/>
<rect x="178" y="148" width="164" height="42" rx="7" fill="#E6FBF9" stroke="#14B8A6" stroke-width="1.6"/>
<rect x="342" y="148" width="164" height="42" rx="7" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.6"/>
<rect x="506" y="148" width="164" height="42" rx="7" fill="#FDE9C8" stroke="#D97706" stroke-width="1.6"/>
<rect x="670" y="148" width="164" height="42" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.6"/>
<text x="260" y="164" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">÷ 20</text>
<text x="260" y="182" text-anchor="middle" font-size="10" fill="#0F766E">1 action / 20 giây / user</text>
<text x="424" y="164" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">× 3</text>
<text x="424" y="182" text-anchor="middle" font-size="10" fill="#92400E">3 API call mỗi action</text>
<text x="588" y="164" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">× 0,2</text>
<text x="588" y="182" text-anchor="middle" font-size="10" fill="#92400E">CDN hấp thụ 80%</text>
<text x="752" y="164" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">× 5 rồi × 0,1</text>
<text x="752" y="182" text-anchor="middle" font-size="10" fill="#BE123C">fan-out 5, cache hit 90%</text>
<rect x="30" y="212" width="840" height="78" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="50" y="236" font-size="12" font-weight="800" fill="#374151">Mỗi thừa số đổi một chút là kết quả đổi rất nhiều</text>
<text x="50" y="258" font-size="11" fill="#57534E">• Cache hit tụt từ 90% xuống 70% → DB QPS tăng từ 15.000 lên <tspan font-weight="700" fill="#DC2626">45.000</tspan> (gấp 3) mà CCU không đổi một người nào.</text>
<text x="50" y="278" font-size="11" fill="#57534E">• Frontend mới gọi 6 API thay vì 3 → RPS <tspan font-weight="700" fill="#DC2626">gấp đôi</tspan>. Vẫn 1M CCU.</text>
<rect x="30" y="306" width="840" height="100" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="50" y="330" font-size="12" font-weight="800" fill="#9F1239">Hai thứ hay bị quên trong phép tính</text>
<text x="50" y="352" font-size="11" fill="#BE123C"><tspan font-weight="700">1. Burst.</tspan> Traffic đều 30.000 RPS khác hẳn trung bình 30.000 RPS nhưng dồn thành đỉnh 200.000 RPS trong 10 giây.</text>
<text x="50" y="372" font-size="11" fill="#BE123C"><tspan font-weight="700">2. Retry.</tspan> Khi hệ thống chậm, client retry → tải TĂNG đúng lúc hệ thống yếu nhất. Đây là vòng lặp tự khuếch đại.</text>
<text x="50" y="394" font-size="11" font-weight="700" fill="#DC2626">→ Luôn hỏi "đều hay burst?" và "retry policy là gì?" trước khi chốt con số RPS.</text>
</svg>
</div>

> 📌 **Chuỗi quan hệ đúng** nên là ba tầng, không phải một con số:
>
> `(Business demand) CCU + hành vi user` → `(Technical workload) RPS, message/s, bandwidth, connection, DB QPS` → `(Resource demand) CPU, memory, network, storage, connection pool`
>
> Bỏ qua tầng giữa là đi thẳng từ "một triệu user" tới "chắc cần Kafka".

---

# Cùng 1M CCU, hai bài toán khác nhau hoàn toàn

Thử đặt cùng con số 1M CCU vào hai hệ thống: **E-commerce** và **Realtime Gaming**. Tôi tính cụ thể cả hai với giả định ghi rõ.

## E-commerce — 1M CCU

| Chỉ số | Giá trị | Giả định |
|---|---:|---|
| Action/giây | 50.000 | 1 action / 20 giây / user |
| RPS tại edge | 150.000 | 3 API call mỗi action |
| **RPS tới origin** | **30.000** | CDN hấp thụ 80% |
| Internal call/giây | 150.000 | fan-out × 5 microservice |
| **DB QPS** | **15.000** | cache hit 90% |
| Bandwidth origin | 1,2 Gbps | 5 KB mỗi response |

Phần lớn user đang **browse catalog, search, xem hình** — tải **đọc**, đẩy được qua CDN, cache và read replica. Nhưng tới lúc **checkout** thì hệ thống phải quan tâm tới inventory consistency, idempotency của payment, và chuyện **không bán âm số lượng tồn kho**.

Và trong flash sale: 100.000 user vào checkout trong 60 giây = **1.667 checkout/giây** — tất cả tranh chấp **cùng vài trăm dòng tồn kho**. Con số 1.667/s nghe nhỏ so với 30.000 RPS, nhưng đây mới là phần khó nhất của cả hệ thống.

## Realtime Gaming — 1M CCU

| Chỉ số | Giá trị | Giả định |
|---|---:|---|
| Connection giữ liên tục | 1.000.000 | WebSocket/UDP sống lâu |
| **Message/giây VÀO** | **10.000.000** | 10 input/giây/player |
| **Message/giây RA** | **20.000.000** | tick rate 20/giây |
| Bandwidth VÀO | 16 Gbps | 200 bytes mỗi message |
| **Bandwidth RA** | **32 Gbps** | |
| RAM chỉ cho connection | 3,4 GB | **3.501 B/conn — đo thật** |
| DB QPS | gần 0 | state nằm trong RAM, persist theo chu kỳ |

## So sánh

| | E-commerce | Gaming | Tỉ lệ |
|---|---:|---:|---:|
| Message/RPS vào hệ thống | 30.000 | 10.000.000 | **333×** |
| Bandwidth ra | 1,2 Gbps | 32 Gbps | **26×** |
| DB QPS | 15.000 | ~0 | — |
| Nút thắt chính | **Hot resource lúc checkout** | **Network + latency** |  |

**Cùng một con số 1M CCU, nhưng gaming tạo ra 333 lần lượng việc.** Và nút thắt nằm ở hai chỗ hoàn toàn khác nhau.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="tw-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai hệ thống, cùng "1M CCU" — diagram tổng quát giống nhau, bên dưới khác hoàn toàn</text>
<rect x="24" y="48" width="420" height="412" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#tw-sh)"/>
<rect x="456" y="48" width="420" height="412" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#tw-sh)"/>
<rect x="24" y="48" width="420" height="34" rx="12" fill="#D97706"/>
<rect x="24" y="68" width="420" height="14" fill="#D97706"/>
<rect x="456" y="48" width="420" height="34" rx="12" fill="#0D9488"/>
<rect x="456" y="68" width="420" height="14" fill="#0D9488"/>
<text x="234" y="71" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">E-COMMERCE · 1M CCU</text>
<text x="666" y="71" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">REALTIME GAMING · 1M CCU</text>
<text x="44" y="106" font-size="11" font-weight="800" fill="#374151">User đang làm gì</text>
<text x="44" y="126" font-size="11" fill="#92400E">Browse catalog, search, xem ảnh.</text>
<text x="44" y="144" font-size="11" fill="#92400E">Phần lớn là tải ĐỌC.</text>
<text x="476" y="106" font-size="11" font-weight="800" fill="#374151">User đang làm gì</text>
<text x="476" y="126" font-size="11" fill="#0F766E">Giữ connection, liên tục trao đổi</text>
<text x="476" y="144" font-size="11" fill="#0F766E">state theo tick hoặc event.</text>
<rect x="44" y="158" width="380" height="96" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="1.6"/>
<rect x="476" y="158" width="380" height="96" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.6"/>
<text x="62" y="178" font-size="11" font-weight="800" fill="#7C4A03">Workload tính ra</text>
<text x="62" y="198" font-size="11" font-family="monospace" fill="#92400E">30.000 RPS tới origin</text>
<text x="62" y="216" font-size="11" font-family="monospace" fill="#92400E">15.000 DB QPS</text>
<text x="62" y="234" font-size="11" font-family="monospace" fill="#92400E">1,2 Gbps bandwidth</text>
<text x="494" y="178" font-size="11" font-weight="800" fill="#134E4A">Workload tính ra</text>
<text x="494" y="198" font-size="11" font-family="monospace" font-weight="700" fill="#0F766E">10.000.000 msg/s VÀO</text>
<text x="494" y="216" font-size="11" font-family="monospace" font-weight="700" fill="#0F766E">20.000.000 msg/s RA</text>
<text x="494" y="234" font-size="11" font-family="monospace" font-weight="700" fill="#0F766E">32 Gbps bandwidth RA</text>
<text x="44" y="278" font-size="11" font-weight="800" fill="#374151">Hot Path cần bảo vệ</text>
<rect x="44" y="288" width="380" height="54" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="234" y="308" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">CHECKOUT — tồn kho, payment</text>
<text x="234" y="328" text-anchor="middle" font-size="10" fill="#BE123C">1.667 checkout/s tranh vài trăm dòng tồn kho</text>
<text x="476" y="278" font-size="11" font-weight="800" fill="#374151">Hot Path cần bảo vệ</text>
<rect x="476" y="288" width="380" height="54" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="2"/>
<text x="666" y="308" text-anchor="middle" font-size="11" font-weight="800" fill="#3730A3">TICK LOOP — độ trễ từng frame</text>
<text x="666" y="328" text-anchor="middle" font-size="10" fill="#4338CA">p99 latency, jitter, regional placement</text>
<text x="44" y="366" font-size="11" font-weight="800" fill="#374151">Cách partition dữ liệu</text>
<text x="44" y="386" font-size="11" fill="#92400E">Theo user_id / product_id / shard DB.</text>
<text x="44" y="404" font-size="11" fill="#92400E">Dữ liệu phân tán khá đều.</text>
<text x="476" y="366" font-size="11" font-weight="800" fill="#374151">Cách partition dữ liệu</text>
<text x="476" y="386" font-size="11" fill="#0F766E">Theo room / match / vùng game.</text>
<text x="476" y="404" font-size="11" fill="#0F766E">Player cùng room PHẢI cùng node.</text>
<rect x="44" y="416" width="380" height="32" rx="7" fill="#FDE9C8"/>
<rect x="476" y="416" width="380" height="32" rx="7" fill="#CCFBF1"/>
<text x="234" y="436" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Failure mode tệ nhất: bán âm tồn kho</text>
<text x="666" y="436" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Failure mode tệ nhất: lag / mất đồng bộ state</text>
</svg>
</div>

Cả hai hệ thống **vẫn có** Load Balancer, Cache, Queue và database. Nhìn ở mức diagram tổng quát chúng thậm chí khá giống nhau. Nhưng **trách nhiệm của từng component, Hot Path cần bảo vệ, cách partition dữ liệu và failure mode quan trọng lại khác hoàn toàn.**

Không thể lấy thiết kế E-commerce, đổi chữ "product" thành "player" rồi gọi đó là hệ thống Gaming 1M CCU được. 😄

---

# Hot Resource: chỗ CCU hoàn toàn không nói gì

Đây là phần tôi muốn đào sâu, vì nó là ví dụ rõ nhất cho việc **cùng một con số CCU có thể cho ra hai khả năng chịu tải chênh nhau hàng chục lần**.

Bài toán: giảm tồn kho. 120 "user" đồng thời, 20.000 lượt mua. Chỉ đổi **một** thứ — tải dồn vào một dòng, hay rải ra 200 dòng.

| Kịch bản | Thời gian | **Throughput** | p50 | p99 |
|---|---:|---:|---:|---:|
| **MySQL — 1 hot row** (tất cả tranh `id=1`) | 51,70 s | **387 ops/s** | **309,9 ms** | 345,2 ms |
| MySQL — 200 row rải đều | 1,68 s | **11.940 ops/s** | 9,4 ms | 22,5 ms |
| Redis — 1 hot key (`DECR stock:1`) | 233 ms | 85.949 ops/s | 1,19 ms | 2,9 ms |
| Redis — 200 key rải đều | 212 ms | 94.175 ops/s | 1,15 ms | 2,8 ms |

Hai điều rút ra:

**1. Trên MySQL, một hot row làm throughput sụp 30,8 lần** (387 vs 11.940) và p50 latency tăng **33 lần** (310ms vs 9,4ms). Nguyên nhân: row lock của InnoDB **tuần tự hoá** mọi transaction đụng vào dòng đó. Thêm CPU, thêm replica, thêm connection — không cái nào giúp được, vì nút thắt là **thứ tự**, không phải năng lực.

**2. Trên Redis, hot key gần như không ảnh hưởng** (85.949 vs 94.175, chỉ 1,1×). Vì `DECR` là một lệnh atomic đơn trên một server single-thread — mọi thứ **vốn đã tuần tự**, nên tập trung vào một key không thêm chi phí tuần tự hoá nào nữa.

Đây chính là lý do kỹ thuật đằng sau việc mọi thiết kế flash sale đều **đưa bộ đếm tồn kho ra khỏi RDBMS**: không phải vì Redis "nhanh hơn", mà vì **mô hình đồng thời của nó không bị sụp khi có hot key**.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="hr-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="420" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng 120 user đồng thời — chỉ đổi chỗ tải rơi vào</text>
<text x="440" y="48" text-anchor="middle" font-size="11" fill="#78716C">20.000 lượt giảm tồn kho · MySQL 8.0.40 · đo thật</text>
<line x1="100" y1="300" x2="800" y2="300" stroke="#57534E" stroke-width="2"/>
<line x1="100" y1="300" x2="100" y2="76" stroke="#57534E" stroke-width="2"/>
<rect x="160" y="293" width="100" height="7" rx="2" fill="#DC2626"/>
<rect x="320" y="90" width="100" height="210" rx="2" fill="#F5A623"/>
<rect x="480" y="92" width="100" height="208" rx="2" fill="#14B8A6"/>
<rect x="640" y="86" width="100" height="214" rx="2" fill="#0D9488"/>
<text x="210" y="283" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">387</text>
<text x="370" y="82" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">11.940</text>
<text x="530" y="84" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">85.949</text>
<text x="690" y="78" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">94.175</text>
<text x="210" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">MySQL</text>
<text x="210" y="338" text-anchor="middle" font-size="11" font-weight="800" fill="#DC2626">1 HOT ROW</text>
<text x="370" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">MySQL</text>
<text x="370" y="338" text-anchor="middle" font-size="11" fill="#7C4A03">200 row rải đều</text>
<text x="530" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">Redis</text>
<text x="530" y="338" text-anchor="middle" font-size="11" fill="#0F766E">1 hot key</text>
<text x="690" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">Redis</text>
<text x="690" y="338" text-anchor="middle" font-size="11" fill="#134E4A">200 key rải đều</text>
<text x="66" y="190" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 66 190)">ops/giây</text>
<rect x="150" y="100" width="150" height="76" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#hr-sh)"/>
<text x="225" y="124" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">sụp 30,8×</text>
<text x="225" y="144" text-anchor="middle" font-size="11" fill="#BE123C">p50: 9,4ms → 310ms</text>
<text x="225" y="163" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">chậm hơn 33 lần</text>
<rect x="100" y="350" width="700" height="66" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="118" y="371" font-size="12" font-weight="800" fill="#7C4A03">Vì sao thêm tài nguyên không cứu được?</text>
<text x="118" y="391" font-size="11" fill="#92400E">Row lock của InnoDB TUẦN TỰ HOÁ mọi transaction đụng cùng một dòng.</text>
<text x="118" y="409" font-size="11" fill="#92400E">Nút thắt là THỨ TỰ, không phải năng lực — thêm CPU, replica hay connection đều vô ích.</text>
</svg>
</div>

Và đây là điểm then chốt cho chủ đề của bài: **"1M CCU" không hề nói cho bạn biết tải sẽ rơi vào đâu.** Một triệu user browse 200.000 sản phẩm là tải rải đều, dễ shard. Một triệu user tranh 100 suất flash sale là tải tập trung tuyệt đối — và hai thứ đó cách nhau 30 lần năng lực, trên cùng một database.

---

# Và đừng quên FR trước NFR

Bài toán chịu tải ở đây chủ yếu đang nói về **NFR** (Non-Functional Requirement): throughput, latency, availability, durability.

Nhưng nếu thiếu **FR** (Functional Requirement) — hệ thống *làm gì*, luồng nào quan trọng, cái gì bắt buộc phải đúng — thì giải pháp dù đẹp cũng vô nghĩa. Bạn không thể biết cần bảo vệ Hot Path nào nếu chưa biết Hot Path đó là gì.

| | **FR** | **NFR** |
|---|---|---|
| Trả lời câu hỏi | Hệ thống **làm gì**? | Làm **tốt tới mức nào**? |
| Ví dụ | "User đặt hàng, trừ tồn kho, gửi email" | "p99 < 200ms, 99,95% uptime" |
| Thiếu nó thì sao | **Không biết đang scale cái gì** | Không biết bao nhiêu là đủ |
| Trong đề bài 1M CCU | **Thường bị bỏ trống** | Chỉ có một con số duy nhất |

Vấn đề không phải đề bài mở. Nếu người phỏng vấn **cố tình** để đề mở nhằm xem ứng viên có biết khai thác requirement, xây workload model và đặt assumption hay không, thì đây lại là một câu hỏi **khá hay**.

Vấn đề là khi **cả hai bên cùng bỏ qua bước đó**: ứng viên vội vàng scale một hệ thống chưa được định nghĩa, còn người phỏng vấn giữ sẵn trong đầu một diagram "đúng" rồi chấm xem ứng viên có đoán trúng Kafka, Redis hay Sharding giống đáp án của mình hay không.

---

# Năm câu hỏi nên hỏi trước khi vẽ

Ứng viên tốt **chưa cần vẽ ngay**. Trước hết nên làm rõ:

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="q5-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="q5-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="480" fill="#FFFDF7" rx="16"/>
<text x="430" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Năm câu hỏi — rồi mới tới cái bút vẽ</text>
<rect x="30" y="48" width="800" height="46" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="30" y="102" width="800" height="46" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="30" y="156" width="800" height="46" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="30" y="210" width="800" height="46" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="30" y="264" width="800" height="46" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="30" y="48" width="38" height="46" rx="9" fill="#0D9488"/>
<rect x="30" y="102" width="38" height="46" rx="9" fill="#0D9488"/>
<rect x="30" y="156" width="38" height="46" rx="9" fill="#0D9488"/>
<rect x="30" y="210" width="38" height="46" rx="9" fill="#0D9488"/>
<rect x="30" y="264" width="38" height="46" rx="9" fill="#0D9488"/>
<text x="49" y="77" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">1</text>
<text x="49" y="131" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">2</text>
<text x="49" y="185" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">3</text>
<text x="49" y="239" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">4</text>
<text x="49" y="293" text-anchor="middle" font-size="15" font-weight="800" fill="#FFFFFF">5</text>
<text x="84" y="70" font-size="12" font-weight="800" fill="#374151">User đang làm gì, và đâu là các flow quan trọng nhất?</text>
<text x="84" y="87" font-size="11" fill="#57534E">→ Xác định FR. Không có bước này thì không biết đang scale cái gì.</text>
<text x="84" y="124" font-size="12" font-weight="800" fill="#374151">CCU nghĩa là online, active, hay giữ connection?</text>
<text x="84" y="141" font-size="11" fill="#57534E">→ Ba câu trả lời khác nhau. Đo được: connection idle tốn 3,5 KB và 0% CPU.</text>
<text x="84" y="178" font-size="12" font-weight="800" fill="#374151">Mỗi user tạo bao nhiêu action / request / message? Đều hay burst?</text>
<text x="84" y="195" font-size="11" fill="#57534E">→ Đây là chỗ CCU biến thành RPS. Và burst mới là thứ làm sập hệ thống.</text>
<text x="84" y="232" font-size="12" font-weight="800" fill="#374151">Dữ liệu phân tán, hay nhiều user cùng tranh một Hot Resource?</text>
<text x="84" y="249" font-size="11" fill="#57534E">→ Đo được: hot row làm throughput sụp 30,8× trên cùng một MySQL.</text>
<text x="84" y="286" font-size="12" font-weight="800" fill="#374151">Điều gì BẮT BUỘC phải đúng? Mục tiêu latency / availability / durability?</text>
<text x="84" y="303" font-size="11" fill="#57534E">→ NFR. Và "bắt buộc phải đúng" quyết định chỗ nào được phép eventual consistency.</text>
<rect x="30" y="330" width="800" height="50" rx="9" fill="#374151"/>
<text x="430" y="352" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">Rồi MỚI map hành vi user → workload → xác định Hot Path → chọn component và pattern</text>
<text x="430" y="371" text-anchor="middle" font-size="11" fill="#D6D3D1">Load Balancer, Cache, Queue, Sharding mới chỉ là VOCABULARY của System Design</text>
<rect x="30" y="392" width="800" height="76" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="430" y="418" text-anchor="middle" font-size="14" font-weight="800" fill="#7C4A03">Thay vì hỏi "Hệ thống 1M CCU nên thiết kế thế nào?"</text>
<text x="430" y="444" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">hãy hỏi: "Một triệu user đó đang làm gì, với ai,</text>
<text x="430" y="462" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">và điều gì tuyệt đối không được phép sai?"</text>
</svg>
</div>

---

# Vậy vì sao người ta vẫn bắt đầu bằng CCU?

Tôi không phản đối các bài toán 1M CCU. Con số đó **vẫn hữu ích** để đặt ra một *scale envelope* ban đầu. Và có lý do rất chính đáng để business dùng nó:

| Lý do | Giải thích |
|---|---|
| **Business hiểu user, không hiểu request** | Số người online, người chơi trong game, người xem livestream, người tham gia flash sale — đây là thứ PM và sales dự báo được |
| **CCU ít phụ thuộc implementation hơn** | Cùng lượng user, nhưng batching, caching, polling, retry hay frontend mới có thể làm **RPS thay đổi mạnh** |
| **Với realtime, CCU là đại lượng vật lý thật** | Game, chat, WebSocket, livestream: số user đồng thời **trực tiếp** quyết định connection, per-user state, presence và fan-out |
| **Trong phỏng vấn, nó là bài test** | CCU thường là đầu vào để xem ứng viên có biết chuyển từ product demand sang workload model hay không |

Nhưng **1M CCU không phải là một kiến trúc.** Load Balancer, Cache, Queue và Sharding cũng mới chỉ là *vocabulary* của System Design — biết từ vựng không có nghĩa là viết được câu.

Và điều này không chỉ đúng trong phỏng vấn. Các pattern đó **không sai** — phần lớn hệ thống lớn đều dùng một vài trong số đó. Nhưng **đúng pattern chưa có nghĩa là đúng design.**

---

# Checklist: chuyển CCU thành workload model

```bash
BƯỚC 1 — LÀM RÕ ĐỀ BÀI (chưa vẽ gì)
□ CCU nghĩa là online / giữ connection / đang active?
□ FR là gì — hệ thống làm gì, luồng nào quan trọng nhất?
□ Điều gì bắt buộc phải đúng (không được eventual consistency)?
□ Mục tiêu p50/p99 latency, availability, durability?

BƯỚC 2 — TÍNH WORKLOAD (vẫn chưa vẽ)
□ Active users × action/giây × request/action = RPS lớp ngoài
□ Fan-out: 1 request thành bao nhiêu internal call?
□ Cache hit rate bao nhiêu → DB QPS còn lại bao nhiêu?
□ Retry policy là gì? (retry nhân tải đúng lúc hệ thống yếu nhất)
□ Traffic ĐỀU hay BURST? Tỉ lệ đỉnh/trung bình bao nhiêu?
□ Kích thước payload → bandwidth bao nhiêu Gbps?

BƯỚC 3 — TÌM HOT PATH
□ Tải rải đều hay tập trung vào Hot Resource?
□ Nếu có hot row/hot key: đã biết nó sụp 30× chưa? Kế hoạch là gì?
□ Dữ liệu partition theo chiều nào? Có skew không?
□ Failure mode tệ nhất là gì — và nó xảy ra ở đâu?

BƯỚC 4 — GIỜ MỚI CHỌN COMPONENT
□ Mỗi component giải quyết CHÍNH XÁC bottleneck nào trong bảng trên?
□ Có component nào thêm vào chỉ vì "hệ thống lớn thì phải có"?
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| CCU và RPS khác nhau thế nào? | CCU = **bao nhiêu người trong cửa hàng**. RPS = **mỗi giây họ tạo bao nhiêu việc** |
| 1M CCU có thể bằng 0 RPS không? | **Có.** Đo thật: 100.000 connection idle = **0,0% CPU**, 328 MB RAM |
| Một connection idle tốn bao nhiêu? | **3.501 bytes** (đo tại mốc 100k) → 1M conn ≈ **3,4 GB RAM** |
| CCU quyết định tài nguyên nào? | **RAM** và connection. **RPS** mới quyết định CPU, I/O, DB |
| Công thức map CCU → RPS? | `Active users × action/giây × request/action`, rồi × fan-out, × (1 − cache hit) |
| Cùng 1M CCU thì workload giống nhau? | **Không.** Gaming tạo **333×** lượng message so với origin RPS của e-commerce |
| Hot Resource ảnh hưởng bao nhiêu? | Đo thật: MySQL 1 hot row **sụp 30,8×** throughput, p50 chậm **33×** |
| Vì sao Redis không bị vậy? | `DECR` atomic đơn lệnh trên server single-thread — **vốn đã tuần tự** |
| Vậy nên hỏi gì trước khi vẽ? | *"Một triệu user đó đang làm gì, với ai, và điều gì tuyệt đối không được phép sai?"* |

---

# Tham khảo

- [Google SRE Book — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — vì sao retry khuếch đại tải đúng lúc tệ nhất
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) — khung tính tài nguyên từ workload
- [MySQL — InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html) — cơ chế row lock đằng sau hot row
- [The C10K / C10M problem](http://www.kegel.com/c10k.html) — bài kinh điển về chi phí connection
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc của series
- Series System Design trên blog: [P01 — Redis không chỉ có GET/SET](/posts/system-design-p01-redis/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/) · [P04 — Streams vs Kafka](/posts/system-design-p04-streams-vs-kafka/)
