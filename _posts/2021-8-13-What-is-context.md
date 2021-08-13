---
 layout: post
 title: What is context
 date:   2021-08-13 14:00:00 +0800
 subtitle: A Context carries a deadline, a cancellation signal, and other values across API boundaries.
 tags: [golang]
---
context 是 golang 一個獨特的用法，在其他語言比較少看到這個設計，原因是 context 主要是設計用來連接 goroutine 使用的。go 在接收 server request 的時候，都會開啟一個 goroutine 來處理，而且這個 goroutine 可能會再開另一個 goroutine ，比如說 db query, RPC services 等，但是如果其中一個 goroutine 取消或是處理逾時了，這一串的 goroutine  都應該要立即拋棄手邊的工作，以避免佔用系統資源。要怎麼達成管理這一連串的 goroutine ，就是使用 context 了。

context 可以很簡單的在一群 goroutines 中傳遞資料、發送取消信號、以及建立 deadline 的機制等等。

以下是 context 's spec

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

- Deadline: 回傳一個時間，用來表示工作已經結束或是被取消，ok = false 表示沒有設置 deadline。
- Done: 回傳一個 read only 的 channel ，當這個 channel 被關閉的時候，表示工作已經結束，這個 context 要被取消。
- Err: 如 Done 還沒有被 close 的話回傳 nil，Done 被 close 的話，則是回傳一個 none-nil error 用來解釋為什麼被取消。
- Value: context  Key-value  request-scoped data

context 是由上而下的，如果最上層的 context 被取消了，下面全部的 context 也會被取消。但是下面的 context 被取消，並不會影響到上層的 context 。

# Source Code

接下來解釋一下 context 的原始碼跟用法：

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

ctx := context.Background() 

func TODO() Context {
	return todo
}

ctx := context.TODO() 
```

context.Background()  是一個 emptyCtx ，他是一個空的 context ，雖然有實做 Done(), Deadline(), Err(), Value()  不過都是 nil ，代表他永遠不會被取消，一般是當作 context root 使用。

ps. TODO 也類似，不過是用在不確定的情境下，可以先用 TODO 代替。

當使用了 root context 之後，就可以視使用情況運用以下的方法來建立新的 context 

```go
// WithCancel(parent Context) (ctx Context, cancel CancelFunc)
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
doSomeJobs(ctx) 
```

建立一個有 cancel() 的 context ，當執行 cancel() ，←ctx.Done() 就會收到信號。

```go
// WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()
doSomeJobs(ctx) 

// WithDeadline is analopgy with func WithTimeout, but it given time.Time
```

建立一個有 timeout 機制的 context ，當執行 cancel() 或是 timeout 之後 ，←ctx.Done() 就會收到信號。 (WithDeadline 跟 timeout 類似，就不額外說明了)

```go
// WithValue(parent Context, key, val interface{}) Context
ctx := context.WithValue(ctx, userIPKey, userIP) // userip package
```

也可以建立存放資訊的 context。

以上建立 context 的方法可以參照[原始碼](https://github.com/golang/go/blob/2ebe77a2fda1ee9ff6fd9a3e08933ad1ebaea039/src/context/context.go#L232)，其實還蠻好理解的。

---
### Example:

以下示範一個簡單的情境來使用 context ：

```go
func main() {
	ctx1 := context.Background()
	ctx2, cancel2 := context.WithCancel(ctx1)
	ctx3, cancel3 := context.WithDeadline(ctx2, time.Now().Add(1*time.Second))
	defer cancel3()

	go func() {
		err := handle(ctx2, 500*time.Millisecond)
		if err != nil {
			cancel2()
		}

	}()

	select {
	case <-ctx3.Done():
		fmt.Println("ctx3 is finished")
	}
}

func handle(ctx context.Context, duration time.Duration) error {
	select {
	case <-ctx.Done():
		fmt.Println("handle:", ctx.Err())
		return errors.New("something error")
	case <-time.After(duration): // will cause memory leak, but easy to use here
		fmt.Println("process with in", duration)
		return nil
	}
}
```

- ctx1 是 root ，不會被取消
- ctx2 是 ctx1 的 child ，使用了 WithCancel ，所以會有一個 cancel2 的 func 來取消 ctx2
- ctx3 是 ctx2 的 child ，使用了 WithDeadline ，也會有一個 cancel3 的 func 來取消 ctx3

handle 這個 func 模擬了一段工作正在進行，它接收 context, duration 兩個參數，使用了 context 來管理工作的進行，duration 則是告訴這個 func 一個工作要做多久才會做完。select 有兩個 cases：

1. <-ctx.Done(): 表示這個 context 即將要結束，必須執行相對應的處理
2. <-time.After(duration): 在 duration 的時間過後，會完成這個工作

```go
go func() {
		err := handle(ctx2, 500*time.Millisecond)
		if err != nil {
			cancel2()
		}

	}()
```

我們使用 goroutine 來執行 handle ，帶入了 ctx2 以及 500ms 的執行時間，執行完成如果發現問題，就會執行 ctx2 的取消。

最後使用一個 select 等待 ctx3 完成。ctx3 有兩個條件會完成，一個是自己本身帶的 deadline ，會在 1s 之後執行，另一個則是等待 ctx2 的錯誤產生，執行 ctx2 的 cancel2，因為 ctx3 是 ctx2 的 child ，因此也會被連帶影響取消。

執行結果如下：

```go
process with in 500ms
ctx3 is finished
```

可以把執行時間改成 1500ms 試看看，會變成 ctx3 的 deadline 先到達，所以程式就結束了

[程式碼可以看這邊](https://play.golang.org/p/sg2tF3_z7WW)  

---

### Example: Google Search API

這個範例使用了三個 package 

- [server](https://blog.golang.org/context/server/server.go) 負責 server 以及 handleSearch
- [userip](https://blog.golang.org/context/userip/userip.go) 負責 user ip
- [google](https://blog.golang.org/context/google/google.go) 負責 google search 的 query

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // Cancel ctx as soon as handleSearch returns.

		// Check the search query.
    query := req.FormValue("q")
    if query == "" {
        http.Error(w, "no query", http.StatusBadRequest)
        return
    }

    // Store the user IP in ctx for use by code in other packages.
    userIP, err := userip.FromRequest(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    ctx = userip.NewContext(ctx, userIP)
		// Run the Google search and print the results.
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
		if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }

	// ..... print result 
}
```

一開始先定義了 ctx 跟 cancel 這兩個變數，再根據 request 傳來的 timeout ，建立 timeout 的 context。

接下來是 userip 的部分：

```go
// userIP, err := userip.FromRequest(req)
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
}

// ctx = userip.NewContext(ctx, userIP)
func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}

func FromContext(ctx context.Context) (net.IP, bool) {
	// ctx.Value returns nil if ctx has no value for the key;
	// the net.IP type assertion returns ok=false for nil.
	userIP, ok := ctx.Value(userIPKey).(net.IP)
	return userIP, ok
}
```

userip 把會把分析到的 ip 放進 context 的 Value 裏面，另外也提供了把 userip 放入 ctx 以及從 context 中取出 ip 的 method。

```go
func Search(ctx context.Context, query string) (Results, error) {
	// Prepare the Google Search API request.
	req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
	if err != nil {
		return nil, err
	}
	q := req.URL.Query()
	q.Set("q", query)

	if userIP, ok := userip.FromContext(ctx); ok {
		q.Set("userip", userIP.String())
	}
	req.URL.RawQuery = q.Encode()

	// Issue the HTTP request and handle the response. The httpDo function
	// cancels the request if ctx.Done is closed.
	var results Results
	err = httpDo(ctx, req, func(resp *http.Response, err error) error {
		if err != nil {
			return err
		}
		defer resp.Body.Close()

		// Parse the JSON search result.
		// https://developers.google.com/web-search/docs/#fonje
		var data struct {
			ResponseData struct {
				Results []struct {
					TitleNoFormatting string
					URL               string
				}
			}
		}
		if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
			return err
		}
		for _, res := range data.ResponseData.Results {
			results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
		}
		return nil
	})
	// httpDo waits for the closure we provided to return, so it's safe to
	// read results here.
	return results, err
}

// httpDo issues the HTTP request and calls f with the response. If ctx.Done is
// closed while the request or f is running, httpDo cancels the request, waits
// for f to exit, and returns ctx.Err. Otherwise, httpDo returns f's error.
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	// Run the HTTP request in a goroutine and pass the response to f.
	c := make(chan error, 1)
	req = req.WithContext(ctx)
	go func() { c <- f(http.DefaultClient.Do(req)) }()
	select {
	case <-ctx.Done():
		<-c // Wait for f to return.
		return ctx.Err()
	case err := <-c:
		return err
	}
}
```

google search 的部分則是處理對 Google API 的處理

```go
req = req.WithContext(ctx)
```

httpDo 會把之前傳進來的 context 包在 request 中送出去，在建立一個 select-case 等待 request 是否有錯。

```go
err = httpDo(ctx, req, func(resp *http.Response, err error) error {...}
```

回到 search 的部分，在呼叫 httpDo 的時候，宣告了一個處理的 func 處理 request 回應的數據，並再建立一個 response 返回給 client 。

比較有趣的地方是下面第一段 httDo:

```go
	select {
	case <-ctx.Done():
		<-c // Wait for f to return.
		return ctx.Err()
	case err := <-c:
		return err
	}
```

什麼時候會收到 <-ctx.Done() 呢？我們先回到上一層的 context，search 這個 method 的 context 是從哪邊來的呢？  

 

```go
ctx = userip.NewContext(ctx, userIP)

func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

NewContext 是一個 context.WithValue 所以沒有取消機制，因此要再往上找

```go
timeout, err := time.ParseDuration(req.FormValue("timeout"))
if err == nil {
   ctx, cancel = context.WithTimeout(context.Background(), timeout)
} else {
   ctx, cancel = context.WithCancel(context.Background())
}
defer cancel() // Cancel ctx as soon as handleSearch returns.
```

原來在 handleSearch 一開始就建立了 timeout or cancel 的 context。所以當時 timeout 設定的時間到了之後，還沒有取得 response ，就會執行 <-ctx.Done() 了。

### 結論

context.Context 並不是很難理解的設計，只要能理解他的運作方式，在設計分散式處理的系統架構時，一定能派上用場。

ps. 原本以為 gin.Context 跟 context.Context 是類似的東西，不過研究完了之後才知道對象是不一樣的東西，context.Context 串連了 go routine ，而 gin.Context 串連了 http handler。

### Reference

- [https://blog.golang.org/context](https://blog.golang.org/context)
- [https://segmentfault.com/a/1190000022484275](https://segmentfault.com/a/1190000022484275)