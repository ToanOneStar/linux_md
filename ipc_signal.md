Vấn đề xử lí việc giao tiếp giữa các process là vấn đề thường xuyên xảy ra và quan trọng trong hệ thống nhúng . Trong bài viết này ta sẽ tìm hiểu về Signal là một giao thức IPC (Inter-Process Communication).

Signal là một trong những phương thức dùng để giao tiếp liên tiến trình lâu đời nhất của Unix System. Signal là một software interrupt, là cơ chế xử lý các sự kiện bất đồng bộ (async).

Về vòng đời của Signal (Signal Lifecycle)

- **Generation**: Đầu tiên một tín hiệu signal được raised/sent/generated.
- **Delivery**: Một signal được pending cho tới khi nó được phân phối.
- **Processing**: Một khi tín hiệu được phân phối, nó có thể được xử lý bởi nhiều cách.
  - **Ignore the signal**: Không action nào được thực hiện. SIGKILL và SIGSTOP không thể bị ignore.
  - **Catch and handle the signal**: Kernel sẽ tạm dừng thực thi main thread và nhảy tới hàm xử lý signal được user đăng kí trong process. SIGINT và SIGTERM là hai signal thường được dùng. SIGKILL và SIGSTOP không thể catch.
  - **Perform the default action**: Hành động này phụ thuộc từng loại signal.

# 1. Signal Handler

Chúng ta đăng kí việc xử lý một signal thông qua system call **signal()**.

```c
#include<signal.h>
sighandler_t signal(int signo, sighandler_t handler);
```

Với hai đối số là signo(signal number) và handler(signal handler). Khi một process nhận được tín hiệu signal sẽ nhảy vào handler.

Signal là một software interrupt nên nó khá nhạy cảm về mặt thời gian thực thi. Khi signal handler được thực thi nó sẽ chiếm hoàn toàn CPU của process. Vì vậy cần phải thoát ra hàm xử lý signal nhanh nhất có thể.

## 1.1. Một số signal cơ bản

Chúng ta có lệnh kill -l để in ra các signal:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/13/Screenshot_2024_11_13_163734_8a76b19faa.png)
Trong đó ta có thể kể đến các signal cơ bản như:

- **SIGKILL**: Chỉ có thể gửi bằng system call kill(). Process không thể caught hoặc ignored. Mặc định sẽ kết thúc tiến trình được chỉ định.
- **SIGTERM**: chỉ có thể gửi bằng system call kill(). Mặc định sẽ kết thúc tiến trình được chỉ định, tuy nhiên process có thể catch tín hiệu này và dọn dẹp trước khi kết thúc.
- **SIGINT**: Tín hiệu này được gửi tới các process trong nhóm foreground process. Mặc định sẽ kết thúc tiến trình hiện tại.
- **SIGCHLD**: Bất cứ khi nào một tiến trình dừng lại, nó sẽ gửi SIGCHLD tới process cha của nó. Mặc định SIGCHLD bị ignored.
- **SIGSTOP**: Chỉ có thể gửi bằng system call kill(). Process không thể caught hoặc ignored. Mặc định sẽ tạm dừng process được chỉ định.
- **SIGUSR1/SIGUSR2**: Signals có sẵn cho người dùng tự định nghĩa.

## 1.2. Ví dụ

Dưới đây là ví dụ khi ta đăng kí một signal và sửa đổi tác vụ của SIGINT. Mỗi lần bấm ctrl C sẽ in ra màn hình "hello world" và signal number chen ngang vào process main().

```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <signal.h>

void signal_1(int num)
{
    printf("\nhello world %d\n", num);
}

int main()
{
    if (signal(2, signal_1) == SIG_ERR) // 2 == SIGINT
    {
        fprintf(stderr, "failed\n");
        exit(EXIT_FAILURE);
    }
    while (1)
    {
        printf("I'm main()\n");
        sleep(1);
    }
}
```

Kết quả thu được:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/14/Screenshot_2024_11_14_135236_45d788d639.png)
Việc xử lí trên được gọi là Catch and handle the signal. Tuy nhiên có 2 signal không thể catch là SIGKILL và SIGSTOP.

# 2. Sending Signals

Signal có thể gửi được qua hàm system call kill() trong mã nguồn. Ngoài ra có thể gửi thông qua command kill trên terminal.

Có thể tự gửi signal đến bản thân tiến trình đó thông qua việc sử dụng hàm getpid().

Ta có ví dụ sau:

```c
#include <stdio.h>
#include <signal.h>

int main()
{
    int k = 0;
    while (1)
    {
        k++;
        printf("hello world\n");
        if (k == 3)
        {
            kill(getpid(), SIGINT);
        }
    }
}
```

Kết quả thu được:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/14/Screenshot_2024_11_14_141418_e275c6ad88.png)

# 3. Blocking và unblocking signal

Signal làm gián đoạn quá trình thực thi của process. Điều này trong nhiều trường hợp không được mong muốn xảy ra khi process đang thực thi một số đoạn mã quan trọng. Blocking signal sẽ giúp giải quyết vấn đề này.

Mỗi một process có thể chỉ định signal cụ thể nào mà nó muốn block. Nếu signal bị block vẫn xảy ra thì nó sẽ được kernel giữ vào hàng chờ xử lý (pending).

Tín hiệu chỉ được gửi tới process sau khi nó được unblocking. Danh sách các signal bị block được gọi là signal mask.

Chúng ta có một số hàm sau:

- int **sigemptyset**(sigset_t \*set)
- int **sigfillset**(sigset_t \*set)
- int **sigaddset**(sigset_t \*set, int signum)
- int **sigdelset**(sigset_t \*set, int signum)

Để block signal ta sử dụng **setpromask()**

```c
#include <signal.h>
int sigprocmask (int how, const sigset_t *newset, sigset_t *oldset);
```

**how** có thể là:

- SIG_SETMASK: signal mask của process sẽ bị thay đổi thành newset.
- SIG_BLOCK: newset sẽ được thêm vào signal mask (phép OR).
- SIG_UNBLOCK: newset sẽ bị xóa khỏi signal mask.

Nếu oldset khác NULL, sigprocmask sẽ lấy ra được signal mask hiện tại và lưu vào oldset. Nếu newset là NULL, sigprocmask sẽ bỏ qua việc thay đổi giá trị của signal mask, nhưng nó sẽ lấy ra được signal mask hiện tại và lưu vào oldset. Nói cách khác, truyền null vào set như một cách lấy ra signal mask hiện tại.

Ta có ví dụ về việc block signal khiến không thể truy cập vào signal_1:

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
void signal_1(int num)
{
    printf("hello world\n");
}

int main()
{
    sigset_t new_set, old_set;
    if (signal(SIGINT, signal_1) == SIG_ERR)
    {
        fprintf(stderr, "failed\n");
        exit(1);
    }

    sigemptyset(&new_set);
    sigaddset(&new_set, SIGINT);

    if (sigprocmask(SIG_SETMASK, &new_set, NULL) == 0)
    {
        if (sigismember(&new_set, SIGINT) == 1)
        {
            printf("SIGINT exist\n");
        }
        else if (sigismember(&new_set, SIGINT) == 0)
        {
            printf("SIGINT does not exist\n");
        }
    }
    while (1);
    return 0;
}
```

Kết quả thu được:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/14/Screenshot_2024_11_14_152555_ee017ca4a9.png)

# 4. Kết luận

Sau bài viết này chúng ta cần nắm được cách signal handler hoạt động, các signals thường dùng và việc blocking hay unblocking signal.
