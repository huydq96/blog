+++
date = '2025-06-25T14:00:26+07:00'
draft = false
title = 'MySQL Index: Composite vs Single'
author = 'Huy Dang Quang'
categories = ["MySQL"]
tags = ["mysql", "database", "indexing"]
description = 'So sánh index đa cột và đơn cột trong MySQL'
+++

# Tổng quan

Khi gặp các truy vấn lọc trên nhiều cột, ví dụ `col_a` và `col_b`, đâu là lựa chọn tối ưu hơn – tạo một index đa cột **(composite index)** duy nhất trên ``(col_a, col_b)`` hay tạo hai index đơn cột **(single-column indexes)** riêng biệt trên `(col_a)` và `(col_b)` ?

Quyết định tối ưu phụ thuộc hoàn toàn vào các yếu tố phức tạp và liên kết với nhau, bao gồm **mẫu truy vấn (query patterns)** đặc thù của ứng dụng, **cấu trúc và tính chọn lọc của dữ liệu**, và sự cân bằng chiến lược giữa **hiệu năng đọc (read performance)** và **chi phí ghi (write overhead)**.


# Kiến trúc của index trong MySQL

Hầu hết các loại index phổ biến trong MySQL, bao gồm `PRIMARY KEY`, `UNIQUE`, và `INDEX` thông thường, đều được lưu trữ dưới dạng cấu trúc dữ liệu cây `B-Tree`. Đặc tính cơ bản và quan trọng nhất của cây B-Tree là nó luôn duy trì dữ liệu ở trạng thái **đã được sắp xếp**.

Đọc thêm: [MySQL Indexing](https://blog.huydang.dev/posts/mysql-indexing/)


- **Index đa cột ``(col_a, col_b)``:** Được xem như một mảng đã được sắp xếp, trong đó các phần tử được tạo ra bằng cách nối các giá trị của các cột được đánh index lại với nhau. Dữ liệu được sắp xếp trước hết theo col_a. Với những hàng có cùng giá trị col_a, chúng sẽ được sắp xếp tiếp theo col_b.
- **Hai index đơn cột `(col_a)` và `(col_b)`:** Tạo ra hai cấu trúc cây B-Tree hoàn toàn riêng biệt và độc lập. Một cây chỉ chứa và sắp xếp các giá trị của col_a, trong khi cây còn lại chỉ chứa và sắp xếp các giá trị của col_b.


# Index đa cột - Quy tắc Tiền tố Trái (Left-Prefix Rule)

MySQL có thể sử dụng hiệu quả một index đa cột nếu các điều kiện trong mệnh đề `WHERE` của truy vấn khớp với một hoặc nhiều cột đầu tiên trong định nghĩa index, theo thứ tự từ trái sang phải, một cách liền kề. Nói cách khác, MySQL có thể sử dụng index cho các truy vấn kiểm tra cột đầu tiên, hai cột đầu tiên, ba cột đầu tiên, và cứ thế tiếp diễn.

## Hành vi của Bộ tối ưu hóa

- **Khớp Toàn bộ Tiền tố:** Với truy vấn `WHERE col_a = 'X' AND col_b = 'Y'` trên index `(col_a, col_b)`, đây là trường hợp lý tưởng. Bộ tối ưu hóa có thể thực hiện một thao tác tìm kiếm trực tiếp (seek) trong cây B-Tree đến đúng vị trí, giúp truy vấn cực kỳ hiệu quả.
- **Khớp một phần Tiền tố:** Với truy vấn `WHERE col_a = 'X'`, bộ tối ưu hóa vẫn có thể sử dụng phần đầu của index `(col_a, col_b)` để lọc các hàng một cách hiệu quả. Một hệ quả quan trọng của điều này là nếu bạn có index `(col_a, col_b)`, thì việc tạo thêm một index riêng `(col_a)` là **thừa thãi** và nên được loại bỏ để tiết kiệm không gian và chi phí ghi.
- **Không khớp Tiền tố:** Với truy vấn `WHERE col_b = 'Y'`, index `(col_a, col_b)` không thể được sử dụng để tìm kiếm. Bộ tối ưu hóa sẽ bỏ qua nó và có thể phải thực hiện quét toàn bộ bảng **(full table scan)** hoặc tìm một index khác phù hợp hơn (ví dụ, một index đơn trên `col_b`).
- **Vấn đề "Khoảng trống" (The "Gap" Problem):** Giả sử có index `(col_a, col_b, col_c)` và truy vấn `WHERE col_a = 'X' AND col_c = 'Z'`. Bộ tối ưu hóa chỉ có thể sử dụng index cho `col_a`. Cột `col_b` bị thiếu trong điều kiện WHERE đã tạo ra một "khoảng trống", ngăn cản bộ tối ưu hóa sử dụng `col_c` để thu hẹp phạm vi tìm kiếm.
- **Giới hạn "Phạm vi" (The "Range" Limitation):** Giả sử có index `(col_a, col_b, col_c)` và truy vấn `WHERE col_a = 'X' AND col_b > 'Y' AND col_c = 'Z'`. Bộ tối ưu hóa sẽ sử dụng index cho `col_a` (điều kiện bằng) và `col_b` (điều kiện phạm vi). Tuy nhiên, việc sử dụng index để thu hẹp tìm kiếm sẽ dừng lại ở điều kiện phạm vi đầu tiên `(col_b > 'Y')`. Sau khi tìm thấy tất cả các hàng khớp với `col_a = 'X' và col_b > 'Y'`, nó sẽ phải quét qua các hàng kết quả đó để lọc theo `col_c = 'Z'` thay vì sử dụng index cho `col_c`.

Từ những quy tắc trên, thứ tự các cột trong một index đa cột sẽ ảnh hưởng trực tiếp đến hiệu năng:
- **Tính chọn lọc (Cardinality):** Một nguyên tắc chung là đặt cột có tính chọn lọc cao nhất (nhiều giá trị duy nhất nhất, ví dụ: user_id) ở vị trí đầu tiên. Điều này giúp index loại bỏ được nhiều hàng không phù hợp nhất ngay từ bước đầu tiên, làm tăng hiệu quả lọc.
- **Loại điều kiện:** Các cột được sử dụng trong điều kiện bằng `(=, IN)` nên được đặt trước các cột được sử dụng trong điều kiện phạm vi `(>, <, BETWEEN, LIKE 'prefix%')`. Điều này là do việc sử dụng index sẽ dừng lại ở điều kiện phạm vi đầu tiên.
- **Quy tắc:** Một chiến lược sắp xếp cột hiệu quả thường tuân theo thứ tự: **Cột lọc bằng -> Cột dùng cho GROUP BY / ORDER BY -> Cột lọc phạm vi**.


# Tối ưu hóa Kết hợp index (Index Merge Optimization)

Khi không có một index đa cột hoàn hảo cho một truy vấn, thay vì thực hiện quét toàn bộ bảng, bộ tối ưu hóa có thể sử dụng một cơ chế dự phòng thông minh gọi là "Tối ưu hóa Kết hợp index" **(Index Merge Optimization)**.

Index Merge là một phương pháp truy cập dữ liệu cho phép MySQL thực hiện quét trên nhiều index của cùng một bảng, sau đó hợp nhất các kết quả lại với nhau. Khi có các index đơn cột riêng biệt trên
`col_a` và `col_b`, bộ tối ưu hóa có thể sử dụng cả hai để đáp ứng một truy vấn lọc trên cả hai cột.

## Các thuật toán của Index Merge

Cơ chế của Index Merge bao gồm ba thuật toán khác nhau, được lựa chọn dựa trên loại điều kiện trong mệnh đề `WHERE`. Chúng được hiển thị trong **cột Extra của EXPLAIN**.

- **Intersection (Using intersect(...)):**
    - **Kịch bản:** Được sử dụng cho các điều kiện kết hợp bằng toán tử `AND`, ví dụ: `WHERE col_a = ? AND col_b = ?`.
    - **Cơ chế:** MySQL thực hiện hai lần quét index đồng thời. Lần thứ nhất trên `idx_a` để lấy danh sách con trỏ hàng (row pointers) cho các hàng khớp với col_a. Lần thứ hai trên `idx_b` để lấy danh sách con trỏ hàng cho các hàng khớp với col_b. Sau đó, nó thực hiện một phép toán giao `(intersection)` trên hai danh sách này để tìm ra các con trỏ hàng xuất hiện trong cả hai. Chỉ những hàng này mới được truy xuất từ bảng chính để lấy dữ liệu đầy đủ.
- **Union (Using union(...)):**
    - **Kịch bản:** Được sử dụng cho các điều kiện kết hợp bằng toán tử `OR`, ví dụ: `WHERE col_a = ? OR col_b = ?`.
    - **Cơ chế:** Tương tự như Intersection, MySQL quét cả hai index `idx_a` và `idx_b`. Nhưng thay vì tìm giao điểm, nó thực hiện một phép toán hợp `(union)`, kết hợp tất cả các con trỏ hàng từ cả hai lần quét và loại bỏ các giá trị trùng lặp trước khi truy xuất dữ liệu từ bảng.
- **Sort-Union (Using sort_union(...)):**
    - **Kịch bản:** Đây là một biến thể của Union, được sử dụng khi các điều kiện `OR` liên quan đến các phạm vi (ví dụ: `col_a < 10 OR col_b < 20`) mà thuật toán Union thông thường không thể áp dụng.
    - **Cơ chế:** Thuật toán này phải lấy tất cả các ID hàng từ các lần quét index, sau đó thực hiện một bước sắp xếp (sort) chúng để loại bỏ các giá trị trùng lặp. Bước sắp xếp bổ sung này làm cho nó tốn kém hơn so với thuật toán Union thông thường.

Việc MySQL có sử dụng Index Merge hay không là một quyết định dựa trên chi phí ước tính **(cost estimate)**. Bộ tối ưu hóa sẽ so sánh chi phí của việc sử dụng Index Merge với các phương pháp khác, chẳng hạn như chỉ sử dụng một index duy nhất (index có tính chọn lọc cao nhất) và sau đó lọc các điều kiện còn lại, hoặc thậm chí là quét toàn bộ bảng.

Nếu một trong các index đơn có tính chọn lọc rất cao (ví dụ: một cột `UNIQUE`), bộ tối ưu hóa có thể quyết định rằng cách rẻ nhất là chỉ sử dụng index đó để tìm một vài hàng, sau đó kiểm tra các điều kiện khác trên tập kết quả nhỏ đó, thay vì phải chịu chi phí quét hai index và hợp nhất chúng.


# So sánh dựa trên kịch bản

| Kịch bản Truy vấn | Index đa cột `(col_a, col_b)` | Hai Index đơn `(col_a)`, `(col_b)` | Phân tích |
| -------- | ------- | ------- | ------- |
| `WHERE col_a=? AND col_b=?` | **Xuất sắc.** Thực hiện tìm kiếm B-Tree trực tiếp (Direct Seek) trên một cấu trúc dữ liệu đã được sắp xếp sẵn cho cặp (a, b). | **Khá.** Sử dụng Index Merge Intersection. Chậm hơn đáng kể do phải thực hiện hai lần quét index riêng biệt và sau đó hợp nhất kết quả. Chi phí hợp nhất có thể đáng kể. | **Index Đa cột** là lựa chọn vượt trội và được khuyến nghị mạnh mẽ. |
| `WHERE col_a=? OR col_b=?` | **Không hiệu quả.** Truy vấn OR không thể tận dụng cấu trúc sắp xếp của index đa cột. Bộ tối ưu hóa không thể sử dụng index này một cách hiệu quả và có thể dẫn đến quét toàn bộ bảng. | **Tốt.** Sử dụng Index Merge Union. Đây là kịch bản chính mà Index Merge được thiết kế để giải quyết, cung cấp hiệu năng tốt hơn nhiều so với quét toàn bộ bảng. | **Nhiều index Đơn** là lựa chọn khả thi duy nhất để tối ưu hóa truy vấn `OR`. |
| `WHERE col_a=?` | **Xuất sắc.** Tuân thủ Quy tắc Tiền tố Trái. index được sử dụng hiệu quả như một index đơn trên `col_a`. | **Xuất sắc.** Sử dụng index chuyên dụng `idx_a`. | **Hòa về hiệu năng.** Tuy nhiên, index đa cột `(col_a, col_b)` làm cho index đơn `(col_a)` trở nên dư thừa, có thể tiết kiệm không gian lưu trữ và giảm chi phí bảo trì khi ghi dữ liệu. |
| `WHERE col_b=?` | **Không hiệu quả.** Vi phạm Quy tắc Tiền tố Trái vì `col_b` không phải là cột đầu tiên. index sẽ không được sử dụng để tìm kiếm. | **Xuất sắc.** Sử dụng index chuyên dụng `idx_b`. | Cần có nhiều index đơn hoặc một chiến lược kết hợp (ví dụ: INDEX`(col_a, col_b)` VÀ INDEX`(col_b)`). |
| `ORDER BY col_a, col_b` | **Xuất sắc.** Dữ liệu trong index đã được sắp xếp sẵn theo đúng thứ tự này. MySQL có thể đọc trực tiếp từ index mà không cần thực hiện thao tác sắp xếp tốn kém | **Không hiệu quả.** Không có index nào chứa thứ tự sắp xếp kết hợp này. MySQL sẽ phải thu thập dữ liệu và thực hiện một thao tác sắp xếp tốn nhiều tài nguyên. | **Index Đa cột** hiệu quả hơn rõ ràng cho việc tối ưu hóa sắp xếp trên nhiều cột. |
| `ORDER BY col_b, col_a` | **Không hiệu quả.** Thứ tự sắp xếp của truy vấn ngược lại với thứ tự trong index. | **Không hiệu quả.** Không có index nào phù hợp. | **Không chiến lược nào phù hợp.** Cần một index đa cột riêng biệt được định nghĩa theo đúng thứ tự: `INDEX(col_b, col_a)`. |

## Chiến lược Kết hợp (Hybrid Strategy)

Trong nhiều trường hợp thực tế, một chiến lược kết hợp mang lại sự cân bằng tốt nhất giữa hiệu năng và tính linh hoạt. Một mẫu thiết kế rất phổ biến và mạnh mẽ là tạo cả index đa cột và index đơn cột bổ sung.

Ví dụ, với các cột col_a và col_b, chúng ta có thể tạo:
1. `CREATE INDEX idx_ab ON table_name (col_a, col_b);`
2. `CREATE INDEX idx_b ON table_name (col_b);`

Sự kết hợp này mang lại lợi ích:
- Index `idx_ab` phục vụ tối ưu cho các truy vấn `WHERE col_a = ? AND col_b = ?` và `WHERE col_a = ?`
- Index `idx_b` phục vụ tối ưu cho các truy vấn `WHERE col_b = ?`


# Tổng hợp

Chiến lược Thiết kế Index:
- **Phân tích Tải công việc (Analyze Workload):** Bước đầu tiên và quan trọng nhất là thu thập và ưu tiên các truy vấn quan trọng nhất, thường xuyên nhất của ứng dụng.
- **Ưu tiên các truy vấn AND và ORDER BY:** Nếu các cột thường xuyên được lọc cùng nhau bằng `AND` hoặc sắp xếp cùng nhau, một index đa cột gần như luôn là lựa chọn đúng đắn. Hãy dành thời gian để xác định thứ tự cột tối ưu dựa trên quy tắc: **điều kiện bằng, sắp xếp, rồi đến điều kiện phạm vi**.
- **Hướng đến Covering Index:** Khi thiết kế một index đa cột, hãy kiểm tra xem có thể thêm các cột từ mệnh đề SELECT vào cuối index để tạo ra một index bao phủ hay không. Lợi ích về hiệu năng thường rất lớn.
- **Giải quyết các truy vấn OR và các cột không phải tiền tố:** Nếu có các truy vấn `OR` quan trọng, hoặc cần lọc hiệu quả trên cột thứ hai/thứ ba của một index đa cột (ví dụ `WHERE col_b = ?` khi có `INDEX(col_a, col_b)`), hãy tạo các index đơn cột riêng biệt hoặc áp dụng chiến lược kết hợp.
- **Cân bằng Đọc/Ghi:** Đối với các bảng có tỷ lệ ghi rất cao (heavy-write), hãy thận trọng với việc thêm quá nhiều index. Mỗi index đều có chi phí. Hãy đảm bảo lợi ích về hiệu năng đọc vượt trội so với chi phí ghi.

## Sử dụng EXPLAIN để xác thực lựa chọn

Khi so sánh các chiến lược index, hãy chú ý đến các cột sau trong kết quả của `EXPLAIN`:
- **type:** Cho biết cách MySQL truy cập bảng. Hãy tìm kiếm ref hoặc range (tốt), index (chỉ quét index, thường tốt cho index bao phủ), hoặc index_merge (khá). Hãy cố gắng tránh ALL (quét toàn bộ bảng).
- **possible_keys:** Các index mà MySQL nghĩ rằng nó có thể sử dụng cho truy vấn
- **key:** index mà MySQL thực sự đã chọn để sử dụng. Nếu là NULL, không có index nào được sử dụng. Nếu nó liệt kê nhiều index, đó là dấu hiệu của Index Merge.
- **key_len**: Cột này cực kỳ quan trọng khi làm việc với index đa cột. Nó cho biết có bao nhiêu byte (và do đó là bao nhiêu phần) của một index đa cột đang được sử dụng. Bằng cách so sánh giá trị này với kích thước của các cột trong index, bạn có thể xác minh chính xác Quy tắc Tiền tố Trái đang hoạt động như thế nào.
- **extra:** Chứa thông tin bổ sung quan trọng. Hãy tìm kiếm Using index (dấu hiệu của một index bao phủ - rất tốt), Using where (lọc sau khi lấy dữ liệu), Using filesort (sắp xếp tốn kém - tệ), hoặc Using intersect(...), Using union(...) (dấu hiệu của Index Merge).