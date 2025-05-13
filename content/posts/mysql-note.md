+++
date = '2025-05-07T23:25:08+07:00'
draft = false
title = 'MySQL Notes'
author = 'Huy Dang Quang'
categories = ["MySQL"]
tags = ["database", "mysql"]
description = 'Một số nội dung về MySQL'
+++

# Tổng quan
Đây là một số note về MySQL.
Nội dung đươc lấy từ EBook [MySQL cho người đi làm](https://huynt.dev/products/mysql-cho-nguoi-di-lam).


## Có nên sử dụng ràng buộc (Constraint) ở tầng cơ sở dữ liệu ?

Ràng buộc (**constraints**) trong cơ sở dữ liệu là các quy tắc được áp dụng trên các bảng để đảm bảo
tính toàn vẹn và nhất quán của dữ liệu. Một
số ràng buộc phổ biến như `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`,...

Không nhất thiết phải cài đặt ràng buộc `FOREIGN KEY`. Một số hạn chế như sau:
- **Giảm hiệu suất cập nhật**: Ràng buộc `FOREIGN KEY` có thể làm giảm hiệu suất ghi chép dữ liệu, đặc biệt khi có nhiều thao tác ghi hoặc cập nhật dữ liệu vì cơ sở dữ liệu phải thực hiện các kiểm tra bổ sung để duy trì tính toàn vẹn tham chiếu.
- **Phức tạp trong quản lý**: Việc thêm, sửa đổi hoặc loại bỏ ràng buộc `FOREIGN KEY` có thể phức tạp, đặc biệt khi cơ sở dữ liệu lớn hoặc có nhiều mối quan hệ tham chiếu phức tạp.
- **Giới hạn tính linh hoạt**: Ràng buộc `FOREIGN KEY` có thể làm giảm tính linh hoạt của cơ sở dữ liệu, đặc biệt khi có các yêu cầu thay đổi cấu trúc dữ liệu hoặc nghiệp vụ thường xuyên.
- **Khó khăn trong phát triển và kiểm tra**: Trong quá trình phát triển, ràng buộc `FOREIGN KEY` có thể làm cho việc nhập dữ liệu thử nghiệm trở nên khó khăn hơn, vì bạn phải đảm bảo rằng tất cả các mối quan hệ tham chiếu đều hợp lệ.

Do vậy trong thực tế không cần bắt buộc triển khai ràng buộc `FOREIGN KEY`. Ta có thể chấp nhận việc dư thừa dữ liệu để đảm bảo tính linh hoạt của cơ sở dữ liệu.


## Tính chất ACID

- **Atomicity (Tính Nguyên Tử)**: Transaction phải hoạt động như một đơn vị công việc không thể chia nhỏ, toàn bộ giao dịch hoặc được áp dụng hoàn toàn hoặc sẽ không thành công toàn bộ
- **Consistency (Tính Nhất Quán)**: Tính nhất quán đảm bảo rằng một transaction sẽ đưa cơ sở dữ liệu từ một trạng thái nhất quán này sang một trạng thái nhất quán khác. Tính nhất quán duy trì các ràng buộc toàn vẹn dữ liệu, như ràng buộc khóa chính, khóa ngoại, hoặc các ràng buộc không null.
- **Isolation (Tính Cách Ly)**: Tính cách ly đảm bảo rằng các transaction đồng thời không ảnh hưởng lẫn nhau. Các thay đổi được thực hiện bởi một transaction chưa hoàn tất sẽ không hiển thị cho các giao dịch khác.
- **Durability (Tính Bền Vững)**: Tính bền vững đảm bảo rằng một khi transaction đã được commit, các thay đổi của nó sẽ được lưu trữ vĩnh viễn, ngay cả khi hệ thống gặp sự cố ngay sau đó.


## Các mức cách ly (Isolation level)

Mức cách ly xác định các quy tắc cho những thay đổi nào là hiển thị và không hiển thị bên trong và bên ngoài một transaction. Điều này là quan trọng, vì nó có thể ảnh hưởng đến tinh chính xác và cả hiệu suất của một transaction.

### READ UNCOMMITTED

Đây là mức cách ly thấp nhất trong các mức cách ly của cơ sở dữ liệu. Ở mức này, các transaction có thể đọc các thay đổi chưa được commit từ các transaction khác. Điều này có nghĩa là một transaction có thể thấy dữ liệu bị thay đổi bởi một transaction khác, ngay cả khi transaction đó chưa hoàn thành và commit.

**Đặc điểm:**
- **Dirty Read** (Đọc dữ liệu “bẩn”): Transaction có thể đọc các thay đổi chưa được commit từ các transaction khác. Điều này có thể dẫn đến việc đọc dữ liệu không nhất quán.
- **Không có bảo đảm về tính nhất quán**: Vì các thay đổi chưa được commit có thể bị rollback, dữ liệu đọc được ở mức cách ly này có thể không chính xác hoặc không đáng tin cậy.
- **Hiệu suất cao**: Mức cách ly này có thể cải thiện hiệu suất vì nó ít tốn kém về mặt tài nguyên và khóa dữ liệu so với các mức cách ly cao hơn.

Ví dụ có 2 transaction đang chạy đồng thời:

```sql
--- Transaction 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- Transaction 1 chưa COMMIT


--- Transaction 2
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1;
-- Transaction 2 đọc được dữ liệu chưa được commit từ Transaction 1
COMMIT;
```

Trong ví dụ này, transaction 2 có thể đọc số dư tài khoản đã bị giảm 100 mặc dù transaction 1 chưa được commit. Nếu sau đó transaction 1 bị rollback, transaction 2 đã đọc dữ liệu không chính xác.

### READ COMMITTED

Ở mức cách ly này, một transaction chỉ có thể đọc các thay đổi đã được commit bởi các transaction khác. Điều này giúp tránh được các vấn đề liên quan đến dirty read (đọc dữ liệu “bẩn”), đảm bảo rằng các dữ liệu đọc được luôn nhất quán và đáng tin cậy.

**Đặc điểm:**
- **Không có dirty read**: Transaction chỉ có thể đọc các dữ liệu đã được commit, do đó tránh được việc đọc dữ liệu không nhất quán.
- **Non-repeatable read**: có thể xảy ra hiện tượng non-repeatable read, nghĩa là dữ liệu có thể thay đổi giữa các lần đọc trong cùng một giao dịch nếu một transaction khác commit thay đổi dữ liệu đó.
- **Hiệu suất tốt**: Cân bằng giữa hiệu suất và tính nhất quán, phù hợp cho nhiều ứng dụng.

Ví dụ có 2 transaction đang chạy đồng thời:

```sql
--- Transaction 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- Transaction 1 chưa COMMIT


--- Transaction 2
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1;
COMMIT;
```

Trong ví dụ này, Transaction 2 sẽ không thấy số dư tài khoản đã bị giảm 100 bởi Transaction 1 vì Transaction 1 chưa được commit. Transaction 2 sẽ chỉ đọc dữ liệu đã được commit trước đó.

### REPEATABLE READ

Đây là mức cách ly mặc định trong MySQL và được coi là một trong những mức cách ly mạnh mẽ nhất. Ở mức cách ly này, một transaction sẽ thấy cùng một tập hợp dữ liệu nếu nó đọc cùng một truy vấn nhiều lần trong suốt quá trình thực thi, ngay cả khi các transaction khác đã thay đổi dữ liệu đó và commit các thay đổi của chúng. `REPEATABLE READ` ngăn chặn cả **dirty read** và **non-repeatable read**.

**Đặc điểm:**
- **Ngăn chặn dirty read**: Giao dịch chỉ có thể đọc dữ liệu đã được commit.
- **Ngăn chặn non-repeatable read**: Dữ liệu đọc được trong transaction sẽ không thay đổi cho đến khi giao dịch kết thúc, ngay cả khi các giao dịch khác đã cam kết thay đổi
- **Phantom Read**: Vẫn có thể xảy ra hiện tượng **phantom read**, nghĩa là các row mới có thể xuất hiện hoặc các row hiện có có thể biến mất nếu một transaction khác chèn hoặc xóa row trong phạm vi đã được đọc
- **Sử dụng MVCC (Multiversion Concurrency Control)**: MySQL sử dụng MVCC để đảm bảo các transaction đọc có thể thấy trạng thái nhất quán của dữ liệu.
- Hiệu suất sẽ giảm hơn so với hai mức cách ly bên trên.

Ví dụ có 2 transaction đang chạy đồng thời:

```sql
--- Transaction 1
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1;
-- Transaction 1 đọc số dư tài khoản lần đầu
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- Transaction 1 chưa COMMIT

--- Transaction 2
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1;
-- Transaction 2 đọc số dư tài khoản, thấy số dư trước khi Transaction 1 cập
nhật
COMMIT;
```

Trong ví dụ này, Transaction 2 sẽ thấy số dư ban đầu của tài khoản trước khi Transaction 1 cập nhật. Ngay cả khi Transaction 1 commit thay đổi, Transaction 2 cũng sẽ không thấy thay đổi đó nếu nó thực hiện lại truy vấn.

### SERIALIZABLE

SERIALIZABLE là mức cách ly cao nhất trong các mức cách ly của cơ sở dữ liệu. Ở mức cách ly này, mỗi giao dịch được thực hiện một cách tuần tự, không có transaction nào có thể nhìn thấy các thay đổi của giao dịch khác cho đến khi transaction đó hoàn thành. Điều này đảm bảo rằng không có phantom read, non-repeatable read hay dirty read.

**Đặc điểm:**
- **Không có dirty read**: Giao dịch chỉ đọc các thay đổi đã được commit.
- **Không có non-repeatable read**: Dữ liệu đọc được sẽ không thay đổi trong suốt thời gian transaction.
- **Không có phantom read**: Transaction sẽ không thấy dữ liệu mới được thêm vào bởi các transaction khác
- **Khóa toàn bộ các hàng dữ liệu**: Mỗi transaction sẽ khóa các dãy dữ liệu nó truy cập, ngăn chặn các transaction khác thay đổi dữ liệu trong các hàng đó.
- Hiệu suất của SERIALIZABLE sẽ là thấp nhất trong bốn loại cách ly được đề cập ở đây.

Ví dụ có 2 transaction đang chạy đồng thời:

```sql
--- Transaction 1
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1 FOR UPDATE;
-- Transaction 1 khóa tài khoản 1 để tránh các thay đổi từ các giao dịch khác
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
COMMIT;


--- Transaction 2
START TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 1;
-- Transaction 2 sẽ bị chặn lại cho đến khi Transaction 1 hoàn thành
COMMIT;
```

Trong ví dụ này, Transaction 2 sẽ bị chặn lại cho đến khi Transaction 1 hoàn thành và commit. Điều này đảm bảo rằng Transaction 2 sẽ không thấy bất kỳ thay đổi nào từ Transaction 1 cho đến khi Transaction 1 hoàn tất.

## Deadlocks

Deadlocks (Khoá chết) xảy ra khi hai hoặc nhiều transaction đồng thời giữ các khóa trên một tập hợp tài nguyên và mỗi transaction chờ đợi một khóa đang được giữ bởi một transaction khác trong tập
hợp đó (Transaction A đợi Transaction B, nhưng B cũng lại đợi A). Điều này tạo ra một vòng lặp phụ thuộc mà không thể giải quyết được, và tất cả các transaction liên quan sẽ chờ đợi vô thời hạn trừ khi có biện pháp can thiệp để giải quyết tình trạng này.

Để giải quyết vấn đề này, các hệ thống cơ sở dữ liệu triển khai các dạng phát hiện và thời gian chờ deadlock khác nhau. Ví dụ như storage engine nổi tiếng của MySQL là InnoDB: Nó có khả năng phát
hiện các dấu hiệu phụ thuộc vòng tròn, và báo lỗi ngay lập tức. Điều này là rất quan trọng, bởi nếu Deadlocks không được xử lý kịp thời sẽ khiến cả hệ thống chạy rất chậm. Những cơ sở dữ liệu khác
khác sẽ tiến hành rollback transaction sau khi truy vấn vượt quá thời gian chờ khóa, điều này không phải lúc nào cũng tốt. Cách mà InnoDB hiện tại xử lý deadlock là rollback giao dịch khóa ít row nhất (một thước đo gần đúng cho cái nào sẽ dễ rollback nhất).

Cách xử lý trong ứng dụng:
- **Thiết lập thứ tự truy cập tài nguyên**: Luôn thao tác với các bảng và hàng theo cùng thứ tự trong mọi transaction để tránh xung đột khóa
- **Tối ưu hóa transaction**: Giữ transaction càng ngắn càng tốt. Sử dụng `SELECT ... FOR UPDATE` thay vì LOCK TABLES khi cần khóa hàng
- **Chia nhỏ batch**: Tách các câu lệnh ảnh hưởng nhiều hàng thành các phần nhỏ
- **Xử lý khi xảy ra deadlock**: Triển khai cơ chế **Retry** trong ứng dụng

