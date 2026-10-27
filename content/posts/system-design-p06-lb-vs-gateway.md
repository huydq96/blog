+++
date = '2026-10-27T09:00:00+07:00'
draft = false
slug = 'system-design-p06-lb-vs-gateway'
title = 'System Design P06: Các Hiểu Sai Về API Gateway Và Load Balancer'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "load-balancer", "api-gateway", "nginx", "golang"]
description = 'Ranh giới LB và Gateway chồng lấn nhiều hơn bạn nghĩ. Đo thật trên nginx: cùng một tải, L4 lệch tới ∞ còn L7 đều 1.0×, và crash làm 30/30 request thất bại.'
+++

> *Bài viết thuộc series **System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, sơ đồ và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên nginx 1.27 (alpine) + Go 1.24 chạy local.*

# Công thức mặc định

Khi mới bước từ việc code một con app chạy được sang vẽ sơ đồ kiến trúc, rất nhiều người mặc định một công thức:

> Cứ có nhiều server là đặt **Load Balancer**. Hễ tách ra vài service là phải nhét thêm **API Gateway**.

Nhưng công thức đó vỡ ngay khi hỏi hai câu:

- Nếu **Gateway đã chia tải được** cho backend, thì còn cần LB ở phía trước để làm gì?
- Nếu **LB đã route được** request đến đúng service, Gateway sẽ xử lý thêm phần nào?

Bài này trả lời bằng số đo thật, và kết luận có thể hơi phản trực giác: **ranh giới giữa hai thứ này chồng lấn nhiều hơn phần lớn sơ đồ kiến trúc thể hiện.**

---

# Chẻ một request thành ba bước

Giả sử ứng dụng có hai cụm backend — `user-service` và `order-service` — mỗi cụm 3 instance. Một request đi vào hệ thống cần qua:

| Bước | Việc | Cần đọc tới đâu |
|---|---|---|
| **1** | Nhận kết nối từ client, đọc thông tin để biết request gọi `user-service` hay `order-service` | **HTTP** (host, path, header) |
| **2** | Áp policy chung: validate token, rate limit, kiểm tra quyền | HTTP + logic nghiệp vụ |
| **3** | Chốt được là `order-service` rồi, chọn **1 trong 3 instance đang sống** để chuyển request tới | Chỉ cần danh sách instance |

Chẻ ra như vậy mới thấy rõ chỗ hiểu sai: **nhiều người nghĩ LB chỉ làm bước 3, và chỉ ở L4.**

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ov-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Ba bước, và ai làm được bước nào</text>
<text x="450" y="48" text-anchor="middle" font-size="11" fill="#78716C">Vùng chồng lấn rộng hơn phần lớn sơ đồ kiến trúc thể hiện</text>
<rect x="150" y="66" width="200" height="34" rx="8" fill="#374151"/>
<rect x="360" y="66" width="200" height="34" rx="8" fill="#374151"/>
<rect x="570" y="66" width="200" height="34" rx="8" fill="#374151"/>
<text x="250" y="88" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">① Route tới service nào</text>
<text x="460" y="88" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">② API policy</text>
<text x="670" y="88" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">③ Chọn 1/3 instance</text>
<rect x="24" y="116" width="120" height="48" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="84" y="136" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">LB L4</text>
<text x="84" y="153" text-anchor="middle" font-size="10" fill="#BE123C">TCP/IP + port</text>
<rect x="150" y="116" width="200" height="48" rx="8" fill="#F5F5F4" stroke="#D6D3D1" stroke-width="1.5" stroke-dasharray="5 4"/>
<rect x="360" y="116" width="200" height="48" rx="8" fill="#F5F5F4" stroke="#D6D3D1" stroke-width="1.5" stroke-dasharray="5 4"/>
<rect x="570" y="116" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="250" y="145" text-anchor="middle" font-size="11" fill="#A8A29E">✗ không đọc được HTTP</text>
<text x="460" y="145" text-anchor="middle" font-size="11" fill="#A8A29E">✗</text>
<text x="670" y="145" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ LÀM ĐƯỢC</text>
<rect x="24" y="176" width="120" height="48" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="84" y="196" text-anchor="middle" font-size="12" font-weight="800" fill="#7C4A03">LB L7</text>
<text x="84" y="213" text-anchor="middle" font-size="10" fill="#92400E">nginx, ALB</text>
<rect x="150" y="176" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="360" y="176" width="200" height="48" rx="8" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<rect x="570" y="176" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="250" y="205" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ LÀM ĐƯỢC</text>
<text x="460" y="198" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">~ một phần</text>
<text x="460" y="214" text-anchor="middle" font-size="10" fill="#92400E">ALB có auth sẵn</text>
<text x="670" y="205" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ LÀM ĐƯỢC</text>
<rect x="24" y="236" width="120" height="48" rx="8" fill="#EEF2FF" stroke="#6366F1" stroke-width="2"/>
<text x="84" y="256" text-anchor="middle" font-size="12" font-weight="800" fill="#3730A3">Gateway</text>
<text x="84" y="273" text-anchor="middle" font-size="10" fill="#4338CA">Kong, Envoy</text>
<rect x="150" y="236" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="360" y="236" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<rect x="570" y="236" width="200" height="48" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="250" y="265" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ LÀM ĐƯỢC</text>
<text x="460" y="258" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ ĐIỂM MẠNH</text>
<text x="460" y="274" text-anchor="middle" font-size="10" fill="#0F766E">gom về một chỗ</text>
<text x="670" y="258" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">✓ LÀM ĐƯỢC</text>
<text x="670" y="274" text-anchor="middle" font-size="10" fill="#0F766E">Kong: Upstream + Target</text>
<rect x="24" y="306" width="852" height="74" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="330" font-size="13" font-weight="800" fill="#7C4A03">Hai kết luận rút ra từ bảng trên</text>
<text x="44" y="352" font-size="11" fill="#92400E">• Chỉ cần route theo path + chia tải backend? <tspan font-weight="700">Một LB L7 là đủ</tspan> — chưa cần Gateway riêng.</text>
<text x="44" y="372" font-size="11" fill="#92400E">• Gateway đã chia tải xuống target? <tspan font-weight="700">Chưa cần LB giữa Gateway và backend</tspan> chỉ để làm lại việc đó.</text>
<rect x="24" y="392" width="852" height="70" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="44" y="416" font-size="13" font-weight="800" fill="#9F1239">Chỗ hiểu sai phổ biến nhất</text>
<text x="44" y="438" font-size="11" fill="#BE123C">"LB chỉ làm bước 3". Thực tế <tspan font-weight="700">LB L7 làm được cả bước 1</tspan>, và một số LB cloud có sẵn authentication.</text>
<text x="44" y="456" font-size="11" fill="#BE123C">Riêng <tspan font-weight="700">LB L4 thì đúng là không đọc được HTTP</tspan> — nó không biết request gọi /users hay /orders.</text>
</svg>
</div>

## Đo thật: LB L7 làm được bước 1

Chỉ cần một khối `location` trong nginx:

```nginx
upstream user_svc  { server 127.0.0.1:9001; }
upstream order_svc { server 127.0.0.1:9002; }

server {
  listen 8083;
  location /users/  { proxy_pass http://user_svc; }
  location /orders/ { proxy_pass http://order_svc; }
  location /        { return 404 "no route\n"; }
}
```

Kết quả:

```
/users/42    → HTTP 200  be-1
/orders/99   → HTTP 200  be-2
/unknown     → HTTP 404  no route
```

Bước 1 xong, không có Gateway nào tham gia. Nếu nhu cầu của bạn dừng ở đây thì **thêm Gateway chỉ là thêm một hop và một thứ phải vận hành**.

---

# Phát hiện quan trọng nhất: L4 chia theo kết nối, L7 chia theo request

Đây là chỗ con số nói to hơn mọi lời giải thích.

Tình huống đúng như bài gốc nêu: hai client, mỗi client **một kết nối keep-alive**. Client A gửi **100 request** trên kết nối đang có, client B chỉ gửi **2**. Ba backend đang sống. Tôi chạy cùng một tải qua hai chế độ nginx:

| Chế độ nginx | be-1 | be-2 | be-3 | Độ lệch max/min |
|---|---:|---:|---:|---|
| **L7** (`http` + `proxy_pass`) | **34** | **34** | **34** | **1.0×** |
| **L4** (`stream` + `proxy_pass`) | 2 | **100** | **0** | **∞ — một instance nhận 0 request** |

Đọc kỹ dòng L4: **be-2 nhận 100 request, be-3 nhận không một request nào.** LB đã chia *hai kết nối* sang *hai backend* — nó làm đúng việc của nó. Nhưng **số request thì lệch hoàn toàn**.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="l4-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="l7-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="d2-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="440" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Cùng một tải — hai cách chia, hai kết cục</text>
<text x="450" y="46" text-anchor="middle" font-size="11" fill="#78716C">Client A: 1 kết nối · 100 request   ·   Client B: 1 kết nối · 2 request</text>
<rect x="24" y="62" width="420" height="198" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#d2-sh)"/>
<text x="234" y="86" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">L4 — chia theo KẾT NỐI</text>
<rect x="44" y="100" width="76" height="30" rx="6" fill="#57534E"/>
<rect x="44" y="140" width="76" height="30" rx="6" fill="#A8A29E"/>
<text x="82" y="120" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">A · 100 req</text>
<text x="82" y="160" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">B · 2 req</text>
<rect x="320" y="92" width="104" height="40" rx="6" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.8"/>
<rect x="320" y="140" width="104" height="40" rx="6" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<rect x="320" y="188" width="104" height="40" rx="6" fill="#F5F5F4" stroke="#A8A29E" stroke-width="1.5" stroke-dasharray="4 3"/>
<text x="372" y="108" text-anchor="middle" font-size="10" fill="#9F1239">be-1</text>
<text x="372" y="124" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">2</text>
<text x="372" y="156" text-anchor="middle" font-size="10" fill="#9F1239">be-2</text>
<text x="372" y="172" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">100</text>
<text x="372" y="204" text-anchor="middle" font-size="10" fill="#78716C">be-3</text>
<text x="372" y="220" text-anchor="middle" font-size="13" font-weight="800" fill="#A8A29E">0</text>
<path d="M124,112 C200,112 250,150 316,158" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#l4-a)"/>
<path d="M124,152 C200,150 250,120 316,110" stroke="#A8A29E" stroke-width="2" fill="none" marker-end="url(#l4-a)"/>
<text x="222" y="190" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">1 kết nối = 1 backend, mãi mãi</text>
<rect x="44" y="236" width="380" height="18" rx="5" fill="#FFD9DE"/>
<text x="234" y="249" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">lệch ∞ — be-3 không nhận gì cả</text>
<rect x="456" y="62" width="420" height="198" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#d2-sh)"/>
<text x="666" y="86" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">L7 — chia theo REQUEST</text>
<rect x="476" y="100" width="76" height="30" rx="6" fill="#57534E"/>
<rect x="476" y="140" width="76" height="30" rx="6" fill="#A8A29E"/>
<text x="514" y="120" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">A · 100 req</text>
<text x="514" y="160" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">B · 2 req</text>
<rect x="752" y="92" width="104" height="40" rx="6" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="752" y="140" width="104" height="40" rx="6" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="752" y="188" width="104" height="40" rx="6" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="804" y="108" text-anchor="middle" font-size="10" fill="#134E4A">be-1</text>
<text x="804" y="124" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">34</text>
<text x="804" y="156" text-anchor="middle" font-size="10" fill="#134E4A">be-2</text>
<text x="804" y="172" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">34</text>
<text x="804" y="204" text-anchor="middle" font-size="10" fill="#134E4A">be-3</text>
<text x="804" y="220" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">34</text>
<path d="M556,112 C620,112 690,106 748,108" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#l7-a)"/>
<path d="M556,118 C620,130 690,150 748,156" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#l7-a)"/>
<path d="M556,124 C620,160 690,196 748,204" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#l7-a)"/>
<text x="640" y="218" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">mỗi request chọn lại backend</text>
<rect x="476" y="236" width="380" height="18" rx="5" fill="#CCFBF1"/>
<text x="666" y="249" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">lệch 1.0× — đều tuyệt đối</text>
<rect x="24" y="276" width="852" height="70" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="300" font-size="13" font-weight="800" fill="#7C4A03">HTTP/2 làm chuyện này tệ hơn nữa</text>
<text x="44" y="322" font-size="11" fill="#92400E">Đo trên là HTTP/1.1 keep-alive. HTTP/2 <tspan font-weight="700">multiplexing nhiều request song song</tspan> trên cùng một kết nối,</text>
<text x="44" y="340" font-size="11" fill="#92400E">nên số request dồn vào một đích càng lớn. Một client HTTP/2 duy nhất có thể làm nghẽn đúng một instance.</text>
<rect x="24" y="358" width="852" height="68" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="44" y="382" font-size="12" font-weight="800" fill="#9F1239">Khi thấy một instance bận hơn hẳn các instance còn lại</text>
<text x="44" y="404" font-size="11" fill="#BE123C">Câu hỏi đầu tiên: <tspan font-weight="700">LB đang chia theo KẾT NỐI hay theo REQUEST?</tspan></text>
<text x="44" y="420" font-size="11" fill="#BE123C">Và kể cả khi số request đã đều, vẫn phải xem <tspan font-weight="700">từng loại request tốn bao nhiêu công xử lý</tspan>.</text>
</svg>
</div>

> 📌 Đo trên dùng **HTTP/1.1 keep-alive**, không phải HTTP/2 — tức vấn đề này tồn tại ngay cả với HTTP/1.1, không cần tới HTTP/2. HTTP/2 chỉ làm nó tệ hơn vì multiplexing cho phép **nhiều request song song** trên cùng một kết nối.

---

# Vậy sao vẫn đặt Load Balancer trước API Gateway?

Vì lúc đó bạn đang **scale cho chính bản thân Gateway**.

Chạy 4 instance Gateway để dự phòng và chia tải — traffic của người dùng được phân phối tới 4 instance đó bằng cách nào? Đó là một bài toán chia tải **hoàn toàn khác** với việc chọn 1 trong 3 instance của `order-service`.

**Hai bước chia tải, hai nhóm destination khác nhau.** Tôi dựng đúng topology đó và đo cùng một lúc:

```
Client (1 kết nối, 100 req)  ─┐
                               ├─→  LB L4  ─→  2 Gateway (L7)  ─→  3 Backend
Client (1 kết nối, 2 req)    ─┘
```

| Tầng | Nhóm destination | Kết quả đo | Độ lệch |
|---|---|---|---|
| **1 — LB (L4)** | cụm **Gateway** | `gw-1=100  gw-2=2` | **50.0×** |
| **2 — Gateway (L7)** | cụm **Backend** | `be-1=34  be-2=34  be-3=34` | **1.0×** |

Cùng một traffic, cùng một thời điểm, hai tầng cho ra hai phân bố hoàn toàn khác nhau. Tầng L4 lệch 50 lần vì chia theo kết nối; tầng L7 đều tuyệt đối vì chia theo request.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="t2-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="t2-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai bước chia tải, hai nhóm destination — đo cùng lúc</text>
<rect x="24" y="52" width="130" height="86" rx="10" fill="#FFFFFF" stroke="#57534E" stroke-width="2" filter="url(#t2-sh)"/>
<text x="89" y="76" text-anchor="middle" font-size="11" font-weight="700" fill="#374151">Client</text>
<rect x="38" y="86" width="102" height="20" rx="4" fill="#F5F5F4"/>
<rect x="38" y="110" width="102" height="20" rx="4" fill="#F5F5F4"/>
<text x="89" y="100" text-anchor="middle" font-size="9" fill="#57534E">A: 1 conn · 100 req</text>
<text x="89" y="124" text-anchor="middle" font-size="9" fill="#57534E">B: 1 conn · 2 req</text>
<rect x="196" y="52" width="130" height="86" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5" filter="url(#t2-sh)"/>
<text x="261" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">LB (L4)</text>
<text x="261" y="98" text-anchor="middle" font-size="10" fill="#BE123C">chia theo KẾT NỐI</text>
<rect x="210" y="106" width="102" height="22" rx="5" fill="#FFD9DE"/>
<text x="261" y="121" text-anchor="middle" font-size="10" font-weight="800" fill="#9F1239">TẦNG 1</text>
<rect x="368" y="40" width="150" height="48" rx="9" fill="#FFD9DE" stroke="#DC2626" stroke-width="2.5"/>
<rect x="368" y="102" width="150" height="48" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.8"/>
<text x="443" y="60" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Gateway gw-1</text>
<text x="443" y="79" text-anchor="middle" font-size="15" font-weight="800" fill="#DC2626">100 req</text>
<text x="443" y="122" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Gateway gw-2</text>
<text x="443" y="141" text-anchor="middle" font-size="15" font-weight="800" fill="#9F1239">2 req</text>
<rect x="560" y="52" width="130" height="86" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#t2-sh)"/>
<text x="625" y="78" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">Gateway (L7)</text>
<text x="625" y="98" text-anchor="middle" font-size="10" fill="#0F766E">chia theo REQUEST</text>
<rect x="574" y="106" width="102" height="22" rx="5" fill="#A7F3D0"/>
<text x="625" y="121" text-anchor="middle" font-size="10" font-weight="800" fill="#134E4A">TẦNG 2</text>
<rect x="732" y="34" width="144" height="34" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="732" y="78" width="144" height="34" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<rect x="732" y="122" width="144" height="34" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="804" y="56" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">be-1 · 34 req</text>
<text x="804" y="100" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">be-2 · 34 req</text>
<text x="804" y="144" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">be-3 · 34 req</text>
<line x1="158" y1="95" x2="192" y2="95" stroke="#57534E" stroke-width="2" marker-end="url(#t2-a)"/>
<path d="M330,88 C348,80 352,70 364,64" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#t2-a)"/>
<path d="M330,102 C348,110 352,120 364,126" stroke="#DC2626" stroke-width="1.6" fill="none" marker-end="url(#t2-a)"/>
<line x1="522" y1="95" x2="556" y2="95" stroke="#57534E" stroke-width="2" marker-end="url(#t2-a)"/>
<path d="M694,88 C710,74 716,60 728,51" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#t2-a)"/>
<line x1="694" y1="95" x2="728" y2="95" stroke="#0D9488" stroke-width="1.8" marker-end="url(#t2-a)"/>
<path d="M694,102 C710,116 716,130 728,139" stroke="#0D9488" stroke-width="1.8" fill="none" marker-end="url(#t2-a)"/>
<rect x="24" y="172" width="420" height="78" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="234" y="196" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Tầng 1 lệch 50.0×</text>
<text x="44" y="218" font-size="11" fill="#BE123C">gw-1 nhận 100, gw-2 nhận 2. LB chia đúng</text>
<text x="44" y="236" font-size="11" fill="#BE123C">2 kết nối sang 2 gateway — nhưng request thì không.</text>
<rect x="456" y="172" width="420" height="78" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="666" y="196" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">Tầng 2 đều 1.0×</text>
<text x="476" y="218" font-size="11" fill="#0F766E">Gateway đọc được HTTP nên chọn lại backend</text>
<text x="476" y="236" font-size="11" fill="#0F766E">cho TỪNG request → 34/34/34.</text>
<rect x="24" y="264" width="852" height="60" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="288" font-size="12" font-weight="800" fill="#374151">Hai tầng này KHÔNG dư thừa — chúng chia cho hai nhóm máy khác nhau</text>
<text x="44" y="310" font-size="11" fill="#57534E">Nhưng nếu dịch vụ Gateway managed đã lo sẵn entry point và phân phối traffic thì <tspan font-weight="700">không cần tự dựng thêm LB</tspan> chỉ để làm lại việc đó.</text>
<rect x="24" y="336" width="852" height="52" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="358" font-size="12" font-weight="800" fill="#7C4A03">Hệ quả vận hành của số 50.0×</text>
<text x="44" y="378" font-size="11" fill="#92400E">Dựng 4 Gateway để chia tải, nhưng nếu LB phía trước là L4 thì <tspan font-weight="700">tải thực tế có thể dồn vào một con</tspan>. Thêm instance không giải quyết được.</text>
</svg>
</div>

---

# Lúc deploy: port mở không có nghĩa là sẵn sàng

Đây là lỗi gây ra sự cố ngay trong lúc deploy, và nó rất dễ bị bỏ qua.

Tình huống: một Gateway vừa khởi động **đã mở port**, nhưng vẫn đang nạp cấu hình routing. LB thử kết nối thấy thành công nên bắt đầu chuyển request vào. Gateway chưa có đủ config để route — người dùng gặp lỗi.

Tôi dựng một instance mở port ngay nhưng chỉ sẵn sàng sau 2,5 giây:

```
Instance vừa khởi động: port 9004 đã MỞ, config chưa nạp xong
  → TCP connect: THÀNH CÔNG     ← đây là tất cả những gì LB L4 kiểm tra
  → GET /readyz : HTTP 503 not ready        ← readiness check thấy đúng
  → Request thật: HTTP 500 CONFIG_NOT_LOADED

Sau 2,5s (config đã nạp) → HTTP 200
```

**TCP connect thành công trong khi request thật trả về 500.** Nếu health check của bạn chỉ là "mở được port", LB sẽ gửi traffic vào từ giây đầu tiên và người dùng ăn lỗi 500 suốt 2,5 giây mỗi lần deploy một instance.

```nginx
# nginx mã nguồn mở không có active health check — nó chỉ passive
# (đánh dấu fail sau max_fails lần lỗi). Muốn readiness thật thì cần
# nginx Plus, hoặc để tầng orchestration lo.
upstream gateways {
  server gw-1:8080 max_fails=3 fail_timeout=10s;
  server gw-2:8080 max_fails=3 fail_timeout=10s;
}
```

```yaml
# Kubernetes: readinessProbe là thứ quyết định có nhận traffic hay không.
# TCP socket KHÔNG đủ — phải là endpoint tự báo "config đã nạp xong".
readinessProbe:
  httpGet: {path: /readyz, port: 8080}
  periodSeconds: 2
  failureThreshold: 2
livenessProbe:                      # khác mục đích: phát hiện treo, để RESTART
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 10
```

> 💡 **Phân biệt rõ hai probe**: `readiness` trả lời *"có nên gửi traffic vào không"*; `liveness` trả lời *"có nên restart không"*. Dùng cùng một endpoint cho cả hai là lỗi hay gặp — khi đó một instance đang nạp config sẽ bị **restart liên tục** thay vì được chờ.

---

# Lúc tắt: graceful shutdown và crash là hai chuyện khác nhau

Chiều ngược lại: ngừng chuyển request mới vào instance đó, **chờ các request đang chạy hoàn tất** rồi mới tắt.

Tôi đo với 30 request đang chạy dở (mỗi request mất 800ms), rồi tắt instance theo hai cách:

| Cách tắt | Request hoàn tất | Kết quả |
|---|---:|---|
| **Graceful shutdown** | **30/30** | ✅ không mất request nào |
| **Crash** (`Close()` ngay) | **0/30** | ❌ **cả 30 request thất bại** |

Chênh lệch tuyệt đối. Và điều quan trọng cần hiểu rõ:

> **LB không mang request đang xử lý dở sang instance khác để chạy tiếp.** Nó chỉ ngừng gửi request *mới*. Request đang chạy trên instance crash là mất.

```go
// Graceful shutdown trong Go — đúng thứ tự
func main() {
    srv := &http.Server{Addr: ":8080", Handler: h}
    go srv.ListenAndServe()

    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
    <-stop

    // 1. Báo readiness = false TRƯỚC, để LB ngừng gửi request mới
    ready.Store(false)
    // 2. Chờ LB nhận ra (= chu kỳ probe × failureThreshold)
    time.Sleep(5 * time.Second)
    // 3. Giờ mới drain request đang chạy
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

Bước 2 là bước hay bị thiếu nhất. Nếu `Shutdown()` ngay khi nhận `SIGTERM`, LB vẫn đang gửi request mới vào trong vài giây nữa — và những request đó bị từ chối. Phải **ngừng quảng bá mình là ready, chờ LB biết, rồi mới drain**.

> ⚠️ Kubernetes cũng vậy: `terminationGracePeriodSeconds` phải **dài hơn** (thời gian chờ LB + thời gian drain). Và `preStop` hook là chỗ đặt bước sleep đó.

---

# Service Discovery

Để LB và Gateway tự biết có instance mới, instance đổi địa chỉ, hoặc instance ngừng hoạt động, cần **Service Discovery kết hợp health check** — danh sách destination được cập nhật để phân phối traffic cho phù hợp.

| Cách | Cập nhật danh sách bằng | Phù hợp |
|---|---|---|
| **Cấu hình tĩnh** | Sửa file + reload | Số instance cố định, ít đổi |
| **DNS** | Resolve lại theo TTL | Đơn giản, nhưng TTL gây trễ và client hay cache quá lâu |
| **Kubernetes Service / Endpoints** | Control plane tự cập nhật | Đã chạy k8s |
| **Consul / etcd** | Instance tự đăng ký + health check | Nhiều môi trường, không k8s |
| **xDS (Envoy)** | Control plane đẩy xuống | Service mesh, cần cập nhật tức thì |

Phần này đáng một bài riêng nên tôi không đi sâu ở đây. Nhưng có một điểm đáng nhớ ngay: **health check của Service Discovery và readiness check của LB không nhất thiết là cùng một thứ** — và nếu chúng khác nhau, bạn sẽ có instance đã bị xoá khỏi discovery mà LB vẫn gửi traffic vào.

---

# Vậy chọn thế nào?

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ch6-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ch6-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="440" fill="#FFFDF7" rx="16"/>
<text x="430" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hỏi theo CÔNG VIỆC, không hỏi theo tên component</text>
<rect x="30" y="48" width="800" height="44" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="104" width="800" height="44" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="160" width="800" height="44" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="216" width="800" height="44" rx="9" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.8"/>
<rect x="30" y="48" width="10" height="44" rx="4" fill="#0D9488"/>
<rect x="30" y="104" width="10" height="44" rx="4" fill="#F5A623"/>
<rect x="30" y="160" width="10" height="44" rx="4" fill="#6366F1"/>
<rect x="30" y="216" width="10" height="44" rx="4" fill="#DC2626"/>
<text x="56" y="68" font-size="12" font-weight="800" fill="#374151">Chỉ cần route theo path + chia tải backend?</text>
<text x="56" y="85" font-size="11" fill="#0F766E">→ <tspan font-weight="700">Một LB L7 là đủ.</tspan> Đã đo: nginx làm được cả bước 1 và bước 3. Chưa cần Gateway.</text>
<text x="56" y="124" font-size="12" font-weight="800" fill="#374151">Cần gom token validation, rate limit, transform về một chỗ?</text>
<text x="56" y="141" font-size="11" fill="#92400E">→ <tspan font-weight="700">Gateway.</tspan> Đây là bước 2 — điểm mạnh thật của nó. Và Gateway chia tải backend luôn được.</text>
<text x="56" y="180" font-size="12" font-weight="800" fill="#374151">Cần scale CHÍNH Gateway (nhiều instance Gateway)?</text>
<text x="56" y="197" font-size="11" fill="#4338CA">→ <tspan font-weight="700">LB trước Gateway.</tspan> Hai nhóm destination khác nhau, không dư thừa.</text>
<text x="56" y="236" font-size="12" font-weight="800" fill="#374151">Dùng Gateway managed đã có entry point sẵn?</text>
<text x="56" y="253" font-size="11" fill="#BE123C">→ <tspan font-weight="700">Không tự dựng thêm LB</tspan> chỉ để làm lại việc dịch vụ đã lo. Xem cách triển khai cụ thể.</text>
<rect x="30" y="280" width="800" height="66" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<text x="50" y="304" font-size="13" font-weight="800" fill="#9F1239">Và bất kể chọn gì, ba câu này vẫn phải trả lời</text>
<text x="50" y="326" font-size="11" fill="#BE123C">① LB đang chia theo <tspan font-weight="700">kết nối</tspan> hay theo <tspan font-weight="700">request</tspan>?  ② Readiness check có <tspan font-weight="700">hơn</tspan> việc mở port không?</text>
<text x="50" y="342" font-size="11" fill="#BE123C">③ Lúc tắt instance có <tspan font-weight="700">ngừng nhận mới rồi mới drain</tspan> không?</text>
<rect x="30" y="362" width="800" height="62" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="50" y="386" font-size="12" font-weight="800" fill="#134E4A">Cách nghĩ thay cho công thức "nhiều server = LB, nhiều service = Gateway"</text>
<text x="50" y="408" font-size="11" fill="#0F766E">Liệt kê <tspan font-weight="700">ba bước một request phải đi qua</tspan>, rồi hỏi: thứ tôi đang có đã làm bước nào? Thiếu bước nào thì mới thêm.</text>
</svg>
</div>

| Nhu cầu | Đủ dùng | Chưa cần |
|---|---|---|
| Route theo path, chia tải backend | **LB L7** (nginx, ALB, Traefik) | Gateway riêng |
| Gateway đã chia tải xuống target | **Gateway** | LB giữa Gateway và backend |
| Scale chính Gateway, nhiều instance | **LB trước Gateway** | — |
| Gateway managed có entry point sẵn | **Chỉ Gateway** | LB tự dựng |

---

# Checklist

```
RANH GIỚI COMPONENT
□ Liệt kê 3 bước (route / policy / chọn instance) — thứ đang có làm bước nào?
□ Có component nào trong sơ đồ đang làm lại việc component khác đã làm?
□ Có LB nào đặt giữa Gateway và backend chỉ để chia tải lần hai không?
□ Dùng Gateway managed? → nó đã có entry point chưa, có cần LB nữa không?

PHÂN BỐ TẢI
□ LB đang chia theo KẾT NỐI (L4) hay theo REQUEST (L7)?
□ Client có dùng keep-alive / HTTP/2 không? (càng dùng, L4 càng lệch)
□ Đã ĐO phân bố request thật trên từng instance chưa, hay chỉ tin là đều?
□ Một instance bận hơn hẳn → kiểm cách chia TRƯỚC khi thêm instance
□ Request có đồng đều về chi phí xử lý không? (số request đều ≠ tải đều)

DEPLOY
□ Health check có hơn việc "mở được port" không?
□ readiness và liveness có TÁCH RIÊNG không? (dùng chung = restart khi đang nạp config)
□ Graceful shutdown: có NGỪNG báo ready, CHỜ LB biết, rồi mới drain?
□ terminationGracePeriod > (thời gian chờ LB + thời gian drain)?
□ Đã test kill instance giữa lúc có tải và ĐẾM request thất bại chưa?

SERVICE DISCOVERY
□ Danh sách destination cập nhật bằng gì khi instance đổi?
□ Health check của discovery và của LB có nhất quán không?
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| LB chỉ làm bước 3 phải không? | **Không.** LB **L7** làm được cả bước 1 (route theo path), một số LB cloud có sẵn auth |
| Gateway có chia tải được không? | **Có.** Kong có Upstream + Target, Envoy có cluster — tức Gateway làm luôn bước 3 |
| Chỉ route path + chia tải thì cần gì? | **Một LB L7 là đủ.** Thêm Gateway chỉ là thêm một hop |
| Sao vẫn đặt LB trước Gateway? | Để **scale chính Gateway** — hai nhóm destination khác nhau |
| Đặt LB rồi thì tải có đều? | **Chưa chắc.** Đo thật: L4 cho `100 / 2 / 0`, L7 cho `34 / 34 / 34` |
| Vì sao L4 lệch? | Nó chia theo **kết nối**. Keep-alive và HTTP/2 dồn nhiều request vào một kết nối |
| Hai tầng chia tải lệch bao nhiêu? | Đo cùng lúc: tầng L4 lệch **50.0×**, tầng L7 lệch **1.0×** |
| Mở được port là sẵn sàng? | **Không.** Đo thật: TCP connect OK trong khi request thật trả **500 CONFIG_NOT_LOADED** |
| Crash khác graceful bao nhiêu? | Đo thật: graceful **30/30** hoàn tất, crash **0/30** |
| LB có cứu request đang chạy dở không? | **Không.** Nó chỉ ngừng gửi request mới |

---

# Tham khảo

- [nginx — `ngx_stream_core_module`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html) (L4) · [`ngx_http_upstream_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html) (L7)
- [Kong — Upstreams and Targets](https://docs.konghq.com/gateway/latest/key-concepts/upstreams/) — Gateway làm bước 3
- [Kubernetes — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [AWS — Application Load Balancer authentication](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html) — LB cloud có auth sẵn
- [Envoy — Load balancing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview)
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — Redis không chỉ có GET/SET](/posts/system-design-p01-redis/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/) · [P03 — Pub/Sub vs Streams](/posts/system-design-p03-pubsub-vs-streams/) · [P04 — Streams vs Kafka](/posts/system-design-p04-streams-vs-kafka/) · [P05 — Redis Queue hay RabbitMQ?](/posts/system-design-p05-queue/)
