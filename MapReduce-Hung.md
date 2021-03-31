# MapReduce: Simplified Data Processing on Large Clusters

##1, Giới thiệu
* Trong thời điểm bấy giờ, các hệ thống tính toán có cơ chế thực hiện xử lý dữ liệu lớn một cách khá trực tiếp. Điều này gây ra các vấn đề nhưi gian chạy lớn, khó kiểm soát lỗi, độ phức tạp cao,...
* Các tác giả thiết kế ra một trường phái gọi là MapReduce với mục đích giải quyết hết các vấn đề tồn đọng của tính toán song song, của hệ phân tán trong việc xử lý dữ liệu lớn.
## 2.Mô hình
### a,Các khái niệm
* Mô hình lấy đầu vào là 1 tập các cặp key/value, và tạo đầu ra cũng là 1 tập các cặp key/value. Người dùng sử dụng thư viện MapReduce để biểu diễn 2 hàm: Map và Reduce.
* Hàm Map nhận đầu vào là 1 cặp giá trị và tạo đầu ra là 1 tập trung gian gồm các cặp key/value.Từ các cặp đầu ra này, thư viện MapReduce sẽ nhóm tất cả các giá trị (value) mà có cùng khóa (key) với nhau và truyền vào hàm Reduce.

**map (k1,v1) => list(k2,v2)**
* Hàm Reduce nhận 1 khóa I và 1 tập giá trị ứng với khóa này. Từ các đầu vào này, đầu ra của hàm là 1 tập nhỏ hơn ( hoặc bằng) tập đầu vào. Mỗi lần gọi hàm Reduce, thông thường sẽ chỉ có 1 giá trị hoặc không có giá trị đầu ra nào. Do vậy, mô hình này sẽ cho phép xử lý những đầu vào lớn vượt mức cho phép của bộ nhớ.

**reduce (k2,list(v2)) => list(v2)**
### b,Một số ví dụ
* Đếm tấn suất truy cập URL: Hàm Map tiếp nhận, xử lý các log của các URL request và cho ra đầu ra là từng {URL,1}. Hàm Reduce sẽ cộng tất cả giá trị cùng khóa URL và cho ra kết quẩ là **{URL, total_count}**.
* Inverted Index: Đầu vào là một số các văn bản (document) được đánh số (ID). Nhiệm vụ hàm Map là xử lý từng văn bản môt, với mỗi văn bản thì sẽ cho ra 1 tập các cặp {word, document_ID}. Hàm Reduce nhận vào tất cả cặp ứng với từ khóa target_word cho trước, sắp xếp lại các document_ID với ứng với từ khóa này và cho ra 1 cặp **{target_word, list(document_ID)}**.

## 3.Triển khai
* Có nhiều cách để triển khai chương trình MapReduce. Tuy vậy, để lựa chọn cách phù hợp nhất thi nên dựa vào môi trường cài đặt. Cách triển khai được bàn ở dưới đây nhắm đến môi trường tính toán được sử dụng phổ biến tại Google.
### a,Tổng quan
* Lời gọi hàm Map sẽ được phân tán trên các máy con bằng cách chia input thành M phần. Các phần này sẽ được xử lý song song. Lời gọi hàm Reduce cũng được phân tán bằng cách sử dụng 1 hàm chia để chia thành R vùng cho các khóa trung gian. Số vùng (R) và hàm chia sẽ được người dùng quyết định.
* Các bước trong cách triển khai này:

1, Thư viện MapReduce chia đầu vào thành M phần, mỗi phần có dung lượng khoảng 16-64MB. Sau đó thư viện sẽ khởi động các bản sao của chương trình trên cụm các máy cọn

2, Một trong các bản sao là một bản sao đặc biệt, gọi là master. Các bản sao còn lại là worker, được giao việc bới master. Có tổng cộng M công việc Map và R công việc Reduce cần thực hiện.

3, Mỗi worker được giao công việc Map sẽ tiếp nhận một phần của input (đã chia ở bước 1).Sau đó worker xử lý rồi đưa từng cặp key/value vào hàm Map. Các cặp đầu ra key/value của hàm Map sẽ được đưa vào bộ nhớ đệm.

4, Các cặp key/value trên bộ nhớ đệm sau đó đều được vào bộ nhớ cục bộ (local), và được chia thành R vùng bởi hàm chia. Địa chỉ của các cặp trung gian trên bộ nhớ cục bộ đều được lưu trên master, nơi có trách nhiệm sẽ gửi các thông tin này đến các worker làm công việc Reduce.

5, Worker làm nhiệm vụ Reduce sẽ dùng RPC (remote procedure call) để đọc dữ liệu từ bộ nhớ cục bộ. Khi đã có đủ dữ liệu trung gian, các worker này sẽ sắp xếp các khóa (key) để có thể nhóm các cặp có cùng khóa với nhau.

6, Worker sẽ lặp qua các khóa đã được sắp xếp này. Với từng khóa riêng biệt, worker sẽ đưa khóa này và các giá trị (value) ứng với khóa vào hàm Reduce. Các dữ liệu đầu ra sẽ được viết nói tiếp nhau (append) trong file output cuối cùng.

7, Cuối cùng, khi đã hoàn thành xong tất cả công việc Map và Reduce, master sẽ thông báo và trả kết quả cho người dùng.

### b, Dữ liệu trong Master.
* Master lưu trữ nhiều kiểu cấu trúc dữ liệu khác nhau. Với từng công việc Map và Reduce, master lưu trạng thái (rảnh, đang thực hiện, hoàn thành) và định dang của máy worker tương ứng.
* Master cũng lưu lại vị trí và kích thước của R vùng chứa các dữ liễu trung gian (đầu ra của Map).

### c,Khả năng chịu lỗi
* Vì thư viện MapReduce được tạo ra nhằm sử dụng hàng trăm, hàng nghìn máy tính để xử lý lượng dữ liệu khổng lồ, nên thư viện phải có khả năng chịu lỗi tốt.

**Worker gặp lỗi**
* Cứ sau 1 khoảng thời gian nhất định, master sẽ ping đến các worker. Nếu không nhận được phản hồi từ các worker sau 1 số lần ping nhất định, master sẽ coi như worker đó gặp lỗi.
* Tất cả công việc đang dở (của Map và Reduce) hoặc đã hoàn thành ( chỉ của Map) của worker lỗi sẽ được đưa về lại trạng thái nhàn rỗi, và sẵn sàng để được xếp lịch cho một worker khác. 
* Sở dĩ phải reset lại những công việc Map đã hoàn thành của máy lỗi vì các đầu ra của Map đều được lưu trên bộ nhớ cục bộ của máy lỗi, do đó không thể truy cập được bộ nhớ này. Công việc Reduce của máy lỗi thì không cần thực hiện lại vì đầu ra được lưu trên hệ thống global.

**Master gặp lỗi**
* Không khó để thiết kế cho master chức năng tự lưu lại các checkpoint định kì về các loại dữ liệu đã nêu ở trên. Nếu như master gặp lỗi, một bản sao master mới sẽ được tạo ra và sẽ bắt đầu từ checkpoint gần nhất
* Trong trường hợp chỉ có 1 master duy nhất và master này gặp lỗi, cách triển khai MapReduce này sẽ hủy toàn bộ chương trình luôn. Các máy khách vẫn có thể tự thử tiếp tục MapReduce nếu muốn.




### d,Tính cục bộ
* Băng thông mạng là một loại tài nguyên tương đối hiếm trong môi trường tính toán. Chúng ta tiết kiệm nó bằng cách tận dụng việc lưu trữ dữ liệu trên các bộ nhớ cục bộ. GFS sẽ chia mỗi file thành từng phần (mỗi phần 64MB), và sẽ sao chép vài bản sao của mỗi phần này trên các máy con khác nhau.
* MapReduce master sẽ xếp lịch chp máy con xử lý với bộ dữ liệu lưu trên chính máy con này. Như vậy, khi hoạt động với 1 lượng lớn các worker trên 1 cụm máy, phần lớn dữ liệu sẽ được xử lý trên chính máy lưu trữ nó (read locally), và không tiêu tốn băng thông mạng.

### e, Chi tiết về công việc
* Như đã nêu ở trên, ta chia giai đoạn thực hiện Map thành M phần, giai đoạn thực hiện Reduce thành R phần. Trong trường hợp lý tưởng, M và R sẽ lớn hơn nhiều so với số máy worker. Điều này khiến 1 worker cần thực hiện nhiều task, giúp load balancing ( cân bằng công việc giữa các worker) trở nên linh hoạt hơn và đồng thời đẩy nhanh quá trình recovery khi 1 worker gặp lỗi.
* Trong thực tế, cần phải giới hạn độ lớn của M và R vì master sẽ cần phải thực hiện xếp lịch với độ phức tạp O(M+R) và lưu trữ trạng thái độ phức tạp O(M*R).
* Hơn nữa, R thường bị giới hạn bởi chính người dùng vì mỗi đầu ra của 1 công việc Reduce sẽ đều được lưu vào 1 file đầu ra cuối cùng của toàn hệ thống. Thông thường, chúng ta sẽ chạy MapReduce với M=200,000 và R=5000, sử dụng khoảng 2000 máy worker

### f, Backup công việc
* Một nguyên nhân phổ biến gây kéo dài thời gian chạy MapReduce là "straggler", tức là khi gần hoàn thành chạy MapReduce, 1 máy con chạy 1 công việc chậm hơn hẳn so với các máy còn lại. Có nhiều nguyên nhân gây ra hiện tượng này. Một trong đó có thể là ổ đĩa lưu trữ của máy đó gặp lỗi Correctable error, giảm tốc độ đọc từ 30MB xuống còn 1MB/s.
* Các tác giả đã dựng 1 cơ chế chung để giải quyết straggler. Khi gần hoàn thành quá trình chạy, master sẽ backup lại những nhiệm vụ của các công việc đang chạy dở. Các công việc này sẽ được đánh nhãn hoàn thành nếu hoặc nhiệm vụ gốc hoặc nhiệm vụ backup hoàn thành.

## 4, Sàng lọc
### a, Hàm phân vùng
* Người dùng tự xác định số lượng công việc Reduce (R) mà họ muốn (như đã nêu). Dữ liệu sẽ được phân vùng bằng 1 hàm phân vùng dựa vào khóa trung gian. Mặc định thì hàm được dùng là hàm băm (hash(key) mod R) vì sẽ giúp cân bằng lượng chia vùng hơn.
* Trong vài trường hợp đặc biệt, ví dụ như output là URL, thì hàm phân vùng sẽ nên là hash(hostname(urlkey) mod R), như vậy các URL từ cùng 1 host sẽ được nhóm lại với nhau.

### b, Đảm bảo thứ tự
* Các tác giả đảm bảo rằng: trong 1 phân vùng, các cặp key/value sẽ được sắp xếp theo thứ tự tăng dần của key. Việc này giúp cho đầu ra cũng sẽ được sắp xếp, thuận tiện cho người dùng tra cứu bằng key hơn.
#
## c, Hàm hợp (Combiner function)
* Trong một số trường hợp, các công việc Map tạo ra nhiều khóa trung gian giống nhau, và người dùng lại viết hàm Reduce có tính giao hoán và kết hợp. Một ví dụ tiêu biểu là bài toán đếm số từ: nhiều công việc Map khả năng cao sẽ có cùng đầu ra dạng [the, 1] (từ the xuất hiện trong Tiếng Anh rất nhiều). Các tác giả vì vậy đã tạo ra 1 hàm hợp cho phép người dùng trộn (merge) các dữ liệu trùng lặp này với nhau trước khi chuyển cho các công việc Reduce.
* Hàm hợp sẽ được thực hiện trên các máy worker chạy công việc Map. Thường thì đoạn mã cho hàm hợp và hàm Reduce sẽ giống nhau. Khác biệt duy nhất có lẽ là đầu ra hàm Reduce sẽ được lưu vào file output cuối cùng, đầu ra của 1 hàm hợp sẽ được lưu vào file trung gian, sau đó file này sẽ được gửi vào worker chạy Reduce.

### d, Các loại đầu vào và đầu ra.
* Thư viện MapReduce hỗ trợ đọc đầu vào dưới nhiều định dạng khác nhau. Ví dụ, định dạng "text" thì sẽ coi từng dòng của đầu vào là 1 cặp key/value: khóa key sẽ là offset của file, còn giá trị value là nội dung của dòng đó.
* Đa số người dùng sẽ sử dụng 1 trong số các định dạng hỗ trợ sẵn. Người dùng có thể tự thêm hỗ trợ cho định dạng mới bằng cách implement lại 1 interface đơn giản có tên là *reader*.
*Tương tự, thư viện cũng hỗ trợ một số định dạng cho đầu ra và cách thức thêm hỗ trợ cho định dạng mới cũng tương tự như đầu vào.

### e, Tác dụng phụ
* Trong một số trường hợp, người dùng cảm thấy tiện hơn khi tạo ra 1 file mới để lưu các đầu ra được bổ sung vào từ các công việc Map/Reduce. Các tác giả sẽ để cho cho bên phát triển ứng dụng xử lý các tác dụng phụ kiểu này.

###f, Bỏ qua các record xấu.
* Các bug ở trong các đoạn code của người dùng thường khó tránh khỏi. Chúng khiến các hàm Map/Reduce không thể chạy được với một số record. Cách giải quyết phổ biến sẽ là fix các bug đó, nhưng đôi khi việc này không khả thi. Vì vậy, thư viện MapReduce cung cấp cơ chế phát hiện record nào gây ra lỗi và bỏ qua (skip) không xử lý record này.

###g,Thực thi cục bộ (Local execution)
* Debug lỗi với các hàm Map/Reduce không đơn giản bởi thực tế các tính toán được chạy trên hệ phân tán với hàng nghìn máy con.
* Thư viện MapReduce cung cấp một chức năng giúp người dụng triển khai toàn bộ chương trình trên 1 máy cục bộ. Người dùng nắm quyền điều hành với chương trình và có thể dễ dàng lựa chọn công cụ debug mà họ muốn, ví dụ như gdb.

###h, Thông tin trạng thái
* Master sẽ chạy một server HTTP của riêng nó và xuất ra các trang chứa thông tin trạng thái của chương trình. Các trang thông tin này cho người dùng biết tiến độ của chương trình, ví dụ bao nhiêu công việc hoàn thành, dung lương dữ liệu trung gian, dung lương của đầu ra, v.v. Người dùng dựa vào đây để dự đoán thời gian tính toán hoặc có nên thêm tài nguyên hay không.
* Chức năng này cũng rất tiện cho người dùng debug code của họ khi cũng chỉ ra các thông tin những worker nào gặp lỗi, công việc nào đang được chạy trên các worker đó.

###i, Bộ đếm
* Thư viện MapReduce cung cấp 1 cơ chế bộ đếm giúp đếm số lần xuất hiện của các sự kiện. Ví dụ như đếm số từ được xử lý trong chương trình đếm từ.
* Người dùng sẽ tạo 1 object của lớp Counter và tạo bộ đếm phù hợp trong hàm Map/Reduce.
* Một số giá trị có sẵn bộ đếm được tạo bởi thư viện như số cặp key/value đầu vào được xủ lý hay đầu ra có bao nhiêu số cặp key/value.

## 5, Performance
Phần này bàn về hiệu năng của 1 chương trình MapReduce được viết bởi người dùng.

### a, Cấu hình cluster.
* Cluster có khoảng 1800 máy. Mỗi máy được trang bị 2 bộ xử lý Inter Xeon 2GHz, 4GB Ram, 2 ổ đĩa IDE 160GB và 1 đường truyền Ethernet.
* Trong 4GB Ram, sẽ có khoảng 1-1.5GB dành cho các tác vụ không liên quan đến chương trình MapReduce. Chương trình được chạy vào chiều cuối tuần, khi mà các CPU, ổ đĩa hay mạng đều ở trạng thái rảnh.

###b, Grep
* Chương trình grep sẽ quét qua 10^10 bản ghi 100B, để tìm ra 1 mẫu ba kí tự tương đối hiếm (xuất hiện 1 lần trong 92337 bản ghi). Đầu vào được chia thành các phần 64MB (M=15000) vào đầu ra được đặt vào 1 file duy nhất (R=1).

### c, Sắp xếp
* Như đã nêu, đầu vào được chia thành M=15000 phần, mỗi phần 64MB. Đầu ra được hàm phân vùng phân thành 4000 file (R=4000).
* Kết quả thời gian sắp xếp cụ thể như sau:
- Tốc độ đọc đạt tối đa 13GB/s, thời gian đọc là 200 giây.
- Thời gian chuyển dữ liệu từ công việc Map sang Reduce qua 1 đường truyền mạng: Sau 300 giây đầu thì có một số worker Reduce hoàn thành công việc. Thời giạn để chuyển dữ liệu qua đường truyền mạng của cả hệ thống là khoảng 600 giây.
- Lưu dữ liệu: Tốc độ lưu dữ liệu là khoảng 2-4GB/s, tổng thời gian lưu là 850 giây

###d, Hiệu quả của Backup công việc.
* Sau khoảng 960 giây, tất cả trừ 5 công việc Reduce đã hoàn thành. 5 straggler này thì tốn tận 300 giây nữa để hoàn thành. Với cơ chế backup, tổng thời gian chạy đã được cải thiện 44%.

###e, Dừng chạy 200 máy
* Trong trường hợp cố tình dừng chạy 200 máy trong 1746 máy worker sau một vài phút, tổng thời gian chạy là 933 giây, cải thiện khoảng 5% so với chế độ bình thường.

## 6, Các thành quả
*tính đến thời điểm paper được xuất bản
* Các tác giả viết phiên bản đầu tiên của MapReduce vào tháng 2/2003, cùng với cải tiến đáng kể vào tháng 8/2023 như là tối ưu hóa cục bộ, cân bằng load linh hoạt (dynamic load balancing) giữa các worker.
* Các thống kế cho thấy một lượng tăng triển không nhỏ trong số chương trình MapReduce được tích hợp. Đến tháng 9/2004, có khoảng 900 ví dụ về MapReduce đã được phát triển. Lý do cho sự phát triển này là nó giúp viết những chương trình đơn giản trên các cụm máy phức tạp trong một khoảng thời gian ngắn. Hơn nữa nó giúp các lập trình viên chưa có nhiều kinh nghiệm hệ phán tán hay tính toán song song có thể tiếp cận với một lượng dữ liệu khổng lồ dễ dàng.
* Một trong những thành quả nổi bật của MapReduce là đã hoàn toàn xây dựng lại hệ thống đánh chỉ số (indexing system). Một quá trình của hệ thống đã được giảm từ 3800 dòng code C++ xuống còn khoảng 700 dòng. Thời gian cài đặt hệ thống cũng giảm từ vài tháng xuống còn vài ngày.