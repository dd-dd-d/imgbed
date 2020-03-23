

## 线程间的关系——同步和互斥

### 线程的互斥

定义：互斥

在Posix Thread中定义了一套专门用于线程互斥的mutex函数。mutex是一种简单的加锁的方法来控制对共享资源的存取，这个互斥锁只有两种状态（上锁和解锁），可以把互斥锁看作某种意义上的全局变量。为什么需要加锁，就是因为多个线程共用进程的资源，要访问的是公共区间时（全局变量），当一个线程访问的时候，需要加上锁以防止另外的线程对它进行访问，实现资源的独占。在一个时刻只能有一个线程掌握某个互斥锁，拥有上锁状态的线程能够对共享资源进行操作。若其他线程希望上锁一个已经上锁了的互斥锁，则该线程就会挂起，直到上锁的线程释放掉互斥锁为止。

---



#### 创建和销毁锁

有两种方法创建互斥锁，**静态方式和动态方式**。

* 静态方式(不常用)：

POSIX定义了一个宏PTHREAD_MUTEX_INITIALIZER来静态初始化互斥锁，方法如下：

``` c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

在Linux Threads实现中，pthread_mutex_t是一个结构，而PTHREAD_MUTEX_INITIALIZER则是一个宏常量。

* 动态方式：

##### 		创建锁：

pthread_mutex_init()函数来初始化互斥锁，API定义如下：

``` c
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
```

建立锁。放到mutexeattr目录下

其中mutexattr用于指定互斥锁属性（见下），如果为NULL则使用缺省属性。通常为NULL

##### 		销毁锁：

​		pthread_mutex_destroy()用于注销一个互斥锁，API定义如下：

``` c
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

销毁一个互斥锁即意味着释放它所占用的资源，且要求锁当前处于开放状态。

由于在Linux中，互斥锁并不占用任何资源，因此Linux Threads中的pthread_mutex_destroy()除了检查锁状态以外（锁定状态则返回EBUSY）没有其他动作。

___



futex（ - fast user-space locking）:一个系统调用，代表用户态快速锁。

``` c
#include <linux/futex.h>
#include <sys/time.h>
int futex(int *uaddr, int futex_op, int val,
const struct timespec *timeout,   /* or: uint32_t val2 */
int *uaddr2, int val3);
```

gettimeofday：距离现在多长时间

``` c
#include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tz);
int settimeofday(const struct timeval *tv, const struct timezone *tz);
```

$man pthread_mutexattr_getpshared

``` c
#include <pthread.h>
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *attr, int *pshared);
 int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared);
```



不要用锁赋值，实现20000000相加

结构体包含数组，在结构体中添加一个锁，使两个进程互斥

$vim pthread_add2000.c

``` c
#include <func.h>
#define N 20000000
typedef struct{//结构体中定义的是结构体变量和锁
    int val;
    pthread_mutex_t mutex;
}Data_t;
void* threadFunc(void* p)
{
    int i;
    Data_t* threadInfo=(Data_t*)p;
    for(i=0;i<N;i++)
    {
        pthread_mutex_lock(&threadInfo->mutex);
//不等于400000000，概率死锁
        threadInfo->val+=1;
        pthread_mutex_unlock(&threadInfo->mutex);
     }
}
int main(int argc,char* argv[])
{
    pthread_t pthid;
    int ret;
    Data_t threadInfo;
    threadInfo.val=0;
    ret=pthread_mutex_init(&threadInfo.mutex,NULL);//初始化锁
    THREAD_ERROR_CHECK(ret,"pthread_mutex_init");
    struct timeval start,end;//定义结构体类型
    gettimeofday(&start,NULL);//时间函数
    ret=pthread_create(&pthid,NULL,threadFunc,&threadInfo);
    THREAD_ERROR_CHECK(ret,"pthread_create");
    int i;
    for(i=0;i<N;i++)
    {
        pthread_mutex_lock(&threadInfo.mutex);
        threadInfo.val+=1;
        pthread_mutex_unlock(&threadInfo.mutex);
    }
    pthread_join(pthid,NULL);//等待子线程,会死锁
    gettimeofday(&end,NULL);//时间函数
    printf("I am main thread,val=%d,use time=%ld\n",threadInfo.val,(end.tv_sec-start.tv_sec)*1000000+end.tv_usec-start.tv_usec);//end-start
    /*获取线程的运行时间*/
    return 0;
}
```

$make

$./pthread_add2000

I am main thread,val=40000000,use time=3687351

I am main thread,val=40000000,use time=3664528

I am main thread,val=40000000,use time=4162840

信号量sem_add1000.c 大约需要40s 

$grep pthread_mutexattr /usr/include/pthread.h 

``` c
			       const pthread_mutexattr_t *__mutexattr)
extern int pthread_mutexattr_init (pthread_mutexattr_t *__attr)
extern int pthread_mutexattr_destroy (pthread_mutexattr_t *__attr)
extern int pthread_mutexattr_getpshared (const pthread_mutexattr_t *
extern int pthread_mutexattr_setpshared (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_gettype (const pthread_mutexattr_t *__restrict
extern int pthread_mutexattr_settype (pthread_mutexattr_t *__attr, int __kind)
extern int pthread_mutexattr_getprotocol (const pthread_mutexattr_t *
extern int pthread_mutexattr_setprotocol (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getprioceiling (const pthread_mutexattr_t *
extern int pthread_mutexattr_setprioceiling (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getrobust (const pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getrobust_np (const pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_setrobust (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_setrobust_np (pthread_mutexattr_t *__attr,
```

**与sem_add1000.c做对比，线程的更快**

---



#### 锁操作

锁操作主要包括，**加锁，解锁，测试加锁**

##### 		加锁

``` c 
 int pthread_mutex_lock(pthread_mutex_t *mutex)
```

* pthread_mutex_lock：加锁，不论哪种类型的锁，都不可能被两个不同的线程同时得到，而必须等待解锁。对于普通锁类型，解锁者可以是同进程内任何线程；而检错锁则必须由加锁者解锁才有效，否则返回EPERM；对于嵌套锁，文档和实现要求必须由加锁者解锁，但实验结果表明并没有这种限制，这个不同目前还没有得到解释。在同一进程中的线程，如果加锁后没有解锁，则任何其他线程都无法再获得锁。

加锁成功返回0，不成功 会卡住。

##### 		解锁

``` c
int pthread_mutex_unlock(pthread_mutex_t *mutex)
```

* pthread_mutex_unlock：根据不同的锁类型，实现不同的行为：

对于**快速锁**，pthread_mutex_unlock解除锁定；

对于**递规锁**，pthread_mutex_unlock使锁上的引用计数减1；

对于**检错锁**，如果锁是当前线程锁定的，则解除锁定，否则什么也不做。

##### 		测试加锁

``` c 
int pthread_mutex_trylock(pthread_mutex_t *mutex)
```

* pthread_mutex_trylock：语义与pthread_mutex_lock()类似，不同的是在锁已经被占据时返回EBUSY而不是挂起等待。

线程中获取自己线程的id:

$man pthread_self

``` c
#include <pthread.h>
pthread_t pthread_self(void);
Compile and link with -pthread.
```



向func.h中添加宏定义**CHILD_THREAD_ERROR_CHECK**

加一个宏判断trylock,因为trylock和其他的函数返回值不相同

$sudo vim /usr/include/func.h

``` c
#define CHILD_THREAD_ERROR_CHECK(ret,funcName){if(ret!=0){printf("%s:%s\n",funcName,strerror(ret));return (void*)-1;}}
```

# 

$vim pthread_mutex_trylock.c

再次加锁死锁

``` c
#include <func.h>
/*定义一个结构体，存放锁*/
typedef struct{
    pthread_mutex_t mutex;//锁
}Data_t;
void* threadFunc(void* p)//子线程
{
    Data_t* pArg=(Data_t*)p;
    int ret=pthread_mutex_trylock(&pArg->mutex);//测试加锁
    CHILD_THREAD_ERROR_CHECK(ret,"pthread_mutex_trylock");
/*THREAD_ERROR_CHECK(ret,"pthread_mutex_trylock");//会报错，)发生警告
原因：返回值是（void*）类型，THREAD_ERROR_CHECK在头文件中定义的是int类型*/
    printf("child lock success\n");/*主线程先加锁，子线程再加锁，主线程在等子线程发送结束信号，会死锁，这句不打印在屏幕上*/
    pthread_exit(NULL);//返回值，相当于return NULL;
    /*不写返回值，默认返回类型为(void*)*/
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;//线程锁
    pthread_t pthid;//线程id
    pthread_mutex_init(&threadInfo.mutex,NULL);//在子线程创建之前进行初始化
    pthread_create(&pthid,NULL,threadFunc,&threadInfo);//子线程创建
    pthread_mutex_lock(&threadInfo.mutex);//主线程加锁（这句注释掉子进程会成功打印）
    printf("main thread lock success\n");//加锁成功
    int ret=pthread_join(pthid,NULL);//等待子线程退出
    THREAD_ERROR_CHECK(ret,"pthread_join");
    return 0;
}
```

$make

$./pthread_mutex_trylock

main thread lock success
pthread_mutex_trylock:Device or resource busy

主线程不加锁时编译：

main thread lock success
child lock success

---



* 如果主线程加锁，子线程解锁，是否可以解锁？

pthread_mutex_unlock_other.c:

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
}Data_t;
void* threadFunc(void* p)
{
    Data_t* pArg=(Data_t*)p;
    /*子线程解锁*/
    pthread_mutex_unlock(&pArg->mutex);//子进程解锁
    pthread_mutex_lock(&pArg->mutex);//加锁，如果加锁成功，则上一句解锁成功
    printf("child lock success\n");
    pthread_exit(NULL);
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_t pthid;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_create(&pthid,NULL,threadFunc,&threadInfo);
    pthread_mutex_lock(&threadInfo.mutex);
    printf("main thread lock success\n");
    int ret=pthread_join(pthid,NULL);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    return 0;
}
```

$make

$./pthread_mutex_unlock_other

main thread lock success
child lock success

子线程可以解锁主线程

---



* 线程取消(cancel)以后，如何清理锁资源？
* 创建两个子线程，每一个都对子线程进行加锁

当写加解锁和cancel组合使用的时候，一线程在加锁中间被cancel掉，会造成已经加锁的线程永远处于加锁状态，不会被cancel掉。所以离开的时候要解锁，在加锁睡眠的时候cancel不成功，必须使用**cleanup**来清理加锁后被cancel,但未解锁的情况。

* $vim  pthread_mutex_cancel.c

pthread_mutex_cancel.c:

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
}Data_t;
void cleanup(void *p)
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock success\n");
}
void* threadFunc(void* p)
{
    Data_t* pArg=(Data_t*)p;
    pthread_mutex_lock(&pArg->mutex);
    /*当一个线程在加锁状态被cancel掉时，必须用cleanup函数清理掉锁，才不会被卡住。*/
    pthread_cleanup_push(cleanup,&pArg->mutex);//清理函数，传锁的地址，锁是一个空间，必须要操作同一片内存空间才能加锁、解锁、清理资源。
    printf("child lock success\n");
    sleep(2);//这时，线程1，2被cancel掉
/*如果会被cancel，这里不能解锁，否则加锁一次，解锁两次*/
    pthread_cleanup_pop(1);//实际跟unlock配对的是pop
    printf("you can't see me\n");//如果被cancel成功，不能打印这一句
    pthread_exit(NULL);
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_t pthid,pthid1;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_create(&pthid,NULL,threadFunc,&threadInfo);//子线程1
    pthread_create(&pthid1,NULL,threadFunc,&threadInfo);//子线程2
    int ret; 
    sleep(1);
    ret=pthread_cancel(pthid);//进程1被cancel
    THREAD_ERROR_CHECK(ret,"pthread_cancel");
    ret=pthread_cancel(pthid1);//进程2被cancel
    THREAD_ERROR_CHECK(ret,"pthread_cancel");
    long threadRet;//定义长整型，用来存放join返回值（指针类型）
    ret=pthread_join(pthid1,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    printf("pthid1 ret=%ld\n",threadRet);//打印返回值看进程2是否cancel成功
    ret=pthread_join(pthid,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");//打印返回值看进程1是否cancel成功
    printf("pthid ret=%ld\n",threadRet);
    return 0;
}
```

$make

$./pthread_mutex_cancel

child lock success
unlock success
pthid1 ret=-1
child lock success
unlock success
pthid ret=-1

线程结束后，不会在$ps -elLf|grep pthread中看到僵尸

线程只有运行和休眠两种状态

线程中没有僵尸一说！！！

如果先join pthid 则会卡住，因为pthid还没被cancel。

原因：后创建的线程先被cancel，单个线程只拥有专属栈空间，是先入后出的。

printf和sleep都是cancel点，lock不是cancel点。

**锁资源也是一种资源，也需要清理。**

---

通过设置线程锁属性，用mutex实现两个进程各加2千万，最终实现4千万。



pthread_add2000.c

``` c
#include <func.h>
#define N 20000000
typedef struct{
    int val;
    pthread_mutex_t mutex;
}Data_t;
void* threadFunc(void* p)
{
    int i;
    /*定义锁时，不能写成
Data_t threadInfo=*(Data_t*)p;//错误
Data_t *threadInfo=(Data_t*)p;//正确
 pthread_mutex_lock(&pArg->mutex);//*/
    Data_t* threadInfo=(Data_t*)p;
    for(i=0;i<N;i++)
    {
        pthread_mutex_lock(&threadInfo->mutex);//上锁
        threadInfo->val+=1;
        pthread_mutex_unlock(&threadInfo->mutex);//解锁
    }
}
int main(int argc,char* argv[])
{
    pthread_t pthid;
    int ret;
    Data_t threadInfo;
    threadInfo.val=0;
    ret=pthread_mutex_init(&threadInfo.mutex,NULL);//创建锁
    THREAD_ERROR_CHECK(ret,"pthread_mutex_init");
    struct timeval start,end;
    gettimeofday(&start,NULL);//时间函数记录开始时间
    ret=pthread_create(&pthid,NULL,threadFunc,&threadInfo);//创建子进程
    THREAD_ERROR_CHECK(ret,"pthread_create");
    int i;
    for(i=0;i<N;i++)
    {
        pthread_mutex_lock(&threadInfo.mutex);
        threadInfo.val+=1;
        pthread_mutex_unlock(&threadInfo.mutex);
    }
    pthread_join(pthid,NULL);//等待子线程
    gettimeofday(&end,NULL);//时间函数记录结束时间
    printf("I am main thread,val=%d,usetime=%ld\n",threadInfo.val,(end.tv_sec-start.tv_sec)*1000000+end.tv_usec-start.tv_usec);
    return 0;
}
```

$make

$./pthread_add2000 
I am main thread,val=40000000,use time=3882928
$./pthread_add2000 
I am main thread,val=40000000,use time=3822733
$./pthread_add2000 
I am main thread,val=40000000,use time=3233650

把已经锁住的锁传递给子线程，子线程不能再次加锁，导致**死锁**（不是每次都死锁，死锁是概率发生的），主线程这时处于join。

主线程子线程的锁设置为不相同的两把锁。

---

#### 互斥锁属性设置

查看所有互斥锁锁属性：

$grep pthread_mutexattr /usr/include/pthread.h 

``` c
			       const pthread_mutexattr_t *__mutexattr)
extern int pthread_mutexattr_init (pthread_mutexattr_t *__attr)
extern int pthread_mutexattr_destroy (pthread_mutexattr_t *__attr)
extern int pthread_mutexattr_getpshared (const pthread_mutexattr_t *
extern int pthread_mutexattr_setpshared (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_gettype (const pthread_mutexattr_t *__restrict
extern int pthread_mutexattr_settype (pthread_mutexattr_t *__attr, int __kind)
extern int pthread_mutexattr_getprotocol (const pthread_mutexattr_t *
extern int pthread_mutexattr_setprotocol (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getprioceiling (const pthread_mutexattr_t *
extern int pthread_mutexattr_setprioceiling (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getrobust (const pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_getrobust_np (const pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_setrobust (pthread_mutexattr_t *__attr,
extern int pthread_mutexattr_setrobust_np (pthread_mutexattr_t *__attr,
```

**get** 获得锁的属性，**set** 设置锁的属性。

$man的时候，最好是man get_balabala,这样同组的set_balabala也会同时显示。

$man pthread_mutexattr_getpshared

``` c 
#include <pthread.h>
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *attr, int *pshared);
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared); 
```

$man pthread_mutexattr_gettype

``` c
#include <pthread.h>
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr, int *restrict type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);

PTHREAD_MUTEX_NORMAL PTHREAD_MUTEX_ERRORCHECK PTHREAD_MUTEX_RECURSIVE PTHREAD_MUTEX_DEFAULT
```

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               

互斥锁的属性在创建锁的时候指定，不同的锁类型在试图对一个已经被锁定的互斥锁加锁时表现不同也就是是否阻塞等待。有三个值可供选择：

* **PTHREAD_MUTEX_TIMED_NP**，这是缺省值（直接写NULL就是表示这个缺省值），也就是普通锁(或快速锁)。

  当一个线程加锁以后，其余请求锁的线程将形成一个阻塞等待队列，并在解锁后按优先级获得锁。这种锁策略保证了资源分配的公平性

* **PTHREAD_MUTEX_RECURSIVE**，嵌套锁，允许同一个线程对同一个锁成功获得多次，并通过多次unlock解锁。

  如果是不同线程请求，则在加锁线程解锁时重新竞争。

* **PTHREAD_MUTEX_ERRORCHECK**，检错锁，如果同一个线程请求同一个锁，则返回EDEADLK，否则与PTHREAD_MUTEX_TIMED类型动作相同。这样就保证当不允许多次加锁时不会出现最简单情况下的死锁。

  **检测同一个进程被加锁两次的死锁。**

  如果锁的类型是快速锁，一个线程加锁之后，又加锁，则此时就是死锁。

$vim pthread_mutex_locktwice.c

``` c 
#include <func.h>

int main(int argc,char* argv[])
{
    pthread_mutex_t mutex;
    pthread_mutexattr_t mattr;//定义一个锁属性
    int ret;
    ret=pthread_mutexattr_init(&mattr);
    THREAD_ERROR_CHECK(ret,"pthread_mutexattr_init");
    pthread_mutexattr_settype(&mattr,PTHREAD_MUTEX_RECURSIVE);//允许同一个线程对同一把锁加锁多次
    pthread_mutex_init(&mutex,&mattr);//初始化锁用pthread_mutex_init
    //加锁两次
    pthread_mutex_lock(&mutex);
    ret=pthread_mutex_lock(&mutex);
    THREAD_ERROR_CHECK(ret,"pthread_mutex_lock");
    printf("you can see me\n");
    return 0;
}
```

$./pthread_mutex_locktwice.c

you can see me

 检测锁（）：

$vim pthread_mutex_errorcheck.c

``` c
#include <func.h>

int main(int argc,char* argv[])
{
    pthread_mutex_t mutex;
    pthread_mutexattr_t mattr;
    int ret;
    ret=pthread_mutexattr_init(&mattr);
    THREAD_ERROR_CHECK(ret,"pthread_mutexattr_init");
    pthread_mutexattr_settype(&mattr,PTHREAD_MUTEX_ERRORCHECK);
    /*检测同一个进程被加锁两次的死锁*/
    pthread_mutex_init(&mutex,&mattr);
    pthread_mutex_lock(&mutex);
    ret=pthread_mutex_lock(&mutex);//看第二次
    THREAD_ERROR_CHECK(ret,"pthread_mutex_lock");
    printf("you can see me\n");
    return 0;
}
```

---

$./pthread_mutex_errorcheck 

pthread_mutex_lock:Resource deadlock avoided

---

 两个进程之间通过mutex，加到40000000（信号量效率低的改进）

比线程慢，比只用信号量快

$vim fork_mutex.c

fork_mutex.c:

``` c
#include <func.h>
#define N 20000000
typedef struct{
    int val;
    pthread_mutex_t mutex;
}Data_t,*pData_t;
int main(int argc,char* argv[])
{
    int semArrId=semget(1000,1,IPC_CREAT|0600);
    ERROR_CHECK(semArrId,-1,"semget");                                                                                    
    int shmid;
    shmid=shmget(1000,1<<20,IPC_CREAT|0600);
    ERROR_CHECK(shmid,-1,"shmget");
    pData_t p;
    p=(pData_t)shmat(shmid,NULL,0);//锁使用的区域是在共享内存内的
    ERROR_CHECK(p,(pData_t)-1,"shmat");
    p->val=0;
    pthread_mutexattr_t mattr;
    pthread_mutexattr_init(&mattr);
    pthread_mutexattr_setpshared(&mattr,PTHREAD_PROCESS_SHARED);//设置为进程共享
    int ret=pthread_mutex_init(&p->mutex,&mattr);//改为NULL，死锁
    THREAD_ERROR_CHECK(ret,"pthread_mutex_init");
    struct timeval start,end;
    gettimeofday(&start,NULL);
    if(!fork())
    {
        int i;
        for(i=0;i<N;i++)
        {
            //加锁
            pthread_mutex_lock(&p->mutex);
            p->val+=1;
            pthread_mutex_unlock(&p->mutex);
            //解锁
        }
    }else{
        int i;
        for(i=0;i<N;i++)
        {
            pthread_mutex_lock(&p->mutex);
            p->val+=1;
            pthread_mutex_unlock(&p->mutex);
        }
        wait(NULL);
        gettimeofday(&end,NULL);
        printf("result=%d,use time=%ld\n",p->val,(end.tv_sec-start.tv_sec)*1000000+end.tv_usec-start.tv_usec);
    }
    return 0;
}
```

$./fork_mutex 
result=40000000,use time=3744092
$./fork_mutex 
result=40000000,use time=3787529

所需时间和硬件有关！

---

优先级翻转：

低优先级抢锁，让高优先级线程得不到运行。

0 S fd        31276  30631  31276  6    1  80   0 -  1895 wait   03:02 pts/2    00:00:01 ./fork_mutex
1 S fd        31277  31276  31277  0    1  80   0 -  1895 futex_ 03:02 pts/2    00:00:00 ./fork_mutex

getprocontrol

加锁注意事项：

如果线程在加锁后解锁前被取消，锁将永远保持锁定状态，因此如果在关键区段内有取消点存在，则必须在退出回调函数pthread_cleanup_push/pthread_cleanup_pop中解锁。同时不应该在信号处理函数中使用互斥锁，否则容易造成死锁。

死锁是指多个进程因竞争资源而造成的一种僵局（互相等待），若无外力作用，这些进程都将无法向前推进。

##### 死锁产生的原因：

1. 系统资源的竞争 

   ​	系统资源的竞争导致系统资源不足，以及资源分配不当，导致死锁。 

2.  进程运行推进顺序不合适 

   ​	进程在运行过程中，请求和释放资源的顺序不当，会导致死锁。

##### 死锁的四个必要条件： 

- 互斥条件：

一个资源每次只能被一个进程使用，即在一段时间内某 资源仅为一个进程所占有。

此时若有其他进程请求该资源，则请求进程只能等待。 

- 请求与保持条件：

进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。 

- 不可剥夺条件：

进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走，即只能 由获得该资源的进程自己来释放（只能是主动释放)。 

- 循环等待条件：

若干进程间形成首尾相接循环等待资源的关系。

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

##### 死锁的预防

**破坏死锁产生的4个必要条件**，资源互斥是资源使用的固有特性是无法改变的。

-  破坏“不可剥夺”条件：

一个进程不能获得所需要的全部资源时便处于等待状态，等待期间他占有的资源将被隐式的释放重新加入到 系统的资源列表中，可以被其他的进程使用，而等待的进程只有重新获得自己原有的资源以及新申请的资源才可以重新启动，执行。 

- 破坏”请求与保持条件“：

第一种方法静态分配即每个进程在开始执行时就申请他所需要的全部资源。第二种是动态分配即每个进程在申请所需要的资源时他本身不占用系统资源。 

- 破坏“循环等待”条件：

采用资源有序分配其基本思想是将系统中的所有资源顺序编号，将紧缺的，稀少的采用较大的编号，在申请资源时必须按照编号的顺序进行，一个进程只有获得较小编号的进程才能申请较大编号的进程。

---

#### 互斥锁在进程间使用

newtest

#### 互斥锁实例火车站售票

（如若不加锁，会出现卖出负数票的情况）

1.加解锁效率

2.公平性

vim sale_tickets.c

``` c
#include <func.h>
typedef struct{
    int tickets;//票数
    pthread_mutex_t mutex;//锁
}Data_t;
void cleanup(void *p)
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock success\n");
}

void* saleWindows1(void* p)
{
    Data_t* pArg=(Data_t*)p;
    int i=0;//变量i用于统计本窗口卖票数
    while(1)//循环卖票，直到票卖光
    {//有票，先加锁，再卖票
        pthread_mutex_lock(&pArg->mutex);//加锁
        if(pArg->tickets>0)
        {
            //printf("I am saleWindows1 start sale,%d\n",pArg->tickets);//卖票前打印剩余票数
            pArg->tickets--;//票-1
            i++;
            //printf("I am saleWindows1 sale finish,%d\n",pArg->tickets);//卖票后打印剩余票数
            pthread_mutex_unlock(&pArg->mutex);//解锁
        }else{//没票
            pthread_mutex_unlock(&pArg->mutex);//解锁
            printf("I am saleWindows1,%d\n",i);
            break;
        }
    }
    return NULL;
}
void* saleWindows2(void* p)
{
    Data_t* pArg=(Data_t*)p;
    int i=0;//变量i用于统计本窗口卖票数
    while(1)//循环卖票，直到票卖光
    {//有票，先加锁，再卖票
        pthread_mutex_lock(&pArg->mutex);//加锁
        if(pArg->tickets>0)//有票
        {
            //printf("I am saleWindows2 start sale,%d\n",pArg->tickets);//卖票前打印当前窗口剩余票数
            pArg->tickets--;//票-1
            i++;
            //printf("I am saleWindows2 sale finish,%d\n",pArg->tickets);/卖票后打印当前窗口剩余票数
            pthread_mutex_unlock(&pArg->mutex);//解锁
        }else{
            pthread_mutex_unlock(&pArg->mutex);//解锁
            printf("I am saleWindows2,%d\n",i);
            break;
        }
    }
    return NULL;
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    threadInfo.tickets=20000000;//初始化tickets的数量
    pthread_t pthid,pthid1;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_create(&pthid,NULL,saleWindows1,&threadInfo);
    pthread_create(&pthid1,NULL,saleWindows2,&threadInfo);
    int ret; 
    long threadRet;
    ret=pthread_join(pthid1,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    ret=pthread_join(pthid,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    printf("sale over\n");
    return 0;
}
```

先判断，后加锁时，

``` c
if(pArg->tickets>0)
{
    pthread_mutex_lock(&pArg->mutex);//加锁
    //printf("I am saleWindows1 start sale,%d\n",pArg->tickets);//卖票前打印剩余票数
    pArg->tickets--;//票-1
    i++;
    //printf("I am saleWindows1 sale finish,%d\n",pArg->tickets);//卖票后打印剩余票数
    pthread_mutex_unlock(&pArg->mutex);//解锁
}

```

$./sale_tickets 
I am saleWindows1,10058393
I am saleWindows2,9941607
sale over

会出现负票情况：当只剩一张的时候，两个线程都可进入，第一个卖出去了，第二个也能卖出去，卖了两张，出现负票。

 

票数越多越均衡：

创建子线程需要消耗时间，后创建的线程先执行，窗口2比窗口1 卖票多。

 



 

---



#### 总结：线程互斥mutex：

**加锁步骤如下：**

1. 定义一个全局的pthread_mutex_t lock; 
2. 或者用pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; //则main函数中不用init
3. 在main中调用 pthread_mutex_init函数进行初始化
4. 在子线程函数中调用pthread_mutex_lock加锁
5. 在子线程函数中调用pthread_mutex_unlock解锁
6. 最后在main中调用 pthread_mutex_destroy函数进行销毁

#### 

---

# 到这啦！！！

### 线程的同步

进程间通信：管道，消息队列，信号量

进程间同步：管道（每条管道占4k空间，会在内核中占较大空间），互斥锁（不常用），消息队列，信号量

线程的同步：互斥锁和条件变量同时使用（errorentfd占一个字节）

#### 条件变量cond

条件变量是利用线程间共享的全局变量进行同步的一种机制，主要包括两个动作：一个线程等待条件变量的条件成立而挂起；另一个线程使条件成立（给出条件成立信号）。为了防止竞争，条件变量的使用总是和一个互斥锁结合在一起。

man sem_wait

man sem_trywait

加锁解锁不常用

####  创建和注销

线程的创建：

* 静态方式使PTHREAD_COND_INITIALIZER常量，如下：

``` c 
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```

* 动态方式调用pthread_cond_init()函数，API定义如下：

``` c 
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);
```

尽管POSIX标准中为条件变量定义了属性，但在Linux Threads中没有实现，因此**cond_attr值通常为NULL**，且被忽略。

注销一个条件变量需要调用pthread_cond_destroy()，只有在没有线程在该条件变量上等待的时候能注销这个条件变量，否则返回EBUSY。因为Linux实现的条件变量没有分配什么资源，所以注销动作只包括检查是否有等待线程。API定义如下：

``` c 
int pthread_cond_destroy(pthread_cond_t *cond);
```

#### 等待和激发

线程解开mutex指向的锁，并被条件变量cond阻塞。

等待的两种方式：

* 无条件等待pthread_cond_wait();

``` c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```

* 计时等待pthread_cond_timedwait():

``` c 
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime);
```

计时等待方式表示经历abstime段时间后，即使条件变量不满足，阻塞也被解除。

---

无论哪种等待方式，都必须和一个互斥锁配合，以防止多个线程同时请求pthread_cond_wait(&cond,&mutex)（或pthread_cond_timedwait(&cond,&mutex)）的竞争条件（Race Condition）。

mutex互斥锁必须是普通锁（PTHREAD_MUTEX_TIMED_NP），且在调用pthread_cond_wait(&cond,&mutex)前必须由本线程加锁（pthread_mutex_lock(&mutex)），而在更新条件等待队列以前，mutex保持锁定状态，并在线程挂起进入等待前解锁。

在条件满足从而离开pthread_cond_wait(&cond,&mutex)之前，mutex将被重新加锁，以与进入pthread_cond_wait(&cond,&mutex)前的加锁动作对应。（也就是说在做pthread_cond_wait(&cond,&mutex)之前，往往要用pthread_mutex_lock(&mutex)进行加锁，而调用pthread_cond_wait(&cond,&mutex)函数会将锁解开，然后将线程挂起阻塞。）

---

无条件等待pthread_cond_wait();

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```

pthread_cond_wait(&cond,&mutex)内部

* 上半部

1. 排队在对应的条件变量cond上

2. 解锁

3. 等待

* 下半部

1. 醒来

2. 加锁(利于检测死锁问题)

3. 返回

 ``` c
pthread_mutex_lock(&mutex);
pthread_cond_wait(&cond,&mutex);
pthread_mutex_unlock(&mutex);
 ```

---

让条件变量成立（激发条件）

* pthread_cond_signal()激活一个等待该条件的线程，存在多个等待线程时按入队顺序激活其中一个。再将锁状态恢复为锁定状态，最后再用pthread_mutex_unlock进行解锁）。

``` c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
```

* pthread_cond_broadcast()激活所有等待线程

超时是指 当前时间+多长时间超时

---

一个线程等待条件变量（主线程等待wait，子线程继续执行）。

vim pthread_cond.c

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}
Data_t,*pData_t;void* threadFunc(void* p)
{
    pData_t pArg=(pData_t)p;//强转成条件变量指针
    pthread_mutex_lock(&pArg->mutex);//加锁
    pthread_cond_wait(&pArg->cond,&pArg->mutex);//变量1，条件变量，变量2，锁
    pthread_mutex_unlock(&pArg->mutex);//解锁
    printf("I am child thread wake up\n");
    pthread_exit(NULL);//退出
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_mutex_init(&threadInfo.mutex,NULL);//初始化锁，互斥
    pthread_cond_init(&threadInfo.cond,NULL);//初始化条件变量
    pthread_t pthid;
    pthread_create(&pthid,NULL,threadFunc,&threadInfo);//创建子线程
    usleep(5000);//主线程休眠,为了让子线程执行
    pthread_cond_signal(&threadInfo.cond);//让条件变量成立
    pthread_join(pthid,NULL);
    printf("I am main thread\n");
    return 0;
}
```

如果主进程不休眠，那么子进程还没wait,主进程已经让条件变量成立了。

创建子进程也需要时间，所以主进程休眠时间不能太短。

$./pthread_cond 
I am child thread wake up
I am main thread

---

内核线程？（为什么不是内核进程？）

所有内核线程共享同一块进程地址空间

---

两个线程等待条件变量	**wait是cancel点**
vim pthread_cond_twowait.c

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}Data_t,*pData_t;
void cleanup(void *p)//解锁，对已经解锁过的锁再次进行解锁是没问题的
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock ok\n");
}
void* threadFunc(void* p)
{
    pData_t pArg=(pData_t)p;
    pthread_mutex_lock(&pArg->mutex);//加锁，lock不是cancel点
    pthread_cleanup_push(cleanup,&pArg->mutex);//清理函数栈，可以避免死锁问题
    pthread_cond_wait(&pArg->cond,&pArg->mutex);
    /*wait是cancel点,未解锁时，cancel会死锁*/
    printf("wait return\n");
    //pthread_mutex_lock(&pArg->mutex);改成用cleanup函数解锁
    pthread_cleanup_pop(1);//清理函数栈，解锁
    printf("I am child thread wake up\n");
    pthread_exit(NULL);
}
#define N 2//线程数为2，如果需要多创建几个线程，在这里修改
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_cond_init(&threadInfo.cond,NULL);
    pthread_t pthid[N];
    int i;
    for(i=0;i<N;i++)
    {
        pthread_create(pthid+i,NULL,threadFunc,&threadInfo);//创建子线程
    }
    int ret;
    for(i=0;i<N;i++)
    {/*遇到wait，线程可能处于休眠状态，这时线程会被cancel*/
        ret=pthread_cancel(pthid[i]);//遇到cancel点，cancel掉对应子进程
        THREAD_ERROR_CHECK(ret,"pthread_cancel");
    }
    for(i=0;i<N;i++)
    {
        pthread_join(pthid[i],NULL);//等待删除子进程
    }
    printf("I am main thread\n");
    return 0;
}
```

情形1：（卡住）

$grep -elLf |grep wait   看到两个wait子线程状态mutex-

情形2：（用cleanup代替unlock避免死锁）

如何避免死锁问题？	cleanup_push放到wait前面

$./pthread_cond_twowait 
unlock ok
unlock ok
I am main thread

---

唤醒两次（一个signal只能唤醒一次线程）

vim pthread_cond_twosignal.c

成功返回0，失败返回错误码

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}Data_t,*pData_t;
void cleanup(void *p)
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock ok\n");
}
void* threadFunc(void* p)
{
    pData_t pArg=(pData_t)p;
    pthread_mutex_lock(&pArg->mutex);
    pthread_cleanup_push(cleanup,&pArg->mutex);
    printf("start wait\n");//一个人等待，解锁，第二个人也会进入等待
    ret=pthread_cond_wait(&pArg->cond,&pArg->mutex);//1等待卡住，然后2等待卡住
    printf("wait return\n");//打印，先等待先唤醒
    pthread_cleanup_pop(1);
	printf("I am child thread wake up,ret=%d\n",ret);
    printf("I am child thread wake up\n");//先抢到锁先打印
    pthread_exit(NULL);
}
#define N 2
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_cond_init(&threadInfo.cond,NULL);
    pthread_t pthid[N];
    int i;
    for(i=0;i<N;i++)
    {
        pthread_create(pthid+i,NULL,threadFunc,&threadInfo);
    }
    int ret;
    sleep(3);
    for(i=0;i<N;i++)
    {
        ret=pthread_cond_signal(&threadInfo.cond);
        THREAD_ERROR_CHECK(ret,"pthread_cond_signal");
    }
    for(i=0;i<N;i++)
    {
        pthread_join(pthid[i],NULL);
    }
    printf("I am main thread\n");
    return 0;
}
```

$./pthread_cond_twosignal 
start wait
start wait
wait return
unlock ok
I am child thread wake up
wait return
unlock ok
I am child thread wake up
I am main thread

---

计时等待：

``` 
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime);
```

绝对时间：abstime——time(NULL+3)

pthread_cond_timedwait.c

``` c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}Data_t,*pData_t;
void* threadFunc(void* p)
{
    pData_t pArg=(pData_t)p;
    struct timespec t;
    //t.tv_sec=3;//会超时，返回值110
    t.tv_sec=time(NULL)+3;//3s后超时，，如果在3s内发送signal，程序正常唤醒，返回值0
    t.tv_nsec=0;
    int ret;
    pthread_mutex_lock(&pArg->mutex);
    ret=pthread_cond_timedwait(&pArg->cond,&pArg->mutex,&t);
    pthread_mutex_unlock(&pArg->mutex);
    printf("I am child thread wake up,ret=%d\n",ret);
    pthread_exit(NULL);
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_cond_init(&threadInfo.cond,NULL);
    pthread_t pthid;
    pthread_create(&pthid,NULL,threadFunc,&threadInfo);
    sleep(1);
    pthread_cond_signal(&threadInfo.cond);//发送signal，程序正常唤醒，返回值0
    pthread_join(pthid,NULL);
    printf("I am main thread\n");
    return 0;
}
```

$./pthread_cond_timedwait （等待1s）
I am child thread wake up,ret=0
I am main thread

---



#### 其他

pthread_cond_wait()和pthread_cond_timedwait()都被实现为取消点。

也就是说如果pthread_cond_wait()被取消，则退出阻塞，然后将锁状态恢复，则此时mutex是保持锁定状态的，而当前线程已经被取消掉，那么解锁的操作就会得不到执行，此时锁得不到释放，就会造成死锁，因而需要定义退出回调函数来为其解锁。

回调函数保护，等待条件前锁定，pthread_cond_wait()返回后解锁。
条件变量机制和互斥锁一样，不能用于信号处理中，在信号处理函数中调用pthread_cond_signal()或者pthread_cond_broadcast()很可能引起死锁。

---

##### 火车站售票2.0（卖完再放票，分时段放票）

趁锁

``` c
#include <func.h>
typedef struct{
    int tickets;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}Data_t,*pData_t;
void cleanup(void *p)
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock success\n");
}

void* saleWindows1(void* p)
{
    Data_t* pArg=(Data_t*)p;
    int i=0;
    while(1)
    {
        pthread_mutex_lock(&pArg->mutex);
        if(pArg->tickets>0)
        {
            printf("I am saleWindows1 start sale,%d\n",pArg->tickets);
            pArg->tickets--;
            i++;
            if(!pArg->tickets)
            {
                pthread_cond_signal(&pArg->cond);
            }
            printf("I am saleWindows1 sale finish,%d\n",pArg->tickets);
            pthread_mutex_unlock(&pArg->mutex);
            sleep(1);
        }else{
            pthread_mutex_unlock(&pArg->mutex);
            printf("I am saleWindows1,%d\n",i);
            break;
        }
    }
    return NULL;
}
void* saleWindows2(void* p)
{
    Data_t* pArg=(Data_t*)p;
    int i=0;
    while(1)
    {
        pthread_mutex_lock(&pArg->mutex);
        if(pArg->tickets>0)
        {
            printf("I am saleWindows2 start sale,%d\n",pArg->tickets);
            pArg->tickets--;
            i++;
            if(!pArg->tickets)//判断票数是否等于0
            {
                pthread_cond_signal(&pArg->cond);//唤醒重新放票函数
            }
            printf("I am saleWindows2 sale finish,%d\n",pArg->tickets);
            pthread_mutex_unlock(&pArg->mutex);
            sleep(1);
        }else{
            pthread_mutex_unlock(&pArg->mutex);
            printf("I am saleWindows2,%d\n",i);
            break;
        }
    }
    return NULL;
}
void* setTickets(void* p)//重新放票
{
    Data_t* pArg=(pData_t)p;//
    pthread_mutex_lock(&pArg->mutex);//加锁
    if(pArg->tickets>0)//票数大于0
    {//睡觉
        pthread_cond_wait(&pArg->cond,&pArg->mutex);
    }
    pArg->tickets=20;//每次放票20张
    pthread_mutex_unlock(&pArg->mutex);//解锁
}
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    threadInfo.tickets=20;//最开始的票数20张
    pthread_t pthid,pthid1,pthid2;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_cond_init(&threadInfo.cond,NULL);
    pthread_create(&pthid,NULL,saleWindows1,&threadInfo);
    pthread_create(&pthid1,NULL,saleWindows2,&threadInfo);
    pthread_create(&pthid2,NULL,setTickets,&threadInfo);
    int ret; 
    long threadRet;
    ret=pthread_join(pthid1,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    ret=pthread_join(pthid,(void**)&threadRet);
    THREAD_ERROR_CHECK(ret,"pthread_join");
    printf("sale over\n");
    return 0;
}
```

$./sale_tickets 

``` 
I am saleWindows2 start sale,20
I am saleWindows2 sale finish,19
I am saleWindows1 start sale,19
I am saleWindows1 sale finish,18
I am saleWindows2 start sale,18
I am saleWindows2 sale finish,17
I am saleWindows1 start sale,17
I am saleWindows1 sale finish,16
I am saleWindows2 start sale,16
I am saleWindows2 sale finish,15
I am saleWindows1 start sale,15
I am saleWindows1 sale finish,14
I am saleWindows2 start sale,14
I am saleWindows2 sale finish,13
I am saleWindows1 start sale,13
I am saleWindows1 sale finish,12
I am saleWindows2 start sale,12
I am saleWindows2 sale finish,11
I am saleWindows1 start sale,11
I am saleWindows1 sale finish,10
I am saleWindows2 start sale,10
I am saleWindows2 sale finish,9
I am saleWindows1 start sale,9
I am saleWindows1 sale finish,8
I am saleWindows2 start sale,8
I am saleWindows2 sale finish,7
I am saleWindows1 start sale,7
I am saleWindows1 sale finish,6
I am saleWindows2 start sale,6
I am saleWindows2 sale finish,5
I am saleWindows1 start sale,5
I am saleWindows1 sale finish,4
I am saleWindows2 start sale,4
I am saleWindows2 sale finish,3
I am saleWindows1 start sale,3
I am saleWindows1 sale finish,2
I am saleWindows2 start sale,2
I am saleWindows2 sale finish,1
I am saleWindows1 start sale,1
I am saleWindows1 sale finish,0
I am saleWindows2 start sale,20
I am saleWindows2 sale finish,19
I am saleWindows1 start sale,19
I am saleWindows1 sale finish,18
I am saleWindows2 start sale,18
I am saleWindows2 sale finish,17
I am saleWindows1 start sale,17
I am saleWindows1 sale finish,16
I am saleWindows2 start sale,16
I am saleWindows2 sale finish,15
I am saleWindows1 start sale,15
I am saleWindows1 sale finish,14
I am saleWindows2 start sale,14
I am saleWindows2 sale finish,13
I am saleWindows1 start sale,13
I am saleWindows1 sale finish,12
I am saleWindows2 start sale,12
I am saleWindows2 sale finish,11
I am saleWindows1 start sale,11
I am saleWindows1 sale finish,10
I am saleWindows2 start sale,10
I am saleWindows2 sale finish,9
I am saleWindows1 start sale,9
I am saleWindows1 sale finish,8
I am saleWindows2 start sale,8
I am saleWindows2 sale finish,7
I am saleWindows1 start sale,7
I am saleWindows1 sale finish,6
I am saleWindows2 start sale,6
I am saleWindows2 sale finish,5
I am saleWindows1 start sale,5
I am saleWindows1 sale finish,4
I am saleWindows2 start sale,4
I am saleWindows2 sale finish,3
I am saleWindows1 start sale,3
I am saleWindows1 sale finish,2
I am saleWindows2 start sale,2
I am saleWindows2 sale finish,1
I am saleWindows1 start sale,1
I am saleWindows1 sale finish,0
I am saleWindows2,20
I am saleWindows1,20
sale over
```

卖完20张票后再放一次票（20张）

---

pthread_cond_broadcast()：条件一瞬间完毕

 ``` c
#include <pthread.h>
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_signal(pthread_cond_t *cond);
线程全部创建好，等用户发请求
vim pthread_cond_broadcast.c
#include <func.h>
typedef struct{
    pthread_mutex_t mutex;
    pthread_cond_t cond;
}Data_t,*pData_t;
void cleanup(void *p)
{
    pthread_mutex_unlock((pthread_mutex_t*)p);
    printf("unlock ok\n");
}
void* threadFunc(void* p)
{
    pData_t pArg=(pData_t)p;
    pthread_mutex_lock(&pArg->mutex);
    pthread_cleanup_push(cleanup,&pArg->mutex);
    printf("start wait\n");
    pthread_cond_wait(&pArg->cond,&pArg->mutex);
    printf("wait return\n");
    pthread_cleanup_pop(1);
    printf("I am child thread wake up\n");
    pthread_exit(NULL);
}
#define N 2
int main(int argc,char* argv[])
{
    Data_t threadInfo;
    pthread_mutex_init(&threadInfo.mutex,NULL);
    pthread_cond_init(&threadInfo.cond,NULL);
    pthread_t pthid[N];
    int i;
    for(i=0;i<N;i++)
    {
        pthread_create(pthid+i,NULL,threadFunc,&threadInfo);
    }
    int ret;
    sleep(3);
    ret=pthread_cond_broadcast(&threadInfo.cond);
    THREAD_ERROR_CHECK(ret,"pthread_cond_broadcast");
    for(i=0;i<N;i++)
    {
        pthread_join(pthid[i],NULL);
    }
    printf("I am main thread\n");
    return 0;
}
 ```

$./pthread_cond_broadcast 
start wait
start wait
wait return
unlock ok
I am child thread wake up
wait return
unlock ok
I am child thread wake up
I am main thread

---



#### 生产者消费者问题

* 链表实现

``` c
#include <pthread.h>
#include <unistd.h>
#include<stdio.h>
#include<malloc.h>
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
struct node {
int n_number;
struct node *n_next;
} *head = NULL;
static void cleanup_handler(void *arg)
{
printf("Cleanup handler of second thread\n");
free(arg);
pthread_mutex_unlock(&mtx);
}
static void *thread_func(void *arg)//消费者
{
struct node *p = NULL;
pthread_cleanup_push(cleanup_handler, p);
while (1)
{
If(
pthread_mutex_lock(&mtx);
while (head == NULL || (flag=0)){ pthread_cond_wait(&cond,&mtx);}
p = head;
head = head->n_next;
printf("Got %d from front of queue\n", p->n_number);
free(p);
pthread_mutex_unlock(&mtx);
else
}
pthread_exit(NULL);
pthread_cleanup_pop(0); //必须放在最后一行
}
int main(void)
{
pthread_t tid;
int i;
struct node *p;
pthread_create(&tid, NULL, thread_func, NULL);
for (i = 0; i < 10; i++)//生产者
{
p = (struct node*)malloc(sizeof(struct node));
p->n_number = i;
pthread_mutex_lock(&mtx);//因为head是共享的，访问共享数据必须要加锁
p->n_next = head;
head = p;
    pthread_cond_signal(&cond);
pthread_mutex_unlock(&mtx);
sleep(1);
}
printf("thread 1 wanna end the line.So cancel thread 2.\n");
pthread_cancel(tid);
pthread_join(tid, NULL);
printf("All done------exiting\n");
return 0;
}
```

* 队列实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <pthread.h>
#define BUFFER_SIZE 16 //表示一次最多可以不间断的生产16个产品
#define N 50  //总共生产产品数
struct prodcons {
int buffer[BUFFER_SIZE]; /* 数据 */
pthread_mutex_t lock; /* 加锁 */
int readpos, writepos; /* 读pos 写位置 */
pthread_cond_t notempty; /* 不空，可以读 */
pthread_cond_t notfull; /* 不满，可以写 */
};
/* 初始化*/
void init(struct prodcons * b)
{
pthread_mutex_init(&b->lock, NULL); //初始化锁
pthread_cond_init(&b->notempty, NULL); //初始化条件变量
pthread_cond_init(&b->notfull, NULL); //初始化条件变量
b->readpos = 0; //初始化读取位置从0开始
b->writepos = 0; //初始化写入位置从0开始
}
/* 销毁操作 */
void destroy(struct prodcons *b)
{
pthread_mutex_destroy(&b->lock);
pthread_cond_destroy(&b->notempty);
pthread_cond_destroy(&b->notfull);
}
void put(struct prodcons * b, int data)//生产者
{
pthread_mutex_lock(&b->lock);
while ((b->writepos + 1) % BUFFER_SIZE == b->readpos) {//判断是不是满了
printf("wait for not full\n");
pthread_cond_wait(&b->notfull, &b->lock); //此时为满，不能生产，等待不满的信号
    }
/*下面表示还没有满，可以进行生产*/
b->buffer[b->writepos] = data;
b->writepos++; //写入点向后移一位
if (b->writepos >= BUFFER_SIZE) b->writepos = 0; //如果到达最后，就再转到开头
pthread_cond_signal(&b->notempty); //此时有东西可以消费，发送非空的信号
pthread_mutex_unlock(&b->lock);
}
int get(struct prodcons * b)//消费者
{
pthread_mutex_lock(&b->lock);
while (b->writepos == b->readpos) {//判断是不是空
printf("wait for not empty\n");
pthread_cond_wait(&b->notempty, &b->lock); //此时为空，不能消费，等待非空信号
}
/*面表示还不为空，可以进行消费*/
int data = b->buffer[b->readpos];
b->readpos++; //读取点向后移一位
if (b->readpos >= BUFFER_SIZE) b->readpos = 0; //如果到达最后，就再转到开头
pthread_cond_signal(&b->notfull); //此时可以进行生产，发送不满的信号
pthread_mutex_unlock(&b->lock);
return data;
}
/*--------------------------------------------------------*/
#define OVER (-1) //定义结束标志
struct prodcons buffer; //定义全局变量
/*--------------------------------------------------------*/
void * producer(void * data)
{
int n = 0;
for (; n < N; n++) {
printf(" put-->%d\n", n);
put(&buffer, n);
}
put(&buffer, OVER);
printf("producer stopped!\n");
pthread_exit(NULL);
}
/*--------------------------------------------------------*/
void * consumer(void * data)
{
while (1) {
int d = get(&buffer);
if (d == OVER ) break;
printf(" %d-->get\n", d);
    }
printf("consumer stopped!\n");
pthread_exit(NULL);
}
/*--------------------------------------------------------*/
int main(void)
{
pthread_t th_a, th_b;
init(&buffer);
pthread_create(&th_a, NULL, producer, 0);
pthread_create(&th_b, NULL, consumer, 0);
pthread_join(th_a, NULL);
pthread_join(th_b, NULL);
destroy(&buffer);
return 0;
}
```

wait for not empty
 put-->0
 put-->1
 put-->2
 0-->get
 1-->get
wait for not empty
 put-->3
 put-->4
wait for not full
 2-->get
 3-->get
wait for not empty
wait for not full
 4-->get
wait for not empty
producer stopped!
consumer stopped!

---

