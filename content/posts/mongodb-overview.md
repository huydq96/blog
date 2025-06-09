+++
date = '2025-06-05T14:19:52+07:00'
draft = false
title = 'MongoDB Overview'
author = 'Huy Dang Quang'
categories = ["MongoDB"]
tags = ["mongodb", "nosql", "database"]
description = 'Tổng quan về MongoDB'
+++

# Tổng quan

**MongoDB** là một hệ quản trị cơ sở dữ liệu **NoSQL** hướng tài liệu (**document-oriented**), được thiết kế để mang lại tính linh hoạt, hiệu suất cao, tính sẵn sàng cao và khả năng mở rộng vượt trội.


## Mô hình dữ liệu

Khác với các hệ quản trị cơ sở dữ liệu quan hệ (**RDBMS**) truyền thống lưu trữ dữ liệu trong các bảng và hàng, **MongoDB** sử dụng mô hình tài liệu, nơi dữ liệu được tổ chức thành các tập hợp (**collections**) – tương tự như bảng trong RDBMS – và mỗi mục nhập là một tài liệu (**document**) – tương tự như một hàng, nhưng với schema động. 

Document là một đối tượng **BSON** (Binary JSON) chứa các **key-value**. Cấu trúc này cho phép dữ liệu có tính linh hoạt cao, với các trường (fields) không cần được định nghĩa trước và có thể thay đổi giữa các document trong cùng một collection.

Việc MongoDB sử dụng BSON thay vì JSON thuần túy là một quyết định thiết kế cốt lõi nhằm đảm bảo hiệu suất và tính năng nâng cao. JSON là định dạng văn bản dễ đọc, nhưng BSON là nhị phân, được thiết kế để máy tính dễ dàng phân tích và xử lý. Bằng cách mã hóa thông tin kiểu và độ dài, BSON cho phép MongoDB thực hiện lập index mạnh mẽ và khả năng truy vấn sâu vào các tài liệu lồng nhau (ngay cả trong các tài liệu nhúng nhiều lớp), điều mà JSON thuần túy (thường được lưu trữ dưới dạng chuỗi) không thể làm được một cách hiệu quả.   

So sánh JSON và BSON:

| Tiêu chí | JSON | BSON |
| -------- | ------- | ------- |
| Định dạng | Văn bản (UTF-8 string) | Nhị phân |
| Tốc độ | Nhanh để đọc, chậm hơn để xây dựng | Chậm để đọc, nhanh hơn để xây dựng và quét |
| Kích thước | Nhỏ hơn một chút về kích thước byte | Lớn hơn một chút về kích thước byte |
| Mã hóa/Giải mã | Có thể gửi qua API mà không cần mã hóa/giải mã | Cần được mã hóa trước khi lưu trữ và giải mã trước khi hiển thị |
| Khả năng đọc | Con người và máy tính | Chỉ máy tính |
| Kiểu dữ liệu | String, boolean, number, array, object, null | Hỗ trợ thêm: BinData, Decimal128, ObjectID, Date, Regular Expression, JavaScript, MinKey, MaxKey  |
| Mục đích sử dụng | Gửi dữ liệu (chủ yếu qua API) | Cơ sở dữ liệu sử dụng để lưu trữ dữ liệu |


### So sánh cơ bản với cơ sở dữ liệu quan hệ (RDBMS)

- **Schema**: MongoDB nổi bật với schema động (schema-less), cho phép các tài liệu trong cùng một collection có cấu trúc khác nhau, lý tưởng cho các ứng dụng có cấu trúc dữ liệu thay đổi thường xuyên. Ngược lại, RDBMS yêu cầu một schema cố định và được định nghĩa trước, đảm bảo tính toàn vẹn dữ liệu chặt chẽ nhưng kém linh hoạt hơn khi yêu cầu thay đổi.   
- **Lưu trữ dữ liệu**: MongoDB được tối ưu hóa để lưu trữ dữ liệu phi cấu trúc hoặc bán cấu trúc trong các tài liệu BSON. RDBMS được thiết kế cho dữ liệu có cấu trúc, lưu trữ trong các bảng với hàng và cột.   
- **Mối quan hệ**: RDBMS thiết lập mối quan hệ thông qua khóa ngoại (foreign keys) và các phép JOIN. MongoDB sử dụng các chiến lược như Embedded Documents (nhúng tài liệu) hoặc Reference Documents (tham chiếu tài liệu) để quản lý mối quan hệ, giảm thiểu nhu cầu JOIN.
- **Scalability**: MongoDB được thiết kế để mở rộng theo chiều ngang (horizontal scaling) thông qua sharding, phân phối dữ liệu trên nhiều máy chủ để xử lý tải lớn. Điều này cho phép sử dụng các máy chủ thông thường và mở rộng dung lượng khi cần. RDBMS truyền thống thường mở rộng theo chiều dọc (vertical scaling) bằng cách nâng cấp tài nguyên của một máy chủ duy nhất (CPU, RAM, Storage), nhưng cách này có giới hạn và tốn kém hơn.
- **ACID vs. CAP**: RDBMS tuân thủ các thuộc tính ACID (Atomicity, Consistency, Isolation, Durability), đảm bảo tính toàn vẹn dữ liệu mạnh mẽ và độ tin cậy của giao dịch. MongoDB ban đầu tập trung vào định lý CAP (Consistency, Availability, Partition tolerance) để ưu tiên khả năng mở rộng và tính sẵn sàng. Tuy nhiên, từ phiên bản 4.0, MongoDB đã tích hợp hỗ trợ giao dịch ACID đa tài liệu, mở rộng đáng kể khả năng ứng dụng của nó cho các trường hợp yêu cầu tính đúng đắn cao, như hệ thống tài chính. Sự phát triển này đã dịch chuyển cách nhìn về NoSQL từ "No SQL" sang "Not Only SQL", cho phép các hệ thống NoSQL kết hợp những ưu điểm cốt lõi của RDBMS trong khi vẫn giữ được tính linh hoạt và khả năng mở rộng vốn có.   
- **Trường hợp sử dụng**: MongoDB lý tưởng cho các ứng dụng cần schema linh hoạt, xử lý dữ liệu phi cấu trúc/bán cấu trúc, ứng dụng dữ liệu lớn, phân tích thời gian thực và IoT. RDBMS vẫn là lựa chọn hàng đầu cho dữ liệu có cấu trúc, các ứng dụng giao dịch phức tạp và hệ thống tài chính yêu cầu tính đúng đắn cao.


## Kiến trúc cốt lõi

### Replication (Replica Sets) - Tính sẵn sàng cao và chống lỗi

MongoDB Replica Sets là một nhóm các máy chủ MongoDB (nodes) duy trì cùng một tập dữ liệu. Chúng là thành phần thiết yếu để đảm bảo tính sẵn sàng cao, tính dư thừa dữ liệu và khả năng chịu lỗi trong các ứng dụng cơ sở dữ liệu hiện đại.   

- **Cấu trúc Replica Set**: Một replica set điển hình bao gồm một nút chính (Primary Node) và một hoặc nhiều nút thứ cấp (Secondary Nodes). Một nút trọng tài (Arbiter) tùy chọn có thể được sử dụng để phá vỡ các trường hợp hòa trong quá trình bầu cử nút chính mới. Để triển khai trên product, khuyến nghị sử dụng replica set ba thành viên (Primary-Secondary-Secondary) để cung cấp hai bản sao đầy đủ của tập dữ liệu ngoài nút chính, tăng cường khả năng chịu lỗi và tính sẵn sàng.   
- **Vai trò của các thành viên**:
    - **Primary Node**: Nút chính là nút duy nhất trong replica set chấp nhận tất cả các hoạt động ghi. Tất cả các thay đổi dữ liệu đều bắt đầu và được thực hiện ban đầu tại nút này.
    - **Secondary Nodes**: Các nút thứ cấp sao chép dữ liệu từ nút chính. Chúng duy trì một bản sao giống hệt dữ liệu của nút chính để đảm bảo khả năng chịu lỗi và tính sẵn sàng cao. Các nút thứ cấp có thể được sử dụng để phân tán khối lượng công việc đọc và cân bằng tải, vì chúng chủ yếu được sử dụng cho các hoạt động đọc.   
    - **Arbiter**: Một nút trọng tài là một instance MongoDB không lưu trữ dữ liệu nhưng chỉ tham gia vào quá trình bỏ phiếu trong quá trình bầu cử. Nó giúp đảm bảo luôn có đa số thành viên để bầu chọn nút chính mới, đặc biệt hữu ích trong các cấu hình có số lượng thành viên chẵn.   
    - **Cơ chế hoạt động và lợi ích**: Khi dữ liệu được ghi vào nút chính, nó sẽ được sao chép sang tất cả các nút thứ cấp, đảm bảo mỗi nút trong replica set có một bản sao dữ liệu giống hệt nhau. Quá trình đồng bộ hóa này giúp duy trì tính nhất quán và tính sẵn sàng của dữ liệu. Trong trường hợp nút chính không khả dụng, một nút thứ cấp sẽ tự động được bầu làm nút chính mới mà không cần can thiệp thủ công, giảm thiểu thời gian ngừng hoạt động. Replica sets cũng hỗ trợ phục hồi nút tự động; nếu một nút thứ cấp không khả dụng, MongoDB sẽ tự động cố gắng kết nối lại nút đó khi nó khả dụng trở lại.

### Sharding - Khả năng mở rộng theo chiều ngang

Sharding là quá trình phân chia tập dữ liệu lớn thành các "chunk" nhỏ hơn và phân phối chúng trên nhiều instance MongoDB. Mỗi shard lưu trữ một tập con của tổng dữ liệu, đảm bảo các truy vấn và giao dịch được phân phối đều.

Lợi ích chính của sharding bao gồm: tăng dung lượng bằng cách phân phối dữ liệu và tải trên nhiều máy chủ, cải thiện hiệu suất đọc/ghi bằng cách cân bằng tải, tính sẵn sàng cao thông qua việc sao chép dữ liệu trên nhiều máy chủ, và hiệu quả chi phí bằng cách sử dụng phần cứng thông thường.   

**Các thành phần của Sharded Cluster**: Một MongoDB Sharded Cluster bao gồm ba loại thành phần chính:
- **Shards**: Đây là các máy chủ riêng lẻ hoặc replica sets lưu trữ các tập con dữ liệu. Mỗi shard là một replica set, cung cấp tính sẵn sàng cao và khả năng mở rộng đọc.   
- **Config Servers**: Các máy chủ này lưu trữ siêu dữ liệu và cài đặt cấu hình cho cluster, bao gồm thông tin về phân phối dữ liệu.
- **Query Routers** (`mongos`): Đây là các máy chủ định tuyến các truy vấn từ ứng dụng đến các shard thích hợp. Client kết nối với `mongos`, và `mongos` xử lý việc định tuyến truy vấn, tổng hợp kết quả và quản lý các hoạt động cluster.
- **Shard Key và quản lý Chunks**: Dữ liệu được phân phối trên các shard dựa trên một shard key. Shard key là một trường được lập index xác định cách dữ liệu được phân vùng trên các shard. Việc chọn shard key phù hợp là rất quan trọng để phân phối dữ liệu cân bằng và hiệu suất tối ưu. MongoDB sử dụng shard key để phân vùng dữ liệu thành các "chunk" thuộc sở hữu của một shard cụ thể. Một chunk bao gồm một dải dữ liệu đã được sharding. Khi một chunk đạt kích thước tối đa (mặc định 128 MB), nó sẽ được chia thành hai chunk mới. Balancer là một quá trình nền tự động di chuyển dữ liệu giữa các shard để duy trì sự phân phối đều dữ liệu khi có sự chênh lệch đáng kể về lượng dữ liệu giữa các shard.   


## Tối ưu hiệu suất trong MongoDB

### Mô hình hóa dữ liệu nâng cao (Data Modeling): Embedding vs. Referencing

Mô hình hóa dữ liệu hiệu quả là một quyết định then chốt trong thiết kế schema của MongoDB, ảnh hưởng trực tiếp đến hiệu suất, khả năng mở rộng và khả năng bảo trì của ứng dụng. Có hai cách chính để biểu diễn mối quan hệ giữa các dữ liệu: nhúng (embedding) và tham chiếu (referencing).   

1. **Embedded Documents**

- Trong mô hình này, dữ liệu liên quan được lưu trữ trực tiếp bên trong một document khác, tạo thành một cấu trúc phân cấp lồng nhau. Ví dụ, các mục hàng trong một đơn đặt hàng có thể được nhúng trực tiếp vào document đơn đặt hàng.
- **Ưu điểm**: Cải thiện hiệu suất đọc vì tất cả dữ liệu liên quan có thể được truy xuất trong một truy vấn duy nhất, tránh các phép nối phức tạp như trong RDBMS. Các hoạt động cập nhật trên tài liệu nhúng là nguyên tử, đảm bảo tính nhất quán dữ liệu.   
- **Hạn chế**: Kích thước tài liệu có giới hạn (16MB). Nếu dữ liệu nhúng phát triển nhanh hoặc rất lớn, nó có thể vượt quá giới hạn này. Việc cập nhật thường xuyên dữ liệu nhúng có thể dẫn đến việc ghi lại toàn bộ tài liệu, ảnh hưởng đến hiệu suất ghi.   

2. **Referenced Documents**

- Mô hình tham chiếu lưu trữ các mối quan hệ giữa dữ liệu bằng cách bao gồm các liên kết (thường là `ObjectId`) từ một tài liệu đến một tài liệu khác được lưu trữ trong một collection riêng biệt.   
- **Ưu điểm**: Cho phép mô hình dữ liệu linh hoạt và chuẩn hóa hơn, giúp quản lý các mối quan hệ phức tạp. Giúp giữ kích thước tài liệu nhỏ hơn, tránh giới hạn 16MB. Thích hợp cho các mối quan hệ nhiều-nhiều hoặc khi dữ liệu liên quan được truy cập độc lập.
- **Hạn chế**: Yêu cầu nhiều truy vấn hơn (hoặc sử dụng `$lookup` trong aggregation pipeline) để truy xuất dữ liệu liên quan, có thể ảnh hưởng đến hiệu suất đọc so với mô hình nhúng.   

So sánh trường hợp sử dụng:
- **Ưu tiên Embedded theo mặc định**: Khi thiết kế schema cho MongoDB, nên ưu tiên nhúng theo mặc định, đặc biệt đối với dữ liệu liên quan chặt chẽ và thường xuyên được truy cập cùng nhau. Điều này tối ưu hóa hiệu suất đọc và đơn giản hóa các hoạt động.   
- **Sử dụng Referenced khi**:
    - Dữ liệu nhúng có thể phát triển rất lớn hoặc vượt quá giới hạn 16MB.
    - Dữ liệu liên quan được cập nhật thường xuyên và độc lập với tài liệu chính.
    - Cần biểu diễn các mối quan hệ phức tạp như nhiều-nhiều.
    - Thực thể liên quan thường xuyên được truy vấn riêng lẻ.
    - Cần tránh trùng lặp dữ liệu quá mức mà không mang lại lợi ích hiệu suất đáng kể.

### Indexing và tối ưu hóa truy vấn

Một truy vấn không có index phù hợp sẽ phải quét toàn bộ collection (**COLLSCAN**), điều này kém hiệu quả và chậm hơn đáng kể so với quét index (**IXSCAN**).

Các loại index:
- **Single Field Indexes**: Tối ưu hóa tìm kiếm trên một trường duy nhất, bao gồm cả các trường tài liệu nhúng.  
- **Compound Indexes**: Lập index nhiều trường cùng nhau, cải thiện hiệu suất cho các truy vấn lọc hoặc sắp xếp theo nhiều trường.   
- **Multikey Indexes**: Tự động được tạo khi lập index một trường chứa giá trị mảng, tạo các mục index riêng lẻ cho mỗi phần tử trong mảng.   
- **Text Indexes**: Hỗ trợ truy vấn tìm kiếm văn bản đầy đủ (full text) trên các trường chứa nội dung chuỗi.
- **Geospatial Indexes (2d và 2dsphere)**: Dành cho dữ liệu tọa độ địa lý, hỗ trợ các truy vấn dựa trên vị trí như tìm điểm gần đó hoặc tính toán khoảng cách.   
- **Hashed Indexes**: Lập index giá trị băm của một trường, hữu ích để phân phối dữ liệu cân bằng trên nhiều máy chủ trong môi trường sharded cluster, đặc biệt cho các trường có cardinality cao.   
- **Sparse Indexes**: Chỉ lập index các tài liệu có trường được lập index tồn tại, giúp giảm kích thước index và tối ưu hóa việc sử dụng bộ nhớ cho dữ liệu không đồng đều.   
- **Partial Indexes**: Lập index chỉ một tập hợp con các tài liệu trong một collection dựa trên một biểu thức lọc, hữu ích để giảm kích thước index và tối ưu hóa truy vấn cho các tập con dữ liệu cụ thể.   
- **Covering Indexes**: Một index bao phủ chứa tất cả các trường cần thiết cho một truy vấn (lọc, sắp xếp, trả về), cho phép MongoDB trả về kết quả trực tiếp từ index mà không cần truy cập tài liệu gốc, cải thiện đáng kể hiệu suất.

Chiến lược lập index và tối ưu hóa truy vấn:
- **Quy tắc ESR (Equality, Sort, Range)**: Khi tạo index tổng hợp, thứ tự các trường rất quan trọng. Các trường được sử dụng cho các phép so sánh bằng (Equality) nên đứng đầu, tiếp theo là các trường phản ánh thứ tự sắp xếp (Sort), và cuối cùng là các trường được sử dụng cho các bộ lọc dải (Range).
- **Phân tích hiệu suất truy vấn**: Sử dụng `db.collection.explain("executionStats")` để hiểu cách truy vấn được thực thi, xác định xem index có được sử dụng hiệu quả hay không, và kiểm tra các chỉ số như `nReturned`, `totalKeysExamined`, `totalDocsExamined`, và `executionTimeMillis`.
- **Tránh các truy vấn không được lập index**: Các truy vấn không có index trên các tập dữ liệu lớn có thể gây ra hiệu suất kém nghiêm trọng.   
- **Giới hạn kết quả và sử dụng Projection**: Sử dụng `limit()` để giảm số lượng kết quả trả về. Sử dụng `projection` để chỉ trả về các trường cần thiết, giảm sử dụng bộ nhớ và thời gian xử lý.
**Giám sát và điều chỉnh thường xuyên**: Thường xuyên xem xét việc sử dụng index và hiệu suất để xác định các index dư thừa hoặc không được sử dụng. Các index không cần thiết có thể làm chậm hoạt động ghi và tiêu tốn tài nguyên.


## Aggregation Framework - Xử lý và phân tích dữ liệu

MongoDB Aggregation Framework là một công cụ mạnh mẽ để thực hiện phân tích dữ liệu phức tạp trực tiếp trong MongoDB. Nó cho phép chuyển đổi dữ liệu đa giai đoạn thông qua một **pipeline** tổng hợp, lý tưởng cho các kịch bản cần xử lý và phân tích các tập dữ liệu lớn.

Aggregation pipeline là một chuỗi các giai đoạn (**stages**) mà dữ liệu đi qua, với đầu ra của một giai đoạn trở thành đầu vào cho giai đoạn tiếp theo. Mỗi giai đoạn thực hiện một thao tác cụ thể trên các tài liệu, biến đổi chúng khi chúng đi qua pipeline. Điều này khác với các truy vấn `find()` đơn giản chỉ tìm nạp dữ liệu trực tiếp.

Các giai đoạn (Stages) và toán tử (Operators) phổ biến:
- `$match`: Lọc tài liệu dựa trên các điều kiện được chỉ định, thường được sử dụng sớm trong pipeline để giảm số lượng tài liệu được xử lý.
- `$group`: Nhóm các tài liệu đầu vào theo một biểu thức nhận dạng và áp dụng các biểu thức tích lũy (accumulator expressions) như `$sum`, `$avg`, `$max`, `$min`, `$count` cho mỗi nhóm.
- `$project`: Định hình lại tài liệu, bao gồm, loại trừ hoặc đổi tên các trường, hoặc tạo các trường được tính toán.
- `$sort`: Sắp xếp tất cả các tài liệu đầu vào theo thứ tự được chỉ định (tăng dần 1 hoặc giảm dần -1).
- `$limit`: Giới hạn số lượng tài liệu được trả về.
- `$skip`: Bỏ qua một số lượng kết quả nhất định, thường được sử dụng cho phân trang.
- `$unwind`: Phân tách một trường mảng từ tài liệu đầu vào thành nhiều tài liệu, mỗi tài liệu cho một phần tử mảng.
- `$lookup`: Thực hiện phép **left outer join** với một collection khác trong cùng một database, cho phép kết hợp dữ liệu từ nhiều collection.
- `$addFields`: Thêm các trường mới vào tài liệu đầu ra.
- `$count`: Đếm nhanh số lượng tài liệu trong một pipeline.

Tối ưu hóa hiệu suất Aggregation:
- **Đặt `$match` và `$limit` sớm**: Đặt các giai đoạn `$match` và `$limit` càng sớm càng tốt trong pipeline để giảm lượng dữ liệu được xử lý bởi các giai đoạn tiếp theo, cải thiện đáng kể hiệu suất.   
- **Sử dụng Indexes hiệu quả**: Đảm bảo các giai đoạn `$match`, `$sort` và `$group` có thể sử dụng index.
- **Chỉ project các trường cần thiết**: Sử dụng `$project` hoặc `$unset` để chỉ bao gồm các trường cần thiết cho quá trình tổng hợp, giảm sử dụng bộ nhớ và cải thiện hiệu suất.   
- **Xử lý các tập hợp lớn**: Các hoạt động tổng hợp có giới hạn bộ nhớ mặc định là 100 MB. Đối với các hoạt động có thể vượt quá giới hạn này, sử dụng tùy chọn `allowDiskUse: true` để cho phép MongoDB sử dụng disk, mặc dù điều này có thể ảnh hưởng đến hiệu suất. Đối với các tập kết quả quá lớn để nằm trong bộ nhớ, có thể sử dụng giai đoạn `$merge` để xuất kết quả vào một collection mới hoặc hiện có.