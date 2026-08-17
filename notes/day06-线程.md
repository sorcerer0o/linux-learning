# Day06 - 复习回顾与线程局部存储 (TLS)

## 一、问题回顾 (复习)

### 1、如何解决虚假唤醒?
- 虚假唤醒: 线程被唤醒后发现共享条件其实并不满足
- 产生原因: broadcast 唤醒全部但只有部分满足条件; 内核信号中断/调度策略/实现优化; signal 被其他线程抢走
- **解决方案: 将判断条件的 if 改为 while 循环**, 被唤醒后重新检查, 不满足继续等待
```c
while (count == 0) {           // 不能用 if
    pthread_cond_wait(&cond, &mutex);
}
```

### 2、pthread_cond_timedwait 的特点
```c
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
                           const struct timespec *abstime);
```
- 相对于 pthread_cond_wait: 既可以被 signal/broadcast **唤醒**, 也可以因为设置的 **abstime 绝对时间**到达而**超时返回**
- 返回值: 被唤醒返回 0; 超时返回错误码 **ETIMEDOUT**
- 注意: abstime 是**绝对时间** (用 clock_gettime(CLOCK_REALTIME) 加超时值构造)

### 3、常见网络模型
- OSI 七层: 应用层、表示层、会话层、传输层、网络层、数据链路层、物理层
- TCP/IP 四层: 应用层、传输层、网络层、网络接口层
- 五层 (折中): 应用层、传输层、网络层、数据链路层、物理层

### 4、HTTP 请求报文 (四部分)
```
请求行: 请求方法(GET/POST)、资源路径(URL)、协议与版本(HTTP/1.1)
请求头: key-value 键值对 (User-Agent、Host、Content-Type 等)
空行:   分隔请求头与请求正文
请求体: 请求正文 (POST 提交的数据)
```
- GET 主要用来查询数据 (数据附着在地址栏); POST 主要用来提交数据 (数据在请求体, 不上地址栏)

### 5、HTTP 响应报文 (四部分)
```
响应行: 协议与版本、状态码(1xx/2xx/3xx/4xx/5xx)、原因短语
响应头: 附加信息 (Set-Cookie、Content-Type、Content-Length 等)
空行:   分隔响应头与响应体
响应体: 响应正文 (文本或二进制)
```
- 状态码: 1xx 正在处理; 2xx 正常; 3xx 重定向; 4xx 资源不存在(服务器正常); 5xx 服务器异常

### 6、http 与 https
- **http**: 未加密**明文传输**, 内容易被窃听, 不安全; 但传输效率高
- **https**: **加密传输**, 更安全

## 二、线程局部存储变量 (__thread)

### 1、概念
- 线程局部存储变量使用 **`__thread`** 关键字修饰:
```c
__thread int g_tls_var;
```
- **定义位置**: 在**全局变量的位置**定义
- **特性**: 虽然在全局位置定义, 但是在**每个线程中独一份** (各线程持有的地址不同)
  - 每个线程访问 `g_tls_var` 时, 实际访问的是**属于自己的那份副本**
  - 不同线程访问同一个 TLS 变量名, 得到的是**不同的地址、不同的值**, 互不影响

### 2、与其他变量的对比
| 变量类型 | 定义位置 | 线程间特性 |
|---|---|---|
| 全局变量 | 全局 | **共享**: 所有线程同一份、同一地址, 需加锁保护 |
| 静态变量 | 函数内/全局 | **共享**: 同一份 |
| 局部变量 | 函数内 (栈) | 每次调用独立, 但其他线程可通过地址越权访问 (危险) |
| **__thread 变量** | **全局位置** | **每线程独一份 (地址不同)**, 天然线程私有 |

### 3、使用限制 (重点)
- `__thread` **只能修饰全局变量或静态变量**
- **不能**修饰局部变量、函数参数、函数的返回值 (编译报错)
- 只能修饰 POD 类型 (整型、指针、结构体等简单类型), 不能修饰需要构造函数/析构函数的 C++ 类对象

### 4、底层实现
- TLS 变量存储在进程的 **TLS 段** 中 (二进制文件的一个独立节)
- 每个线程的 **线程控制块 (TCB)** 中保存一个 **TLS 指针 (TLS_PTR)**, 指向自己的 TLS 副本区域
- 线程访问 `__thread` 变量时, 通过 TLS 指针 + 变量偏移定位到自己的副本 → 所以各线程地址不同
- 本质: 用**空间换时间** — 每线程一份拷贝, 无需加锁即可并发访问

### 5、示例
```c
#include <stdio.h>
#include <pthread.h>

__thread int g_tls = 0;   // 全局位置定义, 每线程一份

void *thread_func(void *arg) {
    g_tls = (int)(long)arg;              // 只修改自己这份
    printf("tid=%ld addr=%p value=%d\n",
           pthread_self(), (void *)&g_tls, g_tls);
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, thread_func, (void *)1);
    pthread_create(&t2, NULL, thread_func, (void *)2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("main  addr=%p value=%d\n", (void *)&g_tls, g_tls);
    return 0;
}
```
- 运行结果: 三个线程打印的 `&g_tls` **地址互不相同**, 各自的值互不影响

## 二、TLS 函数接口 (pthread 方式)

除了 `__thread` 关键字, POSIX 还提供一套运行时 API, 动态创建线程局部存储:

```c
#include <pthread.h>

// 创建 (分配) 一个 TLS 键, 每个线程可以关联独立的值
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *));

// 设置本线程与 key 关联的值
int pthread_setspecific(pthread_key_t key, const void *value);

// 获取本线程与 key 关联的值
void *pthread_getspecific(pthread_key_t key);

// 删除 key
int pthread_key_delete(pthread_key_t key);
```
- **pthread_key_create**: 一次性创建一个"键" (全局唯一), 之后每个线程用 setspecific 存放自己的值
- **pthread_setspecific / getspecific**: 存取的是**本线程**的值 (内部按线程 id 区分), 天然线程私有
- **destructor**: 线程退出时自动调用, 用于释放线程私有资源
- 对比: `__thread` 是编译期静态分配, 简单高效; pthread_key 系列是运行期动态分配, 更灵活 (可用于库中)

### 使用流程
```c
pthread_key_t g_key;

void free_buf(void *p) { free(p); }   // 线程退出时自动释放

void *thread_func(void *arg) {
    char *buf = malloc(100);
    pthread_setspecific(g_key, buf);          // 存
    char *mine = pthread_getspecific(g_key);  // 取 (自己那份)
    ...
    return NULL;
}

int main(void) {
    pthread_key_create(&g_key, free_buf);
    ...
    pthread_key_delete(g_key);
}
```

## 三、应用场景

- 每个线程需要"独立的全局变量"而不想通过参数层层传递:
  - 线程私有的错误码 (类比 errno 的线程安全版本)
  - 线程私有缓存 (每个线程自己的缓冲, 避免加锁)
  - 线程名、线程私有配置等
- 典型例子: C 库中 `errno` 的现代实现就是线程局部存储 (早期是全局变量, 多线程会互相干扰)

## 练习

1. 写两个线程各自修改 __thread 变量, 打印地址和值, 验证每线程独一份
2. 对比: 同样代码换成普通全局变量, 观察地址相同、值互相影响
3. 尝试用 __thread 修饰局部变量, 观察编译报错, 记住使用限制
4. 用 pthread_key_create/setspecific/getspecific 实现线程私有错误码: 一个线程失败不影响另一个
5. 给 pthread_key_create 传 destructor, 观察线程退出时自动释放 malloc 的内存

## 自测

1. __thread 修饰的变量定义在什么位置? 每个线程访问到的是同一份吗?
2. 线程局部存储与全局变量、局部变量的区别?
3. __thread 能修饰哪些变量? 不能修饰哪些?
4. TLS 变量底层存在哪? 线程如何找到自己的副本?
5. pthread_key_create / setspecific / getspecific / key_delete 分别做什么?
6. destructor 参数什么时候被调用?
7. __thread 和 pthread_key 系列有什么区别? 各适合什么场景?
8. 现代 errno 为什么是线程安全的? 与 TLS 什么关系?

## 复习记录

- [ ] R1 (次日)
- [ ] R3 (3天后)
- [ ] R7 (7天后)
- [ ] R14 (14天后)