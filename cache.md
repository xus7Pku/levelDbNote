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
			pb0 := (*mBucket)(atomic.LoadPointer(&p.buckets[i]))
			if pb0 == nil {
				pb0 = p.initBucket(i)
			}
			// 不太明白啊
			pb1 := (*mBucket)(atomic.LoadPointer(&p.buckets[i+uint32(len(n.buckets))]))
			if pb1 == nil {
				pb1 = p.initBucket(i + uint32(len(n.buckets)))
			}
			// 冻结就是说当前这个不用了吧
			m0 := pb0.freeze()
			m1 := pb1.freeze()
			// Merge nodes.
			node = make([]*Node, 0, len(m0)+len(m1))
			node = append(node, m0...)
			node = append(node, m1...)
		}
		// 这就是新的 mBucket 了
		b := &mBucket{node: node}
		// 为什么原来是 nil 啊...？
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





