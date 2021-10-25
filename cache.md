包 cache 提供了缓存算法的接口和实现；

Cacher 是实现的接口，接口实现必须要并发安全；

Cache 中有许多个 mNode，mNode 中有许多个桶，桶中还有许多个 Node；

NewCache：

生成一个新的 cache map（似乎可以类比于 Go 的 map）；

cache 中 mHead 是一个 mNode；

mNode 中初始化 16 个 buckets，均是空 mBucket；

mask 用于二进制的掩码，初始值 15；

growThreshold 为扩容的界限；

cache 中 cacher 是可选的，cacher 是接口好像有很多方法；

Cache.getBucket(hash uint32)：

通过 hash 来获取相应的 mBucket；

mNode 是 cache.mHead；

hash & mask 来映射到相应的桶索引；

如果当前的桶不存在，那就在该位置上创建新的 mBucket；

Cache.delete(n *Node)：

要用 for 的无限循环，不知道是为什么？

根据 n.hash 获取到 mBucket；

运行 mBucket.delete 来进行删除；

Cache.Nodes() 和 Cache.Size() 都是包外可见的，返回 Node 的数量和占空间大小；

Cache.Capacity() 和 Cache.SetCapacity() 用于获取和设置 capacity；

Cache.Get(ns, key uint64, setFunc)：

Get 根据 namespace 和 key 获取相应的 cache Node，如果有就返回；

没有的话如果有 setFunc 就根据 setFunc 创建新的 Node；

返回的 cache Node Handle 在使用后必须要调用 Release 方法来释放；

给 Cache 增加 RLock；

对 ns 和 key 进行 hash 算法获得 hash 值，用于映射到 mBucket 数组的索引；

要用 for 进行循环，为什么？

通过 hash 获取 mBucket，并根据 mBucket 中的 Node 数组遍历获取相应的 Node；

若查到了 Node



包 cache 提供了缓存的接口和实现；

```go
// Cacher 是接口，提供缓存的一些功能
// 必须是并发安全的
// 好像是用户可以实现自己的缓存方式
// leveldb 帮助实现了简单的 lru 缓存
type Cacher interface {
    Capacity() int
    SetCapacity(capacity int)
    // 不太明白是什么意思，提升缓存节点？
    Promote(n *Node)
    Ban(n *Node)
    Evict(n *Node)
    EvictNS(ns uint64)
    EvictAll()
    Close() error
}
```

Value 是存储在缓存节点中的值：

```go
// Value 是缓存节点中的内容
// Value 实现 util.Releaser 接口
// Releaser 的 Release 方法会在释放的时候调用
type Value interface {}
```

NamespaceGetter

提供命名空间和对应缓存的封装；

```go
type NamespaceGetter interface {
    // 对应缓存
    Cache *Cache
    // 命名空间的标识
    NS uint64
}
```

对外可见的方法：

```go
func (g *NamespaceGetter) Get(key uint64, setFunc func() (size int, value Value)) *Handle  {
    // 对 g.Cache 的 Get 调用进行了封装
    return g.Cache.Get(g.NS, key, setFunc)
}
```

cache 实现了动态增长的非阻塞哈希表；

初始值：

```go
const (
    // Cache mNode 中 mBucket 的初始数量
	mInitialSize           = 1 << 4
    // 桶中 Node 的数量上界
	mOverflowThreshold     = 1 << 5
	mOverflowGrowThreshold = 1 << 7
)
```

mBucket 是缓存 mNode 中的桶，内部存放缓存节点，结构体对外不可见；

```go
type mBucket struct {
    // 保护 mBucket 的锁
    mu     sync.Mutex
    // 内部的 Node 数组
	node   []*Node
    // 桶是否被冻结弃用的标志
	frozen bool
}
```

mBucket 的方法：

```go
// 对 mBucket 进行冻结，可能是使其被弃用
// 不过为什么要返回 []*Nod，有点奇怪？
func (b *mBucket) freeze() []*Node {
	b.mu.Lock()
	defer b.mu.Unlock()
	if !b.frozen {
		b.frozen = true
	}
	return b.node
}
```

```go
// mBucket.get 取出对应的节点，并根据情况返回 done 和 added
// 在取点的过程中不会用到 r 和 h，这是在找不到节点且创建新节点的过程中会使用的
func (b *mBucket) get(r *Cache, h *mNode, hash uint32, ns, key uint64, noset bool) (done, added bool, n *Node) {
	b.mu.Lock()

	// 如果桶被冻结，就不再进行查找
	if b.frozen {
		b.mu.Unlock()
		return
	}

	// Scan the node.
	// 遍历桶中的节点
    // 在桶内的查找是直接进行顺序遍历和判断的
	for _, n := range b.node {
		if n.hash == hash && n.ns == ns && n.key == key {
			// 找到这个 node，就给这个节点的被引用数 n.ref +1
			atomic.AddInt32(&n.ref, 1)
			b.mu.Unlock()
			return true, false, n
		}
	}

	// Get only.
	// 不进行新节点的创建，根据 noset 来确定
	if noset {
		b.mu.Unlock()
		return true, false, nil
	}

	// Create node.
	// 创建新的节点，后续会根据 setFunc 加入相应的值
    // 初始化 ref 设置为 1
	n = &Node{
		r:    r,
		hash: hash,
		ns:   ns,
		key:  key,
		ref:  1,
	}
	// Add node to bucket.
	// 将新的 Node 加入到 mBucket 中
	b.node = append(b.node, n)
	bLen := len(b.node)
	b.mu.Unlock()

	// Update counter.
	// 为 r.nodes 的节点数量 +1 并判断是否超过了上限
    // h.growThreshold 是整个 mNode 的存储 Node 数量的上限
	grow := atomic.AddInt32(&r.nodes, 1) >= h.growThreshold
    
    // 如果当前 mBucket 的数量超过了单个桶的上限
	if bLen > mOverflowThreshold {
		// h.overflow 没有初始化
        // 在 h.overflow +1 且判断是否超过了 mOverflowGrowThreshold
        // 目测 h.overflow 是超过的数量
		grow = grow || atomic.AddInt32(&h.overflow, 1) >= mOverflowGrowThreshold
	}
    // 是否扩容还是看整体 mNode 中存储了多少数据（好像

	// Grow.
	// h.resizeInProgress 指的是是否在 resize，0 代表没有，1 代表正在进行
	if grow && atomic.CompareAndSwapInt32(&h.resizeInProgess, 0, 1) {
        // 两倍的 mBucket 数量
		nhLen := len(h.buckets) << 1
		nh := &mNode{
			buckets:         make([]unsafe.Pointer, nhLen),
			mask:            uint32(nhLen) - 1,
            // pred 指向扩容前的 mNode
			pred:            unsafe.Pointer(h),
			growThreshold:   int32(nhLen * mOverflowThreshold),
            // 缩容的界限为扩容前的 mBucket 大小
			shrinkThreshold: int32(nhLen >> 1),
		}
		// grow
        // 将新的 mNode 的地址赋值到 r.mHead 中，进行交换
        // 旧的 mNode 地址依然存储在新 mNode 的 pred 字段中
		ok := atomic.CompareAndSwapPointer(&r.mHead, unsafe.Pointer(h), unsafe.Pointer(nh))
		if !ok {
			panic("BUG: failed swapping head")
		}
        // 为新的 mNode 中的 mBucket 进行初始化
		go nh.initBuckets()
	}

	return true, true, n
}
```

```go
// mBucket.delete 删除相应的缓存节点，并考虑是否要进行缩容
// delete 应该是 Node.ref 为 0 后调用的方法吧...
func (b *mBucket) delete(r *Cache, h *mNode, hash uint32, ns, key uint64) (done, deleted bool) {
	b.mu.Lock()

	if b.frozen {
		b.mu.Unlock()
		return
	}

	// Scan the node.
	var (
		n    *Node
		bLen int
	)
    // 遍历 mBucket 中的 Node 数组
	for i := range b.node {
		n = b.node[i]
        // 这里都不管 hash 了，不是同一个人写的吧
		if n.ns == ns && n.key == key {
            // 如果 n.ref 理论上是 Node 的被引用次数为 0，说明啥？
			if atomic.LoadInt32(&n.ref) == 0 {
				deleted = true

				// Call releaser.
				if n.value != nil {
                    // 应该是调用 value 的 Release 方法来释放存储 value 的空间
					if r, ok := n.value.(util.Releaser); ok {
						r.Release()
					}
					n.value = nil
				}

				// Remove node from bucket.
                // 在 Node 数组中删除该 Node
				b.node = append(b.node[:i], b.node[i+1:]...)
				bLen = len(b.node)
			}
			break
		}
	}
	b.mu.Unlock()

	if deleted {
		// Call OnDel.
        // 节点删除的回调
		for _, f := range n.onDel {
			f()
		}

		// Update counter.
        // r 缓存整体的 size 调整
		atomic.AddInt32(&r.size, int32(n.size)*-1)
        // 根据缓存中的节点数量，进行是否要缩容的判断
		shrink := atomic.AddInt32(&r.nodes, -1) < h.shrinkThreshold
        // 不太明白 h.overflow 是用来做什么的...
		if bLen >= mOverflowThreshold {
			atomic.AddInt32(&h.overflow, -1)
		}

		// Shrink.
        // mNode 缩容的最小是 mInitialSize，以及是否在进行缩容的过程
		if shrink && len(h.buckets) > mInitialSize && atomic.CompareAndSwapInt32(&h.resizeInProgess, 0, 1) {
			// 缩减为原来的一半
            nhLen := len(h.buckets) >> 1
			nh := &mNode{
				buckets:         make([]unsafe.Pointer, nhLen),
				mask:            uint32(nhLen) - 1,
				pred:            unsafe.Pointer(h),
				growThreshold:   int32(nhLen * mOverflowThreshold),
				shrinkThreshold: int32(nhLen >> 1),
			}
            // 进行赋值
			ok := atomic.CompareAndSwapPointer(&r.mHead, unsafe.Pointer(h), unsafe.Pointer(nh))
			if !ok {
				panic("BUG: failed swapping head")
			}
			go nh.initBuckets()
		}
	}

	return true, deleted
}
```

mNode 结构体其实代表的是实际的缓存实体，里面包含了所有的 mBucket；

```go
type mNode struct {
    buckets []unsafe.Pointer // []*mBucket
	// mask 是 buckets 的长度 -1，用于 hash 计算得到后映射到 buckets 的下标
	mask uint32
	// pred 存储了扩缩容前的 mNode 地址
	pred            unsafe.Pointer // *mNode
    // 是否正在进行 resize
	resizeInProgess int32

    // overflow 真的看不太明白...
	overflow        int32
    // 扩容的界限，像是 Node 数量来决定
	growThreshold   int32
    // 缩容的界限，也是 Node 数量来决定
	shrinkThreshold int32
}
```

```go
// mNode.initBucket 对 i 下标的 mBucket 进行初始化
func (n *mNode) initBucket(i uint32) *mBucket {
    // 如果已经创建过了，ok
	if b := (*mBucket)(atomic.LoadPointer(&n.buckets[i])); b != nil {
		return b
	}

    // p 是当前 mNode 的上一个样子
	p := (*mNode)(atomic.LoadPointer(&n.pred))
    // 如果上一个存在的话
	if p != nil {
		var node []*Node
        // 通过直接比较当前的 mNode 和上一个 mNode 的大小来判断是扩容还是缩容
		if n.mask > p.mask {
			// Grow.
            // 找到之前的 i 对应的 mBucket？
			pb := (*mBucket)(atomic.LoadPointer(&p.buckets[i&p.mask]))
			if pb == nil {
				pb = p.initBucket(i & p.mask)
			}
            // 将 i 对应的以前的 mBucket 冻结并返回内部存储的 Nodes
			m := pb.freeze()
			// Split nodes.
            // 将被冻结 mBucket 的存储到新的 mBucket 中
            // 目测理论上 x.hash&m.mask == i 是必然成立的？（有疑问
			for _, x := range m {
				if x.hash&n.mask == i {
					node = append(node, x)
				}
			}
		} else {
			// Shrink.
            // 当前的 mBucket 是前任缩容得到的，意味着需要将前任的两个 mBucket 的 Node 都转移过来
			pb0 := (*mBucket)(atomic.LoadPointer(&p.buckets[i]))
			if pb0 == nil {
				pb0 = p.initBucket(i)
			}
            // i+uint32(len(n.buckets)) 是原来的缩容方法来的
			pb1 := (*mBucket)(atomic.LoadPointer(&p.buckets[i+uint32(len(n.buckets))]))
			if pb1 == nil {
				pb1 = p.initBucket(i + uint32(len(n.buckets)))
			}
			// 冻结
			m0 := pb0.freeze()
			m1 := pb1.freeze()
			// Merge nodes.
			node = make([]*Node, 0, len(m0)+len(m1))
			node = append(node, m0...)
			node = append(node, m1...)
		}
		// 这就是新的 mBucket 了
		b := &mBucket{node: node}
		// 为什么原来是 nil 啊...？新的 mBucket 可不就是 nil 嘛
		if atomic.CompareAndSwapPointer(&n.buckets[i], nil, unsafe.Pointer(b)) {
			if len(node) > mOverflowThreshold {
				atomic.AddInt32(&n.overflow, int32(len(node)-mOverflowThreshold))
			}
			return b
		}
	}

	return (*mBucket)(atomic.LoadPointer(&n.buckets[i]))
}
```

```go
// 初始化 mNode 的 buckets 数组
func (n *mNode) initBuckets() {
	for i := range n.buckets {
		n.initBucket(uint32(i))
	}
    // 初始化后，就可以把以前的 mNode 去掉了，因为内部的东西都转移了
	atomic.StorePointer(&n.pred, nil)
}
```

Cache 是包外可见的，代表了 cache map；

```go
type Cache struct {
    // 读写锁保护
	mu     sync.RWMutex
    // 实际上的桶数组
	mHead  unsafe.Pointer // *mNode
    // 内含 Node 数量
	nodes  int32
    // 内含缓存占用空间
	size   int32
    // 接口的实现
	cacher Cacher
	// Cache 是否已关闭
	closed bool
}
```

```go
// NewCache 是包外可见的
// 创建新的 Cache 并初始化
func NewCache(cacher Cacher) *Cache {
	h := &mNode{
        // 初始化 16 个桶
		buckets: make([]unsafe.Pointer, mInitialSize),
        // mask 掩码，又相当于桶的数量 -1
		mask:    mInitialSize - 1,
		// 扩容的界限
		growThreshold: int32(mInitialSize * mOverflowThreshold),
		// 缩容，初始化的 16 个桶是最小的缩容目标，不能再小了
		shrinkThreshold: 0,
	}
    // 空桶
	for i := range h.buckets {
		h.buckets[i] = unsafe.Pointer(&mBucket{})
	}
	r := &Cache{
		mHead:  unsafe.Pointer(h),
		cacher: cacher,
	}
	return r
}
```

```go
// getBucket 根据 hash 来获取桶
func (r *Cache) getBucket(hash uint32) (*mNode, *mBucket) {
	h := (*mNode)(atomic.LoadPointer(&r.mHead))
	// mask 是用来从 hash 映射到索引的
	i := hash & h.mask
	b := (*mBucket)(atomic.LoadPointer(&h.buckets[i]))
	if b == nil {
		// 如果当前 mBucket 不存在，就需要初始化一个桶
		b = h.initBucket(i)
	}
	return h, b
}
```

```go
// delete 删除某个 Node
func (r *Cache) delete(n *Node) bool {
	// 为什么要使用 for？可能 b.delete 没法一次就 done 吗
    // done 只有在 b 被冻结的时候才会为 false，难道说是在扩缩容的时候？
	for {
		h, b := r.getBucket(n.hash)
        // 调用 mBucket.delete 来进行 Node 的删除
		done, deleted := b.delete(r, h, n.hash, n.ns, n.key)
		if done {
			return deleted
		}
	}
}
```

```go
func (r *Cache) Nodes() int {
	return int(atomic.LoadInt32(&r.nodes))
}

func (r *Cache) Size() int {
	return int(atomic.LoadInt32(&r.size))
}

// 完全不知道 cacher 是做什么的...
func (r *Cache) Capacity() int {
	if r.cacher == nil {
		return 0
	}
	return r.cacher.Capacity()
}

func (r *Cache) SetCapacity(capacity int) {
	if r.cacher != nil {
		r.cacher.SetCapacity(capacity)
	}
}
```

```go
// Cache.Get 是包外可见的
// 根据 ns、key 来进行 Node 对应 Handle 的获取
// 如果 setFunc 有设置的话，且没有对应 Node，就根据 setFunc 来进行 Node 的创建和值的设置
func (r *Cache) Get(ns, key uint64, setFunc func() (size int, value Value)) *Handle {
	// 给整个 cache 加上读锁
	r.mu.RLock()
	defer r.mu.RUnlock()
	// 如果 cache 关闭了就算了
	if r.closed {
		return nil
	}

	// 利用 ns 和 key 进行 hash，hash 值用于对应到 mBucket 数组中的索引
	hash := murmur32(ns, key, 0xf00)

	// 为什么也是用 for
    // 感觉原因可能和 delete 的 for 一样，只有桶被冻结了才不会返回 true 的 done
	for {
		// 获取相应的存储位置
		h, b := r.getBucket(hash)
		// 从相应的 mBucket 中查询对应的 Node
		// 同时会给 Node 的引用数 +1，也就是说被一个 Handle 引用了？
		// done 是查询是否结束，n 是查到的 Node
		done, _, n := b.get(r, h, hash, ns, key, setFunc == nil)
		if done {
			// 查到 Node n
			if n != nil {
				n.mu.Lock()
				// 没有 value
				if n.value == nil {
					// 由于没有 setFunc，意味着 b.get 传入 noset 为 true，没有创建新的 Node
					if setFunc == nil {
						n.mu.Unlock()
						// 减少一个引用数，因为无值不会有对应的 Handle
						n.unref()
						return nil
					}

					// 如果是新建了 Node
					// 其实里面还没有填充 size 和 value 的，需要根据 setFunc 来进行
					n.size, n.value = setFunc()

					// 没有值，setFunc 也没有给值，不会返回 Handle，自然要去引用
					if n.value == nil {
						n.size = 0
						n.mu.Unlock()
						n.unref()
						return nil
					}
					// 将 Node 的 size 加到 cache 的 size 中去
					atomic.AddInt32(&r.size, int32(n.size))
				}
				n.mu.Unlock()
				if r.cacher != nil {
					// 提升节点？不太懂 cacher 是干啥的
					r.cacher.Promote(n)
				}
                // Handle 类似于对 Node 的处理器，防止外界对 Node 有不允许的改动
				return &Handle{unsafe.Pointer(n)}
			}

			break
		}
	}
	return nil
}
```

```go
// Cache.Delete 用于删除和禁用相关节点
// 被 ban 的节点永远不会再被加入到缓存树中
// ban 只是对于一个点的属性，新创建的该位置的节点并不会受到影响？
// onDel 会在节点不存在或被释放的时候调用
// Delete 会返回 true 再节点存在的时候
func (r *Cache) Delete(ns, key uint64, onDel func()) bool {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if r.closed {
		return false
	}

	hash := murmur32(ns, key, 0xf00)
	for {
		h, b := r.getBucket(hash)
		done, _, n := b.get(r, h, hash, ns, key, true)
		if done {
			// 查到了这个 Node
			if n != nil {
				if onDel != nil {
					n.mu.Lock()
                    // n 的 onDel 数组
					n.onDel = append(n.onDel, onDel)
					n.mu.Unlock()
				}
                // 用 cacher.Ban 对节点进行 ban 了
				if r.cacher != nil {
					r.cacher.Ban(n)
				}
				// 如果引用下降到 0 启动 delete
                // 初始化是 1
				n.unref()
				return true
			}

			break
		}
	}

    // 调用参数中的回调
	if onDel != nil {
		onDel()
	}

	return false
}
```

```go
// Cache.Evict 调用 Cacher.Evict
// 不太明白 Delete 和 Evict 有什么区别，只是调用的 Cacher 方法不同
func (r *Cache) Evict(ns, key uint64) bool {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if r.closed {
		return false
	}

	hash := murmur32(ns, key, 0xf00)
	for {
		h, b := r.getBucket(hash)
		done, _, n := b.get(r, h, hash, ns, key, true)
		if done {
			if n != nil {
				if r.cacher != nil {
					r.cacher.Evict(n)
				}
				// 这也要解除引用，不知道是啥意思...
				n.unref()
				return true
			}

			break
		}
	}

	return false
}
```

```go
func (r *Cache) EvictNS(ns uint64) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if r.closed {
		return
	}

	if r.cacher != nil {
		r.cacher.EvictNS(ns)
	}
}

func (r *Cache) EvictAll() {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if r.closed {
		return
	}

	if r.cacher != nil {
		r.cacher.EvictAll()
	}
}
```

```go
// Cache.Close() 关闭 Cache，且释放其中的所有 Node
func (r *Cache) Close() error {
	r.mu.Lock()
	if !r.closed {
		r.closed = true

		// mNode 就是存储的实际结构体
		h := (*mNode)(r.mHead)
        // 桶数组初始化？主要还是去掉 pred 吧，个人认为
		h.initBuckets()

		for i := range h.buckets {
			b := (*mBucket)(h.buckets[i])
			for _, n := range b.node {
				// Call releaser.
				if n.value != nil {
					// n.value 实现了 Releaser 的接口
					if r, ok := n.value.(util.Releaser); ok {
						r.Release()
					}
					n.value = nil
				}

				// Call OnDel.
				for _, f := range n.onDel {
					f()
				}
				n.onDel = nil
			}
		}
	}
	r.mu.Unlock()

	// Avoid deadlock.
    // 如何防止死锁的？调用了 Cacher 的 Close？
	if r.cacher != nil {
		if err := r.cacher.Close(); err != nil {
			return err
		}
	}
	return nil
}
```

```go
// Cache.CloseWeak() 不强制释放所有的 Node，只是进行 Evict？
func (r *Cache) CloseWeak() error {
	r.mu.Lock()
	if !r.closed {
		r.closed = true
	}
	r.mu.Unlock()

	// Avoid deadlock.
	if r.cacher != nil {
		r.cacher.EvictAll()
		if err := r.cacher.Close(); err != nil {
			return err
		}
	}
	return nil
}
```

Node 是缓存节点，也是包外可见的；

```go
type Node struct {
	// 所在的缓存
	r *Cache

	// hash 用来寻找 Node 所在的 mBucket 吧
	hash uint32
	// namespace 和 key
	ns, key uint64

	mu sync.Mutex
	// 占用空间大小
	size  int
    // 存储的值实体
	value Value

    // 引用的数量，为 0 时调用 delete 从桶中删除
	ref   int32
    // 释放删除的回调
	onDel []func()

    // Cacher 中可能会用到？
	CacheData unsafe.Pointer
}
```

```go
// NS returns this 'cache node' namespace.
func (n *Node) NS() uint64 {
	return n.ns
}

// Key returns this 'cache node' key.
func (n *Node) Key() uint64 {
	return n.key
}

// Size returns this 'cache node' size.
func (n *Node) Size() int {
	return n.size
}

// Value returns this 'cache node' value.
func (n *Node) Value() Value {
	return n.value
}

// Ref returns this 'cache node' ref counter.
// ref 的计数器，引用的计数器
func (n *Node) Ref() int32 {
	return atomic.LoadInt32(&n.ref)
}
```

```go
// Node.GetHandle 返回对应节点的 Handle
func (n *Node) GetHandle() *Handle {
	if atomic.AddInt32(&n.ref, 1) <= 1 {
		// 从零引用的缓存节点获取 Handle 需要报错，初始化为 1，0 的时候已经是要 delete 的了
		panic("BUG: Node.GetHandle on zero ref")
	}
	return &Handle{unsafe.Pointer(n)}
}
```

```go
// Node.unref 引用计数 -1
func (n *Node) unref() {
	// 如果引用全部去掉，就调用 cache 的 delete 方法删除
	// atomic.AddInt32 是先 -1 再返回减之后的数值
	if atomic.AddInt32(&n.ref, -1) == 0 {
		n.r.delete(n)
	}
}
```

```go
// 带读锁的引用计数 -1
// 可能是用到没有锁直接操作 Node 的地方？
func (n *Node) unrefLocked() {
	if atomic.AddInt32(&n.ref, -1) == 0 {
		n.r.mu.RLock()
		if !n.r.closed {
			n.r.delete(n)
		}
		n.r.mu.RUnlock()
	}
}
```

Handle 是缓存节点处理器，目测是为了封装 Node 防止做一些危险操作；

```go
type Handle struct {
	n unsafe.Pointer // *Node
}
```

```go
// Handle.Value() 获取 Node 的缓存值
func (h *Handle) Value() Value {
	n := (*Node)(atomic.LoadPointer(&h.n))
	if n != nil {
		return n.value
	}
	return nil
}
```

```go
// Handle.Release() 释放 Handle
func (h *Handle) Release() {
	nPtr := atomic.LoadPointer(&h.n)
	// 这里把 Handle 的 n 给置为 nil 了
	if nPtr != nil && atomic.CompareAndSwapPointer(&h.n, nPtr, nil) {
		n := (*Node)(nPtr)
        // 加锁的解除引用
		n.unrefLocked()
	}
}
```

Murmur32 是哈希算法；

```go
func murmur32(ns, key uint64, seed uint32) uint32 {
    //...
}
```



lru.go

lruNode 实现 lru 的 Node，并不是包外可见的，应该是对 Node 的封装了；

```go
type lruNode struct {
	n *Node
	// Node 对应的 Handle
	h *Handle
	// 是不是被禁用
	ban bool

	// 节点的前后指针
	next, prev *lruNode
}
```

```go
// lruNode.insert 是将 n 插入到 at 后面
// 节点被访问或新建后会被插入到 recent 后面作为最近被使用的节点
func (n *lruNode) insert(at *lruNode) {
	x := at.next
	at.next = n
	n.prev = at
	n.next = x
	x.prev = n
}
```

```go
// 移除节点 n
// 如果 n 的 prev 为 nil 说明已经移除了，不应该再访问到
func (n *lruNode) remove() {
	if n.prev != nil {
		n.prev.next = n.next
		n.next.prev = n.prev
		n.prev = nil
		n.next = nil
	} else {
		panic("BUG: removing removed node")
	}
}
```

lru 结构体是包外不可见的，但是实现了 Cacher 接口之后可以返回出去；

```go
type lru struct {
	mu sync.Mutex
	// 容量
	capacity int
	// 已使用的大小
	used int
	// 双向循环链表的最前面一个点，越往后越久未使用
    // 找最久未使用的就是 recent.prev 找到队末即可
	recent lruNode
}
```

```go
// lru.reset 重置 lru 对象；
func (r *lru) reset() {
    // 将双向链表清空
	r.recent.next = &r.recent
	r.recent.prev = &r.recent
    // 将已使用大小清零
	r.used = 0
}
```

```go
// lru.Capacity 返回容量大小
func (r *lru) Capacity() int {
	r.mu.Lock()
	defer r.mu.Unlock()
	return r.capacity
}
```

```go
// lru.SetCapacity 设置容量大小
func (r *lru) SetCapacity(capacity int) {
    // 要回收的 lruNode，就是将其 Handle 回收掉？
	var evicted []*lruNode

	r.mu.Lock()
	r.capacity = capacity
	for r.used > r.capacity {
        // 最久未使用的链表末尾节点
		rn := r.recent.prev
		if rn == nil {
			panic("BUG: invalid LRU used or capacity counter")
		}
        // 从 lru 双向链表中删除
		rn.remove()
		// 将 Node 对应的 lruNode 结构体清空，就是告诉 Node 其不属于 lru 链表了
		rn.n.CacheData = nil
		r.used -= rn.n.Size()
		// 要释放掉 Handle 的节点
		evicted = append(evicted, rn)
	}
	r.mu.Unlock()

	for _, rn := range evicted {
		rn.h.Release()
	}
}
```

```go
// lru.Promote 理论上应该是把 lruNode 提升到最近使用的那边，其他的方法都没有实现这个功能嘛
// 就是从原来的位置移除并插入到 recent 后面
func (r *lru) Promote(n *Node) {
	var evicted []*lruNode

	r.mu.Lock()
	// CacheData 是在 Node 里面存储的指向 lruNode 的指针
	// 如果 n.CacheData == nil 说明还没有被加入到 lru 的双向链表中
	if n.CacheData == nil {
        // 如果可以存进去
		if n.Size() <= r.capacity {
            // 新建 lruNode 存储着 Node
			rn := &lruNode{n: n, h: n.GetHandle()}
			// 插入新节点到 lru 的双向链表中
			rn.insert(&r.recent)
            // 在 Node 中存储 lruNode 的地址
			n.CacheData = unsafe.Pointer(rn)
			// 已占用空间++
			r.used += n.Size()

			// 如果超过了规定的容量的话，就需要删除只容量允许范围内
			for r.used > r.capacity {
				// recent.prev 就是链表末尾节点
				rn := r.recent.prev
				if rn == nil {
					panic("BUG: invalid LRU used or capacity counter")
				}
                // 链表移除
				rn.remove()
                // Node 的 lru 缓存信息删除，不对应到 lruNode 上
				rn.n.CacheData = nil
				r.used -= rn.n.Size()
                // 要释放 Handle 的数组
				evicted = append(evicted, rn)
			}
		}
	} else {
		// 已经在 lru 的双向链表中了
		rn := (*lruNode)(n.CacheData)
        // 该 lruNode 没有被禁用
		if !rn.ban {
			rn.remove()
			rn.insert(&r.recent)
		}
	}
	r.mu.Unlock()

	// 把 Handle Release 掉
	for _, rn := range evicted {
		rn.h.Release()
	}
}
```

```go
// lru.Ban 禁用掉某一 Node
func (r *lru) Ban(n *Node) {
	r.mu.Lock()
	// 如果 Node 还没有加入 lru
    // 直接创建 lruNode 且 ban 掉，且不加入 lru 的双向链表中
	if n.CacheData == nil {
		n.CacheData = unsafe.Pointer(&lruNode{n: n, ban: true})
	} else {
		// 如果 Node 已加入到 lru 双向链表中了
		rn := (*lruNode)(n.CacheData)
        // 如果该 lruNode 没有被禁用
        // ban 字段为 true 意味着对应 lruNode 从 lru 链表移除，占用空间移除，释放 Handle
        // 不过 lruNode 本身不会被释放，Node.CacheData 还是会保存指针
		if !rn.ban {
            // 移除
			rn.remove()
            // 设置 ban 字段为 true
			rn.ban = true
			r.used -= rn.n.Size()
			r.mu.Unlock()

            // 释放 Handle
			rn.h.Release()
			rn.h = nil
			return
		}
	}
	r.mu.Unlock()
}
```

```go
// lur.Evict 回收某一 Node
func (r *lru) Evict(n *Node) {
	r.mu.Lock()
    // 对应 lruNode
	rn := (*lruNode)(n.CacheData)
	if rn == nil || rn.ban {
		r.mu.Unlock()
		return
	}
    // 设置 Node.CacheData 为 nil，不管 lruNode 是否在 lru 中
	n.CacheData = nil
	r.mu.Unlock()

    // 释放 Handle
	rn.h.Release()
}
```

```go
// lru.EvictNS 回收某一 namespace 下的所有在 lru 中的 Node
// 这里回收做的事情和 Evict 还不太一样...是对 lru 双向链表中的回收
func (r *lru) EvictNS(ns uint64) {
	var evicted []*lruNode

	r.mu.Lock()
	for e := r.recent.prev; e != &r.recent; {
		rn := e
		e = e.prev
        // 找到所有对应 namespace 的点
		if rn.n.NS() == ns {
            // 移除
			rn.remove()
            // 设置 Node.CacheData 为 nil
			rn.n.CacheData = nil
            // 占用空间释放
			r.used -= rn.n.Size()
			evicted = append(evicted, rn)
		}
	}
	r.mu.Unlock()

	for _, rn := range evicted {
		rn.h.Release()
	}
}
```

```go
// lru.EvictAll 回收所有 Node
func (r *lru) EvictAll() {
	r.mu.Lock()
	back := r.recent.prev
	for rn := back; rn != &r.recent; rn = rn.prev {
		rn.n.CacheData = nil
	}
	r.reset()
	r.mu.Unlock()

	for rn := back; rn != &r.recent; rn = rn.prev {
		rn.h.Release()
	}
}
```

```go
// 这个甚至没有实现...
func (r *lru) Close() error {
	return nil
}
```

```go
func NewLRU(capacity int) Cacher {
	r := &lru{capacity: capacity}
	r.reset()
	return r
}
```



