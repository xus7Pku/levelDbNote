storage.go；

提供了关于存储的一些定义吧算是；

FileType 定义了文件类型，本身是 int 类型的；

```go
type FileType int

const (
	TypeManifest FileType = 1 << iota // 清单文件
	TypeJournal                       // 日志文件
	TypeTable                         // 表格文件
	TypeTemp                          // 临时文件

	TypeAll = TypeManifest | TypeJournal | TypeTable | TypeTemp // 全部文件
)
```

```go
// FileType.String() 返回文件类型的字符串格式
func (t FileType) String() string {
	switch t {
	case TypeManifest:
		return "manifest"
	case TypeJournal:
		return "journal"
	case TypeTable:
		return "table"
	case TypeTemp:
		return "temp"
	}
	return fmt.Sprintf("<unknown:%d>", t)
}
```

```go
// 通用错误类型
var (
    // 文件不合法
	ErrInvalidFile = errors.New("leveldb/storage: invalid file for argument")
    // 文件被锁定
	ErrLocked      = errors.New("leveldb/storage: already locked")
    // 文件已关闭
	ErrClosed      = errors.New("leveldb/storage: closed")
)
```

ErrCorrupted 封装了文件出错的错误；

Package 存储有本身自身的错误类型；

ErrCorrupted 防止了循环导入；

```go
type ErrCorrupted struct {
	// 文件描述符结构体实例
	Fd FileDesc
	// 错误
	Err error
}
```

```go
// isCorrupted 判断是否是 ErrCorrupted 类型的错误
func isCorrupted(err error) bool {
	switch err.(type) {
	case *ErrCorrupted:
		return true
	}
	return false
}
```

```go
// ErrCorrupted.Error 返回内部保存的错误
func (e *ErrCorrupted) Error() string {
    // 若 e.Fd 不是 FileDesc 的空结构体
	if !e.Fd.Zero() {
		return fmt.Sprintf("%v [file=%v]", e.Err, e.Fd)
	}
	return e.Err.Error()
}
```

Syncer 封装了基础的 Sync 方法；

```go
type Syncer interface {
	// Sync 提交文件内当前的内容到稳定的存储中
	Sync() error
}
```

Reader 组装了 Read、Seek、ReadAt、Close 等方法接口；

```go
type Reader interface {
	io.ReadSeeker
	io.ReaderAt
	io.Closer
}
```

Writer 组装了 Write、Close、Sync 等方法接口；

```go
type Writer interface {
	io.WriteCloser
	Syncer
}
```

Locker 封装了 Unlock 方法；

```go
type Locker interface {
	Unlock()
}
```

FileDesc 是文件描述符的封装；

```go
type FileDesc struct {
    // Type 指文件类型
	Type FileType
    // Num 就和平时的文件描述符一样，是用数字来对应
	Num  int64
}
```

```go
// FileDesc.String 返回 FileDesc 的格式化名称
func (fd FileDesc) String() string {
	switch fd.Type {
	case TypeManifest:
		return fmt.Sprintf("MANIFEST-%06d", fd.Num)
	case TypeJournal:
		return fmt.Sprintf("%06d.log", fd.Num)
	case TypeTable:
		return fmt.Sprintf("%06d.ldb", fd.Num)
	case TypeTemp:
		return fmt.Sprintf("%06d.tmp", fd.Num)
	default:
		return fmt.Sprintf("%#x-%d", fd.Type, fd.Num)
	}
}
```

```go
// FileDesc.Zero 判断 fd 是不是空的
func (fd FileDesc) Zero() bool {
	return fd == (FileDesc{})
}
```

```go
// FileDescOk 包外可见，判断 FileDesc 是否合法
func FileDescOk(fd FileDesc) bool {
    // 是否是已定义的类型
	switch fd.Type {
	case TypeManifest:
	case TypeJournal:
	case TypeTable:
	case TypeTemp:
	default:
		return false
	}
    // 相应数字是否合法
	return fd.Num >= 0
}
```

Storage 是满足并发安全的存储接口；

```go
type Storage interface {
	// Lock locks the storage. Any subsequent attempt to call Lock will fail
	// until the last lock released.
	// Caller should call Unlock method after use.
	// Lock 会锁住存储，并拒绝其他获取锁的尝试
	// 调用者必须要调用 Locker 的 Unlock 来释放锁
	Lock() (Locker, error)

	// Log logs a string. This is used for logging.
	// An implementation may write to a file, stdout or simply do nothing.
	// 用于记录日志
	Log(str string)

	// SetMeta store 'file descriptor' that can later be acquired using GetMeta
	// method. The 'file descriptor' should point to a valid file.
	// SetMeta should be implemented in such way that changes should happen
	// atomically.
	// 存储文件描述符的元数据
	SetMeta(fd FileDesc) error

	// GetMeta returns 'file descriptor' stored in meta. The 'file descriptor'
	// can be updated using SetMeta method.
	// Returns os.ErrNotExist if meta doesn't store any 'file descriptor', or
	// 'file descriptor' point to nonexistent file.
	// 获取文件描述符的元数据
	// 如果文件描述符不存在或者文件不存在，那么返回 os.ErrNotExist
	GetMeta() (FileDesc, error)

	// List returns file descriptors that match the given file types.
	// The file types may be OR'ed together.
	// 返回给定文件类型的文件描述符
	List(ft FileType) ([]FileDesc, error)

	// Open opens file with the given 'file descriptor' read-only.
	// Returns os.ErrNotExist error if the file does not exist.
	// Returns ErrClosed if the underlying storage is closed.
	// 打开文件，返回只读 Reader 用于读取内容
	// 如果文件不存在则返回 os.ErrNotExist
	// 如果存储关闭了则返回 ErrClosed
	Open(fd FileDesc) (Reader, error)

	// Create creates file with the given 'file descriptor', truncate if already
	// exist and opens write-only.
	// Returns ErrClosed if the underlying storage is closed.
	// 创建文件描述符对应的文件，如果以存在就 Reset，只写打开
	Create(fd FileDesc) (Writer, error)

	// Remove removes file with the given 'file descriptor'.
	// Returns ErrClosed if the underlying storage is closed.
	// Remove 移除文件描述符给定的文件
	Remove(fd FileDesc) error

	// Rename renames file from oldfd to newfd.
	// Returns ErrClosed if the underlying storage is closed.
	// 更改文件对应的文件描述符
	Rename(oldfd, newfd FileDesc) error

	// Close closes the storage.
	// It is valid to call Close multiple times. Other methods should not be
	// called after the storage has been closed.
	// 关闭存储
	Close() error
}
```



mem_storage.go；

内存存储；

```go
// 能够代表文件类型的位数
const typeShift = 4
// 判断 typeShift 是否足够大
var _ [0]struct{} = [TypeAll >> typeShift]struct{}{}
```

memStorageLock，内存存储的 Locker；

```go
type memStorageLock struct {
    // 所属的内存存储实例
	ms *memStorage
}
```

```go
// memStorage Unlock 是为所属的内存存储解锁
func (lock *memStorageLock) Unlock() {
	// 获取 ms
	ms := lock.ms
	// 锁住 ms 的 mu 锁
	ms.mu.Lock()
	defer ms.mu.Unlock()
	// 如果 ms.slock 存储了当前的内存存储锁，那么就置空，因为调用 Unlock 已经释放了
	// 调用 ms.Lock 后都会设置这个内存存储锁，并用于后续的解锁
    // 相当于说 ms.slock 设置值就像是 ms 获取了锁
	if ms.slock == lock {
		ms.slock = nil
	}
	return
}
```

memStorage 是内存存储；

```go
type memStorage struct {
	// 锁
	mu sync.Mutex
	// 用于 Unlock 的
	slock *memStorageLock
	// 对应的内存文件字典
	files map[uint64]*memFile
	// 是该内存存储对应的文件描述符，不是内存文件对应的
	meta FileDesc
}
```

```go
// memStorage.Lock 锁住 memStorage 并返回 Locker
func (ms *memStorage) Lock() (Locker, error) {
	// ms.mu 的 Lock
	ms.mu.Lock()
	defer ms.mu.Unlock()
	// 如果 ms.slock 不是 nil，说明之前有锁住的，返回
	if ms.slock != nil {
		return nil, ErrLocked
	}
	// 设置 ms.slock 内存存储锁表示已锁住
	ms.slock = &memStorageLock{ms: ms}
	return ms.slock, nil
}
```

```go
// memStorage.Log 内存存储没有存储日志
func (*memStorage) Log(str string) {}
```

```go
// memStorage.SetMeta 设置该内存存储的元数据 FileDesc
func (ms *memStorage) SetMeta(fd FileDesc) error {
	if !FileDescOk(fd) {
		return ErrInvalidFile
	}

	ms.mu.Lock()
	ms.meta = fd
	ms.mu.Unlock()
	return nil
}
```

```go
// memStorage.GetMeta 获取该内存存储的元数据
func (ms *memStorage) GetMeta() (FileDesc, error) {
	ms.mu.Lock()
	defer ms.mu.Unlock()
	if ms.meta.Zero() {
		return FileDesc{}, os.ErrNotExist
	}
	return ms.meta, nil
}
```

```go
// memStorage.List 返回该内存存储中给定文件类型的存储文件对应的文件描述符
func (ms *memStorage) List(ft FileType) ([]FileDesc, error) {
	ms.mu.Lock()
	var fds []FileDesc
	for x := range ms.files {
		// x 为 ms.Files 的索引，就对应 FileDesc 的 1 2 3
		// 键转 FileDesc
		fd := unpackFile(x)
		if fd.Type&ft != 0 {
			fds = append(fds, fd)
		}
	}
	ms.mu.Unlock()
	return fds, nil
}
```

```go
// memStorage.Open 打开 FileDesc 对应的文件并返回读接口实例
func (ms *memStorage) Open(fd FileDesc) (Reader, error) {
	// 文件描述符是否有问题
	if !FileDescOk(fd) {
		return nil, ErrInvalidFile
	}

	ms.mu.Lock()
	defer ms.mu.Unlock()
	// 如果文件描述符对应的键在 ms.files 里面
	if m, exist := ms.files[packFile(fd)]; exist {
		// 如果文件已经被打开了，说明已经有 Reader 或 Writer 实例了，不能再有第二个实例
		if m.open {
			return nil, errFileOpen
		}
		m.open = true
		// bytes.NewReader 用于读 m.Bytes，实现了 Reader 接口
		return &memReader{Reader: bytes.NewReader(m.Bytes()), ms: ms, m: m}, nil
	}
	// 没有找到文件描述符对应的存储
	return nil, os.ErrNotExist
}
```

```go
// memStorage.Create 创建 FileDesc 对应的文件并返回写接口实例
func (ms *memStorage) Create(fd FileDesc) (Writer, error) {
    // 文件描述符是否有问题
	if !FileDescOk(fd) {
		return nil, ErrInvalidFile
	}

	x := packFile(fd)
	ms.mu.Lock()
	defer ms.mu.Unlock()
    // ms.files 是否有键 x
	m, exist := ms.files[x]
	if exist {
		if m.open {
			return nil, errFileOpen
		}
		// 如果是关闭的没人使用，就重置清空
        // 是 buffer.Reset 方法
		m.Reset()
	} else {
		// 如果不存在，就新建 memFile
		m = &memFile{}
		// ms.files 是字典
		ms.files[x] = m
	}
	m.open = true
    // buffer 本身就实现了 Writer 接口
	return &memWriter{memFile: m, ms: ms}, nil
}
```

```go
// memStorage.Remove 移除 FileDesc 对应的 memFile
func (ms *memStorage) Remove(fd FileDesc) error {
	if !FileDescOk(fd) {
		return ErrInvalidFile
	}

	x := packFile(fd)
	ms.mu.Lock()
	defer ms.mu.Unlock()
	if _, exist := ms.files[x]; exist {
        // 在 m.files 内移除键 x
		delete(ms.files, x)
		return nil
	}
	return os.ErrNotExist
}
```

```go
// memStorage.Rename 更换文件对应的文件描述符 FileDesc
func (ms *memStorage) Rename(oldfd, newfd FileDesc) error {
	if !FileDescOk(oldfd) || !FileDescOk(newfd) {
		return ErrInvalidFile
	}
	if oldfd == newfd {
		return nil
	}

	oldx := packFile(oldfd)
	newx := packFile(newfd)
	ms.mu.Lock()
	defer ms.mu.Unlock()
	oldm, exist := ms.files[oldx]
	if !exist {
		return os.ErrNotExist
	}
	newm, exist := ms.files[newx]
	// 如果文件被打开说明不能关闭啊
	if (exist && newm.open) || oldm.open {
		return errFileOpen
	}
	delete(ms.files, oldx)
	ms.files[newx] = oldm
	return nil
}
```

```go
// memStorage.Close 无事发生
func (*memStorage) Close() error { return nil }
```

memFile 是内存存储的对应文件结构体

```go
type memFile struct {
	// 字节缓冲区，存储字节的地方
	bytes.Buffer
	// 是否被打开
	open bool
}
```

memReader 是打开 memFile 且只读的结构体；

```go
type memReader struct {
	// 对应的 Reader 实例
	*bytes.Reader
	// 所在的 memStorage
	ms *memStorage
	// 所属的 memFile
	m *memFile
	// 是否关闭
	closed bool
}
```

```go
// memReader.Close 关闭 memReader
func (mr *memReader) Close() error {
	mr.ms.mu.Lock()
	defer mr.ms.mu.Unlock()
	// 已关闭
	if mr.closed {
		return ErrClosed
	}
	// 将 memFile 设置为已关闭，因为 memReader 也要关闭了嘛
	mr.m.open = false
    // 为什么没有设置 mr.closed 呢？
	return nil
}
```

memWriter 是打开 memFile 的写接口实例；

```go
type memWriter struct {
	// 所属的 memFile
	// bytes.Buffer 本身就有实现 Writer 的接口
	*memFile
	// 所在的 memStorge
	ms *memStorage
	// 是否关闭
	closed bool
}
```

```go
// 内存存储不需要持久化？
func (*memWriter) Sync() error { return nil }
```

```go
// memWriter.Close 关闭写接口实例
func (mw *memWriter) Close() error {
	mw.ms.mu.Lock()
	defer mw.ms.mu.Unlock()
	if mw.closed {
		return ErrClosed
	}
	mw.memFile.open = false
	return nil
}
```

```go
// 文件描述符转变为数字
func packFile(fd FileDesc) uint64 {
	return uint64(fd.Num)<<typeShift | uint64(fd.Type)
}

// 数字转变为文件描述符
func unpackFile(x uint64) FileDesc {
	return FileDesc{FileType(x) & TypeAll, int64(x >> typeShift)}
}
```



