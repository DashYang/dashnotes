#Effective Java读书笔记
@(读书笔记)[Java|使用方法]

-------------------
[TOC]

##创建和销毁对象

###考虑用静态工厂方法替代构造器

```java
public static Boolean valueof(boolean b) {
	return b ? Boolean.TRUE : Boolean.FALSE;
}
```

####优点
1. 静态工厂方法有名称(valueof):构造函数名字一样,不便与区分不同的参数列表的构造函数
2.  不必在每次调用的时候创建一个新的对象,上面的方法其实就是在返回现有的对象
3. 可以返回原类型的任意子对象
4. 代码变得更加简洁

####缺点
1. 不含有公有的或者受保护的构造器,就不能被子类化
2.  与其他静态方法没有区别,在API文档中难以查明

相关的应用框架:**基于接口的框架Java Collection Framework**,**服务提供者框架(JDBC)**

###遇到多个构造器参数时要考虑用构建器

当参数众多并且数目不确定的情况下,使用尽可能多的构造去去覆盖所有情况是不明智的,可以采取JavaBean模式,所谓JavaBean模式即采用无参构造器,调用setter方法来设置每个必要的参数.但是由于创建过程被分配到了几个setter中,类无法通过**检验构造器参数的有效性来保证一致性**.而且JavaBean模式阻止了把类做成不可变的可能.最好是采用**Builder模式**

```java
public class NutritionFacts {
	private final int servingSize;
	private final int servings;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;

	public static class Builder {
		// Required parameters
		private final int servingSize;
		private final int servings;

		// Optional parameters - initialized to default values
		private int calories = 0;
		private int fat = 0;
		private int carbohydrate = 0;
		private int sodium = 0;

		public Builder(int servingSize, int servings) {
			this.servingSize = servingSize;
			this.servings = servings;
		}

		public Builder calories(int val) {
			calories = val;
			return this;
		}

		public Builder fat(int val) {
			fat = val;
			return this;
		}

		public Builder carbohydrate(int val) {
			carbohydrate = val;
			return this;
		}

		public Builder sodium(int val) {
			sodium = val;
			return this;
		}

		public NutritionFacts build() {
			return new NutritionFacts(this);
		}
	}

	private NutritionFacts(Builder builder) {
		servingSize = builder.servingSize;
		servings = builder.servings;
		calories = builder.calories;
		fat = builder.fat;
		sodium = builder.sodium;
		carbohydrate = builder.carbohydrate;
	}

	public static void main(String[] args) {
		NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).calories(100).sodium(35).carbohydrate(27).build();
	}
}
```
**Builder模式**的优点:
1. 可以对参数强加约束条件
2. 参数可变灵活,在构造的时候可以干很多事情

Java中传统的抽象工厂是实现Class对象.用newInstance充当build方法的一部分,newInstance总是试图调用无参构造器,而有些类可能根本就没有,编译的时候也检查不出来,需要自行捕获相应的异常.破坏的编译时的异常检查

**Builder模式的缺点**: 
1. 创建构建器的开销在注重性能的情况下可能会有影响
2. Builder模式比重叠构造器更加冗长

Tips:如果以后会添加参数的类最好一开始就采用Builder模式,不然构造器和静态工厂的方法后期会看起来很奇怪,当一个类的参数过多并且可选的情况下,最好一开始就采用builder模式

###用私有构造器或者枚举类型强化Singleton
Singleton通常表示那些本质上唯一的类,这种通常会使得调试变得异常艰难.以前两种方式来创建,这两种方法都是采用私有的构造器
1. 公有静态成员是个final域（变量申明）： 享有特权的客户端可以通过反射机制的AccessibleObject.setAccessible的方法来调用私有构造类，不过可以通过修改构造器使得调用第二次的时候抛出异常
2. 公有成员是个静态工厂方法（函数生命）：静态调用内联化，static确保唯一，灵活。如果改为可序列化对象，单纯implements Serializable是不够的，比如声明所有实例都是瞬时的
3. 包含一个单个元素的枚举类型：枚举其实就是一个类，枚举的成员就是类的静态成员。本身对于序列化和反序列化的支持就比较好。枚举类型绝对防止多次实例化，即使反射攻击和复杂的序列化也没有问题

```java
//使用枚举的单例模式
public class EnumSingleton{
    private EnumSingleton(){}
    public static EnumSingleton getInstance(){
        return Singleton.INSTANCE.getInstance();
    }
    
    private static enum Singleton{
        INSTANCE;
        
        private EnumSingleton singleton;
        //JVM会保证此方法绝对只调用一次
        private Singleton(){
            singleton = new EnumSingleton();
        }
        public EnumSingleton getInstance(){
            return singleton;
        }
    }
}
```

###通过私有的构造器强化不可实例化的能力
通常来说工具类不希望被实例化，因此构造器其实是不必要的，但是在不显示指定构造器的情况下会生成缺省的无参构造器，因此我们只要让这个类包含私有构造器，他就不能被实例化了。但是缺点是无法被子类化，因为子类必须调用构造器，但父类的构造器是私有的。

###避免创建不必要的对象
1. 最好重用对象而不是新建对象，以免引起不必要的性能开销，比如避免new的使用。此外，尽量使用静态工厂方法而不是构造器，避免多次创建对象。
2. 优先使用基本类型（long）而不是装箱基本类型（Long），避免无意识的自动装箱。但是小对象的创建和回收是比较廉价的
3. 对象池一般是创建大型对象（数据库连接）。

创建对象和避免创建对象通常是基于安全和性能的折衷，应该具体问题具体分析

###消除过期的对象引用

1. 只要是自己管理的内存，就需要小心内存泄露的问题，一旦元素被释放，则该元素包含的任何引用都应该被清空
2. 内存泄露的另一个常见是缓存，一旦把对象放入缓存（weakhashmap），就很有可能被遗忘。可以设置定时清理或者每次查询等操作的时候检查一下（有点像redis）
3. 监听器和其他回调：注册了回调但是没有显示的取消注册，最好是保存弱引用

Heap profiler和代码检测是最好的

补充资料，四种引用
⑴强引用（StrongReference）
强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。
 
⑵软引用（SoftReference）
如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存（下文给出示例）。
软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。
 
⑶弱引用（WeakReference）
弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。
 
⑷虚引用（PhantomReference）
“虚引用”顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。
虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。

###避免使用终结方法
终结方法和C++的析构函数并不一样，他并不保证及时的执行，从变得不可到达到执行终结方法的时间是不确定的，java的语言规范并不保证终结方法会被及时的执行，也不保证会执行，依赖终结方法来释放分布式系统中的永久锁，很容易让分布式系统垮掉。终结方法有严重的性能损失。
因此，一般不建议使用终结方法（finalize），一般显示的调用close等自带的终止方法

使用几种场景
1. 安全网：跟try finally结合使用，尽可能的回收资源，尽管无法得到保证，报出bug
2. 终止非关键的本地资源（JNI创建对象，socket和文件的补充）

子类被终结父类需要显示终结否则会报错，因此需要添加一个守卫者（一个私有类去覆盖finalize）

##对于所有对象都通用的方法（通用约定）

###equal
如果类具有自己特定的“逻辑相等”概念，并且超类还没有覆盖equal以实现期望的行为，具有逻辑相等的概念，通常枚举类型不需要equal，因为他是唯一的。需要满足四个特性：自反传递对称一致非空

高质量的方法
1. 使用==来判断是否为对象的一个引用
2. 使用instanceof来判断是否为正确类型
3. 参数换成正确的类型
4. 对于该类中每个关键域，是否于该对象对应的域相匹配
5. 覆盖euqal时总要覆盖hashcode
6. 不要企图让equal方法过于智能（会增加不必要的麻烦）
7. 不要将equal声明中的object替换为其他的类型（无法覆盖函数）

###覆盖equal时总要覆盖hashcode

如果覆盖equal却没有覆盖hashcode，会知道基于散列的一些类无法正常工作
1. 只要equal用到的信息没有被修改，那么两个类hashcode 也应该一致
2. 只要两个类euqal，那么hashcode一样
3. 两个类不equal不一定hashcode不一致，但是不一致会提高散列性能

###始终覆盖toString
便于调试，更加容易看懂

###谨慎的覆盖clone

clone与final是不兼容的，clone最好把class中的逐项拷贝，如果是链表的话要注意浅拷贝问题，最好不要使用

###考虑使用Comparable方法
当对象小于等于或者大于制定对象的时候，分别返回一个负整数、零或者正整数。如果由于无法比较则跑出ClassCastException，跟equal一样需要考虑自反性，传递性和对称性

##类和接口

###使类和成员的可访问性最小化
类尽量隐藏自己的信息，只提供必要的借口供外部调用，从而使得系统解耦。**访问控制**机制决定了类、接口、成员的可访问性，实体的可访问性通过修饰符实现（private、default,protect、public，一般不添加任何修饰符就是default）

| 同      |    同一个类  | 同一个包  | 不同包的子类| 不同包的非子类|
| :-------- | --------:| :--:    |  ----     |  ----------|
| private   |     O    |         |           |            |
| default   |     O    |  O      |           |            |
| protected |     O    |  O      |   O       |            |
| public    |     O    |  O      |   O       |    O       |

有几个原则如下
1. 尽可能的使每个类或者成员不被外界访问，给予尽可能小的访问权限，如果这个类实现了serializable接口，就有可能泄露，覆盖超类的方法必须要访问权限不低于超类（比如超类为public，子类不能为private，不然无法访问，会编译出错），接口中的方法必须为公有的
2.  实例域(非final)决不能是公有的

###公有类中使用访问方法而非公有域

通过函数访问可以控制，而域一旦公有出去就很难再控制了，暴露不可变的域危害较小

###使可变性最小化
不可变类需要遵循五条规则
1. 不要提供任何可以修改对象状态的方法
2. 保证类不会被扩展，用final修饰类
3. 使所有的域都是final的
4. 确保对于任何可变组件的互斥访问
5. 不可变对象本质上是线程安全的

###复合优先于继承

跨越包边界的继承是很危险的，因为包内可能是同一个人控制下的，继承本身就是打破封装性的手段。
超类的很多实现细节是对子类隐藏的，覆盖的方法可能调用了其他方法，而其他方法可能又会被覆盖等产生不可控行为，本质上在于超类在子类的实现是不可控的，超类的具体逻辑也是不可控的，最后产生的结果自然也是不可控的

因此最好在原来的基础上包装一个类，但是包装类不适合回调框架，因为回调框架需要把自身传给别人，可能他自己并不知道自己被包装类覆盖，从而导致奇怪的事情发生。继承还会导致超类的缺陷传播到子类中

###要么为继承设计并提供文档，要么就禁止

对于可覆盖的方法应当给出详细的调用行为描述，比如调用了那些可覆盖方法以及调用顺序等等，或者确保可以被覆盖的方法永远不会调用其他可以被覆盖的方法。


###接口优于抽象类

1. 现有的类很容易被更新从而实现新的接口，类的继承要复杂很多，可能会牵扯到很多部相关的继承链中的类
2. 接口是定义混合类型的理想选择，类可以实现多个借口
3. 接口允许构造非层次结构的类型框架

**抽象骨架类**是指实现了一个或多个接口的抽象类，对于特定的方法进行实现

###接口只用于定义类型

在接口的内部使用常量，这是不合适的，如果接口的值发生改变，类不在需要使用，依然要通过类来获取，污染类。最好是放到类是中使用常量，如果想避免出现类名，可以使用静态导入（static import）

###类层次优于标签类

如果一个类有多种标签，最好把共性提取出来作为超类，不同的子类以各自的方式实现他。（标签类指一个类中有多套比较独立的对象和函数）

###用函数表示策略
首先声明一个接口表示策略，并为该策略声明一个实现该接口的类，如果只用一次，可以通过匿名类来实例化，如果要被重复使用则应当声明为私有的静态成员类，通过公有的静态final域导出

###优先考虑静态成员类

嵌套类有四种：静态成员类、非静态成员类、匿名类和局部类


###泛型 

###请不要在新代码中使用原生态类型
泛型中尽量制定相应的类型而不应该缺省

###消除非受检警告
尽可能在保持代码规范，消除类型不匹配等问题导致额非受检警告

###列表优先于数组
列表相对于数组有类型检查，通过相应的接口来访问对象更加安全

###优先考虑泛型类和泛型方法


####枚举和注解
1. enum代替int常量
2. 实例域代替序数（枚举的序数导出与他关联的值）
3. 用EnumSet代替位域
4. 用Enum代替序数索引
5. 坚持使用override注解


（一大部分没营养的东西）

6. 尽量使用基本类型替代装箱类型（装箱拆箱有问题）
7. 避免使用字符串
8. 当心字符串的连接（+）性能，Stringbuilder代替String +会创建新的Stringbuilder
9. 接口优于类和反射机制

###并发编程的哲学（注意事项）

1. 同步访问共享可变的数据
2. 避免过度同步：避免在同步区域内调用外来不可控的方法
3. executor和task优先于线程
4. 并发工具优于wait和notify
5. 慎用延迟初始化
6. 不要依赖于线程调度器
7. 避免使线程组

###序列化
序列化和反序列化指的是将java对象转换成对应的字节数据,需要调用writeObject方法,除了默认三种类型(string,array, enum),还有实现了Serializable接口的类,其中被static和transient修饰的变量无法被序列化

1. 最好使用自定义的序列化方式
2. 保护性的编写readObject方法
3. 如果singleton实现了serializable,那么他就不再是singleton,因为每次readObject都是新的,所以要readresolve,返回一个唯一引用
4. 考虑序列化代理代替序列化实例,在需要序列化的类中添加一个私有的静态嵌套类



