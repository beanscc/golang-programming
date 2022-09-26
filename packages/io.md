## IO



## ioutil.NopCloser() 的使用



什么时候使用？





当某个实现了 io.Reader 接口的对象，需要转化为 io.ReaderCloser 接口时。例如 `bytes.Buffer`



标准包中的使用示例：

```go
// 文件位置：net/http/httputil/dump.go
// drainBody reads all of b to memory and then returns two equivalent
// ReadClosers yielding the same bytes.
//
// It returns an error if the initial slurp of all bytes fails. It does not attempt
// to make the returned ReadClosers have identical error-matching behavior.
func drainBody(b io.ReadCloser) (r1, r2 io.ReadCloser, err error) {
	if b == nil || b == http.NoBody {
		// No copying needed. Preserve the magic sentinel meaning of NoBody.
		return http.NoBody, http.NoBody, nil
	}
	var buf bytes.Buffer
	if _, err = buf.ReadFrom(b); err != nil {
		return nil, b, err
	}
	if err = b.Close(); err != nil {
		return nil, b, err
	}
	return io.NopCloser(&buf), io.NopCloser(bytes.NewReader(buf.Bytes())), nil
}
```



上面代码中 buf 变量作为 bytes.Buffer 类型，已实现了 io.Reader 接口，想要将它用做 io.ReadCloser 接口，需要再实现一个 io.Closer 接口即可，而 io.NopCloser() 方法就是提供了这样一个包装，给 io.Reader 对象加一个 无操作的 `Close()` 方法，使其实现了 io.Closer 接口，如下：

```go
// NopCloser returns a ReadCloser with a no-op Close method wrapping
// the provided Reader r.
func NopCloser(r Reader) ReadCloser {
	return nopCloser{r}
}

type nopCloser struct {
	Reader
}

func (nopCloser) Close() error { return nil }
```



