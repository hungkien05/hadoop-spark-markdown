## Kiến trúc HDFS

### 1, Giới thiệu
* Hadoop Distributed File System (HDFS) là file system kiểu phân tán được thiết kế chạy trên các phần cứng thương mại. Nó có nhiều điểm tương đồng với các file system phân tán khác. Tuy vậy, HDFS có nhiều điểm đặc trưng riêng. 
* HDFS có khả năng chịu lỗi cao và được thiết kế để triển khai trên các phần cứng giá rẻ. HDFS cung cấp thông lượng cao để truy cập dữ liệu và phù hợp với dữ liệu lớn.
* HDFS được khởi đầu trên nền tảng tìm kiếm trên web của Apache Nutch. Project URL: http://hadoop.apache.org/hdfs/.

### 2,  Các giả thiết và mục tiêu.
#### a, Lỗi phần cứng
* Lỗi phần cứng xảy ra ngày càng thường xuyên hơn. 1 hệ thống HDFS có từ vài trăm đến vài nghìn máy, mỗi máy chứa 1 phần dữ liệu. Mỗi thành phần lại có xác suất gặp lỗi là không nhỏ., có nghĩa rằng luôn luôn sẽ có 1 thành phần nào đó gặp lỗi. Vì vậy, phát hiện lỗi, sửa lỗi nhanh và tự động là mục tiêu cốt lõi trong kiến trúc HDFS.

#### b, Truy cập dữ liệu theo luồng.
* Ứng dụng chạy HDFS cần truy cập dữ liệu theo luồng (streaming data access). HDFS được thiết kế để phù hợp với việc xử lý hàng loạt hơn là các tương tác với người dùng. Vì vậy cần tập trung vào cung cấp thông lượng cao để truy cập hơn là giảm độ trễ truy cập dữ liệu.

#### c, Các tập dữ liệu lớn.
* Ứng dụng chạy HDFS  chứa các tập dữ liệu lớn. 1 file HDFS cỡ khoảng vài GB đến vài TB. HDFS cần phải hỗ trợ các file lớn này. Nó cần cung cấp bằng thông dữ liệu lớn và có thể mở rộng đến hằng trăm node trong 1 cluster. Nó nên hỗ trỡ khoảng 10 triệu file trong 1 hệ thống HDFS.

#### d, Mô hình đơn giản và dễ hiểu.
* Ứng dụng HDFS  cần kiểu truy cập viết một lần đọc mọi nơi (write once read many). 1 file khi đã tạo ra, viết và đóng lại rồi thì sẽ không được thay đổi nữa. Giả thuyết này giúp dữ liệu mạch lạc, dễ hiểu hơn và tăng thông lượng truy cập. Ứng dụng như MapReduce hay web crawler rất phù hợp với mô hình kiểu này.

#### e, Di chuyển tính toán tiết kiệm hơn di chuyển dữ liệu.
* Một yêu cầu tính toán sẽ xử lý hiệu quả hơn nếu nó ở gần phần dữ liệu phép tính đó cần. Điều này càng đúng hơn khi phần dữ liệu đó rất lớn. Phương pháp này giúp giảm nghẽn mạng và tăng tổng thông lượng của hệ thống. HDFS cung cấp interface cho ứng dụng để di chuyển chính ứng dụng đến gần với dữ liệu hơn.

#### f, Tính di dộng  của phàn cứng không đồng nhất và phần mềm.
* HDFS được thiết kế để dễ dàng di chuyển từ nền tảng này sang nền tảng khác. Điều này tạo điều kiện để áp dụng HDFS rộng rãi như một lựa chọn phổ biến cho đa số ứng dụng.

### 3, Namenode và Datanode
* HDFS có kiến trúc chủ/nô.  1 HDFS chứa 1 NameNode duy nhất, đóng vai trò server quản lý không gian file và truy cập của các client. 
* Thông thường, 1 file sẽ chia nhỏ thành 1 hoặc nhiều block. Những block này sẽ được chứa trong các DataNode. DataNode có nhiệm vụ quản lý lưu trữ của node đó và nhận lệnh từ NameNode để tạo, xóa hoặc nhân bản block chứa trong DataNode đó.
* NameNode và DataNode là 1 phần của phần mềm thiết kể để chạy trên các phần cứng thương mại. Các phần cứng này chủ yếu dùng hệ điều hành GNU/Linux. HDFS được build bằng Java, bất cứ máy nào hỗ trợ Java thì đều có thể là NameNode hoặc DataNode. Cũng nhờ tính di động của  Java mà HDFS có thể chạy trên nhiều loại máy khác nhau. Thông thường sẽ có 1 máy chỉ để dùng để chạy phần mềm NameNode. Các máy còn lại trọng cụm sẽ là DataNode.
* Việc chỉ có 1 NameNode giúp tối giản kiến trúc của hệ thống. NameNode chỉ là nơi chứa và xử lý tất cả metadata của HDFS. Hệ thống đảm bảo rằng dữ liệu người dùng sẽ không bao giờ đi qua NameNode.

### 4, Namespace của hệ thống 
* HDFS hỗ trợ quản lý file theo cấp bậc (hierarchical). Người dụng hoặc ứng dụng có quyền tạo thư mục và lưu file trong thư mục đó. Phương pháp namespace này cũng tương tự đa số các file system khác. 
* HDFS chưa hỗ trợ các tính năng như user quotas, hard links hay soft links. Dù vậy, HDFS cũng không cấm implement các tính năng vào hệ thống.
* NameNode sẽ duy trì namespace. Bất cứ thay đổi nào về namespace sẽ được ghi lại vào NameNode. Ứng dụng có thể chỉ rõ số bản sao của 1 file được lưu trong HDFS. Thông tin số bản sao này cũng được lưu vào NameNode.

### 5, Nhân bản data
* HDFS được thiết kế để lưu trữ các file cực lớn trên cụm máy lớn. Mỗi file sẽ được lưu dưới dạng các block con (các block có kích thước bằng nhau trừ block cuối). Các block sẽ được nhân bản để tăng khả năng chịu lỗi. Người dùng hay ứng dụng có thể tùy chỉnh kích thước block và số nhân bản block của từng file.
* NameNode nhận Heartbeat và BlockReport từ mỗi DataNode. Heartbeat cho biết DataNode có đang chạy ổn hay không. Blockreport chứa danh sách các block của DataNode đó.

#### a, Vị trí của bản sao
* Đặt các bản sao ở đâu ảnh hướng lớn đến độ tin cậy và hiệu suất của HDFS. Tối ưu vị trí bản sao là khác biệt lớn của HDFS với các file system khác.  Phương pháp tại thời điểm viết bài đang dùng là first effort, tức đặt bản sao vào rack đầu tiên tìm được.
* Rack là 1 tập các DataNode gần nhau. Phương pháp tối ưu mà tác giả hướng đến là rack awareness (nhận thức được vị trí các rack). NameNode sẽ gán rack_id cho từng DataNode thông qua Hadoop Rack Awareness. Sau đó sẽ đặt các bản sao ở các rack khác nhau.  Nó giúp ngăn chặn việc mất dữ liệu khi 1 rack nào đó gặp lỗi và cho phép băng thông trên nhiều rack khi đọc dữ liệu. Phương pháp cũng giúp phân bố các bản sao đều nhau, cân bằng tải giữa các thành phần. Tuy vậy, phương pháp này tăng chi phí khi viết dữ liệu vì cần phải di chuyển các block giữa các rack.
* Trong trường hợp phổ biến, khi hệ số nhân bản là 3, rack aware sẽ đặt 1 bản sao vào 1 node của rack hiện tại, 1 bản sao vào rack khác (remote rack), và bản sao cuối cùng vào chính rack khác đấy, nhưng khác node với bản sao thứ 2.

#### b, Lựa chọn bản sao 
* Để tối thiểu băng thông toàn cục và độ trễ khi đọc (read latency), HDFS cố gắng thực request đọc từ bản sao gần với reader nhất. Nếu có 1 bản sao cùng rack với reader thì nó sẽ được ưu tiên để thực hiện request đọc. Nếu cụm máy HDFS nằm trên nhiều data center, thì bản sao nào ở cùng data center với reader sẽ được ưu tiên.

#### c, Safemode
* Khi khởi động NameNode sẽ tiến vào trạng thái đặc biệt gọi là Safemode. Khi đó quá trình nhân bản các block sẽ không được xảy ra. Namenode nhận Heartbeat và Blockreport từ DataNode, chứa danh sách các block ở trong DataNode đó. 
* Một block được xem là an toàn nếu NameNode báo rằng đã có đủ tối thiểu số bản sao của block này trên hệ thống. Nếu kiểm tra đủ một lượng nhất định các block ( tùy chỉnh được số block kiểm tra), NameNode sẽ thoát khỏi Safemode. Những block không an toàn sẽ được ghi lại để nhân bản sau đó.

### 6, Khă năng chống chịu của metadata.
* HDFS namespace được lưu bởi NameNode. NameNode sử dụng 1 log gọi là EditLog để luôn ghi lại các thay đổi xảy ra với metadata. Ví dụ, tạo file mới hay thay đổi hê số nhân bản sẽ đều được EditLog ghi chép lại. Toàn bộ namespace của file system, bao gồm các ánh xạ từ block đến file và các thuộc tính đều được lưu tại 1 file gọi là FsImage.
* NameNode giữ 1 image của toàn bộ namespace của hệ thống và các file trong memory. Image này được thiết kế nhỏ gọn để phù hợp với 4GB RAM của NameNode. Khi NameNode khởi động, nó sẽ load FsImage và EditLog từ ổ đĩa vào 1 phiên bản ổn định FsIamge mới. Sau đó EditLog cũ có thể được xóa đi. Quá trình này gọi là checkpoint, và chỉ xảy ra khi NameNode khởi động.
* DataNode lưu dữ liệu HDFS trong các file ở file system của chính node đó. Nó lưu từng block của dữ liệu trong từng file riêng. DataNode tất nhiên không tạo tất cả các file trong cùng 1 thư mục. Nó dùng heuristic để tới ưu số file trong từng thư mục và tạo thư mục mới 1 cách hợp lý. Khi DataNode khởi động, nó quét toàn bộ file system của nó, tạo danh sách các block ứng với file nào rồi gửi báo cáo này cho NameNode. Báo cáo này chính là Blockreport.

### 7, Các giao thức giao tiếp
* Các giao thức giao tiếp của HDFS đều ở trên tầng TCP/IP. Client tạo ra 1 kết nối đến 1 cổng TCP của NameNode qua giao thức ClientProtocol DataNode kết nối với NameNode qua giao thức DataNode Protocol. Cả 2 giao thức này đều dựa trên Remote Procedure Call (RPC). Theo thiết kế, NameNode không bao giờ khởi tạo bất cứ RPC nào cả, nó chỉ trả lời các request RPC từ DataNode và client.

### 8, Sự chắc chắn
* Nhiệm vụ chính của HDFS là lưu trữ dữ liệu 1 cách tin cậy kể cả khi gặp sự cố. Ba sự cố thường gặp là lỗi NameNode, lỗi DataNode và network partition - phân vùng mạng.
#### a, Lỗi ổ cứng, heartbeats và nhân bản lại (re-replication)
* Mỗi DataNode gửi tin nhắn Heartbeat đến NameNode định kì. Phân vùng mạng khiến cho 1 phần DataNode mất kết nối với NameNode. NameNode phát hiện vấn đề này qua việc không nhận được tin nhắn Heartbeat. Khi đó, NameNode cho rằng DataNode này hỏng và sẽ không gửi đến yêu cầu nào nữa. NameNode sẽ truy xuất lại các block năm trong DataNode bị hỏng và sẽ nhân bản lại chúng nếu cần.

#### b, Tái cân bằng cluster
* Kiến trúc HDFS tương thích với phương pháp tái cân bằng dữ liệu. Phương pháp này di chuyển dữ liệu từ 1 Datanode sang Datanode khác nếu còn trống. Trong trường hợp có yêu cầu đột biến cho 1 file nào đó, phương pháp này sẽ linh hoạt và tạo ra ,các nhân bản cần thiết và tái cân bằng dữ liệu trong cluster. Tuy vậy, tại thời điếm viết bài, phương pháp này chưa được triển khai.

#### c, Sự chắc chắn của dữ liệu.
* Việc lấy dữ liệu từ 1 block của DataNode có thể gặp lỗi. Nguyên nhân có thể bắt nguồn từ lỗi ổ cứng, lỗi mạng hoặc bug của phần mềm. 
* Phần mềm phía client của HDFS triển khai kiểm tra checksum với các file HDFS. Khi tạo 1 file HDFS mới, client sẽ tính ra giá trị checksum của từng block của file đó và lưu vào 1 file ẩn của chính namespace đó. Khi client quay lại lấy nội dung file này, nó sẽ kiểm tra xem dữ liệu lấy về cũng khớp với checksum đã tính hay không. Nếu không, nó sẽ tìm tới các nhân bản của block đó.

#### d, Lỗi ổ cứng metadata.
* FsImage và EditLog là 2 cấu trúc dữ liệu trung tâm của HDFS. Xảy ra lỗi ở 2 file này khiên HDFS không thể hoạt động. Vì vậy, NameNode sẽ được cấu hình để hỗ trợ các bản sao của 2 file này. Bất kì cập nhật nào liên quan đến 2 file này sẽ được cập nhật đồng thời ở các bản sao.
* Nếu NameNode gặp lỗi, thì sẽ cần phải có can thiệp thủ công từ bên ngoài.

#### e, Snapshot
* Snapshot hỗ trợ lưu lại bản sao của dữ liệu ở 1 thời điểm nhất đinh. Một ứng dụng của Snapshot là giúp roll back lại 1 hệ thống HDFS gặp lỗi về thời điểm trước đó khi chưa gặp lỗi.
* HDFS chưa hỗ trợ Snapshot tại thời điểm viết bài báo này.

### 9. Tổ chức dữ liệu
#### a, Block 
* Các ứng dụng tương thích với HDFS đều phải xử lý với dữ liệu lớn. 1 block điển hình của HDFS có dung lượng 64MB. Vì vậy, 1 file HDFS sẽ được chia nhỏ thành các phần 64MB, và nếu có thể, mỗi phần sẽ được lưu vào 1 DataNode.

#### b,Staging
* Yêu cầu tạo file mới của client sẽ không được gửi NameNode ngay lập tức. Client sẽ đưa dữ liệu của file vào 1 file local tạm thời. Khi file local này có dung lượng lớn hơn 1 block, client sẽ kết nối đến NameNode. NameNode sẽ đưa tên file này vào file system và cấp 1 block cho nó. NameNode trả lời client với định dang DataNode và địa chỉ của block đó.
* Phương pháp này được triển khai sau những cân nhắc kĩ càng của các ứng dụng chạy HDFS. Các ứng dụng này viết file theo luồng (streaming). Nếu client viết 1 file remote mà không dùng đến buffer của chính client đó, tốc độ của mạng và nghẽn mạng sẽ ảnh hưởng nghiêm trọng.

#### c, Nhân bản kiểu Pipeline.
* Cách client viết file trong hệ thông HDFS được miêu tả ở phần trên. Giả sử file này có hệ số nhân bản bằng 3. Khi local file có đủ dung lượng bằng 1 block, client nhận lại danh sách DataNode từ NameNode. Client sẽ chuyển dữ liệu block đến DataNode đầu tiên. DataNode này nhận dữ liệu theo từng phần nhỏ (4KB), viết vào local repo của nó và đồng thời chuyển từng phần này đến DataNode tiếp theo. Các DataNode kế tiếp cũng làm tương tự để tạo ra các nhân bản của block.

### 10, Tiếp cận
* HDFS có thể được truy cập nhiều cách khác nhau. Cách tự nhiên nhất là dùng Java API cho các ứng dụng. Các cách khác có thể kể đến là dùng FS Shell, DFSAdmin hoặc dùng thẳng giao diện của trình duyệt.

### 11, Lấy lại không gian bộ nhớ
#### a, Xóa và hủy xóa (undelete) file.
* Khi một file bị xóa bởi user, nó sẽ được chuyển tới thư mục /trash, chứ chưa bị xóa hẳn khỏi HDFS. File này có thể được restore miễn là nó vẫn ở trong /trash. File này ở trong /trash trong 1 khoảng thời gian nhất định, sau đó nó sẽ được xóa khỏi /trash và biến mất hoàn toàn. Các block được gắn với file này sẽ được giải phóng hoàn toàn. Chú ý rằng sẽ có độ trễ từ khi xóa file đến khi giải phóng được bộ nhớ trong HDFS.

#### b,  Giảm hệ số nhân bản.
* Khi giảm hệ số nhân bản, NameNode sẽ chọn các nhân bản thừa ra để xóa. Heartbeat kế tiếp sẽ chứa thông tin này. Các block và không gian bộ nhớ sẽ được giải phóng. Và cũng như xóa file, sẽ có độ trễ từ khi xóa đến khi giải phóng bộ nhớ.

### 12, Tài liệu tham khảo.
* HDFS Java API: http://hadoop.apache.org/core/docs/current/api/
* HDFS source code: http://hadoop.apache.org/hdfs/version_control.htm 
