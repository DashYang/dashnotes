# LeetCode 与现场 Coding

> 核验日期：2026-07-15
>
> 代码默认 Go 1.22+，省略题库已提供的 `ListNode` 等定义。每题统一关注：识别信号、核心不变量、实现、复杂度、易错点和变体。

## 主题简介：What / Why / How

- **What**：一组高频数据结构、图、字符串、并发和排序题型，而不是孤立题解集合。
- **Why**：面试主要评估建模、不变量、边界、复杂度和沟通；背代码无法应对变体。
- **How**：先识别模式，再写循环/状态不变量，选择数据结构，手算样例，最后编码并覆盖空值、重复值和溢出。

## 1. 现场 Coding 方法

### 1.1 开始前澄清

1. 输入规模、是否有序、是否允许重复、值域和溢出范围。
2. 是否允许修改输入；期望时间/空间复杂度。
3. 返回任意解还是稳定/字典序解；无解如何表示。
4. 并发题要问阻塞、取消、重复调用和 goroutine 生命周期。
5. 系统题要问线程安全、容量、过期、持久化和一致性语义。

### 1.2 写不变量

不变量是“每次循环开始/结束都成立的事实”。例如二分的目标始终在 `[lo, hi)`；LRU 链表从头到尾按最近使用排序；拓扑排序队列只放入当前入度为 0 的点。

推荐现场表达：

> 我选择半开区间。循环前答案若存在一定在 `[lo, hi)`；判断后丢弃不可能包含答案的一半；退出时 `lo == hi`，再验证是否命中。

### 1.3 收尾检查

- 空输入、单元素、全重复、已排序/逆序。
- 整数中点用 `lo + (hi-lo)/2`；乘法与总和是否需要 `int64`。
- map 查找是否区分不存在与零值。
- 递归深度是否可能 O(n)。
- goroutine/channel 是否有退出路径。

## 2. 原有经典题

### 2.1 LRU Cache — Hash Map + 双向链表

- **识别信号**：固定容量，`Get/Put` O(1)，淘汰最久未使用。
- **不变量**：map 指向唯一节点；链表从头到尾为新→旧；访问/更新后节点移到头，超容删除尾。

```go
type lruNode struct { key, val int; prev, next *lruNode }
type LRUCache struct { cap int; m map[int]*lruNode; head, tail *lruNode }

func Constructor(capacity int) LRUCache {
	h, t := &lruNode{}, &lruNode{}
	h.next, t.prev = t, h
	return LRUCache{cap: capacity, m: map[int]*lruNode{}, head: h, tail: t}
}
func (c *LRUCache) remove(n *lruNode) { n.prev.next, n.next.prev = n.next, n.prev }
func (c *LRUCache) front(n *lruNode) {
	n.next, n.prev = c.head.next, c.head
	c.head.next.prev, c.head.next = n, n
}
func (c *LRUCache) Get(key int) int {
	n, ok := c.m[key]; if !ok { return -1 }
	c.remove(n); c.front(n); return n.val
}
func (c *LRUCache) Put(key, value int) {
	if n, ok := c.m[key]; ok { n.val = value; c.remove(n); c.front(n); return }
	n := &lruNode{key: key, val: value}; c.m[key] = n; c.front(n)
	if len(c.m) > c.cap { old := c.tail.prev; c.remove(old); delete(c.m, old.key) }
}
```

- **复杂度**：`Get/Put` O(1)，空间 O(capacity)。
- **易错点**：更新已有 key 不增加长度；容量 0；删除链表时同步删 map。
- **变体**：带 TTL、并发安全、按权重容量；[LC 146](https://leetcode.com/problems/lru-cache/)。

### 2.2 LFU Cache — 频率桶

- **识别信号**：淘汰最少使用，同频率淘汰最久未使用。
- **不变量**：每个 key 位于其频率桶；每个桶内部是 LRU；`minFreq` 指向非空最小频率。

```go
type LFUCache struct {
	cap, min int
	value, freq map[int]int
	groups map[int]*list.List
	where map[int]*list.Element
}
type lfuEntry struct{ key int }

func NewLFU(capacity int) *LFUCache {
	return &LFUCache{cap: capacity, value: map[int]int{}, freq: map[int]int{},
		groups: map[int]*list.List{}, where: map[int]*list.Element{}}
}
func (c *LFUCache) touch(key int) {
	f := c.freq[key]; c.groups[f].Remove(c.where[key])
	if c.groups[f].Len() == 0 && c.min == f { c.min++ }
	c.freq[key] = f + 1
	if c.groups[f+1] == nil { c.groups[f+1] = list.New() }
	c.where[key] = c.groups[f+1].PushFront(lfuEntry{key})
}
func (c *LFUCache) Get(key int) int {
	v, ok := c.value[key]; if !ok { return -1 }; c.touch(key); return v
}
func (c *LFUCache) Put(key, val int) {
	if c.cap == 0 { return }
	if _, ok := c.value[key]; ok { c.value[key] = val; c.touch(key); return }
	if len(c.value) == c.cap {
		e := c.groups[c.min].Back(); key0 := e.Value.(lfuEntry).key
		c.groups[c.min].Remove(e); delete(c.value, key0); delete(c.freq, key0); delete(c.where, key0)
	}
	c.value[key], c.freq[key], c.min = val, 1, 1
	if c.groups[1] == nil { c.groups[1] = list.New() }
	c.where[key] = c.groups[1].PushFront(lfuEntry{key})
}
```

需 `import "container/list"`。复杂度 `Get/Put` O(1)，空间 O(capacity)。易错点是同频 LRU、空桶与 `minFreq` 更新。变体：频率衰减、带 TTL；[LC 460](https://leetcode.com/problems/lfu-cache/)。

### 2.3 Logger Rate Limiter — 时间窗口

- **识别信号**：相同 key 在固定秒数内只允许一次。
- **不变量**：map 保存每个 message 下一次允许时间。

```go
type Logger struct{ next map[string]int }
func NewLogger() Logger { return Logger{next: map[string]int{}} }
func (l *Logger) ShouldPrintMessage(ts int, msg string) bool {
	if ts < l.next[msg] { return false }
	l.next[msg] = ts + 10
	return true
}
```

- **复杂度**：均摊 O(1)，空间 O(不同消息数)。
- **易错点**：题目时间戳单调时才可简单清理；真实系统需要并发保护、TTL、分布式一致性。
- **变体**：滑动窗口、token bucket、按租户限流；[LC 359](https://leetcode.com/problems/logger-rate-limiter/)。

### 2.4 Task Scheduler — 贪心计数

- **识别信号**：相同任务间至少间隔 `n`，求最短总时间。
- **不变量**：最高频任务形成骨架；其他任务填空，答案至少为任务总数。

```go
func leastInterval(tasks []byte, n int) int {
	var cnt [26]int
	for _, t := range tasks { cnt[t-'A']++ }
	maxF, numMax := 0, 0
	for _, c := range cnt {
		if c > maxF { maxF, numMax = c, 1 } else if c == maxF { numMax++ }
	}
	frame := (maxF-1)*(n+1) + numMax
	if frame < len(tasks) { return len(tasks) }
	return frame
}
```

- **复杂度**：O(N+alphabet)，空间 O(alphabet)。
- **易错点**：多个最高频任务；`n=0`；若任务耗时/冷却不同，公式失效。
- **变体**：输出具体调度用 max-heap + cooldown queue；[LC 621](https://leetcode.com/problems/task-scheduler/)。

### 2.5 Course Schedule — 拓扑排序 / DAG

- **识别信号**：先修依赖、能否完成全部节点，即有向图判环。
- **不变量**：队列中只有入度 0 节点；弹出节点等价于删除它和出边。

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
	g := make([][]int, numCourses); indeg := make([]int, numCourses)
	for _, p := range prerequisites { g[p[1]] = append(g[p[1]], p[0]); indeg[p[0]]++ }
	q := make([]int, 0)
	for i, d := range indeg { if d == 0 { q = append(q, i) } }
	seen := 0
	for len(q) > 0 {
		u := q[0]; q = q[1:]; seen++
		for _, v := range g[u] { indeg[v]--; if indeg[v] == 0 { q = append(q, v) } }
	}
	return seen == numCourses
}
```

- **复杂度**：O(V+E)，空间 O(V+E)。
- **易错点**：边方向、重复边、孤立节点；Go 中用 head index 可避免反复切片保留大底层数组。
- **变体**：输出顺序、DFS 三色判环、并行批次；[LC 207](https://leetcode.com/problems/course-schedule/)。

### 2.6 Decode String — 栈 / 递归下降

- **识别信号**：`k[encoded]`、括号嵌套。
- **不变量**：遇到 `[` 保存外层字符串和重复次数；`]` 恢复一层。

```go
func decodeString(s string) string {
	counts := []int{}; parts := [][]byte{}
	cur := []byte{}; num := 0
	for _, ch := range s {
		switch {
		case ch >= '0' && ch <= '9': num = num*10 + int(ch-'0')
		case ch == '[':
			counts = append(counts, num); parts = append(parts, cur); num = 0; cur = []byte{}
		case ch == ']':
			k := counts[len(counts)-1]; counts = counts[:len(counts)-1]
			prev := parts[len(parts)-1]; parts = parts[:len(parts)-1]
			piece := cur; cur = prev
			for i := 0; i < k; i++ { cur = append(cur, piece...) }
		default: cur = append(cur, string(ch)...)
		}
	}
	return string(cur)
}
```

时间和输出空间均 O(展开后长度)。易错点：多位数字、嵌套、错误复制非零 `strings.Builder`；使用 `[]byte` 可避免 builder copy 语义。变体：表达式解析；[LC 394](https://leetcode.com/problems/decode-string/)。

### 2.7 Trie — 前缀树

- **识别信号**：大量字符串插入、精确查找和前缀查找。
- **不变量**：从根到节点的边拼出前缀；`end` 区分完整单词与前缀。

```go
type Trie struct { next [26]*Trie; end bool }
func (t *Trie) Insert(word string) {
	p := t
	for _, ch := range word { i := ch-'a'; if p.next[i] == nil { p.next[i] = &Trie{} }; p = p.next[i] }
	p.end = true
}
func (t *Trie) walk(s string) *Trie {
	p := t
	for _, ch := range s { p = p.next[ch-'a']; if p == nil { return nil } }
	return p
}
func (t *Trie) Search(word string) bool { p := t.walk(word); return p != nil && p.end }
func (t *Trie) StartsWith(prefix string) bool { return t.walk(prefix) != nil }
```

- **复杂度**：O(L) 时间；空间 O(总字符数 × 分支表示)。
- **易错点**：字符集不只 a-z 时数组不可用；删除要处理共享前缀；Unicode 按 rune 还是 byte 要明确。
- **变体**：压缩 Trie、通配符、自动补全；[LC 208](https://leetcode.com/problems/implement-trie-prefix-tree/)。

### 2.8 并发顺序题 — Channel 作为一次性信号

- **识别信号**：多个 goroutine 必须按阶段执行。
- **不变量**：关闭 `firstDone` 前 Second 不能继续；关闭 `secondDone` 前 Third 不能继续。

```go
type Foo struct{ firstDone, secondDone chan struct{} }
func NewFoo() *Foo { return &Foo{make(chan struct{}), make(chan struct{})} }
func (f *Foo) First(printFirst func()) { printFirst(); close(f.firstDone) }
func (f *Foo) Second(printSecond func()) { <-f.firstDone; printSecond(); close(f.secondDone) }
func (f *Foo) Third(printThird func()) { <-f.secondDone; printThird() }
```

交替打印用 token channel；channel 中有 token 的角色才允许输出：

```go
func printFooBar(n int, printFoo, printBar func()) {
	foo, bar := make(chan struct{}, 1), make(chan struct{}, 1)
	foo <- struct{}{}
	var wg sync.WaitGroup; wg.Add(2)
	go func() { defer wg.Done(); for i := 0; i < n; i++ { <-foo; printFoo(); bar <- struct{}{} } }()
	go func() { defer wg.Done(); for i := 0; i < n; i++ { <-bar; printBar(); if i+1 < n { foo <- struct{}{} } } }()
	wg.Wait()
}

func printZeroEvenOdd(n int, printNumber func(int)) {
	zero, odd, even := make(chan struct{}, 1), make(chan struct{}, 1), make(chan struct{}, 1)
	zero <- struct{}{}
	var wg sync.WaitGroup; wg.Add(3)
	go func() { defer wg.Done(); for x := 1; x <= n; x++ { <-zero; printNumber(0); if x%2 == 1 { odd <- struct{}{} } else { even <- struct{}{} } } }()
	go func() { defer wg.Done(); for x := 1; x <= n; x += 2 { <-odd; printNumber(x); if x < n { zero <- struct{}{} } } }()
	go func() { defer wg.Done(); for x := 2; x <= n; x += 2 { <-even; printNumber(x); if x < n { zero <- struct{}{} } } }()
	wg.Wait()
}
```

需 `import "sync"`。复杂度 O(n)，额外空间 O(1)，等待不 busy-spin。易错点：最后一次打印后不应再发送无人接收的 token；方法若可能重复调用，close 会 panic；回调 panic 会让后续永久阻塞；生产代码需 context/error propagation。对应题目：[Print in Order](https://leetcode.com/problems/print-in-order/)、[Print FooBar Alternately](https://leetcode.com/problems/print-foobar-alternately/)、[Print Zero Even Odd](https://leetcode.com/problems/print-zero-even-odd/)。

### 2.9 Merkle Root 与 Merkle Proof

- **识别信号**：对集合做短承诺，证明单个 leaf 包含于 root。
- **不变量**：每层将相邻 hash 按约定顺序组合；proof 每层提供兄弟 hash 和左右方向。

```go
type ProofStep struct { Sibling [32]byte; SiblingOnLeft bool }
func hashPair(a, b [32]byte) [32]byte {
	buf := make([]byte, 0, 64); buf = append(buf, a[:]...); buf = append(buf, b[:]...)
	return sha256.Sum256(buf)
}
func VerifyMerkle(leaf []byte, proof []ProofStep, root [32]byte) bool {
	h := sha256.Sum256(append([]byte{0}, leaf...)) // leaf domain separation
	for _, p := range proof {
		if p.SiblingOnLeft { h = hashPair(p.Sibling, h) } else { h = hashPair(h, p.Sibling) }
	}
	return h == root
}
```

需 `import "crypto/sha256"`。验证时间/证明空间 O(log N)，建树 O(N)。易错点：奇数 leaf 规则、左右方向、leaf/internal domain separation、序列化歧义。Ethereum 使用 Keccak/MPT/RLP，不应把此二叉树模板直接称为 Ethereum state proof。变体：sparse Merkle tree、Merkle Patricia Trie、multiproof。

## 3. 二分查找专题

### 3.1 统一模板：半开区间 `[lo, hi)`

```go
// 返回第一个 >= target 的位置，若都小于则返回 len(nums)。
func lowerBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target { lo = mid + 1 } else { hi = mid }
	}
	return lo
}
func upperBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target { lo = mid + 1 } else { hi = mid }
	}
	return lo
}
```

- **不变量**：`[0, lo)` 全部 `< target`，`[hi, n)` 全部 `>= target`；未知区间为 `[lo, hi)`。
- **复杂度**：O(log N) 时间，O(1) 空间。
- **边界**：不要在闭区间模板中混用 `hi=mid`；中点防溢出；退出后验证索引。

### 3.2 标准二分与左右边界

标准查找：`i := lowerBound(nums, target)`，若 `i<n && nums[i]==target` 则命中。左右范围：`left=lowerBound(target)`，`right=upperBound(target)-1`。

- **识别信号**：有序数组、找第一个/最后一个满足条件。
- **变体**：[LC 704](https://leetcode.com/problems/binary-search/)、[LC 34](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)。

### 3.3 搜索旋转数组

```go
func searchRotated(a []int, target int) int {
	lo, hi := 0, len(a)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if a[mid] == target { return mid }
		if a[lo] <= a[mid] { // 左半有序
			if a[lo] <= target && target < a[mid] { hi = mid-1 } else { lo = mid+1 }
		} else { // 右半有序
			if a[mid] < target && target <= a[hi] { lo = mid+1 } else { hi = mid-1 }
		}
	}
	return -1
}
```

不变量是每轮至少一半有序，据 target 是否落在有序值域丢弃另一半。O(log N)，但允许大量重复时无法总判断哪半有序，最坏退化 O(N)。[LC 33](https://leetcode.com/problems/search-in-rotated-sorted-array/)。

### 3.4 旋转数组最小值与峰值

```go
func findMin(a []int) int {
	lo, hi := 0, len(a)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if a[mid] > a[hi] { lo = mid+1 } else { hi = mid }
	}
	return a[lo]
}
func findPeak(a []int) int {
	lo, hi := 0, len(a)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if a[mid] < a[mid+1] { lo = mid+1 } else { hi = mid }
	}
	return lo
}
```

两题均 O(log N)/O(1)。`findMin` 用右端值判断 pivot 所在侧；重复值版本遇到相等只能 `hi--`。峰值利用局部坡度保证上坡方向存在峰，`mid+1` 安全来自 `lo<hi`。[LC 153](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)、[LC 162](https://leetcode.com/problems/find-peak-element/)。

### 3.5 对答案二分

当答案空间单调可判定，例如“速度为 x 能否在 H 小时内完成”，在值域上二分最小可行值。

```go
func minEatingSpeed(piles []int, h int) int {
	lo, hi := 1, 0
	for _, p := range piles { if p > hi { hi = p } }
	for lo < hi {
		mid := lo + (hi-lo)/2
		var hours int64
		for _, p := range piles { hours += int64((p + mid - 1) / mid) }
		if hours <= int64(h) { hi = mid } else { lo = mid+1 }
	}
	return lo
}
```

复杂度 O(N log M)，M 为答案值域。易错点：先证明 predicate 单调；累计用 `int64`；上下界必须包含答案。[LC 875](https://leetcode.com/problems/koko-eating-bananas/)。

## 4. 快速排序专题

### 4.1 三路随机快排

对重复值多的输入，三路 partition 将区间分为 `< pivot`、`== pivot`、`> pivot`。

```go
func quickSort(a []int) {
	var sortRange func(int, int)
	sortRange = func(lo, hi int) {
		for lo < hi { // 尾递归消除：继续处理较大侧
			p := lo + rand.Intn(hi-lo+1)
			a[lo], a[p] = a[p], a[lo]; pivot := a[lo]
			lt, i, gt := lo, lo+1, hi
			for i <= gt {
				if a[i] < pivot { a[lt], a[i] = a[i], a[lt]; lt++; i++
				} else if a[i] > pivot { a[i], a[gt] = a[gt], a[i]; gt--
				} else { i++ }
			}
			if lt-lo < hi-gt { sortRange(lo, lt-1); lo = gt+1
			} else { sortRange(gt+1, hi); hi = lt-1 }
		}
	}
	if len(a) > 1 { sortRange(0, len(a)-1) }
}
```

需 `import "math/rand"`。partition 不变量：`[lo,lt)<p`、`[lt,i)==p`、`(gt,hi]>p`、`[i,gt]` 未分类。期望 O(N log N)，最坏 O(N²)；尾递归处理让额外栈期望/受控为 O(log N)。生产排序通常采用 introsort/pdqsort 等混合算法，不应手写替代标准库。

### 4.2 Lomuto 与 Hoare

- **Lomuto**：扫描边界简单，pivot 最终归位；交换多，在有序/重复输入上表现差。
- **Hoare**：双指针向内，交换少；返回的是分割边界，pivot 未必在最终位置，递归区间容易写错。
- **随机 pivot**：让输入难以稳定触发最坏情况，但不改变理论最坏复杂度。
- **小区间优化**：低于阈值改 insertion sort，减少递归与分支开销。

### 4.3 Quickselect / 第 K 大

```go
func findKthLargest(a []int, k int) int {
	target, lo, hi := len(a)-k, 0, len(a)-1
	for {
		p := lo + rand.Intn(hi-lo+1); a[p], a[hi] = a[hi], a[p]
		j := lo
		for i := lo; i < hi; i++ { if a[i] <= a[hi] { a[i], a[j] = a[j], a[i]; j++ } }
		a[j], a[hi] = a[hi], a[j]
		if j == target { return a[j] }; if j < target { lo = j+1 } else { hi = j-1 }
	}
}
```

期望 O(N)，最坏 O(N²)，原地 O(1)。易错点：第 K 大对应升序下标 `n-k`；partition 会修改输入；稳定最坏线性需要 median-of-medians。[LC 215](https://leetcode.com/problems/kth-largest-element-in-an-array/)。

### 4.4 Top K Frequent

```go
func topKFrequent(nums []int, k int) []int {
	freq := map[int]int{}
	for _, x := range nums { freq[x]++ }
	buckets := make([][]int, len(nums)+1)
	for x, f := range freq { buckets[f] = append(buckets[f], x) }
	ans := make([]int, 0, k)
	for f := len(buckets)-1; f >= 0 && len(ans) < k; f-- {
		for _, x := range buckets[f] { ans = append(ans, x); if len(ans) == k { break } }
	}
	return ans
}
```

Bucket 解 O(N) 时间/空间；若唯一值 U 很大且 k 小，可用大小 k 的 min-heap，O(U log k)。题目未要求同频稳定顺序，若要求需额外 tie-break。[LC 347](https://leetcode.com/problems/top-k-frequent-elements/)。

### 4.5 Sort Colors — 三路 Partition

```go
func sortColors(a []int) {
	lo, i, hi := 0, 0, len(a)-1
	for i <= hi {
		switch a[i] {
		case 0: a[lo], a[i] = a[i], a[lo]; lo++; i++
		case 2: a[i], a[hi] = a[hi], a[i]; hi--
		default: i++
		}
	}
}
```

O(N)/O(1)。交换 2 后不能 `i++`，因为换来的值尚未分类。不变量与三路快排相同。[LC 75](https://leetcode.com/problems/sort-colors/)。完整排序题见 [LC 912](https://leetcode.com/problems/sort-an-array/)。

## 5. 建议扩展的基础模式

### 5.1 链表与双指针

- 快慢指针：环检测、中点、倒数第 K。
- dummy head：统一删除头节点等边界。
- 原地反转不变量：`prev` 已反转，`cur` 为未处理头。
- 推荐：[LC 206](https://leetcode.com/problems/reverse-linked-list/)、[LC 141](https://leetcode.com/problems/linked-list-cycle/)、[LC 19](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)。

### 5.2 栈、队列与单调结构

- 括号/嵌套用栈；层序和无权最短路用队列。
- 单调栈维护尚未找到下一个更大/更小元素的候选。
- 单调队列维护窗口极值，队首始终是当前最优且未过期下标。
- 推荐：[LC 20](https://leetcode.com/problems/valid-parentheses/)、[LC 739](https://leetcode.com/problems/daily-temperatures/)、[LC 239](https://leetcode.com/problems/sliding-window-maximum/)。

### 5.3 BFS / DFS

- DFS 适合遍历、连通分量、回溯；BFS 适合单位边权最短步数。
- 图题要区分“本轮 visiting”和“全局 visited”；网格先定义方向和边界。
- 深图递归可能栈溢出，改显式 stack。
- 推荐：[LC 200](https://leetcode.com/problems/number-of-islands/)、[LC 102](https://leetcode.com/problems/binary-tree-level-order-traversal/)、[LC 127](https://leetcode.com/problems/word-ladder/)。

### 5.4 Heap / Top K

维护最大的 K 个用 min-heap，堆顶是当前集合淘汰门槛；维护最小 K 个反之。合并 K 路有序流时 heap 保存每路当前头。

推荐：[LC 23](https://leetcode.com/problems/merge-k-sorted-lists/)、[LC 295](https://leetcode.com/problems/find-median-from-data-stream/)。Go 使用 `container/heap`，实现 `Len/Less/Swap/Push/Pop` 时注意 `Push/Pop` 是指针 receiver。

### 5.5 滑动窗口

窗口适用于连续子数组/子串，右指针扩张，条件不满足时左指针收缩。关键是证明窗口条件具有单调性；含负数的“和至少 K”通常不能用普通双指针。

推荐：[LC 3](https://leetcode.com/problems/longest-substring-without-repeating-characters/)、[LC 76](https://leetcode.com/problems/minimum-window-substring/)、[LC 209](https://leetcode.com/problems/minimum-size-subarray-sum/)。

### 5.6 基础动态规划

回答四步：状态定义、转移、初值、遍历顺序。先写二维正确版本，再判断是否能滚动压缩；压缩后遍历方向取决于是否允许当前轮重复使用状态。

推荐：[LC 70](https://leetcode.com/problems/climbing-stairs/)、[LC 198](https://leetcode.com/problems/house-robber/)、[LC 322](https://leetcode.com/problems/coin-change/)、[LC 300](https://leetcode.com/problems/longest-increasing-subsequence/)。

## 6. 高频追问与答题骨架

1. **为什么是这个数据结构？** 先列所需操作复杂度，再映射到 map/list/heap/trie，而不是说“这题大家都这么写”。
2. **如何证明正确？** 给循环不变量、初始化成立、每轮保持、退出推导结果。
3. **最坏复杂度？** 区分快排期望/最坏、哈希均摊/攻击输入、递归栈和输出空间。
4. **如何应对数据量扩大？** 外排、分片、流式 heap、近似算法；先说明单机边界。
5. **如何并发安全？** 定义 linearization point、锁粒度、取消/关闭和 race test；不要只在所有方法外加一把锁就结束分析。
6. **如何测试？** 表驱动边界 + 随机 differential test + property（排序后有序且 multiset 不变）+ race/benchmark。

## 7. 容易混淆或说错的点

- O(1) map 操作通常是平均/均摊，不是所有输入严格最坏 O(1)。
- 二分不是只能搜数组；只要答案域有单调 predicate 就能使用。
- 快排不是稳定排序，随机 pivot 也没有消除 O(N²) 最坏情况。
- BFS 只有在边权一致时直接给最短路；0/1 权重和一般权重需不同算法。
- Channel 题的“输出正确”不等于没有 goroutine leak 或重复 close。
- Merkle proof 必须绑定 hash/编码/左右顺序和可信 root。
- LRU/LFU 的 O(1) 依赖哈希表和双向链表/频率桶，不包含并发锁竞争和 TTL 清理成本。
- 复杂度要说明 N 的定义；展开字符串题的时间下界由输出长度决定。

## 8. 复习顺序

1. 先手写 `lowerBound`、三路 partition、BFS/DFS、heap 接口。
2. 再练 LRU、拓扑排序、滑动窗口和基础 DP，重点口述不变量。
3. 最后练 LFU、并发顺序、Merkle proof 和系统型变体。
4. 每题在纸上覆盖：空、单元素、重复、极值、恶意输入，并用 2 分钟说明复杂度。

## 9. 官方资料与延伸阅读

- [LeetCode Problemset](https://leetcode.com/problemset/)
- [Go `container/list`](https://pkg.go.dev/container/list) 与 [Go `container/heap`](https://pkg.go.dev/container/heap)
- [Go `sync` package](https://pkg.go.dev/sync)
- [Go Memory Model](https://go.dev/ref/mem)
- [The Algorithms — Go](https://github.com/TheAlgorithms/Go)（社区实现，仅用于对照，不替代自行推导）
