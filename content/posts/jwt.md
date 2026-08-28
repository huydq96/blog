+++
date = '2026-08-25T10:00:00+07:00'
draft = false
slug = 'jwt'
title = 'JWT: Những Điều Cần Biết Trước Khi Chọn Nó'
author = 'Huy Dang Quang'
categories = ["Security"]
tags = ["jwt", "authentication", "security", "golang", "session"]
description = 'JWT không dở, nhưng phần lớn chúng ta chọn nó theo bản năng chứ không phải theo bài toán. So sánh JWT với session, 5 câu hỏi trước khi chọn, và cái giá thật của logout khi dùng JWT.'
+++

> *Bài viết dựa trên và mở rộng từ [một bài viết gốc trên Facebook](https://www.facebook.com/permalink.php?story_fbid=pfbid0Vm8tLQg2ZT6AfLsHDTRgmEnPbKkbDfebQEpQHFUE3Z5ix48SPVbxeiZMDqox4Fmul&id=61590244986305), có bổ sung thêm code minh hoạ, sơ đồ, và vài lưu ý bảo mật.*

Tôi từng nhét JWT vào mọi chỗ, chỉ vì nghĩ nó mới và nó ngầu.

Có một thời JWT là thứ ai cũng nhắc tới. Stateless, hiện đại, không cần session store, nghe là thấy đúng kiểu kiến trúc mà một backend developer muốn đem đi khoe. Tôi dùng nó cho mọi dự án, không phân biệt cái nào cần cái nào không. Có dự án tôi còn thấy hơi tự hào vì đã bỏ được Redis ra khỏi luồng đăng nhập.

Gần đây đọc thiết kế của anh em, tôi thấy lại y nguyên cái tâm lý đó. Chọn JWT gần như là một phản xạ chứ không phải một quyết định. Hỏi "sao em chọn JWT" thì câu trả lời hay gặp nhất là "cho nó stateless anh ạ". Mà "stateless" là tên của giải pháp, không phải mô tả của vấn đề.

Không phải vì JWT dở, mà vì tôi đã áp dụng sai bài toán. Bài toán đó là một yêu cầu nghe rất bình thường: "cho tôi chức năng đăng xuất khỏi tất cả thiết bị, và sửa quyền nhân viên thì phải có hiệu lực ngay".

Nghe qua tưởng một ngày là xong. Thực tế thì không.

## Đọc nhanh: 5 câu hỏi trước khi chọn JWT

Nếu chỉ có hai phút, đọc phần này thôi cũng đủ.

<div style="max-width:800px;margin:2rem auto;">
<svg viewBox="0 0 800 800" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="jq-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="jq-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
<marker id="jq-arrow-amber" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#B45309"/>
</marker>
<marker id="jq-arrow-teal" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/>
</marker>
</defs>
<rect x="0" y="0" width="800" height="800" fill="#FFFDF7" rx="16"/>

<rect x="370" y="20" width="380" height="90" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="560" y="50" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">① Verify có tra được Redis/DB</text>
<text x="560" y="68" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">của bạn không?</text>
<rect x="30" y="20" width="300" height="90" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="180" y="48" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">→ CHỌN JWT</text>
<text x="180" y="68" text-anchor="middle" font-size="10.5" fill="#92400E">Sân chơi của nó: CDN/edge,</text>
<text x="180" y="84" text-anchor="middle" font-size="10.5" fill="#92400E">cross-region, đối tác ngoài</text>
<line x1="370" y1="65" x2="330" y2="65" stroke="#B45309" stroke-width="2" marker-end="url(#jq-arrow-amber)"/>
<rect x="270" y="38" width="90" height="18" rx="4" fill="#FFFDF7"/>
<text x="360" y="51" text-anchor="end" font-size="10" font-weight="700" fill="#B45309">KHÔNG tra được</text>
<line x1="560" y1="110" x2="560" y2="148" stroke="#57534E" stroke-width="2" marker-end="url(#jq-arrow)"/>
<rect x="574" y="123" width="150" height="18" rx="4" fill="#FFFDF7"/>
<text x="576" y="136" font-size="10" font-weight="700" fill="#57534E">tra được thoải mái</text>

<rect x="370" y="150" width="380" height="90" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="560" y="180" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">② Tuổi thọ token có NGẮN HƠN</text>
<text x="560" y="198" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">độ trễ nghiệp vụ chấp nhận?</text>
<rect x="30" y="150" width="300" height="90" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="180" y="178" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">→ CHỌN JWT</text>
<text x="180" y="198" text-anchor="middle" font-size="10.5" fill="#92400E">Token tự chết trước khi</text>
<text x="180" y="214" text-anchor="middle" font-size="10.5" fill="#92400E">kịp cần thu hồi</text>
<line x1="370" y1="195" x2="330" y2="195" stroke="#B45309" stroke-width="2" marker-end="url(#jq-arrow-amber)"/>
<rect x="300" y="170" width="65" height="18" rx="4" fill="#FFFDF7"/>
<text x="360" y="183" text-anchor="end" font-size="10" font-weight="700" fill="#B45309">NGẮN hơn</text>
<line x1="560" y1="240" x2="560" y2="278" stroke="#57534E" stroke-width="2" marker-end="url(#jq-arrow)"/>
<rect x="574" y="253" width="70" height="18" rx="4" fill="#FFFDF7"/>
<text x="576" y="266" font-size="10" font-weight="700" fill="#57534E">DÀI hơn</text>

<rect x="370" y="280" width="380" height="90" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="560" y="310" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">③ Cần thu hồi tức thì (vài giây):</text>
<text x="560" y="328" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">logout-all, khoá TK, sửa quyền?</text>
<rect x="30" y="280" width="300" height="90" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="180" y="332" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">→ CHỌN SESSION_ID</text>
<line x1="370" y1="325" x2="330" y2="325" stroke="#0D9488" stroke-width="2" marker-end="url(#jq-arrow-teal)"/>
<rect x="270" y="297" width="95" height="18" rx="4" fill="#FFFDF7"/>
<text x="360" y="310" text-anchor="end" font-size="10" font-weight="700" fill="#0D9488">CÓ, phải ăn ngay</text>
<line x1="560" y1="370" x2="560" y2="408" stroke="#57534E" stroke-width="2" marker-end="url(#jq-arrow)"/>
<rect x="574" y="376" width="220" height="32" rx="4" fill="#FFFDF7"/>
<text x="576" y="388" font-size="9.5" font-weight="700" fill="#57534E">không gấp (hoặc chấp nhận trễ</text>
<text x="576" y="401" font-size="9.5" font-weight="700" fill="#57534E">5-15 phút, ghi rõ trong tài liệu)</text>

<rect x="370" y="410" width="380" height="90" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="560" y="440" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">④ User cần thấy danh sách thiết bị</text>
<text x="560" y="458" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">& tự đăng xuất từng cái?</text>
<rect x="30" y="410" width="300" height="90" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="180" y="462" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">→ CHỌN SESSION_ID</text>
<line x1="370" y1="455" x2="330" y2="455" stroke="#0D9488" stroke-width="2" marker-end="url(#jq-arrow-teal)"/>
<rect x="335" y="434" width="30" height="18" rx="4" fill="#FFFDF7"/>
<text x="360" y="447" text-anchor="end" font-size="10" font-weight="700" fill="#0D9488">CÓ</text>
<line x1="560" y1="500" x2="560" y2="538" stroke="#57534E" stroke-width="2" marker-end="url(#jq-arrow)"/>
<rect x="574" y="513" width="55" height="18" rx="4" fill="#FFFDF7"/>
<text x="576" y="526" font-size="10" font-weight="700" fill="#57534E">KHÔNG</text>

<rect x="370" y="540" width="380" height="100" rx="14" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="560" y="570" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">⑤ Chịu được bản chất stateless —</text>
<text x="560" y="588" text-anchor="middle" font-size="12.5" font-weight="700" fill="#374151">phát ra rồi không rút lại được?</text>
<rect x="30" y="540" width="300" height="100" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#jq-shadow)"/>
<text x="180" y="596" text-anchor="middle" font-size="13.5" font-weight="800" fill="#7C4A03">→ DÙNG JWT THOẢI MÁI</text>
<line x1="370" y1="585" x2="330" y2="585" stroke="#B45309" stroke-width="2" marker-end="url(#jq-arrow-amber)"/>
<rect x="276" y="553" width="90" height="20" rx="4" fill="#FFFDF7"/>
<text x="360" y="567" text-anchor="end" font-size="10" font-weight="700" fill="#B45309">chấp nhận được</text>
<line x1="560" y1="640" x2="560" y2="678" stroke="#57534E" stroke-width="2" marker-end="url(#jq-arrow)"/>
<rect x="574" y="653" width="75" height="18" rx="4" fill="#FFFDF7"/>
<text x="576" y="666" font-size="10" font-weight="700" fill="#57534E">rợn người</text>

<rect x="370" y="680" width="380" height="80" rx="14" fill="#FFE9EC" stroke="#E11D48" stroke-width="2.5" filter="url(#jq-shadow)"/>
<text x="560" y="712" text-anchor="middle" font-size="13.5" font-weight="800" fill="#9F1239">→ ĐỪNG DÙNG JWT</text>
<text x="560" y="732" text-anchor="middle" font-size="10.5" fill="#BE123C">dù mọi tiêu chí khác có hợp tới đâu</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">5 câu hỏi tự kiểm tra trước khi chọn JWT — trả lời "CÓ" ở câu 3 hoặc 4 là dừng ngay ở session_id.</p>

Gói lại thành một câu để nhớ:

> **JWT là để nói chuyện qua một ranh giới mà bạn không hỏi lại được, hoặc để cấp một tấm vé sống ngắn.** Nếu bài toán của bạn không phải hai cái đó, khả năng cao bạn đang chọn nó vì quán tính.

Và một câu kiểm tra chéo tôi hay dùng khi review thiết kế của anh em: nếu bạn không nêu được một lý do cụ thể **khiến bạn không thể** tra một cái store ở mỗi request, thì bạn không có lý do để dùng JWT.

## Phần 1: Vì sao ai cũng chọn JWT theo bản năng

Phần này tôi để thật ngắn, nhưng phải nói, để công bằng với JWT. Điểm mạnh của nó là thật.

Server không phải lưu gì về phiên đăng nhập. Mọi thông tin nằm sẵn trong token, verify chữ ký xong là biết bạn là ai. Không tra Redis, không tra database, không bước chân ra khỏi tiến trình.

### JWT trông như thế nào?

Nếu bạn chưa quen định dạng, một JWT chỉ là 3 khối base64 nối bằng dấu chấm: `header.payload.signature`. Không có gì bí ẩn cả — bạn có thể copy bất kỳ token nào, dán vào [jwt.io](https://jwt.io) là thấy ngay 2 khối đầu chỉ là JSON thường.

<div style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="ja-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="ja-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#57534E"/>
</marker>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>

<rect x="40" y="30" width="260" height="50" rx="25" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="170" y="61" text-anchor="middle" font-size="12.5" font-family="monospace" fill="#0F766E">eyJhbGciOiJSUzI1NiJ9</text>
<rect x="320" y="30" width="260" height="50" rx="25" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="450" y="61" text-anchor="middle" font-size="12.5" font-family="monospace" fill="#92400E">eyJ1aWQiOiJ1XzEyMyJ9</text>
<rect x="600" y="30" width="260" height="50" rx="25" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="730" y="61" text-anchor="middle" font-size="12.5" font-family="monospace" fill="#9F1239">SflKxwRJSMeKKF2QT4f...</text>

<line x1="170" y1="80" x2="170" y2="118" stroke="#57534E" stroke-width="2" marker-end="url(#ja-arrow)"/>
<line x1="450" y1="80" x2="450" y2="118" stroke="#57534E" stroke-width="2" marker-end="url(#ja-arrow)"/>
<line x1="730" y1="80" x2="730" y2="118" stroke="#57534E" stroke-width="2" marker-end="url(#ja-arrow)"/>

<rect x="40" y="120" width="260" height="140" rx="14" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="170" y="148" text-anchor="middle" font-size="14" font-weight="800" fill="#134E4A">HEADER</text>
<text x="60" y="175" font-size="11.5" font-family="monospace" fill="#0F766E">{ "alg": "RS256",</text>
<text x="60" y="193" font-size="11.5" font-family="monospace" fill="#0F766E">  "typ": "JWT" }</text>
<text x="170" y="222" text-anchor="middle" font-size="10.5" fill="#0F766E">thuật toán ký +</text>
<text x="170" y="238" text-anchor="middle" font-size="10.5" fill="#0F766E">loại token</text>

<rect x="320" y="120" width="260" height="300" rx="14" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="450" y="148" text-anchor="middle" font-size="14" font-weight="800" fill="#7C4A03">PAYLOAD</text>
<text x="336" y="172" font-size="10.5" font-family="monospace" fill="#92400E">{ "uid": "u_123",</text>
<text x="336" y="190" font-size="10.5" font-family="monospace" fill="#92400E">  "jti": "a1b2c3",</text>
<text x="336" y="208" font-size="10.5" font-family="monospace" fill="#92400E">  "iat": 1755590000,</text>
<text x="336" y="226" font-size="10.5" font-family="monospace" fill="#92400E">  "exp": 1755590900,</text>
<text x="336" y="244" font-size="10.5" font-family="monospace" fill="#92400E">  "aud": "warehouse-svc" }</text>
<rect x="332" y="280" width="236" height="60" rx="8" fill="#FFE9EC" stroke="#E11D48" stroke-width="1.5"/>
<text x="450" y="304" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">⚠️ Chỉ encode base64,</text>
<text x="450" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">KHÔNG mã hoá — đừng nhét mật khẩu!</text>
<text x="450" y="360" text-anchor="middle" font-size="10.5" fill="#92400E">dữ liệu — ai đọc được token</text>
<text x="450" y="376" text-anchor="middle" font-size="10.5" fill="#92400E">cũng đọc được phần này</text>

<rect x="600" y="120" width="260" height="140" rx="14" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#ja-shadow)"/>
<text x="730" y="148" text-anchor="middle" font-size="14" font-weight="800" fill="#9F1239">SIGNATURE</text>
<text x="620" y="175" font-size="10.5" font-family="monospace" fill="#BE123C">RS256(header+payload,</text>
<text x="620" y="193" font-size="10.5" font-family="monospace" fill="#BE123C">  private key)</text>
<text x="730" y="222" text-anchor="middle" font-size="10.5" fill="#BE123C">chỉ auth-service</text>
<text x="730" y="238" text-anchor="middle" font-size="10.5" fill="#BE123C">có private key để ký</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Anatomy của 1 JWT: header và payload chỉ encode (ai cũng đọc được), chỉ signature là thứ chứng minh được ai đã tạo ra token này.</p>

Tôi vốn tin ngược lại, rằng JWT nặng hơn vì phải parse và verify chữ ký, nên ngồi đo thật (Go 1.24, container giới hạn 1 CPU core, Redis 7 container riêng cùng network). Số liệu bác bỏ tôi:

| | Thời gian | Ghi chú |
|---|---|---|
| Verify 1 JWT (RS256) | **5.03 μs** | chạy ngay trong tiến trình |
| Tra 1 session trong Redis | **112.30 μs** | một lượt đi-về qua mạng |

Rẻ hơn 22 lần. Khác biệt lớn hơn nằm ở khả năng mở rộng: verify JWT đạt khoảng **152 nghìn lượt/giây trên mỗi core**, thêm server là thêm tuyến tính, không phụ thuộc một component trung tâm nào.

Nhanh hơn, mở rộng tốt hơn. Đó là lý do JWT thành phản xạ mặc định, và tôi cho rằng phản xạ đó có cơ sở — miễn là bạn dừng lại đúng lúc.

## Phần 2: Những thiết kế không nên dùng JWT (và đó là phần lớn ứng dụng)

Toàn bộ lợi thế ở Phần 1 đứng trên một giả định duy nhất: bạn không cần thu hồi token. Ngay khi nghiệp vụ yêu cầu thu hồi, mọi thứ đổi. Đây là phần chính của bài, và là chỗ tôi va tường ngày xưa.

### 2.1 Logout với session_id là chuyện vặt

Sự thật về phiên nằm ở server, nên logout chỉ là xoá nó đi:

```bash
DEL sess:<session_id>
SREM user_sessions:<uid> <session_id>
```

Xong. Hiệu lực tức thì, không cửa sổ trễ. Request tiếp theo mang `session_id` đó tra không thấy, trả 401. Logout tất cả thiết bị thì lấy danh sách phiên của user rồi xoá cả loạt: với một user có 5 thiết bị là khoảng **6 lệnh Redis**, chạy đúng một lần trong đời tài khoản đó. Thu hồi quyền và khoá tài khoản dùng chung cơ chế này.

Giữ con số 6 lệnh trong đầu, lát nữa nó có ích.

### 2.2 Logout với JWT, mặc định, là logout giả

Đây là điều tôi mong anh em nắm cho thật chắc, vì ngày xưa tôi không nắm.

Khi client bấm logout trong một hệ JWT thuần, cái duy nhất xảy ra là client tự xoá token khỏi bộ nhớ trình duyệt. Server không được thông báo gì, và kể cả có được thông báo cũng không làm gì được. Token đã ký là đã phát ra ngoài thế giới. Chữ ký vẫn hợp lệ, hạn vẫn còn. Bất kỳ ai đang giữ một bản sao chuỗi token đó — qua log, qua proxy, qua extension trình duyệt, qua một ảnh chụp màn hình DevTools — vẫn gọi API bình thường cho tới lúc token hết hạn.

Nói cách khác: bạn không đăng xuất người dùng. Bạn chỉ yêu cầu trình duyệt của họ quên đi cái chìa khoá, còn ổ khoá vẫn mở với bất cứ ai còn giữ bản sao chìa.

Với nhiều hệ thống thì như vậy chấp nhận được, và tôi không cho là sai. Một blog cá nhân, một công cụ nội bộ mà mô hình đe doạ không bao gồm "token bị đánh cắp", thì logout giả hoàn toàn ổn. Nhưng nó phải là một **quyết định có ý thức**, được ghi lại, được bên nghiệp vụ biết. Vấn đề là trong phần lớn dự án tôi từng thấy, đó không phải quyết định. Đó là hệ quả không ai nhận ra, cho tới ngày có yêu cầu mới.

### 2.3 Bốn cách vá, và giá của từng cách

Khi sếp yêu cầu logout phải là logout thật, bạn có bốn lựa chọn. Tôi liệt kê theo đúng thứ tự các team thường mò ra.

**Cách A: Denylist theo `jti`**

Mỗi token mang claim `jti` (JWT ID), là một mã ngẫu nhiên riêng cho từng token, kiểu số seri in trên tờ vé. Hai lần đăng nhập ra hai `jti` khác nhau.

```go
claims := Claims{
    UID: user.ID,
    RegisteredClaims: jwt.RegisteredClaims{
        ID:        uuid.NewString(),                              // jti
        IssuedAt:  jwt.NewNumericDate(now),
        ExpiresAt: jwt.NewNumericDate(now.Add(15 * time.Minute)),
        Issuer:    "auth-service",
    },
}
token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
signed, err := token.SignedString(privateKey)
```

Lúc logout, ghi mã đó vào Redis với TTL đúng bằng thời gian sống còn lại của token:

```bash
SET deny:jti:<jti> 1 EX <so_giay_con_lai>
```

Mỗi request, sau khi verify chữ ký, phải hỏi thêm `EXISTS deny:jti:<jti>`.

Có một nỗi sợ phổ biến ở đây mà tôi muốn dập đi, vì nó khiến nhiều người loại phương án này vì lý do sai: sợ denylist phình to vô hạn. Không hề — TTL mỗi bản ghi chỉ bằng thời gian sống còn lại của token nên nó tự dọn. Thử tính: 1 triệu user, 5% logout mỗi ngày là 50 nghìn lượt; access token sống 15 phút thì số bản ghi tồn tại đồng thời chỉ khoảng **520**, tức vài chục KB.

Cái đắt không phải bộ nhớ. Cái đắt là bạn vừa thêm một lượt đi-về qua mạng vào **mọi** request — đúng cái thứ mà bạn chọn JWT để tránh.

**Cách B: Một cột mốc thời gian theo user**

Lưu ở server một mốc cho mỗi user, kiểu `uinvalid:<uid> = 10:30 sáng nay`. Token thì đã sẵn có claim `iat` (issued at), đóng dấu lúc ký. Mỗi request so `iat` với mốc đó, token nào phát ra **trước** mốc thì từ chối. Đăng xuất tất cả thiết bị chỉ là ghi mốc bằng thời điểm hiện tại.

Có biến thể dùng số nguyên: lưu `uver:<uid> = 7`, token mang thêm claim `ver: 7`, logout tất cả là tăng bộ đếm lên 8. Tôi thích biến thể thời gian hơn — nó không cần thêm claim nào vì `iat` có sẵn, và nó cộng dồn được theo tầng: một mốc toàn hệ thống dùng khi có sự cố bảo mật, một mốc theo tenant, một mốc theo user, rồi lấy giá trị lớn nhất để so với `iat`. Với số nguyên thì hai bộ đếm độc lập không so với nhau được.

Cả hai biến thể có chung hai giới hạn mà ít bài nói ra. Thứ nhất, nó **không** làm được logout **một** thiết bị: đẩy mốc lên là đá văng toàn bộ thiết bị của user. Thứ hai, và quan trọng hơn: nó vẫn bắt bạn tra Redis ở mọi request.

**Cách C: Danh sách phiên đang hoạt động**

Token mang claim `sid`. Server giữ một tập các `sid` còn sống của mỗi user, tra ở mỗi request. Cách này giải quyết được mọi yêu cầu nghiệp vụ — nhưng hãy nhìn thẳng vào thứ bạn vừa dựng lên: một bảng ánh xạ từ mã phiên sang trạng thái phiên, tra ở mỗi request, thu hồi được tức thì. Đó chính xác là định nghĩa của **session**. Khác biệt duy nhất còn lại là bạn vẫn phải verify chữ ký, vẫn gửi token to hơn hàng chục lần, vẫn phải quản lý và xoay khoá ký. Trả toàn bộ chi phí của JWT để nhận về đúng ngữ nghĩa của session.

**Cách D: Access token ngắn cộng refresh token có state**

Access token là JWT sống rất ngắn (5-15 phút) và **không** kiểm tra thu hồi ở mỗi request. Refresh token là chuỗi ngẫu nhiên thường, lưu database, thu hồi được.

```go
type RefreshToken struct {
    ID        string
    UserID    string
    DeviceID  string
    TokenHash string // hash SHA-256, không lưu plaintext
    ExpiresAt time.Time
    RevokedAt *time.Time
}

// Rotation: mỗi lần refresh, thu hồi bản ghi cũ và phát bản ghi mới.
// Nếu một refresh token đã-bị-thu-hồi bị dùng lại -> dấu hiệu bị đánh cắp,
// thu hồi luôn TOÀN BỘ refresh token của user đó.
func Rotate(ctx context.Context, db *sql.DB, presentedHash string) (accessJWT, newRefresh string, err error) {
    rt, err := findByHash(ctx, db, presentedHash)
    if err != nil {
        return "", "", err
    }
    if rt.RevokedAt != nil {
        revokeAllForUser(ctx, db, rt.UserID) // reuse detection
        return "", "", errTokenReuse
    }
    revoke(ctx, db, rt.ID)
    newRT := issueRefreshToken(ctx, db, rt.UserID, rt.DeviceID)
    accessJWT = signAccessToken(rt.UserID, 15*time.Minute)
    return accessJWT, newRT, nil
}
```

Logout một thiết bị là xoá bản ghi refresh token của thiết bị đó. Logout tất cả là xoá toàn bộ bản ghi của user.

Đây là phương án duy nhất **giữ được** lợi thế của JWT, vì đường đi nóng — đoạn code chạy ở mọi request — không chạm database. Đó là lý do nó là mặc định hợp lý cho phần lớn ứng dụng web.

Nhưng nó có một cái giá không thương lượng được: **cửa sổ trễ**. Trong 5-15 phút đó, người vừa bị đuổi việc vẫn gọi API được. Kẻ vừa bị phát hiện gian lận vẫn đặt đơn được. Quyền vừa bị thu hồi vẫn dùng được.

Còn một cách nữa nhưng nó không phải cơ chế logout: **đổi khoá ký**. Xoay khoá làm mọi token cũ vô hiệu ngay lập tức, nhưng đá văng toàn bộ user của hệ thống. Đây là nút bấm khi khoá bị lộ, không phải nút bấm khi một user logout.

### 2.4 Logout tất cả thiết bị: chỗ Cách A gãy hoàn toàn

Đây là điểm logic tôi cho là đáng giá nhất trong cả bài, và ít thấy ai nói ra.

Denylist theo `jti` **tự nó** không giải được bài toán logout tất cả thiết bị. Lý do rất đơn giản: để chặn toàn bộ token của một user, bạn phải biết toàn bộ `jti` đã phát cho user đó và còn hạn. Nhưng JWT là stateless — bạn không lưu chúng ở đâu cả, không có danh sách để mà chặn.

Muốn có danh sách đó, bạn phải bắt đầu lưu mọi `jti` đã phát ra kèm chủ sở hữu và kèm hạn. Ngay giây phút đó, bạn đã có một session store, chỉ là được gọi bằng cái tên khác và được truy cập kèm một bước verify chữ ký thừa.

Cho nên trong thực tế, mọi hệ JWT có chức năng logout tất cả thiết bị đều rơi vào một trong ba trạng thái: hoặc dùng cột mốc thời gian và mất khả năng logout từng thiết bị, hoặc dùng danh sách phiên và đã là session store, hoặc chấp nhận cửa sổ trễ của Cách D. Không có lối thoát thứ tư — đây không phải hạn chế của thư viện hay của cách triển khai, mà là hệ quả tất yếu của việc chọn một tấm vé tự chứng minh.

### 2.5 Thiết kế đầy đủ nhất mà JWT đạt tới được, và năm cái bẫy

Kết hợp Cách A với Cách B thì bạn có thiết kế đầy đủ nhất mà một hệ JWT làm được:

- Logout **một** thiết bị: chặn theo `jti`, tức số seri của đúng tấm vé đó.
- Logout **tất cả** thiết bị: so `iat` của token với cột mốc thời gian lưu theo user.

Thiết kế này đáp ứng được cả hai yêu cầu nghiệp vụ, và tôi cho rằng đây là điểm cuối của con đường: bạn không làm tốt hơn thế được mà vẫn còn gọi nó là JWT. Nhưng nó đi kèm năm cái bẫy, và điểm chung của cả năm là chúng im lặng — không cái nào ném ra lỗi, không cái nào làm test đỏ.

**Bẫy 1 — Hai cơ chế nghĩa là hai lần tra.** Viết ngây thơ thì mỗi request gọi `EXISTS deny:jti:<jti>` rồi gọi tiếp `GET uinvalid:<uid>` — hai lượt đi-về tuần tự. Gộp lại bằng một lệnh là xong:

```bash
MGET deny:jti:<jti> uinvalid:<uid>
```

Nhưng nếu chạy Redis Cluster, hai key này có thể rơi vào hai node khác nhau và `MGET` bị từ chối thẳng. Lúc đó phải đặt tên key khéo để ép chúng về cùng một hash slot (dùng hash tag `{uid}`), hoặc gói hai lệnh vào một Lua script. Đây là loại chi tiết mà không xử lý thì hệ chạy ngon ở dev một node, rồi vỡ đúng ngày lên cluster.

**Bẫy 2 — Lỗ hổng một giây của `iat`.** Claim `iat` tính bằng **giây**, không phải mili giây (chuẩn của JWT là vậy — xem [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519#section-2)). Mọi token phát ra trong cùng một giây đều có `iat` giống hệt nhau.

Hình dung: token phát lúc `10:30:45.200`, người dùng bấm đăng xuất tất cả thiết bị lúc `10:30:45.800`. Bạn ghi `uinvalid = 10:30:45`. Điều kiện từ chối là `iat < uinvalid`, mà `10:30:45` không nhỏ hơn `10:30:45`, nên **cái token đó sống sót** qua lệnh đăng xuất tất cả thiết bị.

Đổi sang `iat <= uinvalid` thì hết lỗ hổng, nhưng sinh vấn đề ngược: người dùng đăng nhập lại trong cùng giây đó sẽ nhận token mới cũng bị từ chối, không vào được cho tới khi sang giây kế tiếp. Cách xử lý sạch là đừng dùng `iat` cho việc này — thêm một claim riêng, ví dụ `iat_ms` tính bằng mili giây, rồi so sánh trên thang mili giây:

```go
func CheckRevoked(ctx context.Context, rdb *redis.Client, jti, uid string, iatMs int64) (bool, error) {
    vals, err := rdb.MGet(ctx, "deny:jti:"+jti, "uinvalid_ms:"+uid).Result()
    if err != nil {
        return false, err
    }
    if vals[0] != nil {
        return true, nil // jti nằm trong denylist
    }
    if vals[1] != nil {
        cutoffMs, _ := strconv.ParseInt(vals[1].(string), 10, 64)
        if iatMs <= cutoffMs {
            return true, nil // token phát ra trước-hoặc-đúng mốc logout-all
        }
    }
    return false, nil
}
```

Nghe nhỏ nhặt, nhưng "đăng xuất tất cả thiết bị mà một thiết bị vẫn vào được" là loại bug bạn không tài nào tái hiện được khi ngồi debug, vì nó chỉ xảy ra khi hai sự kiện rơi đúng vào cùng một giây.

**Bẫy 3 — Lấy thời gian từ đâu.** `iat` do auth service đóng dấu, mốc `uinvalid` do service xử lý logout ghi. Nếu hai service chạy trên hai máy có đồng hồ lệch nhau vài giây, bạn gặp một trong hai hậu quả: token hợp lệ vừa phát ra bị từ chối oan, hoặc token đáng lẽ phải chết thì vẫn sống. Nguyên tắc: cả hai mốc phải lấy từ **cùng một nguồn** — dùng lệnh `TIME` của Redis hoặc `now()` của database, chứ đừng lấy đồng hồ của từng application server. NTP chỉ giảm lệch chứ không xoá được lệch, và lúc container vừa khởi động lại là lúc đồng hồ hay sai nhất.

**Bẫy 4 — TTL của bản ghi chặn phải tính từ `exp`.** Khi ghi `deny:jti`, TTL phải đúng bằng thời gian sống **còn lại** của token, tức `exp` trừ đi thời điểm hiện tại. Code đặt TTL cố định 5 phút cho gọn, trong khi token còn sống 30 phút, sẽ dẫn tới: sau 5 phút bản ghi chặn bốc hơi và cái token đã bị logout **sống lại**, dùng tiếp được 25 phút nữa. Không lỗi nào ném ra, không log nào bất thường — chức năng logout chỉ đơn giản là ngừng có tác dụng sau 5 phút.

**Bẫy 5, nguy hiểm nhất — Redis đuổi key là huỷ lệnh thu hồi.** Nhiều hệ dùng chung một Redis cho cả cache lẫn các bản ghi thu hồi này, với `maxmemory-policy` là `allkeys-lru`. Khi đầy bộ nhớ, Redis tự đuổi bớt key ít được dùng nhất để lấy chỗ.

Bản ghi `deny:jti` của một token đã bị logout là key gần như không bao giờ được đọc trúng (chủ nhân của nó đã bị đá ra rồi) — nó nằm đúng nhóm bị đuổi đầu tiên. Khi bị đuổi, token bị thu hồi **hợp lệ trở lại**. Tương tự với mốc `uinvalid`: key đó bị đuổi thì mọi token cũ của user sống lại hết.

Đây là kiểu hỏng tệ nhất trong bảo mật: hệ thống tự mở khoá cho kẻ tấn công đúng lúc nó đang chịu tải cao, và không ai được cảnh báo. Nguyên tắc: dữ liệu thu hồi **không phải cache**. Nó phải nằm ở nơi bền vững, hoặc ít nhất là một Redis riêng có bật persistence và đặt:

```
maxmemory-policy noeviction
```

Với `session_id` thì lỗi này cũng tồn tại nhưng theo hướng **ngược lại** và an toàn hơn nhiều: session bị đuổi thì người dùng bị đăng xuất oan — hỏng theo kiểu **đóng**. Còn JWT thì hỏng theo kiểu **mở**. Cùng một sự cố hạ tầng, một bên gây phiền, một bên gây thủng.

### 2.6 Một lỗi cụ thể hay gặp: race khi logout-all gặp refresh

Tình huống: user bấm "đăng xuất tất cả thiết bị" trên điện thoại. Đúng lúc đó, laptop của họ đang gọi refresh vì access token vừa hết hạn.

Nếu luồng refresh đọc mốc thu hồi **trước** khi lệnh ghi mốc kịp hoàn tất, laptop sẽ nhận được một access token mới hợp lệ và sống thêm nguyên một chu kỳ nữa. Người dùng đã bấm đăng xuất tất cả, hệ thống báo thành công, mà thiết bị kia vẫn vào được.

Cách xử lý là để việc phát token mới và việc đọc mốc nằm trong cùng một thao tác nguyên tử, và luôn kiểm tra lại mốc ngay trước khi ký. Với `session_id` thì lỗi này không tồn tại, vì không có bước phát lại chìa khoá nào cả.

Tôi nêu nó ra vì nó minh hoạ một điểm chung: mỗi lớp vá thêm vào JWT lại mở ra một loại lỗi mới mà mô hình session đơn giản không hề có.

### 2.7 Phép tính khiến tôi đổi ý hoàn toàn

So chi phí thật của cùng một chức năng:

| Phương án | Chi phí logout-all (1 lần) | Chi phí mỗi request (mãi mãi) |
|---|---|---|
| `session_id`, user 5 thiết bị | ~6 lệnh Redis | 1 lượt tra store |
| JWT + cột mốc thời gian | 1 lệnh `SET` (rẻ hơn) | 1 lượt tra store **cộng thêm** verify chữ ký |

Giả sử user đó dùng 5 thiết bị, mỗi thiết bị 100 request/ngày. Chỉ riêng user đó, chỉ trong một ngày, JWT đã tốn thêm **500 lệnh Redis**, so với 6 lệnh một lần của session. Ngày mai lại 500 lệnh nữa.

Bạn vừa dời chi phí từ một sự kiện **hiếm** (logout, vài lần một năm cho mỗi user) sang một sự kiện **liên tục** (mọi request, hàng nghìn lần một ngày). Trong tối ưu hệ thống, đó là hướng đánh đổi sai một cách kinh điển.

Nhưng phần cay đắng nhất không cần con số nào để chứng minh. Hãy nhìn hình dạng của hai đường đi — đường đi thứ hai chứa trọn đường đi thứ nhất rồi cộng thêm việc. Nó **không bao giờ** rẻ hơn được, dù bạn tối ưu tới đâu, dù Redis của bạn nhanh cỡ nào. Cái lợi thế 22 lần ở Phần 1 chỉ tồn tại chừng nào bạn không cần thu hồi.

Trả nhiều tiền hơn để mua một thứ kém hơn. Đó là câu tôi muốn anh em nhớ nhất từ bài này.

### 2.8 Tóm lại phần logout

Nếu nghiệp vụ của bạn có một trong các yêu cầu sau, hãy nghiêm túc cân nhắc `session_id`:

- Đăng xuất khỏi tất cả thiết bị, phải hiệu lực ngay
- Đăng xuất từng thiết bị cụ thể, và người dùng nhìn thấy danh sách thiết bị
- Thu hồi quyền, khoá tài khoản, đình chỉ nhân viên, phải ăn ngay
- Đổi mật khẩu thì phải đá hết mọi phiên khác
- Cần biết ai đang online, bao nhiêu phiên, đăng nhập từ đâu

Còn nếu bạn dùng JWT, hãy trả lời được câu này trước khi code dòng đầu tiên: **cửa sổ thu hồi của hệ thống này là bao nhiêu phút, và bên nghiệp vụ đã đồng ý với con số đó chưa?** Trong phần lớn dự án, con số đó chưa bao giờ được hỏi. Nó được quyết định tình cờ bởi một dòng `expiresIn` mà ai đó copy từ Stack Overflow. Ngày xưa tôi cũng vậy.

## Phần 3: Vậy khi nào nên dùng JWT?

Nói vậy không có nghĩa JWT yếu. Nó mạnh thật, nhưng mạnh ở một sân chơi rất cụ thể, còn chúng ta thì hay mang nó sang sân chơi khác.

Tính chất gốc của JWT gói trong một câu: bên nhận tự quyết định được "đây là ai, được làm gì" mà không phải hỏi ai cả, chỉ cần giữ khoá. Mọi điểm mạnh là hệ quả của câu đó, và mọi điểm yếu cũng vậy: đã nói ra rồi thì không rút lại được. Từ đó suy ra luật chọn đáng nhớ nhất:

> Nếu tuổi thọ của token **ngắn hơn** độ trễ thu hồi mà nghiệp vụ chấp nhận được, thì thu hồi không còn là vấn đề và JWT thắng tuyệt đối. Nếu **dài hơn**, bạn đang tự dựng lại session store, chỉ là dựng xấu hơn.

**1. Đăng nhập liên tổ chức (SSO)**

Login bằng Google, Okta, Keycloak. Bạn không có quyền tra session store của Google, chữ ký là cách duy nhất để tin. Nhưng để ý chi tiết nhiều người bỏ qua: cái token Google trả về được dùng đúng **một lần** lúc login để dựng phiên phía bạn, rồi vứt đi. Người ta copy cái định dạng mà không copy cái vòng đời.

**2. Giao tiếp giữa các service, khi danh tính người dùng phải đi qua nhiều chặng**

Đây là lý do chính đáng nhất để dùng JWT trong hệ thống nội bộ, mà lại hay bị nói lướt.

Hình dung luồng đặt hàng thật: người dùng bấm Thanh toán, request vào `order-service`, `order-service` gọi sang `inventory-service` để trừ tồn kho, `inventory-service` gọi tiếp sang `warehouse-service` để chọn kho xuất. Ba service, ba team, ba repo. Câu hỏi: `warehouse-service`, ở chặng thứ ba, làm sao biết thao tác này đang thay mặt nhân viên nào, và nhân viên đó có quyền xuất kho ở chi nhánh này không?

Với `session_id`, hai cách làm hiển nhiên nhất đều dở. Cách thứ nhất: mỗi service tự cầm `session_id` rồi gọi ngược về session store để tra — `warehouse-service`, vốn chẳng liên quan gì tới đăng nhập, giờ phải có mật khẩu Redis auth và phải xử lý tình huống Redis chết, nhân với 20 service thì cả hệ thống cùng phụ thuộc vào **một** component. Cách thứ hai: `order-service` tra một lần rồi truyền xuống dưới bằng header thường kiểu `X-User-Id: 12345` — nhanh, gọn, và sai về bảo mật, vì header đó không chứng minh được gì cả. Bất kỳ ai chọc được vào mạng nội bộ cũng đặt được `X-User-Id: 1` để đóng vai admin. Bạn vừa biến toàn bộ phân quyền thành hệ danh dự.

Với JWT thì bài toán tan biến. Nhưng có đúng một chi tiết phải làm cho chuẩn, và đây là chỗ tôi thấy nhiều người hiểu sai nhất:

> **Chỉ có một nơi được ký token, và nơi đó không phải `order-service`.**

Auth service giữ private key và là nơi duy nhất ký. Nó công bố public key qua một endpoint chuẩn tên là **JWKS** (JSON Web Key Set) — thực chất chỉ là một file JSON chứa public key, để service nào cần thì tự tải về rồi cache lại. `order-service` **không** tạo token mới, nó chỉ chuyển tiếp nguyên cái token nó nhận được xuống dưới. `warehouse-service` verify bằng public key tải từ JWKS, không gọi đi đâu cả, cũng không cần biết Redis auth tồn tại.

```go
jwks, _ := keyfunc.NewDefaultCtx(ctx, []string{
    "https://auth.internal/.well-known/jwks.json",
})

token, err := jwt.ParseWithClaims(tokenString, &Claims{}, jwks.Keyfunc,
    jwt.WithValidMethods([]string{"RS256"}), // chặn "alg confusion" — xem khung dưới
    jwt.WithAudience("warehouse-service"),   // chặn token bị dùng sai chỗ nó được cấp cho
    jwt.WithIssuer("auth-service"),
)
```

> **Hai điểm tôi muốn thêm vào bài gốc, vì đây là chỗ nhiều thư viện JWT từng dính CVE thật:**
>
> - **Alg confusion.** Đừng bao giờ để code verify tự đọc `alg` trong header rồi làm theo — luôn khai báo cứng thuật toán được chấp nhận (`WithValidMethods`). Có một lớp bug JWT nổi tiếng: nếu code verify tin vào `alg` trong token, kẻ tấn công đổi `alg` thành `HS256` rồi lấy chính **public key RS256** (vốn công khai qua JWKS) làm secret để ký — nhiều thư viện cũ verify HS256 bằng đúng chuỗi bytes đó, thế là token giả được coi là hợp lệ. Cố định thuật toán ở phía verify là cách chặn triệt để.
> - **Kiểm tra `aud`/`iss`, không chỉ chữ ký.** Chữ ký hợp lệ chỉ chứng minh token do đúng auth service ký ra — nó không chứng minh token này được cấp *cho service của bạn*. Một token hợp lệ cấp cho `payment-service` mà bị phát hiện dùng sai chỗ ở `warehouse-service` (do log rò rỉ, do proxy debug) vẫn nên bị từ chối. Đó là việc của claim `aud` (audience) và `iss` (issuer), không phải việc của signature.

Nên số khoá phải quản lý là **một** cặp cho cả hệ thống, không phải mỗi service một cặp. Nếu thiết kế của bạn để service nào cũng ký được token thì đó là thiết kế sai, và sai nặng — bạn vừa cho 20 service quyền tự phong danh tính cho bất kỳ ai. Đây cũng là lý do phải dùng RS256 hoặc ES256 chứ không phải HS256 cho hệ nhiều service: HS256 dùng chung một khoá cho cả ký lẫn verify, tức là phát khoá ký cho mọi service.

Đây là lúc "không cần một nơi verify chung" là lợi ích thật, chứ không phải câu chữ trên slide. Để ý cách nói cho chính xác: **không cần một nơi verify chung, nhưng vẫn có một nơi ký chung.** Hai chuyện đó khác nhau, và lẫn lộn chúng là nguồn gốc của phần lớn thiết kế JWT hỏng mà tôi từng đọc.

### Phản biện đáng giá nhất: "session_id cộng gateway thì sao?"

Tới đây kiểu gì cũng có bạn hỏi ngược, và đây là câu hỏi hay nhất trong cả chủ đề: sao không làm giống hệt JWT — cũng có một auth service và một gateway, mọi request từ ngoài đều bắt buộc đi qua gateway, gateway tra `session_id` một lần rồi bơm thông tin người dùng xuống dưới, còn các service nội bộ gọi nhau thì khỏi verify?

Được. Nó chạy thật, và rất nhiều công ty đang chạy đúng như vậy. Mô hình này có tên: **mô hình vành đai**, nói dân dã là vỏ cứng ruột mềm. Tôi công nhận trước ba điểm đúng của nó, rồi mới nói chỗ nó gãy.

Thứ nhất, gateway chỉ tra store **một lần** cho mỗi request, không phải mỗi service tra một lần — nên cái lo "20 service cùng phụ thuộc vào Redis auth" ở trên **không** áp dụng cho thiết kế này. Thứ hai, thu hồi vẫn tức thì, vì gateway tra store thật ở mỗi request — đây đúng là thứ JWT thuần không làm được, và ở đây nó là điểm mạnh. Thứ ba, chi phí rẻ: một lượt tra cho cả chuỗi gọi, dù chuỗi đó dài ba chặng hay mười chặng.

Đem so với JWT thuần thì mô hình này thắng ở chỗ thu hồi và hoà ở chỗ chi phí. Nếu chỉ dừng ở đó thì nó là lựa chọn tốt hơn. Chỗ gãy nằm ở bảo mật, không phải hiệu năng.

**Điểm gãy: niềm tin bắc cầu**

Gateway làm tốt việc của nó. Chỗ gãy nằm ở đoạn sau gateway. Ở chặng thứ ba, `warehouse-service` nhận được header `X-User-Id: 12345`. Nó không có cách nào phân biệt "gateway đặt header đó, `inventory-service` chỉ chuyển tiếp trung thực" với "`inventory-service` tự bịa ra con số đó" — vì một chuỗi ký tự trần không mang theo bằng chứng nào cả.

<div style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="jt-shadow" x="-20%" y="-20%" width="140%" height="140%">
<feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="#00000022"/>
</filter>
<marker id="jt-arrow-grey" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#9CA3AF"/>
</marker>
<marker id="jt-arrow-green" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M0,0 L10,5 L0,10 z" fill="#16A34A"/>
</marker>
</defs>
<rect x="0" y="0" width="900" height="560" fill="#FFFDF7" rx="16"/>
<text x="450" y="34" text-anchor="middle" font-size="15" font-weight="800" fill="#DC2626">❌ Gateway + header nội bộ — niềm tin bắc cầu</text>
<rect x="40" y="60" width="150" height="70" rx="12" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="115" y="100" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Gateway</text>
<rect x="230" y="60" width="150" height="70" rx="12" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="305" y="100" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">order-svc</text>
<rect x="420" y="60" width="150" height="70" rx="12" fill="#FFE9EC" stroke="#E11D48" stroke-width="2.5" filter="url(#jt-shadow)"/>
<text x="495" y="92" text-anchor="middle" font-size="12" font-weight="700" fill="#9F1239">inventory-svc</text>
<text x="495" y="110" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">✕ bị chiếm</text>
<rect x="610" y="60" width="150" height="70" rx="12" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="685" y="100" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">warehouse-svc</text>
<line x1="190" y1="95" x2="228" y2="95" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="4 4" marker-end="url(#jt-arrow-grey)"/>
<line x1="380" y1="95" x2="418" y2="95" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="4 4" marker-end="url(#jt-arrow-grey)"/>
<line x1="570" y1="95" x2="608" y2="95" stroke="#9CA3AF" stroke-width="2" stroke-dasharray="4 4" marker-end="url(#jt-arrow-grey)"/>
<text x="400" y="45" text-anchor="middle" font-size="10" font-family="monospace" fill="#9CA3AF">X-User-Id: 12345 (chỉ 1 chuỗi, không bằng chứng)</text>
<rect x="420" y="150" width="340" height="50" rx="8" fill="#FFE9EC" stroke="#E11D48" stroke-width="1.5"/>
<text x="590" y="170" text-anchor="middle" font-size="10.5" fill="#9F1239">inventory-svc tự bịa X-User-Id: 1 (admin)</text>
<text x="590" y="186" text-anchor="middle" font-size="10.5" fill="#9F1239">→ warehouse-svc tin luôn, không cách nào phát hiện</text>
<text x="450" y="230" text-anchor="middle" font-size="11.5" fill="#6B7280">Một service thủng ⇒ đóng vai BẤT KỲ ai, ở MỌI service phía sau, ngay lập tức</text>
<line x1="450" y1="260" x2="450" y2="290" stroke="#D1D5DB" stroke-width="1.5" stroke-dasharray="2 4"/>
<text x="450" y="320" text-anchor="middle" font-size="15" font-weight="800" fill="#16A34A">✅ Token có chữ ký — verify từng chặng</text>
<rect x="40" y="346" width="150" height="70" rx="12" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="115" y="378" text-anchor="middle" font-size="12" font-weight="700" fill="#134E4A">Auth</text>
<text x="115" y="394" text-anchor="middle" font-size="9.5" fill="#0F766E">(ký token)</text>
<rect x="230" y="346" width="150" height="70" rx="12" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="305" y="386" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">order-svc</text>
<rect x="420" y="346" width="150" height="70" rx="12" fill="#FFE9EC" stroke="#E11D48" stroke-width="2.5" filter="url(#jt-shadow)"/>
<text x="495" y="378" text-anchor="middle" font-size="12" font-weight="700" fill="#9F1239">inventory-svc</text>
<text x="495" y="396" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">✕ bị chiếm</text>
<rect x="610" y="346" width="150" height="70" rx="12" fill="#F3F4F6" stroke="#6B7280" stroke-width="2" filter="url(#jt-shadow)"/>
<text x="685" y="386" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">warehouse-svc</text>
<line x1="190" y1="381" x2="228" y2="381" stroke="#16A34A" stroke-width="2" marker-end="url(#jt-arrow-green)"/>
<line x1="380" y1="381" x2="418" y2="381" stroke="#16A34A" stroke-width="2" marker-end="url(#jt-arrow-green)"/>
<line x1="570" y1="381" x2="608" y2="381" stroke="#16A34A" stroke-width="2" marker-end="url(#jt-arrow-green)"/>
<text x="400" y="336" text-anchor="middle" font-size="10" font-family="monospace" fill="#16A34A">JWT (có chữ ký) — mỗi chặng tự verify</text>
<rect x="420" y="436" width="340" height="60" rx="8" fill="#F0FDF4" stroke="#16A34A" stroke-width="1.5"/>
<text x="590" y="456" text-anchor="middle" font-size="10.5" fill="#166534">inventory-svc đọc được token đi qua nó,</text>
<text x="590" y="472" text-anchor="middle" font-size="10.5" fill="#166534">nhưng KHÔNG tự ký được token mới</text>
<text x="590" y="488" text-anchor="middle" font-size="10.5" fill="#166534">(không có private key)</text>
<text x="450" y="524" text-anchor="middle" font-size="11.5" fill="#166534">Một service thủng ⇒ chỉ lộ đúng phần nó đã thấy, không giả danh được ai khác</text>
</svg>
</div>
<p style="text-align:center;font-size:0.9em;color:#8A8272;font-style:italic;">Niềm tin bắc cầu (trust A→B→C mà chưa từng kiểm tra C) so với chữ ký verify độc lập ở từng chặng.</p>

Niềm tin ở đây bắc cầu: A tin B, B tin C, nên A phải tin C mà chưa từng kiểm tra C bao giờ. Hệ quả nằm ở phạm vi thiệt hại: chỉ cần **một** service bị chiếm là kẻ tấn công đóng vai được **bất kỳ** người dùng nào, ở **mọi** service còn lại, ngay lập tức — không phải leo thang từng bước, mà là toàn quyền ngay từ bước đầu.

So với token có chữ ký cho rõ. Kẻ chiếm được `inventory-service` vẫn nguy hiểm, nó vẫn dùng lại được những token đã đi qua nó. Nhưng nó **không** tự phát ra danh tính mới được, vì nó không giữ private key. Nó bị nhốt trong tập những người dùng thật sự đã đi qua nó, trong khoảng thời gian token còn hạn. Đó là khác biệt giữa **mất một phần** và **mất tất cả**.

**Ba đường vào có thật, xếp theo mức hay gặp:**

1. **Gateway quên gỡ header do client gửi lên.** Lỗi kinh điển và hay gặp nhất. Nếu gateway chỉ biết **thêm** header `X-User-Id` mà không **xoá** cái header cùng tên do client gửi kèm, thì bất kỳ ai ngồi ngoài Internet cũng gửi lên `X-User-Id: 1` và thành admin. Vành đai sập từ ngoài vào, không cần chiếm service nào cả. Nguyên tắc: mọi header nội bộ phải nằm trong danh sách bị gỡ bắt buộc ở gateway, và phải có test tự động cho việc đó — đây là loại lỗi lặng lẽ, thêm một route mới là nó tái xuất hiện.
2. **SSRF ở bất kỳ service nào.** SSRF (server side request forgery) là lỗi khiến bạn dụ được server gọi hộ mình tới một địa chỉ do mình chọn, ví dụ chức năng "nhập ảnh từ URL" mà không kiểm tra URL. Chỉ cần một chỗ như thế ở bất kỳ service nào trong mạng, kẻ tấn công đã có sẵn một bàn đạp nằm bên trong vành đai để gọi thẳng vào `warehouse-service` kèm header tuỳ ý. Lỗ hổng ở service A lại phá vỡ phân quyền của service C — hai thứ chẳng liên quan gì nhau về nghiệp vụ.
3. **Một service được expose tạm để test rồi quên gỡ**, hoặc một ingress cấu hình nhầm trỏ thẳng vào service thay vì qua gateway. Cái này không phải lỗ hổng phần mềm, nó là chuyện vận hành, và chính vì vậy nó xảy ra thường xuyên hơn hai cái trên cộng lại.

**Chỗ hai mô hình gặp nhau**

Nếu đẩy câu hỏi thêm một bước nữa, bạn sẽ tự đi tới lời giải: để **gateway đổi `session_id` lấy một token nội bộ sống 60 giây**, rồi mới forward xuống dưới.

Lúc đó bạn có cả hai thứ cùng lúc. Thu hồi tức thì ở rìa, vì gateway vẫn tra store thật ở mỗi request từ ngoài vào. Danh tính verify được ở mọi chặng bên trong, vì token có chữ ký. Chỉ **một** nơi ký là gateway nên không nở khoá. Và cửa sổ 60 giây ngắn tới mức toàn bộ phần thu hồi rắc rối ở Phần 2 không còn ý nghĩa: token chết trước khi bạn kịp muốn thu hồi nó.

Tôi cho rằng đây là thiết kế đúng cho phần lớn hệ nhiều service. Gói cả mục này lại thành một câu:

> **Mô hình gateway thì niềm tin bắc cầu, một service thủng là thủng hết. Token có chữ ký thì niềm tin verify được ở từng chặng, một service thủng thì thiệt hại dừng ở phạm vi những gì nó đã nhìn thấy.**

**3. Verify ở nơi không với tới được auth store**

Code chạy trên CDN. Gateway chặn request rác trước khi chạm application. Và mạnh nhất là multi-region: session nằm ở Singapore, request rơi vào Frankfurt. JWT biến bài toán đồng bộ dữ liệu thành bài toán phân phối khoá, mà khoá thì một tháng đổi một lần còn session thì mỗi giây đổi hàng nghìn. Đây là lập luận mạnh nhất bênh JWT và tôi công nhận nó hoàn toàn.

**4. Token một mục đích, sống ngắn, dùng như một tấm vé**

Link tải file có hạn, link xác thực email, link mời vào tổ chức, magic link, token callback cho webhook thanh toán. Nhóm này JWT không những hợp mà còn đang bị **dùng thiếu**: tôi thấy nhiều team dựng hẳn một bảng trong database để quản lý mấy cái link tạm, kèm cron dọn rác chạy hàng đêm, trong khi một token ký sống 15 phút là đủ và không cần dọn gì cả.

**5. Fan-out đọc rất lớn mà hậu quả của dữ liệu cũ là nhỏ**

API nội dung công khai, feature flag, personalization. Thông tin cũ 5 phút thì hậu quả là gợi ý sai, không phải lộ đơn hàng.

**Điều kiện tiên quyết cho cả năm trường hợp trên**: hệ thống của bạn phải chịu được bản chất stateless của JWT. Token đã phát ra thì không rút lại được. Ứng dụng nào không chịu được điều đó thì đừng dùng, dù mọi tiêu chí còn lại có hợp tới đâu.

Ngoại lệ đáng nói nhất: **API key cấp cho bên thứ ba**. Nghe rất hợp lý để dùng JWT, nhưng mọi nhà cung cấp API lớn đều dùng key dạng chuỗi ngẫu nhiên kèm tra store, vì hai lý do rất đời: cần chặn được **ngay** trong vài giây khi có ai đó lỡ đẩy key lên GitHub, và cần đếm usage theo từng key để tính tiền. Đây là chỗ trực giác đánh lừa nhiều người nhất.

### Câu trả lời thực dụng cho phần lớn ứng dụng web

Vẫn là kiến trúc lai ở Cách D: access token JWT sống ngắn, không kiểm tra thu hồi ở đường đi nóng, cộng refresh token dạng chuỗi ngẫu nhiên lưu DB và thu hồi được.

Nhưng phải trung thực với chính mình về hai điều. Thứ nhất, cửa sổ trễ là có thật và bên nghiệp vụ phải biết nó dài bao nhiêu. Thứ hai, rất nhiều team đặt access token sống 24 giờ để giảm lưu lượng refresh, và thế là cửa sổ thu hồi thành 24 giờ — họ quay về đúng vấn đề ban đầu, nhưng lần này tự tin hơn vì "đã dùng kiến trúc chuẩn".

### Bảng tra nhanh theo loại hệ thống

Bạn dò xem dự án mình rơi vào đâu.

| Loại hệ thống | Chọn gì | Vì sao |
|---|---|---|
| Web nội bộ công ty: ERP, CRM, quản lý kho, nhân sự | `session_id` | Phân quyền phức tạp, có đình chỉ nhân viên, một region — JWT chỉ tổ thêm việc |
| Sàn thương mại điện tử, app đặt hàng, ví điện tử | `session_id` (+ JWT cho vé phụ) | Có khoá tài khoản gian lận, đăng xuất tất cả thiết bị, đổi mật khẩu phải đá hết phiên. JWT dùng cho link xác thực, link tải hoá đơn |
| App mobile đăng nhập một lần, dùng cả tháng | Kiến trúc lai | Access token JWT 15 phút + refresh token thường lưu DB, mỗi thiết bị một bản ghi — chỗ danh sách thiết bị làm sạch nhất |
| Hệ thống nhiều service gọi nhau | JWT nội bộ (60s) sau gateway | Gateway đổi `session_id` thành JWT nội bộ sống ngắn — thu hồi tức thì ở rìa, stateless ở trong |
| Public API cho bên thứ ba, cấp API key | Key ngẫu nhiên, tra store | Cần chặn ngay khi lộ key, cần đếm usage để tính tiền |
| SSO nội bộ, đăng nhập một lần cho nhiều app con | JWT ở tầng liên ứng dụng | Mỗi app con nên đổi token đó lấy phiên riêng rồi vứt token đi |
| Trang tin tức, blog, nội dung công khai có cá nhân hoá nhẹ | JWT | Hậu quả dữ liệu cũ rất nhỏ |
| Nhiều region, cùng tài khoản dùng ở VN và Mỹ | JWT cho access token | Thay thế duy nhất là replicate session store xuyên lục địa |
| Game, chat, realtime qua WebSocket | JWT lúc bắt tay | Danh tính gắn với kết nối, không verify lại mỗi message. Thu hồi = đóng kết nối (vốn đã có state) |
| Ngân hàng, bảo hiểm, y tế, nơi có kiểm toán | `session_id` | "Khoảng 15 phút nữa họ tự hết hạn" không phải câu trả lời kiểm toán chấp nhận |

Nếu dự án của bạn không nằm trong danh sách trên, quay lại 5 câu hỏi ở đầu bài.

## Kết

Ngày trước tôi chọn JWT vì nó mới và nó ngầu. Tôi không phân tích gì cả, và tôi nghĩ nhiều bạn bây giờ cũng đang ở đúng chỗ tôi đứng khi đó.

Điều buồn cười là khi ngồi đo lại tử tế, tôi phát hiện mình sai cả ở chiều ngược lại: JWT không hề nặng, verify nó rẻ hơn tra Redis tới 22 lần. Nhưng ngay khi nghiệp vụ yêu cầu logout thật, đăng xuất tất cả thiết bị, hay sửa quyền ăn ngay, bạn buộc phải tra một cái store ở mỗi request và toàn bộ lợi thế đó bốc hơi. Bạn quay về đúng mô hình session, nhưng cõng thêm chi phí verify, cõng thêm token to, cõng thêm quản lý khoá, và vẫn kém chính xác hơn.

Cho nên chọn JWT hay session không phải một quyết định kỹ thuật. Nó là câu hỏi: **sự thật về danh tính người dùng nên nằm ở đâu, và bạn chấp nhận nó sai trong bao lâu.** Trả lời được câu đó rồi thì phần còn lại chỉ là chi tiết triển khai.
