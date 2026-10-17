+++
date = '2026-10-17T10:30:00+07:00'
draft = false
slug = 'system-design-p03-pubsub-vs-streams'
title = 'System Design P03: Redis Đã Có Pub/Sub, Sao Còn Cần Streams?'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "redis", "event-driven", "kafka", "golang"]
description = 'Pub/Sub mất tin khi worker offline, Streams thì không. Đo thật sự khác biệt, ngữ nghĩa fan-out, PEL, idempotency — và khi nào thì nên dùng Kafka.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, ví dụ, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên Redis 7.4.2 local.*

# Tổng quan

Khi đi họp và nghe các bên nói về Event Driven, hầu hết đều nói về **Pub/Sub** và **Queue**. Cứ đụng tải thì lôi Queue ra, đụng fan-out thì lôi Pub/Sub ra.

Rồi **Streams** đâu? Nếu Pub/Sub đã đủ, vậy Streams sinh ra để làm gì? Đó là còn chưa nói tới **Kafka** — một hệ thống "chuyên trị" event streaming ở quy mô lớn.

Bài này làm rõ Pub/Sub và Streams khác nhau ở đâu, qua một tình huống rất nhỏ: **worker ngắt kết nối đúng lúc có sự kiện cần xử lý.**

---

# Tình huống: xuất file báo cáo

App có tính năng xuất file báo cáo. Service xuất file làm xong sẽ phát một tin *"file đã sẵn sàng"*. Worker phụ trách thông báo nhận tin đó rồi tạo thông báo để người dùng vào tải file.

Cách giao tiếp này thường gọi là **Fire & Forget**: bên phát gửi tin rồi tiếp tục làm việc, **không chờ bên nhận trả kết quả xử lý**. Vì vậy, "gửi tin xong" chưa có nghĩa là "thông báo đã được tạo".

```go
// Service xuất file — Pub/Sub
func (s *ExportService) OnFileReady(ctx context.Context, fileID string) error {
    // ... ghi file vào storage ...
    return rdb.Publish(ctx, "report.ready", fileID).Err()
    // PUBLISH trả về SỐ SUBSCRIBER đang nghe.
    // Nếu là 0 — tin vừa rồi bốc hơi, và hàm này vẫn trả về nil.
}
```

Dòng comment cuối chính là mấu chốt. `PUBLISH` trả về số subscriber nhận được tin. Nếu không ai đang nghe, nó trả về `0` — **và đó không phải lỗi**. Code trên coi như đã xong việc.

Giả sử worker bị crash hoặc shutdown **ngay trước lúc** service xuất file phát tin. Với Pub/Sub, Redis chỉ gửi cho những bên **đang nghe**, không lưu tin để chờ worker chạy lại. Vài giây sau worker lên lại, nó vẫn nhận được các tin mới bình thường — **nhưng tin "file đã sẵn sàng" vừa rồi không được gửi bù**. File đã làm xong, còn người dùng thì chờ mãi không thấy thông báo.

## Đo thật

Kịch bản: phát 300 tin, worker chết ở tin thứ 100 và lên lại ở tin thứ 200. Chạy y hệt nhau trên cả hai cơ chế:

| | Tin đã phát | Worker nhận được | **Mất** |
|---|---:|---:|---:|
| **Pub/Sub** | 300 | 199 | **101 tin (34%)** |
| **Streams** | 300 | **300** | **0** |

101 tin biến mất, không có lỗi nào được ghi lại ở bất kỳ đâu. Publisher báo thành công, Redis khoẻ, worker khoẻ — và 101 người dùng không nhận được thông báo.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 520" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ps-arrR" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="ps-arrT" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="ps-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="520" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng một kịch bản: worker chết 3 giây rồi lên lại</text>
<rect x="20" y="44" width="860" height="216" rx="14" fill="#FFF7F7" stroke="#FFC9CC" stroke-width="1.5"/>
<rect x="20" y="276" width="860" height="228" rx="14" fill="#F2FCFB" stroke="#9BE5DC" stroke-width="1.5"/>
<text x="42" y="68" font-size="14" font-weight="800" fill="#9F1239">❌ PUB/SUB — Redis không giữ gì cả</text>
<text x="42" y="300" font-size="14" font-weight="800" fill="#134E4A">✅ STREAMS — tin nằm lại trong log</text>
<rect x="44" y="84" width="120" height="48" rx="9" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2" filter="url(#ps-sh)"/>
<text x="104" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Export Service</text>
<text x="104" y="121" text-anchor="middle" font-size="10" fill="#BE123C">PUBLISH</text>
<rect x="340" y="84" width="150" height="48" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#ps-sh)"/>
<text x="415" y="104" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Redis Pub/Sub</text>
<text x="415" y="121" text-anchor="middle" font-size="10" fill="#BE123C">chuyển tiếp, KHÔNG lưu</text>
<rect x="680" y="84" width="150" height="48" rx="9" fill="#F5F5F4" stroke="#A8A29E" stroke-width="2" stroke-dasharray="5 4"/>
<text x="755" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#78716C">Notifier Worker</text>
<text x="755" y="121" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">🔴 ĐANG CHẾT</text>
<line x1="166" y1="108" x2="336" y2="108" stroke="#DC2626" stroke-width="2" marker-end="url(#ps-arrR)"/>
<text x="251" y="102" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">"report_8842 ready"</text>
<line x1="492" y1="108" x2="600" y2="108" stroke="#DC2626" stroke-width="2" stroke-dasharray="5 4"/>
<text x="628" y="113" font-size="18" font-weight="800" fill="#DC2626">✗</text>
<text x="546" y="102" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">0 subscriber</text>
<rect x="44" y="150" width="786" height="44" rx="9" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="62" y="170" font-size="11" font-weight="800" fill="#9F1239">PUBLISH trả về 0 — và 0 KHÔNG PHẢI LÀ LỖI.</text>
<text x="62" y="188" font-size="11" fill="#BE123C">Export service coi như đã xong việc. Không retry, không log, không alert. Tin bốc hơi hoàn toàn.</text>
<rect x="44" y="206" width="786" height="40" rx="9" fill="#FFD9DE" stroke="#DC2626" stroke-width="2"/>
<text x="437" y="231" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">Worker lên lại sau 3 giây → nhận tin MỚI bình thường, nhưng tin cũ KHÔNG được gửi bù</text>
<rect x="44" y="318" width="120" height="48" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#ps-sh)"/>
<text x="104" y="338" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Export Service</text>
<text x="104" y="355" text-anchor="middle" font-size="10" fill="#0F766E">XADD</text>
<rect x="300" y="310" width="230" height="110" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2.5" filter="url(#ps-sh)"/>
<rect x="300" y="310" width="230" height="26" rx="9" fill="#0D9488"/>
<rect x="300" y="326" width="230" height="10" fill="#0D9488"/>
<text x="415" y="328" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">Redis Stream (log có thứ tự)</text>
<rect x="314" y="344" width="202" height="18" rx="4" fill="#E6FBF9"/>
<rect x="314" y="366" width="202" height="18" rx="4" fill="#CCFBF1"/>
<rect x="314" y="388" width="202" height="18" rx="4" fill="#CCFBF1"/>
<text x="415" y="357" text-anchor="middle" font-size="9" font-family="monospace" fill="#0F766E">1738…-0  report_8840  ✓ đã ACK</text>
<text x="415" y="379" text-anchor="middle" font-size="9" font-family="monospace" font-weight="700" fill="#134E4A">1738…-1  report_8841  ⏳ chờ</text>
<text x="415" y="401" text-anchor="middle" font-size="9" font-family="monospace" font-weight="700" fill="#134E4A">1738…-2  report_8842  ⏳ chờ</text>
<rect x="680" y="318" width="150" height="48" rx="9" fill="#F5F5F4" stroke="#A8A29E" stroke-width="2" stroke-dasharray="5 4"/>
<text x="755" y="338" text-anchor="middle" font-size="11" font-weight="700" fill="#78716C">Notifier Worker</text>
<text x="755" y="355" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">🔴 ĐANG CHẾT</text>
<line x1="166" y1="342" x2="296" y2="342" stroke="#0D9488" stroke-width="2" marker-end="url(#ps-arrT)"/>
<text x="231" y="336" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">ghi vào log</text>
<line x1="676" y1="390" x2="534" y2="390" stroke="#0D9488" stroke-width="2.5" marker-end="url(#ps-arrT)"/>
<text x="605" y="384" text-anchor="middle" font-size="10" font-weight="700" fill="#0F766E">XREADGROUP ">"</text>
<rect x="596" y="398" width="234" height="22" rx="5" fill="#CCFBF1"/>
<text x="713" y="414" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">lên lại → đọc tiếp từ chỗ bỏ dở</text>
<rect x="44" y="436" width="786" height="56" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="62" y="458" font-size="12" font-weight="800" fill="#134E4A">Mỗi consumer group giữ một con trỏ riêng: "nhóm này đã đọc tới đâu".</text>
<text x="62" y="478" font-size="11" fill="#0F766E">Worker nghỉ bao lâu không quan trọng — miễn là tin chưa bị cắt khỏi stream, nó vẫn đọc tiếp được từ đúng vị trí cũ.</text>
</svg>
</div>

---

# Khác biệt gốc rễ

Nói ngắn gọn: **Pub/Sub là một đường ống, Streams là một cuốn sổ.**

| | **Pub/Sub** | **Streams** |
|---|---|---|
| Mô hình | Phát sóng (broadcast) | **Log có thứ tự, ghi thêm** |
| Lưu trữ | **Không lưu gì cả** | Lưu tới khi bị cắt bớt (trim) |
| Bên nhận offline | **Mất tin vĩnh viễn** | Đọc bù khi quay lại |
| Theo dõi tiến độ | Không có | Consumer group giữ con trỏ riêng |
| Xác nhận xử lý (ACK) | Không có | `XACK` + danh sách đang chờ (PEL) |
| Đọc lại lịch sử | Không | Có (trong phạm vi còn giữ) |
| Sống sót khi Redis restart | **Không** | Có (qua RDB/AOF) |
| Ngữ nghĩa giao hàng | At-most-once | **At-least-once** |
| Chi phí bộ nhớ | ~0 | Tỉ lệ với số tin còn giữ |

Hai dòng cuối đáng chú ý nhất:

**At-most-once vs at-least-once.** Pub/Sub giao nhiều nhất một lần — có thể không lần nào. Streams giao ít nhất một lần — nghĩa là **có thể hơn một lần**, và đó là vấn đề ta phải xử lý ở phần sau.

**Sống sót khi Redis restart.** Tin trong Pub/Sub không bao giờ được ghi xuống đĩa. Restart Redis giữa lúc phát tin là mất tin, kể cả khi subscriber đang online.

> 🔔 **Hệ quả ít ai để ý**: [Keyspace notifications](https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/) của Redis (`__keyevent@0__:expired`, ...) chạy **trên Pub/Sub**. Nghĩa là chúng mang đúng ngữ nghĩa mất-tin ở trên. Nếu bạn đang dựa vào sự kiện key hết hạn để kích hoạt một việc gì đó quan trọng — ví dụ huỷ đơn hàng chưa thanh toán — thì mỗi lần worker restart là một lần có đơn không bao giờ bị huỷ.

---

# Ngữ nghĩa fan-out: phần hay bị hiểu nhầm nhất

Nhiều người nghĩ "Pub/Sub để fan-out, Streams để làm queue". Thực ra **Streams làm được cả hai**, tuỳ cách bạn tổ chức consumer group. Đo thật với 300 tin và 3 bên nhận:

| Cấu hình | Mỗi bên nhận | Ngữ nghĩa |
|---|---|---|
| Pub/Sub, 3 subscriber | `[300, 300, 300]` | **Bản sao đầy đủ** cho mỗi bên |
| Streams, **1 group** 3 consumer | `[95, 105, 100]` → tổng 300 | **Chia tải** — mỗi tin đúng 1 consumer |
| Streams, **3 group riêng** | `[300, 300, 300]` | **Fan-out thật** — mỗi group nhận đủ |

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="fo-a1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="fo-a2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#D97706"/></marker>
<marker id="fo-a3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="fo-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Ba cách chia tin cho 3 bên nhận — đo thật với 300 tin</text>
<rect x="24" y="46" width="278" height="334" rx="12" fill="#FFF7F7" stroke="#FF6B6B" stroke-width="2" filter="url(#fo-sh)"/>
<rect x="311" y="46" width="278" height="334" rx="12" fill="#FFFAF0" stroke="#F5A623" stroke-width="2" filter="url(#fo-sh)"/>
<rect x="598" y="46" width="278" height="334" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#fo-sh)"/>
<text x="163" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">Pub/Sub · 3 subscriber</text>
<text x="450" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">Streams · 1 group, 3 consumer</text>
<text x="737" y="70" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">Streams · 3 group riêng</text>
<rect x="98" y="84" width="130" height="34" rx="8" fill="#DC2626"/>
<rect x="385" y="84" width="130" height="34" rx="8" fill="#D97706"/>
<rect x="672" y="84" width="130" height="34" rx="8" fill="#0D9488"/>
<text x="163" y="106" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">300 tin</text>
<text x="450" y="106" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">300 tin</text>
<text x="737" y="106" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">300 tin</text>
<rect x="44" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2"/>
<rect x="124" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2"/>
<rect x="204" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2"/>
<rect x="331" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="411" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="491" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="618" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="698" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="778" y="186" width="80" height="48" rx="8" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<line x1="163" y1="120" x2="90" y2="182" stroke="#DC2626" stroke-width="1.8" marker-end="url(#fo-a1)"/>
<line x1="163" y1="120" x2="163" y2="182" stroke="#DC2626" stroke-width="1.8" marker-end="url(#fo-a1)"/>
<line x1="163" y1="120" x2="238" y2="182" stroke="#DC2626" stroke-width="1.8" marker-end="url(#fo-a1)"/>
<line x1="450" y1="120" x2="375" y2="182" stroke="#D97706" stroke-width="1.8" marker-end="url(#fo-a2)"/>
<line x1="450" y1="120" x2="450" y2="182" stroke="#D97706" stroke-width="1.8" marker-end="url(#fo-a2)"/>
<line x1="450" y1="120" x2="527" y2="182" stroke="#D97706" stroke-width="1.8" marker-end="url(#fo-a2)"/>
<line x1="737" y1="120" x2="662" y2="182" stroke="#0D9488" stroke-width="1.8" marker-end="url(#fo-a3)"/>
<line x1="737" y1="120" x2="737" y2="182" stroke="#0D9488" stroke-width="1.8" marker-end="url(#fo-a3)"/>
<line x1="737" y1="120" x2="814" y2="182" stroke="#0D9488" stroke-width="1.8" marker-end="url(#fo-a3)"/>
<text x="163" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">mỗi bên 1 BẢN SAO</text>
<text x="450" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#7C4A03">CHIA ĐỀU, không trùng</text>
<text x="737" y="146" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">mỗi group 1 BẢN SAO</text>
<text x="84" y="206" text-anchor="middle" font-size="10" fill="#57534E">sub-1</text>
<text x="164" y="206" text-anchor="middle" font-size="10" fill="#57534E">sub-2</text>
<text x="244" y="206" text-anchor="middle" font-size="10" fill="#57534E">sub-3</text>
<text x="371" y="206" text-anchor="middle" font-size="10" fill="#57534E">w-1</text>
<text x="451" y="206" text-anchor="middle" font-size="10" fill="#57534E">w-2</text>
<text x="531" y="206" text-anchor="middle" font-size="10" fill="#57534E">w-3</text>
<text x="658" y="202" text-anchor="middle" font-size="9" fill="#57534E">notify</text>
<text x="738" y="202" text-anchor="middle" font-size="9" fill="#57534E">analytics</text>
<text x="818" y="202" text-anchor="middle" font-size="9" fill="#57534E">audit</text>
<text x="84" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">300</text>
<text x="164" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">300</text>
<text x="244" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">300</text>
<text x="371" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">95</text>
<text x="451" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">105</text>
<text x="531" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">100</text>
<text x="658" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">300</text>
<text x="738" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">300</text>
<text x="818" y="226" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">300</text>
<rect x="44" y="252" width="240" height="26" rx="6" fill="#FFE9EC"/>
<rect x="331" y="252" width="240" height="26" rx="6" fill="#FDE9C8"/>
<rect x="618" y="252" width="240" height="26" rx="6" fill="#CCFBF1"/>
<text x="164" y="269" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">tổng xử lý: 900</text>
<text x="451" y="269" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">tổng xử lý: 300</text>
<text x="738" y="269" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">tổng xử lý: 900</text>
<text x="44" y="300" font-size="11" font-weight="800" fill="#374151">Dùng khi</text>
<text x="331" y="300" font-size="11" font-weight="800" fill="#374151">Dùng khi</text>
<text x="618" y="300" font-size="11" font-weight="800" fill="#374151">Dùng khi</text>
<text x="44" y="320" font-size="11" fill="#57534E">Tín hiệu tức thời, mọi</text>
<text x="44" y="337" font-size="11" fill="#57534E">instance đều cần biết:</text>
<text x="44" y="354" font-size="11" fill="#57534E">invalidate cache cục bộ,</text>
<text x="44" y="371" font-size="11" fill="#57534E">"đang gõ", presence.</text>
<text x="331" y="320" font-size="11" fill="#57534E">Chia việc cho nhiều</text>
<text x="331" y="337" font-size="11" fill="#57534E">worker giống nhau:</text>
<text x="331" y="354" font-size="11" fill="#57534E">gửi mail, xử lý ảnh.</text>
<text x="331" y="371" font-size="11" font-weight="700" fill="#7C4A03">Thêm worker = nhanh hơn.</text>
<text x="618" y="320" font-size="11" fill="#57534E">Nhiều hệ thống KHÁC</text>
<text x="618" y="337" font-size="11" fill="#57534E">nhau cùng quan tâm 1</text>
<text x="618" y="354" font-size="11" fill="#57534E">sự kiện, mỗi bên làm</text>
<text x="618" y="371" font-size="11" font-weight="700" fill="#134E4A">việc riêng. Và KHÔNG mất tin.</text>
</svg>
</div>

Cột cuối là thứ Pub/Sub không làm được: **fan-out mà vẫn bền**. Mỗi bounded context có consumer group riêng, đọc cùng một stream, không ảnh hưởng nhau, và ai nghỉ thì tự đọc bù khi quay lại.

---

# "Giữ được tin vẫn chưa xong"

Đây là phần mà bài gốc nói rất đúng và tôi muốn đào sâu: **lưu được tin mới giải quyết một nửa bài toán.**

## Vấn đề 1 — Worker nhận tin rồi chết TRƯỚC khi xử lý xong

`XREADGROUP` giao tin cho consumer và đưa tin đó vào **PEL** (Pending Entries List) — danh sách "đã giao, chưa xác nhận". Tin chỉ rời PEL khi có `XACK`. Nếu worker chết giữa chừng, **Redis vẫn giữ tin trong PEL vô thời hạn** — nó không tự giao lại cho ai.

Đo thật:

```
worker-A đọc 10 job, ACK 4 rồi crash
→ XPENDING: 6 message còn treo trong PEL  (Redis VẪN GIỮ, không mất)
→ worker-B dùng XAUTOCLAIM nhận lại 6 message, xử lý xong
→ XPENDING sau khi phục hồi: 0
```

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="pel-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="pel-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<marker id="pel-r" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="pel-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="440" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Vòng đời một message trong consumer group</text>
<rect x="30" y="56" width="150" height="62" rx="10" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#pel-sh)"/>
<rect x="268" y="56" width="180" height="62" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2.5" filter="url(#pel-sh)"/>
<rect x="536" y="56" width="150" height="62" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#pel-sh)"/>
<text x="105" y="80" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">① Trong stream</text>
<text x="105" y="98" text-anchor="middle" font-size="10" fill="#0F766E">chưa ai đọc</text>
<text x="105" y="112" text-anchor="middle" font-size="10" font-family="monospace" fill="#0F766E">XADD</text>
<text x="358" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">② Trong PEL</text>
<text x="358" y="94" text-anchor="middle" font-size="10" fill="#92400E">đã giao, CHƯA xác nhận</text>
<text x="358" y="110" text-anchor="middle" font-size="10" font-family="monospace" fill="#92400E">XREADGROUP ... &gt;</text>
<text x="611" y="80" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">③ Hoàn tất</text>
<text x="611" y="98" text-anchor="middle" font-size="10" fill="#0F766E">rời khỏi PEL</text>
<text x="611" y="112" text-anchor="middle" font-size="10" font-family="monospace" fill="#0F766E">XACK</text>
<line x1="182" y1="87" x2="264" y2="87" stroke="#57534E" stroke-width="2" marker-end="url(#pel-a)"/>
<line x1="450" y1="87" x2="532" y2="87" stroke="#0D9488" stroke-width="2.5" marker-end="url(#pel-g)"/>
<path d="M358,120 L358,150 L240,150 L240,190" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#pel-r)"/>
<text x="368" y="140" font-size="11" font-weight="700" fill="#DC2626">worker chết trước khi ACK</text>
<rect x="100" y="192" width="280" height="86" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#pel-sh)"/>
<text x="240" y="216" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Kẹt trong PEL — VÔ THỜI HẠN</text>
<text x="240" y="236" text-anchor="middle" font-size="11" fill="#BE123C">Redis KHÔNG tự giao lại cho ai khác.</text>
<text x="240" y="254" text-anchor="middle" font-size="11" fill="#BE123C">Tin không mất, nhưng cũng không ai xử lý.</text>
<text x="240" y="271" text-anchor="middle" font-size="11" font-weight="700" fill="#DC2626">App PHẢI tự đi đòi lại.</text>
<rect x="470" y="192" width="380" height="86" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#pel-sh)"/>
<text x="660" y="214" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">Phục hồi: XAUTOCLAIM (Redis 6.2+)</text>
<text x="490" y="236" font-size="10" font-family="monospace" fill="#0F766E">XAUTOCLAIM key group worker-B 30000 0-0</text>
<text x="490" y="254" font-size="10" fill="#0F766E">→ chuyển quyền sở hữu mọi tin idle &gt; 30 giây</text>
<text x="490" y="271" font-size="10" font-weight="700" fill="#134E4A">→ delivery_count tăng lên mỗi lần bị claim lại</text>
<line x1="384" y1="235" x2="466" y2="235" stroke="#0D9488" stroke-width="2.5" marker-end="url(#pel-g)"/>
<path d="M660,280 L660,300 L611,300 L611,124" stroke="#0D9488" stroke-width="2" stroke-dasharray="5 4" fill="none" marker-end="url(#pel-g)"/>
<text x="700" y="296" font-size="10" font-weight="700" fill="#0F766E">xử lý lại → ACK</text>
<rect x="30" y="320" width="820" height="104" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="50" y="342" font-size="12" font-weight="800" fill="#7C4A03">Ba điều phải tự làm trong app — Redis không làm hộ</text>
<text x="50" y="364" font-size="11" fill="#92400E">1. <tspan font-weight="700">Chạy XAUTOCLAIM định kỳ</tspan> (ví dụ mỗi 30s) — không có nó, tin treo trong PEL mãi mãi.</text>
<text x="50" y="384" font-size="11" fill="#92400E">2. <tspan font-weight="700">Đọc lại PEL của chính mình lúc khởi động</tspan>: XREADGROUP với ID "0" thay vì "&gt;" — lấy phần dang dở của lần chạy trước.</text>
<text x="50" y="404" font-size="11" fill="#92400E">3. <tspan font-weight="700">Dead letter khi delivery_count vượt ngưỡng</tspan> — một message "độc" sẽ bị claim lại vô hạn nếu không có đường thoát.</text>
</svg>
</div>

Code phục hồi, gồm cả ba việc trên:

```go
// Lúc KHỞI ĐỘNG: đọc lại phần dang dở của CHÍNH consumer này.
// ID "0" nghĩa là "pending của tôi", khác hẳn ">" nghĩa là "tin mới chưa ai nhận".
func (w *Worker) recoverOwnPending(ctx context.Context) error {
    res, err := w.rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
        Group: w.group, Consumer: w.name,
        Streams: []string{w.stream, "0"},   // <— "0", KHÔNG phải ">"
        Count:   100,
    }).Result()
    if err != nil {
        return err
    }
    for _, s := range res {
        for _, m := range s.Messages {
            w.handle(ctx, m)
        }
    }
    return nil
}

// Chạy ĐỊNH KỲ: đòi lại tin của những consumer đã chết hẳn.
func (w *Worker) reclaimStale(ctx context.Context) error {
    start := "0-0"
    for {
        msgs, next, err := w.rdb.XAutoClaim(ctx, &redis.XAutoClaimArgs{
            Stream: w.stream, Group: w.group, Consumer: w.name,
            MinIdle: 30 * time.Second,   // phải DÀI HƠN thời gian xử lý bình thường
            Start:   start, Count: 100,
        }).Result()
        if err != nil {
            return err
        }
        for _, m := range msgs {
            w.handle(ctx, m)
        }
        if next == "0-0" || len(msgs) == 0 {
            return nil
        }
        start = next
    }
}
```

> ⚠️ **`MinIdle` phải dài hơn thời gian xử lý bình thường.** Nếu job mất 60 giây mà `MinIdle` là 30 giây, worker thứ hai sẽ đòi mất tin của worker thứ nhất **trong khi nó vẫn đang làm** — và bạn có hai worker cùng xử lý một tin. Đây là lỗi hay gặp nhất khi mới dùng `XAUTOCLAIM`.

Và dead letter, dựa vào `delivery_count`:

```go
const maxDeliveries = 5

pending, _ := rdb.XPendingExt(ctx, &redis.XPendingExtArgs{
    Stream: stream, Group: group, Start: "-", End: "+", Count: 100,
}).Result()

for _, p := range pending {
    if p.RetryCount >= maxDeliveries {
        // Tin này đã thử 5 lần vẫn hỏng → đẩy sang stream riêng để người xem,
        // rồi ACK để nó thôi quay vòng vô hạn.
        rdb.XAdd(ctx, &redis.XAddArgs{Stream: stream + ":dlq",
            Values: map[string]interface{}{"orig_id": p.ID, "retries": p.RetryCount}})
        rdb.XAck(ctx, stream, group, p.ID)
    }
}
```

Không có bước này, một message gây panic sẽ bị claim → fail → claim → fail mãi mãi, và nó chặn luôn việc xử lý những tin phía sau nếu worker của bạn xử lý tuần tự.

## Vấn đề 2 — Xử lý xong rồi chết TRƯỚC khi ACK

Trường hợp ngược lại: worker **đã tạo thông báo**, nhưng chưa kịp `XACK` thì crash. Khi `XAUTOCLAIM` giao lại, nó tạo thông báo **lần thứ hai**. Người dùng nhận hai thông báo cho cùng một file.

Đây là hệ quả tất yếu của **at-least-once**, và không có cách nào tránh được ở tầng message broker — kể cả Kafka. Chỗ duy nhất sửa được là **điểm ghi kết quả**.

Đo thật, mô phỏng message bị giao lại 3 lần:

| | Số thông báo được tạo |
|---|---:|
| Không bảo vệ | **3** ❌ user nhận 3 lần |
| Có khoá idempotency (`SET NX`) | **1** ✅ |

```go
func (w *Worker) handle(ctx context.Context, m redis.XMessage) {
    fileID := m.Values["file_id"].(string)

    // Khoá idempotency: dùng khoá NGHIỆP VỤ, không dùng message ID.
    // Vì nếu publisher retry, cùng một sự kiện sẽ có HAI message ID khác nhau.
    key := "processed:notification:" + fileID
    ok, err := w.rdb.SetNX(ctx, key, "1", 24*time.Hour).Result()
    if err != nil {
        return // lỗi Redis → KHÔNG ack, để được giao lại
    }
    if !ok {
        w.rdb.XAck(ctx, w.stream, w.group, m.ID) // đã làm rồi → chỉ cần ack
        return
    }

    if err := w.createNotification(ctx, fileID); err != nil {
        w.rdb.Del(ctx, key) // thất bại → nhả khoá để lần sau làm lại được
        return
    }
    w.rdb.XAck(ctx, w.stream, w.group, m.ID)
}
```

Hai chi tiết quan trọng trong đoạn này:

- **Khoá theo `file_id`, không theo `m.ID`.** Nếu publisher retry (bản thân nó cũng có thể crash giữa chừng), cùng một sự kiện nghiệp vụ sẽ được `XADD` hai lần với hai message ID khác nhau. Khoá theo message ID không chặn được trường hợp đó.
- **Nhả khoá khi xử lý thất bại.** Quên bước này thì một lần lỗi tạm thời sẽ khiến file đó **vĩnh viễn** không bao giờ có thông báo — khoá đã chiếm nhưng việc chưa làm.

Cách bền hơn nữa, nếu bên ghi là database: dùng **unique constraint**.

```sql
CREATE TABLE notifications (
  id      BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  file_id VARCHAR(64) NOT NULL,
  UNIQUE KEY uk_notification_file (user_id, file_id)   -- DB tự từ chối bản trùng
);
```

Database đảm bảo tính duy nhất bằng chính ràng buộc dữ liệu, không phụ thuộc vào việc Redis có còn khoá hay không. Lỗi duplicate key lúc đó không phải lỗi — nó là tín hiệu "đã làm rồi", cứ `XACK` và đi tiếp.

> 📌 Dự án [Honeydue](https://github.com/huydq96) của mình đặt quy ước này ngay trong `CLAUDE.md`: *"Consumers must always be idempotent — reprocessing the same message must never cause a duplicate side effect (upsert / check-before-act, not blind insert)."* Đây không phải lời khuyên chung chung, nó là hệ quả trực tiếp của ngữ nghĩa at-least-once.

---

# Streams không phải kho lịch sử vô hạn

Stream nằm trong RAM. Không cắt bớt thì nó lớn mãi tới khi Redis hết bộ nhớ. Đo thật với 50.000 entry:

| Cách ghi | Số entry còn lại | Bộ nhớ |
|---|---:|---:|
| `XADD` không giới hạn | 50.000 | **2.862.616 bytes (2,7 MB)** |
| `XADD MAXLEN ~ 1000` | 1.074 | **66.952 bytes (0,1 MB)** |

```go
rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "report.ready",
    MaxLen: 10000,
    Approx: true,          // dấu ~ : cắt GẦN ĐÚNG, nhanh hơn nhiều
    Values: map[string]interface{}{"file_id": fileID},
})
```

Để ý con số **1.074** chứ không phải đúng 1.000 — đó chính là ý nghĩa của dấu `~`. Stream lưu các entry theo **macro-node**; cắt gần đúng chỉ xoá nguyên cả node, nên số còn lại luôn **≥** ngưỡng bạn đặt. Đổi lại nó chạy `O(1)` thay vì `O(N)`. Trong hot path, luôn dùng `~`.

Cắt theo **thời gian** thường hợp lý hơn cắt theo số lượng, vì nó map thẳng sang yêu cầu nghiệp vụ "worker được phép nghỉ bao lâu":

```go
// Giữ 24 giờ: worker nghỉ dưới 24h vẫn đọc bù được đầy đủ
cutoff := time.Now().Add(-24 * time.Hour).UnixMilli()
rdb.XTrimMinIDApprox(ctx, "report.ready", strconv.FormatInt(cutoff, 10), 0)
```

Và đây là ràng buộc cần nói rõ: **nếu dọn tin quá sớm, worker nghỉ lâu vẫn mất phần cần xử lý bù.** Chính sách retention không phải quyết định kỹ thuật, nó là câu trả lời cho câu hỏi *"worker của chúng ta được phép chết bao lâu?"*

**Về độ bền khi Redis sập**: Streams được ghi xuống RDB/AOF, nhưng mức bảo đảm phụ thuộc cấu hình. Với `appendfsync everysec` (mặc định), bạn có thể mất tới 1 giây dữ liệu cuối. Nếu không chấp nhận được, đó là tín hiệu nên nhìn sang Kafka.

---

# Cái giá: throughput

Streams không miễn phí. Đo ghi 20.000 tin:

| | Thời gian | Throughput |
|---|---:|---:|
| `PUBLISH` (không lưu gì) | 20 ms | **~1.010.000 ops/s** |
| `XADD` (có lưu trữ) | 65 ms | ~306.000 ops/s |

`XADD` chậm hơn **3,3 lần**. Nhưng hãy đặt con số này đúng bối cảnh: **306.000 sự kiện/giây trên một node** vẫn thừa sức cho gần như mọi ứng dụng. Nếu bạn thật sự cần hơn thế, vấn đề của bạn không còn là chọn giữa Pub/Sub và Streams nữa.

---

# Chọn cái nào?

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 500" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="dp-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="dp-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="500" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Một câu hỏi quyết định tất cả</text>
<rect x="190" y="50" width="480" height="56" rx="10" fill="#374151" filter="url(#dp-sh)"/>
<text x="430" y="74" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">Nếu tin này mất, có ai phải đi sửa bằng tay không?</text>
<text x="430" y="94" text-anchor="middle" font-size="11" fill="#D6D3D1">(hoặc: người dùng có bị thiệt hại gì không?)</text>
<rect x="30" y="138" width="370" height="104" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#dp-sh)"/>
<text x="215" y="164" text-anchor="middle" font-size="15" font-weight="800" fill="#9F1239">PUB/SUB</text>
<text x="215" y="186" text-anchor="middle" font-size="11" fill="#BE123C">Tin chỉ có ý nghĩa NGAY LÚC nó xảy ra.</text>
<text x="215" y="206" text-anchor="middle" font-size="11" fill="#BE123C">Nhận muộn 5 phút thì cũng vô dụng.</text>
<text x="215" y="228" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Đơn giản nhất, nhanh nhất, ~1M ops/s</text>
<rect x="460" y="138" width="370" height="104" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#dp-sh)"/>
<text x="645" y="164" text-anchor="middle" font-size="15" font-weight="800" fill="#134E4A">STREAMS</text>
<text x="645" y="186" text-anchor="middle" font-size="11" fill="#0F766E">Việc cần làm không nên mất theo</text>
<text x="645" y="206" text-anchor="middle" font-size="11" fill="#0F766E">khi bên nhận tạm thời không chạy.</text>
<text x="645" y="228" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Thêm: PEL, retention, idempotency</text>
<line x1="330" y1="108" x2="240" y2="134" stroke="#57534E" stroke-width="2" marker-end="url(#dp-arr)"/>
<line x1="530" y1="108" x2="620" y2="134" stroke="#57534E" stroke-width="2" marker-end="url(#dp-arr)"/>
<rect x="240" y="104" width="70" height="20" rx="5" fill="#DC2626"/>
<rect x="550" y="104" width="52" height="20" rx="5" fill="#0D9488"/>
<text x="275" y="119" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="576" y="119" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<rect x="30" y="256" width="370" height="130" rx="10" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<text x="50" y="278" font-size="12" font-weight="800" fill="#9F1239">Ví dụ hợp</text>
<text x="50" y="300" font-size="11" fill="#57534E">• Tín hiệu "đang gõ" trong chat</text>
<text x="50" y="318" font-size="11" fill="#57534E">• Presence: ai đang online</text>
<text x="50" y="336" font-size="11" fill="#57534E">• Invalidate cache cục bộ ở mọi instance</text>
<text x="50" y="354" font-size="11" fill="#57534E">• Giá chứng khoán realtime (có nguồn khác)</text>
<text x="50" y="375" font-size="11" font-weight="700" fill="#0F766E">Điểm chung: mất tin thì lấy lại trạng thái</text>
<rect x="460" y="256" width="370" height="130" rx="10" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="480" y="278" font-size="12" font-weight="800" fill="#134E4A">Ví dụ hợp</text>
<text x="480" y="300" font-size="11" fill="#57534E">• "File báo cáo đã sẵn sàng"</text>
<text x="480" y="318" font-size="11" fill="#57534E">• "Đơn hàng hoàn tất" → gửi mail/SMS</text>
<text x="480" y="336" font-size="11" fill="#57534E">• Đồng bộ dữ liệu sang hệ thống khác</text>
<text x="480" y="354" font-size="11" fill="#57534E">• Các bước tiếp theo của một workflow</text>
<text x="480" y="375" font-size="11" font-weight="700" fill="#0F766E">Điểm chung: mất tin = mất việc, không lấy lại được</text>
<rect x="30" y="400" width="800" height="84" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="50" y="424" font-size="12" font-weight="800" fill="#7C4A03">Phép thử nhanh khi phân vân</text>
<text x="50" y="446" font-size="11" fill="#92400E">Hỏi: <tspan font-weight="700">"Nếu worker này offline 5 phút, khi lên lại nó CẦN biết chuyện gì đã xảy ra không?"</tspan></text>
<text x="50" y="468" font-size="11" fill="#92400E">Cần → Streams. Không cần, vì nó sẽ tự lấy lại trạng thái hiện tại từ DB → Pub/Sub.</text>
</svg>
</div>

Tách hai nhu cầu cho rõ:

- **Pub/Sub**: báo ngay cho các bên đang nghe; tin bị lỡ có thể bỏ qua, hoặc ứng dụng lấy lại trạng thái từ một nguồn khác.
- **Streams**: giữ tin để đọc và xử lý tiếp, kể cả khi bên nhận tạm thời không chạy. Chấp nhận thêm phần quản lý lịch sử và tiến độ xử lý.

Với tín hiệu **"đang gõ"** trong chat: nếu bạn offline lúc người kia gõ, vài phút sau mới nhận lại tín hiệu đó cũng chẳng để làm gì. Với tin **"xuất file xong"**: worker phải xử lý được kể cả khi nó vừa nghỉ vài giây.

---

# Còn Kafka thì sao?

Câu hỏi tiếp theo rất tự nhiên: nếu cần độ bền thì dùng luôn Kafka cho xong?

| | **Redis Pub/Sub** | **Redis Streams** | **Kafka** |
|---|---|---|---|
| Lưu trữ | Không | RAM (+ RDB/AOF) | **Đĩa, replicated** |
| Retention điển hình | 0 | Giờ → ngày (giới hạn bởi RAM) | **Ngày → vĩnh viễn** |
| Throughput 1 node | ~1.000.000/s | ~300.000/s | ~100.000–1.000.000/s |
| Mở rộng ngang | Không | Hạn chế (1 stream = 1 node) | **Partition — mở rộng thật** |
| Consumer group | Không | Có | Có |
| Đọc lại từ đầu | Không | Có (trong retention) | **Có, kể cả rất xa** |
| Thứ tự | Theo channel | Theo stream | Theo partition |
| Chi phí vận hành | ~0 | **~0 nếu đã có Redis** | Cao (broker, KRaft, monitoring) |
| Mất dữ liệu khi node chết | Mất hết | Tới 1s (`appendfsync everysec`) | **0 nếu `acks=all`** |

**Dùng Redis Streams khi**: bạn **đã có Redis**, lượng sự kiện vừa phải, retention tính bằng giờ hoặc ngày, và không muốn thêm một hệ thống phân tán nữa vào stack. Chi phí thực tế gần bằng không.

**Chuyển sang Kafka khi** có ít nhất một trong các điều sau:
- Cần giữ sự kiện **nhiều ngày trở lên** (event sourcing, replay để dựng lại state, audit)
- Lượng dữ liệu vượt RAM của một node Redis
- Cần mở rộng ngang thật sự bằng partition
- Không chấp nhận mất **bất kỳ** sự kiện nào (`acks=all` + replication)

> Trong Honeydue mình dùng **Kafka** cho các domain event (`transaction.created`, `budget.limit_changed`), vì đó là sự kiện nghiệp vụ cần replay và audit. Nhưng vẫn dùng **Redis Pub/Sub** cho fan-out WebSocket giữa các instance (`user:{user_id}`) — vì nếu một tin đẩy realtime bị lỡ, client **tự lấy lại trạng thái qua REST khi reconnect**. Đúng cái phép thử ở sơ đồ trên: WS không phải nguồn sự thật, nên mất tin không sao.

Hai cơ chế, hai bài toán khác nhau, trong cùng một hệ thống. Đó mới là cách chọn đúng — không phải chọn một cái rồi dùng cho tất cả.

---

# Checklist

```
KHI DÙNG PUB/SUB
□ Tin này mất có sao không? Đã trả lời THẬT chưa, hay chỉ cho rằng không sao?
□ Client có đường lấy lại trạng thái khi reconnect không? (REST, snapshot)
□ Có đang dùng keyspace notification cho việc quan trọng không? → nó là Pub/Sub, sẽ mất tin
□ Subscriber chậm có bị Redis ngắt không? (client-output-buffer-limit pubsub: 32MB cứng / 8MB-60s mềm)
□ Chạy Redis Cluster? → cân nhắc SSUBSCRIBE (sharded pub/sub, Redis 7.0+) để tránh broadcast toàn cluster

KHI DÙNG STREAMS
□ Đã đặt MAXLEN hoặc MINID chưa? (không có = stream lớn vô hạn)
□ Dùng dấu ~ (approx) trong hot path chưa?
□ Retention đủ dài cho "worker được phép chết bao lâu" chưa?
□ Có XAUTOCLAIM chạy định kỳ không? (không có = tin kẹt PEL vĩnh viễn)
□ MinIdle có DÀI HƠN thời gian xử lý bình thường không?
□ Lúc khởi động có đọc lại PEL của chính mình (XREADGROUP ID "0") không?
□ Có dead letter khi delivery_count vượt ngưỡng không?
□ Handler đã idempotent chưa? Khoá theo ID NGHIỆP VỤ hay theo message ID?
□ Khi xử lý thất bại, khoá idempotency có được nhả ra không?
□ Có theo dõi XINFO GROUPS (lag, pending) không?
```

---

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| Khác biệt cốt lõi? | Pub/Sub là **đường ống** (không lưu), Streams là **cuốn sổ** (lưu lại) |
| Mất bao nhiêu tin thật sự? | Đo được: worker offline giữa chừng → Pub/Sub mất **34%**, Streams mất **0** |
| `PUBLISH` trả về 0 nghĩa là gì? | "Không ai nghe" — và **đó không phải lỗi**, code của bạn vẫn chạy tiếp bình thường |
| Streams chỉ làm queue thôi à? | Không. **1 group** = chia tải; **nhiều group** = fan-out bền |
| Lưu được tin là xong chưa? | **Chưa.** Còn PEL (nhận rồi chết) và idempotency (làm rồi nhưng chưa ACK) |
| Redis có tự giao lại tin treo không? | **Không.** App phải tự chạy `XAUTOCLAIM` định kỳ |
| Chống trùng ở đâu? | Ở **điểm ghi kết quả** — `SET NX` theo khoá nghiệp vụ, hoặc unique constraint trong DB |
| Streams giữ tin bao lâu? | Bạn quyết định bằng `MAXLEN`/`MINID`. Câu hỏi thật: *worker được phép chết bao lâu?* |
| Cái giá của Streams? | `XADD` chậm hơn `PUBLISH` **3,3 lần** (306k vs 1M ops/s) — vẫn thừa cho hầu hết hệ thống |
| Khi nào cần Kafka? | Retention nhiều ngày, vượt RAM một node, cần partition, hoặc không được mất tin nào |

---

# Tham khảo

- [Redis — Streams tutorial](https://redis.io/docs/latest/develop/data-types/streams/) — đọc kỹ phần consumer group và PEL
- [Redis — Pub/Sub](https://redis.io/docs/latest/develop/pubsub/) — gồm cả sharded pub/sub cho Cluster
- [Redis — `XAUTOCLAIM`](https://redis.io/docs/latest/commands/xautoclaim/) · [`XPENDING`](https://redis.io/docs/latest/commands/xpending/)
- [Redis — Keyspace notifications](https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/) — lưu ý phần "events are not reliable"
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc của series
- Bài trước: [P01 — Redis Không Chỉ Có GET/SET](/posts/system-design-p01-redis/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/)
- Trên blog: [Apache Kafka P1](/posts/apache-kafka-p1/) · [Apache Kafka P2](/posts/apache-kafka-p2/)
