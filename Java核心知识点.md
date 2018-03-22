#Java核心知识点
@(读书笔记)[Java|精品]

##JUC（Java并发工具包）

1. volatile 关键字
2. 原子变量AtomicInteger
3. CAS 算法，乐观锁，如果跟预期的不一样则不更新
4. 并发容器类（1）ConcurrentHashMap 同步容器类是 Java5 增加的一个线程安全的哈希表;介于 HashMap 与 Hashtable 之间;内部采用"锁分段"机制替代Hashtable的独占锁,进而提高性能
5. 线程池
6. 线程ScheduledThreadPool
7. 锁机制(ReentrantLock, ReadWriteLock)
8. fork&join大任务分解最后合并
9. threadlocal

##volatile
volatile变量每次读取之间会刷新自己,从主内存中读取.因此他在读取的那时应该是没有问题的,但是后面的一致性就无法保证了.(load,use),(assign,stor)连续出现,其他的没有这种硬性要求
volatile禁止指令重排优化,为了更快的实现读写内存而将指令不按照顺序执行,在多线程环境下可能会导致问题,线程之间
不适用修改基于上一次的（i++），适用于设置不依赖上一次结果的状态
拥有可见性

##原子变量AtomicInteger
通过cas实现

##CAS compare and swap
如果更新之前发现值改变则不更新，反之则更新

##并发容器
   
1. Blockingqueue：具体参考线程池
2. ConcurrentHashMap,使用粒度更细的加锁机制来实现更大程度的共享——分段锁.任意数量的读取线程可以并发访问Map
执行读取操作的线程和执行写入操作的线程可以并发访问Map,一定数量的写入线程可以并发修改Map
ConcurrentHashMap的权衡：size和isEmpty这样的方法只是一个估计值，被换取了更重要的性能优化，包括get,put,containsKey,remove等
ConcurrentHashMap代替Map进一步提高代码的可伸缩性。只有当应用程序需要加锁Map进行独占访问时，才应该放弃用ConcurrentHashMap
ConcurrentMap接口：定义了原子“复合”操作（If-No-Add, Remove-If-Equal, Replace-If-Equal）
3. CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。


##Java线程池
假设一个服务器完成一项任务所需时间为：T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间。 如果：T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。
一个线程池包括以下四个基本组成部分：
1. 线程池管理器（ThreadPool）：用于创建并管理线程池，包括 创建线程池，销毁线程池，添加新任务；
2. 工作线程（PoolWorker）：线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；
3. 任务接口（Task）：每个任务必须实现的接口，以供工作线程调度任务的执行，它主要规定了任务的入口，任务执行完后的收尾工作，任务的执行状态等；
4. 任务队列（taskQueue）：用于存放没有处理的任务。提供一种缓冲机制。

好处
1. 节约进程创建和销毁时间
2. 通过调整线程数量充分利用计算机资源

 线程池的工作过程如下：
1.  如果当前池大小 poolSize 小于 corePoolSize ，则创建新线程执行任务。
2.  如果当前池大小 poolSize 大于 corePoolSize ，且等待队列未满，则进入等待队列
3.  如果当前池大小 poolSize 大于 corePoolSize 且小于 maximumPoolSize ，且等待队列已满，则创建新线程执行任务。
4.  如果当前池大小 poolSize 大于 corePoolSize 且大于 maximumPoolSize ，且等待队列已满，则调用拒绝策略来处理该任务。拒绝策略包括（1.不执行不抛异常 2、不执行抛异常 3、丢弃最老任务，并尝试重新提交 4、直接在 execute 方法的调用线程中运行被拒绝的任务，可能无法成功 ）
5.  线程池里的每个线程执行完任务后不会立刻退出，而是会去检查下等待队列里是否还有线程任务需要执行，如果在 keepAliveTime 里等不到新的任务了，那么线程就会退出。 

SingleThreadExecutor：单个后台线程，管理所有工作线程，添加和移除，异常退出会有一个新的顶替LinkedBlockingQueue

FixedThreadPool：只有核心线程的线程池,大小固定。创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

CachedThreadPool：无界线程池，可以进行自动线程回收。如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小

ScheduledThreadPool：核心线程池固定，大小无限的线程池。此线程池支持定时以及周期性执行任务的需求，创建一个周期性执行任务的线程池。如果闲置,非核心线程池会在DEFAULT_KEEPALIVEMILLIS时间内回收。跟Timer的区别在于Timer是一个单线程，可能会出现某个定时任务特别长阻塞其他定时任务的情况。该线程池多线程可以避免该情况

几种队列（是接口Blockingqueue的实现）
0. SynchronousQueue，它将任务直接提交给线程而不保持它们
1. ArrayBlockingQueue是数组实现的线程安全的有界的阻塞队列。
2.  LinkedBlockingQueue是单向链表实现的(指定大小)阻塞队列，该队列按 FIFO（先进先出）排序元素。可能会永远无法产生新的线程
3.  LinkedBlockingDeque是双向链表实现的(指定大小)双向并发阻塞队列，该阻塞队列同时支持FIFO和FILO两种操作方式。
4.  ConcurrentLinkedQueue是单向链表实现的无界队列，该队列按 FIFO（先进先出）排序元素。
5.  ConcurrentLinkedDeque是双向链表实现的无界队列，该队列同时支持FIFO和FILO两种操作方式

ArrayBlockingQueue和LinkedBlockingQueue的区别：
1. 队列中锁的实现不同，ArrayBlockingQueue实现的队列中的锁是没有分离的，即生产和消费用的是同一个锁；LinkedBlockingQueue实现的队列中的锁是分离的，即生产用的是putLock，消费是takeLock
2. 在生产或消费时操作不同，ArrayBlockingQueue实现的队列中在生产和消费的时候，是直接将枚举对象插入或移除的；LinkedBlockingQueue实现的队列中在生产和消费的时候，需要把枚举对象转换为Node<E>进行插入或移除，会影响性能
3. 队列大小初始化方式不同，ArrayBlockingQueue实现的队列中必须指定队列的大小；LinkedBlockingQueue实现的队列中可以不指定队列的大小，但是默认是Integer.MAX_VALUE

#锁机制(ReentrantLock, ReadWriteLock)
*Java几乎没有不可重入的锁，重入指的是同一个线程获得锁之后可以多次进入
reentrantlock具有三个特点
1. ReenTrantLock提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程。
2. 有超时机制，tryLock(long timeout, TimeUnit unit),能够中断等待锁的线程的机制，通过lock.lockInterruptibly()来实现这个机制
3. 可以通过AQS实现公平锁（先来先获得）

synchronized（S）    VS     lock（L） 1） L 是接口，S 是关键字 2）S 在发生异常时，会自动释放线程占有的锁，不会发生死锁。L 在发生异常时，若没 有主动通过 unlock（）释放锁，则很有可能造成死锁。所以用 lock 时要在 finally 中释放锁。
 3)L 可以当等待锁的线程响应中断，而 S 不行，使用 S 时，等待的线程将会一直等下去， 不能响应中断。 4）通过 L 可以知道是否成功获得锁，S 不可以。 5）L 可以提高多个线程进行读写操作的效率。 
公平的获取锁，也就是等待时间最长的线程最优获取到锁，ReentraantLock是基于同步队列AQS来管理获取锁的线程。在公平的机制下，线程依次排队获取锁，先进入队列排队的线程，等到时间越长的线程最优获取到锁。

ReentrantReadWriteLock 有读锁和写锁，读锁之间共享，写锁之间互斥，读写互斥

8. fork&join大任务分解最后合并
主要过程为继承RecursiveTask实现compute方法，然后调用fork启动线程，最后join结果

##线程同步的几种方法

1. 同步方法synchronize，非static锁对象，static锁住整个类
2. 同步代码块 synchronize
3. wait & notify 无法指明唤醒的进程，区别JUC的condition
4. volatile，一般用于修改状态或者值不依赖上一个状态
5. 重入锁reenterantlock
6. 使用局部变量（TLAB） threadlocal，有get set方法
7. 阻塞队列BlockingQueue，类似生产者消费者，LinkedBlokcingQueue有读锁和写锁，ArrayBlockingqueue只有一把锁
8. 使用原子变量实现线程同步，一般是自增自减的场景比较多

#threadlocal

线程独有的变量，一般用来存放线程独立的资源，比如spring中一些线程独有的bean从而实现数据隔离





 