+++
date = '2026-09-21T10:00:00+07:00'
draft = false
slug = 'mysql-join'
title = 'MySQL JOIN: Từ cú pháp đến thuật toán'
author = 'Huy Dang Quang'
categories = ["MySQL"]
tags = ["mysql", "database", "join", "performance"]
description = 'Các loại JOIN trong MySQL, thuật toán bên dưới (Nested Loop, Hash Join, Sort-Merge), cách đọc EXPLAIN và tối ưu thực tế'
+++

# Tổng quan

`JOIN` là hệ quả trực tiếp của **chuẩn hoá dữ liệu**. Ta tách `customers` và `orders` thành hai bảng để không lặp lại tên khách hàng 10.000 lần — nhưng cái giá phải trả là: mỗi lần cần xem "đơn hàng này của ai", database phải *ghép* hai bảng lại. `JOIN` chính là thao tác ghép đó.

Hầu hết dev backend viết `JOIN` khá thành thạo ở mức cú pháp. Nhưng khi query chậm 500ms mà không hiểu vì sao, vấn đề gần như luôn nằm ở tầng dưới: **MySQL thực thi JOIN bằng thuật toán nào, theo thứ tự nào, và index có được dùng không**. Bài này đi từ trên xuống dưới, và mọi con số trong bài đều được đo thật trên **MySQL 8.0.40**.

> 📌 **Lưu ý phiên bản**: hash join chỉ có từ **MySQL 8.0.18**, và từ **8.0.20** nó thay thế hoàn toàn Block Nested Loop. Nếu bạn còn chạy MySQL 5.7, phần lớn mục "Hash Join" trong bài này **không áp dụng** — 5.7 chỉ có Nested Loop.

---

## Dữ liệu mẫu

Toàn bộ ví dụ trong bài dùng schema này (tạo bằng MySQL 8.0.40):

```sql
CREATE TABLE customers (
  id         INT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  city       VARCHAR(50)  NOT NULL,
  created_at DATETIME NOT NULL,
  KEY idx_customers_city (city)
) ENGINE=InnoDB;

CREATE TABLE orders (
  id           INT PRIMARY KEY,
  customer_id  INT NOT NULL,
  status       VARCHAR(20) NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL,
  ordered_at   DATETIME NOT NULL,
  KEY idx_orders_customer (customer_id),
  KEY idx_orders_status (status)
) ENGINE=InnoDB;

CREATE TABLE order_items (
  id         INT PRIMARY KEY,
  order_id   INT NOT NULL,
  product_id INT NOT NULL,
  qty        INT NOT NULL,
  price      DECIMAL(10,2) NOT NULL,
  KEY idx_items_order (order_id)
) ENGINE=InnoDB;

CREATE TABLE products (
  id INT PRIMARY KEY, category_id INT NOT NULL,
  name VARCHAR(100), price DECIMAL(10,2),
  KEY idx_products_category (category_id)
) ENGINE=InnoDB;
```

Số dòng: `customers` 1.000 · `orders` 10.000 · `order_items` 30.000 · `products` 500.

Và một cặp bảng tí hon để minh hoạ các loại JOIN cho dễ nhìn:

```sql
-- emp                                    -- dept
+----+-------+---------+------------+     +----+-------------+
| id | name  | dept_id | manager_id |     | id | name        |
+----+-------+---------+------------+     +----+-------------+
|  1 | An    |       1 |       NULL |     |  1 | Engineering |
|  2 | Binh  |       1 |          1 |     |  2 | Sales       |
|  3 | Cuong |       2 |          1 |     |  3 | Legal       |
|  4 | Dung  |    NULL |          2 |     +----+-------------+
+----+-------+---------+------------+
```

Để ý hai "ca đặc biệt" được cài sẵn:
- **Dung** có `dept_id = NULL` → không thuộc phòng ban nào.
- **Legal** không có nhân viên nào.

Đây chính là hai dòng quyết định sự khác nhau giữa các loại JOIN.

---

# Phần 1 — Các loại JOIN

## JOIN thực sự làm gì ở mức dòng

Trước khi nói INNER hay LEFT, cần hiểu bản chất: JOIN duyệt qua các **cặp dòng** `(dòng bảng trái, dòng bảng phải)`, kiểm tra điều kiện `ON`, và quyết định giữ cặp nào. Loại JOIN chỉ khác nhau ở chỗ: **những dòng không tìm được bạn nhảy thì xử lý ra sao**.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 470" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="jn-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="jn-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="470" fill="#FFFDF7" rx="16"/>
<text x="450" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">JOIN ghép dòng như thế nào</text>
<rect x="60" y="55" width="230" height="30" rx="8" fill="#0D9488"/>
<rect x="60" y="85" width="230" height="152" fill="#FFFFFF" stroke="#0D9488" stroke-width="2"/>
<rect x="610" y="55" width="230" height="30" rx="8" fill="#F5A623"/>
<rect x="610" y="85" width="230" height="114" fill="#FFFFFF" stroke="#F5A623" stroke-width="2"/>
<rect x="60" y="199" width="230" height="38" fill="#FFE9EC"/>
<rect x="610" y="161" width="230" height="38" fill="#FFE9EC"/>
<rect x="60" y="199" width="230" height="38" fill="none" stroke="#FF6B6B" stroke-width="2.5"/>
<rect x="610" y="161" width="230" height="38" fill="none" stroke="#FF6B6B" stroke-width="2.5"/>
<text x="175" y="75" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">emp (bảng trái)</text>
<text x="725" y="75" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">dept (bảng phải)</text>
<line x1="60" y1="123" x2="290" y2="123" stroke="#E7E5E4" stroke-width="1"/>
<line x1="60" y1="161" x2="290" y2="161" stroke="#E7E5E4" stroke-width="1"/>
<line x1="60" y1="199" x2="290" y2="199" stroke="#E7E5E4" stroke-width="1"/>
<line x1="610" y1="123" x2="840" y2="123" stroke="#E7E5E4" stroke-width="1"/>
<line x1="610" y1="161" x2="840" y2="161" stroke="#E7E5E4" stroke-width="1"/>
<text x="78" y="110" font-size="12" fill="#374151">An</text>
<text x="200" y="110" font-size="11" fill="#78716C">dept_id=1</text>
<text x="78" y="148" font-size="12" fill="#374151">Binh</text>
<text x="200" y="148" font-size="11" fill="#78716C">dept_id=1</text>
<text x="78" y="186" font-size="12" fill="#374151">Cuong</text>
<text x="200" y="186" font-size="11" fill="#78716C">dept_id=2</text>
<text x="78" y="224" font-size="12" font-weight="700" fill="#9F1239">Dung</text>
<text x="196" y="224" font-size="11" font-weight="700" fill="#9F1239">dept_id=NULL</text>
<text x="628" y="110" font-size="12" fill="#374151">Engineering</text>
<text x="820" y="110" text-anchor="end" font-size="11" fill="#78716C">id=1</text>
<text x="628" y="148" font-size="12" fill="#374151">Sales</text>
<text x="820" y="148" text-anchor="end" font-size="11" fill="#78716C">id=2</text>
<text x="628" y="186" font-size="12" font-weight="700" fill="#9F1239">Legal</text>
<text x="820" y="186" text-anchor="end" font-size="11" font-weight="700" fill="#9F1239">id=3</text>
<path d="M292,104 L608,104" stroke="#0D9488" stroke-width="2" marker-end="url(#jn-arr)"/>
<path d="M292,142 C420,142 480,110 608,110" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#jn-arr)"/>
<path d="M292,180 C420,180 480,148 608,148" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#jn-arr)"/>
<path d="M292,218 L400,218" stroke="#FF6B6B" stroke-width="2" stroke-dasharray="5 4" fill="none"/>
<path d="M608,186 L500,186" stroke="#FF6B6B" stroke-width="2" stroke-dasharray="5 4" fill="none"/>
<text x="412" y="223" font-size="13" font-weight="800" fill="#FF6B6B">✗</text>
<text x="482" y="191" font-size="13" font-weight="800" fill="#FF6B6B">✗</text>
<text x="450" y="90" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">3 cặp khớp ON dept.id = emp.dept_id</text>
<text x="450" y="250" text-anchor="middle" font-size="11" fill="#9F1239">2 dòng mồ côi — chính chúng quyết định bạn chọn loại JOIN nào</text>
<rect x="45" y="275" width="195" height="170" rx="12" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2" filter="url(#jn-sh)"/>
<rect x="260" y="275" width="195" height="170" rx="12" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#jn-sh)"/>
<rect x="475" y="275" width="195" height="170" rx="12" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#jn-sh)"/>
<rect x="690" y="275" width="195" height="170" rx="12" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2" filter="url(#jn-sh)"/>
<text x="142" y="300" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">INNER JOIN</text>
<text x="357" y="300" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">LEFT JOIN</text>
<text x="572" y="300" text-anchor="middle" font-size="13" font-weight="800" fill="#7C4A03">RIGHT JOIN</text>
<text x="787" y="300" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">FULL OUTER</text>
<text x="142" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">3 dòng</text>
<text x="357" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#92400E">4 dòng</text>
<text x="572" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#92400E">4 dòng</text>
<text x="787" y="322" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">5 dòng</text>
<text x="142" y="346" text-anchor="middle" font-size="11" fill="#57534E">chỉ 3 cặp khớp</text>
<text x="142" y="364" text-anchor="middle" font-size="11" fill="#57534E">Dung: loại</text>
<text x="142" y="382" text-anchor="middle" font-size="11" fill="#57534E">Legal: loại</text>
<text x="357" y="346" text-anchor="middle" font-size="11" fill="#57534E">3 cặp khớp</text>
<text x="357" y="364" text-anchor="middle" font-size="11" font-weight="700" fill="#92400E">+ Dung, dept=NULL</text>
<text x="357" y="382" text-anchor="middle" font-size="11" fill="#57534E">Legal: loại</text>
<text x="572" y="346" text-anchor="middle" font-size="11" fill="#57534E">3 cặp khớp</text>
<text x="572" y="364" text-anchor="middle" font-size="11" fill="#57534E">Dung: loại</text>
<text x="572" y="382" text-anchor="middle" font-size="11" font-weight="700" fill="#92400E">+ Legal, emp=NULL</text>
<text x="787" y="346" text-anchor="middle" font-size="11" fill="#57534E">3 cặp khớp</text>
<text x="787" y="364" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">+ Dung + Legal</text>
<text x="787" y="382" text-anchor="middle" font-size="11" fill="#57534E">giữ cả hai bên</text>
<rect x="55" y="398" width="175" height="34" rx="8" fill="#CCFBF1"/>
<rect x="270" y="398" width="175" height="34" rx="8" fill="#FDE9C8"/>
<rect x="485" y="398" width="175" height="34" rx="8" fill="#FDE9C8"/>
<rect x="700" y="398" width="175" height="34" rx="8" fill="#FFD9DE"/>
<text x="142" y="419" text-anchor="middle" font-size="10" fill="#134E4A">mặc định, dùng nhiều nhất</text>
<text x="357" y="419" text-anchor="middle" font-size="10" fill="#7C4A03">giữ trọn bảng trái</text>
<text x="572" y="419" text-anchor="middle" font-size="10" fill="#7C4A03">giữ trọn bảng phải</text>
<text x="787" y="413" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">MySQL KHÔNG hỗ trợ</text>
<text x="787" y="427" text-anchor="middle" font-size="10" fill="#9F1239">phải giả lập bằng UNION</text>
</svg>
</div>

---

## INNER JOIN

Chỉ giữ những cặp dòng **khớp được điều kiện `ON`**. Dòng mồ côi ở cả hai bên đều bị loại.

```sql
SELECT e.name AS emp, d.name AS dept
FROM emp e
INNER JOIN dept d ON d.id = e.dept_id;
```

```
+-------+-------------+
| emp   | dept        |
+-------+-------------+
| An    | Engineering |
| Binh  | Engineering |
| Cuong | Sales       |
+-------+-------------+
```

`Dung` biến mất (không có phòng ban), `Legal` biến mất (không có nhân viên).

**Ý nghĩa**: "cho tôi những bản ghi có quan hệ đầy đủ ở cả hai bên".

> `INNER JOIN`, `JOIN`, và `CROSS JOIN ... ON` trong MySQL là **đồng nghĩa hoàn toàn** — cả ba đều parse ra cùng một cây thực thi. Nên viết `INNER JOIN` cho rõ ràng, hoặc tối thiểu là `JOIN`.

**Khi nào dùng**: mặc định. ~80% JOIN trong ứng dụng thật nên là INNER JOIN — nếu bạn *biết chắc* khoá ngoại luôn tồn tại thì INNER JOIN vừa đúng nghĩa vừa cho optimizer nhiều tự do nhất (nó được phép đổi thứ tự bảng thoải mái).

---

## LEFT JOIN / RIGHT JOIN (OUTER JOIN)

`LEFT JOIN` giữ **toàn bộ** dòng bảng trái. Dòng nào không tìm được bạn nhảy bên phải thì các cột bên phải được điền `NULL`.

```sql
SELECT e.name AS emp, d.name AS dept
FROM emp e
LEFT JOIN dept d ON d.id = e.dept_id;
```

```
+-------+-------------+
| emp   | dept        |
+-------+-------------+
| An    | Engineering |
| Binh  | Engineering |
| Cuong | Sales       |
| Dung  | NULL        |   <-- giữ lại, cột phải = NULL
+-------+-------------+
```

`RIGHT JOIN` là bản gương: giữ toàn bộ bảng phải.

```
+-------+-------------+
| emp   | dept        |
+-------+-------------+
| Binh  | Engineering |
| An    | Engineering |
| Cuong | Sales       |
| NULL  | Legal       |   <-- giữ lại, cột trái = NULL
+-------+-------------+
```

**Thực tế: hầu như không ai dùng RIGHT JOIN.** `A RIGHT JOIN B` luôn viết lại được thành `B LEFT JOIN A`, và đọc từ trái sang phải dễ hơn nhiều. MySQL cũng chuẩn hoá RIGHT thành LEFT ngay trong optimizer. Giữ quy ước "chỉ dùng LEFT JOIN" giúp codebase bớt một lớp phải suy nghĩ.

**Khi nào dùng LEFT JOIN**:
- Dữ liệu bên phải **có thể không tồn tại** một cách hợp lệ: user chưa có avatar, đơn hàng chưa có mã giảm giá, sản phẩm chưa có review.
- Báo cáo cần đủ mọi dòng bên trái kể cả khi không có dữ liệu: "doanh thu 12 tháng" — tháng nào không có đơn vẫn phải hiện 0.
- Tìm dòng **không có** quan hệ (anti-join, xem phần dưới).

**Chi phí ẩn**: LEFT JOIN **trói tay optimizer**. Với INNER JOIN, MySQL được phép chọn bảng nào chạy trước tuỳ ý. Với LEFT JOIN, ngữ nghĩa "giữ trọn bảng trái" buộc bảng trái phải là bảng ngoài (trừ vài trường hợp optimizer chứng minh được là có thể viết lại). Đừng dùng LEFT JOIN vì "cho chắc" khi INNER JOIN mới đúng.

---

## FULL OUTER JOIN — MySQL không có

Chuẩn SQL có `FULL OUTER JOIN` (giữ trọn cả hai bên). **MySQL không hỗ trợ cú pháp này** — PostgreSQL, Oracle, SQL Server thì có. Cách giả lập:

```sql
SELECT e.name AS emp, d.name AS dept FROM emp e LEFT  JOIN dept d ON d.id = e.dept_id
UNION
SELECT e.name,        d.name        FROM emp e RIGHT JOIN dept d ON d.id = e.dept_id;
```

```
+-------+-------------+
| emp   | dept        |
+-------+-------------+
| An    | Engineering |
| Binh  | Engineering |
| Cuong | Sales       |
| Dung  | NULL        |
| NULL  | Legal       |
+-------+-------------+
```

⚠️ **Bẫy `UNION` vs `UNION ALL`**: `UNION` khử trùng lặp (tốn một bước sort/hash), nên nếu dữ liệu gốc **đã có dòng trùng hợp lệ**, chúng sẽ bị gộp mất. Bản đúng-về-ngữ-nghĩa mà vẫn nhanh:

```sql
SELECT e.name, d.name FROM emp e LEFT JOIN dept d ON d.id = e.dept_id
UNION ALL
SELECT e.name, d.name FROM emp e RIGHT JOIN dept d ON d.id = e.dept_id
WHERE e.id IS NULL;   -- chỉ lấy thêm phần mồ côi bên phải, không trùng với nhánh trên
```

---

## CROSS JOIN

Tích Descartes — ghép **mọi** dòng bên trái với **mọi** dòng bên phải, không có điều kiện `ON`.

```sql
SELECT COUNT(*) FROM emp CROSS JOIN dept;   -- 4 x 3 = 12
```

Số dòng ra là `M × N`. Với hai bảng 10.000 dòng thì đó là **100 triệu dòng** — đây là lý do `CROSS JOIN` vô tình (quên `ON`, hoặc `ON` sai) là một trong những cách giết server nhanh nhất.

**Dùng hợp lệ khi nào**: sinh dữ liệu tổ hợp. Ví dụ báo cáo cần đủ mọi cặp (tháng × phòng ban) kể cả khi không có giao dịch:

```sql
SELECT m.month_label, d.name, COALESCE(SUM(o.total_amount), 0) AS revenue
FROM   calendar_months m
CROSS JOIN dept d
LEFT  JOIN orders o
       ON  YEAR(o.ordered_at)  = m.y AND MONTH(o.ordered_at) = m.m
       AND o.dept_id = d.id
GROUP BY m.month_label, d.name;
```

Đây là mẫu "khung báo cáo đầy đủ" kinh điển: `CROSS JOIN` tạo khung, `LEFT JOIN` đổ dữ liệu vào khung.

---

## SELF JOIN

Không phải một loại JOIN riêng — chỉ là JOIN một bảng với **chính nó**, dùng alias để phân biệt. Dùng cho quan hệ phân cấp trong cùng bảng.

```sql
SELECT e.name AS emp, m.name AS manager
FROM emp e
LEFT JOIN emp m ON m.id = e.manager_id;
```

```
+-------+---------+
| emp   | manager |
+-------+---------+
| An    | NULL    |   <-- sếp to nhất, không có manager
| Binh  | An      |
| Cuong | An      |
| Dung  | Binh    |
+-------+---------+
```

Chú ý phải là `LEFT JOIN`: nếu dùng `INNER JOIN` thì `An` (không có manager) sẽ biến mất khỏi kết quả — một bug rất hay gặp trong query cây tổ chức.

**Ứng dụng khác**: tìm cặp trùng lặp, so sánh dòng với dòng liền kề, tìm khoảng trống trong dãy số.

---

## NATURAL JOIN — nên tránh

`NATURAL JOIN` tự động join trên **mọi cột trùng tên** ở hai bảng, không cần viết `ON`.

```sql
SELECT * FROM emp NATURAL JOIN dept;   -- tự join trên cột `name` (!!)
```

Trong ví dụ của chúng ta, cả `emp` và `dept` đều có cột `name` → MySQL sẽ join trên `emp.name = dept.name`, hoàn toàn không phải ý định của ai cả.

Tệ hơn: **thêm một cột mới vào bảng có thể âm thầm đổi ngữ nghĩa của query cũ**. Hôm nay bạn `ALTER TABLE dept ADD created_at DATETIME`, ngày mai `NATURAL JOIN` bắt đầu join thêm cả `created_at` và trả về 0 dòng. Không có lỗi, không có cảnh báo.

👉 **Luôn viết `ON` tường minh.** Lợi ích của `NATURAL JOIN` là tiết kiệm 20 ký tự; cái giá là một class bug không thể grep ra được.

---

## SEMI JOIN và ANTI JOIN

Hai khái niệm này **không có từ khoá riêng** trong SQL, nhưng là cách optimizer *nghĩ* — và bạn sẽ thấy chúng trong `EXPLAIN`.

### SEMI JOIN — "có tồn tại không?"

Trả về dòng bảng trái **có ít nhất một** dòng khớp bên phải, nhưng **không nhân dòng** và không lấy cột nào bên phải.

```sql
-- Khách hàng có ít nhất 1 đơn đã thanh toán
SELECT c.name FROM customers c
WHERE c.id IN (SELECT o.customer_id FROM orders o WHERE o.status = 'paid');

-- hoặc, tương đương về ngữ nghĩa:
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'paid');
```

MySQL 8 tự nhận ra đây là semi-join và có 4 chiến lược để thực thi (`FirstMatch`, `LooseScan`, `Materialization`, `DuplicateWeedout`). Với query trên, nó chọn **Materialization**:

```
-> Nested loop inner join
    -> Table scan on c  (rows=1000)
    -> Single-row index lookup on <subquery2> using <auto_distinct_key> (customer_id=c.id)
        -> Materialize with deduplication  (rows=2500)
            -> Index lookup on o using idx_orders_status (status='paid')  (rows=2500)
```

Nó gom sẵn danh sách `customer_id` đã khử trùng lặp vào một bảng tạm có key, rồi tra bảng tạm đó — thay vì chạy subquery 1.000 lần.

**❌ Tại sao KHÔNG nên viết bằng `JOIN` thường**:

```sql
-- SAI nếu bạn không để ý: khách có 5 đơn 'paid' sẽ xuất hiện 5 lần
SELECT c.name FROM customers c JOIN orders o ON o.customer_id = c.id WHERE o.status='paid';
```

Phải thêm `DISTINCT` để sửa — mà `DISTINCT` thì tốn thêm một bước sort/hash trên toàn bộ kết quả. `EXISTS` diễn đạt đúng ý định hơn *và* nhanh hơn.

### ANTI JOIN — "KHÔNG tồn tại"

Trả về dòng bảng trái **không có** dòng khớp nào bên phải. Có hai cách viết:

```sql
-- Cách 1: LEFT JOIN ... IS NULL
SELECT c.id FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- Cách 2: NOT EXISTS
SELECT c.id FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

Cả hai đều được MySQL 8 nhận diện thành `Nested loop antijoin`. Nhưng chi phí ước tính khác nhau rõ rệt:

| Cách viết | Cost (đo thật) |
|---|---|
| `LEFT JOIN ... IS NULL` | **13009** |
| `NOT EXISTS` | **1620** |

`NOT EXISTS` rẻ hơn ~8× vì optimizer biết nó được phép **dừng ngay khi tìm thấy dòng khớp đầu tiên** (short-circuit), trong khi `LEFT JOIN` phải sinh đủ mọi cặp khớp rồi mới lọc `IS NULL` ở bước sau.

> 🚫 **Tuyệt đối tránh `NOT IN (subquery)`** khi cột trong subquery có thể `NULL`. Theo chuẩn SQL ba giá trị, `x NOT IN (1, 2, NULL)` cho ra `UNKNOWN` chứ không phải `TRUE` — nên query trả về **0 dòng**, im lặng, không báo lỗi. `NOT EXISTS` không có vấn đề này. Đây là một trong những bug SQL khó chịu nhất vì nó không crash, chỉ trả sai.

---

## Bảng tổng hợp các loại JOIN

| Loại | Giữ dòng mồ côi bên trái? | Giữ bên phải? | MySQL hỗ trợ | Dùng khi |
|---|:---:|:---:|:---:|---|
| `INNER JOIN` | ✗ | ✗ | ✓ | Mặc định — cần quan hệ đầy đủ hai bên |
| `LEFT JOIN` | ✓ (NULL) | ✗ | ✓ | Bên phải có thể không tồn tại |
| `RIGHT JOIN` | ✗ | ✓ (NULL) | ✓ | Nên viết lại thành LEFT JOIN |
| `FULL OUTER JOIN` | ✓ | ✓ | ✗ | Giả lập bằng `UNION` |
| `CROSS JOIN` | — (tích Descartes) | — | ✓ | Sinh khung tổ hợp cho báo cáo |
| `SELF JOIN` | tuỳ loại JOIN dùng | | ✓ | Quan hệ phân cấp trong một bảng |
| `NATURAL JOIN` | ✗ | ✗ | ✓ | **Đừng dùng** |
| SEMI JOIN (`EXISTS`) | ✗ | không lấy cột | ✓ (tự động) | Kiểm tra tồn tại, không nhân dòng |
| ANTI JOIN (`NOT EXISTS`) | chỉ dòng không khớp | không lấy cột | ✓ (tự động) | Tìm dòng thiếu quan hệ |

---

## Hai cái bẫy kinh điển

### Bẫy 1 — `ON` và `WHERE` không giống nhau trong LEFT JOIN

Với `INNER JOIN`, đặt điều kiện ở `ON` hay `WHERE` cho kết quả **y hệt**. Với `LEFT JOIN` thì **khác hẳn**:

```sql
-- (A) điều kiện trong ON: lọc bảng PHẢI trước khi ghép
SELECT e.name, d.name FROM emp e
LEFT JOIN dept d ON d.id = e.dept_id AND d.name = 'Engineering';
```
```
+-------+-------------+
| emp   | dept        |
+-------+-------------+
| An    | Engineering |
| Binh  | Engineering |
| Cuong | NULL        |   <-- vẫn giữ, chỉ là không khớp điều kiện
| Dung  | NULL        |
+-------+-------------+   4 dòng
```

```sql
-- (B) điều kiện trong WHERE: lọc SAU khi đã ghép
SELECT e.name, d.name FROM emp e
LEFT JOIN dept d ON d.id = e.dept_id
WHERE d.name = 'Engineering';
```
```
+------+-------------+
| emp  | dept        |
+------+-------------+
| An   | Engineering |
| Binh | Engineering |
+------+-------------+   2 dòng
```

Vì `WHERE d.name = 'Engineering'` loại bỏ mọi dòng có `d.name IS NULL`, nó **huỷ luôn tác dụng của LEFT JOIN**. MySQL biết điều đó và tự viết lại thành INNER JOIN — nhìn thấy rõ trong `EXPLAIN`:

```
-> Inner hash join (e.dept_id = d.id)      <-- viết LEFT JOIN, thực thi INNER JOIN!
    -> Table scan on e
    -> Hash
        -> Filter: (d.name = 'Engineering')
            -> Table scan on d
```

👉 **Quy tắc**: điều kiện lọc **bảng phải** của LEFT JOIN → đặt trong `ON`. Điều kiện lọc **bảng trái** → đặt trong `WHERE`. Ngoại lệ duy nhất dùng `WHERE` trên cột bảng phải là `WHERE right.col IS NULL` — đó là anti-join có chủ đích.

### Bẫy 2 — JOIN nhân dòng, làm hỏng `SUM`/`COUNT`

```sql
-- Mỗi đơn có 3 order_items => total_amount bị cộng 3 lần!
SELECT c.name, SUM(o.total_amount)
FROM customers c
JOIN orders o      ON o.customer_id = c.id
JOIN order_items i ON i.order_id    = o.id
GROUP BY c.name;
```

JOIN quan hệ 1-n làm dòng bên trái bị nhân lên theo số dòng khớp bên phải. Mọi hàm tổng hợp sau đó đều sai. Ba cách sửa:

```sql
-- Cách 1: gộp trước rồi mới join (tốt nhất khi cần nhiều bảng chi tiết)
SELECT c.name, agg.revenue
FROM customers c
JOIN (SELECT customer_id, SUM(total_amount) AS revenue
      FROM orders GROUP BY customer_id) agg
  ON agg.customer_id = c.id;

-- Cách 2: dùng subquery tương quan trong SELECT (đơn giản, ít bảng)
SELECT c.name,
       (SELECT SUM(o.total_amount) FROM orders o WHERE o.customer_id = c.id) AS revenue
FROM customers c;

-- Cách 3: SUM(DISTINCT ...) - chỉ đúng khi giá trị đảm bảo không trùng, RẤT DỄ SAI
```

---

# Phần 2 — Thuật toán JOIN

Ở đây SQL kết thúc và hệ quản trị bắt đầu. `INNER JOIN` chỉ nói **cần gì**; thuật toán JOIN nói **làm thế nào**. Có ba thuật toán kinh điển trong lý thuyết CSDL:

| Thuật toán | MySQL 8.0 | PostgreSQL | Oracle | SQL Server |
|---|:---:|:---:|:---:|:---:|
| Nested Loop Join | ✓ | ✓ | ✓ | ✓ |
| Hash Join | ✓ (từ 8.0.18) | ✓ | ✓ | ✓ |
| Sort-Merge Join | **✗** | ✓ | ✓ | ✓ |

**MySQL không có Sort-Merge Join.** Đây là khác biệt kiến trúc quan trọng cần nhớ khi chuyển query từ PostgreSQL sang, hoặc khi đọc sách CSDL rồi tìm mãi không thấy trong `EXPLAIN` của MySQL. Vẫn sẽ trình bày Sort-Merge ở dưới vì hiểu nó giúp hiểu vì sao MySQL chọn hai cái còn lại.

---

## Nested Loop Join (NLJ)

Thuật toán đơn giản nhất, và là thuật toán **mặc định, phổ biến nhất** trong MySQL. Ý tưởng đúng như tên gọi: hai vòng lặp lồng nhau.

```
for mỗi dòng r trong bảng NGOÀI (driving / outer table):
    for mỗi dòng s trong bảng TRONG (driven / inner table):
        if r khớp s:  xuất ra (r, s)
```

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="nl-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="nl-arrT" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="nl-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="560" fill="#FFFDF7" rx="16"/>
<rect x="20" y="18" width="860" height="242" rx="14" fill="#FFF7F7" stroke="#FFC9CC" stroke-width="1.5"/>
<rect x="20" y="278" width="860" height="262" rx="14" fill="#F2FCFB" stroke="#9BE5DC" stroke-width="1.5"/>
<text x="42" y="44" font-size="14" font-weight="800" fill="#9F1239">① Simple Nested Loop — KHÔNG có index trên cột join</text>
<text x="42" y="304" font-size="14" font-weight="800" fill="#134E4A">② Index Nested Loop — CÓ index trên cột join của bảng trong</text>
<rect x="50" y="62" width="150" height="178" rx="10" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="2" filter="url(#nl-sh)"/>
<rect x="50" y="62" width="150" height="28" rx="10" fill="#FF6B6B"/>
<rect x="50" y="80" width="150" height="10" fill="#FF6B6B"/>
<text x="125" y="81" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Bảng NGOÀI (M dòng)</text>
<rect x="64" y="100" width="122" height="26" rx="5" fill="#FFE9EC"/>
<rect x="64" y="134" width="122" height="26" rx="5" fill="#FFE9EC"/>
<rect x="64" y="168" width="122" height="26" rx="5" fill="#FFE9EC"/>
<rect x="64" y="202" width="122" height="26" rx="5" fill="#F5F5F4"/>
<text x="125" y="118" text-anchor="middle" font-size="11" fill="#9F1239">dòng 1</text>
<text x="125" y="152" text-anchor="middle" font-size="11" fill="#9F1239">dòng 2</text>
<text x="125" y="186" text-anchor="middle" font-size="11" fill="#9F1239">dòng 3</text>
<text x="125" y="220" text-anchor="middle" font-size="11" fill="#A8A29E">... M</text>
<rect x="360" y="96" width="210" height="34" rx="7" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<rect x="360" y="130" width="210" height="34" rx="7" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<rect x="360" y="164" width="210" height="34" rx="7" fill="#FFFFFF" stroke="#FF6B6B" stroke-width="1.8"/>
<rect x="360" y="198" width="210" height="34" rx="7" fill="#FAFAF9" stroke="#D6D3D1" stroke-width="1.5" stroke-dasharray="4 3"/>
<text x="465" y="118" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">QUÉT TOÀN BỘ bảng trong (N dòng)</text>
<text x="465" y="152" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">QUÉT TOÀN BỘ bảng trong (N dòng)</text>
<text x="465" y="186" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">QUÉT TOÀN BỘ bảng trong (N dòng)</text>
<text x="465" y="220" text-anchor="middle" font-size="11" fill="#A8A29E">... lặp lại M lần</text>
<line x1="202" y1="113" x2="356" y2="113" stroke="#57534E" stroke-width="1.6" marker-end="url(#nl-arr)"/>
<line x1="202" y1="147" x2="356" y2="147" stroke="#57534E" stroke-width="1.6" marker-end="url(#nl-arr)"/>
<line x1="202" y1="181" x2="356" y2="181" stroke="#57534E" stroke-width="1.6" marker-end="url(#nl-arr)"/>
<rect x="612" y="110" width="238" height="108" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="731" y="136" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Chi phí</text>
<text x="731" y="160" text-anchor="middle" font-size="15" font-weight="800" fill="#BE123C">M × N</text>
<text x="731" y="182" text-anchor="middle" font-size="11" fill="#9F1239">phép so sánh — O(M·N)</text>
<text x="731" y="202" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">1.000 × 10.000 = 10 triệu</text>
<rect x="50" y="322" width="150" height="178" rx="10" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#nl-sh)"/>
<rect x="50" y="322" width="150" height="28" rx="10" fill="#0D9488"/>
<rect x="50" y="340" width="150" height="10" fill="#0D9488"/>
<text x="125" y="341" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">Bảng NGOÀI (M dòng)</text>
<rect x="64" y="360" width="122" height="26" rx="5" fill="#E6FBF9"/>
<rect x="64" y="394" width="122" height="26" rx="5" fill="#E6FBF9"/>
<rect x="64" y="428" width="122" height="26" rx="5" fill="#E6FBF9"/>
<rect x="64" y="462" width="122" height="26" rx="5" fill="#F5F5F4"/>
<text x="125" y="378" text-anchor="middle" font-size="11" fill="#134E4A">dòng 1</text>
<text x="125" y="412" text-anchor="middle" font-size="11" fill="#134E4A">dòng 2</text>
<text x="125" y="446" text-anchor="middle" font-size="11" fill="#134E4A">dòng 3</text>
<text x="125" y="480" text-anchor="middle" font-size="11" fill="#A8A29E">... M</text>
<path d="M330,340 L470,340 L400,380 Z" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.6"/>
<path d="M330,340 L470,340 L400,380 Z" fill="none" stroke="#0D9488" stroke-width="1.6"/>
<line x1="365" y1="360" x2="400" y2="400" stroke="#0D9488" stroke-width="1.4"/>
<line x1="435" y1="360" x2="400" y2="400" stroke="#0D9488" stroke-width="1.4"/>
<rect x="356" y="398" width="88" height="22" rx="5" fill="#0D9488"/>
<text x="400" y="356" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">B-TREE INDEX</text>
<text x="400" y="413" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">lá: dòng khớp</text>
<line x1="202" y1="373" x2="326" y2="352" stroke="#0D9488" stroke-width="1.6" marker-end="url(#nl-arrT)"/>
<line x1="202" y1="407" x2="326" y2="356" stroke="#0D9488" stroke-width="1.6" marker-end="url(#nl-arrT)"/>
<line x1="202" y1="441" x2="326" y2="360" stroke="#0D9488" stroke-width="1.6" marker-end="url(#nl-arrT)"/>
<text x="265" y="470" text-anchor="middle" font-size="10" fill="#0F766E">mỗi dòng ngoài = 1 lần tra index</text>
<rect x="500" y="352" width="120" height="116" rx="10" fill="#FFFFFF" stroke="#14B8A6" stroke-width="1.8"/>
<text x="560" y="376" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Bảng TRONG</text>
<text x="560" y="394" text-anchor="middle" font-size="11" fill="#0F766E">(N dòng)</text>
<rect x="516" y="406" width="88" height="18" rx="4" fill="#CCFBF1"/>
<rect x="516" y="430" width="88" height="18" rx="4" fill="#F5F5F4"/>
<text x="560" y="419" text-anchor="middle" font-size="9" fill="#134E4A">chỉ đọc dòng cần</text>
<text x="560" y="443" text-anchor="middle" font-size="9" fill="#A8A29E">phần còn lại không đụng</text>
<rect x="640" y="352" width="210" height="116" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="745" y="378" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">Chi phí</text>
<text x="745" y="402" text-anchor="middle" font-size="15" font-weight="800" fill="#0F766E">M × log N</text>
<text x="745" y="424" text-anchor="middle" font-size="11" fill="#134E4A">O(M·log N) — rẻ hơn rất nhiều</text>
<text x="745" y="444" text-anchor="middle" font-size="11" font-weight="700" fill="#0F766E">1.000 × ~13 ≈ 13.000</text>
<text x="450" y="522" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">👉 Cùng một thuật toán. Khác biệt duy nhất là có index hay không — và đó là khác biệt 770 lần.</text>
</svg>
</div>

### Ba biến thể

**1. Simple Nested Loop** — không index, quét toàn bảng trong cho mỗi dòng bảng ngoài. `O(M × N)`. Thảm hoạ.

**2. Index Nested Loop (INLJ)** — bảng trong có index trên cột join, mỗi dòng ngoài chỉ cần một lần tra B-tree. `O(M × log N)`. Đây là dạng bạn muốn thấy, và là dạng MySQL chọn trong đại đa số trường hợp.

```sql
EXPLAIN SELECT c.name, o.id, o.total_amount
FROM customers c JOIN orders o ON o.customer_id = c.id
WHERE c.city = 'Hanoi';
```

```
+----+-------+------+---------------------+---------------------+---------+------------------+------+
| id | table | type | possible_keys       | key                 | key_len | ref              | rows |
+----+-------+------+---------------------+---------------------+---------+------------------+------+
|  1 | c     | ref  | PRIMARY,idx_..._city| idx_customers_city  | 202     | const            |  200 |
|  1 | o     | ref  | idx_orders_customer | idx_orders_customer | 4       | join_demo.c.id   |   12 |
+----+-------+------+---------------------+---------------------+---------+------------------+------+
```

Đọc dạng cây cho rõ hơn:

```
-> Nested loop inner join  (cost=876 rows=2435)
    -> Index lookup on c using idx_customers_city (city='Hanoi')  (cost=23.8 rows=200)
    -> Index lookup on o using idx_orders_customer (customer_id=c.id)  (cost=3.05 rows=12.2)
```

Đọc từ trong ra: lấy 200 khách ở Hà Nội qua index `city`, rồi với **mỗi** khách đó tra `idx_orders_customer` để lấy ~12 đơn. Tổng cộng 200 lần tra index — không có lần quét toàn bảng nào.

**3. Block Nested Loop (BNL)** — biến thể cũ: gom nhiều dòng bảng ngoài vào `join_buffer` rồi quét bảng trong **một lần cho cả lô**, giảm số lần quét từ `M` xuống `M / số dòng vừa buffer`. **Đã bị gỡ bỏ hoàn toàn từ MySQL 8.0.20** và thay bằng hash join. Nếu bạn thấy `Using join buffer (Block Nested Loop)` trong `EXPLAIN`, bạn đang chạy MySQL ≤ 8.0.19.

### Ưu / nhược điểm

| | Nested Loop |
|---|---|
| ✅ **Ưu** | Không tốn bộ nhớ phụ (streaming — trả dòng đầu tiên ngay lập tức) |
| | Dùng được với **mọi** điều kiện join: `=`, `<`, `>`, `BETWEEN`, `LIKE`, hàm... |
| | Cực nhanh khi bảng ngoài nhỏ + bảng trong có index (`O(M·log N)`) |
| | Hợp với OLTP: query trả vài chục dòng, có `LIMIT`, cần latency thấp |
| ❌ **Nhược** | Không index → `O(M·N)`, chậm theo cấp số nhân |
| | Random I/O: mỗi lần tra index là một lần nhảy ngẫu nhiên trên đĩa/buffer pool |
| | Kém khi cả hai bảng đều lớn và cần join phần lớn dữ liệu (OLAP) |

**👉 Dùng khi**: bảng ngoài đã được lọc còn ít dòng, bảng trong có index tốt. Đây là 90% query OLTP.

---

## Hash Join

Có từ **MySQL 8.0.18**, mở rộng ở **8.0.20**. Ý tưởng: thay vì tìm kiếm lặp đi lặp lại, **xây một bảng băm trong RAM** rồi tra một lần.

Hai pha:

1. **Build phase** — đọc bảng **nhỏ hơn** (build input), băm cột join của từng dòng, nhét vào hash table trong `join_buffer`.
2. **Probe phase** — đọc bảng **lớn hơn** (probe input) theo kiểu streaming, băm cột join của từng dòng, tra thẳng vào hash table. Trúng thì xuất ra.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 520" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="hj-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<marker id="hj-arrA" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#D97706"/></marker>
<filter id="hj-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="520" fill="#FFFDF7" rx="16"/>
<text x="450" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hash Join — Build &amp; Probe</text>
<rect x="24" y="48" width="410" height="228" rx="14" fill="#F2FCFB" stroke="#9BE5DC" stroke-width="1.5"/>
<rect x="24" y="292" width="852" height="150" rx="14" fill="#FFFAF0" stroke="#F8D48A" stroke-width="1.5"/>
<text x="44" y="72" font-size="13" font-weight="800" fill="#134E4A">① BUILD — đọc bảng NHỎ, dựng hash table trong RAM</text>
<text x="44" y="316" font-size="13" font-weight="800" fill="#7C4A03">② PROBE — bảng LỚN chảy qua, mỗi dòng tra hash table đúng 1 lần</text>
<rect x="46" y="86" width="120" height="172" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2" filter="url(#hj-sh)"/>
<rect x="46" y="86" width="120" height="26" rx="9" fill="#0D9488"/>
<rect x="46" y="102" width="120" height="10" fill="#0D9488"/>
<text x="106" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">customers (nhỏ)</text>
<rect x="58" y="122" width="96" height="24" rx="5" fill="#E6FBF9"/>
<rect x="58" y="152" width="96" height="24" rx="5" fill="#E6FBF9"/>
<rect x="58" y="182" width="96" height="24" rx="5" fill="#E6FBF9"/>
<rect x="58" y="212" width="96" height="24" rx="5" fill="#F5F5F4"/>
<text x="106" y="138" text-anchor="middle" font-size="10" fill="#134E4A">id=1</text>
<text x="106" y="168" text-anchor="middle" font-size="10" fill="#134E4A">id=2</text>
<text x="106" y="198" text-anchor="middle" font-size="10" fill="#134E4A">id=3</text>
<text x="106" y="228" text-anchor="middle" font-size="10" fill="#A8A29E">...</text>
<rect x="192" y="148" width="58" height="48" rx="8" fill="#0D9488"/>
<text x="221" y="168" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">hash()</text>
<text x="221" y="184" text-anchor="middle" font-size="9" fill="#CCFBF1">băm key</text>
<line x1="168" y1="172" x2="188" y2="172" stroke="#0D9488" stroke-width="1.8" marker-end="url(#hj-arr)"/>
<rect x="276" y="86" width="142" height="172" rx="9" fill="#FFFFFF" stroke="#0D9488" stroke-width="2.5" filter="url(#hj-sh)"/>
<rect x="276" y="86" width="142" height="26" rx="9" fill="#134E4A"/>
<rect x="276" y="102" width="142" height="10" fill="#134E4A"/>
<text x="347" y="104" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">HASH TABLE (RAM)</text>
<rect x="288" y="120" width="118" height="22" rx="4" fill="#CCFBF1"/>
<rect x="288" y="146" width="118" height="22" rx="4" fill="#CCFBF1"/>
<rect x="288" y="172" width="118" height="22" rx="4" fill="#CCFBF1"/>
<rect x="288" y="198" width="118" height="22" rx="4" fill="#CCFBF1"/>
<rect x="288" y="224" width="118" height="22" rx="4" fill="#F5F5F4"/>
<text x="296" y="135" font-size="9" fill="#0F766E">bucket 0 → [id=3]</text>
<text x="296" y="161" font-size="9" fill="#0F766E">bucket 1 → [id=1]</text>
<text x="296" y="187" font-size="9" fill="#0F766E">bucket 2 → [ ]</text>
<text x="296" y="213" font-size="9" fill="#0F766E">bucket 3 → [id=2]</text>
<text x="296" y="239" font-size="9" fill="#A8A29E">...</text>
<line x1="252" y1="172" x2="272" y2="172" stroke="#0D9488" stroke-width="1.8" marker-end="url(#hj-arr)"/>
<rect x="458" y="66" width="418" height="210" rx="12" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="478" y="92" font-size="12" font-weight="800" fill="#374151">Vì sao nhanh?</text>
<text x="478" y="116" font-size="11" fill="#57534E">• Mỗi bảng chỉ đọc <tspan font-weight="700">đúng 1 lần</tspan> — không quét lặp.</text>
<text x="478" y="140" font-size="11" fill="#57534E">• Tra hash table là <tspan font-weight="700">O(1)</tspan>, không phải O(log N).</text>
<text x="478" y="164" font-size="11" fill="#57534E">• Sequential I/O, thân thiện với cache CPU.</text>
<text x="478" y="188" font-size="11" fill="#57534E">• Tổng chi phí: <tspan font-weight="700" fill="#0F766E">O(M + N)</tspan> thay vì O(M × N).</text>
<text x="478" y="218" font-size="12" font-weight="800" fill="#9F1239">Đánh đổi</text>
<text x="478" y="240" font-size="11" fill="#9F1239">• Cần RAM (join_buffer_size) cho hash table.</text>
<text x="478" y="260" font-size="11" fill="#9F1239">• Bắt buộc có điều kiện <tspan font-family="monospace" font-weight="700">=</tspan> (equi-join).</text>
<rect x="46" y="332" width="120" height="92" rx="9" fill="#FFFFFF" stroke="#F5A623" stroke-width="2" filter="url(#hj-sh)"/>
<rect x="46" y="332" width="120" height="24" rx="9" fill="#F5A623"/>
<rect x="46" y="346" width="120" height="10" fill="#F5A623"/>
<text x="106" y="349" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">orders (lớn)</text>
<text x="106" y="380" text-anchor="middle" font-size="10" fill="#92400E">10.000 dòng</text>
<text x="106" y="400" text-anchor="middle" font-size="10" fill="#92400E">đọc streaming</text>
<text x="106" y="416" text-anchor="middle" font-size="10" fill="#A8A29E">không cần vào RAM</text>
<rect x="212" y="352" width="58" height="48" rx="8" fill="#D97706"/>
<text x="241" y="372" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">hash()</text>
<text x="241" y="388" text-anchor="middle" font-size="9" fill="#FFF3DB">cùng hàm</text>
<line x1="168" y1="376" x2="208" y2="376" stroke="#D97706" stroke-width="1.8" marker-end="url(#hj-arrA)"/>
<rect x="316" y="340" width="150" height="72" rx="9" fill="#FFFFFF" stroke="#D97706" stroke-width="2"/>
<text x="391" y="362" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Tra bucket</text>
<text x="391" y="382" text-anchor="middle" font-size="10" fill="#92400E">O(1) — 1 phép tính</text>
<text x="391" y="400" text-anchor="middle" font-size="10" fill="#92400E">so khớp key thật</text>
<line x1="272" y1="376" x2="312" y2="376" stroke="#D97706" stroke-width="1.8" marker-end="url(#hj-arrA)"/>
<line x1="391" y1="336" x2="391" y2="262" stroke="#0D9488" stroke-width="1.8" stroke-dasharray="5 4" marker-end="url(#hj-arr)"/>
<rect x="512" y="340" width="150" height="72" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="587" y="366" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">Trúng → xuất</text>
<text x="587" y="388" text-anchor="middle" font-size="10" fill="#0F766E">(customer, order)</text>
<line x1="470" y1="376" x2="508" y2="376" stroke="#0D9488" stroke-width="1.8" marker-end="url(#hj-arr)"/>
<rect x="700" y="340" width="156" height="72" rx="9" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2"/>
<text x="778" y="362" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Tràn RAM?</text>
<text x="778" y="382" text-anchor="middle" font-size="10" fill="#BE123C">chia chunk ra đĩa tmpdir</text>
<text x="778" y="400" text-anchor="middle" font-size="10" fill="#BE123C">(on-disk hash join)</text>
<line x1="666" y1="376" x2="696" y2="376" stroke="#FF6B6B" stroke-width="1.8" stroke-dasharray="4 3"/>
<text x="450" y="472" text-anchor="middle" font-size="12" font-weight="700" fill="#374151">Đo thật trên 1.000 × 10.000 dòng, không index: Hash Join 3,25 ms — Nested Loop 515 ms</text>
<text x="450" y="496" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">nhanh hơn ~158 lần</text>
</svg>
</div>

### Nhận biết trong EXPLAIN

```sql
EXPLAIN FORMAT=TREE
SELECT COUNT(*) FROM orders o JOIN order_items i ON i.price = o.total_amount;
```

```
-> Aggregate: count(0)
    -> Inner hash join (i.price = o.total_amount)     <-- đây
        -> Table scan on i  (rows=30336)
        -> Hash                                       <-- bảng nhỏ được đưa vào hash table
            -> Table scan on o  (rows=9740)
```

Trong `EXPLAIN` cổ điển, dấu hiệu nằm ở cột `Extra`:

```
Extra: Using where; Using join buffer (hash join)
```

### Khi nào MySQL chọn hash join

- Có ít nhất **một điều kiện equi-join** (`=`) giữa hai bảng, **và**
- **không có index dùng được** cho điều kiện join đó.

Nếu có index phù hợp, optimizer gần như luôn chọn Index Nested Loop vì rẻ hơn. Nói cách khác: **hash join là lưới an toàn, không phải mục tiêu**. Thấy hash join trên bảng lớn trong query OLTP = dấu hiệu bạn đang thiếu index.

Từ 8.0.20, hash join còn dùng được cho outer join, semi-join, anti-join. Với **non-equi join** thì nó vẫn xuất hiện nhưng ở dạng thoái hoá — hash join không điều kiện (tích Descartes) rồi lọc sau:

```sql
EXPLAIN FORMAT=TREE SELECT COUNT(*) FROM orders o JOIN order_items i ON i.price < o.total_amount;
```
```
-> Filter: (i.price < o.total_amount)  (cost=29.5e+6 rows=98.5e+6)
    -> Inner hash join (no condition)  (cost=29.5e+6 rows=98.5e+6)
```

`(no condition)` + 98,5 triệu dòng ước tính = đỏ rực. Hash join **không cứu được** non-equi join; cấu trúc băm chỉ trả lời được "bằng nhau hay không", không trả lời được "lớn hơn hay không".

### Khi hash table không vừa RAM

Nếu build input lớn hơn `join_buffer_size` (mặc định **256 KB** — khá nhỏ), MySQL chuyển sang **on-disk hash join** (biến thể Grace Hash Join): băm cả hai bảng thành các chunk file trên `tmpdir`, rồi join từng cặp chunk một. Vẫn tốt hơn nested loop nhiều, nhưng đã phải chạm đĩa.

```sql
SHOW VARIABLES LIKE 'join_buffer_size';   -- 262144 (256 KB)
SET SESSION join_buffer_size = 8 * 1024 * 1024;   -- nâng cho query phân tích nặng
```

⚠️ `join_buffer_size` được cấp phát **cho mỗi lần join, cho mỗi connection**. Đặt 256 MB ở global với 200 connection = 51 GB RAM. Luôn chỉnh ở mức **session**, ngay trước query nặng.

### Ưu / nhược điểm

| | Hash Join |
|---|---|
| ✅ **Ưu** | `O(M + N)` — mỗi bảng đọc đúng một lần |
| | Không cần index → cứu được query join trên cột không index |
| | Sequential I/O, tận dụng tốt băng thông đĩa |
| | Lý tưởng cho OLAP / báo cáo / ETL join bảng lớn |
| ❌ **Nhược** | **Bắt buộc equi-join** (`=`), vô dụng với `<`, `>`, `BETWEEN` |
| | Tốn RAM; tràn thì phải ghi chunk ra đĩa |
| | **Blocking**: phải build xong toàn bộ hash table mới ra được dòng đầu tiên → xấu với `LIMIT 10` |
| | Không tận dụng được thứ tự sẵn có; kết quả ra không có thứ tự |

**👉 Dùng khi**: join hai bảng lớn, lấy phần lớn dữ liệu, điều kiện `=`, không có/không đáng đánh index. Tức là: báo cáo và phân tích.

---

## Sort-Merge Join — MySQL không có

Thuật toán thứ ba trong lý thuyết. Ý tưởng: **sắp xếp cả hai bảng theo cột join, rồi duyệt song song bằng hai con trỏ** — hệt như bước merge trong merge sort.

<div class="zoomable-diagram" style="max-width:860px;margin:2rem auto;">
<svg viewBox="0 0 860 430" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="sm-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#6366F1"/></marker>
<filter id="sm-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="860" height="430" fill="#FFFDF7" rx="16"/>
<text x="430" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Sort-Merge Join — hai con trỏ chạy song song</text>
<text x="430" y="52" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">⚠ MySQL KHÔNG cài đặt thuật toán này — có trong PostgreSQL / Oracle / SQL Server</text>
<rect x="60" y="74" width="180" height="26" rx="7" fill="#6366F1"/>
<rect x="60" y="100" width="180" height="222" fill="#FFFFFF" stroke="#6366F1" stroke-width="2"/>
<rect x="400" y="74" width="180" height="26" rx="7" fill="#8B5CF6"/>
<rect x="400" y="100" width="180" height="222" fill="#FFFFFF" stroke="#8B5CF6" stroke-width="2"/>
<text x="150" y="93" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">R đã SORT theo key</text>
<text x="490" y="93" text-anchor="middle" font-size="12" font-weight="700" fill="#FFFFFF">S đã SORT theo key</text>
<rect x="74" y="112" width="152" height="28" rx="5" fill="#EEF2FF"/>
<rect x="74" y="148" width="152" height="28" rx="5" fill="#C7D2FE"/>
<rect x="74" y="184" width="152" height="28" rx="5" fill="#EEF2FF"/>
<rect x="74" y="220" width="152" height="28" rx="5" fill="#EEF2FF"/>
<rect x="74" y="256" width="152" height="28" rx="5" fill="#F5F5F4"/>
<rect x="414" y="112" width="152" height="28" rx="5" fill="#F5F3FF"/>
<rect x="414" y="148" width="152" height="28" rx="5" fill="#DDD6FE"/>
<rect x="414" y="184" width="152" height="28" rx="5" fill="#DDD6FE"/>
<rect x="414" y="220" width="152" height="28" rx="5" fill="#F5F3FF"/>
<rect x="414" y="256" width="152" height="28" rx="5" fill="#F5F5F4"/>
<text x="150" y="131" text-anchor="middle" font-size="11" fill="#3730A3">key = 1</text>
<text x="150" y="167" text-anchor="middle" font-size="11" font-weight="800" fill="#3730A3">key = 3  ◀ con trỏ R</text>
<text x="150" y="203" text-anchor="middle" font-size="11" fill="#3730A3">key = 7</text>
<text x="150" y="239" text-anchor="middle" font-size="11" fill="#3730A3">key = 9</text>
<text x="150" y="275" text-anchor="middle" font-size="11" fill="#A8A29E">...</text>
<text x="490" y="131" text-anchor="middle" font-size="11" fill="#5B21B6">key = 2</text>
<text x="490" y="167" text-anchor="middle" font-size="11" font-weight="800" fill="#5B21B6">key = 3  ◀ con trỏ S</text>
<text x="490" y="203" text-anchor="middle" font-size="11" font-weight="700" fill="#5B21B6">key = 3</text>
<text x="490" y="239" text-anchor="middle" font-size="11" fill="#5B21B6">key = 8</text>
<text x="490" y="275" text-anchor="middle" font-size="11" fill="#A8A29E">...</text>
<line x1="230" y1="162" x2="408" y2="162" stroke="#6366F1" stroke-width="2" marker-end="url(#sm-arr)"/>
<text x="319" y="155" text-anchor="middle" font-size="10" font-weight="700" fill="#4338CA">3 = 3 → khớp!</text>
<line x1="230" y1="180" x2="408" y2="198" stroke="#8B5CF6" stroke-width="1.6" stroke-dasharray="4 3" marker-end="url(#sm-arr)"/>
<text x="319" y="212" text-anchor="middle" font-size="10" fill="#6D28D9">rồi tiến con trỏ bên nhỏ hơn</text>
<rect x="612" y="106" width="212" height="216" rx="12" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="630" y="132" font-size="12" font-weight="800" fill="#374151">Đặc tính</text>
<text x="630" y="156" font-size="11" fill="#0F766E">✅ Dùng được với &lt;, &gt;, BETWEEN</text>
<text x="630" y="176" font-size="11" fill="#0F766E">✅ Kết quả ra đã sẵn thứ tự</text>
<text x="630" y="196" font-size="11" fill="#0F766E">✅ Rất rẻ nếu index đã sort sẵn</text>
<text x="630" y="216" font-size="11" fill="#0F766E">✅ Bộ nhớ ít hơn hash join</text>
<text x="630" y="242" font-size="11" fill="#9F1239">❌ Phải sort trước: O(n log n)</text>
<text x="630" y="262" font-size="11" fill="#9F1239">❌ Blocking — sort xong mới ra</text>
<text x="630" y="282" font-size="11" fill="#9F1239">❌ Kém nếu key trùng nhiều</text>
<text x="630" y="306" font-size="11" font-weight="700" fill="#57534E">Chi phí: O(M log M + N log N)</text>
<rect x="60" y="340" width="764" height="70" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="80" y="363" font-size="12" font-weight="800" fill="#7C4A03">Vì sao MySQL bỏ qua thuật toán này?</text>
<text x="80" y="383" font-size="11" fill="#92400E">InnoDB lưu bảng theo clustered index → rất nhiều truy cập vốn đã có sẵn thứ tự khoá chính.</text>
<text x="80" y="400" font-size="11" fill="#92400E">Với equi-join thì hash join rẻ hơn; với non-equi join thì index nested loop đã đủ cho workload OLTP.</text>
</svg>
</div>

Cách hoạt động:

```
i, j = 0, 0
while i < len(R) and j < len(S):
    if   R[i].key < S[j].key:  i += 1          # tiến bên nhỏ hơn
    elif R[i].key > S[j].key:  j += 1
    else:                                       # khớp
        xuất mọi tổ hợp của nhóm key bằng nhau ở hai bên
        tiến cả hai
```

**Vì sao MySQL không cài đặt nó?** Vài lý do thực dụng:
- InnoDB lưu bảng theo **clustered index** — dữ liệu đã sắp sẵn theo khoá chính, nên rất nhiều truy cập vốn đã có thứ tự, giảm động lực làm thêm một toán tử sort-merge riêng.
- Với equi-join, hash join nhanh hơn sort-merge trong hầu hết trường hợp (`O(M+N)` so với `O(M log M + N log N)`).
- Với non-equi join, index nested loop đã xử lý được và MySQL vốn thiên về workload OLTP.

**Hệ quả thực tế cần nhớ**: query dạng *range join* (`ON a.ts BETWEEN b.start AND b.end`) chạy rất tốt trên PostgreSQL nhờ merge join, nhưng trên MySQL sẽ rơi về nested loop. Nếu port query kiểu này sang MySQL, hãy chuẩn bị đánh index cẩn thận hoặc đổi cách viết.

---

## So sánh ba thuật toán

| | **Nested Loop** | **Hash Join** | **Sort-Merge** |
|---|---|---|---|
| **MySQL 8.0** | ✅ mặc định | ✅ từ 8.0.18 | ❌ không có |
| **Độ phức tạp** | `O(M·N)` không index<br>`O(M·log N)` có index | `O(M + N)` | `O(M·logM + N·logN)` |
| **Điều kiện join** | bất kỳ (`=`, `<`, `>`, `LIKE`...) | **chỉ `=`** | `=`, `<`, `>`, `BETWEEN` |
| **Cần index?** | Rất nên có | Không | Không (nhưng index sort sẵn thì rẻ) |
| **Bộ nhớ** | ~0 | cao (hash table) | trung bình (sort buffer) |
| **Trả dòng đầu tiên** | **ngay lập tức** (streaming) | sau khi build xong | sau khi sort xong |
| **Hợp với `LIMIT`** | ✅ rất tốt | ❌ kém | ❌ kém |
| **Kiểu I/O** | random | sequential | sequential |
| **Thứ tự kết quả** | theo bảng ngoài | không xác định | đã sắp xếp |
| **Workload** | OLTP, query chọn lọc cao | OLAP, join bảng lớn | join có range, cần sort |

### Đo thật — cùng một query, ba cách thực thi

Join `customers` (1.000 dòng, lọc còn 200) với `orders` (10.000 dòng):

```sql
SELECT COUNT(*) FROM customers c JOIN orders o ON o.customer_id = c.id WHERE c.city='Hanoi';
```

| Tình huống | Thuật toán | Thời gian thật | Dòng đọc |
|---|---|---:|---|
| Có index `idx_orders_customer` | Index Nested Loop | **1,25 ms** | 200 lần tra index |
| Không index, hash join bật | Hash Join | **3,25 ms** | 1.000 + 10.000 |
| Không index, `block_nested_loop=off` | Simple Nested Loop | **515 ms** | 200 × 10.000 = 2 triệu |

Kết quả `EXPLAIN ANALYZE` cho trường hợp tệ nhất nói rõ mọi thứ:

```
-> Nested loop inner join  (actual time=0.0264..515 rows=2000 loops=1)
    -> Filter: (c.city = 'Hanoi')  (actual time=0.0221..0.369 rows=200 loops=1)
        -> Table scan on c  (actual rows=1000 loops=1)
    -> Filter: (o.customer_id = c.id)  (actual time=0.615..2.57 rows=10 loops=200)
        -> Table scan on o  (actual time=700e-6..2.29 rows=10000 loops=200)
                                                              ^^^^^^^^^^^^^^^^^^^
                                               quét 10.000 dòng, LẶP LẠI 200 LẦN
```

`rows=10000 loops=200` — đó là chữ ký của Simple Nested Loop, và là thứ cần tìm đầu tiên khi một query JOIN chậm bất thường.

**Kết luận thực dụng**: chênh lệch giữa hash join và nested loop là **158×**, nhưng chênh lệch giữa "có index" và "không index" mới là điều bạn kiểm soát được — và nó đưa query về **1,25 ms**. Index vẫn là đòn bẩy lớn nhất.

---

## MySQL chọn thuật toán nào?

<div class="zoomable-diagram" style="max-width:820px;margin:2rem auto;">
<svg viewBox="0 0 820 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="fc-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="fc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="820" height="480" fill="#FFFDF7" rx="16"/>
<text x="410" y="30" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Optimizer chọn thuật toán JOIN như thế nào (MySQL 8.0.20+)</text>
<rect x="290" y="52" width="240" height="46" rx="10" fill="#374151" filter="url(#fc-sh)"/>
<text x="410" y="80" text-anchor="middle" font-size="13" font-weight="700" fill="#FFFFFF">Cần join bảng A với bảng B</text>
<rect x="270" y="128" width="280" height="50" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#fc-sh)"/>
<text x="410" y="149" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Có index dùng được cho</text>
<text x="410" y="167" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">điều kiện ON trên bảng trong?</text>
<rect x="70" y="212" width="250" height="62" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#fc-sh)"/>
<text x="195" y="236" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">INDEX NESTED LOOP</text>
<text x="195" y="256" text-anchor="middle" font-size="11" fill="#0F766E">O(M · log N) — đây là thứ bạn muốn</text>
<rect x="470" y="212" width="290" height="50" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2" filter="url(#fc-sh)"/>
<text x="615" y="233" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Có ít nhất 1 điều kiện equi-join</text>
<text x="615" y="251" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">dạng a.col = b.col ?</text>
<rect x="452" y="300" width="216" height="62" rx="10" fill="#E6FBF9" stroke="#14B8A6" stroke-width="2.5" filter="url(#fc-sh)"/>
<text x="560" y="324" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">HASH JOIN</text>
<text x="560" y="344" text-anchor="middle" font-size="11" fill="#0F766E">O(M + N) — chấp nhận được</text>
<rect x="452" y="386" width="308" height="66" rx="10" fill="#FFE9EC" stroke="#FF6B6B" stroke-width="2.5" filter="url(#fc-sh)"/>
<text x="606" y="410" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">HASH JOIN (no condition) + Filter</text>
<text x="606" y="429" text-anchor="middle" font-size="11" fill="#BE123C">≈ tích Descartes rồi lọc — O(M · N)</text>
<text x="606" y="445" text-anchor="middle" font-size="11" font-weight="700" fill="#BE123C">🚨 viết lại query hoặc đánh index gấp</text>
<rect x="70" y="300" width="250" height="152" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="1.5"/>
<text x="88" y="324" font-size="12" font-weight="800" fill="#374151">Ghi nhớ</text>
<text x="88" y="348" font-size="11" fill="#57534E">Hash join là <tspan font-weight="700">lưới an toàn</tspan>,</text>
<text x="88" y="366" font-size="11" fill="#57534E">không phải đích đến.</text>
<text x="88" y="392" font-size="11" fill="#57534E">Thấy hash join trên bảng lớn</text>
<text x="88" y="410" font-size="11" fill="#57534E">trong query OLTP =</text>
<text x="88" y="428" font-size="11" font-weight="700" fill="#9F1239">dấu hiệu thiếu index.</text>
<line x1="410" y1="100" x2="410" y2="124" stroke="#57534E" stroke-width="2" marker-end="url(#fc-arr)"/>
<path d="M270,153 L195,153 L195,208" stroke="#0D9488" stroke-width="2" fill="none" marker-end="url(#fc-arr)"/>
<path d="M550,153 L615,153 L615,208" stroke="#57534E" stroke-width="2" fill="none" marker-end="url(#fc-arr)"/>
<line x1="560" y1="264" x2="560" y2="296" stroke="#0D9488" stroke-width="2" marker-end="url(#fc-arr)"/>
<path d="M700,262 L700,330 L700,382" stroke="#FF6B6B" stroke-width="2" fill="none" marker-end="url(#fc-arr)"/>
<rect x="212" y="142" width="46" height="20" rx="5" fill="#0D9488"/>
<rect x="562" y="142" width="48" height="20" rx="5" fill="#57534E"/>
<rect x="516" y="268" width="42" height="20" rx="5" fill="#0D9488"/>
<rect x="706" y="288" width="52" height="20" rx="5" fill="#FF6B6B"/>
<text x="235" y="157" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<text x="586" y="157" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
<text x="537" y="283" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">CÓ</text>
<text x="732" y="303" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">KHÔNG</text>
</svg>
</div>

---

# Phần 3 — Đọc EXPLAIN cho query JOIN

MySQL 8 có ba định dạng `EXPLAIN`, dùng cho ba mục đích khác nhau:

| Lệnh | Có từ | Dùng để |
|---|---|---|
| `EXPLAIN` | luôn có | Nhìn nhanh bảng nào dùng index gì |
| `EXPLAIN FORMAT=TREE` | 8.0.16 | **Thấy rõ thuật toán JOIN và thứ tự thực thi** |
| `EXPLAIN ANALYZE` | 8.0.18 | **Chạy thật**, đo thời gian và số dòng *thực tế* |

Với JOIN, `FORMAT=TREE` là định dạng quan trọng nhất — nó là nơi duy nhất ghi thẳng ra `Nested loop inner join` hay `Inner hash join`. Bảng `EXPLAIN` cổ điển che giấu thông tin này.

## Cột `type` — chỉ báo sức khoẻ quan trọng nhất

Xếp từ tốt nhất đến tệ nhất:

| `type` | Ý nghĩa | Đánh giá |
|---|---|---|
| `system` | Bảng có đúng 1 dòng | 🟢 hoàn hảo |
| `const` | Khớp tối đa 1 dòng qua PK/UNIQUE với hằng số | 🟢 hoàn hảo |
| `eq_ref` | Với mỗi dòng bảng ngoài, khớp **đúng 1** dòng qua PK/UNIQUE | 🟢 tốt nhất có thể cho JOIN |
| `ref` | Khớp **nhiều** dòng qua index không unique | 🟢 tốt |
| `range` | Quét một khoảng của index (`BETWEEN`, `>`, `IN`) | 🟡 chấp nhận được |
| `index` | Quét **toàn bộ** index (full index scan) | 🟠 đáng ngờ |
| `ALL` | Quét **toàn bộ bảng** (full table scan) | 🔴 báo động nếu bảng lớn |

Ví dụ `eq_ref` — join vào khoá chính:

```sql
EXPLAIN SELECT i.id, p.name FROM order_items i
JOIN products p ON p.id = i.product_id WHERE i.order_id = 42;
```
```
| table | type   | key             | ref                     | rows |
| i     | ref    | idx_items_order | const                   |    3 |
| p     | eq_ref | PRIMARY         | join_demo.i.product_id  |    1 |
```

`eq_ref` + `rows=1`: với mỗi dòng `order_items`, MySQL biết chắc chỉ có đúng một `product` khớp. Đây là dạng JOIN rẻ nhất có thể.

> ⚠️ `ALL` **không phải lúc nào cũng xấu**. Với bảng 20 dòng (`categories`), quét toàn bảng rẻ hơn đọc index rồi nhảy về bảng. Chỉ lo lắng khi `type=ALL` đi kèm `rows` lớn — hoặc khi nó nằm ở **bảng trong** của một nested loop (khi đó nó bị lặp lại `M` lần).

## Các dấu hiệu trong cột `Extra`

| Giá trị | Nghĩa | Đánh giá |
|---|---|---|
| `Using index` | **Covering index** — lấy đủ dữ liệu từ index, không cần đọc bảng | 🟢 rất tốt |
| `Using where` | Có lọc thêm sau khi đọc dòng | ⚪ bình thường |
| `Using join buffer (hash join)` | Đang dùng hash join | 🟡 thiếu index trên cột join |
| `Using join buffer (Block Nested Loop)` | BNL — MySQL ≤ 8.0.19 | 🟠 thiếu index, và MySQL cũ |
| `Using temporary` | Phải tạo bảng tạm (thường do `GROUP BY`/`DISTINCT`) | 🟠 tốn kém |
| `Using filesort` | Phải sắp xếp thêm một bước | 🟠 tốn kém |
| `Range checked for each record` | Chọn lại chiến lược index cho **từng dòng** | 🔴 rất tệ |

## `EXPLAIN ANALYZE` — nơi sự thật lộ ra

`EXPLAIN` chỉ là **ước tính** dựa trên thống kê (có thể cũ, có thể sai). `EXPLAIN ANALYZE` **chạy thật** query rồi báo cáo số liệu thật:

```
-> Nested loop inner join  (cost=318 rows=2435) (actual time=0.0595..1.1 rows=2000 loops=1)
    -> Index lookup on c using idx_customers_city (city='Hanoi')
         (cost=23.8 rows=200) (actual time=0.0479..0.327 rows=200 loops=1)
    -> Covering index lookup on o using idx_orders_customer (customer_id=c.id)
         (cost=0.259 rows=12.2) (actual time=0.00249..0.00325 rows=10 loops=200)
```

Cách đọc từng con số:

| Trường | Nghĩa |
|---|---|
| `cost=318` | Chi phí **ước tính** (đơn vị nội bộ, chỉ dùng để so sánh tương đối) |
| `rows=2435` | Số dòng **ước tính** |
| `actual time=0.0595..1.1` | Thời gian **thật** (ms): `lấy dòng đầu tiên .. lấy xong dòng cuối` |
| `rows=2000` | Số dòng **thật** trả ra |
| `loops=200` | Node này được thực thi lại **200 lần** |

**Ba điều cần soi**:

1. **`rows` ước tính lệch xa `rows` thật** → thống kê cũ. Chạy `ANALYZE TABLE <tên_bảng>`. Optimizer ra quyết định sai vì nó đang nhìn vào dữ liệu cũ.
2. **`loops` lớn ở một node đắt tiền** → nhân lên đó là tổng chi phí thật. `Table scan ... rows=10000 loops=200` = 2 triệu dòng được đọc.
3. **`actual time` nhảy vọt giữa cha và con** → chi phí nằm ở chính toán tử đó (sort, hash build, materialize), không phải ở việc đọc dữ liệu.

⚠️ `EXPLAIN ANALYZE` **thực sự chạy query**. Với `UPDATE`/`DELETE` nó sẽ thay đổi dữ liệu thật. Chỉ dùng với `SELECT`, hoặc bọc trong transaction rồi `ROLLBACK`.

---

# Phần 4 — Thứ tự JOIN: quyết định quan trọng nhất của optimizer

Với `A JOIN B JOIN C`, MySQL có thể chạy theo thứ tự `A→B→C`, `C→B→A`, `B→A→C`... Với `n` bảng có tới `n!` thứ tự khả dĩ. **Thứ tự nào cũng cho cùng kết quả, nhưng chi phí chênh nhau hàng chục lần.**

Nguyên tắc vàng: **lọc càng sớm càng tốt**. Bảng chạy đầu tiên (bảng *driving*) nên là bảng sau khi lọc còn **ít dòng nhất**, vì mọi bảng sau đó đều bị lặp lại theo số dòng của nó.

## Demo: cùng query, hai thứ tự

```sql
SELECT c.name, p.name, i.qty
FROM customers c
JOIN orders      o ON o.customer_id = c.id
JOIN order_items i ON i.order_id    = o.id
JOIN products    p ON p.id          = i.product_id
WHERE c.city = 'Can Tho' AND o.status = 'paid';
```

**Optimizer tự chọn** (`cost=2211`, **7,11 ms**):

```
-> Nested loop inner join
    -> Nested loop inner join
        -> Nested loop inner join
            -> Index lookup on o using idx_orders_status (status='paid')  (rows=2500)
            -> Single-row index lookup on c using PRIMARY (id=o.customer_id)
                 -> Filter: (c.city = 'Can Tho')
        -> Index lookup on i using idx_items_order (order_id=o.id)  (rows=3.03)
    -> Single-row index lookup on p using PRIMARY (id=i.product_id)
```

Nó bắt đầu từ `orders` (lọc `status='paid'` qua index còn 2.500 dòng), rồi thu hẹp ngay bằng `customers`.

**Ép sai thứ tự bằng `STRAIGHT_JOIN`** (`cost=16822`, **23,5 ms**):

```
-> Nested loop inner join
    -> Nested loop inner join
        -> Nested loop inner join
            -> Table scan on i  (rows=30336)      <-- bắt đầu từ bảng LỚN NHẤT
            -> Single-row index lookup on o using PRIMARY (id=i.order_id)
                 -> Filter: (o.status = 'paid')   <-- lọc MUỘN, sau khi đã đọc 30k dòng
```

Bắt đầu từ `order_items` (30.336 dòng, quét toàn bảng) rồi mới lọc — **chậm hơn 3,3 lần** với kết quả y hệt.

## Khi nào cần can thiệp thủ công?

**Hầu như không bao giờ.** Optimizer của MySQL 8 khá tốt. Nó tìm thứ tự bằng thuật toán greedy có cắt tỉa, điều khiển bởi:

```sql
SELECT @@optimizer_search_depth;   -- 62 (mặc định: tự động chọn độ sâu)
SELECT @@optimizer_prune_level;    -- 1  (bật cắt tỉa heuristic)
```

Chỉ can thiệp khi bạn đã **chứng minh bằng `EXPLAIN ANALYZE`** rằng optimizer chọn sai — và nguyên nhân gốc thường là **thống kê lỗi thời**. Thử cái này trước:

```sql
ANALYZE TABLE customers, orders, order_items, products;
```

Nếu vẫn sai, MySQL 8.0.20+ có bộ optimizer hint để ép thứ tự — **nên dùng hint thay vì `STRAIGHT_JOIN`**, vì hint chỉ ảnh hưởng đúng khối query đó:

```sql
-- Ép thứ tự cụ thể cho một số bảng
SELECT /*+ JOIN_ORDER(o, c, i, p) */ ...

-- Ép bảng nào chạy trước tiên
SELECT /*+ JOIN_PREFIX(o) */ ...

-- Ép đúng thứ tự viết trong FROM (tương đương STRAIGHT_JOIN nhưng phạm vi rõ ràng hơn)
SELECT /*+ JOIN_FIXED_ORDER() */ ...

-- Cấm hash join cho một bảng cụ thể (buộc nested loop)
SELECT /*+ NO_BNL(o) */ ...
```

> `HASH_JOIN` / `NO_HASH_JOIN` chỉ có tác dụng ở MySQL 8.0.18–8.0.19. Từ 8.0.20 phải dùng `BNL` / `NO_BNL` để điều khiển hash join — tên hint giữ lại từ thời Block Nested Loop.

Để **thử nghiệm** (chỉ trong session, không bao giờ đặt ở global production):

```sql
SET SESSION optimizer_switch = 'block_nested_loop=off';  -- tắt hash join, buộc nested loop
SET SESSION optimizer_switch = 'block_nested_loop=on';   -- trả lại mặc định
```

---

# Phần 5 — Tối ưu JOIN trong thực tế

## 1. Index đúng chỗ — đòn bẩy lớn nhất

Với `A JOIN B ON B.a_id = A.id`, index cần nằm trên **cột join của bảng trong**. Trong hầu hết trường hợp đó là **khoá ngoại**:

```sql
-- Bắt buộc phải có
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_items_order     ON order_items(order_id);
```

> 🔑 **MySQL không tự tạo index cho khoá ngoại nếu bạn không khai báo `FOREIGN KEY`.** Nhiều codebase hiện đại (kể cả dự án Honeydue mình đang làm) **cố ý bỏ `FOREIGN KEY` constraint** để tránh khoá và để dễ migrate — nhưng khi đó **bạn phải tự nhớ đánh index cho cột khoá ngoại**. Đây là nguồn gốc của rất nhiều query JOIN chậm.

Kiểm tra nhanh xem có cột khoá ngoại nào đang thiếu index không:

```sql
SELECT t.TABLE_NAME, t.COLUMN_NAME
FROM   information_schema.COLUMNS t
LEFT JOIN information_schema.STATISTICS s
       ON  s.TABLE_SCHEMA = t.TABLE_SCHEMA
       AND s.TABLE_NAME   = t.TABLE_NAME
       AND s.COLUMN_NAME  = t.COLUMN_NAME
       AND s.SEQ_IN_INDEX = 1                 -- chỉ tính khi là cột ĐẦU của index
WHERE  t.TABLE_SCHEMA = DATABASE()
  AND  t.COLUMN_NAME LIKE '%\_id'
  AND  s.COLUMN_NAME IS NULL;
```

## 2. Kiểu dữ liệu và collation phải khớp tuyệt đối

Đây là thủ phạm âm thầm phổ biến nhất. Nếu hai cột join khác kiểu, MySQL phải ép kiểu — và **ép kiểu làm index mất tác dụng cho việc tra cứu**.

```sql
-- INT = INT (đúng)
EXPLAIN SELECT COUNT(*) FROM t_int a JOIN t_int b ON b.ref_id = a.ref_id WHERE a.id = 1;
| table | type | key     | ref   | rows |
| b     | ref  | idx_ref | const |   13 |     <-- tra index, đọc 13 dòng

-- INT = VARCHAR (lệch kiểu)
EXPLAIN SELECT COUNT(*) FROM t_int a JOIN t_str b ON b.ref_id = a.ref_id WHERE a.id = 1;
| table | type  | key     | ref  | rows |
| b     | index | idx_ref | NULL | 5000 |     <-- QUÉT TOÀN BỘ index, đọc 5000 dòng
```

`type` tụt từ `ref` xuống `index`, `rows` từ 13 lên **5.000**. Index vẫn xuất hiện ở cột `key` nên rất dễ nhìn nhầm là "vẫn dùng index" — nhưng `ref=NULL` cho thấy nó chỉ đang **quét** index chứ không **tra** index.

Vấn đề tương tự với **collation**. Khi join hai cột khác charset/collation và MySQL không chuyển đổi được theo hướng có lợi, nó phải rơi về hash join:

```
| table | type  | key      | Extra                                           |
| b     | index | idx_code | Using index                                     |
| a     | index | idx_code | Using where; Using index; Using join buffer (hash join) |
```

👉 **Quy tắc**: cột khoá ngoại phải có **cùng kiểu, cùng độ dài, cùng charset, cùng collation** với khoá chính mà nó trỏ tới. Chuẩn hoá toàn DB về `utf8mb4` + một collation duy nhất ngay từ đầu.

## 3. Lọc sớm để bảng driving nhỏ lại

Cách rẻ nhất để tăng tốc JOIN là **giảm số dòng bảng ngoài**, vì mọi thứ phía sau đều nhân theo nó.

```sql
-- Chậm: JOIN trước, lọc sau — bảng ngoài là toàn bộ orders
SELECT c.name, o.id FROM customers c JOIN orders o ON o.customer_id = c.id
WHERE o.ordered_at >= '2026-01-01';

-- Nhanh hơn: index composite cho phép lọc NGAY trong lúc quét
CREATE INDEX idx_orders_date_cust ON orders(ordered_at, customer_id);
```

Với JOIN nhiều bảng có `LIMIT`, đôi khi đáng đẩy phần lọc + phân trang vào derived table trước:

```sql
SELECT c.name, o.*
FROM (SELECT * FROM orders WHERE status='paid' ORDER BY ordered_at DESC LIMIT 20) o
JOIN customers c ON c.id = o.customer_id;
```

Bảng ngoài giờ chỉ còn **20 dòng** thay vì 2.500, và `customers` chỉ bị tra 20 lần.

## 4. Covering index — loại bỏ hẳn bước đọc bảng

Nếu index chứa **đủ mọi cột** mà query cần, MySQL không cần quay lại đọc bảng (không có "bookmark lookup"). Trong `EXPLAIN ANALYZE` nó hiện là `Covering index lookup`:

```
-> Covering index lookup on o using idx_orders_customer (customer_id=c.id)
     (actual time=0.00249..0.00325 rows=10 loops=200)
```

Vì InnoDB lưu bảng theo clustered index, **mọi secondary index đã ngầm chứa sẵn khoá chính**. Nên `idx_orders_customer(customer_id)` thực chất là `(customer_id, id)` — đủ cho query chỉ cần `o.id`. Muốn cover thêm cột khác thì đưa vào index:

```sql
CREATE INDEX idx_orders_cust_cover ON orders(customer_id, status, total_amount);
```

Đánh đổi: index rộng hơn → ghi chậm hơn, tốn đĩa hơn. Chỉ làm cho query nóng, đo trước đo sau.

## 5. JOIN hay N+1 hay `IN`?

Ba cách lấy "10 đơn hàng kèm tên khách":

```sql
-- (A) JOIN: 1 round-trip                          ✅ thường tốt nhất
SELECT o.*, c.name FROM orders o JOIN customers c ON c.id = o.customer_id LIMIT 10;

-- (B) N+1: 1 + 10 round-trip                      ❌ gần như luôn tệ nhất
SELECT * FROM orders LIMIT 10;
-- rồi với mỗi dòng:  SELECT name FROM customers WHERE id = ?

-- (C) Hai query + IN: 2 round-trip                ✅ tốt khi dữ liệu bên phải rộng
SELECT * FROM orders LIMIT 10;
SELECT id, name FROM customers WHERE id IN (1,5,7,...);
```

| | JOIN | N+1 | Hai query + `IN` |
|---|---|---|---|
| Round-trip mạng | 1 | **1 + N** 🔴 | 2 |
| Dữ liệu truyền | lặp cột bảng trái | tối thiểu | tối thiểu |
| DB làm việc | tối ưu | N lần parse + tra | 2 lần |
| Đọc code | trung bình | dễ nhất | trung bình |

**N+1 hầu như luôn sai** — đây là bug kinh điển của ORM (Hibernate lazy loading, GORM `Preload` thiếu, ActiveRecord). Với N=100 và latency mạng 1ms, bạn mất 100ms chỉ để đi lại.

Nhưng cách **(C)** đôi khi thắng JOIN: khi bảng trái có cột lớn (`TEXT`/`JSON`/`BLOB`) và quan hệ là 1-n, JOIN sẽ **lặp lại cột lớn đó trên mỗi dòng** qua đường truyền. Hai query gửi mỗi giá trị đúng một lần. Đây chính là lý do `Preload` (2 query) của GORM thường nhanh hơn `Joins` khi preload quan hệ has-many.

## 6. `join_buffer_size` cho query phân tích

```sql
SET SESSION join_buffer_size = 16 * 1024 * 1024;   -- 16MB, chỉ cho session này
-- ... chạy query báo cáo nặng ...
```

Chỉ giúp khi `EXPLAIN` cho thấy **hash join**. Nếu đang là index nested loop thì tăng buffer không có tác dụng gì. Và nhắc lại: **đặt ở session, không đặt global**.

## 7. Khi nào nên *thôi* JOIN

- **JOIN quá 5-6 bảng trong một query**: không gian tìm kiếm của optimizer bùng nổ, ước tính số dòng sai lệch tích luỹ qua từng tầng. Cân nhắc tách query hoặc dựng bảng tổng hợp.
- **JOIN xuyên service/database**: nếu `orders` và `users` thuộc hai service khác nhau, JOIN ở tầng DB phá vỡ ranh giới. Gọi API rồi ghép ở application, hoặc giữ một bản sao read-model.
- **Dữ liệu tra cứu nhỏ và gần như không đổi** (mã tỉnh thành, danh mục, cấu hình): cache ở application, đừng JOIN mỗi request.
- **Denormalize có chủ đích**: nếu 95% query cần `orders` kèm `customer_name`, lưu luôn `customer_name` vào `orders`. Bạn đánh đổi tính nhất quán lấy tốc độ — hợp lệ, miễn là **quyết định có ý thức** và có cơ chế đồng bộ.

---

# Checklist khi một query JOIN chậm

□ 1. Chạy EXPLAIN FORMAT=TREE  -> thuật toán gì? Nested loop hay hash join?  
□ 2. Chạy EXPLAIN ANALYZE      -> node nào có `loops` lớn? `actual time` nhảy ở đâu?  
□ 3. Có `type = ALL` trên bảng lớn không?  
□ 4. Cột join của bảng trong đã có index chưa?  
□ 5. Hai cột join có CÙNG kiểu / độ dài / charset / collation không?  
□ 6. `rows` ước tính có lệch xa `rows` thật không?  -> ANALYZE TABLE  
□ 7. Bảng driving đã được lọc nhỏ nhất có thể chưa?  
□ 8. Có LEFT JOIN nào đang bị WHERE biến thành INNER JOIN không?  
□ 9. Có JOIN 1-n nào làm nhân dòng, sai SUM/COUNT không?  
□ 10. Có thể chuyển sang covering index không?  

# Tóm tắt

| Câu hỏi | Trả lời ngắn |
|---|---|
| Mặc định nên dùng JOIN nào? | `INNER JOIN`. Chỉ dùng `LEFT JOIN` khi bên phải thực sự có thể không tồn tại |
| MySQL có `FULL OUTER JOIN` không? | Không. Giả lập bằng `UNION` |
| MySQL có Sort-Merge Join không? | **Không.** Chỉ có Nested Loop và Hash Join |
| Hash join có từ bao giờ? | 8.0.18; từ 8.0.20 thay thế hoàn toàn Block Nested Loop |
| Thuật toán nào tốt nhất? | Index Nested Loop cho OLTP, Hash Join cho OLAP |
| Thấy hash join trên bảng lớn nghĩa là gì? | Thiếu index trên cột join |
| `loops=200` kèm `Table scan rows=10000` nghĩa là gì? | Simple Nested Loop — 2 triệu dòng được đọc, cần index gấp |
| Đòn bẩy tối ưu lớn nhất? | Index đúng kiểu, đúng cột, trên cột join của bảng trong |

---

# Tham khảo

- [MySQL 8.0 Reference — Nested-Loop Join Algorithms](https://dev.mysql.com/doc/refman/8.0/en/nested-loop-joins.html)
- [MySQL 8.0 Reference — Hash Join Optimization](https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html)
- [MySQL 8.0 Reference — Optimizer Hints](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)
- [MySQL 8.0 Reference — Obtaining Execution Plan Information](https://dev.mysql.com/doc/refman/8.0/en/execution-plan-information.html)
- Bài liên quan trên blog: [MySQL Indexing](/posts/mysql-indexing/) · [MySQL Index: Composite vs Single](/posts/mysql-index-composite-vs-single/)
