+++
date = '2025-05-27T16:09:22+07:00'
draft = false
title = 'Apache Kafka P2'
author = 'Huy Dang Quang'
categories = ["Kafka"]
tags = ["kafka", "message queue", "interview"]
description = 'Một số câu hỏi phỏng vấn liên quan đến Kafka'
+++


# Có thể sử dụng Kafka mà không cần ZooKeeper không ?

## Chức năng cốt lõi của ZooKeeper

**ZooKeeper** đã đóng vai trò thiết yếu trong kiến trúc Kafka trong hơn một thập kỷ. Hệ thống này chịu trách nhiệm cho các tác vụ quan trọng bao gồm **quản lý metadata của cụm Kafka**, lưu trữ thông tin về vị trí các broker, cấu hình topic và các phân vùng partition. ZooKeeper cũng thực hiện bầu cử **leader** cho mỗi partition của topic, đảm bảo chỉ có một broker chịu trách nhiệm xử lý ghi cho một partition cụ thể tại bất kỳ thời điểm nào. Ngoài ra, ZooKeeper quản lý **consumer group offsets**, lưu trữ vị trí đọc hiện tại của mỗi consumer group, cho phép consumer tiếp tục đọc từ vị trí cũ khi bị lỗi hay khởi động lại.

Một chức năng quan trọng khác của ZooKeeper là **giám sát sức khỏe** của cụm Kafka thông qua cơ chế **ephemeral znode**. Mỗi broker khi tham gia vào cụm Kafka sẽ đăng ký với một ephemeral znode trên ZooKeeper, và controller election được thực hiện thông qua việc tạo ephemeral /controller znode. ZooKeeper cũng lưu trữ **Access Control Lists (ACLs)** và quotas, hỗ trợ việc xác thực và ủy quyền client cũng như giới hạn tài nguyên sử dụng.

## Những thách thức với ZooKeeper

Việc duy trì đồng thời cụm ZooKeeper cùng với cụm Kafka tạo ra **độ phức tạp vận hành cao**, đòi hỏi quản trị viên phải hiểu và quản lý hai hệ thống khác nhau với các file cấu hình, công cụ quản lý và mô hình triển khai riêng biệt. Tương tác giữa Kafka và ZooKeeper có thể tạo ra độ trễ, đặc biệt trong các cụm lớn hoặc khi có nhiều cập nhật metadata.

ZooKeeper cũng trở thành **điểm nghẽn** giới hạn số lượng partition mà một broker có thể xử lý, thường giới hạn ở khoảng 100,000+ partitions. Đặc biệt quan trọng về **khả năng chịu lỗi**, nếu ZooKeeper gặp sự cố hoặc không ổn định, cả cụm Kafka có thể bị ảnh hưởng nghiêm trọng. Những hạn chế này đã thúc đẩy cộng đồng Kafka tìm kiếm giải pháp thay thế tốt hơn.

## Giới thiệu về chế độ KRaft

KRaft (Kafka Raft) là **giao thức đồng thuận** mới được phát triển nhằm mục đích loại bỏ sự phụ thuộc của Kafka vào ZooKeeper. Thay vì lưu trữ metadata thành hai thành phần hệ thống riêng biệt như ZooKeeper và Kafka, KRaft cho phép Kafka tự lưu trữ các metadata trên chính nó. Hệ thống sử dụng **Quorum Controller** mới với giao thức đồng thuận Raft để thực hiện bầu chọn leader và quản lý metadata.

Trong kiến trúc KRaft, các **controller nodes** tạo thành một Raft quorum để quản lý Kafka metadata log. Log này chứa thông tin về mọi thay đổi đối với metadata của cụm, bao gồm tất cả những gì trước đây được lưu trữ trong ZooKeeper như topics, partitions, ISRs, configurations. Sử dụng giao thức đồng thuận Raft, các controller nodes duy trì tính nhất quán và bầu cử leader mà không cần dựa vào bất kỳ hệ thống bên ngoài nào.

## Cơ chế hoạt động của KRaft

KRaft hoạt động dựa trên **event-driven model**, trong đó quorum controller sử dụng event-sourced storage model để đảm bảo rằng các state machine nội bộ luôn có thể được tái tạo chính xác. Event log được sử dụng để lưu trữ state này (còn được gọi là metadata topic) được định kỳ rút gọn bằng snapshots để đảm bảo log không thể phát triển vô hạn. Các controller khác trong quorum theo dõi active controller bằng cách phản hồi các sự kiện mà nó tạo ra và lưu trữ trong log của mình.

Một ưu điểm quan trọng của cơ chế này là khả năng **recovery nhanh chóng**. Nếu một node bị tạm dừng do sự kiện phân vùng mạng, nó có thể nhanh chóng bắt kịp các sự kiện bị lỡ bằng cách truy cập log khi tham gia lại. Điều này giảm đáng kể cửa sổ thời gian không khả dụng, cải thiện thời gian recovery trong trường hợp xấu nhất của hệ thống.

## Kết luận

Apache Kafka hoàn toàn có thể hoạt động mà không cần ZooKeeper nhờ vào sự ra đời của chế độ KRaft. Với việc Kafka 4.0 đã chính thức loại bỏ ZooKeeper và chạy mặc định trong chế độ KRaft, tương lai của Apache Kafka trở nên sáng sủa hơn với kiến trúc đơn giản, hiệu suất cao và dễ quản lý.

- Tham khảo thêm:
    - [ZooKeeper-less Kafka](https://softwaremill.com/zookeeper-less-kafka/)
    - [Kafka without Zookeeper](https://www.redpanda.com/guides/kafka-tutorial-kafka-without-zookeeper)
    - [KRaft Overview for Confluent Platform](https://docs.confluent.io/platform/current/kafka-metadata/kraft.html)


# Vai trò của offset trong Kafka là gì ?

Trong Kafka, **offset** đóng vai trò là một **định danh duy nhất cho mỗi bản ghi (record) trong một phân vùng (partition) cụ thể**. Về cơ bản, offset là một giá trị số nguyên, tăng đơn điệu (strictly monotonically-increasing) trong một không gian địa chỉ thưa thớt (sparse address space). Điều này có nghĩa là **mỗi offset kế tiếp luôn lớn hơn offset trước đó**, và **có thể có các khoảng trống** (gap) giữa các offset liên tiếp do các sự kiện như nén (compaction) hoặc giao dịch (transactions).

Dưới đây là các vai trò chính của offset trong Kafka

## Định danh và Khóa chính

Offset cùng với số phân vùng tạo thành tương đương với "khóa chính" của một bản ghi trong Kafka. **Mỗi offset duy nhất xác định một bản ghi trong phân vùng của nó**.

## Theo dõi vị trí đọc của Consumer

Consumer sử dụng offset để theo dõi những bản ghi nào đã được tiêu thụ (consumed). Consumer duy trì trạng thái nội bộ liên quan đến offset của phân vùng mà nó đang đọc.

## Cam kết Offset (Committing Offsets)

Đây là quá trình consumer lưu trữ vị trí đọc hiện tại (offset) trở lại cluster Kafka. Thông thường, consumer sẽ commit offset của bản ghi cuối cùng đã xử lý cộng thêm một (+1). Điều này đảm bảo rằng khi một consumer mới tiếp quản phân vùng (ví dụ: sau khi rebalance hoặc consumer trước đó bị lỗi), nó sẽ bắt đầu xử lý từ bản ghi ngay sau bản ghi cuối cùng đã được xử lý và commit, tránh xử lý lại bản ghi cuối cùng.

## Lưu trữ Offset Consumer

Kafka sử dụng một cách tiếp cận đệ quy để quản lý offset đã commit, sử dụng chính Kafka để lưu trữ và theo dõi các offset này. Khi một offset được commit, group coordinator sẽ xuất bản một bản ghi nhị phân lên topic nội bộ **__consumer_offsets**. Topic này được nén (compacted) trong nền, chỉ giữ lại điểm commit cuối cùng cho mỗi consumer group. Thời gian giữ lại mặc định cho topic này là 7 ngày (từ Kafka 2.0.0 trở đi, trước đó là 24 giờ).


## Kiểm soát Đảm bảo Phân phối (Delivery Guarantees)

Việc kiểm soát thời điểm commit offset mang lại sự linh hoạt trong việc lựa chọn các đảm bảo phân phối.

- Nếu commit offset trước khi xử lý bản ghi, bạn có thể đạt được đảm bảo at-most-once (tối đa một lần). Nếu consumer thất bại sau khi commit nhưng trước khi xử lý, bản ghi đó sẽ bị bỏ qua.
- Nếu commit offset sau khi xử lý bản ghi, bạn có thể đạt được đảm bảo at-least-once (tối thiểu một lần). Nếu consumer thất bại sau khi xử lý nhưng trước khi commit, bản ghi đó sẽ được đọc lại khi consumer mới tiếp quản.
- Mặc định, Kafka consumer tự động commit offset theo khoảng thời gian. Tuy nhiên, hành vi này có thể không đảm bảo at-least-once trong mọi trường hợp và được khuyến nghị tắt tự động commit (enable.auto.commit=false) để commit thủ công bằng commitAsync() để có quyền kiểm soát hoàn toàn.

## Di chuyển trong Log và Xử lý lại

Consumer có thể điều khiển nơi bắt đầu đọc dữ liệu bằng cách sử dụng các phương thức `seek()` để di chuyển đến một offset cụ thể. Điều này cho phép xử lý lại các bản ghi cũ hoặc bỏ qua các bản ghi hiện tại. Các tùy chọn phổ biến bao gồm:
- `--from-beginning` hoặc `seekToBeginning`: **Bắt đầu từ offset đầu tiên** (low-water mark).
- `--to-latest` hoặc `seekToEnd`: **Bắt đầu từ offset cuối cùng** (high-water mark), bỏ qua các bản ghi cũ hơn.
- `--to-offset`: **Bắt đầu từ một offset cụ thể được chỉ định**.
- `--to-datetime` hoặc `offsetsForTimes`: **Tìm offset dựa trên timestamp của bản ghi**.
- `--shift-by`: **Di chuyển offset tiến hoặc lùi một số lượng cố định**.

## Offset trong Phân vùng

Mỗi phân vùng là một log độc lập. Bản ghi được ghi vào cuối log (**append-only**), và được đọc theo thứ tự từ đầu đến cuối. Offset đại diện cho vị trí của bản ghi trong log của phân vùng đó.

## Quản lý Offset khi lỗi hoặc Rebalance

Khi một consumer rời khỏi nhóm hoặc thất bại, các phân vùng được gán lại cho các consumer khác trong cùng nhóm. Consumer mới sẽ sử dụng offset đã commit cuối cùng cho phân vùng đó để tiếp tục xử lý. Nếu không tìm thấy offset đã commit (ví dụ: offset đã cũ hơn thời gian giữ lại của topic **__consumer_offsets**), Kafka sẽ dựa vào thuộc tính **auto.offset.reset** của consumer để quyết định bắt đầu từ đâu (**earliest**, **latest**, hoặc báo lỗi).

## Kết luận

Tóm lại, offset là nền tảng cho cách Kafka quản lý dữ liệu bên trong phân vùng, cho phép consumer theo dõi tiến trình, kiểm soát các đảm bảo phân phối, và di chuyển linh hoạt trong luồng dữ liệu.


# Tầm quan trọng của Replications trong Kafka

Trong Kafka, **replication** (nhân bản) là một tính năng cốt lõi đóng vai trò cực kỳ quan trọng đối với **độ bền (durability)** và **tính sẵn sàng (availability)** của dữ liệu.

Dưới đây là tầm quan trọng của replication trong Kafka 

## Đảm bảo Độ bền Dữ liệu

Replication là cơ chế chính để đảm bảo dữ liệu được ghi vào cluster Kafka sẽ **tồn tại ngay cả khi các broker node bị lỗi**. Dữ liệu được ghi vào nhiều bản sao (replicas), vì vậy việc một bản sao bị lỗi không dẫn đến mất dữ liệu. Số lượng bản sao (replication factor) ảnh hưởng trực tiếp đến khả năng chống mất dữ liệu: replication factor càng cao thì khả năng mất dữ liệu do lỗi của một bản sao bị cô lập càng thấp. Cấu hình **acks=all** hoặc **acks=-1** trên producer, kết hợp với **min.insync.replicas**, đảm bảo rằng bản ghi được coi là **hoàn thành và được xác nhận (acknowledged)** cho producer chỉ sau khi nó đã được nhân bản một cách bền vững đến tất cả các bản sao "in-sync" (**ISR**). Điều này mang lại sự đảm bảo cao nhất về độ bền.

## Đảm bảo Tính sẵn sàng Dữ liệu

Replication đảm bảo rằng dữ liệu sẽ **luôn có thể truy cập được cho các client** ngay cả khi có sự cố xảy ra với một số broker. Khi một broker làm leader cho một phân vùng bị lỗi, một follower trong tập hợp ISR (In-Sync Replicas) có thể được bầu làm leader mới, cho phép các client tiếp tục đọc và ghi vào phân vùng đó. Một replication factor bằng ba có thể cung cấp cả sẵn sàng đọc và ghi, nếu các điều kiện khác được đáp ứng. Cluster có thể xử lý lỗi của một broker riêng lẻ và tiếp tục phục vụ client nhờ replication.

## Đảm bảo tính chịu lỗi (Fault Tolerance)

Replication là một phần quan trọng trong khả năng chống lỗi của Kafka. Bằng cách phân tán các bản sao của cùng một phân vùng trên các broker khác nhau (và lý tưởng là trên các rack vật lý khác nhau thông qua tính năng rack awareness), Kafka có thể chống lại các lỗi broker hoặc thậm chí là lỗi rack.

## Hỗ trợ Khả năng mở rộng (Đọc)

Mặc dù việc ghi (writes) luôn được chuyển đến leader của phân vùng, cơ chế replication đặt nền tảng cho khả năng mở rộng việc đọc (reads). Mặc định, consumer đọc từ leader. Tuy nhiên, việc có nhiều bản sao dữ liệu mở ra khả năng cho các tối ưu hóa, chẳng hạn như [KIP-392](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica) cho phép consumer đọc từ bản sao gần nhất.


## Đảm bảo Nhất quán (Consistency)

Trong mô hình leader-follower của Kafka, replication đảm bảo rằng các bản sao của cùng một phân vùng sẽ duy trì tính nhất quán về mặt tuần tự (sequential consistency). Mọi thay đổi chỉ được thực hiện bởi leader, và các follower sao chép tất cả các thay đổi này theo thứ tự. Sự nhất quán này rất quan trọng để consumer có thể đọc dữ liệu theo thứ tự như khi nó được publish.

## Hỗ trợ Khôi phục Thảm họa (Disaster Recovery)

Mặc dù Kafka không có chiến lược backup truyền thống giống như cơ sở dữ liệu (chụp snapshot hay backup đĩa), việc nhân bản dữ liệu giữa các cluster là cách phổ biến để đảm bảo khả năng khôi phục sau thảm họa. MirrorMaker, một công cụ đi kèm với Kafka, hoạt động như một cặp consumer/producer để đọc dữ liệu từ một cluster nguồn và ghi (nhân bản) nó sang một cluster đích.

## Kết luận

Replication không chỉ là việc **tạo ra các bản sao dữ liệu**, mà là nền tảng cho các đảm bảo về **độ tin cậy** và **khả năng hoạt động liên tục** của Kafka trong môi trường phân tán. Việc cấu hình replication factor và các tham số liên quan một cách phù hợp là rất quan trọng để đáp ứng yêu cầu của ứng dụng về độ bền, tính sẵn sàng và hiệu suất.


# Cách tối ưu hiệu suất trong Kafka

Việc tối ưu hiệu suất (tuning) trong Kafka là rất quan trọng và phức tạp, đòi hỏi sự cân bằng giữa các yếu tố như thông lượng (throughput), độ trễ (latency), độ bền (durability) và tính sẵn sàng (availability)

Dưới đây là các khía cạnh và cách tiếp cận

## Hiểu rõ các Khái niệm Cốt lõi

Trước khi tối ưu hiệu suất, cần hiểu rõ cách Kafka hoạt động bên trong. Các khái niệm như replication (nhân bản), partitions (phân vùng), leader, follower, In-Sync Replicas (ISR), batching (gom nhóm), buffering (bộ đệm), fetching (tìm nạp) là nền tảng.

## Cấu hình Broker

- Cấu hình **Listeners** và **Advertised Listeners** để đảm bảo kết nối chính xác

- **Quotas** (Hạn ngạch): Kafka hỗ trợ quota để giới hạn việc sử dụng tài nguyên của client hoặc người dùng. Điều này rất quan trọng để đảm bảo hiệu suất ổn định trong môi trường multi-tenant bằng cách ngăn chặn một client hoặc người dùng chiếm dụng quá nhiều tài nguyên của broker. Có thể cấu hình quota theo byte rate (`producer_byte_rate`).

## Cấu hình Topic

- **Số lượng Partitions**: Quyết định số lượng phân vùng khi tạo topic. Số lượng phân vùng ảnh hưởng trực tiếp đến khả năng xử lý song song cho cả producer và consumer. Tuy nhiên, việc tăng số lượng phân vùng sau này có thể phá vỡ thứ tự tin nhắn cho các bản ghi có cùng key.

- **Replication Factor**: Số lượng bản sao cho mỗi phân vùng. Yếu tố nhân bản ảnh hưởng trực tiếp đến độ bền và tính sẵn sàng của dữ liệu. Replication factor cao hơn (ví dụ 3) tăng độ bền nhưng có thể tăng độ trễ khi ghi vì leader phải chờ xác nhận từ các follower.

- **min.insync.replicas (ISR)**: Số lượng bản sao "in-sync" tối thiểu (bao gồm leader) mà producer phải chờ xác nhận ghi trước khi coi yêu cầu ghi là thành công, khi **acks** được đặt là **all** hoặc **-1**. Giá trị mặc định là 1, điều này được coi là "khó bền". Nên đặt tối thiểu là 2 để đảm bảo độ bền. Giá trị này ảnh hưởng trực tiếp đến sự đánh đổi giữa độ bền và hiệu suất/độ trễ khi ghi.

- **Retention Policies (Chính sách lưu giữ)**: Kafka có hai chính sách: **deletion** (xóa theo thời gian hoặc kích thước) và **compaction** (nén). Cấu hình chính sách lưu giữ (`log.retention.ms`, `log.retention.bytes`, `log.segment.bytes`,...) ảnh hưởng đến dung lượng đĩa được sử dụng và lượng dữ liệu có sẵn cho consumer, có thể ảnh hưởng đến hiệu suất đọc và chi phí lưu trữ.

- **Reassigning Replicas & Preferred Leader Election**: Các công cụ quản trị cho phép di chuyển các bản sao phân vùng giữa các broker và bầu leader ưu tiên. Việc phân bố đồng đều leader và follower trên các broker là quan trọng để cân bằng tải và tối ưu hiệu suất cluster.

## Cấu hình Producer

- **acks (Acknowledgements)**: Đặt mức độ xác nhận mà producer chờ đợi từ leader. acks=0 có hiệu suất cao nhất nhưng ít bền nhất (có thể mất dữ liệu). acks=1 chờ leader ghi thành công. acks=all hoặc -1 chờ leader và tất cả các bản sao trong ISR ghi thành công, mang lại độ bền cao nhất nhưng độ trễ cao hơn. Đây là tham số quan trọng nhất ảnh hưởng đến sự cân bằng giữa hiệu suất ghi và độ bền dữ liệu.

- **Batching & Buffering**: Producer gom các bản ghi thành các lô (batches) trước khi gửi để tăng hiệu quả. Các tham số như `linger.ms` (thời gian chờ để tạo batch) và `batch.size` (kích thước batch tối đa) ảnh hưởng đến kích thước lô và tần suất gửi, tác động lớn đến thông lượng và độ trễ.

- **Timeouts**: `delivery.timeout.ms` đặt thời gian tối đa để một bản ghi được gửi thành công (bao gồm thời gian chờ batch và thời gian gửi mạng). Nếu bộ đệm đầy hoặc quá trình batching/gửi chậm, bản ghi có thể bị timeout. Để giải quyết, có thể tăng timeout hoặc giảm kích thước bộ đệm (`buffer.memory`). Tuning bộ đệm và timeout là cần thiết để tránh mất dữ liệu do timeout hoặc gây áp lực ngược (backpressure) lên producer.

## Cấu hình Consumer

- **Fetching**: Consumer tìm nạp các lô bản ghi từ broker. Các tham số như `fetch.max.bytes` (kích thước dữ liệu tối đa trong một lần fetch) và `max.partition.fetch.bytes` (kích thước dữ liệu tối đa cho một phân vùng trong một lần fetch) ảnh hưởng đến lượng dữ liệu consumer kéo về mỗi lần. Tuning các tham số này giúp cân bằng giữa hiệu suất tìm nạp tổng thể và sự công bằng giữa các phân vùng.

- **Polling**: Consumer gọi phương thức **poll()** để lấy các bản ghi. Thời gian chờ trong poll() (ví dụ: `Duration.ofMillis(100)`) và `max.poll.interval.ms` (khoảng thời gian tối đa giữa các lần poll trước khi consumer được coi là chết và gây rebalance) ảnh hưởng đến độ trễ đọc và hành vi rebalancing. Tuning các tham số này giúp đảm bảo consumer xử lý kịp thời và tham gia nhóm consumer (consumer group) một cách ổn định.

- **Offset Management**: Cách consumer quản lý offset (vị trí đọc) ảnh hưởng đến độ bền xử lý (at-most-once, at-least-once, exactly-once). `enable.auto.commit` và tần suất auto-commit ảnh hưởng đến việc khi nào offset được lưu. Tuning việc commit offset thủ công hoặc tự động ảnh hưởng đến sự đánh đổi giữa hiệu suất và đảm bảo xử lý chính xác. `auto.offset.reset` xác định hành vi khi không tìm thấy offset đã commit.

## Kết luận

Tối ưu hiệu suất Kafka là một quá trình liên tục đòi hỏi sự hiểu biết sâu sắc về kiến trúc của nó, cấu hình cẩn thận các tham số liên quan đến producer, consumer, topic và broker, dựa trên yêu cầu cụ thể về độ bền, độ trễ và thông lượng, kết hợp với việc thử nghiệm và theo dõi hệ thống.