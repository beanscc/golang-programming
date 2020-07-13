# bufio

bufio 包，实现了缓冲 I/O，它封装了 io.Reader 或 io.Writer 对象， 创建了另外一个对象（Reader 或 Writer），这个对象也实现了 I/O 接口，但是为文本 I/O 提供了缓冲和一些帮助

## type Reader

缓冲输入

Reader 结构体定义如下：

```go
// Reader implements buffering for an io.Reader object.
type Reader struct {
	buf          []byte
	rd           io.Reader //reader provided by the client
	r, w         int       //buf read and write positions
	err          error
	lastByte     int // 最后一个字节的int值
	lastRuneSize int // 最后一个字符的int值
}
```

### New Reader 的两个函数

#### NewReaderSize(rd io.Reader, size int)

`NewReaderSize` 函数，返回一个含指定buffer大小的新 `Reader` 对象；

若参数 `io.Reader` 是一个已经包含buffer足够大小的 `Reader` 对象，则返回参数 `io.Reader` 底层的 `Reader`;

若参数 size < 16(minReadBufferSize)，则 size = 16

```go
// NewReaderSize returns a new Reader whose buffer has at least the specified
// size. If the argument io.Reader is already a Reader with large enough
// size, it returns the underlying Reader.
func NewReaderSize(rd io.Reader, size int) *Reader {
	// Is it already a Reader?
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}
```

#### NewReader(rd io.Reader)

`NewReader` 返回一个指定 `buffer`大小为默认大小（4096）的新 `Reader` 对象，

```go
// NewReader returns a new Reader whose buffer has the default size.
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}
```


### Reset(r io.Reader)

`Reset` 方法舍弃所有已缓存数据，重置所有参数状态，将缓存输入源切换至 r


```go
// Reset discards any buffered data, resets all state, and switches
// the buffered reader to read from r.
func (b *Reader) Reset(r io.Reader) {
	b.reset(b.buf, r)
}

func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}
```

### Peek(n int)

`Peek`

```go
// Peek returns the next n bytes without advancing the reader. The bytes stop
// being valid at the next read call. If Peek returns fewer than n bytes, it
// also returns an error explaining why the read is short. The error is
// ErrBufferFull if n is larger than b's buffer size.
func (b *Reader) Peek(n int) ([]byte, error) {}
```

### Discard(n int) 

`Discard` 方法，将跳过接下来要读取的 n 个字节，返回跳过的字节数和遇到的错误；

若跳过的字节数小于 n，则同时会返回一个 error；
若 0 <= n <= b.Buffered()， `DisCard` 可以保证成功的从底层 io.Reader 中跳过 n 个字节


```go
// Discard skips the next n bytes, returning the number of bytes discarded.
//
// If Discard skips fewer than n bytes, it also returns an error.
// If 0 <= n <= b.Buffered(), Discard is guaranteed to succeed without
// reading from the underlying io.Reader.
func (b *Reader) Discard(n int) (discarded int, err error) {
```


### Read(p []byte)

`Read` 方法，将数据读入 p；返回读入 p 的字节数量和遇到的错误；

读入 p 的数据，是一次从底层 io.Reader 中读取的；因此返回的字节数可能小于 p 的切片长度；
在 EOF处，返回的字节数 n = 0，且 err = io.EOF

```go
// Read reads data into p.
// It returns the number of bytes read into p.
// The bytes are taken from at most one Read on the underlying Reader,
// hence n may be less than len(p).
// At EOF, the count will be zero and err will be io.EOF.
func (b *Reader) Read(p []byte) (n int, err error) {}
```


### ReadByte()

`ReadByte` 方法返回读取的最后一个字节和遇到的错误

读取最后一个字节，并将字节值转成 int 型存在 `b.lastByte` 中，用于 `b.UnreadByte()` 方法，撤销最后读取的一个字节内容时使用


```go
// ReadByte reads and returns a single byte.
// If no byte is available, returns an error.
func (b *Reader) ReadByte() (byte, error) {}
```

### UnreadByte()

`UnreadByte` 方法撤销最后读取的一个字节内容

撤销最近一次 **任何读取操作** 读取内容的最后一个字节，内部实现是通过将 `b.lastByte` 存储的 int 型数字，转化为对应的 byte 类型，重新赋值给缓冲区，即 b.buf[b.r] = byte(b.lastByte) 

```go
// UnreadByte unreads the last byte. Only the most recently read byte can be unread.
func (b *Reader) UnreadByte() error {}
```

### ReadRune()

`ReadRune` 方法读取单个 UTF-8 编码的 `Unicode` 字符，并返回这个字符的 rune 类型、字节数、error 类型错误

若编码字符是无效的，则消耗一个字节并返回大小为 1 的 `unicode.ReplacementChar (U+FFFD)`

```go
// ReadRune reads a single UTF-8 encoded Unicode character and returns the
// rune and its size in bytes. If the encoded rune is invalid, it consume one byte
// and returns unicode.ReplacementChar (U+FFFD) with a size of 1.
func (b *Reader) ReadRune() (r rune, size int, err error) {}
```

### UnreadRune()

`UnreadRune` 方法撤销读取的最后一个字符。若缓存区上最新读取操作不是 `ReadRune`，则返回 error 错误(在这方面，它比 `UnreadByte` 更严格，`UnreadByte` 将撤销最近的任何一次读取操作的最后一个字节)。

也就是说， `UnreadRune` 只能在 `ReadRune` 读取操作之后执行，

```go
// UnreadRune unreads the last rune. If the most recent read operation on
// the buffer was not a ReadRune, UnreadRune returns an error. （In this
// regard it is stricter than UnreadByte, which will unread the last byte
// from any read operation.)
func (b *Reader) UnreadRune() error {}
```

### Buffered()

`Buffered` 方法返回从当前缓冲区中可以读取的字节数

```go
// Buffered returns the number of bytes that can be read from the current buffer.
func (b *Reader) Buffered() int { return b.w - b.r }
```

### ReadSlice(delim byte)

`ReadSlice` 方法，从输入内容中读取，直到第一次读到 `delim` 字节内容，返回一个指向缓冲区字节的slice切片。

下次读操作时，字节停止有效；

若ReadSlice在读取到 delim 字节之前遇到错误，则返回所有已经读到缓存中的数据，和遇到的错误（经常是 io.EOF）;

若缓存数据中没有 delim 字节，则 ReadSlice 返回错误 ErrBufferFull error；

大多客户端应该 使用 ReadBytes 或者 ReadString 操作来替代 ReadSlice ，因为 ReadSlice 返回的数据将被在下次 I/O 操作时会被覆盖；（line 切片属于引用类型，会被下次I/O操作的结果所覆盖）

当且仅当返回值 line 不以 delim 结尾时， 返回 err！=nil 的 error

```go
// ReadSlice reads until the first occurrence of delim in the input,
// returning a slice pointing at the bytes in the buffer.
// The bytes stop being valid at the next read.
// If ReadSlice encounters an error before finding a delimiter,
// it returns all the data in the buffer and the error itself(often io.EOF).
// ReadSlice fails with error ErrBufferFull if the buffer fills without a delim.
// Because the data returned from ReadSlice will be overwritten
// by the next I/O operation, most clients should use
// ReadBytes or ReadString instead.
// ReadSlice returns err != nil if and only if line does not end in delim.
func (b *Reader) ReadSlice(delim byte) (line []byte, err error) {}
```

### ReadLine()

`ReadLine` 方法是一个低级的原始的按行读取的读操作。大多调用者应该使用 ReadBytes('\n') 或者 ReadString('\n') 替换，或者使用 Scanner

ReadLine 尝试返回单独一行，不包含行结束符。若一行内容长度超过缓冲区长度，则返回值 isPrefix 将被设置为 true, 且这一行的开始部分也一同返回。这一行超过缓冲部分长度将在未来调用时返回。当返回这一行的剩余部分时 isPrefix 将为 false。直到下次调用 ReadLine 方法时，返回的缓冲才有效。

ReadLine 或返回一个error 为非 nil错误的 line，或者返回一个error，不会同时存在两种情况

ReadLine 返回的结果数据不包含行结尾标识符号（"\r\n" 或 "\n"）。

若输入内容没有行结束标识符，则返回值中不会给出 标识符号 或者 error 错误。

在调用 ReadLine 之后调用 UnreadByte 总是撤销最后读取的字节（很可能是行结束标识符号），即使这个字节并非是ReadLine 返回的（说的就是行结束标识符）

```go

func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error) {}
```

### ReadBytes(delim byte)

`ReadBytes` 读取输入，直到读到分隔符（delim），返回一个包含读取到分隔符位置的byte切片（含分隔符）；

在读取到 delim 字节之前遇到错误，则返回所有已经读到缓存中的数据，和遇到的错误（经常是 io.EOF）;
当且仅当返回值 line 不以 delim 结尾时， 返回 err！=nil 的 error；

对于简单的使用， `Scanner` 会更方便

```go

func (b *Reader) ReadBytes(delim byte) ([]byte, error) {}
```

### ReadString(delim byte)

`ReadString` 内部调用 `ReadBytes` 方法，只是将返回的 `[]byte` 切片转了下 `string` 而已

```go
func (b *Reader) ReadString(delim byte) (string, error) {
	bytes, err := b.ReadBytes(delim)
	return string(bytes), err
}
```

### WriteTo(w io.Writer)

`WriteTo` 实现了 io.WriteTo 接口

方法内部实现思路，
(1) 将读取到缓冲区的内容写入 w
(2) 尝试调用 b 底层 io.Reader 的 WriteTo 方法，将未读入缓冲区的数据直接写入 w io.Writer
(3) 尝试调用 w io.Writer 底层的 ReadFrom 方法，将 b 底层 io.Reader 中数据读入 w 中
(4) 循环将缓冲区的内容写入 w

```go
// WriteTo implements io.WriterTo.
func (b *Reader) WriteTo(w io.Writer) (n int64, err error) {}
```

## type Writer

缓冲输出

Writer 结构如下：

```go
type Writer struct {
	err error
	buf []byte    // 数据缓冲byte切片，数据缓存区，默认长度 4096
	n   int       // 标记已写入当前buf的byte数量，即已使用缓存区的长度
	wr  io.Writer // 底层 io.Writer
}
```

Writer 实现了一个 I/O 对象的缓冲。当向 Writer 对象写数据时，如果发生错误，则其他数据都不会被接收，且后续的写入操作都将返回 error。当所有数据被写入后，客户端应调用 Flush 方法，以确保所有数据都已转发到底层的 io.Writer

### New Writer 的两个函数

#### NewWriterSize(w io.Writer, size int)

```go
func NewWriterSize(w io.Writer, size int) *Writer {
	// ...
}
```

#### NewWriter(w io.Writer)

```go
func NewWriter(w io.Writer) *Writer {
	// defaultBufSize = 4096
	return NewWriterSize(w, defaultBufSize)
}
```

通过源码可以看出，NewWriter 内部其实是调用了 NewWriterSize 函数，并且使用了默认的 buffer(4096) 大小

### Reset(w io.Writer)

`Reset` 丢弃当前 Writer 对象缓存的任何未转发至底层 io.Writer 对象的数据，清除错误，并根据参数 w 重置输出对象

```go
// Reset discards any unflushed buffered data, clears any error, and reset b to write its output to w.
func (b *Writer) Reset(w io.Writer) {
	b.err = nil // 清除错误
	b.n = 0     // 已缓存数据byte数量清空
	b.wr = w    // 重置底层 io.Writer
}
```

### Flush()

`Flush()` 将 Writer 对象缓存的所有数据转发至底层 io.Writer

```go
// Flush writes any buffered data to the underlying io.Writer.
func (b *Writer) Flush() error{}
```

### Available()

返回缓冲区还有多少长度的 byte 可以使用（没有被使用）

```go
// Available returns how many bytes are unused in the buffer.
func (b *Writer) Available() int { return len(b.buf) - b.n }
```

### Buffered()

返回已使用当前 buf 的长度

```go
// Buffered returns the number of bytes that have been written into the current buffer.
func (b *Writer) Buffered() int { return b.n }
```

### Write(p []byte)

将 p 的内容写入 buf ；返回写入的数量；若返回写入数量小于 p 的长度，则最后一个返回值返回一个 error 类型错误，以解释为什么没能将 p 全部写入

```go
// Write writes the contents of p into the buffer.
// It returns the number of bytes written.
// If nn < len(p), it also returns an error explaining
// why the write is short.
func (b *Writer) Write(p []byte) (nn int, err error) {}
```

### WriteByte(c byte)

`WriteByte` 写入单个字节；当有错误发生时，返回至不为nil

```go
// WriteByte writes a single byte.
func (b *Writer) WriteByte(c byte) error {
	// ...
}
```

### WriteRune(r rune)

`WriteRune` 写入单个Unicode

```go
// WriteRune writes a single Unicode code point, returning
// the number of bytes written and any error.
func (b *Writer) WriteRune(r rune) (size int, err error) {
	// ...
}
```

### WriteString(s string)

`WriteString` 写入字符串；第一个返回值，返回写入字符串的byte字节长度；第二个返回值，返回error类型；若写入字符串的byte长度 < len(s)，则第二个返回值，error不为nil，以解释为什么未能全部写入

```go
// WriteString writes a string.
// It returns the number of bytes written.
// If the count is less than len(s), it also returns an error explaining
// why the write is short.
func (b *Writer) WriteString(s string) (int, error) {
	// ...
}
```

### ReadFrom()

`ReadFrom` 实现了 io.ReaderFrom 接口；该方法从参数 w 处读取数据，然后写入 b 的底层 io.Writer；返回值分别为：写入byte的长度、error类型错误；若发生错误，则error返回值不为 nil 

```go
// ReadFrom implements io.ReaderFrom.
func (b *Writer) ReadFrom(r io.Reader) (n int64, err error) {
	// ...
}
```

## type ReadWriter

缓存的输入输出

`ReadWriter` 实现了 io.ReadWriter 接口， 保存了 Reader 和 Writer 指针

结构如下：

```go
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
	*Reader
	*Writer
}
```

### NewReadWriter()

`NewReadWriter` 分配一个新的 ReadWriter 结构体，并返回其内存地址；r 是 bufio.Reader 类型，w 是 bufio.Writer 类型；

```go
// NewReadWriter allocates a new ReadWriter that dispatches to r and w.
func NewReadWriter(r *Reader, w *Writer) *ReadWriter {
	return &ReadWriter{r, w}
}
```
