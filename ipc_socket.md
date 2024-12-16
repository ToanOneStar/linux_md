Trong bài viết này ta sẽ tìm hiểu Socket. Socket là cơ chế truyền thông cho phép các tiến trình có thể giao tiếp với nhau dù các tiến trình ở trên cùng thiết bị hay khác thiết bị.

# 1. Giới thiệu

Socket được đại diện bởi một file socket descriptor. Socket được gọi là file point to point, khác với các file thông thường socket cho phép data được gửi ra khỏi máy tính để đi vào mạng Internet. Thông tin được mô tả trong một file socket sẽ gồm: Domain, Type, Protocol.

1. **Domain**: Tiến trình cần giao tiếp nằm trên cùng thiết bị hay khác thiết bị. Socket có 2 domain chính là:

- Internet Domain: Socket hoạt động trên miền Internet Domain nếu các tiến trình gửi nhận thông tin nằm trên các host khác nhau.
- UNIX Domain: Socket hoạt động trên miền UNIX Domain nếu các tiến trình gửi nhận thông tin nằm trên cùng một host.

2. **Type**: Mô tả cơ chế truyền nhận thông tin. Socket có 2 type phổ biến là:

- Stream: tin cậy (đảm bảo dữ liệu nhận được theo thứ tự, có thông báo nếu xảy ra lỗi), yêu cầu tạo kết nối trước khi trao đổi dữ liệu, thường dùng khi cần truyền chuỗi bit.
- Datagram: không tin cậy (dữ liệu nhận được có thể không theo thứ tự, dữ liệu có thể mất trong quá trình truyền mà không có thông báo), không cần tạo kết nối trước khi trao đổi dữ liệu. Dữ liệu có thể được gửi đi ngay cả khi tiến trình đích không tồn tại, thường dùng khi cần truyền là các gói tin.

3. **Protocol**: Cách thức đóng gói dữ liệu. Từ Domain và Type sẽ có một danh sách các protocol tương ứng để ta lựa chọn. Thông thường với một Domain và Type đã chọn chỉ có 1 giao thức có thể dùng nên protocol thường có giá trị 0.

# 2. Flow hoạt động

## 2.1. Flow hoạt động của Stream Socket

Stream socket yêu cầu tạo một kết nối trước khi truyền dữ liệu. Tiến trình khởi tạo kết nối đóng vai trò là client, tiến trình nhận được yêu cầu kết nối là server.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/15/h1_8a01620f08.png)

## 2.2. Flow hoạt động của Datagram Socket

Trong Datagram socket vai trò của client và server khá mờ nhạt.

Về cơ bản các tiến trình có thể gửi dữ liệu đến một địa chỉ bất kể địa chỉ đó có tồn tại hay không.

Trong quá trình truyền nhận ta tạm coi tiến trình muốn gửi dữ liệu là client và tiến trình nhận dữ liệu là server.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/15/h2_d510e9c68c.png)

# 3. Internet Domain Socket

## 3.1. Internet Socket Address

Dùng để giao tiếp giữa các tiến trình nằm trên các thiết bị khác nhau. Bao gồm hai domain: AF_INET/AF_INET6.
Socket chỉ có một kiểu địa chỉ duy nhất là sockaddr. Tuy nhiên để tiện cho việc sử dụng với từng domain socket khác nhau người ta định nghĩa thêm các struct địa chỉ riêng cho từng domain sau đó sẽ ép kiểu về struct socaddr.

```c
#include <sys/socket.h>
struct sockaddr
	{
		sa_family_t sa_family; /* address family, AF_xxx	*/
		char sa_data[14];	   /* 14 bytes of protocol address*/
	};
```

### 3.1.1. IPv4 Socket Address

Địa chỉ của IPv4 socket được chứa trong struct sockaddr_in được định nghĩa trong file <netinet/in.h>.
Địa chỉ của IPv4 Socket đặc trưng bởi giá trị địa chỉ IPv4 và port.

```c
#include <netinet/in.h>
struct sockaddr_in
{
	sa_family_t sin_family;	 /* AF_INET */
	in_port_t sin_port;		 /* Port number */
	struct in_addr sin_addr; /* IPv4 address */
};
struct in_addr {            	/* IPv4 4-byte address */
    in_addr_t s_addr;       	 /* Unsigned 32-bit integer */
};
```

### 3.1.2. IPv6 Socket Address

Địa chỉ của IPv6 Socket được chứa trong struct sockaddr_in6 được định nghĩa trong file <netinet/in.h>
Địa chỉ của IPv6 Socket đặc trưng bởi giá trị địa chỉ IPv6 và port.

```c
#include <netinet/in.h>
struct sockaddr_in6 {
    sa_family_t sin6_family;   	/* Address family (AF_INET6) */
    in_port_t sin6_port;       	/* Port number */
    uint32_t sin6_flowinfo;    	/* IPv6 flow information */
    struct in6_addr sin6_addr; 	/* IPv6 address */
    uint32_t sin6_scope_id;    	/* Scope ID (new in kernel 2.4) */
};

struct in6_addr {              	/* IPv6 address structure */
    uint8_t s6_addr[16];       	/* 16 bytes == 128 bits */
};
```

## 3.2. Chuyển đổi địa chỉ socket

Địa chỉ của Internet Socket được đặc trưng bởi địa chỉ IP và port. Chúng đều được lưu trữ trên thiết bị dưới dạng số integer.

Các thiết bị sử dụng các kiến trúc phần cứng khác nhau sẽ lưu trữ địa chỉ theo thứ tự khác nhau.

Dữ liệu được lưu trữ theo 2 kiểu Big-endian và Little-endian. Big-endian: byte có trọng số cao sẽ được xếp vào địa chỉ thấp. Little-edian thì ngược lại so với Big-endian.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/16/h3_9e03aebb49.png)

Socket sử dụng một quy ước chung về cách lưu trữ địa chỉ gọi là network byte order (thật ra là theo thứ tự của Big-endian).

## 3.3. Các hàm được sử dụng chuyển đổi địa chỉ Socket

Chúng ta có các hàm được sử dụng để chuyển đổi địa chỉ là

- **htons()** (Host to Network Short).
- **ntohs()** (Network to Host Short).
- **htonl()** (Host to Network Long).
- **ntohl()** (Network to Host Long).
  Trong đó Short thực hiện chuyển đổi với loại 2 bytes (sử dụng chuyển đổi cho port), Long là 4 bytes (sử dụng chuyển đổi cho network).

```c
#include <arpa/inet.h>
uint16_t htons(uint16_t host_uint16);
uint32_t htonl(uint32_t host_uint32);
uint16_t ntohs(uint16_t net_uint16);
uint32_t ntohl(uint32_t net_uint32);
```

# 4. Unix Domain Socket

Linux hỗ trợ Internet Socket để giao tiếp giữa các tiến trình trên cùng một thiết bị. Tuy Internet Socket cũng có thể truyền thông trên cùng thiết bị nhưng Unix socket nhanh và dễ sử dụng hơn.

Domain là AF_UNIX

UNIX Domain Socket hỗ trợ 2 loại socket chính là: SOCK_STREAM (stream) và SOCK_DGRAM (datagram).

Sau khi chạy bind() để gán địa chỉ cho socket một socket file sẽ được tạo theo path_name.

```c
struct sockaddr_un {
    sa_family_t sun_family;    /* Always AF_UNIX */
    char sun_path[108];          /* Null-terminated socket pathname */
};
```

Không thể gán một socket vào một path_name đã tồn tại. Một path_name chỉ có thể được gán cho một socket. Path_name có thể là đường dẫn tuyệt đối hoặc tương đối.

Tuy socket được đặc trưng bởi một socket file nhưng ta không thể dùng open() để kết nối socket.
Sau khi socket được đóng hay chương trình đã tắt file path_name vẫn còn. Nếu muốn xóa file này ta có thể dùng unlink() hoặc remove().

Để kết nối hoặc gửi dữ liệu tới socket yêu cầu tiến trình phải có quyền write với file path_name.
Lệnh bind() sẽ tạo socket file với đầy đủ các quyền cho tất cả tài khoản nhưng ta có thể thay đổi quyền hạn của chúng bằng umask() hoặc đơn giản là thay đổi quyền của thư mục chứa file socket.

# 5. Ví dụ

Ta có ví dụ sau về việc truyền nhận data giữa client và server. Trong ví dụ này server sẽ gửi data chứa thời gian thực.
Code trên server:

```c
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#include <stddef.h>
#include <string.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <time.h>

#define PORT 2000

void main()
{
    int listenfd = -1;
    int confd = -1;
    struct sockaddr_in server_addr;
    char send_buffer[1024];
    time_t ticks;

    memset(send_buffer, 0, sizeof(send_buffer));
    memset(&server_addr, 0, sizeof(server_addr));

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    server_addr.sin_port = htons(PORT);

    bind(listenfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    listen(listenfd, 10);

    while (1)
    {
        confd = accept(listenfd, (struct sockaddr *)NULL, NULL);
        ticks = time(NULL);
        sprintf(send_buffer, "Server reply %s", ctime(&ticks));
        write(confd, send_buffer, strlen(send_buffer));
        close(confd);
    }
    close(listenfd);
}
```

Code trên client

```c
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#include <stddef.h>
#include <string.h>
#include <arpa/inet.h>
#include <time.h>
#include <stdio.h>

#define PORT 2000
void main()
{
    int sockfd = -1;
    struct sockaddr_in server_addr;
    char recv_buffer[1024];
    time_t ticks;

    memset(recv_buffer, 0, sizeof(recv_buffer));
    memset(&server_addr, 0, sizeof(server_addr));

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    server_addr.sin_port = htons(PORT);

    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == 0)
    {
        read(sockfd, recv_buffer, sizeof(recv_buffer) - 1);
        printf("\n%s\n", recv_buffer);
        close(sockfd);
    }
}
```

Chạy hai file ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/15/Screenshot_2024_11_15_235734_3317daf367.png)![](https://assets.devlinux.vn/uploads/editor-images/2024/11/15/Screenshot_2024_11_15_235803_4b61ad5712.png)

# 6. Kết luận

Sau bài viết này chúng ta cần nắm được cơ chế hoạt động của socket, về Internet Domain và Unix Domain
