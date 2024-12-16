Linux cung cấp nhiều cơ chế giao tiếp giữa các tiến trình khác nhau. Trong bài viết này, chúng ta sẽ cùng nhau tìm hiểu về cơ chế Message queue (hàng đợi tin nhắn) nhé.

# 1. Khái niệm

Message queue hay hàng đợi tin nhắn cũng là một cơ chế IPC có sẵn trong Linux. Một Message Queues là một danh sách liên kết (link-list) các message được duy trì bởi kernel.

Tất cả các process có thể trao đổi dữ liệu thông qua việc truy cập vào cùng một queues.

Mỗi một message sẽ được đính kèm thông thêm thông tin về type (type message). Dựa vào type message mà các process có thể lấy ra tin nhắn phù hợp.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/23/image_019c2f5cfa.png)

# 2. System V Message Queues

Các bước triển khai bao gồm:

1. Tạo key.
2. Tạo message queue hoặc mở một message queue có sẵn.
3. Ghi dữ liệu vào message queue.
4. Đọc dữ liệu từ message queue.
5. Giải phóng message queue.

## 2.1. Tạo key

Key được sử dụng có thể là một số nguyên bất kì hoặc được tạo ra bởi hàm **ftok()**.

```c
#include <sys/ipc.h>
key_t ftok(char *pathname, int proj);
```

Hàm trả về -1 nếu error.

## 2.2. Tạo message queue

Để tạo mới hoặc mở một message queues đã tồn tại chúng ta sử dụng **msgget()**.

```c
#include <sys/types.h>
#include <sys/msg.h>
int msgget(key_t key, int msgflg);
```

Trong đó:

- key: được tạo từ bước 1.
- msgflg: gồm IPC_CREAT, IPC_EXCL

Trả về message id của message queue nếu thực hiện thành công.

## 2.3. Ghi dữ liệu vào message queue

Để ghi dữ liệu (send/append) vào message queue chúng ta sử dụng **msgsnd()**.

```c
#include <sys/types.h>
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

Hàm trả về 0 nếu thành công, -1 nếu error.

Trong đó:

- msqid: message id thu được từ msgget().
- msgp: con trỏ tới message (send).
- msgsz: kích thước message.
- msgflg: IPC_NOWAIT sẽ return ngay lập tức nếu message trong queue đã full, MSG_NOERROR cắt bớt message nếu kích thước trong mess lớn hơn msgsz.

## 2.4. Đọc dữ liệu từ message queue

Để đọc dữ liệu từ message queue chúng ta sử dụng **msgrcv()**.

```c
#include <sys/types.h>
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t maxmsgsz, long msgtyp, int msgflg);
```

Trong đó:

- msqid: message id thu được từ msgget().
- msgp: con trỏ tới buffer (read).
- maxmsgsz: thường là kích thước của buffer.
- msgtyp: bằng 0 thì message đầu tiên trong queue sẽ được nhận, nếu lớn hơn 0 thì message đầu tiên của type msgtyp sẽ được nhận, nếu nhỏ hơn 0 thì message đầu tiên của type thấp nhất(nhỏ hơn hoặc bằng trị tuyệt đối của msgtyp) sẽ được nhận.
- msgflg: cũng có IPC_NOWAIT, MSG_NOERROR.

## 2.5. Xóa message queue

Để kiểm soát các hoạt động trên message queue chúng ta sử dụng **msgctl()**.

```c
#include <sys/types.h>
#include <sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```

Trong đó:

- msqid: message id thu được từ msgget().
- cmd: IPC_RMID, IPC_STAT, IPC_SET.

Để xóa một message queue thông thường cmd chúng ta dùng là IPC_RMID và buf để thành giá trị NULL.

## 2.6. Ví dụ

Ta có ví dụ sau về việc truyền nhận dữ liệu mô hình giữa producer và consumer.
Chương trình trên producer:

```c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <string.h>

#define BUFFER_SIZE 100
struct mesg_buffer
{
    long mesg_type;
    char mesg_text[BUFFER_SIZE];
} message;

int main()
{
    key_t key = ftok("profile", 65);
    int msgid = msgget(key, 0666 | IPC_CREAT);

    printf("Enter type message: ");
    scanf("%ld", &message.mesg_type);
    getchar();
    printf("Enter message: ");
    fgets(message.mesg_text, BUFFER_SIZE, stdin);

    msgsnd(msgid, &message, sizeof(message), 0);
    return 0;
}
```

Chương trình trên consumer:

```c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/msg.h>

struct mesg_buffer
{
    long mesg_type;
    char mesg_text[100];
} message;

int main()
{
    key_t key = ftok("profile", 65);
    int msgid = msgget(key, 0666 | IPC_CREAT);

    msgrcv(msgid, &message, sizeof(message), 1, 0);
    printf("Data Received is : %s \n", message.mesg_text);

    msgctl(msgid, IPC_RMID, NULL);
    return 0;
}
```

Biên dịch và chạy chương trình:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/23/image_e919a2425c.png)
Khi type từ producer được truyền bằng 1 thì phía consumer sẽ nhận được dữ liệu và in ra màn hình. Còn khi type khác 1 thì msgrcv() bên phía consumer sẽ bị block và không nhận được dữ liệu.
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/23/image_5c86f23d8b.png)

# 3. POSIX Message Queues

Các bước triển khai tương tự như system V.

## 3.1. Tạo message queue

Để tạo mới hoặc mở một message queues đã tồn tại chúng ta sử dụng **mq_open()**.

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr);
```

Trong đó:

- name: tên message queue.
- oflag: O_CREAT, O_EXCL, O_RONLY, O_NONBLOCK.
- mode: mask bit RWE.
- attr: Chỉ định các thuộc tính của message queue. Nếu là NULL sẽ sử dụng các thuộc tính mặc định.

## 3.2. Ghi dữ liệu

Để ghi dữ liệu vào message queue chúng ta sử dụng **mq_send()**.

```c
#include <mqueue.h>

int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, usigned int msg_prio);
```

Trong đó:

- mqdes: mq descriptor được trả về ở bước trước.
- msg_ptr: con trỏ tới message.
- msg_len: kích thước message.
- msg_prio: priority của message.

## 3.3. Receving message

Đọc dữ liệu từ message queue chúng ta sử dụng **mq_receive()**.

```c
#include <mqueue.h>
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, usigned int *msg_prio);
```

Trong đó:

- mqdes: mq descriptor được trả về.
- msg_ptr: con trỏ tới message.
- msg_len: kích thước message.
- msg_prio: priority của message.

Hàm mq_receive () loại bỏ message có mức độ ưu tiên cao nhất khỏi queue, được tham chiếu bởi mqdes và trả về thông báo đó trong bộ đệm do msg_ptr trỏ tới.

## 3.4. Đóng message queue

Để đóng message queue khi không còn sử dụng ta sử sử dụng **mq_close()**.

```c
#include <mqueue.h>
int mq_close(mqd_t mqdes);
```

## 3.5. Xóa message queue

Để xóa message queue khi không còn sử dụng ta sử sử dụng **mq\_ unlink()**.

```c
#include <mqueue.h>
int mq_unlink(mqd_t mqdes);
```

## 3.6. Ví dụ

Chương trình bên producer:

```c
#include <stdio.h>
#include <string.h>
#include <mqueue.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/stat.h>

int main()
{
    mqd_t mqid = mq_open("/mqueue", O_RDWR | O_CREAT, 0644, NULL);

    const char *message = "Hello";
    if (mq_send(mqid, message, strlen(message) + 1, 0) == -1)
    {
        perror("mq_send failed");
        mq_close(mqid);
        return 1;
    }

    printf("Message sent: %s\n", message);
    mq_close(mqid);
    return 0;
}
```

Chương trình bên consumer:

```c
#include <stdio.h>
#include <string.h>
#include <mqueue.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/stat.h>

int main()
{
    mqd_t mqid = mq_open("/mqueue", O_RDONLY);

    char messrv[256];
    ssize_t bytes_read = mq_receive(mqid, messrv, sizeof(messrv), NULL);

    messrv[bytes_read] = '\0';
    printf("Data received: %s\n", messrv);

    mq_close(mqid);
    return 0;
}
```

Biên dịch và chạy chương trình ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/23/image_f0a6a8a955.png)

# 4. Kết luận

Như vậy trong bài viết này chúng ta đã nắm được khái niệm về message queue(hàng đợi tin nhắn) và nắm được các bước triển khai với ví dụ về system V và posix.
