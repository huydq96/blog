+++
date = '2025-05-26T14:07:01+07:00'
draft = false
title = 'MySQL Indexing'
author = 'Huy Dang Quang'
categories = ["MySQL"]
tags = ["mysql", "indexing"]
description = 'Index trong MySQL'
+++

# Tổng quan

Trong thế giới của các hệ quản trị cơ sở dữ liệu, MySQL luôn là một lựa chọn phổ biến và mạnh mẽ. Tuy nhiên, khi dữ liệu ngày càng lớn và số lượng truy vấn tăng lên, việc duy trì hiệu suất cao có thể trở thành một thách thức. Một trong những chiến lược cơ bản và hiệu quả nhất để giải quyết vấn đề này chính là tối ưu hóa sử dụng Index.

Bài blog này sẽ đi sâu vào cách hoạt động của Index trong MySQL, các loại Index phổ biến và những chiến lược để tạo ra các Index hiệu suất cao, giúp tăng tốc độ truy vấn và cải thiện hiệu năng tổng thể của hệ thống.

Nội dung được lấy từ EBook [MySQL cho người đi làm](https://huynt.dev/products/mysql-cho-nguoi-di-lam).


# Index trong MySQL là gì?

Cách dễ nhất để hiểu Index trong MySQL là liên tưởng đến mục lục của một cuốn sách. Khi bạn muốn tìm một chương cụ thể, bạn không cần đọc toàn bộ cuốn sách từ đầu đến cuối. Thay vào đó, bạn tra cứu trong mục lục để biết số trang và nhảy trực tiếp đến đó.

Trong MySQL, Index hoạt động tương tự. Nó là một cấu trúc dữ liệu giúp MySQL tìm kiếm kết quả của truy vấn một cách nhanh chóng. Thay vì phải quét toàn bộ bảng (**full table scan**), Index cho phép storage engine tìm đến các hàng (rows) chứa dữ liệu cần thiết một cách hiệu quả hơn.

## Cấu Trúc của Index

Một Index chứa các giá trị từ một hoặc nhiều cột trong một bảng. Khi bạn tạo Index trên nhiều cột (Index đa cột), **thứ tự của các cột là rất quan trọng**. MySQL chỉ có thể tìm kiếm hiệu quả theo thứ tự **từ trái sang phải** của các cột được định nghĩa trong Index. Việc tạo Index trên hai cột (col1, col2) khác với việc tạo hai Index đơn lẻ (col1) và (col2).

## Lợi ích của Index

- **Tăng tốc độ truy vấn**: Index giúp MySQL tìm kiếm và truy xuất dữ liệu nhanh hơn nhiều so với việc quét toàn bộ bảng.
- **Tối ưu hóa hiệu suất**: Giúp giảm tải hệ thống bằng cách giảm số lượng hàng cần phải đọc và xử lý.

## Hạn chế của Index

- **Tốn không gian lưu trữ**: Index cần không gian lưu trữ bổ sung để lưu trữ cấu trúc dữ liệu của chúng.
- **Tăng thời gian Insert, Update, Delete**: Các thao tác ghi dữ liệu (INSERT, UPDATE, DELETE) có thể chậm hơn vì MySQL cũng phải cập nhật Index.


# Các loại Index chính trong MySQL

Index được triển khai ở tầng **storage engine**, không phải ở lớp SQL, do đó việc hỗ trợ và triển khai có thể khác nhau giữa các engine. Dưới đây là một số loại Index chính:

## B-Tree Indexes:

- Là loại Index phổ biến nhất, thường được ngầm hiểu khi nói đến Index nói chung. Hầu hết các storage engine hỗ trợ loại này.
- **Cách hoạt động**: Sử dụng cấu trúc dữ liệu B-tree (Balance Tree) để lưu trữ dữ liệu theo thứ tự. Các giá trị được sắp xếp và mỗi trang lá cách đều nhau từ gốc. Cấu trúc cây này giúp nhanh chóng tìm được giá trị cần tìm mà không cần Full-Scan.
- **Lợi ích**: 
    - **Tăng tốc độ truy vấn**
    - **Hỗ trợ sắp xếp** (ORDER BY, GROUP BY)
    - **Tăng tốc độ tìm kiếm theo phạm vi** (BETWEEN, >, <, >=, <=)
    - **Giảm I/O**
- **Tối ưu hóa truy vấn**:
    - **Chọn cột để tạo Index**: Tạo index trên các cột mà bạn thường xuyên sử dụng trong các điều kiện WHERE, JOIN, ORDER BY, và GROUP BY.
    - **Tránh Index trên các cột có giá trị thay đổi thường xuyên**: do việc cập nhật index có thể tốn kém tài nguyên.
    - **Sử dụng Index đa cột khi cần thiết**: Nếu thường xuyên sử dụng nhiều cột trong các điều kiện tìm kiếm, hãy xem xét tạo index đa cột để tối ưu hóa hiệu suất truy vấn.
- **Hạn chế**: 
    - Không hữu ích cho các truy vấn không bắt đầu từ cột đầu tiên trong Index nhiều cột
    - Chi phí cập nhật Index
    - Không hiệu quả cho các cột có ít sự đa dạng (tính selective thấp).

## Hash Indexes:

- Sử dụng cấu trúc dữ liệu bảng băm (hash). Đặc biệt hiệu quả cho các truy vấn tìm kiếm chính xác (so sánh bằng).
- **Cách hoạt động**: Chuyển đổi giá trị Index thành số hash, sử dụng số hash này để tìm kiếm nhanh chóng.
- **Lợi ích**: 
    - Hiệu suất cao cho tìm kiếm chính xác
    - Có thể tiết kiệm bộ nhớ hơn so với B-tree trong một số trường hợp.
- **Hạn chế**: 
    - Không hỗ trợ tìm kiếm phạm vi (<, >, BETWEEN) hoặc sắp xếp (ORDER BY)
    - Hiệu suất có thể giảm khi xảy ra xung đột hash
    - Chủ yếu chỉ hỗ trợ bởi storage engine MEMORY trong MySQL (không dùng với InnoDB)
- Bonus: InnoDB có tính năng Adaptive Hash Index (AHI). AHI tự động tạo các hash Index dựa trên các truy vấn thường xuyên, cải thiện khả năng tìm kiếm nhanh chóng cho B-tree Index. Quá trình này hoàn toàn tự động.

## Full-text Indexes

- Loại Index đặc biệt được thiết kế để tìm kiếm từ khóa trong văn bản thay vì so sánh trực tiếp giá trị. Hữu ích cho các ứng dụng cần tìm kiếm phức tạp trong các trường văn bản dài.
- **Cách hoạt động**: Tạo danh sách các từ xuất hiện trong văn bản và lưu trữ vị trí của chúng. Sử dụng Index này để tìm kiếm từ khóa khi có truy vấn. Hỗ trợ các tính năng như từ chặn (stop words), từ gốc (stemming), tìm kiếm Boolean.
- **Cách tạo**: Sử dụng từ khóa `FULLTEXT`.
- **Cách truy vấn**: Sử dụng câu lệnh `MATCH ... AGAINST`. Hỗ trợ Natural Language Search, Boolean Mode, Query Expansion.
- **Lợi ích**: 
    - Tìm kiếm từ khóa nhanh chóng và hiệu quả
    - Hỗ trợ tìm kiếm phức tạp
    - Phù hợp với văn bản dài
- **Hạn chế**: 
    - Chỉ hỗ trợ các cột kiểu CHAR, VARCHAR, và TEXT
    - Không hiệu quả cho các truy vấn ngắn hoặc tìm kiếm chính xác
    - Yêu cầu cấu hình đặc biệt


# Chiến lược tạo Index hiệu suất cao

Để tạo ra một Index "tốt", bạn có thể tham khảo hệ thống đánh giá Index ba sao (three-star index) trong cuốn sách **Relational Database Index Design and the Optimizers** (tác giả *Tapio Lahdenmaki* và *Mike Leach*)

## Các hàng liên quan nằm gần nhau (Adjacent Rows)

Index nên sắp xếp dữ liệu sao cho các hàng có giá trị liên quan nằm gần nhau, giảm số lượng trang dữ liệu cần đọc.

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total DECIMAL(10, 2),
    INDEX (customer_id)
);
```

Ví dụ trong bảng **orders**, nếu thường xuyên truy vấn tất cả các đơn hàng của một khách hàng cụ thể,
index trên **customer_id** sẽ giúp các hàng có cùng **customer_id** nằm gần nhau, giảm thiểu số lượng
trang dữ liệu cần đọc.

```sql
SELECT * FROM orders WHERE customer_id = 123;
```


## Các hàng được sắp xếp theo thứ tự truy vấn yêu cầu (Sorted Rows)

Index nên sắp xếp dữ liệu theo thứ tự mà truy vấn cần (ví dụ: trong mệnh đề ORDER BY), giúp MySQL không phải thực hiện sắp xếp dữ liệu tạm thời.

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    last_name VARCHAR(50),
    first_name VARCHAR(50),
    hire_date DATE,
    INDEX (last_name, first_name)
);
```

Ví dụ Index trên **last_name** và **first_name** giúp sắp xếp các hàng theo thứ tự trên, hữu ích cho các truy vấn sắp xếp theo tên.

```sql
SELECT * FROM employees ORDER BY last_name, first_name;
```

## Index bao gồm tất cả các cột cần thiết cho truy vấn (Covering Index)

Index nên chứa tất cả các cột mà truy vấn SELECT yêu cầu. MySQL có thể lấy dữ liệu trực tiếp từ Index mà không cần truy cập vào bảng chính, giảm I/O

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10, 2),
    category_id INT,
    INDEX (category_id, price)
);
```

Ví dụ câu truy vấn sau có thể được tối ưu hóa bởi index **category_id**, **price**, vì index bao gồm tất cả các cột cần thiết (**category_id** và **price**):

```
SELECT category_id, price FROM products WHERE category_id = 10 ORDER BY price;
```


# Chiến lược chọn index

- **Hiểu rõ dữ liệu và truy vấn**: Phân tích cách dữ liệu được sử dụng và các truy vấn phổ biến để xác định cột cần Index và loại Index phù hợp.
- **Index các cột thường xuyên truy vấn**: Tạo Index trên các cột xuất hiện trong điều kiện **WHERE**, **JOIN**, **ORDER BY**, và **GROUP BY**.
- **Sử dụng Index đa cột**: Cân nhắc Index đa cột nếu bạn thường xuyên truy vấn trên nhiều cột kết hợp.
- **Tối ưu thứ tự cột trong Index đa cột**: Đây là yếu tố cực kỳ quan trọng.
    - Đặt cột được sử dụng thường xuyên nhất trong điều kiện **WHERE** lên đầu.
    - Ưu tiên các cột có độ chọn lọc cao (nhiều giá trị khác nhau).
    - Đặt các cột dùng trong **ORDER BY** hoặc **GROUP BY** vào Index theo thứ tự phù hợp.
    - Đặt các cột dùng trong JOIN vào Index.
    - **Lưu ý**: MySQL chỉ sử dụng hiệu quả phần đầu tiên của Index đa cột nếu các cột đó khớp với điều kiện **WHERE**. Nếu điều kiện **WHERE** không bao gồm cột đầu tiên trong Index, Index sẽ không được sử dụng hiệu quả.
- **Sử dụng Index phủ (Covering Index)**: Thiết kế Index sao cho chứa tất cả các cột mà truy vấn SELECT yêu cầu để tăng hiệu suất bằng cách tránh truy cập bảng.
- **Index Prefixed**: Đối với cột văn bản dài, tạo Index trên một phần giá trị để tiết kiệm không gian mà vẫn hiệu quả.
- **Tránh tạo quá nhiều Index**: Index tốn không gian và làm chậm thao tác ghi. Chỉ tạo Index khi thực sự cần thiết.
