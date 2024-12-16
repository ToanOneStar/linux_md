Trong bài viết này chúng ta sẽ tìm hiểu về cơ chế IPC-Pipes trong linux.

# 1. Pipes là gì?

Pipes là một trong số các phương thức IPC được sử dụng trong việc truyền thông liên tiến trình. Dữ liệu được chuyển theo luồng (stream) theo cơ chế FIFO (first in first out). Một pipe có kích thước giới hạn (thường là 4096 ký tự).
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_6f2729e2d2.png)

Pipes chỉ là giao tiếp một chiều, tức là chúng ta chỉ có thể sử dụng pipe sao cho một quá trình thực hiện ghi vào pipe và quá trình kia đọc từ pipe.

Khi tạo một pipe, nó sẽ nằm trong RAM và được coi là một "virtual file". Pipes có thể sử dụng trong quá trình tạo process.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_c8801e9760.png)
Khi một process ghi vào "virtual file" (hoặc có thể hiểu là pipe) thì một tiến trình liên quan (related-process) khác có thể đọc dữ liệu từ nó.

Một tiến trình chỉ có thể sử dụng một pipe do nó tạo ra hay kế thừa từ tiến trình cha. Hệ điều hành cung cấp các lời gọi hệ thống read/write cho các tiến trình thực hiện thao tác đọc/ghi dữ liệu trong pipe:

**Reading from a pipe**:

- Nếu cố gắng đọc dữ liệu từ một pipe "rỗng", thì đầu read sẽ block cho đến khi đọc được ít nhất 1 bytes.
- Nếu đầu write của một đường ống bị đóng, đầu read đọc lại toàn bộ dữ liệu còn lại trong pipe và return 0.

**Pipes have a limited capacity**:

- Một pipe chỉ đơn giản là một bộ đệm được duy trì trong bộ nhớ.
- Bộ đệm này có dung lượng tối đa. Khi một pipe đã đầy, chỉ khi khi đầu read lấy một số dữ liệu khỏi pipe thì đầu write mới có thể ghi tiếp dữ liệu vào pipe.

Có hai loại pipe:

- **Unnamed pipe**: có ý nghĩa cục bộ, dành cho các tiến trình có quan hệ cha con
- **Named pipe** (còn gọi là FIFO): có ý nghĩa toàn cục, sử dụng cho các tiến trình không có quan hệ cha con.

# 2. Tạo và sử dụng Pipes

## 2.1. Tạo Pipes

Hệ thống cung cấp hàm pipe() để tạo đường ống có khả năng đọc/ghi. Sau khi tạo ra, có thể dùng đường ống để giao tiếp giữa hai tiến trình. Đọc/ghi đường ống hoàn toàn tương đương với đọc/ghi file.

```c
#include <unistd.h>
int pipe(int fds[2]);
```

Khi gọi hàm pipe() sẽ tạo ra một cặp filedescriptors đặt trong một mảng gồm 2 phần tử:

- fds[0] file descriptor chiều read của pipe.
- fds[1] file descriptor chiều write của pipe.

Trả về 0 nếu thành công, -1 nếu thất bại.

Nếu ta tạo pipe trước khi gọi fork(). Thì parent and child (related-process) có thể giao tiếp thông qua pipe.

Các tác vụ trên pipe:

`write(int fd, const void *buf, size_t count)`
`read(int fd, const void *buf, size_t count)`

## 2.2. Giao tiếp hai chiều

Chỉ cần tạo hai pipes để gửi dữ liệu theo từng hướng giữa hai quá trình.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_a83f16c5f4.png)
Nếu sử dụng kỹ thuật này, thì chúng ta cần cảnh giác với nguy cơ xảy ra tình trạng tắc nghẽn (deadlock) : một pipe bị giới hạn về kích thước, do vậy nếu cả hai pipe nối kết hai tiến trình đều đầy(hoặc đều trống) và cả hai tiến trình đều muốn ghi (hay đọc) dữ liệu vào pipe(mỗi tiến trình ghi dữ liệu vào một pipe), chúng sẽ cùng bị khóa và chờ lẫn nhau mãi mãi.

Trong trường hợp: Parent đóng vai trò writer và Child đóng vai trò reader.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_7ff309adb0.png)
Trong trường hợp: Parent đóng vai trò reader và Child đóng vai trò writer.

## 2.3. Ví dụ

Ta sẽ lấy ví dụ về việc tạo pipe, đọc ghi dữ liệu từ pipe. Process cha sẽ ghi dữ liệu vào pipe và process con sẽ đọc dữ liệu từ pipe.

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

int main()
{
    int pipe_fd[2];
    char write_msg[] = "Hello from the pipe!";
    char read_msg[100];

    if (pipe(pipe_fd) == -1)
    {
        perror("Pipe failed");
        return 1;
    }

    pid_t pid = fork();
    if (pid < 0)
    {
        perror("Fork failed");
        return 1;
    }
    if (pid > 0)
    {
        close(pipe_fd[0]);
        printf("Parent: Writing to pipe...\n");
        write(pipe_fd[1], write_msg, strlen(write_msg) + 1);
        close(pipe_fd[1]);
    }
    else
    {
        close(pipe_fd[1]);
        read(pipe_fd[0], read_msg, sizeof(read_msg));
        printf("Child: Read from pipe: %s\n", read_msg);
        close(pipe_fd[0]);
    }
    return 0;
}
```

Biên dịch và chạy chương trình ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_098ae079a8.png)

# 3. FIFOs – named Pipes

## 3.1. Định nghĩa

Đây là một khái niệm mở rộng của pipes. Pipes truyền thống thì không được đặt tên và chỉ tồn tại trong suốt vòng đời của process. Named Pipes có thể tồn tại miễn là hệ thống còn hoạt động. Vượt ra ngoài vòng đời của process. Có thể xóa đi nếu không còn sử dụng.

Sự khác biệt chính là FIFOs có tên trong hệ thống tệp và được mở giống như một tệp thông thường.

Một file FIFO là một file đặc biệt được lưu trong bộ nhớ cục bộ. được tạo ra bởi hàm mkfifo() trong C.

## 3.2. Tạo FIFOs từ trình shell

Ta gõ lệnh tạo FIFOs trực tiếp từ shell command.

`mkfifo  -m mode pathname`

Ta có ví dụ:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_2723c1b667.png)

## 3.3. Tạo FIFOs từ source code

FIFOs là một loại tệp, chúng ta có thể sử dụng tất cả các lệnh gọi hệ thống được liên kết với nó ví dụ open, read, write, close.

```c
int mkfifo(const char *pathname, mode_t mode);
```

Tạo một FIFO mới với đường dẫn cụ thể. Trả về hai file descriptor (fd) nằm trong fds. Trả về 0 nếu thành công, -1 nếu thất bại.

Các đối số:

- pathname: tên file FIFO.
- mode: các quyền đối với file FIFO.

## 3.4. Ví dụ

Ta sẽ lấy ví dụ về chương trình giao tiếp giữa 2 process sử dụng FIFOs.

Chương trình ghi:

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#define FIFO_FILE "./myfifo"
#define BUFF_SIZE 1024

int main()
{
    char buff[BUFF_SIZE];
    int fd;
    mkfifo(FIFO_FILE, 0666);
    while (1)
    {
        printf("Message to comsumer : ");
        fflush(stdin);
        fgets(buff, BUFF_SIZE, stdin);

        fd = open(FIFO_FILE, O_WRONLY);
        write(fd, buff, strlen(buff) + 1);
        return 0;
    }
}
```

Chương trình đọc:

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#define FIFO_FILE "./myfifo"
#define BUFF_SIZE 1024

int main()
{
    char buff[BUFF_SIZE];
    int fd;
    mkfifo(FIFO_FILE, 0666);
    while (1)
    {
        fd = open(FIFO_FILE, O_RDONLY);
        read(fd, buff, BUFF_SIZE);
        printf("producer message: %s", buff);
        close(fd);
        return 0;
    }
}
```

Biên dịch và chạy chương trình bên phía producer ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_790097be3f.png)
Biên dịch và chạy chương trình bên phía consumer ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/29/image_24ba06c675.png)

# 4. Kết luận

Như vậy trong bài viết này ta đã nắm được cơ chế IPC-Pipes, biết tạo và truyền dữ liệu qua pipes và qua FIFOs-named pipes.
