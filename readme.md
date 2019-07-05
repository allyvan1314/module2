# Báo cáo tìm hiểu module 2

---
<!-- TOC -->
- [Báo cáo tìm hiểu module 2](#B%C3%A1o-c%C3%A1o-t%C3%ACm-hi%E1%BB%83u-module-2)
  - [Khái niệm cơ bản về I/O](#Kh%C3%A1i-ni%E1%BB%87m-c%C6%A1-b%E1%BA%A3n-v%E1%BB%81-IO)
  - [File trong Unix](#File-trong-Unix)
  - [Blocking I/O vs Nonblocking I/O](#Blocking-IO-vs-Nonblocking-IO)
    - [Blocking I/O](#Blocking-IO)
    - [Non-blocking I/O](#Non-blocking-IO)
  - [Synchronous vs Asynchronous](#Synchronous-vs-Asynchronous)
    - [Synchronous](#Synchronous)
    - [Asynchronous](#Asynchronous)
    - [Asynchronous – Event-Driven](#Asynchronous-%E2%80%93-Event-Driven)

<!-- /TOC -->
---

## Khái niệm cơ bản về I/O

I/O là quá trình giao tiếp *(lấy dữ liệu vào, trả dữ liệu ra)* giữa một hệ thống thông tin và môi trường bên ngoài. Với CPU, thậm chí mọi giao tiếp dữ liệu với bên ngoài cấu trúc chip như việc nhập/ xuất dữ liệu với memory *(RAM)* cũng là tác vụ I/O. Trong kiến trúc máy tính, sự kết hợp giữa CPU và bộ nhớ chính *(main memory – RAM)* được coi là bộ não của máy tính, mọi thao tác truyền dữ liệu với bộ đôi CPU/Memory, ví dụ đọc ghi dữ liệu từ ổ cứng đều được coi là tác vụ I/O.

Do các thành phần bên trong kiến trúc phụ thuộc vào dữ liệu từ các thành phần khác, mà tốc độ giữa các thành phần này là khác nhau, khi một thành phần hoạt động không theo kịp thành phần khác, khiến thành phần khác phải rảnh rỗi vì không có dữ liệu làm việc, thành phần chậm chạp kia trở thành một bottle-neck, kéo lùi hiệu năng của toàn bộ hệ thống.

Dựa theo các thành phần của kiến trúc máy tính hiện đại, tốc độ thực hiện tiến trình phụ thuộc:

- **CPU Bound:** Tốc độ thực hiện tiến trình bị giới hạn bởi tốc độ xử lý của CPU.
- **Memory Bound:** Tốc độ thực hiện tiến trình bị giới hạn bởi dung lượng khả dụng và tốc độ truy cập của bộ nhớ.
- **Cache Bound:** Tốc độ thực hiện tiến trình bị giới hạn bởi số lượng ô nhớ và tốc độ của các thanh cache khả dụng.
- **I/O Bound:** Tốc độ thực hiện tiến trình bị giới hạn bởi tốc độ của các tác vụ IO.

Do tốc độ I/O thường rất chậm so với các thành phần còn lại, bottle-neck thường xuyên xảy ra ở đây. Người ta thường xét đến I/O Bound và CPU Bound, cố gắng đưa các process bị giới hạn bởi I/O bound về CPU bound để tận dụng tối đa hiệu năng.

## File trong Unix

Chắc hẳn bạn đã nghe `"On a UNIX system, Everything is a file"`, (Thật ra còn có vế sau: `"If something is not a file, it is a process"`, hoặc theo như Linux Torvalds - `"Everything is a file descriptor or a process"`). Mọi I/O devide cũng được coi, và có thể được đối xử như là một file trong Unix filesystem. Điều này tạo ra một abtraction cho các thiết bị I/O, che giấu bản chất thật sự của các thiết bị này, kernel chỉ biết giao tiếp với file chứ không cần thiết phải nhận biết có cách hành xử riêng cho từng thiết bị.

Các hành động `open, read, write` trên file chính là các `I/O operation`. Khi `open()` hay `create()` một file, kernel sẽ tạo ra một cấu trúc `file description` lưu trong `file table` và trả lại một số nguyên chứa referent đến `file description` tương ứng trong bảng.

Mỗi process sẽ lưu một danh sách các file descriptor – các file mà process ấy đang sở hữu, mọi thao tác với file sẽ được thực hiện thông qua một system call với đối số là file descriptor, nghĩa là process ở user mode nhờ kernel thao tác lên một file mà nó sở hữu. Việc này để đảm bảm rằng chỉ kernel mới có quyền tác động đến hệ thống theo một cách an toàn, chương trình chỉ có thể nhờ kernel đại diện thực hiện hành động.

Trong Unix, có thể chia file thành 2 nhóm: `Fast` và `Slow file`. Đừng để cái tên Nhanh và Chậm làm bạn nhầm lẫn rằng nó đề cập đến tốc độ đọc ghi thực tế với file, điều cần quan tâm đến ở đây là `khả năng dự đoán` của nó. Các file có thể đáp ứng yêu cầu đọc/ghi của user trong `một khoảng thời gian dự đoán được` là Nhanh, những file có thể cần đến `vô hạn thời gian` để đáp ứng lại use call, là Chậm.

Ví dụ, khi đọc dữ liệu từ ổ cứng, kernel biết rằng dữ liệu ở đó, nó chỉ cần một thời gian hữu hạn để lấy dữ liệu và trả về cho user, vậy nên việc đọc một `regular file` là Nhanh. Việc đọc từ một `Pipe` hoặc `Socket`, kernel không thể nào biết được khi nào thì trên file đó có dữ liệu, đồng nghĩa `Pipe` hay `Socket` là Chậm.

| File Type        | Category |
|------------------|----------|
| Block Device     | Fast     |
| Pipe             | Slow     |
| Socket           | Slow     |
| Regular          | Fast     |
| Directory        | Fast     |
| Character Device | Varies   |

Khi một process cần lấy dữ liệu từ file, trong user mode, nó gọi system call `read()` trên một file đã mở, kernel đón nhận lời gọi này và thao tác đọc file trong kernel mode rồi trả **data** *(block of bytes)* đọc được về user mode để process xử lý.

Nếu là fast file, kernel đi lấy data ở nơi nó đã biết và trả lại user hoặc thông báo lỗi nếu xảy ra ngoại lệ *(VD: không xác định được vị trí file)*.

Với slow file, kernel sẽ trả lại bất kỳ dữ liệu sẵn có trên file thậm chí chẳng cần bắt gặp ký tự `EOF` *(End Of File)*. Nếu trên file chưa có dữ liệu, kernel chỉ đơn giản là block process *(với chế độ mặc định)*, chuyển nó vào chế độ ngủ và đánh thức nó dậy khi dữ liệu đã sẵn sàng.

Điều này là hoàn toàn hợp lý vì thông thường khi lập trình viên đọc một file, anh ta kỳ vọng một dữ liệu cần thiết cho xử lý tiếp theo trong chương trình. Đấy chính là cái mà người ta thường gọi là **Blocking I/O**.

Trên thực tế, sẽ có những lúc ta muốn process của mình làm một cái gì đấy khác hơn là lăn ra ngủ, kể cả khi I/O operation không thể hoàn thành ngay lập tức. Bằng việc sử dụng *fcntl()* interface get flag `O_NONBLOCK` file descriptor cụ thể, ta có thể chuyển I/O model của file sang **Nonblocking I/O.**

Khi ấy, nếu một operation không thể thực hiện ngay lập tức, hàm gọi sẽ trả về `EAGAIN` *(“Try it again”)*. Cờ `O_NONBLOCK` không có tác dụng với các fast file như regular file, directory file.

## Blocking I/O vs Nonblocking I/O

Minh họa về blocking và non-blocking I/O:

![alt](https://codersontrang.files.wordpress.com/2017/09/new-mockup-1.png?w=593&h=591)

### Blocking I/O

Yêu cầu thực thi một IO operation, sau khi hoàn thành thì trả kết quả lại. Pocess/Theard gọi bị block cho đến khi có kết quả trả về hoặc xảy ra ngoại lệ.

Trong hình trên, phần phía trên miêu tả sự hoạt động theo cơ chế Blocking mà ở đây mặc dù không có sự liên đới giữa 3 việc, nhưng các công việc tiếp sau luôn phải chờ công việc phía trước thực sự xong rồi mới có thể bắt đầu thực hiện. Các bước sẽ được mô tả như dưới đây:

1. Hàm dataSync1.get() được gọi để lấy dữ liệu, vì nó là Blocking nên trước khi công việc này hoàn thành các việc tiếp sau sẽ phải đợi.
2. Hàm printData(d1) được gọi để in dữ liệu lấy về từ dataSync1.get(), tương tự nó cũng là Blocking.
3. Hàm dataSync2.get() được gọi để lấy dữ liệu, mặc dùng là nó không liên quan gì đến hai dòng lệnh trên, nhưng đến tận bây giờ nó mới được thực hiện và là Blocking nên chiếm một khoảng thời gian xử lý nữa.
4. Hàm printData(d2) được gọi để in dữ liệu lấy về từ dataSync2.get(), là Blocking.
5. Hàm dataSync3.get() được gọi để lấy dữ liệu, là Blocking.
6. Hàm printData(d3) được gọi để in dữ liệu lấy về dataSync3.get(), là Blocking.

Ở phần này, mọi thao tác đều là blocking nên thời gian để thực hiện xong hết các thao tác sẽ bằng tổng thời gian của từng thao tác.

### Non-blocking I/O

Yêu cầu thực thi IO operation và trả về ngay lập tức (timeout = 0). Nếu operation chưa sẵn sàng để thực hiện thì thử lại sau. Tương đương với kiểm tra IO operatio có sẵn sàng ngay hay không, nếu có thì thực hiện và trả về, nếu không thì thông báo thử lại sau.

Trong hình trên, phía dưới là phần thể hiện việc làm tất cả những việc trên, các thao tác in dự liệu printData(d1), printData(d2), printData(d3) vẫn là các thao tác Blocking nhưng ở đây có sự tham gia của Non-Blocking trong các thao tác lấy dữ liệu dataAsync1.get(), dataAsync2.get(), dataAsync3.get().

Các thao tác Non-Blocking sẽ được bắt đầu gần như ngay lập tức và không cần phải chờ các thao tác phía trước thực hiện xong. Sau khi có kết quả các thao tác Non-Blocking sẽ gọi lại callback để in kết quả trả về ra màn hình. Cụ thể sẽ được diễn giải như ở dưới đây:

1. Hàm dataAsync1.get() được gọi để lấy dự liệu, vì nó là Non-Blocking nên quá trình thực thi sẽ không phải dừng ở đây mà tiếp tục thực hiện dòng lệnh tiếp sau gần như ngay lập tức, tất nhiên vẫn phải sau khi đăng ký một callback để in ra dữ liệu trả về từ dataAsync1.get().
2. Như nói ở trên, ngay sau đó, hàm dataAsync2.get() được gọi cùng với đăng ký callback. Vì là Non-Blocking nên quá trình cũng giống như trên.
3. Tiếp theo hàm dataAsync3.get() cũng được thực hiện tương tự. Đến đây, 3 hàm gọi để lấy dữ liệu gần như được thực hiện đồng thời mà không cần phải chờ nhau.
4. Trong khi hàm dataAsync1.get() và dataAsync3.get() đang thực hiện thì hàm dataAsync2.get() đã lấy được dữ liệu về, lúc này callback được gọi để in dữ liệu đó ra màn hình, trong callback lúc này printData(d2) được gọi và nó là Blocking.
5. Trong thời gian printData(d2) đang thực hiện, dataAsync1.get() đã hoàn tất việc lấy dữ liệu, callback của nó được gọi tuy nhiên vì printData(d2) là Blocking và đang thực hiện nên việc thực hiện printData(d1) sẽ phải chờ.
6. Cũng tương tự như trên, dataAsync3.get() cũng hoàn tất việc lấy dữ liệu, callback của nó được gọi, lần này printData(d3) không những phải chờ printData(d2) như trên mà nó còn phải chờ thêm cả printData(d1) bởi vì printData(d1) cũng là Blocking. Sau khi cả printData(d2) và printData(d1) được hoàn thành thì printData(d3) được thực hiện và toàn bộ quá trình hoàn tất.

Ví dụ về [Blocking & non-blocking trong lập trình](https://codersontrang.wordpress.com/2017/09/05/blocking-va-non-blocking-trong-lap-trinh/) , demo phía dưới.

## Synchronous vs Asynchronous

### Synchronous

Hiểu đơn giản: Diễn ra theo thứ tự. Một hành động chỉ được bắt đầu khi hành động trước kết thúc.

### Asynchronous

Không theo thứ tự, các hành động có thể xảy ra đồng thời hoặc chí ít, mặc dù các hành động bắt đầu theo thứ tự nhưng kết thúc thì không. Một hành động có thể bắt đầu *(và thậm chí kết thúc) *trước khi hành động trước đó hoàn thành.

Sau khi gọi hành động A, ta không trông chờ kết quả ngay mà chuyển sang bắt đầu hành động B. Hành động A sẽ hoàn thành vào một thời điểm trong tương lai, khi ấy, ta có thể quay lại xem xét kết quả của A hoặc không. Trong trường hợp quan tâm đến kết quả của A, ta cần một sự kiện Asynchronous Notification thông báo rằng A đã hoàn thành.

Vì thời điểm xảy ra sự kiện hành động A hoàn thành là không thể xác định, việc bỏ dở công việc đang thực hiện để chuyển sang xem xét và xử lý kết quả của A gây ra sự thay đổi luồng xử lý của chương trình một cách không thể dự đoán. Luồng của chương trình khi ấy không tuần tự nữa mà phụ thuộc vào các sự kiện xảy ra. Mô hình như vậy gọi là Event-Driven.

### Asynchronous – Event-Driven

Asynchronous và Event-Driven không phải là một điều gì quá mới mẻ. Thực tế nó đã tồn tại trong ngành khoa học máy tính từ những ngày đầu. Cơ chế ngắt *(Interrupt)* là một signal thông báo cho hệ thống biết có một event vừa xảy ra.

Khi ngắt xảy ra, hệ thống buộc phải dừng chương trình đang chạy để ưu tiên chạy một chương trình khác gọi là **Chương Trình Phục Vụ Ngắt** *(Interrupt Service Routine)* rồi mới quay trở lại thực hiện tiếp chương trình đang chạy dở.

- Có hai loại ngắt: **Ngắt Cứng** và **Mgắt Mềm**. **Ngắt mềm** được gọi bằng một lệnh trong ngôn ngữ máy *(và hợp ngữ, dĩ nhiên)*. **Ngắt cứng** thì chỉ có thể được gọi do các linh kiện điện tử tác động lên hệ thống.
  - Một ví dụ về ngắt cứng là khi card mạng nhận được data từ bên ngoài. Card mạng gửi tín hiệu điện lên chân ngắt của CPU, cờ ngắt *(INT)* trên resigter được kích hoạt. CPU dừng lại, kiểm tra mức độ ưu tiên *(tất nhiên không phải thằng nào xin ngắt cũng cho ngắt, biết đâu việc mình đang thực hiện quan trọng hơn)*, kiểm tra tín hiệu ngắt là gì, từ tín hiệu ngắt mà lấy được địa chỉ của chương trình phục vụ ngắt *(chính là driver của device).*
  - Drive sẽ quyết định ghi dữ liệu vào một file trên **Unix Virtual File System**, ở đây là vào một socket file. Việc thao tác với divice *(đọc dữ liệu từ bàn phím, ghi dữ liệu ra màn hình)* thực tế được quyết định bởi device và kernel. Trên góc nhìn của user, device nào thì cũng chỉ là một file đọc được, ghi được, chẳng có gì khác nhau và cũng chẳng khác gì các regular file.
- Một đối tượng ví dụ tiêu biểu cho slow file là socket.
  - Sau khi một socket được tạo *(đại diện bởi một file desciptor)* và lắng nghe trên một cổng, một yêu cầu xin kết nối được gửi từ client tới và được **accept()** để thiết lập connection *(Một file khác được tạo ra, clone từ file socket cũ kết hợp thêm cổng và địa chỉ của client. Mỗi connection tới server sẽ có một file descriptor riêng)*.
  - Sytem call **recv()** được gọi để sẵn sàng đón nhận message từ client, dưới cái nhìn của kernel là đọc dữ liệu từ file socket, hành động mà sẽ return -1 nếu chưa nhận được message từ client *(Và với TCP socket, return 0 nếu connection đã bị ngắt bởi một phía)*. Nếu trong chế độ synchronous theo mặc định, process sẽ sleep cho tới khi phép đọc sẵn sàng để thực hiện. Trong thời điểm này, nếu một client khác gửi yêu cầu đến server, server của bạn sẽ không thể thực hiện yêu cầu của client ngay được, thậm chí server còn không biết đến sự tồn tại của yêu cầu ấy.
