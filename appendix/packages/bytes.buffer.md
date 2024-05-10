# bytes

## type Buffer

用于编码的简单字节缓冲区

`Buffer` 是一个关于bytes的可变大小的缓冲区结构体对象，同时具有 `Read` 和 `Write` 方法； `Buffer` 的零值不用初始化直接可用的

结构定义如下：

```go
// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
	buf       []byte   // contents are the bytes buf[off : len(buf)]
	off       int      // read at &buf[off], write at &buf[len(buf)]
	bootstrap [64]byte // memory to hold first slice; helps small buffers avoid allocation.
	lastRead  readOp   // last read operation, so that Unread* can work correctly.
}
```

### New Buffer

大部分场景下，使用 `New(Buffer)` 或者直接声明一个Buffer变量，就足够初始化一个 `Buffer`结构体对象

#### NewBuffer(buf []byte) 

`NewBuffer` 创建并初始化了一个新的 `Buffer`, 使用byte切片 buf 作为初始化内容。它的目的是准备一个缓冲区来读取现有数据。它也可以被用来调整写入的内部buf的大小，为此，buf应该具有所需的容量，但长度为0。

```go
// NewBuffer creates and initializes a new Buffer using buf as its initial
// contents. It is intended to prepare a Buffer to read existing data. It
// can also be used to size the internal buffer for writing. To do that,
// buf should have the desired capacity but a length of zero.
//
// In most cases, new(Buffer) (or just declaring a Buffer variable) is
// sufficient to initialize a Buffer.
func NewBuffer(buf []byte) *Buffer {
	return &Buffer{buf: buf} 
}
```

#### NewBufferString(s string) 

`NewBufferString` 创建并初始化里一个新 `Buffer`，使用字符串 s 作为初始化内容。它的目的是准备一个缓冲区来读取现有字符串。

```go
// NewBufferString creates and initializes a new Buffer using string s as its
// initial contents. It is intended to prepare a buffer to read an existing
// string.
//
// In most cases, new(Buffer) (or just declaring a Buffer variable) is
// sufficient to initialize a Buffer.
func NewBufferString(s string) *Buffer {
	return &Buffer{buf: []byte(s)}
}
```

### Bytes()

`Bytes` 方法返回 Buffer 缓冲区中未读取的字节切片

```go
// Bytes returns a slice of length b.Len() holding the unread portion of the buffer.
// The slice is valid for use only until the next buffer modification (that is,
// only until the next call to a method like Read, Write, Reset, or Truncate).
// The slice aliases the buffer content at least until the next buffer modification,
// so immediate changes to the slice will affect the result of future reads.
func (b *Buffer) Bytes() []byte { return b.buf[b.off:] }
```

### String()

`String` 返回buffer 未读取的部分的字符串形式； 

> **`注意：`** 若 buffer 是一个空指针，则返回 "<nil>" 这个字符串

```go
// String returns the contents of the unread portion of the buffer
// as a string. If the Buffer is a nil pointer, it returns "<nil>".
func (b *Buffer) String() string {
	if b == nil {
		// Special case, useful in debugging.
		return "<nil>"
	}
	return string(b.buf[b.off:])
}
```

### Len()

`Len` 方法返回缓冲区中未读取的字节数； b.Len() == len(b.Bytes())

```go
// Len returns the number of bytes of the unread portion of the buffer;
// b.Len() == len(b.Bytes()).
func (b *Buffer) Len() int { return len(b.buf) - b.off }
```

### Cap()

`Cap` 方法，返回缓冲区底层的byte 切片的容量，也就是缓冲区的总容量

```go
// Cap returns the capacity of the buffer's underlying byte slice, that is, the
// total space allocated for the buffer's data.
func (b *Buffer) Cap() int { return cap(b.buf) }
```

### Truncate(n int)

`Truncate` 方法，只保留前 n 个未读的字节，其余的全部从缓冲区中清除，但是缓冲区还是之前分配的那个缓冲区； 若 n < 0 or n > b.Len() 则会panic

```go
// Truncate discards all but the first n unread bytes from the buffer
// but continues to use the same allocated storage.
// It panics if n is negative or greater than the length of the buffer.
func (b *Buffer) Truncate(n int) {
	// ...
}
```

### Reset()

`Reset` 方法，内部调用 `Truncate()` 方法实现，只是不保留任何未读字节，将缓冲区全部清除、置空，保留底层存储以留后用

```go
// Reset resets the buffer to be empty,
// but it retains the underlying storage for use by future writes.
// Reset is the same as Truncate(0).
func (b *Buffer) Reset() { b.Truncate(0) }
```

### Grow(n int)

`Grow` 方法，若有必要，增长缓冲区buffer的容量，以保证另外 n 个字节的空间。 `Grow(n)` 之后，至少可以将 n 个字节写入缓冲区，而无需另外分配。若 n < 0，则会 panic；若缓冲区不能増长，它将会产生 ErrTooLarge 的 panic

```go
// Grow grows the buffer's capacity, if necessary, to guarantee space for
// another n bytes. After Grow(n), at least n bytes can be written to the
// buffer without another allocation.
// If n is negative, Grow will panic.
// If the buffer can't grow it will panic with ErrTooLarge.
func (b *Buffer) Grow(n int) {
	if n < 0 {
		panic("bytes.Buffer.Grow: negative count")
	}
	m := b.grow(n)
	b.buf = b.buf[0:m]
}
```

### Write(p []byte)

`Write` 方法，将内容 p 追加到缓冲区，同时，按需増长缓冲区容量。 返回值 n 表示 p 的长度； err 总是 nil。若缓冲区 buffer 变得太大， 方法将产生 ErrTooLarge 的 panic

```go
// Write appends the contents of p to the buffer, growing the buffer as
// needed. The return value n is the length of p; err is always nil. If the
// buffer becomes too large, Write will panic with ErrTooLarge.
func (b *Buffer) Write(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	m := b.grow(len(p))
	return copy(b.buf[m:], p), nil
}
```

### WriteString(s string) 

`WriteString` 方法，将字符串 s 的内容追加到缓冲区，根据需要增长缓冲区容量。返回值 n 是 s 的长度； err 总是 nil。若缓冲区变得太大，方法将产生 ErrTooLarge 的 panic。

```go
// WriteString appends the contents of s to the buffer, growing the buffer as
// needed. The return value n is the length of s; err is always nil. If the
// buffer becomes too large, WriteString will panic with ErrTooLarge.
func (b *Buffer) WriteString(s string) (n int, err error) {
	b.lastRead = opInvalid
	m := b.grow(len(s))
	return copy(b.buf[m:], s), nil
}
```

### ReadFrom(r io.Reader)

`ReadFrom` 方法，从 r 中读取数据，直到 io.EOF，并将读取到的内容追加到缓冲区，按需増长缓冲区容量。返回值 n 就是读取到的字节长度。 err 是 读取过程中发生的错误（除了 io.EOF）。若缓冲区变得太大， 将产生 ErrTooLarge 的 panic

> **`注意：`** `ReadFrom` 方法，从 r 中读取数据，直到遇到 io.EOF 才读取结束


```go
// ReadFrom reads data from r until EOF and appends it to the buffer, growing
// the buffer as needed. The return value n is the number of bytes read. Any
// error except io.EOF encountered during the read is also returned. If the
// buffer becomes too large, ReadFrom will panic with ErrTooLarge.
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
	// ...
}
```

### WriteTo(w io.Writer)

`WriteTo` 方法，将数据写入 w ，直到 b 的缓冲区读完耗尽或发生错误。返回值 n 是写入 w 的字节数； err 表示写入操作过程中发生的错误。

> 若 w 的缓冲区容量小于 b 的缓冲区中未读字节数，则返回 io.ErrShortWrite 类型 err； 写操作完成后，清空 b 的缓冲区

```go
// WriteTo writes data to w until the buffer is drained or an error occurs.
// The return value n is the number of bytes written; it always fits into an
// int, but it is int64 to match the io.WriterTo interface. Any error
// encountered during the write is also returned.
func (b *Buffer) WriteTo(w io.Writer) (n int64, err error) {
	// ...
}
```

### WriteByte(c byte)

`WriteByte` 方法，将字节 c 追加到缓冲区，根据需要増长缓冲区容量。返回的 err 总是 nil，但包含在内以匹配 bufio.Writer 的 WriteByte 。若缓冲区太大，将会返回 ErrTooLarge 的 panic

```go
// WriteByte appends the byte c to the buffer, growing the buffer as needed.
// The returned error is always nil, but is included to match bufio.Writer's
// WriteByte. If the buffer becomes too large, WriteByte will panic with
// ErrTooLarge.
func (b *Buffer) WriteByte(c byte) error {
	b.lastRead = opInvalid
	m := b.grow(1)
	b.buf[m] = c
	return nil
}
```


### WriteRune(r rune)

`WriteRune` 方法，将 Unicode 码 r 的 UTF-8 编码格式追加到缓冲区，返回它的长度和一个总是 nil 的错误，但是被包括来匹配 bufio.Writer 的 WriteRune。缓冲区根据需要増长；若它变得太大，方法将返回 ErrTooLarge 的 panic

```go
// WriteRune appends the UTF-8 encoding of Unicode code point r to the
// buffer, returning its length and an error, which is always nil but is
// included to match bufio.Writer's WriteRune. The buffer is grown as needed;
// if it becomes too large, WriteRune will panic with ErrTooLarge.
func (b *Buffer) WriteRune(r rune) (n int, err error) {
	// ...
}
```

### Read(p []byte)

`Read` 方法，从 b 的缓冲区中读取 len(p) 个字节或直到缓冲区读完耗尽，将内容读入 p 中。返回值 n 表示读入 p 的字节长度； err 表示读取过程中遇到的错误，若 缓冲区没有数据， err = io.EOF （除了 len(p) == 0）

```go
// Read reads the next len(p) bytes from the buffer or until the buffer
// is drained. The return value n is the number of bytes read. If the
// buffer has no data to return, err is io.EOF (unless len(p) is zero);
// otherwise it is nil.
func (b *Buffer) Read(p []byte) (n int, err error) {
	// ...
}
```

### Next(n int)

`Next`方法，从缓冲区中返回一个包含下一个 n 个字节的切片，缓冲区读取记录向前移动，就像字节是 Read 方法返回的一样。若缓冲区中少于 n 个字节， 方法返回整个缓冲区内容。~~`切片只有在下次调用读取或写入方法时才有效`~~

```go
// Next returns a slice containing the next n bytes from the buffer,
// advancing the buffer as if the bytes had been returned by Read.
// If there are fewer than n bytes in the buffer, Next returns the entire buffer.
// The slice is only valid until the next call to a read or write method.
func (b *Buffer) Next(n int) []byte {
	// ...
}
```

> `切片只有在下次调用读取或写入方法时才有效` 注释中这句我在上面画了删除线，表示不太懂，为什么说"返回切片只有在下次调用读取或写入方法时才有效"。   
> 我做了个简单的测试demo，没有看出来why，以下，是测试demo

```go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	s := "this is an test string"
	buf := bytes.NewBufferString(s)

	firstByte, err := buf.ReadByte()

	fmt.Printf("read first byte from string s, want 't', ret: '%v', err: %v \n", string(firstByte), err)

	next3 := buf.Next(3)

	fmt.Printf("read next 3 bytes, want 'his', ret: %v \n", string(next3))

	p := make([]byte, 3)
	n, err := buf.Read(p)
	fmt.Printf("read 3 bytes, want p ' is', ret: '%v', read len=%v, err: %v \n", string(p), n, err)

	fmt.Printf("read next 3 bytes, want 'his', ret: %v \n", string(next3))
}
// 运行结果：
// read first byte from string s, want 't', ret: 't', err: <nil>
// read next 3 bytes, want 'his', ret: his
// read 3 bytes, want p ' is', ret: ' is', read len=3, err: <nil>
// read next 3 bytes, want 'his', ret: his
```

### ReadByte()

`ReadByte` 方法，读取缓冲区中下一个字节并返回；若缓冲区中无内容，则返回 err = io.EOF 的错误。

```go
// ReadByte reads and returns the next byte from the buffer.
// If no byte is available, it returns error io.EOF.
func (b *Buffer) ReadByte() (byte, error) {
	// ...
}
```

### ReadRune()

`ReadRune` 方法，读取并返回缓冲区中下一个 UTF-8 编码的 Unicode 代码点。若没有字节可用，则返回 err = io.EOF 的错误；若字节是错误的 UTF-8 编码，则消耗一个字节并返回 U+FFFD, 1

```go
// ReadRune reads and returns the next UTF-8-encoded
// Unicode code point from the buffer.
// If no bytes are available, the error returned is io.EOF.
// If the bytes are an erroneous UTF-8 encoding, it
// consumes one byte and returns U+FFFD, 1.
func (b *Buffer) ReadRune() (r rune, size int, err error) {
	// ...
}
```

### UnreadRune()

`UnreadRune` 方法，恢复 `ReadRune` 方法返回的最后一个 rune 字符。若缓冲区上的最近读取或写入操作不是 `ReadRune`，则 `UnreadRune` 将返回错误。（在这方面，它比 UnreadByte 更严格，它将读取任何读操作的最后一个字节）

```go
// UnreadRune unreads the last rune returned by ReadRune.
// If the most recent read or write operation on the buffer was
// not a ReadRune, UnreadRune returns an error.  (In this regard
// it is stricter than UnreadByte, which will unread the last byte
// from any read operation.)
func (b *Buffer) UnreadRune() error {
	// ...
}
```

### UnreadByte()

`UnreadByte` 方法，恢复最近读取操作返回的最后一个字节。若上次读取以来发生里写入，则 UnreadByte 将返回错误。

```go
// UnreadByte unreads the last byte returned by the most recent
// read operation. If write has happened since the last read, UnreadByte
// returns an error.
func (b *Buffer) UnreadByte() error {
	// ...
}
```

### ReadBytes(delim byte)

`ReadBytes` 方法，读取，直到输入中第一次出现 delim，返回一个直到分隔符 delim(包含分隔符)的字节切片。若 `ReadBytes` 在查找到分隔符 delim 之前遇到错误，它将返回在错误之前读取的数据和错误本身（通常是 io.EOF）。当且仅当返回的数据不以 delim 结尾时， `ReadBytes` 返回 err != nil 的错误

```go
// ReadBytes reads until the first occurrence of delim in the input,
// returning a slice containing the data up to and including the delimiter.
// If ReadBytes encounters an error before finding a delimiter,
// it returns the data read before the error and the error itself (often io.EOF).
// ReadBytes returns err != nil if and only if the returned data does not end in
// delim.
func (b *Buffer) ReadBytes(delim byte) (line []byte, err error) {
	slice, err := b.readSlice(delim)
	// return a copy of slice. The buffer's backing array may
	// be overwritten by later calls.
	line = append(line, slice...)
	return
}
```

### ReadString()

`ReadString` 方法，读取，直到输入中第一次出现 delim，返回一个直到分隔符 delim(包含分隔符)的字符串。若 `ReadString` 在查找到分隔符 delim 之前遇到错误，它将返回在错误之前读取的数据和错误本身（通常是 io.EOF）。当且仅当返回的数据不以 delim 结尾时， `ReadString` 返回 err != nil 的错误

```go
// ReadString reads until the first occurrence of delim in the input,
// returning a string containing the data up to and including the delimiter.
// If ReadString encounters an error before finding a delimiter,
// it returns the data read before the error and the error itself (often io.EOF).
// ReadString returns err != nil if and only if the returned data does not end
// in delim.
func (b *Buffer) ReadString(delim byte) (line string, err error) {
	slice, err := b.readSlice(delim)
	return string(slice), err
}
```