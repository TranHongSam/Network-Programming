# Source: Beej's Guide to Network Programming Using Internet Sockets

*Beej's Guide to Network Programming is Copyright © 2007 Brian “Beej Jorgensen” Hall.*

## 2. Khái niệm socket
- Sockets là 1 cách để giao tiếp với các chương trình khác sử dụng các mô tả tệp tin của chuẩn Unix. ( a way to speak to other programs using standard Unix file descriptors)
- Ok, bạn có thể đã nghe một số tin tặc Unix nói, "Jeez, mọi thứ trong Unix là một tập tin!" Điều mà người đó có thể đã nói đến là khi các chương trình Unix thực hiện bất kỳ loại nào của I / O, họ làm điều đó bằng cách đọc hoặc viết một mô tả tập tin. Trình mô tả tệp chỉ đơn giản là một số nguyên liên kết với một tệp mở. Nhưng (bắt buộc), tệp đó có thể là kết nối mạng, FIFO, pipe, terminal, một tệp thực trên đĩa, hoặc bất kỳ thứ gì khác. Tất cả mọi thứ trong Unix là một tập tin! Vì vậy, khi bạn muốn giao tiếp với một chương trình khác trên Internet, bạn sẽ làm điều đó thông qua một bộ mô tả tập tin, tốt nhất là bạn nên tin vào điều đó.
- Để lấy được mô tả tệp tin cho sự giao tiếp qua mạng, bạn gọi socket(). Nó sẽ trả về bộ mô tả socket và thực hiện giao tiếp thông qua nó bằng cách sử dụng các lệnh gọi socket chuyên biệt là send() và recv() (man send, man recv). Có thể sử dụng các lệnh gọi read() và write() thông thường để giao tiếp thông qua socket, nhưng send() và recv() cung cấp khả năng kiểm soát dữ liệu của bạn tốt hơn nhiều.
- Có tất cả các loại ổ cắm. Có DARPA Internet address(Internet Sockets), tên đường dẫn trên nút cục bộ (Unix Sockets), CCITT X.25 address (X.25 Socket mà bạn có thể bỏ qua một cách an toàn), và có lẽ nhiều ổ khác tùy thuộc vào phiên bản Unix bạn chạy. Tài liệu này chỉ đề cập đến loại đầu tiên: Internet Socket.

**2.1 Hai loại Internet Socket**
- Có nhiều hơn 2 loại Internet Socket nhưng ở đây chỉ đề cập đến hai loại: Stream Socket (SOCK_STREAM) và Datagram Socket(SOCK_DGRAM). 
- Stream Socket là luồng truyền thông hai chiều đáng tin cậy. Nếu bạn xuất hai mục vào socket theo thứ tự "1, 2", chúng sẽ đến theo thứ tự "1, 2" ở bên nhận mà không có lỗi. Telnet sử dụng Stream Socket. Ngoài ra, các trình duyệt web sử dụng giao thức HTTP cũng sử dụng Stream Socket để nhận các trang. Stream Socket sử dụng giao thức TCP (Transmission Control Protocol). TCP đảm bảo rằng dữ liệu đến theo thứ tự và không xuất hiện lỗi. 
- Datagram Socket đôi khi được gọi là "connectionless sockets". Datagram cũng sử dụng IP để định tuyến nhưng nó sử dụng giao thức UDP (User Datagram Protocol). "Connectionless" là bởi vì không cần duy trì kết nối giống như Stream Socket. Bạn chỉ cần xây dựng gói, thêm IP header với địa chỉ đích và gửi đi mà không cần phải tạo 1 kết nối. Nó thường được sử dụng khi TCP stack không có sẵn hoặc khi một vài gói bị drop. Các ứng dụng tương tự: tftp, bootp, multiplayer games, phát trực tuyến âm thanh ( streaming audio), hội nghị truyền hình (video conferencing), v.v.
- TFTP và BOOTP là các ứng dụng được sử dụng để truyền các bit nhị phân từ host này sang host kia. Cho nên dữ liệu phải được đảm bảo đến nơi an toàn. Do đó, TFTP và một số chương trình tương tự có giao thức riêng của nó đặt ở trên giao thức UDP. Với mỗi gói tin được gửi đi, bên nhận cần phải gửi lại bản tin ACK (acknowledgment) để thông báo cho bên gửi là đã nhận được gói. Nếu bên gửi không nhận được bản tin hồi đáp sau 5 giây, nó sẽ gửi lại gói cho đến khi nào nhận được bản tin ACK thì thôi. Thủ tục nhận gói ACK rất quan trọng cho việc triển khai các ứng dụng tin cậy SOCK_DGRAM.
- Với các ứng dụng không tin cậy như game, bạn chỉ cần không quan tâm đến các gói tin bị drop.

**2.2 Low level Nonsense and Network Theory**

<img src="https://2.pik.vn/2018ecdc4a7c-503a-4e6c-bce4-bc6e6d3a4571.jpg">

- Data Encapsulation: gói được sinh ra, rồi được bọc (đóng gói) trong một header bởi giao thức đầu tiên là TFTP,sau đó toàn bộ gói (bao gồm cả TFTP header) được đóng gói lần nữa bởi giao thức UDP, đóng gói một lần nữa bởi IP, cuối cùng bởi các giao thức trên tầng vật lý (Ethernet).
- Khi 1 máy tính nhận gói, phần cứng bóc Ethernet header, phần nhân (kernel) bóc IP và UDP header, chương trình TFTP bóc TFTP header, cuối cùng ta được dữ liệu gốc.
- Mô hình mạng OSI (Layered Network Model): mô hình mạng này mô tả một hệ thống chức năng mạng có nhiều ưu điểm so với các mô hình khác. Ví dụ, bạn có thể viết các chương trình socket giống hệt nhau mà không quan tâm đến dữ liệu được truyền trên đường truyền vật lý như nào (serial, thin Ethernet, AUI, whatever), do các chương trình ở các mức thấp hơn đối phó với nó cho bạn. Thực tế, phần cứng mạng và topology là trong suốt đối với trình lập trình socket. 
- Các lớp của mô hình: Application, Presentation, Session, Transport, Network, Datalink, Physical.
- Physical Layer là phần cứng (serial, Ethernet,...). Application Layer là nơi người dùng tương tác với mạng.
- Một mô hình phân lớp phù hợp hơn với Unix có thể là:

    Application Layer (telnet, ftp,...)

    Host-to-Host Transpost Layer (TCP, UDP)

    Internet Layer (IP và định tuyến)

    Network Access Layer (Ethernet, ATM, or whatever)

- Tất cả những gì bạn phải làm cho Stream Socket là send() dữ liệu ra ngoài. Tất cả những gì bạn phải làm cho Datagram Socket là đóng gói các gói tin trong phương thức bạn chọn và sendto() nó ra. Kernel xây dựng Lớp Transport và Lớp Internet cho bạn và phần cứng thực hiện Lớp Network Access. 

## 3. structs and Data Handling**
- Phần này trình bày các kiểu dữ liệu khác nhau được sử dụng bởi giao diện socket.
- Đầu tiên: một bộ mô tả socket: int. Có hai thứ tự byte: byte quan trọng nhất (đôi khi được gọi là một "octet") đầu tiên, hoặc byte ít quan trọng nhất đầu tiên (least significant byte first). Trước đây được gọi là "Network Byte Order". Một số máy lưu trữ số nội bộ của chúng theo Network Byte Order, một số thì không. Khi tôi nói điều gì đó phải ở trong Network Byte Order, bạn phải gọi một hàm (chẳng hạn như htons()) để thay đổi nó từ "Host Byte Order". Nếu tôi không nói "Network Byte Order", thì bạn phải để nguyên giá trị "Host Byte Order". "Network Byte Order" còn được gọi là "TBig-Endian Byte Order".
- First Struct-struct sockaddr. Cấu trúc này chứa thông tin địa chỉ socket cho nhiều loại socket:

    <img src="https://2.pik.vn/20182f90fff0-f5dd-4016-a1fa-bd133e863439.jpg">

    sa_family có thể là nhiều thứ, nhưng nó sẽ là AF_INET cho mọi thứ chúng ta làm trong tài liệu. sa_data chứa địa chỉ đích và số cổng cho socket. Điều này là khá khó sử dụng vì bạn không muốn tediously pack địa chỉ trong sa_data bằng tay.

- Để đối phó với struct sockaddr, lập trình viên tạo cấu trúc song song (parallel structure): struct sockaddr_in ("in" for "Internet").

    <img src="https://2.pik.vn/201876ad0614-9441-4fc6-8235-4acef5977828.jpg">

    Cấu trúc này giúp dễ dàng tham chiếu đến các phần tử của địa chỉ socket. Lưu ý rằng sin_zero (được bao gồm để pad cấu trúc đến độ dài của một struct sockaddr) nên được đặt thành tất cả các số 0 với hàm memset(). Ngoài ra, important bit, con trỏ trỏ đến struct sockaddr_in có thể được truyền tới con trỏ trỏ tới struct sockaddr và ngược lại. Vì vậy, mặc dù connect() muốn một struct sockaddr*, bạn vẫn có thể sử dụng struct sockaddr_in và cast nó vào phút cuối! Ngoài ra, lưu ý rằng sin_family tương ứng với sa_family trong struct sockaddr và nên được đặt thành “AF_INET”. Cuối cùng, sin_port và sin_addr phải nằm trong Network Byte Order!

- “Nhưng,” bạn phản đối, “làm thế nào có thể toàn bộ cấu trúc,struct in_addr sin_addr, trong Network Byte Order?” Câu hỏi này yêu cầu kiểm tra cẩn thận cấu trúc struct in_addr, một trong những union tồi tệ nhất còn tồn tại:

    <img src="https://2.pik.vn/20189b9b636e-e189-421d-b3dd-3e14d1c0da0b.jpg">

    Nó từng là một union, nhưng bây giờ nó dường như đã biến mất. Vì vậy, nếu bạn đã khai báo ina là kiểu struct sockaddr_in, thì ina.sin_addr.s_addr sẽ tham chiếu đến 4 byte địa chỉ IP (trong Network Byte Order). Lưu ý rằng ngay cả khi hệ thống của bạn vẫn sử dụng God-awful union cho struct in_addr, bạn vẫn có thể tham chiếu đến 4 byte địa chỉ IP theo cách giống như làm ở trên (điều này do #defines.)

**3.1 Convert the Natives**
- Phần này nói về Network to Host Byte Order.
- Có 2 loại cần phải chuyển đối (convert): short (2 byte) và long (4 byte). Các hàm này hoạt động với các biến không dấu (unsigned). Giả sử bạn muốn chuyển đổi short từ Host Byte Order sang Network Byte Order. Bắt đầu với “h” cho “host”, theo sau với “to”, rồi “n” cho “network” và “s” cho “short”: h-to-n-s, hoặc htons() (đọc: “Host to Network Short”).
- Bạn có thể sử dụng mọi kết hợp của "n", "h", "s" và "l" bạn muốn, không kể những thứ thực sự ngu ngốc. Ví dụ, không có hàm stolh() ("Short to Long Host"). Nhưng mà có:

    htons() host to network short

    htonl() host to network long

    ntohs() network to host short

    ntohl() network to host long

- Hãy nhớ: đặt các byte vào Network Byte Order trước khi đẩy chúng lên mạng.
- Cuối cùng: tại sao sin_addr và sin_port cần phải ở trong Network Byte Order trong struct sockaddr_in, nhưng sin_family thì không? Câu trả lời: sin_addr và sin_port tương ứng được đóng gói trong gói ở lớp IP và UDP. Vì vậy, chúng phải ở trong Network Byte Order. Tuy nhiên, trường sin_family chỉ được sử dụng bởi kernel để xác định loại địa chỉ mà cấu trúc chứa, vì vậy nó phải ở trong Host Byte Order. Ngoài ra, do sin_family không được gửi lên trên mạng nên nó có thể ở trong Host Byte Order.

**3.2 IP address and How to deal with them**
- Thật may mắn, có một loạt các chức năng cho phép bạn thao tác địa chỉ IP. Không cần thực hiện chúng bằng tay và gán vào kiểu long với toán tử <<.
- Trước tiên, giả sử có 1 struct sockaddr_in ina và một địa chỉ IP “10.12.110.57” mà bạn muốn lưu trữ. Hàm inet_addr()), chuyển đổi một địa chỉ IP dạng số và dấu chấm thành kiểu unsigned long. Việc chuyển nhượng có thể được thực hiện như sau:

    ina.sin_addr.s_addr = inet_addr("10.12.110.57");

- Chú ý rằng inet_addr() trả về địa chỉ trong Network Byte Order mà không cần gọi htonl().
- Đoạn mã trên không mạnh mẽ vì không có cơ chế kiểm tra lỗi. inet_addr() trả về -1 khi có lỗi, nhưng số nhị phân (không âm)bằng -1 khi địa chỉ IP là 255.255.255.255! Đó là địa chỉ quảng bá! Hãy nhớ kiểm tra lỗi đúng cách.
- Thực ra, bạn có thể sử dụng interface inet_aton() (aton-ascii to network) thay vì sử dụng inet_addr():

    #include <sys/socket.h>

    #include <netinet/in.h>

    #include <arpa/inet.h>

    int inet_aton (const char *cp, struct in_addr *inp);

- Đây là cách sử dụng mẫu khi đóng gói một cấu trúc sockaddr_in (ví dụ này sẽ có ý nghĩa hơn khi bạn đọc đến các phần bind() và connect()):

    struct sockaddr_in my_addr;

    my_addr.sin_family = AF_INET; // host byte order

    my_addr.sin_port = htons(MYPORT); //short, network byte order

    inet_aton("10.12.110.57",&(my_addr.sin_addr));

    memset(my_addr.sin_zero, '\0', sizeof my_addr.sin_zero);

- inet_aton() không giống như tất cả các hàm liên quan đến socket khác, trả về số khác 0 khi thành công, và 0 khi thất bại. Địa chỉ được truyền lại trong inp.
- Không may, không phải tất cả các nền tảng đều triển khai inet_aton() mặc dù việc sử dụng nó được ưa thích hơn. Trong tài liệu này, inet_addr() sẽ được sử dụng nhiều hơn.
- Bây giờ, cần chuyển đối chuỗi địa chỉ IP thành biểu diễn nhị phân của chúng. Nếu có struct in_addr và muốn in ra màn hình ký tự dạng thập phân của địa chỉ IP, thì sử dụng hàn inet_ntoa() (ntoa-network to ascii):

    printl("%s", inet_ntoa(ina.sin_addr));

- Chú ý rằng inet_ntoa() coi struct in-addr như là 1 đối số (argument), không phải kiểu long. Ngoài ra, nó trả về con trỏ tới kiểu char. Nó trỏ đến một mảng char được lưu trữ tĩnh trong inet_ntoa() để mỗi khi bạn gọi inet_ntoa() nó sẽ ghi đè địa chỉ IP cuối cùng mà bạn yêu cầu. VD:

    char *a1, *a2;

    a1=inet_ntoa(inal.sin_addr); //this is 192.168.4.14

    a2=inet_ntoa(ina2.sin_addr); //this is 10.12.110.57

    printf("address 1: %s\n", a1);

    printf("address 2: %s\n", a2);

    In ra màn hình:

        address 1: 10.12.110.57

        address 2: 10.12.110.57

- Nếu bạn muốn lưu địa chỉ này, strcpy() sẽ lưu địa chỉ đó và mảng ký tự của riêng bạn.

*3.2.1 Private (or Disconnected) Network*
- Rất nhiều nơi có tường lửa (firewall) để ẩn mạng khỏi phần còn lại của thế giới để bảo vệ mình. Thông thường, các firewall dịch địa chỉ IP từ nội miền sang ngoại miền (cái mà tất cả mọi người trên thế giới đều biết) bằng cách sử dụng NAT (Network Address Translation).
- Không cần lo lắng quá nhiều về NAT, vì quá trình này hoàn toàn trong suốt với bạn. 
- VD: tôi có 1 firewall đặt tại nhà, 2 địa chỉ IP tĩnh được phân bổ cho bởi công ty DSL nhưng có đến 7 máy tính kết nối mạng. Sao có thể như thế được? Hai máy tính không thể chia sẻ cùng một địa chỉ IP, nếu không thì dữ liệu sẽ không biết địa chỉ nào để gửi đến! 
- Câu trả lời là: các máy tính không cần dùng chung 1 địa chỉ IP. Có đến 24 triệu địa chỉ IP private được gán cho nó. Đây là những gì đang xảy ra:
    
    Nếu tôi đăng nhập vào máy tính từ xa, nó cho tôi biết tôi đã đăng nhập từ địa chỉ 64.81.52.10 (không phải là IP thực sự của tôi). Nhưng nếu hỏi máy tính của tôi địa chỉ IP của nó là gì, nó nói 10.0.0.5. Ai đang dịch từ địa chỉ IP này sang địa chỉ IP khác? Đúng vậy, firewall, bằng cách sử dụng NAT!

    10.x.x.x là một trong số ít các mạng được sử dụng trong mạng nội bộ hoặc trên các mạng bị ngắt kết nối hoặc trên các mạng phía sau firewall. Các chi tiết về số mạng sử dụng nội bộ được nêu trong RFC 1918, nhưng thông thường là 10.x.x.x và 192.168.x.x, trong đó x là 0-255; ít phổ biến hơn là 172.y.x.x, trong đó y đi từ 16 đến 31.
    
## 4. System Calls or Bust











