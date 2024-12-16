Trong bài viết này chúng ta sẽ cùng nhau tìm hiểu về IPC-Shared memory

# 1. Shared memory là gì?

Shared memory là vùng nhớ cho phép cho phép nhiều tiến trình có thể cùng truy cập tới.

Làm tăng tốc độ xử lý khi các tiến trình không cần gửi gửi nhận dữ liệu cho nhau.

Sau khi được tạo vùng nhớ ảo của chương trình sẽ nằm trong khoảng không gian địa chỉ giữa vùng nhớ heap và vùng nhớ stack.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/26/image_c2822dcb7a.png)
**Vậy tại sao cần shared memory?**.Với các giao thức IPC khác như pipes fifos, message queue trong quá trình truyền nhận dữ liệu giữa các process message sẽ được copy từ user space của process thứ nhất vào kernel space và process thứ hai sẽ nhận message bằng cách copy từ kernel space tới user space của nó. Đối với Shared memory, bộ nhớ được chia sẻ được ánh xạ vào không gian bộ nhớ của cả hai process. Ngay sau khi process đầu tiên ghi dữ liệu vào bộ nhớ được chia sẻ, nó sẽ có sẵn cho process thứ hai truy cập mà không cần thực hiện bất cứ thao tác sao chép nào. Điều này khiến cho Shared memory là cơ chế nhanh nhất trong các phương thức IPC.

Có 2 phương pháp để tạo shared memory:

- Sử dụng API của shm (System V).
- Sử dụng API của mmap (POSIX).

# 2. System V Shared memory

Các bước triển khai:

- Tạo key.
- Cấp phát shared memory segment.
- Map/umap shared memory segment.
- Giải phóng shared memory segment.

## 2.1. Tạo key

Shared memory segment được kernel quản lý bởi một IPC structure. Mỗi IPC structure sẽ được xác định bởi một sô nguyên (không âm) Identifier.

Để có thể map/đọc/ghi dữ liệu vào shared memory thì các process cần phải biết được các số Identifier này. Indentifier có thể được tạo ra thông qua API **ftok()**.

```c
#include <sys/types.h>
#include <sys/ipc.h>

key_t ftok (const char *pathname, int proj_id);
```

Trong đó:

- pathname: file path
- proj_id:8 bits thấp của projectID.

## 2.2. Tạo Shared Memory segment

Mỗi segment được liên kết với một structure về permission cho việc read/write của process.
Trong đó mode là một tổ hợp bitwise.
Để tạo shared memory segment sử dụng API **shmget()**.

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int shmget (key_t key, size_t size, int shmflg);
```

Trong đó:

- key: Identifier key.
- size: kích cỡ của shared memory segment được làm tròn bội của PAGE_SIZE.
- shmflg: mode flags bao gồm IPC_CREAT(sử dụng segment liên kết với key nếu tồn tại, nếu chưa tồn tại thì tạo segment mới), IPC_EXCL(tạo segment mới nếu đã tồn tại thì call fail).

## 2.3. Attach/detach vùng shared memory

Để attach shared memory segment tới một địa chỉ bộ nhớ của chương trình đang gọi chúng ta sử dụng API **shmat()**.

```c
#include <sys/types.h>
#include <sys/shm.h>

void *shmat (int shmid, const void *shmaddr, int shmflg);
```

Trong đó:

- shmid: shared memory ID
- shmaddr: gọi đến địa chỉ sẽ được đính kèm. Nếu để là NULL thì kernel sẽ tự động cấp phát.
- shmflg: SHM_RDONLY, SHM_REMAP.

Nếu thành công hàm **shmat** trả về con trỏ đến shared memory được attach, nếu lỗi thì return -1.

Để detach vùng nhớ của process khỏi shared memory segment chúng ta sử dụng API **shmdt()**.

```c
#include <sys/types.h>
#include <sys/shm.h>

int shmdt (const void *shmaddr);
```

Trong đó: shmaddr là con trỏ tới địa chỉ shared memory detach, là kết quả trả về của **shmat()**.

Lệnh này không xóa identifier và structure của nó khỏi hệ thống. Identifier sẽ tồn tại đến khi một tiến trình trong hệ thống gọi shmctl với IPC_RMID command.

## 2.4. Giải phóng shared memory segment

API shmctl() được sử dụng để kiểm soát các hoạt động của shared memory segment.

Chúng ta thường sử dụng system call này cùng với command IPC_RMID để giải phóng hoàn toàn shared memory segment trên hệ thống.

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int shmctl (int shmid, int cmd, struct shmid_ds *buf);
```

## 2.5. Ví dụ

Ta có ví dụ về giao tiếp giữa hai process như sau.

Chương trình gửi dữ liệu:

```c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/shm.h>

int main()
{
    key_t key = ftok("./shmfile", 65);
    int shmid = shmget(key, 1024, 0666 | IPC_CREAT);
    char *str = (char *)shmat(shmid, NULL, 0);
    printf("Message to shared memory: ");
    fgets(str, 1024, stdin);
    return 0;
}
```

Chương trình đọc dữ liệu:

```c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/shm.h>

int main()
{
    key_t key = ftok("./shmfile", 65);
    int shmid = shmget(key, 1024, 0666 | IPC_CREAT);
    char *shmaddr = (char *)shmat(shmid, NULL, 0);
    printf("Data read from memory: %s\n", shmaddr);
    shmdt(shmaddr);
    shmctl(shmid, IPC_RMID, NULL);
    return 0;
}
```

Kết quả thu được sau khi biên dịch và chạy chương trình:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/27/image_2b259e09c6.png)

# 3. POSIX Shared Memory

POSIX shared memory cho phép chia sẻ vùng được ánh xạ giữa các process không liên quan mà không cần tạo tệp ánh xạ tương ứng. POSIX shared memory được hỗ trợ trên Linux kể từ kernel 2.4.

Linux tạo cho shared memory dưới dạng tệp thông qua hệ thống tệp tmpfs được nằm trong thư mục /dev/shm. Hệ thống tệp này sẽ tồn tại ngay cả khi không có process nào đang mở chúng, chúng chỉ bị mất đi khi hệ thống bị tắt.

Các bước triển khai:

- Tạo shared memory object.
- Set kích thước cho shared memory object.
- Map/unmap shared memory object.
- Giải phóng shared memory object.

## 3.1. Tạo shared memory object

Ở System V shared memory chúng ta có khái niệm shared memory segment. Trong khi đó, ở POSIX chúng ta có khái niệm shared memory object. Cả hai khái niệm trên đều là tương đương và ám chỉ về một vùng nhớ chia sẻ được giữa các process.

Để tạo một shared memory object chúng ta sử dụng API **shm_open()**.

```c
#include <fcntl.h>
#include <sys/mman.h>

int shm_open(const char *name, int oflag, mode_t mode);
```

Trong đó:

- name: Tên của POSIX shared memory object. Trong thực tế, tên này không phải là một đường dẫn trên hệ thống tệp, mà chỉ là một chuỗi ký tự duy nhất để định danh shared memory object.
- oflag: Cờ mở (open flags). Các cờ này có thể là O_RDONLY (chỉ đọc), O_RDWR (đọc và ghi), O_CREAT (tạo nếu không tồn tại), và nhiều cờ khác.Có thể kết hợp chúng bằng cách sử dụng toán tử |
- mode: Quyền truy cập khi tạo mới một POSIX shared memory object (O_CREAT được sử dụng). Quyền này thường được thiết lập bằng các hằng số như S_IRUSR (quyền đọc cho người sở hữu), S_IWUSR (quyền ghi cho người sở hữu) hoặc thiết lập bằng mặt nạ bit ( 0666 ).

Hàm trả về:

- Trong trường hợp thành công, hàm trả về một file descriptor của POSIX shared memory object (kiểu int).
- Trong trường hợp lỗi, hàm trả về -1 và người dùng có thể thiết lập biến errno để chỉ ra lý do lỗi.

## 3.2. Set size

Khi shared memory object được tạo ra. Kích thước của nó được khởi tạo bằng 0.

Do đó ta cần phải set size cho shared memory object thông qua API **ftruncate()**.

```c
#include <unistd.h>

int ftruncate(int fd, off_t length);
```

Trong đó:

- fd: File descriptor của shared memory object cần thay đổi kích thước.
- length: Kích thước mới mà người dùng muốn thiết lập cho shared memory object.

Hàm trả về:

- Trong trường hợp thành công, hàm ftruncate() trả về 0.
- Trong trường hợp lỗi, nó trả về -1 và người dùng có thể thiết lập biến errno để chỉ ra lý do lỗi.

## 3.3. Map/unmap shared memory object

Để map shared memory object vào trong vùng nhớ của process chúng ta cần sử dụng API **mmap()**. Kĩ thuật này còn được gọi là memory mapping.

Chúng ta có 4 kiểu memory mapping như sau:

- Private anonymous mapping.
- Shared anonymous mapping.
- Private file mapping.
- Shared file mapping.

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

Trong đó:

- addr: Địa chỉ bắt đầu của phần bộ nhớ được ánh xạ. Thông thường, bạn sẽ để giá trị này là NULL, để hệ thống tự động chọn địa chỉ thích hợp.
- length: Độ dài của phần bộ nhớ cần ánh xạ, tính bằng byte.
- prot: Quyền truy cập cho phần bộ nhớ được ánh xạ. Các giá trị thường sử dụng là PROT_READ, PROT_WRITE, và/hoặc PROT_EXEC.
- flags: Các cờ kiểm soát hành vi của ánh xạ, như MAP_SHARED để ánh xạ cho shared memory hoặc MAP_PRIVATE để tạo bản sao riêng tư.
- fd: File descriptor của POSIX shared memory.
- offset: Vị trí bắt đầu ánh xạ trong tệp tin, tính bằng byte.

Hàm trả về:

- Trong trường hợp thành công, hàm mmap() trả về địa chỉ bắt đầu của phần bộ nhớ được ánh xạ.
- Trong trường hợp lỗi, nó trả về MAP_FAILED.

Hàm munmap() được sử dụng để giải phóng bộ nhớ đã được ánh xạ bằng hàm mmap() khỏi không gian bộ nhớ của process. Điều này thường được thực hiện sau khi không còn cần truy cập đến dữ liệu tại vùng nhớ được ánh xạ hoặc khi kết thúc việc làm việc với một shared memory.

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
```

Trong đó:

- addr: Địa chỉ bắt đầu của phần bộ nhớ đã ánh xạ cần được giải phóng.
- length: Độ dài của phần bộ nhớ cần giải phóng, tính bằng byte.

Hàm trả về:

- Trong trường hợp thành công, hàm munmap() trả về 0.
- Trong trường hợp lỗi, nó trả về -1 và người dùng có thể thiết lập biến errno để chỉ ra lý do lỗi.

## 3.4. Giải phóng shared memory object

Để giải phóng shared memory object được tạo ra trước đó chúng ta sử dụng API **shm_unlink()**.
Shared memory object sẽ được xóa sau khi process cuối cùng được unmap.

```c
#include <sys/mman.h>
int shm_unlink(const char *name);
```

Hàm shm_unlink() loại bỏ shared memory object được chỉ định theo name.

## 3.5. Ví dụ

Tương tự như system V ta có ví dụ sau về giao tiếp giữa 2 process.

Chương trình gửi dữ liệu:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>
#include <sys/shm.h>

#define SIZE 50
#define FILE_NAME "toanonestar"

int main()
{
    int fd = shm_open(FILE_NAME, O_CREAT | O_RDWR, 0666);
    ftruncate(fd, SIZE);
    char *data = (char *)mmap(0, SIZE, PROT_WRITE | PROT_READ, MAP_SHARED, fd, 0);
    strcpy(data, "Hello world1\n");
    printf("%s: Write data: %s\n", __FILE__, data);
    munmap(data, SIZE);
}
```

Chương trình đọc dữ liệu:

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>
#include <sys/shm.h>

#define SIZE 50
#define FILE_NAME "toanonestar"

int main()
{
    int fd = shm_open(FILE_NAME, O_RDWR, 0);
    if (fd == -1)
    {
        perror("shm_open failed");
        return 1;
    }
    ftruncate(fd, SIZE);
    char *data = (char *)mmap(0, SIZE, PROT_WRITE | PROT_READ, MAP_SHARED, fd, 0);
    if (data == MAP_FAILED)
    {
        perror("mmap failed");
        return 1;
    }
    printf("%s: Read data: %s\n", __FILE__, data);
    munmap(data, SIZE);
    shm_unlink(FILE_NAME);
}
```

Biên dịch và chạy chương trình ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/27/image_e6731dbe5d.png)

# 4. Kết luận

Như vậy trong bài viết này chúng ta đã nắm được shared memory là gì, tại sao shared memory lại cần thiết và là cơ chế IPC nhanh nhất, đã biết cách triển khai system V và POSIX shared memory để giao tiếp giữa hai process.
