#Understanding The JVM
@(读书笔记)[Java|原理]

##JAVA运行时的数据区域

1. 程序计数器:当前程序执行到的字节码的行号表示器,每个线程独立,Native方法为空
2. Java虚拟机栈:线程私有,存储局部变量(8基本数据类型,对象引用),操作数栈,动态链接方法出口
3. 本地方法栈,为Native方法服务
4. Java堆:存放对象实例,是垃圾管理机制的主要区域,基本基于分代手机算法,线程共享,但是也可以分配出线程私有的TLAB(thread local Allocation buffer)
5. 方法区,编译器编译后的数据,线程共享,有点像堆
6. 运行时常量池,方法区的一部分,存放编译期生成的各种字面量和符号引用,常量并不一定编译期产生
7. 直接内存:通过Native函数分配堆外内存

##对象的创建

1. 指针碰撞:有一个指针用来区分已分配内存,和未分配内存,内存是规整的
2. 空闲列表:维护一个列表,记录哪些内存是空闲的,分配之后更新列表

**TLAB**线程独享的内存,不够再分配,互相之间不干扰

Hotspot默认分配策略:相同宽度的字段总是分配到一起,父类中定义的变量会出现在子类之前,子类中较窄的变量会插入父类.对齐填充充当占位符的作用

##对象的访问定位
1. 句柄访问: 引用指向两个指针,实例数据和对象数据的指针,在实例数据频繁移动的情况下有优势
2. 直接指针访问:引用直接指向实例数据,少了一次寻址

##垃圾收集器与内存分配策略

1. 引用计数法有对象引用则+1,没有则-1,到0就回收,实际上java虚拟机并没有采取这种方式,因为无法解决循环引用问题
2. 可达性分析,通过判断对象到GC root是否存在可达链来判断对象是否存活,主要包括虚拟机栈中引用的对象和方法区中类静态属性的引用对象,方法区中常量引用对象,本地方法栈中JNI

1. 强引用: 正常的对象应用都是强引用,除非没了引用否则强引用指向的对象不会被回收
2. 软引用: 在内存不够的时候会强制回收的引用
3. 弱引用: 下一次垃圾回收就会回收他引用的对象
4. 虚引用: 有没有完全不影响垃圾回收,也无法通过他来获得对象,只是通过他在回收的时候给一个系统通知

即使对象已经完全不可达,标记为不可达之后还是试图调用他的finalize方法,如果在该方法中复活则可继续存在,否则一般都会被回收.finalize方法之调用一次,下次将不再调用直接回收,为了防止finalize方法卡死整个回收队列,因此不承诺一定执行或者执行完毕.

###回收方法区

1. 如果没有引用指向常量,那么就可以回收
2. 无用的类:a)类的实例都已经回收,b)加载该类的classLoader已经被回收,c)java.lang.Class对象没有在任何地方被引用,无法在任何地方通过反射该类的方法.

###垃圾收集算法

1. 标记-清除算法:首先标记所有要回收的对象,然后将标记对象清除(效率低,并且产生大量的碎片)
2. 复制算法(新生代回收算法,只有最多10%的会被回收,不够的放到老年代):内存对半分,每次使用其中一半,每次清理将存活的对象放到未使用的分区里,一次清除使用的那一般空间,将新的一半作为当前分配区域
3. 标记-整理算法(老年代),标记完的对象向内存一端对齐,清理掉边界以外的内存
4. 分代收集算法:新生代采用复制算法,老年代采用标记清除或者整理

###HotSpot算法实现

垃圾回收的时候需要在内存区域稳定的时候,否则没有意义,因此内存回收的时候需要停顿所有Java执行线程

1.  通过一种被称之为Oomaps的数据结构来快速定位引用位置
####安全点
指令序列复(方法调用,,循环跳转,异常跳转)用的地方会成为安全区让垃圾回收成为可能,有两种针对安全点的中断方式
1. 抢先式中断,中断到来的时候去尝试中断所有线程,如果线程还不安全那么让他运行到安全点去中断,从而完成GC.
2. 主动式中断:需要中断线程的时候设置一个中断位,当线程跑到安全点的时候去看是否需要中断,需要则中断自己
####安全区域
如果线程sleep或者block那么无法跑到安全点,那么需要设定安全区域.在安全曲雨中,引用关系不再发生变化,,如果在安全区域中完成了根节点枚举或者gc过程,才可以离开,否则等待命令才能离开

####垃圾收集器

1. serial收集器:单线程默认新生代收集器,运行的时候需要暂停所有用户线程
2. ParNew新生代并行多核收集器,多线程的serial收集器,多核下效率会好点
3. Parallel Scavenge:新生代并行多核收集器,控制垃圾收集的吞吐量
4. SerialOld单线程老年代收集器.
5. parallel old并行老年代收集器
6. CMS(并发标记清理):a)初始标记(暂停所有线程):标记GC ROOT;b)并发标记:GC ROOT Tracing,不阻塞线程,c)重新标记则是为了修正并发标记期间因用户程序继续运作导致的变动(阻塞所有线程),d)并发清除,不阻塞线程,会产生"浮动垃圾"(在标记之后产生的垃圾本次无法清理只有留待下次),需要serial的兜底方案,因此性能可能会很差
7. G1:初始标记,跟CMS差不多,并发标记:找出存活对象,最终标记在初始比较之后导致变化的记录log合并到一个set,需要停止线程但是可以并行,筛选回收是对内存的回收价值和成本进行排序来制定合适的回收计划
####内存分配回收策略

1. 对象优先在Eden分配:新生代GC特别快也特别频繁,老年代一般慢10倍以上
2. 大对象直接进入老年代
3. 长期存活的对象(熬过多次新生代垃圾回收的,一般15次)进入老年代
4. 动态判定年代,如果相同年龄的对象大小占suivivor的一般.年龄大于等于他的就自动进入老年代
5. 检查新生代的所有对象是否能够完全移到老年代,如果够就是安全的,否则可能要进行垃圾回收

##虚拟机类记载机制
把描述类的数据从Class文件加载到内存,并对数据进行校验\转换解析和初始化,最终形成可以被虚拟机直接使用的Java类型

###类加载的时机

加载->验证->准备->解析->初始化->使用->卸载

立即对类初始化的五中情况
1. 遇到new,getstatic,putstatic或invokestatic这四条指令,如果没有初始化则需要先触发其初始化
2. 使用reflect对类进行反射调用的时候
3. 初始化一个类发现父类还未初始化
4. 虚拟机启动时需要制定一个要执行的主类
5. invoke.MethodHandle实例最后解析出来的结果为几个static句柄的时候,并且对应的类没有初始化
被动引用的几个例子(不初始化)
1. 通过子类引用父类的静态字段,不会导致子类初始化
2. 通过数组来定义引用类,不会触发此类的初始化
3. 常量在编译阶段会存入调用类的常量池中,本质上没有直接引用定义常量的类

###类的加载过程
加载完成三件事
1. 通过一个类的全限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的java.lang.Class对象

验证:确保Class文件的字节流中包含的信息符合当前虚拟机的要求,并且不会危害虚拟机自身的安全.主要有四个检验动作
1. 文件格式验证,是否符合Class文件的规范.并且能被当前版本的虚拟机理解
2. 元数据验证:对字节码描述的信息进行语义分析,保证符合Java规范要求
3. 字节码验证:通过数据流和控制流分析,确定语义合乎逻辑
4. 符号引用验证:引用的符号能够被正确解析

准备:正式为类变量分类内存并设置初始值的阶段,这些变量(static)使用的内存将在方法区中分配,正常情况下默认是给0或者null值,只有final修饰的会赋值

解析:将常量池内的符号引用替换为直接引用的过程,符号引用和直接引用

初始化(跟构造函数不太一样):收集所有类变量的赋值动作和静态语句快中的语句合并而成,保证父类的优先于子类的

###类加载器

虚拟机通过把类加载阶段中"通过全限定名称来获取描述此类的二进制字节流"这个动作放到Java虚拟机外部实现,以便让应用程序自己决定如何获取所需要的类 只要类加载器不是同一个,即使是同一个类也会被instanceof判定为不一致

####双亲委派模型

虚拟机有两种类加载器
1. 启动类加载器,由C++实现,是虚拟机的一部分
2. java语言自己实现,继承自抽象类java.lang.ClassLoader

如果一个类加载器收到了类加载的请求,它首先不会自己尝试去加载这个类,而是委派给父类去完成,一层一层的委派,只有父类无法加载,子类才会尝试


破坏双亲委派模型,热替换

##Java内存模型与线程
计算机的运算能力相对于磁盘IO,网络通信和数据库访问是很快的,多线程能够压榨处理器运算能力
局部变量和方法参数是线程私有的
###Java内存模型
Java内存模型规定所有变量存储在主内存之中,每条线程还有自己的工作内存,工作内存中保存了该线程使用到的变量的主内存副本拷贝,线程对变量的所有操作都必须在工作内存中进行,不同线程也无法直接访问对象的工作内存中,变量传递通过主内存进行,主内存类似Java堆,工作内存类似虚拟机栈
###内存间交互操作
主内存和工作内存之间通过八种原子操作来完成
1. Lock一个线程独享一个变量,作用于主内存
2. unlock 一个线程释放一个变量,作用于主内存
3. read:从主内存读到工作内存,作用于主内存
4. load 作用于工作内存的变量,把read操作读取的值放入工作内存中的变量副本
5. use:作用工作内存,把工作内存中的值传递给执行引擎,当虚拟机需要读取数据时候使用
6. assign:作用于工作内存,把从执行引擎收到的值放到工作内存
7. store,作用于工作内存,把工作内存中的一个变量传给主内存,供write使用
8. write:作用于主内存,把store的值放到主内存

###volatile变量的特殊规则
volatile变量每次读取之间会刷新自己,从主内存中读取.因此他在读取的那时应该是没有问题的,但是后面的一致性就无法保证了.(load,use),(assign,stor)连续出现,其他的没有这种硬性要求
volatile禁止指令重排优化,为了更快的实现读写内存而将指令不按照顺序执行,在多线程环境下可能会导致问题,线程之间

###long和double的特殊规则
这两个操作是64位的,允许分两次执行8种原子操作(原子操作是32位的),但是一般虚拟都会实现64位原子操作

###原子性,可见性和有序性

原子性:8种基本操作,更高层次的通过synchronizaed实现
可见性:当一个线程修改了共享变量的值,其他的能够立即得知
有序性:线程内的操作都是有序的,线程外的不好说

###先行发生原则

内存模型保障的先后顺序关系
1. 程序次序原则:同一个线程内的按照代码顺序执行
2. 管道锁定规则:unlock现行发生于对同一个对象的lock操作
3. volatile原则:对volatile的写先于读
4. 线程启动规则:Thread的start方法现行于此线程的所有其他操作
5. 线程终止规则:线程中的所有操作先行于终止检测
6. 线程中断原则:对线程中断方法的调用先行检测到中断事件的发生
7. 对象终结规则:构造函数执行结束先于finalize方法
8. 传递性:A->B B->A

时间先后顺序和先行发生原则没有太大关系,一切以先行原则为准

##Java线程
实现线程 的三个方式:
1. 内核线程:操作系统内核支持的线程,轻量级进程(LWP)和内核线程1:1关系,依靠系统调用,代价较高,用户态内核态来回切换,LWP有限
2. 用户线程:完全建立在用户空间上的线程,线程操作完全在用户态中进行,1:N关系.不需要系统内核支援,处理起来异常复杂
3. 用户线程+LWP混合:一些用户线程对应一个LWP,可以利用内核调度功能,N:M关系

###Java线程调度
1. 协同线程调度:自己执行完了通知系统进行切换,比较简单,时间不可控制,风险高
2. 系统主动切换线程,系统分配执行时间

因为最终是映射到操作系统提供的线程的,因此线程优先级最后还是要看系统

###状态转换
一个线程有且只有5种状态的一种
1. 新建,创建后尚未启动
2. 运行:可能正在执行,也可能正在等待cpu分配执行时间
3. 无限期等待:不会被CPU分配执行时间,只能等待其他线程唤醒
4. 限期等待:不会被CPU分配执行时间,一定时间后由系统自动唤醒
5. 阻塞:等待获取一个排它锁,进入同步区域
6. 结束:线程终止

##线程安全与锁优化
###不可变
不可变对象(final,没有指针逃逸),多线程下永远不会出现不一致的情况

###绝对线程安全
多个线程访问一个对象,如果不用考虑这些线程在运行时的调度和交替执行,不需要额外同步,或者其他的协调操作,调用这个对象的行为都可以获得正确结果

###相对线程安全
大部分线程安全的类Vector,hashTable,synchronizationcollections

###线程兼容
本身不是线程安全的,但是通过同步手段可以安全的使用,绝大部分类属于这种情况

###线程对立
无论如何也不安全,极少,resume和suspend

###线程安全的实现方法
####互斥同步(阻塞同步)
1. synchronized,给对象的锁数目+1,释放-1,只有为0的时候才能被前台线程获取,否则阻塞线程,对于同一个线程可重入,重量级操作,可能会引起用户态核心态的切换
2. ReentrantLock:1)如果长时间等待,可以选择放弃等待干其他事情;2)按照申请时间来依次获得锁3)可以绑定多个条件(三个都是synchronized没有或不方便实现的)

后面改进了synchronized.已经没有太大的性能差别了
####非阻塞同步

通过比较这一次操作的值状态是否为最新,来判断方法是否应该执行,不是最新会重复获取值
1:cuurent=get() next = current+1
2:compareAndSet(current, next) 
当current在compareAndSet中发现被改变那些执行失败,重复1,2否则成功

*ABA问题*

####无同步方案
1. 可重入代码在代码执行的任何时候中断他去执行另外一段代码,回来不会出现任何错误,可重入必然是线程安全的,没有线程共享资源,对于确定的输入有确定的输出
2. 线程本地存储:threadLocal,共享数据限制在同一个线程中执行

###锁优化
####自旋锁和自适应自旋
挂起和恢复线程依赖核心态,自选依赖多核处理器,因为获取释放锁的时间一般很短,因此我们让等待的锁的进程原地等待一下(自旋),如果释放锁很快,那么还好,如果不快会白白浪费时间自旋,后面会根据上一次等待时间来调整自旋时间
####锁消除
如果堆上的数据不会逃逸出去被其他线程访问,那么可以当做栈上数据,无需同步加锁,一般是jvm默认函数的锁消除
####锁粗化
如果虚拟机探测到一串零碎的操作对同一个对象加锁,那么锁的范围会扩大到整个操作序列的外部
####轻量级锁
通过cas(状态是否发生变化,有点像非阻塞)判断有没有竞争,如果没有竞争就改变,有就退化为重量级锁
####偏向锁
一个线程获得偏向锁之后,只要没有其他线程过来获取,那么他可以一直无锁的对数据操作,一旦有人来获取就撤销偏向,变成轻量级锁