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
- Các cuộc gọi hệ thống cho phép bạn truy cập vào chức năng mạng của Unix. Khi bạn gọi một trong các hàm này, kernel sẽ tiếp nhận và thực hiện tất cả công việc cho bạn một cách tự động.

**4.1 socket() -Get the file descriptor**

    #include <sys/types.h>

    #include <sys/socket.h>

    int socket(int domain, int type, int protocol);

- Trước tiên, domain phải được đặt thành “PF_INET”. Tiếp theo,type cho kernel biết loại socket này là: SOCK_STREAM hoặc SOCK_DGRAM. Cuối cùng, đặt protocol là “0” để socket() chọn giao thức chính xác dựa trên type. (có nhiều domain, type hơn tôi đã liệt kê. Xem trang socket(). Ngoài ra, có cách "tốt hơn" để đặt protocol, nhưng đặt bằng 0 hoạt động trong 99,9% các trường hợp. Xem trang getprotobyname() nếu bạn hiếu kỳ.)
- socket() trả về cho bạn các miêu tả socket mà bạn có thể sử dụng trong các cuộc gọi hệ thống sau này, hoặc trả về -1 nếu có lỗi. Biến toàn cục (global) errno được đặt thành giá trị của lỗi (xem trangperor()).
- PF_INET gần giống với AF_INET mà bạn đã sử dụng khi khởi tạo trường sin_family trong struct sockaddr_in. Trên thực tế, chúng liên quan chặt chẽ đến mức chúng thực sự có cùng giá trị, nhiều lập trình viên sẽ gọi socket() và chuyển AF_INET làm đối số đầu tiên thay cho PF_INET. Có lẽ nào một dải địa chỉ (“AF” trong “AF_INET”) có thể hỗ trợ một số giao thức được giới thiệu bởi các họ giao thức (“PF” trong “PF_INET”)? Điều đó đã không xảy ra. Vì vậy, điều chính xác nhất cần làm là sử dụng AF_INET trong struct sockaddr_in và PF_INET trong cuộc gọi tới socket().

**4.2 bind() - What port am I on?**
- Một khi bạn đã tạo ra socket, bạn phải liên kết socket đó với 1 port trên máy của bạn. (Điều này thường được thực hiện xong nếu bạn listen() cho các kết nối đến từ 1 cổng cụ thể-MUDs thực hiện  điều này khi chúng cho bạn biết "telnet to x,y,z port 6969"). Số port được sử dụng bởi kernel để khớp với số gói tin đến bộ mô tả socket của một quy trình nhất định. Nếu bạn chỉ thực hiện 1 connect(), điều này dường như không cần thiết.
- Đây là tóm tắt cho cuộc gọi hệ thống bind(): 

    #include <sys/types.h>

    #include <sys/socket.h>

    int bind(int sockfd, struct sockaddr *my_addr, int addrlen);

    //sockfd là file mô tả socket được trả về bởi socket()

    //my_addr là con trỏ tới struct sockaddr chức các thông tin về địa chỉ của bạn, cụ thể là cổng và địa chỉ IP.

    //addrlen có thể đặt thành sizeof *my_addr hay sizeof(struct sockaddr)

- VD: 

    <img src="https://2.pik.vn/20183551cba2-bdfd-4935-9dc3-c6796ceac0b4.jpg">

    Chú ý: my_addr.sin_port nằm trong Network Byte Order. Vì vậy, là my_addr.sin_addr.s_addr. Một điều cần lưu ý là các tệp tiêu đề có thể khác từ hệ thống này qua hệ thống khác. Để chắc chắn, bạn nên kiểm tra lại trong man page.

- Cuối cùng, 1 số quá trình nhận địa chỉ IP được and/or với port 1 cách tự động:

    my_addr.sin_port=0; // chọn ngẫu nhiên 1 cổng k đc sd

    my_addr.sin_addr.s_addr=INADDR_ANY; //sd địa chỉ IP of bạn

    Bằng cách đặt my_addr.sin_port = 0, bind() sẽ chọn cổng cho bạn. Tương tự, đặt my_addr.sin_addr.s_addr = INADDR_ANY thì địa chỉ IP của máy sẽ được điền tự động trong quá trình chạy.

- Nếu bạn chú ý đến những điểm nhỏ, bạn có thể đã thấy rằng tôi không đặt INADDR_ANY vào trong Network Byte Order. Tuy nhiên, INADDR_ANY thực sự là zero. Zero vẫn có số 0 trong bit ngay cả khi bạn sắp xếp lại các byte:

    my_addr.sin_port=htons(0); //chọn ngẫu nhiên cổng k đc sd

    my_addr.sin_addr.s_addr=hton1(INADDR_ANY); //sd đc IP of b

- Hầu hết code bạn gặp phải sẽ không bận tâm chạy INADDR_ANY thông qua htonl().
- bind () cũng trả về -1 trên lỗi và đặt errno thành giá trị của lỗi.
- Một điều cần lưu ý khi gọi bind(): không được sử dụng các cổng dưới 1024, vì chúng được sử dụng cho các tổ chức (trừ khi bạn là superuser)! Bạn có thể sử dụng bất kỳ cổng nào trong các cổng còn lại đến dưới 65535 (miễn là chúng chưa được chương trình khác sử dụng).
- Đôi khi, bạn có thể nhận thấy, bạn cố gắng chạy lại máy chủ,bind() không thành công, xác nhận “Địa chỉ đã được sử dụng”. Điều đó có nghĩa là gì? Một số socket đã được kết nối vẫn còn xung quanh kernel, và nó hogging cổng. Bạn có thể chờ cho đến khi nó mất (một phút hoặc lâu hơn), hoặc thêm code vào chương trình cho phép nó sử dụng lại cổng, như sau:

    int yes=1;

    //char yes='1'; // Solaris people use this

    //lose the pesky "Address already in use" error message

    if (setsockopt(listener,SOL_SOCKET, SO_REUSEADDR, &yes,sizeof(int)) == -1){

        perror("setsockopt");

        exit(1);
    }

- Một lưu ý cuối cùng nhỏ về bind (): có những lúc bạn hoàn toàn không thể gọi nó. Nếu bạn connect() từ một máy từ xa và bạn không quan tâm đến cổng local của bạn (như trường hợp telnet mà chỉ quan tâm đến cổng từ xa), bạn có thể chỉ cần gọi connect(), nó sẽ kiểm tra xem ổ cắm có được gắn kết không và sẽ bind() nó vào một cổng local chưa sử dụng nếu cần thiết.

**4.3 connect() - Hey you!**
- Hãy giả vờ trong vài phút bạn là ứng dụng telnet. Người dùng lệnh cho bạn để có được một bộ mô tả tập tin socket. Bạn tuân thủ và gọi socket(). Tiếp theo, người dùng yêu cầu bạn kết nối với “10.12.110.57” trên cổng “23” (cổng telnet tiêu chuẩn). 
- Cuộc gọi connect() như sau:

    #include <sys/types.h>

    #include <sys/socket.h>

    int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);

    //sockdf là bộ mô tả tệp tin socket vùng lân cận, được trả về bởi socket().
    
    //serv_addr là truct sockaddr chứa cổng đích và địa chỉ IP
    
    //addrlen có thể được đặt thành sizeof *serv_addr hoặc sizeof(struct sockaddr).

- VD:

    <img src="https://2.pik.vn/201829d42a96-ed3c-4c8d-b612-667f26f6f383.jpg">

    <img src="https://2.pik.vn/2018646c82e0-563b-470d-b907-906ab316e6da.jpg">

- Một lần nữa, hãy chắc chắn kiểm tra giá trị trả về từ connect() - nó sẽ trả về -1 nếu có lỗi và thiết lập biến errno. 
- Ngoài ra, lưu ý rằng chúng ta đã không gọi bind(). Về cơ bản, chúng ta không quan tâm đến số cổng local của mình;chúng ta chỉ quan tâm đến nơi chúng ta đến (cổng từ xa). Kernel sẽ chọn một cổng local cho ta và trang web chúng ta kết nối sẽ tự động nhận thông tin này từ chúng ta. 

**4.4 listen() -Will somebody please call me?**
- Điều gì sẽ xảy ra nếu bạn không muốn kết nõi với máy từ xa (remote host)? Bạn muốn chờ đợi các kết nối đến và xử lý chúng theo một số cách. Quá trình gồm hai bước: đầu tiên listen(), sau đó accept().
- Gọi listen khá đơn giản, nhưng cần giải thích một chút:

    int listen(int sockfd, int backlog);

    //sockfd là file mô tả socket thông thường từ hệ thống socket().

    //backlog là số kết nối được chấp nhận vào hàng đợi đến. Các kết nối đến sẽ phải đợi trong hàng đợi đến khi bạn accept() và đây là giới hạn về số lượng có thể vào hàng đợi. Hầu hết các hệ thống ngầm giới hạn số lượng là 20, bạn có thể đặt từ 5 đến 10.

- Như thường lệ, listen() trả về -1 và đặt errno nếu có lỗi.
- Gọi bind() trước khi gọi listen() hay kernel sẽ nghe trên 1 cổng ngẫu nhiên. Nếu bạn định nghe các kết nối đến, chuỗi các cuộc gọi hệ thống bạn sẽ thực hiện là:

    socket();

    bind();

    listen();

    //accept() goes here

**4.5 accept() -"Thank you for calling port 3490**
- Điều gì sẽ xảy ra: ai đó ở rất xa sẽ cố gắng connect() với máy của bạn trên một cổng mà bạn đang listen(). Kết nối của họ sẽ được xếp hàng chờ đợi để được accept(). Bạn gọi accept()và yêu cầu nó nhận kết nối đang chờ xử lý. Nó sẽ trả lại cho bạn một bộ mô tả tập tin socket mới (brand new socket file descriptor) để sử dụng cho kết nối đơn này! Đúng vậy, đột nhiên bạn có hai bộ mô tả tập tin socket cho 1 kết nối. Bản gốc vẫn đang lắng nghe trên cổng của bạn và bản mới được tạo ra cuối cùng cũng sẵn sàng send() và recv(). 
- Cuộc gọi ddc thực hiện như sau:

    #include <sys/types.h>

    #include <sys/socket.h>

    int accept(int sockfd, stuct sockaddr *addr, socklen_t *addrlen);

    //sockfd là bộ mô tả socket listen() . 
    
    //addr thường sẽ là con trỏ chỉ tới struct sockaddr_in cục bộ (local). Đó là nơi thông tin về sự kết nối đến sẽ tới (với nó bạn có thể xác định máy chủ nào đang gọi bạn từ port nào).  

    //addrlen là 1 biến nguyên cục bộ (local integer variable)
    có thể đặt thành sizeof *addr hay sizeof(struct sockaddr_in) trước khi địa chỉ của nó được chuyển tới accept(). Accept() sẽ không đặt nhiều byte hơn vào addr. Nếu đặt ít hơn, nó sẽ thay đổi giá trị của addrlen để phản ánh điều đó. 

- accept() trả về -1 và đặt errno nếu có lỗi xảy ra. 
- VD:

    <img src="https://2.pik.vn/201832462eef-499a-4f32-b9e9-6c98db983511.jpg">

- Một lần nữa, lưu ý rằng chúng ta sẽ sử dụng bộ mô tả socket new_fd cho tất cả các cuộc gọi send() và recv(). Nếu bạn chỉ nhận được một kết nối, bạn có thể close() việc nghe sockfd  để ngăn chặn nhiều kết nối đến trên cùng một cổng, nếu bạn muốn.

**4.6 send() and recv() -Talk to me, baby!**
- Có 2 hàm để truyền thông qua Stream Socket hoặc Datagram Socket. Nếu bạn muốn sử dụng Datagram Socket không được kết nối thông thường, bạn sẽ cần đọc phần sendto() và revcfrom() bên dưới. 
- send():

    int sent(int sockfd, const void *msg, int len, int flags);

    //sockfd là bộ mô tả socket bạn muốn gửi dữ liệu đến (cho dù đó là cái được trả về bởi socket() hoặc cái bạn nhận được với accept()).

    //msg là con trỏ tới dữ liệu bạn muốn gửi.

    //len là chiều dài của dữ liệu dạng byte.

    //chỉ cần thiết lập flags bằng 0 (xem phần send() ở man page để biết thêm thông tin về các cờ liên quan).

- VD: 

    <img src="https://2.pik.vn/201822c584b1-c5a3-4411-85b0-0d5316563c67.jpg">

- send() trả về số lượng byte thực sự được gửi đi - có thể ít hơn số byte bạn gửi. Thỉnh thoảng, bạn yêu cầu nó gửi toàn bộ dữ liệu và nó không thể xử lý. Nó sẽ kích hoạt càng nhiều dữ liệu cần tốt và tin tưởng rằng bạn sẽ gửi phần dữ liệu còn lại sau. Nhớ rằng, nếu giá trị trả về bởi send() không khớp với giá trị len, thì tùy thuộc vào bạn có thể gửi hết phần còn lại của string hay không. Tin tốt là: nếu các gói nhỏ (nhở hơn 1K hoặc hơn) thì có thể gửi toàn bộ mọi thứ trong 1 lần. send() trả về -1 nếu xảy ra lỗi, và errno là số lỗi.
- recv():

    int recv(int sockfd, void *buf, int len, usigned int flags);

    //sockfd là bộ mô tả socket để đọc.

    //buf là bộ đệm (buffer) để đọc thông tin vào.

    //len là chiều dài tối đa của bộ đệm.

    //flags có thể đặt thành 0. (xem recv() ở phần man page để biết thêm thông tin về flag).

- recv() trả về số lượng byte thực sự được đọc từ bộ đệm, hoặc -1 nếu xảy ra lỗi (errno được thiết lập tương ứng).
- recv() có thể trả về 0. Điều đó có nghĩa là: phía đối diện đã đóng kết nối với bạn. 

**4.7 sendto() and recvfrom() -Talk to me, DGRAM-style**
- Vì Datagram Socket không được kết nối với một máy chủ từ xa,hãy đoán xem chúng ta cần cung cấp thông tin gì trước khi gửi gói? Địa chỉ đích!

    int sendto(int sockfd, const void *msg, int len, unsigned int flags, const struct sockaddr *to,  socklen_t tolen);

- Như bạn có thể thấy, cuộc gọi này cơ bản giống với cuộc gọi send() với 2 thông tin được thêm vào. to là con trỏ tới struct sockaddr chứa địa chỉ IP đích và port. tolen, an int deep-down có thể được thiết lập thành sizeof *to hoặc sizeof(struct sockaddr).
- Giống send(), sendto() trả về số lượng byte thực tế được gửi (có thể ít hơn số byte bạn yêu cầu gửi đi), hoặc trả về -1 nếu xảy ra lỗi. 
- Giống như recv(), recvfrom() có công thức:

    int recvfrom(int sockaddr, void *buf, int len, unsigned int flag, struct sockaddr *from, int *fromlen);

- Giống recv(), với một vài trường được thêm vào, from là con trỏ chỉ tới struct sockaddr cục bộ sẽ được điền với địa chỉ IP và port của máy ban đầu. fromlen là con trỏ tới int cục bộ cần được khởi tạo thành sizeof *from hoặc sizeof(struct, sockaddr). Khi hàm này trả về, fromlen sẽ chứa chiều dài thực sự của địa chỉ được lưu trong from.
- recvfrom() trả về số lượng byte nhận được, hoặc trả về -1 nếu xảy ra lỗi (với error được thiết lập tương ứng).
- Nhớ rằng, nếu bạn connect() 1 Datagram Socket, bạn có thể chỉ cần sử dụng send() và recv() cho tất cả các giao dịch của bạn. Socket vẫn là Datagram Socket và các gói vẫn sử dụng UDP,nhưng socket interface sẽ tự động thêm thông tin đích và nguồn cho bạn.

**4.8 close() and shutdown() -Get outta my face!**
- Hàm close():

    close(sockaddr);

- Điều này sẽ ngăn chặn bất kỳ việc đọc và ghi nào vào socket. Bất kỳ ai cố gắng đọc hay ghi vào socket từ thiết bị đầu cuối ở xa sẽ xảy ra lỗi.
- Chỉ trong trường hợp bạn muốn kiểm soát nhiều hơn về cách socket đóng, bạn có thể sử dụng hàm shutdown(). Nó cho phép bạn cắt liên lạc theo 1 hướng nhất định hoặc cả 2 hướng (giống close()):

    int shutdown(int sockfd, int how);

    //sockfd là file mô tả socket bạn muốn shutdown.

    //how là 1 trong các lựa chọn sau đây: 0 (không được phép nhận thêm); 1 (không được phép gửi thêm); 2 (không được phép nhận và gửi thêm)

- shutdown() trả về 0 nếu thành công, -1 nếu xảy ra lỗi (với errno được thiết lập cho phù hợp).
- Nếu bạn phải sử dụng shutdown() trên các Datagram Socket chưa được ngắt kết nối, nó sẽ đơn giản làm cho socket không sẵn sàng cho send() và rec() (nhớ rằng, bạn có thể sử dụng chúng nếu bạn connect() Datagram Socket của bạn).
- Chú ý quan trọng: shutdown() không thực sự đóng được file descriptor, nó chỉ thay đổi khả năng sử dụng. Để giải phóng bộ mô tả socket, bạn cần phải sử dụng close().

**4.9 getpeername() - Who are you?**
- Hàm getpeername() cho bạn biết ai ở phía bên kia được kết nối qua socket:

    #include <sys/socket.h>

    int getpeername(int sockfd, struct sockaddr *addr, int *addr);