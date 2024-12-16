Trong bài viết này ta sẽ tìm hiểu về Quản lí Thread trong Linux. Về cơ bản tương tự như việc quản lí Process tuy nhiên có một vài sự khác biệt chúng ta sẽ tìm hiểu sau đây. Đầu tiên chúng ta sẽ nói về việc kết thúc một Thread (Thread Terminal).

- Thread kết thúc một cách bình thường.
- Thread kết thúc khi gọi hàm **pthread_exit()** hay còn được hiểu là kết thúc **chủ động**.

```c
#include <pthread.h>
int pthread_exit(void *retval);
/* Các đối số *retval: chỉ định các giá trị trả về của thread giá trị này có thể thu được bởi một thread khác thông qua pthread_join(). */
```

- Thread bị hủy bỏ khi gọi hàm **pthread_cancel()** hay được hiểu là kết thúc **thụ động**. Và **pthread_cancel()** gửi một yêu cầu kết thúc thread tới một thread cụ thể.

```c
#include <pthread.h>
int pthread_cancel(pthread_t thread);
/* Các đối số thread: threadID của một thread cụ thể. Trả về 0 nếu thành công nhỏ hơn 0 nếu thất bại.*/
```

Ví dụ có hai thread A và B thì thread A có thể chủ động kết thúc thread B

- Khi bất cứ một Thread nào gọi hàm **exit()** hoặc **main thread** kết thúc thì tất cả các thread còn lại kết thúc ngay lập tức. Ta sẽ lấy ví dụ minh họa cụ thể ở dưới.

# 1. Thread Joining

## 1.1. Tìm hiểu về thread joining và pthread_join()

Trong quản lí Process chúng ta sử dụng cặp **wait()** và **exit()** thì trong quản lí Thread chúng ta cũng có **pthread_join()** gần tương tự như **wait()**. Để thu giá trị kết thúc của thread ta sử dụng **pthread_join()** và hoạt động này gọi là Joining.

Prototype của **pthread_join()** như sau

```c
#include <pthread.h>

int pthread_join(pthread_t thread_id, void **res);
```

**pthread_join()** sẽ block cho đến khi thread truyền đến kết thúc (với threadID được truyền vào làm đối số). Nếu thread đó đã kết thúc thì **pthread_join()** return về giá trị res ngay lập tức.

Lưu ý, khi ta kết thúc một thread nếu không có **pthread_join()** thì thread sẽ rơi vào trạng thái zombie tương tự như zombie process. Tuy nhiên số lượng thread zombie có giới hạn đến lúc không thể tạo thêm được nữa. Khi đó ta cần kill trước khi sinh ra một thread mới và cũng có thể khắc phục bằng **pthread_join()** .

## 1.2. Ví dụ

Để minh họa cho những điều trên ta sẽ thông qua ví dụ sau:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

pthread_t thread_id1, thread_id2;

static void *thr_handle1(void *args)
{
    sleep(1);
    printf("thread1 handler\n");
    pthread_exit(NULL); // exit
}

static void *thr_handle2(void *args)
{
    sleep(5);
    while (1)
    {
        printf("thread2 handler\n");
        sleep(1);
    };
}

int main(int args, char const *argv[])
{
    int ret, counter = 0;
    int retval;

    if (ret = pthread_create(&thread_id1, NULL, &thr_handle1, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }

    if (ret = pthread_create(&thread_id2, NULL, &thr_handle2, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }

    while (1);
    return 0;
}
```

Biên dịch và chạy chương trình trên ta được kết quả sau:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Anh1_8d229e9770.png)

Trong **thread_handle1()** sau 1s sẽ in ra màn hình "thread1 handler" và sẽ kết thúc chủ động bằng pthread_exit(). Còn **thread_handle2()** sau 5s sẽ in ra màn hình "thread2 handler" và sẽ lặp vô hạn 1s in 1 lần.

Sửa đổi một chút ở **thread_handle2()** bằng cách sử dụng **exit()**:

```c
static void *thr_handle2(void *args)
{
    sleep(5);
    exit(1);
    while (1)
    {
        printf("thread2 handler\n");
        sleep(1);
    };
}
```

Biên dịch và chạy chương trình trên ta được kết quả sau:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Screenshot_2024_11_06_212018_5864fdd348.png)

Do sử dụng **exit()** nên sau 5s mọi thread sẽ bị kết thúc ngay lập tức.

Ta sẽ minh họa kết thúc thụ động bằng **pthread_cancel()** trong **main** và bỏ **exit()** trong **thread_handle2()**:

```c
int main(int args, char const *argv[])
{
    int ret, counter = 0;
    int retval;

    if (ret = pthread_create(&thread_id1, NULL, &thr_handle1, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }

    if (ret = pthread_create(&thread_id2, NULL, &thr_handle2, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }
    sleep(8);
    pthread_cancel(thread_id2);
    return 0;
}
```

Khi đó thread_handle2() sẽ bị kết thúc thụ động sau 8s và ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Screenshot_2024_11_06_213232_9fe7ff384e.png)

Minh họa cho **pthread_join()** trong **main**:

```c
static void *thr_handle2(void *args)
{
    sleep(5);
    pthread_exit(NULL);
    while (1)
    {
        printf("thread2 handler\n");
        sleep(1);
    };
}

int main(int args, char const *argv[])
{
    int ret, counter = 0;
    int retval;

    if (ret = pthread_create(&thread_id1, NULL, &thr_handle1, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }

    if (ret = pthread_create(&thread_id2, NULL, &thr_handle2, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }
    pthread_join(thread_id2, NULL);
    printf("hello\n");
    return 0;
}
```

**pthread_join()** sẽ block sau 5s đến khi **thread_handle2** kết thúc và thu được kết quả:

![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Screenshot_2024_11_06_214358_f453d0701d.png)

Tiếp đến là minh họa cho thread zombie:

```c
static void *thr_handle3(void *args)
{
    pthread_exit(NULL);
}

int main(int args, char const *argv[])
{
    int ret, counter = 0;
    int retval;

    if (ret = pthread_create(&thread_id1, NULL, &thr_handle1, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }

    if (ret = pthread_create(&thread_id2, NULL, &thr_handle2, NULL))
    {
        printf("pthread_create() error number = %d\n", ret);
        return -1;
    }
    while (1)
    {
        if (ret = pthread_create(&thread_id3, NULL, &thr_handle3, NULL))
        {
            printf("pthread_create() error number = %d\n", ret);
            break;
        }
        counter++;
				pthread_join(thread_id3, NULL);
        if (counter % 1000 == 0)
        {
            printf("Thread created: %d\n", counter);
        }
    }
    return 0;
}
```

Ta thu được kết quả giống như những gì đã diễn giải ở trên:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Screenshot_2024_11_06_215548_2c294d2801.png)

error number = 11 tài nguyên không khả dụng, nghĩa là bị giới hạn không thể tạo thêm thread nữa.

Ta sẽ khắc phục bằng cách sử dụng **pthread_join()**. Khi đó ta thu được kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/6/Screenshot_2024_11_06_220422_df8017e491.png)

# 2. Detaching

Trong thực tế, nhiều trường hợp chúng ta không cần quan tâm về trạng thái kết thúc của thread. Trường hợp này ta có thể đặt thread vào trạng thái detaching thông qua việc gọi **pthread_detach()** (detach có nghĩa là thờ ơ, không quan tâm).

Ở ví dụ phần trước ta cũng có thể kết thúc trạng thái zombie thread bằng **pthread_detach()** mà không cần dùng **pthread_join()**. Kết quả thu được hoàn toàn tương tự:

```c
static void *thr_handle3(void *args)
{
    pthread_detach(pthread_self()); //pthread_seft tức là thread hiện tại
}
```

**pthread_detach()** sẽ tránh được tình trạng block.

# 3. Thread Synchronization

## 3.1. Shared Resource

Giả sử chúng ta có nhiều hơn 1 thread, và giữa các thread có tài nguyên sử dụng chung gọi là shared resource. Tuy nhiên nếu các thread cùng truy cập vào shared resource tại cùng một thời điểm hay một thread đang cố truy cập trong khi thread khác đang thực thi thì sẽ xảy ra hiện tượng bất đồng bộ dữ liệu dẫn đến dữ liệu có thể bị sai.

## 3.2. Atomic/Non-atomic

**Atomic** là cơ chế độc quyền, chỉ có một thread duy nhất được truy cập thuộc tính tại một thời điểm.
Khi nhiều thread tham chiếu đến nó thì thread này thay đổi giá trị xong thì thread khác mới được quyền thay đổi, đảm bảo chỉ một thread được thay đổi giá trị ở một thời điểm. Vì vậy, atomic là an toàn.

Thuộc tính **nonatomic**, nhiều thread truy cập cùng thời điểm có thể thay đổi thuộc tính, không có cơ chế nào để bảo vệ thuộc tính. Vì vậy thuộc tính nonatomic không an toàn.

## 3.3. Critical Section

**Critical Section** hay được hiểu là vùng trọng yếu, dùng để chỉ đoạn code truy cập vào vùng tài nguyên được chia sẻ (shared resource) giữa các threads và việc thực thi của nó nằm trong bối cảnh atomic. Tức là, thời điểm đoạn code được thực thi sẽ không bị gián đoạn bởi bất cứ một thread nào truy cập đồng thời vào shared resource đó.

## 3.4. Mutex Lock

Để xử lí bất đồng bộ ta có kĩ thuật Mutex. Mutex (mutual exclusion) là một kĩ thuật được sử dụng để đảm bảo rằng tại một thời điểm chỉ có 1 thread mới có quyền truy cập vào các tài nguyên dùng chung (shared resources). Hiểu đơn giản chỉ thread nào có chìa khóa (mutex lock) sẽ được truy cập vào tài nguyên chung cho đến khi giải phóng.

Việc triển khai mutex qua 4 bước sau:

- Khởi tạo khóa mutex. Khóa mutex có thể cấp phát tĩnh hoặc động.

```c
#include <pthread.h>
pthread_mutex_t lock1 = PTHREAD_MUTEX_INITIALIZER; //cấp phát tĩnh
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr); // cấp phát động
```

Tuy nhiên khi không sử dụng đối với cấp phát động ta phải hủy mutex bằng **pthread_mutex_destroy()** còn cấp phát tĩnh thì không cần.

- Thực hiện khóa mutex cho các shared resource trước khi vào critical section vì sau khi khởi tạo khóa mutex ở trạng thái unlocked.

```c
#include<pthread.h>
pthread_mutex_lock(pthread_mutex_t *mutex); //*mutex là con trỏ tới khóa mutex
```

Khi khóa mutex ở trạng thái unlocked, pthread_mutex_lock() sẽ return ngay lập tức. Ngược lại, nếu mutex đang locked bởi một thread khác thì pthread_mutex_lock() sẽ bị block cho tới khi mutex được unlocked.

- Thực hiện truy cập vào shared resource.
- Mở khóa/giải phóng khóa mutex.

```c
#include<pthread.h>
pthread_mutex_unlock(pthread_mutex_t *mutex); //*mutex là con trỏ tới khóa mutex
```

Ta có ví dụ minh họa như sau:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

pthread_mutex_t lock1 = PTHREAD_MUTEX_INITIALIZER; // khởi tạo

int counter = 2; // shared variable/shared resources/global variable

static void *handle_th1(void *args)
{
    pthread_mutex_lock(&lock1);
    printf("thread1 handler, counter: %d\n", ++counter);  // critical section
    pthread_mutex_unlock(&lock1);
}

static void *handle_th2(void *args)
{
    sleep(5);
    pthread_mutex_lock(&lock1);
    printf("thread2 handler, counter: %d\n", ++counter);
    pthread_mutex_unlock(&lock1);
}

int main(int argc, char const *argv[])
{
    int ret;
    pthread_t thread_id1, thread_id2;

    if (ret = pthread_create(&thread_id1, NULL, &handle_th1, NULL))
    {
        printf("pthread_create() error number=%d\n", ret);
        return -1;
    }

    if (ret = pthread_create(&thread_id2, NULL, &handle_th2, NULL))
    {
        printf("pthread_create() error number=%d\n", ret);
        return -1;
    }
    sleep(7);
    return 0;
}
```

Biên dịch và chạy ta có kết quả:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/7/Screenshot_2024_11_07_124044_63c0667f13.png)
In ra dòng đầu tiên và sau 5s in ra dòng tiếp theo.

Hiện tượng một thread khóa một mutex và không thể thoát ra được gọi là mutex deadlock.

## 3.4. Conditional Variables

Ta có kĩ thuật **polling** (thăm dò) tức là một thread sẽ kiểm tra định kì điều kiện của shared resource trong khi thread khác đang lock mutex.

Ta có ví dụ minh họa như sau:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

#define THRESHOLD 3

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int counter = 0; // critical section <=> global resource

static void *handle_th1(void *args)
{
    pthread_mutex_lock(&lock);

    while (counter < THRESHOLD)
    {
        counter += 1;
        printf("Counter: %d\n", counter);
        sleep(1);
    }

    pthread_mutex_unlock(&lock);
    pthread_exit(NULL); // exit
}

int main(int argc, char const *argv[])
{
    int ret;
    pthread_t thread_id1;

    if (ret = pthread_create(&thread_id1, NULL, &handle_th1, NULL))
    {
        printf("pthread_create() error number=%d\n", ret);
        return -1;
    }

    while (1)
    {
        if (counter == THRESHOLD)
        {
            printf("Global variable counter = %d\n", counter);
            break;
        }
				usleep(500);
    }
    pthread_join(thread_id1, NULL);
    return 0;
}
```

Kết quả thu được:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/7/Screenshot_2024_11_07_132911_a66e4d1912.png)

Tuy nhiên phương pháp này có nhược điểm tốn tài nguyên CPU nhất định khi phải kiểm ra định kì.

Để khắc phục điều này ta có kĩ thuật **calling** sử dụng Conditional Variables. Một condition variable được sử dụng để thông báo tới một thead khác về sự thay đổi của một shared variable và cho phép một thread khác block cho tới khi nhận được thông báo.

Condition variable là một biến kiểu **pthread_cond_t**. Trước khi sử dụng thì ta luôn phải khởi tạo nó. Tương tự như mutex ta cũng có hai kiểu cấp phát tĩnh và động.

```c
#include <pthread.h>
pthread_cond_t cond = PTHREAD_COND_INITIALIZER; // cấp phát tĩnh
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr); //cấp phát động
```

Khi không sử dụng ta phải hủy condition variable bằng pthread_cond_destroy (). Khởi tạo tĩnh thì không cần phải gọi hàm này.

Hai hoạt động chính của condition variable là **signal** và **wait**.

```c
#include <pthread.h>
pthread_cond_signal(pthread_cond_t *cond);
pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```

Ta có ví dụ:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

#define THRESHOLD 3

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int counter = 0; // critical section <=> global resource

static void *handle_th1(void *args)
{
    pthread_mutex_lock(&lock);
    while (counter < THRESHOLD)
    {
        counter += 1;
        printf("Counter = %d\n", counter);
        sleep(1);
    }

    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&lock);

    pthread_exit(NULL); // exit or return;
}

int main(int argc, char const *argv[])
{
    int ret;
    pthread_t thread_id1;

    if (ret = pthread_create(&thread_id1, NULL, &handle_th1, NULL))
    {
        printf("pthread_create() error number=%d\n", ret);
        return -1;
    }

    pthread_mutex_lock(&lock);
    while (1)
    {
        pthread_cond_wait(&cond, &lock);
        if (counter == THRESHOLD)
        {
            printf("Global variable counter = %d\n", counter);
            break;
        }
    }
    pthread_mutex_unlock(&lock);
    return 0;
}
```

Kết quả thu được tương tự như polling nhưng đã khắc phục được nhược điểm:
![](https://assets.devlinux.vn/uploads/editor-images/2024/11/7/Screenshot_2024_11_07_140558_e6ce859bd3.png)

# 4. Kết luận

Kết thúc bài viết này chúng ta cần nắm được cách quản lí thread, trạng thái joining và detaching, nắm được việc quản lí tài sử dụng chung shared resource để tránh lỗi bất đồng bộ, kĩ thuật mutex và conditional variable.

#
