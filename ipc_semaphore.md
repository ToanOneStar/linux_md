Trong bài viết này chúng ta sẽ tìm hiểu về IPC-Semaphore và mình sẽ chỉ nói về POSIX Semaphore vì System V Semaphore đã cũ và ít được sử dụng.

# 1. Giới thiệu

Semaphore là cơ chế cho phép đồng bộ truy cập giữa các process và thread.

Mục đích chính được sử dụng để block process và thread truy cập vào 1 vùng nhớ (shared memory) mà đang được sử dụng bởi một process/thread khác.

Có 2 loại semaphore được sử dụng dụng chủ yếu:

- System V semaphore
- POSIX semaphore

## 1.1. System V Semaphores

### 1.1.1. Counting Semaphores

**Mô tả**: Loại semaphore này được sử dụng để theo dõi số lượng tài nguyên có sẵn. Giá trị của semaphore có thể lớn hơn 1 và thường dùng trong các tình huống cần quản lý một nhóm tài nguyên đồng nhất, chẳng hạn như số lượng slot trong một hàng đợi.

**Một số hàm liên quan**:

- semget(): Tạo hoặc mở một tập hợp semaphore.
- semop(): Thực hiện các thao tác (như tăng hoặc giảm giá trị semaphore) trên tập hợp semaphore.
- semctl(): Quản lý và điều khiển các tập hợp semaphore.

### 1.1.2. Binary Semaphores

**Mô tả**: Binary semaphore là một loại semaphore với giá trị chỉ có thể là 0 hoặc 1. Nó thường được sử dụng để đảm bảo rằng chỉ một tiến trình hoặc luồng có thể truy cập tài nguyên tại một thời điểm.

**Một số hàm liên quan**: Tương tự như với counting semaphores, các hàm như semget(), semop(), và semctl() cũng được sử dụng để làm việc với binary semaphores.

## 1.2. Posix Semaphore

Được đưa ra bởi SUSv3 bao gồm có 2 loại như sau:

**Named semaphores**:

- Là semaphore được đặt tên, được tạo open thông quan việc sử dụng hàm sem_open(). Các unrelated-process có thể truy cập tới cùng một semaphore.
- Named semaphores được thiết kế chủ yếu để sử dụng giữa các process khác nhau, nhưng bạn hoàn toàn có thể sử dụng chúng trong cùng một Process giữa các Thread. Tuy nhiên, việc sử dụng named semaphores giữa các Thread trong cùng một Process không phổ biến và thường không cần thiết vì đã có các unnamed semaphores được tối ưu hóa cho mục đích này.
- Sử dụng khi giao tiếp với các process KHÔNG CÓ mối quan hệ cha - con.
- Cũng sử dụng được với các process CÓ mối quan hệ cha - con nhưng KHÔNG tối ưu.
  **Unnamed semaphores**:
- Là semaphore không được đặt tên, do không có name để sử dụng chung nên để sử dụng nó cần phải truy cập vào các vùng nhớ dùng chung (shared memory, mmap(), global variable …)
- Được sử dụng cho việc chia sẻ truy cập giữa các Process hoặc các Thread.
- Đối với các process thì nó là vùng nhớ được shared (sử dụng system shm hoặc POSIX mmap), còn với thread là vùng nhớ mà các thread được chia sẻ trong chương trình ví dụ: global hoặc head.

# 2. Hoạt động của POSIX Semaphore

Hoạt động Semaphore là một số nguyên được duy trì bởi kernel có giá trị bị hạn chế lớn hơn hoặc bằng 0.

Có thể thực hiện nhiều hoạt động khác nhau (tức là các lệnh gọi hệ thống) trên semaphore, bao gồm:

- Tăng giá trị hiện tại của semaphore lên 1 dùng **sem_post()**.
- Trừ giá trị hiện tại của semaphore xuống 1 dùng **sem_wait()**.
- Đọc giá trị hiện tại của semaphore.

![](https://assets.devlinux.vn/uploads/editor-images/2024/11/28/image_878e76c5a7.png)

## 2.1. Posting on semaphore

Hàm sem_post () tăng (tăng 1) giá trị của semaphore.

```c
 #include <semaphore.h>
int sem_post(sem_t *sem);
```

Nếu giá trị của semaphore bằng 0 thì thực hiện tăng giá trị của semaphore value lên 1. Khi đó các process đang chờ sem_wait() sẽ được đánh thức.

Nếu có nhiều process cùng đang chờ thì kernel sử dụng thuật toán round-robin, time-sharing để quyết đinh.

Tức là không phải process là wait trước sẽ được thực hiện trước mà sau khi sem_post() thì process nào là time slot (time slice) tiếp theo của CPU thì sẽ được thực hiện.

## 2.2. Waiting on semaphore

Hàm **sem_wait()** giảm (giảm 1) giá trị của semaphore

```c
#include <semaphore.h>
int sem_wait(sem_t *sem);
```

Nếu semaphore hiện có giá trị lớn hơn 0, **sem_wait ()** trả về ngay lập tức.

Nếu giá trị của semaphore hiện là 0, **sem_wait ()** sẽ block cho đến khi giá trị semaphore tăng trên 0.

Sau khi **sem_wait()** được trả về thì giá trị của semaphore sẽ được giảm đi 1.

Ngoài **sem_wait()** chúng ta còn có thêm các phiên bản của nó với đặc điểm sẽ không block hàm wait khi chưa nhận được sự thay đổi về giá trị (tăng hoặc giảm).

- Hàm **sem_trywait()**: Nếu semaphore có giá trị bằng 0: **sem_trywait()** sẽ không thay đổi giá trị của semaphore và không chặn tiến trình. Thay vào đó, nó trả về lỗi -1 và đặt errno thành EAGAIN, cho biết rằng semaphore không thể giảm được tại thời điểm này.

```c
#include <semaphore.h>
int sem_trywait(sem_t *sem);
```

Hàm **sem_timedwait()**: Chỉ thực hiện chờ trong một thời gian nhất định, nếu sau timeout mà giá trị semaphore vẫn bằng 0 thì lỗi sẽ được trả về ETIMEDOUT.

```c
#include <semaphore.h>
#include <time.h>
int sem_timedwait(sem_t *restrict sem, const struct timespec *restrict abstime);
```

## 2.3. Read semaphore value

Hàm sem_getvalue() trả về giá trị hiện tại của semaphore.

```c
#include <semaphore.h>
int sem_getvalue(sem_t *restrict sem, int *restrict sval);
```

Giá trị semaphore được trả về trong con trỏ sval.

# 3. Named Semaphore

## 3.1. Opening named semaphore

**sem_open()** được sử dụng để tạo một semaphore mới hoặc mở một semaphore đang tồn tại.

```c
#include <fcntl.h>           /* For O_* constants */
#include <sys/stat.h>        /* For mode constants */
#include <semaphore.h>
sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);
```

**name**: tên định danh semaphore. Mỗi tên liên kết với một semaphore object.

**oflag**: cho biết mode hoạt động của sem_open().

- 0: mở một semaphore đang tồn tại.
- O_CREAT: tạo một semaphore mới.
- O_CREAT và O_EXCL: tạo một semaphore mới và trả về lỗi nếu đã tồn tại semaphore liên kết với tên được đưa ra.

**mode**: Quyền truy cập của semaphore, được thiết lập bằng cách kết hợp các quyền thông qua các macro chuẩn như S_IRUSR (quyền đọc của người dùng), S_IWUSR (quyền ghi của người dùng), v.v.. Hoặc thông qua cách phân quyền rõ ràng như sau: 0666, 0777, 0775,...
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/28/image_a0e627edcf.png)

**value**: Giá trị khởi tạo của semaphore nếu nó được tạo mới. Giá trị này xác định số lượng tài nguyên ban đầu mà semaphore quản lý (là 1 số nguyên và lớn hơn hoặc bằng 0).

## 3.2. Closing named semaphore

Khi thực hiện **sem_open()** thì semaphore sẽ được liên kết với process. Hệ thống sẽ ghi lại mối liên kết này.

Tuy nhiên nếu không còn nhu cầu sử dụng semaphore, chúng ta nên đóng lại để kết thúc mối liên kết này.

Hàm **sem_close()** sẽ thực hiện kết thúc mối liên kết trên. Giải phóng bất kỳ tài nguyên nào mà hệ thống đã liên kết với semaphore cho process và giảm số lượng các process tham chiếu đến semaphore.

```c
#include <semaphore.h>
int sem_close(sem_t *sem);
```

## 3.3. Removing named semaphore

Hàm **sem_close()** không xóa semaphore khỏi hệ thống, mà chỉ đóng đối tượng semaphore cho tiến trình đang sử dụng. Sau khi **sem_close()** được gọi, chúng ta không thể sử dụng semaphore đó trong tiến trình này nữa, nhưng semaphore vẫn tồn tại trong hệ thống nếu có các tiến trình khác đang sử dụng nó. Do đó chúng ta nên xóa hoàn toàn semaphore ra khỏi hệ thống trong trường hợp không còn nhu cầu sử dụng nữa.

Hàm **sem_unlink()** sẽ xóa semaphore được xác định bằng name và đánh dấu semaphore sẽ bị hủy sau khi tất cả các process ngừng sử dụng nó. Tức là sau khi gọi hàm **sem_unlink()** thì semaphore chưa thực sự bị hủy. Nó sẽ bị hủy khi và chỉ khi tất cả các tiến trình sử dụng nó gọi **sem_close()**.

```c
#include <semaphore.h>
int sem_unlink(const char *name);
```

# 4. Unnamed Semaphore

## 4.1. Khởi tạo semaphore

Hàm **sem_init()** được sử dụng để khởi tạo semaphore và inform cho kernel semaphore được sử dụng để shared giữa các process hoặc giữa các thread trong một process.

```c
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
```

Đối số **pshared** cho biết liệu semaphore có được chia sẻ giữa các threads hay giữa các process hay không.

- pshared là 0: thì semaphore sẽ được chia sẻ giữa các thread của process. sem được tạo ra trỏ tới một địa chỉ của heap hoặc global.
- pshared khác 0: semaphore được shared giữa các process. sem sẽ là địa chỉ của một vùng nhớ được shared giữa các process (system V hoặc Posix mmap).

**value**: giá trị semaphore được khởi tạo.

## 4.2. Hủy semaphore

Hàm **sem_destroy()** hủy semaphore, nó phải là một unnamed semaphore đã được khởi tạo trước đó bằng **sem_init()**.

Sau khi một unnamed semaphore đã bị hủy bằng **sem_destroy()**, nó có thể được khởi động lại bằng **sem_init()**.

```c
#include <semaphore.h>
int sem_destroy(sem_t *sem);
```

# 5. Ví dụ

Ta sẽ lấy ví dụ về Unnamed Semaphore giữa các thread trong cùng 1 process.

```c
#include <stdio.h>
#include <unistd.h>
#include <semaphore.h>
#include <sys/types.h>
#include <pthread.h>
#include <errno.h>

pthread_t thread1, thread2;
sem_t unnamedSem;

void *threadHandle(void *argv)
{
    pthread_t pthread_id = pthread_self();
    int val;
    if (pthread_equal(pthread_id, thread1))
    {
        sem_wait(&unnamedSem);
        printf("This is Thread 1... \n");
        sem_getvalue(&unnamedSem, &val);
        printf("Gia tri semaphore hien tai: %d\n", val);
        sleep(5);
        printf("Thread 1 da hoan thanh xong cong viec\n");
        sem_post(&unnamedSem);
        sem_getvalue(&unnamedSem, &val);
        printf("Gia tri semaphore hien tai: %d\n", val);
        pthread_exit(&pthread_id);
    }
    else if (pthread_equal(pthread_id, thread2))
    {
        sem_wait(&unnamedSem);
        printf("This is Thread 2... \n");
        sem_getvalue(&unnamedSem, &val);
        printf("Gia tri semaphore hien tai: %d\n", val);
        sleep(5);
        printf("Thread 2 da hoan thanh xong cong viec\n");
        sem_post(&unnamedSem);
        sem_getvalue(&unnamedSem, &val);
        printf("Gia tri semaphore hien tai: %d\n", val);
        pthread_exit(&pthread_id);
    }
}

int main(int argc, char const *argv[])
{
    int fd = sem_init(&unnamedSem, 0, 1);
    int status = pthread_create(&thread1, NULL, threadHandle, NULL);
    if (status != 0)
    {
        perror("Failed to create thread");
        return 1;
    }
    status = pthread_create(&thread2, NULL, threadHandle, NULL);
    if (status != 0)
    {
        perror("Failed to create thread");
        return 1;
    }
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    sem_destroy(&unnamedSem);
    return 0;
}
```

Biên dịch và chạy chương trình ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/28/image_c712837a30.png)
