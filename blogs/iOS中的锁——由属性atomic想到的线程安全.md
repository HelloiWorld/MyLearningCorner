本文不介绍各种锁的高级用法，只是整理锁相关的知识点，帮助理解。

# 锁的作用
防止在多线程（多任务）的情况下对共享资源（临界资源）的脏读或者脏写。

## 自旋锁和互斥锁
**共同点**：都能保证同一时刻只能有一个线程操作锁住的代码。都能保证线程安全。
**不同点**：
  - **互斥锁**（mutex）：当上一个线程的任务没有执行完毕的时候（被锁住），那么下一个线程会进入睡眠状态等待任务执行完毕(sleep-waiting)，当上一个线程的任务执行完毕，下一个线程会自动唤醒然后执行任务。
  - **自旋锁**（Spin lock）：当上一个线程的任务没有执行完毕的时候（被锁住），那么下一个线程会一直等待（busy-waiting），当上一个线程的任务执行完毕，下一个线程会立即执行。
  - 由于自旋锁不会引起调用者睡眠，所以自旋锁的效率远高于互斥锁
  - 自旋锁会一直占用CPU，也可能会造成死锁
> [自旋锁有bug！](https://lists.swift.org/pipermail/swift-dev/Week-of-Mon-20151214/000372.html)  ibireme大神的文章[《不再安全的 OSSpinLock》](http://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
   不同优先级线程调度算法会有优先级反转问题，比如低优先级获锁访问资源，高优先级尝试访问时会等待，这时低优先级又没法争过高优先级导致任务无法完成lock释放不了。

# 1. 原子操作
- `nonatomic`：非原子属性，非线程安全，适合小内存移动设备
- `atomic`：原子属性，default，线程安全（内部使用自旋锁，iOS10后改为了互斥锁），消耗大量资源
  - 单写多读，只为setter方法加锁，不影响getter
  - 相关代码如下：

        static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy) 
        {
           if (offset == 0) {
               object_setClass(self, newValue);
               return;
           }

           id oldValue;
           id *slot = (id*) ((char*)self + offset);

           if (copy) {
               newValue = [newValue copyWithZone:nil];
           } else if (mutableCopy) {
               newValue = [newValue mutableCopyWithZone:nil];
           } else {
               if (*slot == newValue) return;
               newValue = objc_retain(newValue);
           }

           if (!atomic) {
               oldValue = *slot;
               *slot = newValue;
           } else {
               spinlock_t& slotlock = PropertyLocks[slot];
               slotlock.lock();
               oldValue = *slot;
               *slot = newValue;        
               slotlock.unlock();
           }

           objc_release(oldValue);
        }

        void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, id newValue, BOOL atomic, signed char shouldCopy) 
        {
           bool copy = (shouldCopy && shouldCopy != MUTABLE_COPY);
           bool mutableCopy = (shouldCopy == MUTABLE_COPY);
           reallySetProperty(self, _cmd, newValue, offset, atomic, copy, mutableCopy);
        }
很容易理解的代码，可变拷贝和不可变拷贝会开辟新的空间，两者皆不是则持有（引用计数+1），相比`nonatomic`只是多了一步锁操作。

# 2. synchronized 条件锁
使用最简单，性能也最差。
    
    @synchronized(obj) {
        // 内部会添加异常处理，所以耗时
        NSLog(@"自动加锁，自动解锁，自动销毁");
    }
obj为该锁的唯一标识，只有当标识相同时，才为满足互斥

# 3. dispatch_semaphore 信号量
用于线程同步，有序访问。不支持递归。
- 创建信号：`dispatch_semaphore_create(long value)` 传入值必须 >=0, 若传入为 0 则阻塞线程并等待`timeout`，时间到后会执行其后的语句
- 等待信号到达：`dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)` 使得`signal`值`-1`
- 发送信号，解除等待状态：`dispatch_semaphore_signal(dispatch_semaphore_t deem)`使得`signal`值`+1`

> 这里顺便写一下`dispatch_barrier`，一个`dispatch_barrier`允许在一个**并发队列**(如果是串行队列或全局队列相当于`dispatch_(a)sync`)中创建一个同步点。当在并发队列中遇到一个barrier，他会延迟执行barrier的block，等待所有在barrier之前提交的blocks执行结束。 这时，barrier block自己开始执行。 之后， 队列继续正常的执行操作。
`dispatch_barrier_async`和`dispatch_barrier_sync`的区别在于是否会等待自己的block完成再执行后面的任务。

###### 一个同步访问网络的例子
    - (void)syncRequestWithUrl:(NSURL*)url {
        NSMutableURLRequest *req = [[NSMutableURLRequest alloc]initWithURL:url];
        
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
            
        __block NSURLSessionTask *dataTask = [[NSURLSession sharedSession] dataTaskWithRequest:req completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
            if (!error && data) {

            } else {
                NSLog(@"error: %@",[error description]);
            }
            dispatch_semaphore_signal(semaphore);
        }];
        [dataTask resume];
            
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    }

# 4. pthread_mutex 互斥锁
Facebook的 [KVOController](https://github.com/facebook/KVOController) 有使用到。(先使用的是`OSSpinLock`，由于自旋锁的优先级反转问题后改用`pthread_mutex`)

使用需导入头文件：
    #import <pthread.h>
- 声明：`pthread_mutex_t _mutex;`
- 初始化：`pthread_mutex_init(&_mutex, NULL);`
- 加锁：`pthread_mutex_lock(&_mutex);`
- 解锁：`pthread_mutex_unlock(&_mutex);`
- 销毁：`pthread_mutex_destroy(&_mutex);`

# 5. pthread_mutex(recursive) 递归锁
递归锁允许同一个线程在未释放其拥有的锁时反复对该锁进行加锁操作。不会造成死锁。 

    static pthread_mutex_t pLock;
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr); //初始化attr并且给它赋予默认
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE); //设置锁类型，这边是设置为递归锁
    pthread_mutex_init(&pLock, &attr); // 初始化的时候带入参数
    pthread_mutexattr_destroy(&attr); //销毁一个属性对象，在重新进行初始化之前该结构不能重新使用

# 6. OSSpinLock 自旋锁
是性能最佳的锁，但由于**线程调度算法**（高优先级线程始终会在低优先级线程前执行，一个线程不会受到比它更低优先级线程的干扰）优先级反转的原因逐渐被其他锁取代。不支持递归。

使用需导入头文件    
    #import <libkern/OSAtomic.h>
- `OSSpinLock oslock = OS_SPINLOCK_INIT;`：默认值为 0，在 locked 状态时就会大于 0，unlocked状态下为 0
- `OSSpinLockLock(&oslock);`：上锁
- `OSSpinLockUnlock(&oslock);`：解锁
- `OSSpinLockTry(&oslock)`：尝试加锁，可以加锁则立即加锁并返回`YES`，反之返回`NO`

> 提一下`os_unfair_lock`，苹果解决优先级反转的问题整出的。。
    `os_unfair_lock_t unfairLock;`
    `unfairLock = &(OS_UNFAIR_LOCK_INIT);`
    `os_unfair_lock_lock(unfairLock);`
    `os_unfair_lock_unlock(unfairLock);`

# 7. NSLock 
Cocoa提供的最基本的锁对象，实际是在内部封装了`pthread_mutex`。对象锁均实现了`NSLocking`协议。

    @protocol NSLocking

    - (void)lock;
    - (void)unlock;

    @end
- `trylock`：能加锁返回`YES`并执行加锁操作，相当于`lock`，反之返回`NO`
- `lockBeforeDate`：表示会在传入的时间内尝试加锁，若能加锁则执行加锁操作并返回`YES`，反之返回`NO`

# 8. NSRecursiveLock 递归锁
`NSLock`的递归版本，解决在循环或递归时造成的死锁问题。使用同上。

# 9. NSCondition
最基本的条件锁，底层是通过条件变量(condition variable) pthread_cond_t 来实现，实际上封装了一个互斥锁和条件变量。手动控制线程`wait`和`signal`
- `wait`：进入等待状态
- `waitUntilDate:`：让一个线程等待一定的时间
- `signal`：唤醒一个等待的线程
- `broadcast`：唤醒所有等待的线程

# 10. NSConditionLock 条件锁
API名说明一切...

    @interface NSConditionLock : NSObject <NSLocking> {
    @private
        void *_priv;
    }

    - (instancetype)initWithCondition:(NSInteger)condition NS_DESIGNATED_INITIALIZER;

    @property (readonly) NSInteger condition;
    - (void)lockWhenCondition:(NSInteger)condition;
    - (BOOL)tryLock;
    - (BOOL)tryLockWhenCondition:(NSInteger)condition;
    - (void)unlockWithCondition:(NSInteger)condition;
    - (BOOL)lockBeforeDate:(NSDate *)limit;
    - (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;

    @property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

    @end
性能相对较差，但使用`NSConditionLock`可以处理任务间的依赖关系。

# Additional
### 11. pthread_rwlock 读写锁
读写锁是用来解决文件读写问题的，读操作可以共享，写操作是排他的，读可以有多个在读，写只有唯一个在写，同时写的时候不允许读。
- 当读写锁被一个线程以读模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程还可以继续进行
- 当读写锁被一个线程以写模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程也被阻塞


    // 初始化
    pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
    // 读模式
    pthread_rwlock_wrlock(&lock);
    // 写模式
    pthread_rwlock_rdlock(&lock);
    // 读模式或者写模式的解锁
    pthread_rwlock_unlock(&lock);

> **对于读数据比修改数据频繁的应用，用读写锁代替互斥锁可以提高效率**。因为使用互斥锁时，即使是读出数据（相当于操作临界区资源）都要上互斥锁，而采用读写锁，则可以在任一时刻允许多个读出者存在，提高了更高的并发度，同时在某个写入者修改数据期间保护该数据，以免任何其它读出者或写入者的干扰。

### 12. NSDistributedLock 分布式锁
Mac开发使用，mark

# 引用
###### 更详细的资料参考：
> [iOS 开发中的八种锁](http://www.jianshu.com/p/8b8a01dd6356)   
   [iOS中保证线程安全的几种方式与性能对比](http://www.jianshu.com/p/938d68ed832c)
   [深入理解iOS开发中的锁](http://ios.jobbole.com/89470/)
