# Day03 - 线程取消与互斥锁

## 线程取消（pthread_cancel）

### 取消点（Cancellation Point）
- **取消点**：线程被取消时，实际发生终止的位置（需要到达某个函数才会真正被取消）
- 具备取消点的函数特征:
  1. 几乎所有的**阻塞函数都具备取消点**（如 read、write、sleep、wait、accept 等）
  2. 具备 **IO 操作功能**的函数具备取消点
- 使用 `man 7 pthreads` 查询哪些函数具备取消点
- **手动创建取消点**: `pthread_testcancel();`（在代码关键位置主动检查取消标记，自己定义取消时机）

### 线程取消的五个步骤（重点）
1. 在线程中调用 `pthread_cancel(tid)`，传入要取消的目标线程 id，系统根据线程 id 将目标线程的**内部取消标记设置为真**
2. 被取消线程执行时，当执行到**取消点函数**，则检查该标志位
3. 取消点函数一旦检测到线程为取消状态，在终止线程之前，**执行清理函数**：
   - 通过 `pthread_cleanup_push` 注册的清理函数会被执行，用于**释放占用的资源、解锁互斥锁**等
4. 完成数据清理后，线程执行退出操作，操作系统或线程库负责**销毁 thread local 数据**
5. `pthread_join` 返回，接受线程取消状态：**PTHREAD_CANCELED**

### 取消返回值
- 被取消后的返回值，为宏:
```c
#define PTHREAD_CANCELED ((void *)-1)
```

### pthread_cleanup_push / pthread_cleanup_pop
```c
void pthread_cleanup_push(void (*routine)(void *), void *arg);
void pthread_cleanup_pop(int execute);
```
- **要在同一作用域下成对出现**（宏实现，编译错误/未定义行为如果不成对）
- 函数运行到 `pthread_cleanup_pop()` 时：
  - execute 参数为 **0：不执行**清理函数（只弹栈）
  - execute 参数为**非 0：执行**该清理函数（清理后弹栈）
- 注意：当线程因为在 start_routine 入口函数中 `return` 结束进程时，**清理函数栈不会弹栈**（清理函数不执行）

## 互斥锁（pthread_mutex）

- **互斥锁**: 用于保证**任意时刻只有一个线程**能够访问共享资源
```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;   // 静态初始化
// 或 pthread_mutex_init(&mutex, NULL);

// 加锁 / 解锁
pthread_mutex_lock(&mutex);      // 阻塞加锁：获取不到锁会一直等待
pthread_mutex_trylock(&mutex);   // 非阻塞加锁：获取不到立即返回错误码
pthread_mutex_unlock(&mutex);    // 解锁
```
- 使用模式：加锁 → 访问/修改共享资源 → 解锁
- 记住解锁路径：包括异常/提前退出路径都要解锁，否则产生死锁

### 死锁的三种情况（重点）
1. **重复上锁**：上锁之后未解锁的情况下，再次对同一把锁上锁 → 死锁（阻塞在第二次 lock）
2. **持锁线程提前终止**：持有锁的线程在没有释放锁的情况下提前终止（exit），其他线程试图获取该锁 → 死锁
3. **互争锁（环型等待）**：线程 1 拥有锁 1 想获取锁 2，线程 2 拥有锁 2 想获取锁 1，两个线程互相争抢 → 死锁

## HTTP 请求报文

### 请求报文结构
http 请求报文中包括: **请求行、请求头、空行、请求体**

### 请求行构成
- **请求方法**（GET / POST / PUT / DELETE ...）
- **请求资源**（URL，资源的路径）
- **请求协议**（HTTP 版本，如 HTTP/1.1）

形式如: `GET /index.html HTTP/1.1`

### 请求头 / 空行 / 请求体
- 请求头：携带请求的属性信息（User-Agent、Host、Content-Type 等）
- 空行：没有实际意义，用于将请求头与请求体分开
- 请求体：请求正文，传输的数据信息

## 练习

1. 用 man 7 pthreads 查出 3 个具备取消点的函数
2. 线程里调 pthread_testcancel()，验证手动创建的取消点
3. 写一个死锁练习：两个线程互争两把锁，观察程序卡死；再打破环路解决
4. pthread_cleanup_push 注册解锁清理函数，pthread_cancel 取消线程，验证锁被自动释放
5. pthread_cleanup_pop(0) 和 pop(1) 各写一遍，观察清理函数是否执行
6. 向服务器/本地抓包或构造一个 HTTP 请求报文，拆出请求行、请求头、空行、请求体四段

## 自测

1. 什么是取消点？什么样的函数具备取消点？
2. pthread_testcancel 是干什么的？
3. 线程取消的五个步骤是什么？
4. 线程被取消后 pthread_join 得到的返回值是什么？
5. pthread_cleanup_push 和 pop 为什么必须成对出现？
6. pthread_cleanup_pop(0) 和 pop(1) 的区别？return 退出时清理函数栈会怎样？
7. 互斥锁的作用？lock 和 trylock 的区别？
8. 死锁的三种情况是什么？如何避免？
9. HTTP 请求报文由哪四部分组成？
10. 请求行包含哪些字段？举一个例子。
11. 请求报文中的空行有什么作用？
12. 请求头放置什么信息？请求体放置什么信息？

## 复习记录

- [ ] R1 (次日)
- [ ] R3 (3天后)
- [ ] R7 (7天后)
- [ ] R14 (14天后)