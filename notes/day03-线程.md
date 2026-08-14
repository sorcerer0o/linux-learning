具备取消点的函数所有的特征:
    (1)几乎所有的阻塞函数都具备取消点；
    (2)具备IO操作的功能的函数具备取消点；
使用"man 7 pthreads"查询；
手动创建取消点-- pthread_testcancel();
被取消后的值，为宏#define PTHREAD_CANCELED ((void *)-1);
线程取消的五个步骤:
    1.首先在线程中调用pthread_cancel,并传入要取消的目标线程id，系统根据线程id,将目标线程的内部取消标记设置为真；
    2.被取消线程在执行的时候，当执行到取消点函数，则取检查标志位；
    3.取消点函数一旦检测到线程为取消状态，在终止进程之前，将执行清理函数:pthread_cleanup_push，用与释放占用资源，解锁互斥锁等；
    4.完成数据清理后，线程执行退出操作，操作系统或线程库负责销毁thread local数据；
    5.pthread_join 被返回，接受线程取消状态：PTHREAD_CANCELED;
pthread_cleanup_push和pthread_cleanup_pop函数：
    pthread_cleanup_push和pthread_cleanup_pop函数要在同一作用域下成对出现；
    函数运行到pthread_cleanup_pop(),execute参数为"0"不执行，"非0“执行；
    当线程因为在start_routine入口函数中因为return结束进程，清理函数栈不会弹栈；
互斥锁:用于保证任意时刻只有一个线程能够访问共享资源;
http请求报文中包括，请求行，请求头，空行，请求体；
请求行构成:请求方法、请求资源、请求协议
