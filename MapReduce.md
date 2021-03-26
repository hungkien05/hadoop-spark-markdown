# MapReduce: Simplified Data Processing on Large Clusters

## 1. Giới thiệu
Vấn đề của việc xử lý khối lượng dữ liệu khổng lồ là làm thế nào để song song hóa, phân bổ dữ liệu, xử lý lỗi, thực thi lượng lớn code để có thể hoàn thành việc xử lý trong khoảng thời gian chấp nhận được. Vì thế tác giả của paper đã thiết kế ra một thư viện MapReduce có thể thực thi những tính toán đơn giản và giấu đi những chi tiết phức tạp như kiểm soát lỗi, song song hóa, phân bổ dữ liệu...  

## 2. Model
Một số ví dụ cụ thể để áp dụng MapReduce:
- Thống kê tần suất truy cập URL:
    - Map: xử lý log của trang web và có output <URL,1>
    - Reduce: lấy tổng của các cặp <URL,1> có chung key URL và có output <URL,total count>
- Web-Link Graph:
    - Map: output <target,source> là các cặp với target là URL được tìm thấy trong trang source
    - Reduce: đầu ra là các cặp <target,*list*(source)>

## 3. Implementation
Việc triển khai MapReduce phụ thuộc vào môi trường triển khai, trong phần này sẽ tập trung vào môi trường của Google.
### 3.1 Tổng quan
Sau đây sẽ miêu tả tổng quan quá trình một quá trình MapReduce được triển khai trong môi trường của Google:
1. Thư viện MapReduce trong chương trình của người dùng chia file đầu vào thành M phần, thường là 16-64 MB mỗi phần. Sau đó bắt đầu các bản copy của chương trình trên cluster
2. Một bản copy là master, những bản còn lại là worker được giao việc bởi master. Có M map tasks và R reduce tasks, master sẽ chọn một số worker và giao cho mỗi worker một map task hoặc reduce task
3. Worker được giao map task sẽ đọc đầu vào từ các phần input đã được chia. Thực thi Map fucntion và lưu cặp key value kết quả vào buffer
4. Sau đó các cặp này được viết vào local disk, chia thành R phần và master có nhiệm vụ giao lại cho reduce workers
5. Reduce worker sẽ đọc các phần được giao và nhóm, sắp xếp các cặp theo key.
6. Worker thực thi Reduce fucntion và đầu ra được thêm vào cuối một file output của riêng phần reduce đó.
7. Sau khi hoàn thành xong tất cả các task, master sẽ trả kết quả về cho người dùng.

### 3.2 Cấu trúc dữ liệu của Master
Master sẽ lưu trữ trạng thái của mỗi map task và reduce task (chưa được thực thi, đang thực thi, đã hoàn thành) và identit của các worker. Master cũng lưu vị trí và lích thước của các file đầu ra khi map task đã hoàn thành.

### 3.3 Hệ thống chịu lỗi
**Worker Failure**

Master sẽ ping mọi worker định kỳ, nếu sau một khoảng thời gian không có phản hồi, master sẽ đánh dấu worker đấy không còn hoạt động và toàn bộ map task được worker này thực hiện sẽ được reset lại trạng thái là *chưa được thực thi*.
Map task phải thực thi lại do nó được lưu trên local disk và không thể truy cập. Còn với reduce task do được ghi vào global file system nên không cần thực thi lại.

**Master Failure**

Master sẽ lưu lại checkpoint định kỳ, khi một master gặp lỗi, một bản sao sẽ được bắt đầu lại từ trạng thái checkpoint cuối cùng.

### 3.4 Locality
Băng thông mạng là loại tài nguyên khá khan hiếm trong môi trường, vì thế tác giả tận dụng lợi thế của việc dữ liệu đầu vào được lưu trên local disk. Master sẽ lấy thông tin về vị trí lưu trữ của các file và lập lịch cho các task. Việc chạy một MapReduce lớn sẽ không tốn quá nhiều băng thông khi dữ liệu được đọc ở local.

### 3.5 Chi tiết về task
Map phase được chia thành M phần, Reduce phase được chia thành R phần. Cả M và R nên lớn hơn nhiều so với số lượng worker. Việc mỗi worker thực hiện nhiều task khác nhau sẽ cải thiện cân bằng tải, tăng tốc việc phục hồi nếu có một worker bị lỗi.
Thường R được quyết định bởi người dùng dựa vào mong muốn bao nhiêu file output của người dùng. Vì thế M thường được chọn sao cho mỗi task có khoảng 16 - 64 MB dữ liệu đầu vào và R thường là một bội số nhỏ của số lượng worker sẽ được dùng.
VD: M = 200 000, R = 5 000, 2 000 worker.

### 3.6 Backup Task
Một nguyên nhân phổ biến dẫn đến tổng thời gian thực thi kéo dài là một worker nào đó mất một khoảng thời gian dài bất thường để hoàn thành một vài task cuối cùng, nguyên nhân này được gọi là *straggler*. Staggler có thể sinh ra từ nhiều nguyên nhân, ví dụ một ổ đĩa chất lượng không tốt có thể liên tục gặp phải lỗi khi đọc và giảm tốc độ từ 30 xuống còn 1MB/s.
Để giải quyết vấn đề, khi một quá trình MapReduce sắp hoàn thành, master sẽ backup những *in-progress* task còn lại. Các task này sẽ được đánh dấu đã hoàn thành khi một trong 2 bên hoàn thành.

## 4. Refinements

### 4.1 Hàm phân vùng (Partitioning Function)
Người dùng sẽ xác định số lượng reduce task/output file mà họ mong muốn (R). Để có thể phân vùng dữ liệu, cần dùng đến một hàm phân vùng áp dụng cho key. Hàm phân vùng mặc định là hash (VD: `hash(key) mod R`). Với trường hợp output key là URL thì có thể áp dụng `hash(Hostname(urlkey)) mod R`, hàm này sẽ cho những URL từ chung một host vào một output file.

### 4.2 Thứ tự
Với một phân vùng được cho, các cặp key - value cần được xử lý theo thứ tự tăng dần của key. Việc đảm bảo thứ tự này sẽ tạo ra một output file được sắp xếp có thứ tự với mỗi phân vùng, hỗ trợ cho việc truy cập ngẫu nhiên bằng key hoặc giúp dữ liệu đã được sắp xếp thứ tự.

### 4.3 Hàm kết hợp (Combiner Function)
Trong một số trường hợp thì có sự lặp lại đáng kể giữa các key được sinh ra bởi mỗi map task (VD: đếm số lần xuất hiện của mỗi từ trong văn bản sẽ sinh ra hàng trăm, hàng nghìn cặp <the, 1>). Vì vậy người dùng có thể tạo một hàm kết hợp nhằm hợp nhất dữ liệu trước khi gửi qua network.
Hàm kết hợp sẽ được thực thi trên mỗi worker thực hiện map task. Về cơ bản hàm kết hợp và Reduce khá giống nhau, chỉ khác nhau về output của mỗi hàm.

### 4.4 Input, Output Types
MapReduce không hỗ trợ đọc dữ liệu đầu vào dưới nhiều định dạng. Người dùng có thể đọc được những dạng inout mới bằng các cài đặt một *reader* đơn giản.

### 4.5 Tác dụng phụ

### 4.6 Bỏ qua các bản ghi không tốt
Thường sẽ có lỗi trong code của người dùng khiến cho quá trình MapReduce không thể hoàn thành. Một vài trường hợp sẽ cần fix bug, nhưng đôi khi việc này là bất khả thi (VD như lỗi ở trong thư viện từ bên thứ 3). Trong những trường hợp như vậy thì việc bpr qua một vài bản ghi là chấp nhận được để có thể tiếp tục quá trình.

### 4.7 Thực thi cục bộ
Việc debug trong hàm Map hay Reduce sẽ rất khó khăn khi mà nó được triển khai trên một thế thống phân tán có hàng nghìn máy tính và các quyết định được đưa ra bởi master. Để có thể tạo điều kiện cho việc debug, kiểm thử quy mô nhỏ thì cần một thư viện thực thi tuần tự tất cả các nhiệm vụ trên máy cục bộ.

### 4.8 Thông tin trạng thái
Master chạy một server và xuất một bản báo cáo trạng thái cho người dùng. Báo cáo sẽ cho biết về tiến trình thực thi, ví dụ như bao nhiêu task đã được hoàn thành, số lượng task đang được xử lý, những worker gặp lỗi... Dựa vào đó người dùng có thể dự đoán thời gian hoàn thành.

### 4.9 Bộ đếm
MapReduce cung cấp cơ chế bộ đếm để đếm số lần xuất hiên của nhiều sự kiện. Kết quả từ các worker sẽ được gửi định kì tới master sau mỗi task được hoàn thành. Khi MapReduce đã được thực thi xong, master sẽ tổng hợp kết quả cho người dùng, đồng thời kết quả bộ đếm hiện thời cũng được hiển thị trong Báo cáo trạng thái.

## 5. Hiệu năng
### 5.1 Cấu hình Cluster
Tất cả các chương trình đều được chạy trên một cluster gồm khoảng 1800 máy tính. Tất cả các máy đều ở trong cùng một host nên thời gian khứ hồi giữa bất cứ cặp máy nào cũng đều nhỏ hơn 1 milisecond.
Khoảng 1-1.5G trên 4G RAM được sử dụng trước bởi các tác vụ khác chạy trên cluster. Chương trình được chạy vào một buổi chiều cuối tuần, khi mà CPU, disk và mạng đều khá nhàn rỗi.

### 5.2 Grep
Chương trình Grep quét qua 10^10 bản ghi, mỗi bản ghi 100-byte để tìm kiếm cho một pattern gồm 3 ký tự khá hiếm. Input chia ra thành M = 15000 phần, output được ghi vào R = 1 file duy nhất.
Toàn bộ quá trình mất khoảng 150s từ khi bắt đầu đến khi kết thúc, bao gồm 80s tính toán, khoảng 1 phút khởi động và cả thời gian trễ.
### 5.3 Sắp xếp (Sort)
Chương trình sắp xếp thực hiện trên khoảng 10^10 bản ghi, mỗi bản ghi 100 byte (Tổng khoảng 1TB dữ liệu), M = 15000, R = 4000.
- Khi đọc dữ liệu đầu vào, tốc độ đọc đạt tới 13 GB/s, quá trình hoàn thành trong 200s
- Khi dữ liệu được gửi qua network từ map task tới reduce task, quá trình này mất khoảng 600s.
- Sau khi reduce map hoàn thành, dữ liệu đã được sắp xếp và ghi vào final output file với tốc độ khoảng 2-4 GB/s. Quá trình mất khoảng 850s.
Tốc độ đọc input cao hơn so với chuyển dữ liệu và ghi output vì hầu hết dữ liệu đầu vào được đọc từ local disk.

### 5.4 Ảnh hưởng của Backup Task
Khi không có backup task, tốc độ thực thi không khác mấy so với bình thường, ngoại trừ khi phải thực hiện việc ghi dữ liệu thì thời gian kéo dài tới 960s. Tổng thời gian thực thi tăng 44%.

### 5.5 Machine Failures
Khi áp dụng cách bỏ qua các bản ghi không tốt, kết quả cho thấy một số thời điểm có tốc độ đọc âm do một số task đã hoàn thành trước đó biến mất. Tuy nhiên tổng thời gian thực thi chỉ tăng 5%.

## 6. Kinh nghiệm
Thư viện MapReduce đầu tiên được viết vào tháng 2/2003 và được sử dụng rộng rãi trong một loạt lĩnh vực của Google. MapReduce nhanh chóng phát triển và thành công bởi nó hiện thực hóa việc viết một chương trình đơn giản, chạy trên hàng nghìn máy tính trong khoảng thời gian ngắn. Nó cũng cho phép những lập trình viên chưa có kinh nghiệm với hệ thống phân tán có thể tiếp cận lượng lớn tài nguyên một cách dễ dàng.
Large-Scale Indexing là một trong những ứng dụng nổi bật nhất của MapReduce, giúp xử lý lượng dữ liệu thô có kích thước lên tới hơn 20 TB được sử dụng cho ứng dụng web của Google. Việc index trở trên đơn giản hơn, dễ dàng để hiểu và thực thi trong khoảng vài ngày thay vì vài tháng.