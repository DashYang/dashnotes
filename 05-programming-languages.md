# Rust、Go 与 Solidity

> 核验日期：2026-07-17
>
> 示例强调语言语义，不绑定某个框架小版本。EIP/ERC 状态以链接页面为准。

## 主题简介：What / Why / How

- **What**：Rust、Go、Solidity 分别代表编译期内存安全的系统语言、带 GC 与轻量并发运行时的服务端语言、确定性且 gas 受限的链上合约语言。
- **Why**：区块链系统同时需要高性能客户端、并发后端和可公开验证的合约；三类环境的故障成本和资源模型不同。
- **How**：Rust 用 ownership/borrow/type system 限制内存错误，Go 用 goroutine/channel/runtime 简化并发服务，Solidity 把程序编译为 EVM bytecode 并通过 storage layout、ABI 和交易原子性约束执行。

面试不要只列语法。一个好的回答结构是：**语言机制 → 保证了什么 → 没保证什么 → 典型故障 → 如何验证**。

## 1. Rust

### 1.1 Ownership、Move 与 Drop

每个值有一个 owner；owner 离开作用域时运行 `Drop`。把非 `Copy` 值赋给另一个变量或传值参数通常发生 move，原绑定不可再使用。`Copy` 只适用于位复制语义安全的简单类型，不应为拥有资源的类型随意实现。

```rust
fn consume(s: String) -> usize {
    s.len() // 函数结束时释放 String 的堆内存
}

let a = String::from("block");
let n = consume(a);
// println!("{a}"); // 编译错误：a 已 move
```

Move 是编译期所有权转移，通常不等于复制堆数据。返回 ownership、借用或 clone 分别代表不同成本和 API 语义。

### 1.2 Borrow 与 Lifetime

借用规则可概括为：同一时刻可以有多个共享引用 `&T`，或一个独占引用 `&mut T`，不能同时存在冲突访问。编译器通过 lifetime 证明引用不会比被引用值活得久。

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() >= y.len() { x } else { y }
}
```

`'a` 不会延长对象生命周期，它只描述输入与输出引用之间的关系。Non-Lexical Lifetimes 会在引用最后一次使用后结束 borrow，而不必等到词法作用域末尾。

常见陷阱：返回局部变量引用、在持有元素引用时修改 `Vec`（扩容可能移动内存）、过度用 `'static` 掩盖所有权设计问题。

### 1.3 Trait、Generic 与 Associated Type

先把三者看成不同问题的答案：

- **Trait**：值“会做什么”——定义能力契约。
- **Generic**：函数/类型“可以接收什么不同类型”——用类型占位符消除重复。
- **Associated Type**：某个实现“固定关联什么类型”——把实现者自己决定的类型挂在 Trait 上。

它们常一起出现，但不是同义词。`T: Trait` 的完整含义是：`T` 是一个未知的具体类型，编译器只接受那些实现了该 Trait 的具体类型。

#### Trait：先定义能力，再让类型实现它

Trait 类似接口，但它还能提供默认方法、关联类型和关联常量。下面的 `Summary` 不关心文章或评论的字段，只要求“能生成摘要”：

```rust
trait Summary {
    fn author(&self) -> &str;           // implementor 必须提供

    fn summarize(&self) -> String {     // 默认实现，可被覆盖
        format!("{} wrote something", self.author())
    }
}

struct Article { author: String, title: String }
struct Comment { author: String, body: String }

impl Summary for Article {
    fn author(&self) -> &str { &self.author }
    fn summarize(&self) -> String { format!("article: {}", self.title) }
}

impl Summary for Comment {
    fn author(&self) -> &str { &self.author }
}
```

`Article` 与 `Comment` 的内存布局可以完全不同；只要都实现 `Summary`，调用方就可依赖同一组方法。Trait 解决的是**行为抽象**，不会自动创建对象、保存状态或提供运行时多态。

实现 Trait 要遵守 orphan rule：通常只能为“本 crate 定义的 Trait”或“本 crate 定义的类型”写 `impl`。这避免两个依赖为同一外部类型提供相互冲突的实现。

#### Generic + Trait Bound：复用算法，同时保留具体类型

Generic 用 `T`、`I` 等名字表示“编译时尚未确定的具体类型”。如果函数只搬运值，不需要行为，就写 `T`；若要调用比较、格式化、迭代等行为，就加 Trait Bound：

```rust
fn largest<T: Ord>(items: &[T]) -> &T {
    let mut best = &items[0];
    for item in &items[1..] {
        if item > best {
            best = item;
        }
    }
    best
}

fn announce<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}
```

这里 `T` 不是“任意类型”：`largest` 只接受实现 `Ord` 的类型，因为代码需要 `>`；`announce` 只接受实现 `Summary` 的类型，因为代码调用 `summarize`。同一个函数被 `i32`、`String` 或用户自定义类型调用时，编译器会为实际类型生成对应代码（monomorphization），因此通常是**静态分发**：无 vtable 间接调用，便于优化和内联，但可能增加二进制体积与编译时间。

常见等价写法如下；复杂约束用 `where` 更易读：

```rust
fn notify(item: &impl Summary) { // 单个输入位置时等价于 <T: Summary>
    println!("{}", item.summarize());
}

fn compare_and_notify<A, B>(a: &A, b: &B)
where
    A: Summary + std::fmt::Debug,
    B: Summary,
{
    println!("left={a:?}; right={}", b.summarize());
}
```

`impl Trait` 在**参数位置**相当于匿名泛型；`fn f(a: &impl Summary, b: &impl Summary)` 允许 `a`、`b` 是不同实现类型。若两者必须是同一具体类型，应写 `fn f<T: Summary>(a: &T, b: &T)`。返回位置的 `impl Trait` 则是“函数选择一个具体但对调用者隐藏的返回类型”，它与 `Box<dyn Trait>` 的运行时异构返回不同。

#### Associated Type：由实现者固定的输出/元素类型

关联类型写在 Trait 内：`type Item;`。每个 `impl Trait for SomeType` 必须为它指定一个类型，因此调用者可写 `T::Item` 或 `<T as Trait>::Item`。

`Iterator` 是最重要的例子：一个迭代器实现固定产生一种元素类型。

```rust
fn sum<I>(items: I) -> i64
where
    I: IntoIterator<Item = i64>,
{
    items.into_iter().sum()
}
```

逐段读这段签名：

| 片段 | 含义 |
| --- | --- |
| `I` | 调用方传入的容器/迭代器具体类型，尚未确定 |
| `I: IntoIterator` | 它必须能被转换为迭代器 |
| `Item = i64` | `IntoIterator` 的关联类型 `Item` 必须恰好是 `i64` |
| `items.into_iter()` | 取得该实现指定的迭代器 |

如果不用关联类型，而写成 `trait MyIterator<T> { fn next(&mut self) -> Option<T>; }`，同一个 `Counter` 可以分别实现 `MyIterator<u8>` 与 `MyIterator<String>`。这在“同一类型确实要提供多种输入/输出关系”时有用，但调用方常需要额外写类型标注来消除歧义。

```rust
trait Stream {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter(u8);

impl Stream for Counter {
    type Item = u8; // Counter 实现 Stream 时，元素类型固定为 u8
    fn next(&mut self) -> Option<Self::Item> {
        self.0 = self.0.checked_add(1)?;
        Some(self.0)
    }
}
```

所以选择原则不是“Associated Type 更新”，而是问：**对同一个实现，关联类型是否应该唯一？**

| 需求 | 选择 | 例子 |
| --- | --- | --- |
| 每个实现固定一种输出/元素类型 | Associated Type | `Iterator::Item`、`Add::Output` |
| 同一实现可和多种类型形成不同关系 | Trait Generic Parameter | `From<T>`、`Add<Rhs = Self>` 的 `Rhs` |
| 调用者要自由指定算法处理的数据类型 | Function/Struct Generic | `Vec<T>`、`fn largest<T: Ord>` |

`std::ops::Add<Rhs = Self>` 是混合示例：`Rhs` 是可变化的输入泛型参数，`Output` 是该 `Self + Rhs` 实现确定的关联类型。因此 `u32 + u32 -> u32` 与 `Duration + Duration -> Duration` 可以各自定义输出类型，同时调用方不必再额外指定 `Output`。

#### Static Dispatch 与 `dyn Trait`：按集合是否异构选择

| 方式 | 示例 | 编译器何时知道具体类型 | 适用场景 | 代价 |
| --- | --- | --- | --- | --- |
| Generic / 静态分发 | `fn draw<T: Draw>(x: &T)` | 编译时 | 热路径、同质集合、需要内联 | 代码可能为多种 `T` 重复生成 |
| Trait Object / 动态分发 | `Vec<Box<dyn Draw>>` | 运行时 | 插件、异构集合、运行时选择实现 | vtable 间接调用、堆分配/生命周期处理常更复杂 |

`dyn Trait` 是 trait object，通常放在 `&dyn Trait`、`Box<dyn Trait>`、`Arc<dyn Trait + Send + Sync>` 后，因为它的具体大小在编译期未知。它不是“更高级的泛型”，而是运行时多态。只有 **dyn-compatible** 的 Trait 才能这样使用；例如带泛型方法、返回 `Self` 的方法或某些关联项的 Trait 不能直接做 trait object。

#### 面试答题骨架

**问：`T: Trait`、`impl Trait` 和 `dyn Trait` 有何区别？**

答题骨架：前两者通常是编译期泛型/静态分发，调用时具体类型确定；`dyn Trait` 抹去具体类型，在指针后通过 vtable 动态分发，换取异构集合和运行时选择。再补充 `impl Trait` 参数位置允许不同实参类型，而同一个 `T` 可强制多个参数同型。

**问：为什么 `Iterator` 用 `type Item`，而 `From<T>` 用泛型参数？**

答题骨架：一个迭代器实现应固定产出类型，因此用 associated type 减少调用方歧义；同一目标类型可从多个源类型转换，`T` 必须由每个转换关系变化，所以是泛型参数。

### 1.4 智能指针与内部可变性

| 类型 | 所有权/线程 | 可变性策略 | 典型用途 |
| --- | --- | --- | --- |
| `Box<T>` | 单 owner | 普通 borrow | 堆分配、递归类型、trait object |
| `Rc<T>` | 单线程引用计数 | 常配 `RefCell` | 单线程共享图 |
| `Arc<T>` | 跨线程原子计数 | 常配 `Mutex/RwLock` | 多线程共享状态 |
| `RefCell<T>` | 单线程 | 运行时 borrow check | 编译期难表达的内部可变性 |
| `Mutex<T>` | 跨线程排他 | lock guard | 写频繁或复合不变量 |
| `RwLock<T>` | 多读单写 | read/write guard | 读多写少且临界区适中 |

`Arc<T>` 只让引用计数线程安全，不自动让 `T` 可并发修改。`Rc<RefCell<T>>` 不是线程安全组合；引用环会泄漏，父指针等反向边常用 `Weak`。

### 1.5 `Send` / `Sync`

- `T: Send`：`T` 的 ownership 可以安全跨线程转移。
- `T: Sync`：`&T` 可以安全跨线程共享；近似地，`T: Sync` 意味着 `&T: Send`。

它们是 unsafe auto traits，多数类型自动推导。原始指针、`Rc`、`RefCell` 等会阻止相应推导。手写 `unsafe impl Send/Sync` 相当于向编译器承诺全部内部同步正确，必须说明不变量。

### 1.6 Future、Pin、Waker 与 Tokio

#### Future：可暂停和恢复的惰性状态机

`Future` 表示一项可能尚未完成的异步计算。它不是 OS thread，也不是创建后就自动在后台执行的任务；调用 `async fn` 通常只是构造一个匿名 Future，只有 `.await`、`spawn` 或 `block_on` 驱动它时，计算才会向前推进。

```rust
pub trait Future {
    type Output;

    fn poll(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output>;
}
```

executor 调用 `poll` 尝试推进 Future，但 `poll` 必须快速返回，不能原地阻塞：

- `Poll::Ready(value)`：计算已经结束，产出最终值；完成后的 Future 不应再被 poll。
- `Poll::Pending`：当前条件不足，Future 保存执行状态并让出线程；在可能继续前应注册当前任务的 `Waker`。

`.await` 可粗略理解为轮询一个子 Future：子 Future `Ready` 时从下一行继续，`Pending` 时外层 Future 也保存局部变量和执行位置并返回 `Pending`。因此 `.await` 暂停的是 async task，不是 executor 所在线程；该线程可以转去运行其他任务。给普通阻塞调用加 `async` 不会让它自动非阻塞。

```rust
async fn fetch(stream: &mut tokio::net::TcpStream) -> std::io::Result<Vec<u8>> {
    use tokio::io::AsyncReadExt;

    let mut buf = vec![0; 1024];
    let n = stream.read(&mut buf).await?;
    buf.truncate(n);
    Ok(buf)
}
```

这段代码编译后可概念化为 `Start → WaitingForRead → Done` 状态机。暂停在 `read().await` 时，状态机需要保留 `buf`、子 Future 和恢复位置；读取完成后再次 poll，才从 `await` 后继续。具体生成布局是编译器实现细节，不应依赖。

#### Waker：通知 executor“现在可能可以继续”

Future 返回 `Pending` 后，executor 不应在空循环中持续 poll。Future 从 `Context` 取得当前任务的 `Waker`，交给 I/O driver、timer 或其他生产结果的一方保存；条件变化时，对方调用 `wake()`：

```text
executor poll Future
        │
        ├─ 已完成 ─────────────────────→ Ready(value)
        │
        └─ 暂时不能继续
             │
             ├─ 注册当前任务的 Waker
             └─ Pending，线程运行其他任务
                         │
                  I/O / timer 就绪
                         │
                      wake()
                         │
                  任务进入可运行队列
                         │
                     再次 poll
```

`wake()` 通常只是把 task 标记为 runnable 并放回调度队列，不等于立即执行，也不保证下一次 poll 一定得到 `Ready`：事件可能合并、条件可能又发生变化，或资源已被其他任务竞争走。Future 被多次 poll 时，应更新为最近一次 `Context` 中的 Waker；返回 `Pending` 却没有安排未来唤醒，会造成任务永久沉睡。

#### Pin / Unpin：保证地址敏感状态机的安全

Rust 的 move 可能改变值的内存地址。编译器生成的 Future 可能在同一个状态机中保存跨 `.await` 的局部变量和对这些变量的借用，概念上形成 self-referential（自引用）关系：

```text
Future state machine
├── buf
└── read_future ──借用──→ buf
```

若进入这种地址敏感状态后仍把整个 Future 移到新地址，内部引用可能失效。因此 `Future::poll` 接收 `Pin<&mut Self>`：对 `!Unpin` 的 Future，它承诺从被 pin 起到底层值销毁前，安全代码不能把该值移出当前位置。

```rust
let future = fetch(&mut stream);
let pinned: std::pin::Pin<Box<_>> = Box::pin(future);
```

`Pin<Ptr>` 固定的是 `Ptr` 指向的值，不是指针对象本身。`Pin<Box<T>>` 的 Box 句柄仍可移动，但安全代码不能把地址敏感的 `T` 从该分配中移动出来。`Pin` 也不意味着：

- 必须堆分配：值也可以用 `pin!` 固定在栈上。
- 内容 immutable：它限制的是破坏地址稳定性的移动，不禁止所有修改。
- 所有类型都受限制：实现 `Unpin` 的类型明确表示“不依赖固定地址”，即使放进 `Pin` 也仍可安全移动。

绝大多数普通类型会自动实现 `Unpin`；`!Unpin` 才是 Pin 约束真正发挥作用的场景。不要把 `Unpin` 理解成“当前没有被 pin”，它描述的是类型是否需要地址稳定保证。

#### Tokio：把 Future、事件源和调度器连接起来

标准库定义 `Future`、`Poll`、`Context`、`Waker` 和 `Pin`，但不提供完整的网络异步 runtime。Tokio runtime 主要包含：

- scheduler/executor：管理 task 和 runnable queue，在 worker thread 上 poll Future。
- I/O driver（常称 reactor）：监听 socket readiness，并唤醒对应 task。
- timer driver：驱动 `sleep`、`timeout`、`interval` 等计时 Future。
- task 与异步原语：`spawn`、`JoinHandle`、异步 channel、semaphore、`Mutex` 等。

一次 Tokio 网络读取的完整链路是：

1. 调用 `async fn` 构造 Future，Tokio 将其包装成 task。
2. scheduler 第一次 poll；代码运行到 `read().await`。
3. socket 暂不可读，read Future 向 I/O driver 注册兴趣和 Waker，返回 `Pending`。
4. worker thread 转去运行其他 task。
5. OS 报告 socket 可读，I/O driver 调用 Waker。
6. scheduler 将 task 放回 runnable queue，再次 poll。
7. read Future 返回 `Ready(n)`，外层 Future 从 `.await` 后继续，最终返回 `Ready(result)`。

`tokio::spawn` 把 Future 变成可独立调度的 task，并返回可等待的 `JoinHandle`。多线程 runtime 中，被 spawn 的 Future 通常需要 `Send + 'static`：`Send` 允许任务跨 worker thread 移动；这里 `'static` 表示任务不借用可能提前销毁的短命栈数据，不表示任务一定存活到进程结束。常用 `async move` 把所需 ownership 移入任务。

```rust
let name = String::from("Alice");

let handle = tokio::spawn(async move {
    println!("{name}");
});

handle.await?;
```

最终可用一句话串联：**Future 保存执行状态，Pin 保证地址敏感状态机不被移动，Waker 通知任务可能可以继续，Tokio 负责监听事件、调度 task 并再次 poll Future。**

#### 阻塞、锁与公平性风险

```rust
async fn handle(state: std::sync::Arc<tokio::sync::Mutex<State>>) {
    let snapshot = {
        let guard = state.lock().await;
        guard.snapshot()
    }; // guard 在 await 外释放
    send(snapshot).await;
}
```

不要持有 `std::sync::MutexGuard` 跨 `.await`：任务挂起时可能长期占锁、阻塞 executor thread、形成死锁，或使整个 Future 无法满足 `Send`。即使使用异步 Mutex，也应尽量缩小临界区，避免让慢 I/O 占着锁等待。

`poll` 中执行长时间 CPU 计算、`std::thread::sleep` 或传统阻塞 I/O，会独占 worker thread，使同线程上的其他 Future 得不到 poll；同步阻塞工作应使用 `spawn_blocking` 或专用线程池，并设置并发上限。异步任务只有在 `.await` 或主动 yield 等协作点让出执行权，错误地写成长时间不让出的循环也会造成调度饥饿。

### 1.7 Channel 与取消

bounded channel 提供背压；unbounded channel 可能在生产者快于消费者时耗尽内存。多生产者/单消费者、broadcast、watch 和 oneshot 语义不同，应按是否需要每条消息、最新值或单次响应选择。

Future 被 drop 通常表示取消，但外部副作用可能已发生。设计 cancel safety：循环中的 read/write 进度是否丢失、锁是否释放、事务是否提交。超时后对远程写请求属于 unknown outcome，需要幂等键和查询。

### 1.8 错误处理

- `Result<T, E>` 让失败成为类型；`?` 传播并通过 `From` 转换错误。
- `thiserror` 适合 library/domain error，保留可枚举变体。
- `anyhow` 适合 application 边界，便于增加 context，但不应让公共库 API 丢失结构化错误。
- `panic!` 表示不变量被破坏或不可恢复编程错误，不应替代普通输入校验。

错误链要保留 source 和上下文，日志只在有处理决策的边界记录一次，避免每层重复打印。

### 1.9 Unsafe、FFI、布局与性能

`unsafe` 允许解引用 raw pointer、调用 unsafe function、访问 mutable static、实现 unsafe trait 等；它不会关闭 borrow checker，而是把特定不变量交给程序员。把 unsafe 封装在小模块，以安全 API 暴露，并在注释中写明 safety invariant。

FFI 要明确 ABI、alignment、ownership、allocator、panic 边界和 callback 生命周期。`repr(C)` 只稳定字段布局规则，不自动保证对端语言的所有类型等价。

性能诊断先 profile：allocation、clone、lock contention、cache miss、syscall 和 async task 数量。不要为消除一次 clone 引入复杂 lifetime，除非数据证明值得。

### 1.10 性能与内存分析

分析的核心不是先选工具，而是先把问题分成 **CPU 时间、分配速率、存活内存、锁/调度等待、I/O**。推荐流程是：固定输入和并发度，在优化构建下建立基线，采样定位热点，只改一个变量，再用同一负载复测吞吐、P95/P99、CPU 和峰值内存。不要根据一次微基准或一张火焰图直接重写数据结构。

| 问题 | 常用工具 | 重点看什么 |
| --- | --- | --- |
| 函数/算法是否变快 | `cargo bench` + Criterion.rs | `time/iter`、吞吐、置信区间；把数据准备和 I/O 移出计时区 |
| CPU 热点 | Linux `perf`、`cargo flamegraph` | hot stack、调用链；火焰宽度代表采样占比，不代表单次调用延迟 |
| cache miss、分支和系统调用 | `perf stat` / `perf record` | instructions、cycles、cache miss、context switch；先确认瓶颈是否真在用户态代码 |
| 堆峰值和分配来源 | Heaptrack、Valgrind Massif/DHAT | 分配调用栈、峰值、对象 lifetime、临时分配总量 |
| async task 长时间不推进 | `tokio-console` | task 数、poll 时间、busy time、wake 次数和资源等待 |
| `unsafe` / FFI 内存错误 | AddressSanitizer/LeakSanitizer、Valgrind Memcheck | use-after-free、越界、泄漏；它们是正确性检测器，不用于代表正常性能 |

常见命令如下；真实性能用 release/bench profile，保留足够的 symbol/debug info 以便还原调用栈：

```bash
cargo bench
cargo flamegraph --release
perf stat ./target/release/app
perf record -g ./target/release/app
perf report
heaptrack ./target/release/app
valgrind --tool=massif ./target/release/app
```

Rust 没有 tracing GC，但仍会出现“内存持续上涨”：`Rc/Arc` 环、未回收的 task、无界 channel/集合、缓存没有淘汰、`mem::forget`，以及 allocator 保留空闲页。先区分 **live heap** 与进程 **RSS**：live heap 下降而 RSS 不降，可能是 allocator 缓存、内存碎片、线程栈或 `mmap`，不能直接断言业务对象泄漏。两个时点的 allocation profile 对比，通常比只看一次峰值更有解释力。

### 1.11 Rust 高频问答

1. **Move 与浅拷贝区别？** Move 转移可用权，通常不复制资源；原绑定被编译器禁用，避免 double free。
2. **`Arc<Mutex<T>>` 为什么安全？** Arc 管共享 ownership，Mutex 建立互斥和内存同步；两者职责不同。
3. **Lifetime 是运行时对象吗？** 不是，是编译期引用关系；通常被擦除。
4. **Pin 解决什么？** 防止需要稳定地址的值被安全移动；只有针对 `!Unpin` 内容才体现约束。
5. **锁为何不能跨 await？** 挂起期间占锁，阻塞其他任务并可能让 Future 非 Send；缩小临界区或复制快照。
6. **Future 返回 Pending 后如何继续？** 返回前注册当前 task 的 Waker；I/O、timer 或其他事件源在条件变化时调用 `wake()`，executor 把任务放回可运行队列并再次 poll。wake 只表示“可能可继续”，不保证下一次就是 Ready。
7. **Tokio 与 Future 是什么关系？** Future 是标准库定义的惰性异步状态机接口；Tokio 是具体 runtime，提供 scheduler、I/O/timer driver 和 task API 来驱动 Future。async 不等于 Tokio，也不自动产生线程。
8. **Rust 没有 GC，为什么内存仍可能上涨？** 所有权保证资源按规则释放，不保证没有引用环、无界容器、长生命周期 task、allocator retention 或碎片；要对照 live heap、累计分配与 RSS 分层判断。
9. **如何排查 Rust 性能问题？** 先用稳定负载和优化构建设基线，再用 flamegraph/perf 找 CPU 与系统层热点，用 Heaptrack/Massif 看分配和峰值，async 服务再看 tokio-console；修改后用同一基准验证。

## 2. Go

### 2.1 Goroutine 与 G-M-P

goroutine 是 Go runtime 调度的轻量任务。G 表示 goroutine，M 表示 OS thread，P 持有运行 Go 代码所需调度上下文和本地 run queue。scheduler 在 P 间 work stealing，并在 syscall、阻塞、抢占时重新安排 G/M。

goroutine 便宜但不是免费：它有栈、调度和引用对象。无界地“每条消息开一个 goroutine”会导致内存、连接和下游并发爆炸，应使用 semaphore/worker pool 和 context 控制生命周期。

### 2.2 Channel、Select 与关闭

- unbuffered channel 的 send/receive 会 rendezvous，建立同步点。
- buffered channel 只在 buffer 满/空时阻塞；容量不是越大越好，它会隐藏背压。
- close 表示“不会再发送”，应由发送方或拥有发送生命周期的一方执行。
- 从 closed channel 接收立即返回零值且 `ok=false`；向 closed channel 发送会 panic；重复 close 也会 panic。

`select` 在多个 ready case 中伪随机选择。`nil` channel 永久阻塞，可用于动态禁用 case。`default` 会使 select 非阻塞，若放在热循环中可能 busy-spin。

```go
func worker(ctx context.Context, jobs <-chan Job) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case job, ok := <-jobs:
			if !ok {
				return nil
			}
			if err := process(ctx, job); err != nil {
				return err
			}
		}
	}
}
```

### 2.3 Context

`context.Context` 用于在一条请求调用链和它启动的 goroutine 之间传播 **生命周期边界**。它主要传递取消信号、截止时间和少量请求级元数据，让下游知道“这项工作是否还值得继续”。Context 本身不执行业务、不强制杀死 goroutine，也不负责等待 goroutine 结束。

#### 2.3.1 四个主要作用

1. **取消传播（cancellation）**：客户端断开、上游失败或主动调用 `cancel()` 时，下游 DB、HTTP/RPC 调用和工作 goroutine 可以尽快停止，释放连接、CPU 和内存，避免请求已经结束后仍产生幽灵工作。
2. **超时与截止时间（timeout / deadline）**：给整条调用链设置时间预算。下游可通过 `Deadline()` 判断剩余时间，或直接使用支持 Context 的 API；子 Context 的 deadline 不会晚于父 Context。
3. **请求级元数据（request-scoped values）**：传递 trace ID、认证主体等跨 API 边界的少量信息。它不会自动序列化到另一个进程；HTTP/RPC middleware 仍要显式写入和提取 header/metadata。
4. **统一并发任务的生命周期**：多个 goroutine 共享一个 Context 后，可以监听同一个 `Done()` 信号并协作退出。但“通知退出”和“确认已经退出”是两回事；后者仍需 `WaitGroup`、`errgroup` 或结果 channel。

#### 2.3.2 树形传播模型

Context 通常形成一棵树：

```text
request ctx
├── database timeout ctx
├── inventory RPC ctx
└── parallel workers cancel ctx
```

`context.WithCancel`、`WithTimeout`、`WithDeadline` 和 `WithValue` 都从 parent 派生 child：

- parent 取消时，所有后代都会收到取消信号。
- child 取消只影响它自己和它的后代，不会反向取消 parent 或兄弟节点。
- `Done()` 返回一个只读 channel；被关闭后，`Err()` 通常为 `context.Canceled` 或 `context.DeadlineExceeded`。
- 需要记录更具体的失败原因时，可用 `WithCancelCause`，并通过 `context.Cause(ctx)` 读取；普通 `ctx.Err()` 仍主要用于判断取消类别。
- Context 可被多个 goroutine 并发安全地读取和传递。

#### 2.3.3 一个 HTTP → DB 的例子

```go
func loadOrder(ctx context.Context, db *sql.DB, id string) (Order, error) {
	var order Order
	err := db.QueryRowContext(
		ctx,
		`SELECT id, status FROM orders WHERE id = $1`,
		id,
	).Scan(&order.ID, &order.Status)
	return order, err
}

func handleOrder(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// r.Context() 会在客户端断开或请求结束时取消。
		// 子 deadline 进一步限制本次处理最多使用 800ms。
		ctx, cancel := context.WithTimeout(r.Context(), 800*time.Millisecond)
		defer cancel()

		order, err := loadOrder(ctx, db, r.PathValue("id"))
		if err != nil {
			switch {
			case errors.Is(err, context.DeadlineExceeded):
				http.Error(w, "timeout", http.StatusGatewayTimeout)
			case errors.Is(err, context.Canceled):
				return
			default:
				http.Error(w, "internal error", http.StatusInternalServerError)
			}
			return
		}

		_ = json.NewEncoder(w).Encode(order)
	}
}
```

传播链是：`r.Context()` → handler 的 800ms child Context → `QueryRowContext`。客户端断开、800ms 到期或 handler 主动 `cancel()`，都会通知数据库调用停止等待。前提是驱动和下游 API 正确支持 Context；如果函数内部忽略 `ctx.Done()`，Context 不会神奇地中断它。

#### 2.3.4 正确使用约定

- 需要 Context 的函数把 `ctx context.Context` 放在第一个参数，并沿调用链原样向下传；不要在深层函数重新使用 `context.Background()`，否则会切断上游取消。
- 不把 Context 长期存入 struct，不传 `nil`；顶层程序或测试用 `Background()`，暂时无法确定来源时才用 `TODO()`。
- 调用 `WithCancel`、`WithTimeout` 或 `WithDeadline` 后通常立即 `defer cancel()`。即使超时尚未到，主动 cancel 也能尽早停止 timer，并解除 parent 对 child 的引用。
- `context.Value` 只放请求级、跨 API 的附加元数据；分页大小、重试次数、数据库句柄等业务参数应显式传参。key 使用包内自定义类型并提供类型安全 accessor，避免字符串 key 冲突。
- channel 收发、循环和自写阻塞函数要显式监听 `ctx.Done()`；优先调用 `Do(req.WithContext(ctx))`、`QueryContext` 等原生支持 Context 的 API。

#### 2.3.5 Context 没有保证什么

- `cancel()` 只发送协作取消信号，不等待任务停止，也不能回滚已经发生的外部副作用。
- deadline 到期表示调用方不再等待，不代表远端写操作一定没有成功。支付、数据库写入或 RPC 超时可能是 unknown outcome，仍需幂等键、查询和对账。
- Context 不替代并发上限、重试策略、事务、锁或 goroutine join。取消后仍无限重试，会抵消整个生命周期控制。
- `context.WithoutCancel` 会有意切断 parent 的取消和 deadline；只有确需脱离原请求的收尾任务才考虑使用，并应重新设置独立超时，避免产生无界后台任务。

### 2.4 Mutex、RWMutex、Atomic、WaitGroup、Once

- `Mutex` 保护复合不变量；临界区小、不要在锁内慢 I/O。
- `RWMutex` 只在读占绝对多数且临界区足够长时可能受益；写者和调度开销会抵消优势。
- `atomic` 适合单值计数、指针/不可变快照发布；多个字段不变量仍需锁或整体 CAS。
- `WaitGroup` 等待一组任务完成，不传播错误/取消；复杂并发常用 `errgroup`。
- `Once` 保证函数最多执行一次；函数 panic 也通常被视为已经执行，初始化失败重试需另建状态机。

复制已经使用的 Mutex/WaitGroup/Once 会破坏状态；包含它们的 struct 通常用指针传递，并用 `go vet` 检查 copylocks。

### 2.5 Memory Model 与 happens-before

数据竞争是至少两个 goroutine 并发访问同一内存、至少一个写且没有同步。happens-before 可由 channel send/receive、close、mutex unlock/lock、atomic 等建立。没有同步时，即使“在我的机器上先写后读”，编译器和 CPU 也不承诺可见顺序。

`map` 不能在无同步下并发读写。切片 header 是指针、len、cap，append 可能换底层数组；把 slice 传给 goroutine 前要明确 ownership 或复制。

### 2.6 高性能 Worker Pool 设计 `[工程化中]`

本仓库的可运行参考实现见 [`workerpool`](workerpool/README.md)。这里的“线程池”实际是 **固定数量的 worker goroutine**，并不创建或绑定固定 OS thread。设计目标不是代替 Go scheduler，而是在应用层提供有界并发、背压、任务生命周期和可观测性。

#### 2.6.1 CPU、worker 与 OS thread

| 参数 | 控制对象 | 作用域 | 不能保证什么 |
| --- | --- | --- | --- |
| `RuntimeConfig.MaxProcs` | `runtime.GOMAXPROCS` | 整个 Go 进程 | 不是 CPU 时间配额，也不是 Pool 独占 CPU |
| `Config.Workers` | 常驻 worker goroutine | 单个 Pool | 不限制任务自行创建的子 goroutine |
| `Config.QueueCapacity` | 等待执行的任务 | 单个 Pool | 不包含正在执行的任务 |
| OS thread | Go runtime 的 M | 整个进程 | Pool 不控制数量、affinity 或创建时机 |

`GOMAXPROCS` 限制同时执行 Go 代码的并行度；worker 数限制 Pool 同时运行多少个顶层任务。CPU 密集任务可从 `Workers≈GOMAXPROCS` 开始 benchmark，阻塞 I/O Pool 可以配置更多 worker，但还必须服从连接池与下游容量。库只允许在第一个 Pool 创建前显式配置一次 `GOMAXPROCS`；配置为 `0` 时保留 runtime 的容器感知默认行为。

#### 2.6.2 核心接口

```go
type Pool interface {
	TrySubmit(context.Context, Task, ...TaskOption) (Handle, error)
	State() PoolState
	Snapshot() Stats
	Shutdown(context.Context) error
	Stop(context.Context) error
	Done() <-chan struct{}
}

type Task func(context.Context) error

type Handle interface {
	ID() TaskID
	State() TaskState
	Cancel() CancelResult
	Wait(context.Context) (TaskReport, error)
	Done() <-chan struct{}
}
```

`TrySubmit` 永不等待容量：成功即取得任务 ownership，队列满、Pool 已关闭或退化时分别返回稳定错误。`Handle.Wait` 的 Context 只限制调用方等待，不隐式取消任务；`Handle.Cancel` 对排队任务直接生效，对运行任务只发送协作取消信号。Go method 不能声明自己的类型参数，因此带返回值版本采用包级 `SubmitValue[T]`，内部仍提交普通 `Task`。

#### 2.6.3 内部状态与热路径

```go
type pool struct {
	cfg  poolConfig
	jobs chan *taskEnvelope

	lifecycle atomic.Uint32
	degraded  atomic.Bool

	activeSubmitters atomic.Int64
	admissionMu      sync.Mutex
	admissionCond    *sync.Cond
	queueCloseOnce   sync.Once

	rootCtx    context.Context
	rootCancel context.CancelCauseFunc

	workers     []workerSlot
	liveWorkers atomic.Int64
	done        chan struct{}

	idleCount, runningCount atomic.Int64
	queuedCount, stuckCount atomic.Int64

	counters   poolCounters
	histograms histogramSet
}
```

配置在 `New` 中复制、排序和校验，避免调用方之后修改 slice。每个 `taskEnvelope` 保存提交 Context、任务状态、执行 cancel、queue/execution/stuck timer、最终报告和只关闭一次的 `done`。高频状态读取使用 typed atomic；timer、cancel 与未完成报告由任务级 mutex 保护；报告写完后才 `close(done)`，由 channel happens-before 保证等待者看到完整结果。

任务主状态机是：

```text
Queued
├── Running
│   ├── CancelSignaled
│   ├── TimeoutSignaled
│   ├── Stuck
│   └── Succeeded / Failed / Canceled / TimedOut / Panicked
└── Canceled / TimedOut
```

取消、worker 取任务和 queue timeout 竞争同一个 `Queued` CAS，只有胜者减少逻辑排队数并完成 Future，因此任务不会重复执行或重复计数。已取消的 envelope 可能暂时作为 tombstone 留在物理 channel 中，所以监控要区分 logical queued 与 physical slots used。

#### 2.6.4 FIFO、唤醒与关闭

生产实现使用有界 channel，同时完成 FIFO 存储、容量控制和 worker parking。一个 send 只与一个 receiver 配对，不会像每次 `Broadcast` 那样唤醒所有空闲 worker；channel close 仅在 shutdown 时一次性唤醒剩余 receiver。worker 不轮询，也不为每个任务再创建 supervisor goroutine，稳态 Pool goroutine 数约等于 `Workers`。

关闭时存在经典的 send/close race。实现用 admission counter 保护：

1. `TrySubmit` 进入时增加 active submitter，退出时减少。
2. Shutdown 先把 lifecycle 从 running 切到 draining，使新提交快速失败。
3. 等 active submitter 归零后才 close jobs channel。
4. worker 排空 channel 后退出；最后一个 worker 关闭 Pool 的 `Done`。

`Shutdown` 排空已接受任务；`Stop` 还会取消 Pool root Context。两者都无法强杀忽略 Context 的任务，因此等待 Context 到期只代表调用方不再等待，Pool 可能仍处于 stopping。

#### 2.6.5 超时、stuck 与 panic

`MaxQueueWait` 从提交时计算，超时任务用 CAS 在开始前完成；`MaxExecutionTime` 从 worker 真正开始执行时计算。执行超时只调用 cancel，不把任务 goroutine detach，也不补建 worker，否则会悄悄突破并发上限。

取消后超过 `CancelGracePeriod` 仍未返回的任务标记为 stuck。达到 `StuckThreshold` 后 Pool 进入 degraded 并拒绝新任务，stuck 任务退出后自动恢复。这是 bulkhead/circuit-breaker 式止损，不是强制终止；任务已经产生的外部副作用仍需幂等和对账。

worker 在任务边界 recover，将普通 panic 转为 `TaskPanicked` 并保留有限 stack；runtime fatal error 仍会终止进程。Panic handler 本身也要隔离 panic，且不应执行无界阻塞操作。

#### 2.6.6 为什么不承诺 wait-free

- **Wait-free**：每个操作在有限个自身步骤内完成。
- **Lock-free**：系统整体持续推进，但单个 goroutine 可能持续 CAS 失败。
- **Blocking**：操作可能等待 mutex、condition variable 或 channel。

Go 可以在特定架构和固定容量假设下实现算法意义上的 wait-free 结构，但通用库不能给出 portable wait-free MPMC queue 保证：`sync/atomic` 规定原子性和顺序一致性，却没有承诺每个目标平台的原子操作都 wait-free；常见 CAS retry loop 只具备 lock-free 候选性质；bounded sequence ring 的 producer 若预留 slot 后长时间停顿，consumer 仍可能卡在该 slot；真正的 wait-free MPMC 常依赖 helping、operation descriptor、版本标记或 portable multi-word CAS，而标准库没有通用 128-bit CAS。

此外，算法步骤数有界也不等于墙钟延迟有界，Go scheduler 不提供 goroutine 的硬实时调度保证。是否用自研 MPMC ring 应由 `go test -race`、状态机 fuzz、runtime trace 和真实 producer/worker 矩阵 benchmark 决定，不能仅凭“无锁”推断更快。

#### 2.6.7 观测与验证

`Snapshot` 返回 configured/observed `GOMAXPROCS`、worker/idle/running/queued/stuck、物理队列占用、各类结果和拒绝原因。queue wait、execution、end-to-end 使用固定桶直方图；每个 worker 主要写自己的 atomic shard，抓取时聚合，避免在任务热路径调用外部 exporter。

关键验证包括：单 worker FIFO、满队列快速拒绝、取消/启动/关闭竞争、queue/execution timeout、panic 后 worker 继续服务、全部 worker stuck、Future 多等待者、无 send-on-closed-channel，以及 channel 与候选 MPMC ring 的吞吐、P99、park/unpark、mutex contention 和 `allocs/op` 对比。

### 2.7 Interface、nil、defer 与 panic

interface 值包含动态类型和动态值。一个带 `(*T)(nil)` 的 interface 不等于 nil：

```go
var p *MyError
var err error = p
fmt.Println(err == nil) // false
```

方法集决定值/指针是否实现 interface。接口应由使用者定义并保持小；不为“未来可能 mock”预先制造庞大接口。

`defer` 在函数返回时 LIFO 执行，参数在 defer 语句处求值；适合释放锁/文件。`panic/recover` 用于进程内不可恢复边界或框架隔离，不应成为普通错误流；recover 只在同一 goroutine 的 deferred function 中生效。

### 2.8 GC、逃逸分析与栈增长

Go 使用并发 GC，目标是降低暂停，但大量 allocation、指针密集结构和过高 `GOGC`/memory limit 配置仍会影响 CPU 与尾延迟。goroutine 栈按需增长，栈上的对象若生命周期逃出或大小/调用方式不适合会分配到堆。

用 `go build -gcflags=-m` 观察逃逸，不能把输出机械当 bug。优先减少不必要 allocation、复用有界 buffer、避免对象池长期保留巨型对象；`sync.Pool` 可随 GC 清空，不是缓存或资源池。

### 2.9 Benchmark 与逃逸分析

`testing.B` 用于可重复的局部基准；`go test -bench . -benchmem` 同时报告耗时、`B/op` 和 `allocs/op`。基准要覆盖真实输入规模，把建数据、随机 I/O 等非被测工作放在计时区外，并用结果防止编译器消除无副作用计算。优化前后各运行多次，再用 `benchstat` 比较分布；不要拿一次 `ns/op` 的小波动下结论。

```bash
go test -run '^$' -bench . -benchmem -count=10 > old.txt
go test -run '^$' -bench . -benchmem -count=10 > new.txt
benchstat old.txt new.txt
go test -run '^$' -bench BenchmarkFoo -cpuprofile cpu.out -memprofile mem.out
go build -gcflags='all=-m=2' ./...
```

逃逸分析说明“编译器为何把值放到堆上”，不是性能问题清单。把对象改成栈分配可能减少 GC 压力，也可能让 API 更复杂或复制更多；应结合 `allocs/op`、CPU/heap profile 和端到端延迟验证。

### 2.10 Race Detector、pprof 与 trace

`go test -race ./...` 通过运行时插桩发现实际执行路径的数据竞争；未覆盖路径仍可能有 race，且开启后有明显开销。修复 race 应建立 ownership/同步，而不是加 sleep。

pprof 可由 `go test`、`runtime/pprof` 或受保护的 `net/http/pprof` endpoint 采集。线上 endpoint 不应直接暴露公网；一次只采集一种 profile，并先评估采样开销。分析时先看 `top`，再看调用图、source/list 或 flame graph：**flat** 是函数自身成本，**cum** 是函数连同下游调用的累计成本。

| Profile | 回答的问题 | 常见误区 |
| --- | --- | --- |
| CPU | CPU 正在哪些调用栈消耗 | I/O 等待不一定出现在 CPU 热点中 |
| `heap` / `inuse_space` | 当前仍存活的采样对象由谁分配 | 单次快照不能证明泄漏，应在相似 GC/负载阶段对比 |
| `allocs` / `alloc_space` | 进程累计分配压力来自哪里 | 对象已回收也计入，数值大不等于当前占用大 |
| goroutine | goroutine 当前栈与等待位置 | 数量高不一定泄漏，要看是否持续增长且无法退出 |
| mutex | 哪些临界区造成锁竞争 | 栈通常指向释放锁/持锁方，不只是等待方 |
| block | channel、锁、条件变量等阻塞时间 | 需显式设置采样率，开启本身有成本 |

```bash
go tool pprof -http=:0 cpu.out
go tool pprof -http=:0 -inuse_space mem.out
go tool pprof -http=:0 -alloc_space mem.out
go test -run TestScenario -trace trace.out
go tool trace trace.out
```

`go tool trace` 展示 goroutine 调度、阻塞/唤醒、syscall、GC 和处理器利用率的时间线，适合排查“CPU 没跑满但延迟高”、串行化或调度问题；找普通 CPU/内存热点应先用 pprof。GC 压力可结合 `GODEBUG=gctrace=1`、`runtime/metrics` 与 heap/allocs profile 观察；`GOGC`、`GOMEMLIMIT` 是测量后的容量权衡，不是泄漏修复开关。

### 2.11 Go 常见并发故障

- goroutine 等待永不 close/发送的 channel，造成泄漏。
- 无界 fan-out 压垮数据库/RPC。
- 在持锁状态发送到可能阻塞的 channel，形成锁环。
- 循环变量/闭包捕获语义与目标 Go 版本不一致；即使新版本改善 range 变量，也要显式传参保持清晰。
- timer/ticker 未 stop 或结果 channel 无人消费。
- context 被取消后仍重试，造成请求幽灵流量。

### 2.12 Go 高频问答

1. **Channel 与 Mutex 如何选？** ownership transfer/workflow coordination 用 channel；共享内存复合不变量用 Mutex。不要为了口号把简单计数器改成复杂 actor。
2. **buffered channel 是否异步？** 只允许容量内 send 暂不等待，满后仍同步；它不是持久队列。
3. **如何防 goroutine leak？** 明确终止条件、context、关闭所有权、bounded concurrency，并用 goroutine profile 验证。
4. **RWMutex 一定更快吗？** 否；读临界区短、写频繁或核心数少时开销可能更高，benchmark 决定。
5. **Race detector 无报错是否证明安全？** 否；只覆盖本次执行路径。
6. **`heap` 与 `allocs` profile 有何区别？** `heap/inuse` 看仍存活对象，适合保留与泄漏；`allocs/alloc_space` 看累计分配，适合高 churn 和 GC 压力。两者都基于采样。
7. **Go 服务内存上涨如何排查？** 先区分 RSS、Go heap、goroutine stack 和进程外内存；在相似负载及 GC 阶段对比 heap profile，再看 allocs、goroutine 数和 GC 指标，定位是对象保留、分配过快、goroutine 泄漏还是 runtime/系统层占用。
8. **pprof 与 trace 如何选？** CPU/分配/存活对象/锁热点先 pprof；需要看 goroutine 随时间如何调度、阻塞和被唤醒时用 trace。
9. **Context 的主要作用是什么？** 在调用链和 goroutine 之间传播取消、deadline 与少量请求级元数据，统一请求生命周期；取消是协作信号，不负责强杀、等待完成或回滚外部副作用。
10. **`GOMAXPROCS` 与 worker 数有什么区别？** 前者是整个 Go 进程同时执行 Go 代码的并行度，后者是一个 Pool 同时运行的顶层任务数；worker 更多不等于能使用更多 CPU。
11. **空闲 worker 如何避免惊群？** 有界 channel 的一次 send 只匹配一个 receiver；若自研队列使用 `sync.Cond`，正常入队用 `Signal`，只在 shutdown 使用 `Broadcast`，并用同一 mutex 保护谓词检查和 `Wait`。
12. **Go 能否实现 wait-free MPMC queue？** 特定平台和算法假设下可以研究，但标准库没有跨架构进展性保证或 portable multi-word CAS，scheduler 也不保证墙钟上界；生产库不应给出 portable wait-free 承诺。

## 3. Solidity

Ethereum 协议层 EVM 调用与 gas 见 [01-ethereum.md](01-ethereum.md#3-交易执行与-evm)。本节聚焦合约布局、升级与安全。

### 3.1 ABI 与数据位置

ABI 定义函数 selector（签名 Keccak 前 4 bytes）、参数/返回值编码和 Event topic/data。动态类型在 head 中保存 offset，实际内容在 tail；`abi.encodePacked` 会移除边界信息，对多个动态类型可能产生碰撞，不应用作无域分隔签名。

- `calldata`：外部输入，只读，适合 external 参数。
- `memory`：调用期间临时数据，扩张收费。
- `storage`：持久状态，赋值语义可能是引用或复制，gas 高。

`msg.data`、`msg.sender`、`msg.value` 属于调用上下文；delegatecall 会保留 sender/value，却在代理 storage 上运行实现代码。

### 3.2 Storage Layout

静态小类型按声明顺序 packing 到 32-byte slot，跨 slot 或 struct/array 边界有特定规则。mapping 与 dynamic array 的 slot 保存锚点/长度，元素位置由 Keccak 派生。

```solidity
contract Layout {
    uint128 a; // slot 0 lower 16 bytes
    uint128 b; // slot 0 upper 16 bytes
    uint256 c; // slot 1
    mapping(address => uint256) balance; // anchor slot 2
}
```

`balance[key]` 位于 `keccak256(abi.encode(key, uint256(2)))`。继承状态变量按 C3 linearization 参与布局；升级时改变基类顺序也可能破坏槽位。

### 3.3 Call、Delegatecall、Staticcall、Fallback/Receive

低级 `call` 返回 `(success, returndata)`，不会自动 revert，必须检查并正确冒泡错误。`staticcall` 禁止状态修改但被调代码仍可读取环境。`delegatecall` 用实现代码修改调用者状态，是 proxy 核心也是布局/权限风险源。

`receive()` 处理空 calldata 的原生币转账；`fallback()` 处理 selector 不匹配或无 receive 的情况。依赖 2300 gas stipend 的安全假设不稳健，应使用 pull payment、显式 call 并防重入。

### 3.4 Proxy 模式

| 模式 | 升级入口 | 优势 | 风险/代价 |
| --- | --- | --- | --- |
| Transparent | Proxy admin 分流，普通用户 delegate | 管理调用与业务 selector 隔离 | proxy 较重；admin 不能走普通 fallback |
| UUPS | Implementation 中 `_authorizeUpgrade` | proxy 轻、逻辑灵活 | 实现权限/升级函数错误可永久损坏或被接管 |
| Beacon | 多个 proxy 读取同一 beacon implementation | 批量升级实例 | beacon 成为共同故障域 |

EIP-1967 规定 implementation/admin/beacon 的特殊槽，避免与业务变量碰撞。ERC-7201 提供 namespaced storage layout，适合模块化升级，但不会自动验证新旧结构兼容。

升级合约使用 constructor 设置实现自身不会初始化 proxy storage；应使用 initializer，并防重复初始化、实现合约被接管和 reinitializer 次序错误。升级规则：不删除/重排/改类型，通常只在末尾追加；结构、mapping value 和继承改动也要检查。

### 3.5 常见攻击与防护

#### 重入

外部调用在状态完成前把控制权交出。使用 Checks-Effects-Interactions、pull payment、必要的 reentrancy guard；跨函数/跨合约/只读重入也要分析。Guard 不是替代状态机设计。

#### 权限与升级

校验所有 admin、role admin、proxy admin、timelock、guardian 和 pause 权限。`tx.origin` 不用于授权。初始化、升级、签名执行器和多签模块都是高危入口。

#### 签名重放

签名消息绑定 chain ID、verifying contract、action、nonce、deadline 和关键参数；使用 EIP-712 domain separation。标记 nonce consumed 的时机需与外部调用及失败语义匹配。

#### 预言机与价格操纵

不要把单池 spot price 当 oracle。使用时间加权/多源/心跳与 deviation check，处理 decimals、stale round、L2 sequencer 状态和极端流动性。经济攻击可能在一个原子交易中借 flash loan 完成。

#### 精度、DoS 与 Token 兼容

先乘后除可能溢出或损失精度，使用经过审计的 `mulDiv`；明确 rounding 方向。避免对可增长数组无界循环。ERC-20 可能不返回 bool、收转账税、rebasing 或回调，使用 SafeERC20 并用余额差核对真实到账。

### 3.6 MEV、Sandwich 与 Slippage

公开 mempool 中，攻击者可围绕用户 swap 前后交易形成 sandwich。合约/UI 至少要求 `amountOutMin`、deadline 和合理 price impact；批量拍卖、commit-reveal、private transaction 可改变泄露/排序面，但各有延迟与中心化权衡。

MEV 是执行排序问题的完整讨论，Ethereum 文档只保留协议角色；应用侧防护在此落地。不能承诺 `minOut=0` 的交易“只是成交差一点”，它可能被抽走几乎全部可提取价值。

### 3.7 常用标准

| 标准 | 核心对象 | 面试重点 |
| --- | --- | --- |
| ERC-20 | fungible token | allowance race、返回值、decimals 非强制语义 |
| ERC-721 | NFT ownership | safe transfer receiver callback、approval |
| ERC-1155 | multi-token | batch、receiver callback、ID 级余额 |
| EIP-712 | typed structured data | domain separator、nonce、deadline |
| ERC-2612 | permit | allowance by signature、nonce/replay |
| ERC-4626 | single-asset tokenized vault | asset/share 换算、rounding、inflation attack、流动性边界 |
| ERC-4337 | account abstraction | 协议流程与 gasless 见 [Ethereum AA 主线](01-ethereum.md#36-account-abstractionaa与-gas-abstraction) |

### 3.8 ERC-4626 Tokenized Vault

> `[已上线]` ERC-4626 已是 Final 标准；本节产品信息核验于 2026-07-16。

ERC-4626 为“单一底层 ERC-20 资产的代币化金库”统一接口。用户存入 `asset`（如 USDC），获得同样遵循 ERC-20 的 `share`；share 表示对金库管理资产的按比例索取权。标准化的是存取、报价和限额接口，不规定资金必须投向哪里、收益如何产生、管理员权限、估值方法或策略风险，因此“接口兼容”不等于“资金安全”。

#### 关键函数：四种交易意图与三组只读函数

| 类别 | 函数 | 调用者指定什么 | 返回/含义 |
| --- | --- | --- | --- |
| 存入 | `deposit(assets, receiver)` | 精确投入多少 assets | 实际铸造的 shares，向下取整 |
| 存入 | `mint(shares, receiver)` | 精确获得多少 shares | 实际需要的 assets，向上取整 |
| 退出 | `withdraw(assets, receiver, owner)` | 精确取出多少 assets | 实际销毁的 shares，向上取整 |
| 退出 | `redeem(shares, receiver, owner)` | 精确销毁多少 shares | 实际收到的 assets，向下取整 |
| 基础状态 | `asset()`、`totalAssets()` | 底层币与管理资产总量 | `totalAssets` 应包含复利收益和已计入资产的费用 |
| 理想换算 | `convertToShares`、`convertToAssets` | 通用、平均用户报价 | 不含费用/滑点，必须向下取整，不保证等于本次成交结果 |
| 操作预览 | `previewDeposit/Mint/Withdraw/Redeem` | 某一种具体操作 | 尽量贴近当前链上条件，计入相应费用，但不表达限额 |
| 可执行上限 | `maxDeposit/Mint/Withdraw/Redeem` | receiver 或 owner | 结合暂停、额度、余额和可用流动性给出当前上限 |

成功 `deposit`/`mint` 发出 `Deposit`，成功 `withdraw`/`redeem` 发出 `Withdraw`。代 owner 销毁 share 时还要检查 ERC-20 allowance。集成方不能用 `convertToAssets` 代替 `previewRedeem`，也不能只看 preview 而忽略 `max*`；直接面向用户的交易还应由 router/wrapper 增加 `minShares`、`maxAssets`、`minAssets` 或 deadline，因为标准交易函数本身没有 slippage 参数。

#### Share 价格如何形成

最常见的会计模型是：

```text
pricePerShare（以 asset 计价） ≈ totalAssets / totalSupply
sharesOut ≈ assetsIn × totalSupply / totalAssets
assetsOut ≈ sharesIn × totalAssets / totalSupply
share 的美元价格 ≈ pricePerShare × asset 的美元价格
```

例如金库有 1,000,000 USDC 和 1,000,000 shares，1 share 约值 1 USDC。策略赚取 100,000 USDC 且 share 供应量不变后，1 share 约值 1.1 USDC。正常的等比例存款/赎回会同时改变分子与分母，理论上不改变价格；策略收益、利息和直接捐赠提高 `totalAssets`，已确认亏损或费用扣除降低它，铸造/销毁管理费 share 则通过改变 `totalSupply` 稀释或增厚每份权益。实现还可能通过 profit locking 逐步确认利润，或让 `convert*` 使用更稳健但不完全实时的估值，所以公式是常见机制而非标准强制的 oracle 规则。

“share price 上涨”也不等于美元收益：底层 asset 若对美元下跌，share 以 asset 计价的升值可能仍不足以抵消跌幅。`preview*` 依赖即时链上状态且可被操纵，不应直接作为借贷清算等高价值场景的唯一价格预言机。

#### 风险、取舍与失败场景

- **空金库 inflation/donation attack**：攻击者先取得极少 share，再直接向 vault 转入大量 asset，抬高兑换率，使受害者存款向下取整为 0 或极少 shares。常见防护是不可赎回的初始流动性，或 OpenZeppelin 风格的 virtual assets、virtual shares 与 decimals offset；用户侧仍应校验最少 shares。
- **取整必须偏向 vault**：用户得到的 shares/assets 向下取整，用户为精确输出支付/销毁的 assets/shares 向上取整。方向写反会让反复小额操作抽走价值。
- **记账不等于可提现**：`totalAssets` 可包含已借出或投入策略的资产；当市场缺乏流动性、策略暂停或发生损失时，账面权益不代表能立即全额退出，应检查 `maxWithdraw/maxRedeem` 和真实赎回路径。
- **非标准底层 token**：fee-on-transfer、rebasing、回调型或异常 decimals token 会破坏“参数数量等于实际到账”的假设；实现需用余额差记账并明确支持范围。
- **标准没有覆盖所有 vault**：原生币、多资产、排队或异步赎回不是基础 ERC-4626 的同步单资产模型；此类系统需 wrapper 或扩展标准（例如异步 vault 的 ERC-7540），不能伪造即时可赎回语义。

#### 知名实现与产品

| 项目 | 4626 的角色 | 面试时应说明的差异 |
| --- | --- | --- |
| OpenZeppelin Contracts | 通用 `ERC4626` 基类，不是收益产品 | 提供 virtual offset 等安全实现基础；策略、权限、费用和资产部署仍由项目自行设计 |
| Morpho Vaults | Morpho Earn 的 ERC-4626 lending vault | 单一 loan asset，vault 可把流动性配置到多个借贷市场；share 收益来自借款利息等，仍受 curator 配置与市场风险影响 |
| Yearn V3 Vaults | ERC-4626 多策略 meta-vault；其 strategy 也采用 4626 接口 | vault 把单一 asset 分配给多个策略，并可通过锁定/逐步解锁利润平滑 price-per-share |
| Euler Vault Kit（EVK） | 带借款能力的 ERC-4626 credit vault 构建框架 | 不只是被动收益包装；vault 还定义借贷、抵押接受范围及相应风险边界 |

一句话面试回答：**ERC-4626 把收益仓位标准化成“asset 进、ERC-20 share 出”的同步单资产接口；share 对 asset 的兑换率通常由管理资产净值除以 share 供应量形成，但实际成交还受费用、取整、限额、流动性和实现记账影响。**

### 3.9 Smart Account 与 AA 实现审计

ERC-4337、EIP-7702、UserOperation、Bundler 和两种 Gasless/Paymaster 模式的完整协议与系统设计见 [01-ethereum.md](01-ethereum.md#36-account-abstractionaa与-gas-abstraction)。本节只保留 Solidity 实现和审计边界：

- **EntryPoint 边界**：`validateUserOp`、Paymaster validation/post-operation、account 初始化和 aggregator 验签入口必须限制可信 EntryPoint caller；低级外部入口不能绕开验证。
- **签名与 nonce**：`userOpHash` 必须绑定 chain、EntryPoint、account、nonce、有效期和被授权调用；keyed nonce 的每条权限域单独维护、撤销和审计。
- **Proxy 与 storage**：Smart Account 常为 proxy 或模块化账户。升级不能重排 storage；模块应使用明确 namespace，安装/卸载/升级都属于高权限操作。
- **模块权限**：session key、guardian、recovery、spending limit 只拥有所需 selector、目标、额度和时间窗，不能因一个模块获得无限 `execute` 或 upgrade 权。
- **Paymaster**：只信任经链上验证的赞助授权；对 token 回调、`postOp` 失败、fee-on-transfer/rebasing token 与预算耗尽分别设计失败路径。

审计时把 account、factory、EntryPoint 版本、Paymaster、module manager、guardian 和 upgrade admin 画成权限图；任何一个可替换实现或可调用初始化器的地址都属于攻击面。

### 3.10 Solidity 高频问答

1. **Proxy 为什么能升级？** 地址和 storage 留在 proxy，fallback delegatecall 到可变 implementation；升级只是改实现槽。
2. **为何升级不能重排变量？** delegatecall 按 slot 读写，旧数据仍在原槽，新代码布局变化会误解释。
3. **`call` 成功是否表示业务成功？** 只表示被调帧未 revert；还需解码返回值和校验状态/token 行为。
4. **EIP-712 是否防重放？** 只提供结构化域；仍需 nonce、deadline、chain/contract/action 绑定和 consumed 状态。
5. **Smart Account 最容易遗漏的审计项？** 先画 EntryPoint、factory、模块、guardian、upgrade admin 的权限图；再核对 hash 域、nonce、initializer、storage layout 和 Paymaster 的外部调用失败路径。
6. **ERC-4626 的 share price 如何形成？** 通常是 `totalAssets / totalSupply`；收益/亏损改变分子，费用也可能改变分子或通过增发 share 改变分母，但标准允许实现采用自己的会计与稳健换算，不能把 preview 当成无条件安全的 oracle。
7. **`deposit` 与 `mint`、`withdraw` 与 `redeem` 有何区别？** 前者分别固定输入 assets、输出 assets，后者分别固定输出 shares、输入 shares；为了保护 vault，用户收到的数量向下取整，用户付出的数量向上取整。

## 4. 跨语言比较

### 4.1 调度与执行

| 维度 | Rust async | Go goroutine | EVM transaction |
| --- | --- | --- | --- |
| 调度者 | executor/runtime | Go runtime G-M-P | 区块定义顺序，客户端执行 |
| 抢占 | cooperative poll 为主 | runtime 可抢占 | 单交易确定性执行，不以应用线程并发暴露 |
| 阻塞风险 | poll 中阻塞会卡 executor thread | 阻塞通常由 runtime 调度处理，但仍耗资源 | out-of-gas/revert，受 block gas 限制 |
| 状态共享 | type system + locks/channels | locks/channels + memory model | 合约 storage，交易原子提交 |
| 失败恢复 | Result/panic、应用持久化 | error/panic、应用持久化 | revert 回滚当前调用状态，链历史保留 |

Rust Future 和 Go goroutine 是进程内并发抽象；EVM transaction 是复制状态机中的确定性状态转换。把“EVM 并行执行”与 goroutine 数量直接类比会忽略共识等价性要求。

### 4.2 内存、安全与运行时

- Rust 无 tracing GC，资源释放确定；代价是 ownership/lifetime 学习和 API 设计复杂度。
- Go 由 GC 管理内存，迭代快；代价是 allocation/GC、runtime 调度和数据竞争需运行时工具发现。
- Solidity 的 memory/storage 由 EVM 管理，所有节点重复执行；代价被 gas 显式计费，错误一旦上链难以修复。

### 4.3 错误与副作用

Rust/Go 的错误可以由服务重试，但网络写入结果可能未知；Solidity revert 会回滚当前交易状态，却不能回滚已完成的其他链/外部系统。三者共同需要稳定 idempotency identity，只有边界和实现工具不同。

## 5. 容易混淆或说错的点

- Rust lifetime 不创建或延长运行时生命周期。
- `Arc` 不等于对象内部线程安全，`Pin` 不等于堆分配。
- async 不是自动并行；阻塞代码不会因加 `async` 变为非阻塞。
- Go channel 不是消息队列，buffer 不是持久化；goroutine 也不是 OS thread。
- `GOMAXPROCS`、worker goroutine 数和 OS thread 数是三个不同层次，不能用一个“线程数”参数替代。
- lock-free 不等于 wait-free，也不等于在 Go 中一定比 channel 快；要同时验证线性化、进展性、唤醒成本和真实负载。
- Go Context 不是万能参数包，也不会强制终止 goroutine；`cancel()` 后任务是否及时退出取决于代码和下游是否响应取消。
- `interface` 内部 typed nil 不等于 nil。
- `RWMutex` 不是读多场景的无条件最优解。
- Go 的 `allocs` 大不等于当前 heap 大，Go heap 或 Rust live heap 小也不等于进程 RSS 一定小。
- Rust debug build 的热点不能代表 release 性能；火焰图的宽度是采样占比，不是单次调用耗时。
- Solidity `private` 只限制合约调用，不隐藏链上数据。
- Event 不可作为合约内权威状态，Receipt 也没有 Solidity 返回值。
- Proxy upgrade 改代码，不迁移 storage；布局兼容必须单独验证。
- ERC-4626 统一接口，不统一策略、收益率、估值可信度或安全性；`totalAssets / totalSupply` 也不是标准强制的外部价格预言机。
- ERC-4337 UserOperation 不是新的 L1 共识交易类型。

## 6. 官方资料与延伸阅读

### Rust

- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Rust Reference](https://doc.rust-lang.org/reference/)
- [Traits: Defining Shared Behavior](https://doc.rust-lang.org/stable/book/ch10-02-traits.html)
- [Associated Items and Types](https://doc.rust-lang.org/stable/reference/items/associated-items.html)
- [Trait Objects](https://doc.rust-lang.org/stable/reference/types/trait-object.html)
- [Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
- [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html)
- [`std::pin::Pin`](https://doc.rust-lang.org/std/pin/struct.Pin.html)
- [`std::task::Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html)
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
- [Tokio Runtime](https://docs.rs/tokio/latest/tokio/runtime/)
- [Cargo `bench`](https://doc.rust-lang.org/cargo/commands/cargo-bench.html)
- [Criterion.rs User Guide](https://criterion-rs.github.io/book/)
- [`cargo-flamegraph`](https://github.com/flamegraph-rs/flamegraph)
- [Linux `perf`](https://perf.wiki.kernel.org/)
- [Heaptrack](https://github.com/KDE/heaptrack)
- [Valgrind Massif](https://valgrind.org/docs/manual/ms-manual.html)
- [`tokio-console`](https://github.com/tokio-rs/console)

### Go

- [Effective Go](https://go.dev/doc/effective_go)
- [`context` package](https://pkg.go.dev/context)
- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Contexts and structs](https://go.dev/blog/context-and-structs)
- [Go Memory Model](https://go.dev/ref/mem)
- [`runtime.GOMAXPROCS`](https://pkg.go.dev/runtime#GOMAXPROCS)
- [Container-aware `GOMAXPROCS`](https://go.dev/blog/container-aware-gomaxprocs)
- [`sync.Cond`](https://pkg.go.dev/sync#Cond)
- [`sync/atomic`](https://pkg.go.dev/sync/atomic)
- [Go GC Guide](https://go.dev/doc/gc-guide)
- [Data Race Detector](https://go.dev/doc/articles/race_detector)
- [Profiling Go Programs](https://go.dev/blog/pprof)
- [Go Diagnostics](https://go.dev/doc/diagnostics)
- [`testing` benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks)
- [`runtime/pprof`](https://pkg.go.dev/runtime/pprof)
- [`runtime/trace`](https://pkg.go.dev/runtime/trace)
- [`benchstat`](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat)

### Solidity 与标准

- [Solidity Documentation](https://docs.soliditylang.org/en/latest/)
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)
- [Solidity Security Considerations](https://docs.soliditylang.org/en/latest/security-considerations.html)
- [OpenZeppelin: Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 与 [ERC-7201](https://eips.ethereum.org/EIPS/eip-7201)
- [ERC-20](https://eips.ethereum.org/EIPS/eip-20)、[ERC-721](https://eips.ethereum.org/EIPS/eip-721)、[ERC-1155](https://eips.ethereum.org/EIPS/eip-1155)
- [EIP-712](https://eips.ethereum.org/EIPS/eip-712) 与 [ERC-2612](https://eips.ethereum.org/EIPS/eip-2612)
- [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626)、[ERC-7540 异步扩展](https://eips.ethereum.org/EIPS/eip-7540) 与 [OpenZeppelin ERC-4626 指南](https://docs.openzeppelin.com/contracts/5.x/erc4626)
- [Morpho Vaults：ERC-4626 mechanics](https://docs.morpho.org/developers/earn/concepts/vault-mechanics/)
- [Yearn V3 Vault 实现](https://github.com/yearn/yearn-vaults-v3/blob/master/contracts/VaultV3.vy) 与 [Euler Vault Kit](https://docs.euler.finance/developers/evk/)
- [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)
