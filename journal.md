日志包读写日志序列；

日志是字节流，日志在下一日志开始前结束；



读取时，调用 Next 方法获取下一日志的 io.Reader 磁盘读接口；

没有更多的日志时就会返回 io.EOF；

当前日志没有读完就调用 Next 是不合法的；



写入时，调用 Next 方法获取下一日志的 io.Writer 磁盘写接口；

调用 Next 接口会结束当前日志的写；

调用 Close 接口会结束最后日志的写；



可选的，调用 Flush 结束当前的日志写，并不开启新的写接口；

调用 Next 可以开启新的日志写；



Reader 和 Writer 都不能安全地并发使用，会进行判断；



日志被分为 32KB 的 block；

每个 block 包含数个 chunk，chunk 不能跨越 block 边界而存在；

最后一个 block 可以小于 32KB，block 没有被使用的字节都置为空字符；



一个日志对应多个 chunk；

每个 chunk 包含 7 字节的 header，4KB 的 checksum、2KB 的 length、1KB 的 chunk 类型；

checksum 用于校验 chunk 类型和后续的 data 部分；



chunk 有四种类型，包括 full 日志的 full chunk，或是 multi-chunk 日志的 first、middle、last chunk；

multi-chunk 日志包括 first、0 或多个 middle、1 个 last；



固定 chunk 的格式能够查探数据是否出错，并正确进行判断和恢复；



```go
// Type 占 1B
const (
	fullChunkType   = 1
	firstChunkType  = 2
	middleChunkType = 3
	lastChunkType   = 4
)
```

```go
// block 大小为 32KB
// header 大小为 7B
const (
	blockSize  = 32 * 1024
	headerSize = 7
)
```

```go
// Flush 方法的封装
// Flush 方法会结束当前的日志写接口，且不开启下一日志的写接口
type flusher interface {
	Flush() error
}
```

```go
// ErrCorrupted 是错误类型，因为损坏的 block 或 chunk 而产生
type ErrCorrupted struct {
	Size   int
	Reason string
}

// ErrCorrupted.Error 格式化 error
func (e *ErrCorrupted) Error() string {
	return fmt.Sprintf("leveldb/journal: block/chunk corrupted: %s (%d bytes)", e.Reason, e.Size)
}
```

```go
// Dropper 封装了 Drop 方法
// Drop 方法在日志读接口丢弃 block 或 chunk 时调用
type Dropper interface {
	Drop(err error)
}
```

```go
// Reader 通过 io.Reader 读取磁盘的日志
type Reader struct {
	// 磁盘读取接口
	r io.Reader
	// the dropper.
	dropper Dropper
	// 若有损坏的部分直接结束
	strict bool
	// 是否有使用校验码
	checksum bool
	// 当前日志的 sequence number
	seq int
	// buf[i:j] 是当前 chunk 数据的未读部分
	// i 是要除去 chunk header 的
	i, j int
	// n is the number of bytes of buf that are valid. Once reading has started,
	// only the final block can have n < blockSize.
	// n 是 buf 中可用的总字节数，在 buf 中的整个 block 的大小
	// 当读开始，仅最后一个 block 可以 n < blockSize
	n int
	// 是否当前 chunk 是完整 journal 最后的 chunk
	last bool
	// err is any accumulated error.
	err error
	// buf is the buffer.
	buf [blockSize]byte
}

// NewReader 返回新的读结构体实例
func NewReader(r io.Reader, dropper Dropper, strict, checksum bool) *Reader {
	return &Reader{
		r:        r,
		dropper:  dropper,
		strict:   strict,
		checksum: checksum,
        // 为什么要设置成 true？
		last:     true,
	}
}
```

```go
var errSkip = errors.New("leveldb/journal: skipped")
```

```go
// Reader.corrupt 进行 Reader 的错误处理
func (r *Reader) corrupt(n int, reason string, skip bool) error {
	if r.dropper != nil {
		r.dropper.Drop(&ErrCorrupted{n, reason})
	}
	if r.strict && !skip {
		r.err = errors.NewErrCorrupted(storage.FileDesc{}, &ErrCorrupted{n, reason})
		return r.err
	}
	return errSkip
}

// Reader.nextChunk 置 r.buf[r.i:r.j] 存储下一 chunk 的数据
// 且在需要的情况下读取下一 block
func (r *Reader) nextChunk(first bool) error {
    // 令 r.i 和 r.j 在 r.buf 中指向下一 chunk
	for {
		// 目前 r.i 和 r.j 还是上一 chunk 的
        // 进入该逻辑必然 return 的
		if r.j+headerSize <= r.n {
			checksum := binary.LittleEndian.Uint32(r.buf[r.j+0 : r.j+4])
			length := binary.LittleEndian.Uint16(r.buf[r.j+4 : r.j+6])
			chunkType := r.buf[r.j+6]
			// buf 中剩余的空间
			unprocBlock := r.n - r.j
			// 若 chunk 的 header 有问题，则丢弃整个 block
			if checksum == 0 && length == 0 && chunkType == 0 {
				// Drop entire block.
				r.i = r.n
				r.j = r.n
				return r.corrupt(unprocBlock, "zero header", false)
			}
            // 若 chunk 的 type 有问题
			if chunkType < fullChunkType || chunkType > lastChunkType {
				// Drop entire block.
				r.i = r.n
				r.j = r.n
				return r.corrupt(unprocBlock, fmt.Sprintf("invalid chunk type %#x", chunkType), false)
			}
            // 设置 r.i 和 r.j
			r.i = r.j + headerSize
			r.j = r.j + headerSize + int(length)
			// 若 r.j 索引超过了 r.n，说明该 block 存在问题
			if r.j > r.n {
				// Drop entire block.
				r.i = r.n
				r.j = r.n
				return r.corrupt(unprocBlock, "chunk length overflows block", false)
			} else if r.checksum && checksum != util.NewCRC(r.buf[r.i-1:r.j]).Value() {
            // 若 checksum 存在问题
				// Drop entire block.
				r.i = r.n
				r.j = r.n
				return r.corrupt(unprocBlock, "checksum mismatch", false)
			}
            // 若 chunk 类型无法匹配上，仅仅是 skip
			if first && chunkType != fullChunkType && chunkType != firstChunkType {
				chunkLength := (r.j - r.i) + headerSize
				r.i = r.j
				// Report the error, but skip it.
				return r.corrupt(chunkLength, "orphan chunk", true)
			}
			// 设置 r.last 字段，即当前 chunk 是否为 journal 的最后一个 chunk
			r.last = chunkType == fullChunkType || chunkType == lastChunkType
			return nil
		}
        
        // 如果 r.j+headerSize > r.n 的话

		// The last block.
		// 如果当前的是 last block
		if r.n < blockSize && r.n > 0 {
			// first 指的是当前 chunk 是 nulti-chunk journal 的 first chunk
			if !first {
				return r.corrupt(0, "missing chunk part", false)
			}
			r.err = io.EOF
			return r.err
		}
        
        // 如果上一 block 之后没有 chunk 了
        // 且上一 block 不是 last block 的话

		// Read block.
		// 读取下一 block 进入到 r.buf 中
		n, err := io.ReadFull(r.r, r.buf[:])
		if err != nil && err != io.EOF && err != io.ErrUnexpectedEOF {
			return err
		}
		if n == 0 {
            // 如果需要有 middle 或 last 读出来
			if !first {
				return r.corrupt(0, "missing chunk part", false)
			}
			r.err = io.EOF
			return r.err
		}
        
        // 读取下一个 block 成功了，那 r.i r.j r.n 都要从头开始了
		r.i, r.j, r.n = 0, 0, n
	}
}

// Reader.Next 返回下一日志的读接口，若没有更多的日志就返回 io.EOF
// 返回的读接口在下一次调用 Next 时就不再使用
// 若 strict 是 true，那么发现日志错误时会返回 io.ErrUnexpectedEOF

// 是读取下一个日志，而且是读取该日志的第一条 chunk
func (r *Reader) Next() (io.Reader, error) {
    // r 当前日志的 seq++
	r.seq++
	if r.err != nil {
		return nil, r.err
	}
	// 在 r 中设置 r.i 和 r.j 使其为下一个 chunk
	r.i = r.j
	for {
		if err := r.nextChunk(true); err == nil {
			break
		} else if err != errSkip {
        // 若 err 不为 errSkip
			return nil, err
		}
	}
	// singleReader 是该下一个 chunk 的读接口
	return &singleReader{r, r.seq, nil}, nil
}

// Reader.Reset 重置日志 Reader 实例复用，返回积累的 err
func (r *Reader) Reset(reader io.Reader, dropper Dropper, strict, checksum bool) error {
	r.seq++
	err := r.err
	r.r = reader
	r.dropper = dropper
	r.strict = strict
	r.checksum = checksum
	r.i = 0
	r.j = 0
	r.n = 0
	r.last = true
	r.err = nil
	return err
}
```

```go
// singleReader 结构体用于读取一个 journal 的数据
type singleReader struct {
	r   *Reader
	seq int
	err error
}
```

```go
// singleReader.Read 读取 r.buf[r.i:r.j] 的数据到 p 中
func (x *singleReader) Read(p []byte) (int, error) {
	r := x.r
	// req 不同说明是 stale reader
    // r.seq 和 x.seq 不同说明在生成 x 之后，r 进行了新的变化
	if r.seq != x.seq {
		return 0, errors.New("leveldb/journal: stale reader")
	}
	if x.err != nil {
		return 0, x.err
	}
	if r.err != nil {
		return 0, r.err
	}
	// r.i == r.j 是有问题的，在读了 first 之后
	for r.i == r.j {
        // 如果是最后一个 chunk 则说明 journal 完了
		if r.last {
			return 0, io.EOF
		}
		// 读取下一个 chunk，first 已经在 Next 读过了
		x.err = r.nextChunk(false)
		if x.err != nil {
            // 若 err 不为 errSkip
			if x.err == errSkip {
				x.err = io.ErrUnexpectedEOF
			}
			return 0, x.err
		}
	}
	n := copy(p, r.buf[r.i:r.j])
	// 这里会让 r.i == r.j
	r.i += n
	return n, nil
}

// singleReader.ReadByte 读取 chunk 中的单个字节
// 同 singleReader.Read 类似
func (x *singleReader) ReadByte() (byte, error) {
	r := x.r
	if r.seq != x.seq {
		return 0, errors.New("leveldb/journal: stale reader")
	}
	if x.err != nil {
		return 0, x.err
	}
	if r.err != nil {
		return 0, r.err
	}
	for r.i == r.j {
		if r.last {
			return 0, io.EOF
		}
		x.err = r.nextChunk(false)
		if x.err != nil {
			if x.err == errSkip {
				x.err = io.ErrUnexpectedEOF
			}
			return 0, x.err
		}
	}
	c := r.buf[r.i]
	r.i++
	return c, nil
}
```

```go
// Writer 实例通过 io.Writer 写入磁盘
type Writer struct {
	// w is the underlying writer.
	w io.Writer
	// 当前日志的 sequence number
	seq int
	// f is w as a flusher.
	f flusher
	// buf[i:j] is the bytes that will become the current chunk.
	// The low bound, i, includes the chunk header.
    // buf[i:j] 是会成为当前 chunk 的前后索引
	i, j int
    // buf[:written] 是已经通过 w 写入磁盘的字节
    // written 是 0 除非 Flush 被调用？
	written int
	// 当前 chunk 是否是 journal 的第一个 chunk
	first bool
    // 指当前 chunk 在缓冲区但还没有写入磁盘
	pending bool
	// err is any accumulated error.
	err error
	// buf is the buffer.
    // 写入的缓冲区
	buf [blockSize]byte
}

// NewWriter returns a new Writer.
func NewWriter(w io.Writer) *Writer {
	f, _ := w.(flusher)
	return &Writer{
		w: w,
		f: f,
	}
}

// Writer.fillHeader 向 buf 中填充 header
func (w *Writer) fillHeader(last bool) {
    // 若 i,j 索引有问题
	if w.i+headerSize > w.j || w.j > blockSize {
		panic("leveldb/journal: bad writer state")
	}
    
    // 填充 chunkType
	// 如果该 chunk 是 last chunk 类型
	if last {
		// 是否是 first chunk 类型
		if w.first {
			w.buf[w.i+6] = fullChunkType
		} else {
			w.buf[w.i+6] = lastChunkType
		}
	} else {
		if w.first {
			w.buf[w.i+6] = firstChunkType
		} else {
			w.buf[w.i+6] = middleChunkType
		}
	}
	// 写入 checksum 和 length
	binary.LittleEndian.PutUint32(w.buf[w.i+0:w.i+4], util.NewCRC(w.buf[w.i+6:w.j]).Value())
	binary.LittleEndian.PutUint16(w.buf[w.i+4:w.i+6], uint16(w.j-w.i-headerSize))
}

// Writer.writeBlock 将 buf 中的内容通过 io.Writer 写入磁盘
func (w *Writer) writeBlock() {
    // w.written 前面的是已经写入的
	_, w.err = w.w.Write(w.buf[w.written:])
    // 进行 r.i r.j 的重置
	w.i = 0
	w.j = headerSize
	w.written = 0
}

// Writer.writePending 结束当前的 journal，并将其写入磁盘
func (w *Writer) writePending() {
	if w.err != nil {
		return
	}
	if w.pending {
        // true 参数表示是当前日志的 last chunk
		w.fillHeader(true)
		w.pending = false
	}
	_, w.err = w.w.Write(w.buf[w.written:w.j])
	w.written = w.j
}

// Writer.Close 结束当前的 journal，并写入磁盘
func (w *Writer) Close() error {
    // 可能是为下一 journal 的 seq 作准备
	w.seq++
	w.writePending()
	if w.err != nil {
		return w.err
	}
	w.err = errors.New("leveldb/journal: closed Writer")
	return nil
}

// Writer.Flush 结束当前的 journal，以及调用 Flush，且不开启新的写接口
func (w *Writer) Flush() error {
	w.seq++
	w.writePending()
	if w.err != nil {
		return w.err
	}
	if w.f != nil {
		w.err = w.f.Flush()
		return w.err
	}
	return nil
}

// Writer.Reset 会重置实例，且如果没准备好会关闭 journal writer？
func (w *Writer) Reset(writer io.Writer) (err error) {
	w.seq++
	if w.err == nil {
        // 当前 journal last chunk 写入磁盘
        
		w.writePending()
		err = w.err
	}
	w.w = writer
	w.f, _ = writer.(flusher)
	w.i = 0
	w.j = 0
	w.written = 0
	w.first = false
	w.pending = false
	w.err = nil
	return
}

// Writer.Next 返回下一 journal 的写接口
// 该接口在调用 Close、Flush、Next 方法就不能够再使用了，seq 会对不上
func (w *Writer) Next() (io.Writer, error) {
	w.seq++
	if w.err != nil {
		return nil, w.err
	}
    // 要下一个 journal 的写接口了，那么如果 w.pending 为 true，必然是 last chunk 了
	if w.pending {
		w.fillHeader(true)
	}
	w.i = w.j
	w.j = w.j + headerSize
	// 没有足够的空间放下 header
	if w.j > blockSize {
		// 将 w.i 后边的均置为空字符
		for k := w.i; k < blockSize; k++ {
			w.buf[k] = 0
		}
		// 将当前 w 的 buf 写入到磁盘中去
		w.writeBlock()
		if w.err != nil {
			return nil, w.err
		}
	}
	w.first = true
	w.pending = true
    // 返回新的 seq 的写接口
	return singleWriter{w, w.seq}, nil
}
```

```go
// singleWriter 是封装的写结构
type singleWriter struct {
	w   *Writer
	seq int
}

// 将 p 中的存入到 buf 中
func (x singleWriter) Write(p []byte) (int, error) {
	w := x.w
    // seq 不同说明连续调用了 Next 之类的方法
	if w.seq != x.seq {
		return 0, errors.New("leveldb/journal: stale writer")
	}
	if w.err != nil {
		return 0, w.err
	}
	n0 := len(p)
	for len(p) > 0 {
		// Write a block, if it is full.
		// 如果当前 buf 写满了，就通过 io.Writer 写到磁盘中，并且把 buf 清空
		if w.j == blockSize {
            // 不是 last chunk
			w.fillHeader(false)
			w.writeBlock()
            // 到这里 w.j=headerSize, w.i=0, w.written=0
			if w.err != nil {
				return 0, w.err
			}
			w.first = false
		}
		// Copy bytes into the buffer.
		n := copy(w.buf[w.j:], p)
		w.j += n
		p = p[n:]
	}
	return n0, nil
}
```



levelDb 的日志存储有三个概念：

*   block 是固定大小的，算是物理单位，也是刷磁盘时的标准；
*   journal 时日志的逻辑单位；
*   chunk 是迫不得已才存在的；
    *   journal 跨 block 才会有 chunk 的存在必要；
    *   如果 block 剩余空间能够存下整个 journal，那么这个 journal 就只有一个 chunk 为 fullType；

journal 具体的日志内容：

*   是其 full chunk 或从 first chunk 到 last chunk 内 playload 拼合起来的内容，不是由 journal 包决定的；
*   日志内容包括 journal-header 保存 seq 和 batch number，和 journal-data 日志的操作记录数据；

日志写：

*   singleWriter 用于写入一个完整的 journal；
*   通过 singleWriter.Write 写入 journal；
    *   该 journal 没写满当前 block 就存在 w.buf 中存着；
    *   该 journal 写满了就将当前的 block 刷入磁盘，且设置 chunk 为 firstType，并换 block 填 buf；
    *   写同一个 journal 时 seq 是不变的，写完之后调用 Next 来获取下一 journal 写入的 singleWriter；

日志读：

*   singleReader 用于读取一个完整的 journal；
*   singleReader.Read 读取 journal；
    *   每读取一个 chunk 就进行校验，校验出错就进行丢弃；
    *   循环读取 chunk，如果是 last chunk 就返回 io.EOF；
    *   读完之后，调用 Next 就获取下一 journal 读取的 singleReader；

