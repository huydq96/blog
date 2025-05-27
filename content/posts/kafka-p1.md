+++
date = '2025-05-27T14:21:38+07:00'
draft = false
title = 'Apache Kafka P1'
author = 'Huy Dang Quang'
categories = ["Kafka"]
tags = ["kafka", "message queue"]
description = 'Các khái niệm chính của Kafka'
+++

# Tổng quan

**Kafka là gì** ? Về cốt lõi, Kafka là một hệ thống nhắn tin **publish/subscribe**. Nó được thiết kế để cung cấp một bản ghi bền vững (durable record) về tất cả các giao dịch, cho phép chúng được phát lại để xây dựng trạng thái nhất quán của hệ thống. Dữ liệu trong Kafka được lưu trữ bền vững, theo thứ tự và có thể được đọc một cách xác định. Nó cũng được phân tán để cung cấp khả năng chống lại sự cố và cơ hội mở rộng hiệu suất đáng kể.

**Apache Kafka** là một dự án mã nguồn mở, ban đầu được thiết kế tại LinkedIn bởi một nhóm dẫn đầu bởi Jay Kreps, Neha Narkhede, và Jun Rao. Nó được phát hành dưới dạng mã nguồn mở vào cuối năm 2010 và gia nhập Apache Software Foundation vào năm 2011.

- Tài liệu tham khảo: [Kafka And RabbitMQ Books](https://github.com/ramosITBooks/KafkaAndRabbitMQBooks)


# Các thành phần cơ bản

## Message (tin nhắn) / Record (Bản ghi)

Đơn vị dữ liệu trong Kafka. Đối với Kafka, một message chỉ là một mảng byte, không có định dạng hoặc ý nghĩa cụ thể. Message có thể có một phần metadata tùy chọn gọi là **key** (khóa), cũng là một mảng byte. Key được sử dụng để kiểm soát cách message được ghi vào các **partition**.

## Topic (Chủ đề)

Message trong Kafka được phân loại thành các **topic**. Topic giống như một bảng cơ sở dữ liệu hoặc một thư mục trong hệ thống tệp.

## Partition (Phân vùng)

Topic được chia thành một số **partition**. Một partition là một **log** duy nhất, nơi message được ghi theo kiểu chỉ thêm (**append-only**) và được đọc theo thứ tự từ đầu đến cuối. Một topic thường có nhiều partition, do đó không có đảm bảo về thứ tự thời gian của message trên toàn bộ topic, chỉ có thể đảm bảo trong một partition duy nhất. Partition cũng là cách Kafka cung cấp khả năng dự phòng và mở rộng. Một partition duy nhất chỉ tồn tại trên một broker và không bị chia nhỏ giữa các broker. Các partition được chia nhỏ hơn nữa thành các segment file trên đĩa.

## Producer (Bộ xuất bản)

Là thành phần gửi message đến các topic của Kafka. Kafka có khả năng xử lý liền mạch nhiều producer, dù các client này đang sử dụng nhiều topic hay cùng một topic. Producer không cần quan tâm ai đang sử dụng dữ liệu hoặc số lượng ứng dụng consumer.

## Consumer (Bộ tiêu thụ)

Là thành phần truy xuất message từ Kafka. Kafka được thiết kế để nhiều consumer có thể đọc bất kỳ luồng message nào mà không can thiệp lẫn nhau. Các consumer có thể chọn hoạt động như một phần của **consumer group** (nhóm consumer) và chia sẻ một luồng dữ liệu, đảm bảo rằng toàn bộ nhóm xử lý một message nhất định chỉ một lần. Consumer tự kiểm soát tốc độ tiêu thụ dữ liệu (mô hình pull). Consumer theo dõi vị trí đọc của mình trong log bằng cách sử dụng **offset**.

## Broker (Máy chủ Kafka)

Một máy chủ Kafka duy nhất được gọi là broker. Broker nhận message từ producer, gán offset cho chúng và commit message.

## ZooKeeper

Một thành phần giúp duy trì sự đồng thuận trong cluster Kafka. Mặc dù Kafka có thể đứng độc lập, sự phụ thuộc vào ZooKeeper đôi khi gây nhầm lẫn với các hệ sinh thái khác như Hadoop.


# Các khái niệm quan trọng khác

## Commit Log

Như đã đề cập, đây là nền tảng của Kafka. Log này có tính chất chỉ thêm (**append-only**). Người dùng sử dụng **offset** để biết vị trí của họ trong log này.

## Consumer Groups

Nhiều consumer cùng tham gia một nhóm để cùng nhau tiêu thụ dữ liệu từ các partition của một topic. Kafka theo dõi offset cuối cùng được commit cho từng nhóm consumer và từng partition.

## Replication (Sao chép) và Durability (Độ bền)

Kafka cung cấp khả năng sao chép các partition trên nhiều broker. Mỗi partition có một leader và các follower. Các follower trong nhóm sao chép đồng bộ (In-Sync Replicas - ISR) đã nhận và cam kết tất cả message mà leader đã nhận. Khi leader gặp sự cố, một leader mới sẽ được bầu chọn từ các ISR. Cấu hình replication.factor xác định số bản sao của mỗi partition. Có thể chấp nhận Unclean Leader Election để ưu tiên tính sẵn sàng hơn tính nhất quán dữ liệu, dẫn đến khả năng mất dữ liệu.

## Topic Compaction 

Một chính sách dọn dẹp log thay thế việc xóa dữ liệu cũ theo thời gian hoặc kích thước. Với topic compaction, Kafka chỉ giữ lại message cuối cùng cho mỗi key trong một partition. Nó hữu ích cho dữ liệu dạng changelog hoặc trạng thái. Topic Compaction chỉ chạy trên các segment không hoạt động.


# Tại sao nên chọn Kafka

- **Xử lý Nhiều Producer và Consumer**: Kafka có thể xử lý hiệu quả nhiều nguồn dữ liệu ghi vào (producer) và nhiều ứng dụng đọc dữ liệu (consumer) từ cùng một luồng mà không gây trở ngại cho nhau.
- **Lưu giữ Dữ liệu trên Đĩa (Disk-Based Retention)**: message được cam kết vào đĩa và được lưu trữ với các quy tắc giữ lại có thể cấu hình. Các quy tắc này có thể dựa trên kích thước hoặc thời gian, được thiết lập trên cơ sở từng topic. Điều này cho phép consumer không nhất thiết phải hoạt động theo thời gian thực.
- **Khả năng Mở rộng (Scalable)**: Producer, consumer và broker đều có thể mở rộng theo chiều ngang (scale out) để xử lý các luồng message rất lớn một cách dễ dàng. Việc chia topic thành các partition là cơ chế chính cho khả năng mở rộng này.
- **Hiệu suất Cao**: Kết hợp các tính năng này mang lại hiệu suất xuất bản/tiêu thụ xuất sắc dưới tải trọng cao.
- **Ghép nối Lỏng (Loose Coupling)**: Kafka cung cấp một giao diện nhất quán cho tất cả client, cho phép các thành phần trong hệ sinh thái dữ liệu được thêm hoặc xóa mà không ảnh hưởng đến producer hoặc consumer khác.
- **Khả năng Phát lại message (Replay Messages)**: Do message không bị xóa khỏi hệ thống sau khi được consumer đọc, consumer có thể chọn đọc lại message bằng cách tìm kiếm đến một offset trước đó trong log của partition.
- **Exactly Once Semantics (EOS)**: Kafka hỗ trợ đảm bảo rằng một message sẽ được xử lý chính xác một lần bởi consumer.


# Ứng dụng của Kafka

Apache Kafka thường được sử dụng cho nhiều trường hợp ứng dụng khác nhau trong kiến trúc hướng sự kiện (**Event-Driven Architecture** - EDA) và các hệ thống phân tán.

- **Publish-subscribe (Pub/Sub)**: Kafka được sử dụng như một hệ thống nhắn tin **publish/subscribe**, nơi các nhà sản xuất (**producers**) xuất bản tin nhắn lên các **topic** mà không cần biết về người tiêu dùng (**consumers**), và người tiêu dùng chỉ quan tâm đến các danh mục nội dung cụ thể (topic). Mô hình này tạo ra sự kết nối lỏng lẻo giữa các hệ thống sản xuất và tiêu thụ, chỉ cần biết về các topic chung và lược đồ tin nhắn. Kafka có thể cạnh tranh với các message broker truyền thống trong các kịch bản nhắn tin chung, hoạt động tối ưu quanh các chuỗi sự kiện bất biến, không giới hạn.
- **Theo dõi hoạt động (Activity Tracking)**: Đây là trường hợp sử dụng ban đầu của Kafka khi nó được thiết kế tại LinkedIn. Các ứng dụng frontend trên website tạo ra tin nhắn về hành động của người dùng, chẳng hạn như lượt xem trang (page views), theo dõi click, hoặc các hành động phức tạp hơn như cập nhật hồ sơ người dùng. Những tin nhắn này được xuất bản lên các topic và sau đó được tiêu thụ bởi các ứng dụng backend để tạo báo cáo, cấp dữ liệu cho hệ thống học máy, cập nhật kết quả tìm kiếm, hoặc thực hiện các hoạt động khác. LinkedIn đã sử dụng Kafka để xử lý khối lượng lớn sự kiện trang web và dữ liệu log aggregation. Tumblr và Pinterest cũng sử dụng Kafka cho mục đích theo dõi hoạt động và quảng cáo thời gian thực.
- **Log Aggregation**: Kafka rất hữu ích trong việc thu thập các sự kiện ứng dụng được ghi trong các ứng dụng phân tán. Log files được gửi dưới dạng tin nhắn vào Kafka, và các ứng dụng khác nhau có thể tiêu thụ thông tin này từ một topic logic duy nhất. Khả năng xử lý lượng lớn dữ liệu của Kafka làm cho việc thu thập các sự kiện từ nhiều máy chủ hoặc nguồn trở thành một tính năng quan trọng. Kafka cũng được sử dụng trong các công cụ ghi log như Graylog.
- **Stream Processing**: Kafka cung cấp nền tảng để xử lý các luồng sự kiện khi chúng xảy ra. Các ứng dụng tiêu thụ có thể thực hiện các tác vụ như đếm metric, phân vùng tin nhắn để xử lý hiệu quả, hoặc chuyển đổi tin nhắn bằng cách sử dụng dữ liệu từ nhiều nguồn. API Kafka Streams được thiết kế để xây dựng các ứng dụng xử lý luồng dữ liệu một cách dễ dàng, bao gồm xử lý trạng thái cục bộ với khả năng chịu lỗi, xử lý từng tin nhắn một (one-at-a-time) và hỗ trợ Exactly Once Semantics.
- **Messaging**: Kafka cũng được sử dụng cho các trường hợp nhắn tin chung, chẳng hạn như các ứng dụng cần gửi thông báo (ví dụ: email) cho người dùng. Khả năng lưu trữ bền bỉ (durable storage) và tính sẵn sàng cao (high availability) là những yếu tố quan trọng khiến Kafka được lựa chọn cho mục đích này. Kafka không sử dụng các giao thức nhắn tin truyền thống như XMPP, AMQP, hoặc JMS API, mà thay vào đó sử dụng một giao thức nhị phân TCP tùy chỉnh.
- **Event Sourcing**: Kafka có thể hoạt động như một kho lưu trữ sự kiện bền vững (durable event store) trong mô hình Event Sourcing. Bất kỳ người tiêu dùng nào cũng có thể xây dựng lại ảnh chụp nhanh trạng thái ứng dụng tại một điểm thời gian bằng cách phát lại tất cả các bản ghi cho đến thời điểm đó. Kafka hỗ trợ kịch bản Event Sourcing unimodal cổ điển (sử dụng một chế độ hoạt động cho cả hydration ban đầu và sao chép sự kiện tiếp theo) bằng cách sử dụng tính năng log compaction.


# Cấu hình Kafka

Một số config cần chú ý khi cấu hình Kafka

## Độ bền dữ liệu (Durability) và Độ sẵn sàng (Availability)

### replication.factor

Cấu hình **Topic**: Xác định số bản sao (replica) của mỗi partition.
- Được thiết lập khi tạo topic.
- Trong môi trường production, cluster Kafka thường có nhiều node, tối thiểu là ba.

### min.insync.replicas

Cấu hình **Broker/Topic**: Xác định số lượng bản sao tối thiểu (bao gồm leader và các follower đang đồng bộ) phải xác nhận việc ghi một bản ghi thì bản ghi đó mới được coi là đã được ghi bền vững (durable).

- Giá trị mặc định là 1 (chỉ leader), điều này "hầu như không bền vững".
- Để đảm bảo việc ghi bền vững, nên đặt **min.insync.replicas** tối thiểu là 2 (leader cộng thêm ít nhất một follower). Cài đặt này nên phản ánh mức độ chấp nhận mất mát dữ liệu của bạn. Có thể đặt cấu hình này cho toàn bộ cluster hoặc cho từng topic cụ thể.

### acks

Cấu hình **Producer**: Quy định số lượng xác nhận mà producer yêu cầu từ các follower của leader trước khi coi yêu cầu gửi bản ghi là hoàn thành. Thông số này là nền tảng cho độ bền của bản ghi. Cấu hình sai acks có thể dẫn đến mất mát dữ liệu.

- **0**: Độ trễ thấp nhất, nhưng độ bền kém nhất. Không đảm bảo broker có nhận được tin nhắn hay không và không thử lại. Có thể chấp nhận cho dữ liệu không quá quan trọng (ví dụ: theo dõi click trên web).
- **all (-1)**: Đảm bảo mạnh mẽ nhất. Bản ghi được xác nhận bởi leader và tất cả các replica đang đồng bộ (min.insync.replicas).
- Nên đặt **acks=all** trong các trường hợp tính đúng đắn của hệ thống phụ thuộc vào sự đồng thuận giữa producer và Kafka cluster rằng bản ghi đã được xuất bản (ví dụ: các ứng dụng giao dịch tài chính). Khi sử dụng acks=all, cần đảm bảo **min.insync.replicas** được đặt **>= 2** để có ý nghĩa. Cấu hình này có thể làm tăng độ trễ, nhưng có thể chấp nhận được so với rủi ro mất dữ liệu.

## Truyền tin (Messaging) và Đảm bảo phân phối (Delivery Guarantees)

### Delivery Guarantees

Kafka hỗ trợ ba chế độ đảm bảo phân phối:
- **At least once** (Ít nhất một lần): Cấu hình mặc định của Kafka. Producer có thể gửi cùng một tin nhắn nhiều lần, và nó sẽ được ghi vào broker. An toàn cho các trường hợp không thể bỏ sót tin nhắn (ví dụ: thanh toán hóa đơn), nhưng consumer có thể cần xử lý bản sao.
- **At most once** (Nhiều nhất một lần): Tin nhắn có thể bị mất nhưng không bao giờ bị trùng lặp.
- **Exactly once semantics (EOS)** (Chính xác một lần): Producer gửi tin nhắn nhiều lần, nhưng consumer cuối cùng chỉ nhận được một lần.

### enable.idempotence

Cấu hình **Producer**: Khi bật (true), Kafka đảm bảo việc gửi tin nhắn từ một producer sẽ được thực hiện chính xác một lần trong một phiên làm việc. Điều này ngăn chặn bản sao hoặc bản ghi không theo thứ tự trong quá trình thử lại. Kết hợp với tính **idempotence** ở phía consumer (ví dụ: Kafka Streams), có thể đạt được EOS end-to-end.

### auto.offset.reset

Cấu hình **Consumer**: Xác định hành vi của consumer khi không có offset ban đầu được cam kết hoặc offset hiện tại không còn hợp lệ (ví dụ: dữ liệu đã bị xóa).
- **latest** (mặc định): Bắt đầu đọc từ vị trí high-water mark (cuối log).
- **earliest**: Bắt đầu đọc từ đầu log.
- Khi sử dụng latest (mặc định) có tiềm năng gây ra "hỗn loạn" và dẫn đến mất dữ liệu (giảm đảm bảo phân phối xuống "at most once") nếu offset đã cam kết bị mất sau thời gian downtime. Nên đặt auto.offset.reset=earliest khi sử dụng consumer group để duy trì đảm bảo "at least once" trong trường hợp offset bị mất, miễn là dữ liệu còn nằm trong thời gian lưu trữ (retention) của topic.

### enable.auto.commit và auto.commit.interval.ms

Cấu hình **Consumer**: Cho phép consumer tự động cam kết (commit) offset đọc dữ liệu định kỳ.
- Việc auto-commit có thể dẫn đến việc cam kết offset cho các bản ghi chưa được xử lý hoàn toàn, có khả năng vi phạm đảm bảo "at least once".
- Việc sử dụng commit offset thủ công thường được ưa chuộng hơn để có quyền kiểm soát chính xác thời điểm cam kết, đảm bảo bản ghi đã thực sự được xử lý [Information not directly in provided sources, but inferred from discussion of auto-commit drawbacks].

## Tổ chức dữ liệu và Xử lý

### partitioner.class

Cấu hình **Producer**: Cho phép ứng dụng producer ghi đè lên logic phân vùng mặc định.
- Logic mặc định băm (**hash**) key của bản ghi để xác định partition.
- Việc gán key cho bản ghi dựa trên định danh của thực thể (entity) liên quan là thực tế phổ biến. Điều này đảm bảo các bản ghi liên quan đến cùng một thực thể (ví dụ: cùng một khách hàng, cùng một cảm biến) được đưa vào cùng một partition, duy trì thứ tự xử lý cho các sự kiện của thực thể đó.

### group.id

Cấu hình **Consumer**: Định danh duy nhất cho một consumer group. Quan trọng cho cơ chế đăng ký topic và phân chia partition giữa các consumer trong nhóm (load balancing). Kafka đảm bảo tại bất kỳ thời điểm nào, mỗi partition chỉ được gán cho tối đa một consumer trong một group.

## Lưu trữ và Xóa dữ liệu (Data Retention)

### cleanup.policy

- **delete** (mặc định): Xóa các segment log cũ khi log đạt đến một kích thước nhất định hoặc các bản ghi cũ hơn một độ tuổi nhất định.
- **compact**: Xóa các bản ghi cũ hơn dựa trên key, chỉ giữ lại bản ghi mới nhất cho mỗi key. Phù hợp cho các topic cần duy trì trạng thái hiện tại của thực thể (ví dụ: Event Sourcing).
- Có thể kết hợp cả hai chính sách.

## Bảo mật (Security)

Kafka hỗ trợ cấu hình cho **bảo mật lưu lượng (confidentiality)** bằng cách mã hóa (SSL/TLS), **xác thực (authentication)** (ví dụ: SASL, SSL client certificates), và **phân quyền (authorization)** thông qua Access Control Lists (ACLs).