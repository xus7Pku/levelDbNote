包 memdb 对内存键值数据库进行了实现；

DB 中跳表如何实现：

*   DB 中跳表并不是以链表的形式存储的；
*   键值数据的 kv 信息，以追加的方式存储在 kvData 切片中；
*   跳表的逻辑信息存储在 nodeData 中，各节点对应的索引是该节点逻辑信息存储的开始；
    *   顺次存储；
    *   [0] 节点 kv 在 kvData 中的偏移信息、[1] 节点 key 的长度、[2] 节点 value 的长度，通过三者可以对应到 kvData 中该节点存储的具体 key 和 value；
    *   [3] 节点层高 Height，[4...15] 是不同层下一节点的索引，在层高以上都是 0（无下一节点）；
    *   nodeData 也是追加的，物理存储顺序和跳表本身节点逻辑无关；
*   删除节点时，只在 nodeData 中修改跳表索引的逻辑关系，不会进行物理的删除，但这种软删除后无法再次访问已删除信息；
*   新建节点时，在 kvData 和 nodeData 都已追加的方式存储，并修改 nodeData 前后节点的链接指向逻辑关系；
*   修改节点 value 时，在 nodeData 是在原处修改的，不用进行逻辑关系的变化，但在 kvData 是追加的，原先的 kv 存储位置就失效不再访问了；



```go
// 通用的错误
var (
	ErrNotFound     = errors.ErrNotFound
	ErrIterReleased = errors.New("leveldb/memdb: iterator released")
)
```

```go
// 跳表中可达的最大层数
const tMaxHeight = 12
```

```go
// DB 迭代器的结构体
type dbIter struct {
	util.BasicReleaser
    // 迭代器所在的 DB 主体
	p          *DB
    // key 范围
	slice      *util.Range
    // 当前 node 在 p.nodeData 的索引
	node       int
    // 标志方向，前、后
	forward    bool
    // 当前 node 的 key 和 value
	key, value []byte
    // 过程中出现的错误
	err        error
}

// dbIter.fill 用于根据 i.node 填充内部关于 node 信息的字段
// checkStart 和 checkLimit 指示是否要进行前后范围的检查
func (i *dbIter) fill(checkStart, checkLimit bool) bool {
	if i.node != 0 {
		// n 为 i.node 在 kvData 的偏移信息
		n := i.p.nodeData[i.node]
		// n:m 是 i.node 的 key 在 keyData 的位置
		m := n + i.p.nodeData[i.node+nKey]
		// 填充 i.key
		i.key = i.p.kvData[n:m]
		// 检测 i.slice 是否合法
		if i.slice != nil {
			switch {
			case checkLimit && i.slice.Limit != nil && i.p.cmp.Compare(i.key, i.slice.Limit) >= 0:
				fallthrough
			case checkStart && i.slice.Start != nil && i.p.cmp.Compare(i.key, i.slice.Start) < 0:
				i.node = 0
				goto bail
			}
		}
        // 填充 i.value
		i.value = i.p.kvData[m : m+i.p.nodeData[i.node+nVal]]
		return true
	}
bail:
	// i.slice 不合法就 GG
	i.key = nil
	i.value = nil
	return false
}

// dbIter.Valid 返回 i.node 是否有值
func (i *dbIter) Valid() bool {
	return i.node != 0
}

// dbIter.First 根据 i.slice.Start 获取第一个节点存入 i.node 并填充其他字段
// 若 i.slice.Start 没有设置，就直接返回 nodeData 中按插入顺序的最前面那个节点
func (i *dbIter) First() bool {
	// 若 i 已经调用过 Release 了
	if i.Released() {
		i.err = ErrIterReleased
		return false
	}

	// i.forward 指示方向
	i.forward = true
    // 加锁
	i.p.mu.RLock()
	defer i.p.mu.RUnlock()
    // 若 i.slice.Start 有设定，根据 Start 进行查询
	if i.slice != nil && i.slice.Start != nil {
		i.node, _ = i.p.findGE(i.slice.Start, false)
	} else { // 否则返回 nodeData 中最前面的那个？我咋感觉第一个跳过了...吗？
		i.node = i.p.nodeData[nNext]
	}
	return i.fill(false, true)
}

// dbIter.Last 根据 i.sliceLimit 获取最后一个节点存入 i.node 并填充其他字段
// 若 i.slice.Limit 没有设置，就直接返回 nodeData 中按插入顺序的最后一个节点
func (i *dbIter) Last() bool {
    // 若 i 已经释放
	if i.Released() {
		i.err = ErrIterReleased
		return false
	}

    // 没有下一个？还是用来指示方向？
	i.forward = false
	i.p.mu.RLock()
	defer i.p.mu.RUnlock()
    // 若 i.slice.Limit 有设定，根据 Limit 进行查询
	if i.slice != nil && i.slice.Limit != nil {
		i.node = i.p.findLT(i.slice.Limit)
	} else { // 否则返回 findLast 的 node 索引
		i.node = i.p.findLast()
	}
	return i.fill(true, false)
}

// dbIter.Seek 找迭代器中比给定 key 大的 node 并填充字段
func (i *dbIter) Seek(key []byte) bool {
	if i.Released() {
		i.err = ErrIterReleased
		return false
	}

	i.forward = true
	i.p.mu.RLock()
	defer i.p.mu.RUnlock()
	if i.slice != nil && i.slice.Start != nil && i.p.cmp.Compare(key, i.slice.Start) < 0 {
		key = i.slice.Start
	}
	i.node, _ = i.p.findGE(key, false)
	return i.fill(false, true)
}

// dbIter.Next 找到当前 node 的下一个节点并填充到 i 当中
func (i *dbIter) Next() bool {
	if i.Released() {
		i.err = ErrIterReleased
		return false
	}

	// 若当前没有指向节点，需要进行第一个节点的获取
	if i.node == 0 {
        // 如果 forward 是 false，说明 Next 是第一个点
		if !i.forward {
			return i.First()
		}
		return false
	}
	i.forward = true
	i.p.mu.RLock()
	defer i.p.mu.RUnlock()
	i.node = i.p.nodeData[i.node+nNext]
	return i.fill(false, true)
}

// dbIter.Next 找到当前 node 的上一个节点并填充到 i 当中
func (i *dbIter) Prev() bool {
	if i.Released() {
		i.err = ErrIterReleased
		return false
	}

	if i.node == 0 {
        // 如果 forward 是 true，说明 Last 是最后一个点
		if i.forward {
			return i.Last()
		}
		return false
	}
	i.forward = false
	i.p.mu.RLock()
	defer i.p.mu.RUnlock()
	// 找上一个节点是需要从前往后找的，比下一个节点难找
	i.node = i.p.findLT(i.key)
	return i.fill(true, false)
}

// dbIter.Key 返回当前节点的 key
func (i *dbIter) Key() []byte {
	return i.key
}

// dbIter.Value 返回当前节点的 value
func (i *dbIter) Value() []byte {
	return i.value
}

// dbIter.Error 返回当前节点的 err
func (i *dbIter) Error() error { return i.err }

// dbIter.Release 释放当前的迭代器
func (i *dbIter) Release() {
	if !i.Released() {
		i.p = nil
		i.node = 0
		i.key = nil
		i.value = nil
		i.BasicReleaser.Release()
	}
}
```

```go
// 在 nodeData 中，得到每个节点的索引后
// 通过 +nKV +nKey +nVal +nHeight +nNext 来获取相关的信息
const (
	nKV = iota
	nKey
	nVal
	nHeight
	nNext
)
```

```go
// DB is an in-memory key/value database.
// DB 是内存键值数据库
type DB struct {
	// 用于比较的
	cmp comparer.BasicComparer
	// 随机数发生器
	rnd *rand.Rand
	// 锁
	mu sync.RWMutex
	// 键值数据的 kv 信息
	kvData []byte
	// Node data:
	// 本节点 kv 在 kvData 的偏移信息
	// [0]         : KV offset
	// key 的长度
	// [1]         : Key length
	// value 的长度
	// [2]         : Value length
	// 本节点的层高
	// [3]         : Height
	// 每层对应不同层高度下一节点的索引值
	// [4..height] : Next nodes
	nodeData []int
	// 在当前节点的查找过程中的不同层的前节点
	prevNode  [tMaxHeight]int
    // 目前跳表中的最高层
	maxHeight int
    // DB 中节点数量
	n         int
    // DB 中键值数据占用的空间
	kvSize    int
}

// 创建节点时，获取节点的随机高度
func (p *DB) randHeight() (h int) {
	const branching = 4
	h = 1
	for h < tMaxHeight && p.rnd.Int()%branching == 0 {
		h++
	}
	return
}

// 若 prev == true，即要记录查找时不同层的前一节点
// 必须要获取读写锁，因为 prevNode 切片在不同查找过程中是共享使用的
// DB.findGE 根据给定的 key 查找节点在 kvData 的索引
// 找到即找到，未找到就返回第一个比 key 大的节点索引和 false
func (p *DB) findGE(key []byte, prev bool) (int, bool) {
    // 0 节点应该是不存储数据的
	node := 0
	h := p.maxHeight - 1
	for {
		// 当前节点在第 h 层的下一个节点
		next := p.nodeData[node+nNext+h]
		cmp := 1
		// 若索引值不为 0，即该节点在 h 层是有下一节点的
		if next != 0 {
			// o 对应到 next 在 kvData 的 kv 偏移量
			o := p.nodeData[next]
			// 比较 next 节点的 key 和目标节点的 key 大小，因为 kvData 的 key 是升序排列的
			cmp = p.cmp.Compare(p.kvData[o:o+p.nodeData[next+nKey]], key)
		}
		// next 节点的 key 小于目标节点的 key，说明 key 在 next 节点之后，需要再往后寻找
		if cmp < 0 {
			// Keep searching in this list
			node = next
		} else { // next 节点的 key 大于等于目标节点的 key，说明 key 在当前节点和 next 节点之间
			// 若 prev 为 true，即要记录目标节点在不同高度层的前一节点
			if prev {
				p.prevNode[h] = node
			} else if cmp == 0 { // next 节点就是要找的节点
				return next, true
			}
			// 如果已经找到底了，返回是否有找到，以及对应的位置
			// 如果没有找到，说明目标节点比 node 的 key 大，但是比 next 的小，返回 next 节点
			if h == 0 {
				return next, cmp == 0
			}
            // 若目标节点在 node 和 next 之间，才要往更低层找
			h--
		}
	}
}

// DB.findLT 根据给定的 key 查询小于 key 的节点索引
func (p *DB) findLT(key []byte) int {
	node := 0
	h := p.maxHeight - 1
	for {
		// node 第 h 层对应的节点在 p.nodeData 的索引值
		next := p.nodeData[node+nNext+h]
		// next 节点的 kv 在 kvData 中的偏移量
		o := p.nodeData[next]
		// 若 next 是 0，就是说 node 在该 h 层没有指向下一个节点
		// 或 next 节点的 key 大于等于给定 key
		if next == 0 || p.cmp.Compare(p.kvData[o:o+p.nodeData[next+nKey]], key) >= 0 {
			// 若找到了最底层
            // next 为 0 代表在 h == 0 依然没有下一节点，即找到了最后的节点
            // next key 大于等于给定的 key，且 h == 0，说明 node 是要找的节点
			if h == 0 {
				break
			}
            // 否则就往低层找
			h--
		} else {
			// 若 next 节点的 key 小于参数 key，那么需要继续往后找
			node = next
		}
	}
	// 大概是找到最近的小于 key 的节点的索引
	return node
}

// DB.findLast 找到跳表中最后一个节点的索引
func (p *DB) findLast() int {
	node := 0
	h := p.maxHeight - 1
	for {
		next := p.nodeData[node+nNext+h]
        // 若在 h 层没有下一节点
		if next == 0 {
            // 若在 h 层没有下一节点，且 h 为 0，也就是说真没有下一节点了
			if h == 0 {
				break
			}
			h--
		} else {
			node = next
		}
	}
	return node
}

// Put sets the value for the given key. It overwrites any previous value
// for that key; a DB is not a multi-map.
//
// It is safe to modify the contents of the arguments after Put returns.
// Put 为给定的 key 设置 value，并且会进行覆盖
// Put 调用完毕后，更改参数重 key 和 value 的切片值是安全的
// DB 只是单纯的键值字符数组映射，不会涉及 multi-map
func (p *DB) Put(key []byte, value []byte) error {
	// 加锁
	p.mu.Lock()
	defer p.mu.Unlock()

	// 找 key 所在的位置，以及记录下寻找过程中的 prev 节点索引
	// 若找到了...
	if node, exact := p.findGE(key, true); exact {
		// kvData 是直接追加的，具体的偏移是在 nodeData 的 [0] 中记录
		kvOffset := len(p.kvData)
		p.kvData = append(p.kvData, key...)
		p.kvData = append(p.kvData, value...)
		// nodeData[node] 记录当前节点对应到 kvData 的偏移量
		p.nodeData[node] = kvOffset
        // m 是 node value 原先的长度
		m := p.nodeData[node+nVal]
		// nodeData[node+nVal] 记录 value 的长度
		p.nodeData[node+nVal] = len(value)
		// kvSize 的更改
		p.kvSize += len(value) - m
		return nil
	}

	// 若没有找到，就需要创建 node
    // 随机一个高度
	h := p.randHeight()
	// 若 h 超过了目前有的最高高度，那么超过的那些肯定没有 prev 索引都是 0
	if h > p.maxHeight {
		for i := p.maxHeight; i < h; i++ {
			p.prevNode[i] = 0
		}
		// 设置 maxHeight
		p.maxHeight = h
	}

	// 在 kvData 中增加新 Put 的 key 和 value
	kvOffset := len(p.kvData)
	p.kvData = append(p.kvData, key...)
	p.kvData = append(p.kvData, value...)
    
	// Node
	// 为什么新的 node 直接加在最后了？
	// 因为这个跳表的指向关系是直接通过 Next Nodes 那些索引来指向的，不根据存储的顺序
	node := len(p.nodeData)
	p.nodeData = append(p.nodeData, kvOffset, len(key), len(value), h)
	for i, n := range p.prevNode[:h] {
		m := n + nNext + i
        // prevNode 的第 i 层原来的指向赋值到新 node 第 i 层当前的指向
		p.nodeData = append(p.nodeData, p.nodeData[m])
        // prevNode 的第 i 层原来的指向设置为 node
		p.nodeData[m] = node
	}

	p.kvSize += len(key) + len(value)
	p.n++
	return nil
}

// Delete 删除给定 key 的 value，如果没找到就返回 ErrNotFound
// 调用完毕之后，入参 key 的字节切片是可以修改的
func (p *DB) Delete(key []byte) error {
	// 加锁
	p.mu.Lock()
	defer p.mu.Unlock()

	// 找 key 是否存在
	node, exact := p.findGE(key, true)
	if !exact {
		return ErrNotFound
	}

	// h 为当前键为 key 的 node 的 height 值
	h := p.nodeData[node+nHeight]
    // 把各层夹在中间的 node 去掉
	for i, n := range p.prevNode[:h] {
		// 指向第 i 层的 node 的前一节点的索引为 n
		// m = n + nNext + i 为前一节点的指向 node 的索引位置
		m := n + nNext + i
		// 将其更改为当前节点 node 在该层指向的下一节点索引
		p.nodeData[m] = p.nodeData[p.nodeData[m]+nNext+i]
	}

	// 调整 kvSize 和 n，不过本身 nodeData 和 kvData 都是没有改变的
	p.kvSize -= p.nodeData[node+nKey] + p.nodeData[node+nVal]
	p.n--
	return nil
}

// Contains 判断 key 是否在存储中
func (p *DB) Contains(key []byte) bool {
	p.mu.RLock()
	_, exact := p.findGE(key, false)
	p.mu.RUnlock()
	return exact
}

// Get 根据 key 获取 value，不存在则返回 ErrNotFound
// value 切片数据是不能做修改的...
func (p *DB) Get(key []byte) (value []byte, err error) {
	p.mu.RLock()
	if node, exact := p.findGE(key, false); exact {
		o := p.nodeData[node] + p.nodeData[node+nKey]
		value = p.kvData[o : o+p.nodeData[node+nVal]]
	} else {
		err = ErrNotFound
	}
	p.mu.RUnlock()
	return
}

// Find 用于查询比给定的 key 大或相等的 kv 对
// 返回的 key 和 value 切片是不能修改的，否则会影响 DB
func (p *DB) Find(key []byte) (rkey, value []byte, err error) {
	p.mu.RLock()
    // findGE 就是用于找 key 大于或相等的
	if node, _ := p.findGE(key, false); node != 0 {
		n := p.nodeData[node]
		m := n + p.nodeData[node+nKey]
		rkey = p.kvData[n:m]
		value = p.kvData[m : m+p.nodeData[node+nVal]]
	} else {
		err = ErrNotFound
	}
	p.mu.RUnlock()
	return
}

// NewIterator 返回 DB 的迭代器，迭代器本身不是并发安全的，但同时使用多个迭代其实并发安全的
// 使用迭代器进行值的修改也是并发安全的，然而通过迭代器获取 kv 对的结果不保证是始终如一的
// Slice 会获取迭代器的一部分，包含的 key 是给定的范围
// 警告：返回的 slice 都不可以做任何修改
// 迭代器使用完毕之后必须要调用 Release 释放
func (p *DB) NewIterator(slice *util.Range) iterator.Iterator {
	return &dbIter{p: p, slice: slice}
}

// Capacity 返回 kvData 缓存的总体容量
func (p *DB) Capacity() int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return cap(p.kvData)
}

// Size 返回 kv 总大小
// 被删除的 kv 不占用 kvSize 但是在缓存中还是存在的，因为 kvData 是只追加的
func (p *DB) Size() int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return p.kvSize
}

// Free 返回 kvData 中可用的缓存容量
func (p *DB) Free() int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return cap(p.kvData) - len(p.kvData)
}

// Len 返回 DB 中 kv 对的数量
func (p *DB) Len() int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return p.n
}

// Reset 重置 DB 到初始化的空状态
func (p *DB) Reset() {
	p.mu.Lock()
	p.rnd = rand.New(rand.NewSource(0xdeadbeef))
	p.maxHeight = 1
	p.n = 0
	p.kvSize = 0
	p.kvData = p.kvData[:0]
	p.nodeData = p.nodeData[:nNext+tMaxHeight]
	p.nodeData[nKV] = 0
	p.nodeData[nKey] = 0
	p.nodeData[nVal] = 0
	p.nodeData[nHeight] = tMaxHeight
	for n := 0; n < tMaxHeight; n++ {
		p.nodeData[nNext+n] = 0
		p.prevNode[n] = 0
	}
	p.mu.Unlock()
}

// New 创建新初始化的内存键值 DB，capacity 是初始化 kv 缓存容量，且不是固定值
// DB 是只追加的，键值软删除且不会再被使用
// DB 使用并发安全
func New(cmp comparer.BasicComparer, capacity int) *DB {
	p := &DB{
		cmp:       cmp,
		rnd:       rand.New(rand.NewSource(0xdeadbeef)),
		maxHeight: 1,
		kvData:    make([]byte, 0, capacity),
		nodeData:  make([]int, 4+tMaxHeight),
	}
	p.nodeData[nHeight] = tMaxHeight
	return p
}

```

