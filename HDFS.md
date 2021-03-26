# HDFS Architecture Guide

## 1. Giới thiệu
HDFS là một hệ thống tập tin phân tán (*distributed file system*). Nó có nhiều điểm chung với các hệ thống hiện thời, tuy nhiên lại có ưu điểm vượt trội khi được thiết kế để vận hành trên hệ thống phần cứng thương mại (rẻ tiền và phổ biến). Nó cung cấp khả năng kiểm soát lỗi cao và truy cập thông lượng cao, phù hợp với các ứng dụng có tập dữ liệu quy mô lớn.

## 2. Vấn đề và mục tiêu
### 2.1 Lỗi phần cứng (Hardware Failure)
Lỗi phần cứng là việc phổ biến, không phải ngoại lệ. Một hệ thống HDFS có thể bao gồm hàng trăm nghìn máy chỉ, mỗi thành phần lại có một xác xuất hỏng hóc đáng kể. Vì thế phát hiện và khôi phục lỗi một cách nhanh chóng, tự động là mục tiêu cốt lõi của HDFS.

### 2.2 Truy cập dữ liệu
Các ứng dụng chạy trên HDFS cần có khả năng truy cập dữ liệu trực tuyến. HDFS được thiết kế để thiên về xử lý hàng loạt, chú trọng vào thông lượng truy cập dữ liệu hơn là độ trễ trong quá trình truy cập.

### 2.3 Quy mô dữ liệu lớn
Các ứng dụng chạy trên HDFS đều có tập dữ liệu lớn. Thường mỗi file đều lớn từ GB đến TB. HDFS được điều chỉnh để hỗ trợ các tệp lớn, cung cấp băng thông dữ liệu cao, hỗ trợ hàng chục triệu tệp trong một mô hình.

### 2.4 Mô hình
Các ứng dụng HDFS cần một mô hình *write-once-read-many*, một file khi đã được tạo, viết và đóng thì không cần chỉnh sửa. Mô hình này giúp đơn giản hóa và tăng thông lượng truy cập dữ liệu. Một ứng dụng MapReduce hoặc ứng dụng thu thập thông tin sẽ phù hợp với mô hình này.

### 2.5 "Di chuyển việc tính toán rẻ hơn di chuyển dữ liệu"
Một mô hình tính toán, xử lý sẽ hoạt động hiệu quả hơn nếu nó được thực hiện gần dữ liệu mà nó cần xử lý, đặc biệt trong trường hợp quy mô dữ liệu khổng lồ. Việc này giảm tắc nghẽn mạng và tăng thông lượng của hệ thống. HDFS cung cấp những nền tảng cho ứng dụng có thể di chuyển tới gần nơi lưu trữ dữ liệu.

### 2.6 Khả năng di chuyển trên các nền tảng không đồng nhất
HDFS được thiết kế để có thể chuyển đổi từ nền tảng này sang nền tảng khác, giúp nó có thể áp dụng rộng rãi.

## 3. NameNode và DataNode
HDFS có cấu trúc master - slave. Mỗi HDFS cluster sẽ có một NameNode, một master quản lý hệ thống namespace và điều chỉnh quyền truy cập vào tệp của client. Bên cạnh đó sẽ có một số lượng các DataNode, thường là mỗi một DataNode tương ứng với một node trong cluster. DataNode sẽ quản lý bộ nhớ gắn liền với mỗi node đấy.
Thường mỗi file sẽ được chia thành một hoặc nhiều block, các block được lưu trữ trong một tập các DataNode. NameNode sẽ vận hành hệ thống namespace như mở, đóng, đặt lại tên cho file và thư mục. Nó cũng xác định ánh xạ từ các block đến DataNode. DataNode sẽ có nhiệm vụ xử lý các yêu cầu đọc, ghi từ client, tạo, xóa, lặp các block dựa vào chỉ dẫn từ NameNode.

## 4. File System NameSpace
HDFS hỗ trợ một mô hình tổ chức file phân cấp truyền thống. Một người dùng hoặc ứng dụng có thể tạo ra một thư mục, lưu trữ file trong những thư mục đó, có thể tạo, xóa, đổi tên file, di chuyển file từ thư mục này sang thư mục khác.
NameNode sẽ quản lý hệ thống namespace, các thông tin hay bất cứ thay đổi nào trên namespace đều được ghi lại bởi NameNode.

## 5. Data Replication
HDFS lưu trữ mỗi file là một dãy các block, tất cả các block (trừ block cuối) đều có chung kích thước. Các block được nhân bản để đáp ứng khả năng chịu lỗi, kích cỡ size và các yếu tố của bản sao đều được xác định tại thời điểm tạp file và có thể thay đổi sau này.
NameNode đưa ra tất cả các quyết định liên quan đến việc nhân bản các block. Nó nhận một Heartbea và Blockreport từ mỗi DataNode trong cluster định kì. Heartbeat cho biết DataNode đang hoạt động bình thường, còn Blockreport sẽ chứa danh sách tất cả các block trên một DataNode.

### 5.1 Vị trí của bản sao
Vị trí của bản sao là rất quan trọng với độ tin cậy và hiệu suất của HDFS. Tối ưu hóa yếu tố này giúp HDFS có lợi thế so với các DFS khác.
Các mô hình HDFS chạy trên cluster thường được phân bố trên nhiều phân vùng khác nhau. Một chính sách đơn giản là lưu trữ các bản sao trên một phân vùng khác, tránh cho việc mất mát khi toàn bộ một phân vùng gặp sự cố. Bên cạnh đó việc này cũng giúp cân bằng tải khi có thành phần gặp sự cố, đánh đổi lại là tăng chi phí ghi dữ liệu khi phải truyền các block tới nhiều phân vùng khác nhau.
Ví dụ trong trường hợp số lượng bản sao là 3 thì chính sách xác định vị trí sẽ lưu một bản sao trên một node thuộc phân vùng hiện tại (local), hai node thuộc một phân vùng khác (remote).

### 5.2 Lựa chọn bản sao
HDFS sẽ chọn đọc bản sao lưu trữ gần người đọc nhất để giảm tải và độ trễ. Bản sao local sẽ luôn được ưu tiên so với các bản sao remote.

### 5.3 Safemode 
Khi bắt đầu, NameNode có một trạng thái đặc biệt được gọi là Safemode, việc nhân bản các block sẽ không diễn ra. NameNode sẽ nhận Heartbeat và Blockreport từ DataNode, trong đó sẽ bao gồm thông tin về số lượng bản sao tối thiểu của mỗi block. Một block được xem là an toàn khi NameNode đã kiểm tra được các bản sao đã đạt đủ số lượng tối thiểu hay chưa, sau đó NameNode sẽ thoát khỏi Safemode.

## 6. Tính ổn định của Metadata
NameNode dùng một nhật ký được gọi là EditLog để ghi lại mọi thay đổi diễn ra với hệ thống Metadata. Toàn bộ namespace, bao gồm cả ánh xạ của block tới file và cả thông tin hệ thống được lưu vào một file gọi là FsImage (được lưu trong local file system của NameNode).
NameNode cần lưu giữ toàn bộ các thông tin này trong bố nhớ, vì thế metadata cần phải gọn nhẹ cho giới hạn 4GB RAM của NameNode. Khi NameNode bắt đầu hoạt động, nó đọc FsImage và EditLog từ ổ đĩa, sau đó áp dụng những thay đổi trên EditLog lên FsImage và lưu một FsImage mới vào ổ đĩa. Nó có thể cắt bớt đi phần EditLog cũ vì các thay đổi đã được áp dụng lên FsImage. Quá trình này được gọi là checkpoint, một checkpoint chỉ diễn ra khi NameNode được khởi động.

## 7. Giao thức giao tiếp
Tất cả các giao thức giao tiếp của HDFS đều dựa vào giao thức TCP/IP. Client thiết lập một kết nối qua một cổng TCP trên NameNode, giao tiếp bằng ClienProtocol với NameNode. DataNode giao tiếp với NameNode bằng DataNode Protocol.

## 8. Xử lý lỗi
Mục tiêu căn bản nhất của HDFS là lưu trữ dữ liệu an toàn kể cả khi xuất hiện lỗi. 3 loại lỗi phổ biến nhất xảy ra với NameNode, DataNode và phân vùng mạng.
### 8.1 Data Disk Failure, Heartbeats and Re-Replication
Mỗi DataNode sẽ gửi Heartbeat định kì tới NameNode. Phân vùng mạng có thể khiến cho một phần của DataNode mất kết nối với NameNode, NameNode có thể phát hiện ra dựa vào một Heartbeat bị thiếu. NameNode sẽ đánh dấu DataNode đó đã chết (*dead*), không chuyển request nào tới DataNode đó và toàn bộ dữ liệu trên đó đều không thể truy cập nữa. NameNode sẽ kiểm tra xem liệu có block nào bị mất và trở thành thiếu số lượng bản sao đã định và tạo thêm các bản sao nếu cần.

### 8.2 Tái cân bằng cluster
HDFS tương thích với các kế hoạch tái cân bằng dữ liệu. Một kế hoạch có thể chuyển dữ liệu từ một DataNode này sang DataNode khác nếu không gian lưu trữ còn trống trên đó còn quá ít. Hoặc trong trường hợp nhu cầu truy cập vào một file nào đó cao bất thường, nó có thể tạo thêm nhiều bản sao hơn và tái cân bằng dữ liệu trong cluster.

### 8.3 Toàn vẹn dữ liệu
Quá trình lấy một block từ DataNode có thể gặp sự cố vì lỗi của thiết bị lưu trữ, lỗi mạng, chương trình có bug. HDFS cho phép người dùng cài đặt checksum cho các nội dung trong HDFS file. Khi người dùng tạo một file, nó sẽ tính toán checksum của mỗi block của file và lưu trữ checksum trong một file ẩn riêng biệt trong cùng namespace. Khi clinet truy cập nội dung của file, nó sẽ kiểm tra dữ liệu được lấy ra có khớp với checksum đã được lưu hay không. Nếu không, client có thể lấy dữ liệu từ bản sao khác của block đó.

### 8.4 Metadata Disk Failure
FsImage và EditLog là những file quan trọng, nếu có sự hư hỏng với những file này, HDFS sẽ không thể hoạt động. Vì vậy NameNode sẽ lưu trữ nhiều bản sao của hai file này, bất cứ update nào sẽ đều được đồng bộ cho tất cả các bản sao. 
Riêng với trường hợp NameNode machine gặp lỗi, sẽ cần đến sự can thiệp thủ công.

### 8.5 Snapshots
Snapshot hỗ trợ lưu trữ bản sao của dữ liệu tại một thời điểm nào đó. Nó giúp hệ thống đang gặp trục trặc có thể quay về một trạng thái ổn định nào đó trước đây.

## 9. Tổ chức dữ liệu
### 9.1 Data Block
HDFS được thiết kế cho các ứng dụng có tập dữ liệu quy mô rất lớn, ghi dữ liệu một lần nhưng cần đọc dữ liệu nhiều lần với điều kiện tốc độ nhanh. Mỗi block thường có kích thước 64MB.

### 9.2 Hoạt động
Khi client yêu cầu tạp một file mới, client không ngay lập tức làm việc với NameNode mà dữ liệu được lưu vào các temporary local file. Khi dữ liệu tích lũy đủ một block, client mới kết nối với NameNode, NameNode sẽ lưu trữ block, các thông tin liên quan và đưa ra phản hồi cho người dùng về thông tin của DataNode và vị trí của block. Nếu file đã đóng mà vẫn còn dữ liệu ở trong temporary local file thì phần đó sẽ được chuyển vào DataNode.

### 9.3 Nhân bản tuần tự
Khi người dùng ghi một file dữ liệu mới và được đẩy vào block, client sẽ nhận được danh sách các DataNodes từ NameNode, bao gồm cả DataNode sẽ chứa bản sao của block đó. Sau khi dữ liệu được ghi vào block của DataNode đầu tiền, nó sẽ chuyển dữ liệu sang cho DataNode thứ 2 trong danh sách, rồi lần lượt cứ như thế sang DataNode thứ 3...

## 10. Khả năng tiếp cận
HDFS có thể tiếp cận từ nhiều ứng dụng bằng nhiều cách khác nhau. HDFS cung cấp một Java API cho các ứng dụng, ngoài ra còn có cả HTTP browser.

## 11. Không gian bộ nhớ
### 11.1 Xóa file
Khi một file được xóa bởi người dùng hoặc ứng dung, nó không ngay lập tức bị xóa khỏi HDFS mà được chuyển vào thư mục `/trash`. File có thể được khôi phục khi nó vẫn ở trong thư mục này, và một file sẽ ở trong thư mục `/trash` một khoảng thời gian nhất định (có thể cài đặt được theo mong muốn, mặc định là 6h). Sau khoảng thời gian đó, NameNode sẽ xóa file ra khỏi namespace và giải phóng các block.

### 11.2 Giảm số lượng bản sao
Sau khi giảm số lượng bản sao, NameNode sẽ chọn các bản sao dư thừa có thể xóa. Heartbeat tiếp theo sẽ cho biết thông tin đến cho DataNote. DataNote sẽ xóa block tương ứng.