---
 layout: post
 title: Embedded Struct
 date:   2021-08-18 11:20:00 +0800
 subtitle: 
 tags: [golang]
---

golang 透過 embedded struct 實現類似繼承的概念，但是這個特性不是繼承，比較接近的想法應該是 struct composition ，在一個 struct 內，如果有多個 embedded interfaces or structs ，這一個 struct 就擁有了這些 interface or struct 的方法，概念上是很容易明白的，不過實際上。

具體來說，有這幾種實現的方式：

1. embedded struct (or struct pointer) in struct 
2. embedded interface in interface
3. embedded interface in struct 

### Embedded struct in struct

說明一下什麼是 embedded struct 

```go
type Person struct {
 name string
 age int 
}

func (p *Person) SayHello() {
 fmt.Println("Hello, im", p.name)
}

type Coder struct {
 Person
}

type Gopher struct {
 Person
}
```

上述的例子中， Coder 跟 Gopher 都 embedded Person 這一個 struct ，所以我們會看到這樣的使用方式

```go
coder := Coder{Person{"mika", 18}}
coder.SayHello()
// Hello, im mika
gopher := Gopher{Person{"matt", 18}}
gopher.SayHello()
// Hello, im matt
```

Coder 跟 Gopher 都有擁有 Person，所以他們也自動擁有 SayHello() 這個 method，或是應該說 

SayHello() 被 promote 到了上一層的 struct ，同樣的道理 Person 的 varabile 也一樣 [coder.name](http://coder.name) or coder.age 都可以被正常使用，等於 coder.Person.name, coder.Person.age。

當使用有同樣名稱的 variable 或是 method 的時候，會以最上層的為優先，下一層或下下一層的會被覆蓋掉，我們把上面的 Coder 修改一下來示範：

```go
type Coder struct {
 Person
 name string // type of coder
}

// same name method
func (c *Coder) SayHello() {
  fmt.Println("Hello, im", c.Person.name, c.name)
}
```

因此，當呼叫 coder.SayHello() 的時候，就會改成 func (c *Coder) SayHello() 這個方法，想要再呼叫原本的 SayHello() ，就必須寫成 coder.Person.SayHello()。同理 variable name 也是一樣的邏輯。

embedded struct 感覺非常像多重繼承，可能是因為 promote method 的關係，不過實際上他並不是繼承而是組合 (composition, a has b, not a is b) ，Person 是實際存在的變數，能透過 coder.Person 來操作它。

### Embedded interface in interface

embedded interface 比較簡單，可以想成是方法的擴充以及重新定義 interface 的 scope，最有名的範例就是 io.Reader

```go
type Reader interface {
  Read(p []byte) (n int, err error)
}
type Writer interface {
  Write(p []byte) (n int, err error)
}
type Closer interface {
  Close() error
}
type ReadCloser interface {
  Reader
  Closer
}
type WriteCloser interface {
  Writer
  Closer
}
type ReadWriteCloser interface {
  Reader
  Writer
  Closer
}
```

以上全部都是 interface ，透過組合 Reader, Writer, Closer 來產生新的 interface，可能會有人覺得為什麼不直接使用 ReadWriteCloser  這個組合大禮包就好，這是因為 [Single Responsibility  Principle](https://dave.cheney.net/2016/08/20/solid-go-design) 的緣故。

有的用法則是要擴充既有的 interface ，例如 net.Error

```go
// An Error represents a network error.
type Error interface {
  error
  Timeout() bool   // Is the error a timeout?
  Temporary() bool // Is the error temporary?
}
```

增加了兩個新的方法，也還是 error ，因此在 error 適用的情境下依然能夠使用。

### Embedded interface in struct

這個意思是指任何實作 embedded interface 的物件，都可以當作這個 interface，成為 struct 的變數。主要的用途有:

1. interface wrapper

```go
type StatsConn struct {
  net.Conn
  BytesRead uint64
}

func (sc *StatsConn) Read(p []byte) (int, error) {
  n, err := sc.Conn.Read(p)
  sc.BytesRead += uint64(n) // sum all byte has been read
  return n, err
}
```

StatsConn  不需要重寫所有的 Conn 的方法，只要對需要的部分重新定義即可。

類似的做法還有 sort.Reverse，這個作法實在很巧妙

```go
type reverse struct {
 // This embedded Interface permits Reverse to use the methods of
 // another Interface implementation.
 Interface
}

// Less returns the opposite of the embedded implementation's Less method.
func (r reverse) Less(i, j int) bool {
 return r.Interface.Less(j, i)
}

// Reverse returns the reverse order for data.
func Reverse(data Interface) Interface {
 return &reverse{data}
}
```

Reverse 會返回一個私有的 struct reverse ，reverse 只有重寫 Less()，把原本的變數改成相反的，因此當我們呼叫原本的排序 sort.IntSlice{} 的時候，原本的大小定義也顛倒了，我們就能得到一個反過來的排列。

```go
func (x IntSlice) Less(i, j int) bool { return x[i] < x[j] }
```

1. 限制 interface (downgrade capacity)

前面提到 embedded 的時候，大多都是擴充的想法，不過其實也可以逆向操作，限制原本擁有太多 interface 的 struct，以下將會用 os.File 來解釋

```go
func (f *File) Read(b []byte) (n int, err error)
func (f *File) ReadFrom(r io.Reader) (n int64, err error)
func (f *File) Write(b []byte) (n int, err error)
```

os.file 實作了很多方法，這邊只列出幾個需要的部分，我們可以看到 os.File 本身實現了以下的 interface 

```go
type ReadWriter interface
type ReaderFrom interface
```

要討論的問題在於 os.File.ReadFrom()

```go
func (f *File) ReadFrom(r io.Reader) (n int64, err error) {
	if err := f.checkValid("write"); err != nil {
		return 0, err
	}
	n, handled, e := f.readFrom(r)
	if !handled {
		return genericReadFrom(f, r) // without wrapping
	}
	return n, f.wrapErr("write", e)
}

func genericReadFrom(f *File, r io.Reader) (int64, error) {
	return io.Copy(onlyWriter{f}, r)
}

type onlyWriter struct {
	io.Writer
}
```

n, handled, e := f.readFrom(r)  在不同的 os 有不同的實現方式有點複雜，且跟主題無關這邊就不說明了，重點是當 handle == false 的時候，會改呼叫 genericReadFrom 這一個方法。而 genericReadFrom 只做了一件事，把 f 變成 onlyWriter{}，onlyWriter 的名子取的很好，就是只有 Writer ，於是我們把 os.File 變成了一個 onlyWriter，為什麼這麼麻煩呢？答案在 io.Copy() 裏面

```go
// io.go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	// too long, no need
}
```

io.Copy() 接收的是 io.Writer 的 interface ，實際上 os.File 是符合的，但是在 copyBuffer() 裏面會替 dst 做一次型別斷言，問題就在這邊，如果發現 dst 也是 io.ReadFrom 的話，就會呼叫 ReadFrom()

```go
if rt, ok := dst.(ReaderFrom); ok {...}
```

看到這裡是不是覺得怪怪的，怎麼又回到了原本的方法，我們不是就是一路從 ReadFrom() 走下來的嗎？怎麼又跳回去了？形成了邏輯上的無窮迴圈，因此才需要把 os.File 偽裝 (downgrade) 成 onlyWriter{} 才不會導致這個結果！

### More examples

接下來用 golang 的 source code 來舉例，可以透過真實的案例來了解embedded 要怎麼使用。

以 context.Context 這個 package 來說

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
type cancelCtx struct {
	Context              // interface
	mu       sync.Mutex            
	done     chan struct{}         
	children map[canceler]struct{} 
	err      error                 
}
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.
	deadline time.Time
}
```

Context：interface ，是 context package 最重要的部分，所有 context struct 都實現了 Context 的方法，但是他們並沒有開放，因此在外部使用的時候，我們通常是呼叫 context.Context 。

cancelCtx: private struct ，這是 embedded Context interface ，任何有實現 Context interface 的東西都可以使用在這裡。cancelCtx 重新定義了 Value(), Err(), Done() 這幾個 method，以實現 cancel 。

timerCtx: private struct，有 timeout 機制的 context ，跟 cancelCtx 非常像，只有重新定義 Deadline() 這個方法。

timerCtx  是基於 cancelCtx 再加上了 deadline 的概念，透過 embedded 的方式，context 創造出不同用途的 context ，只需要對不同的地方進行 overriding  ，減少了重複的工作，所有的 private context 都有一個符合 Context 的變數，這邊我們可以把這一個 Context 視為父節點，來進行 Context 的邏輯操作（cancelation）。

另外有時候你會看到這樣的 interface

```go
// bufio.go
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
	*Reader // bufio.Reader
	*Writer // bufio.Writer
}

func (b *Reader) Read(p []byte) (n int, err error) {}
func (b *Writer) Write(p []byte) (nn int, err error) {}
```

這是在 bufio 裡面定義的 ReadWriter，這個 struct 有兩個分別指向 Reader & Writer的指針

這是因為 *Reader 與 *Writer 才有實現 Read(), Write() 這兩個方法，有興趣可以看看[這篇討論](https://stackoverflow.com/questions/33587227/method-sets-pointer-vs-value-receiver)。

### 結論

embedded struct 是一個非常方便的功能，可以讓我們少寫很多程式碼，不過 embedded struct 並不是繼承，以下是可能會用到 embedded struct 的情境

- 擴充 struct or interface
- 只需要 overriding 少部分的 method
- 限制 struct 的 scope (downgrade interface)

### Reference

- [https://eli.thegreenplace.net/2020/embedding-in-go-part-1-structs-in-structs/](https://eli.thegreenplace.net/2020/embedding-in-go-part-1-structs-in-structs/)
- [https://eli.thegreenplace.net/2020/embedding-in-go-part-2-interfaces-in-interfaces/](https://eli.thegreenplace.net/2020/embedding-in-go-part-2-interfaces-in-interfaces/)
- [https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)
- [https://github.com/golang/go/issues/22013#issuecomment-331886875](https://github.com/golang/go/issues/22013#issuecomment-331886875)