---
 layout: post
 title: Hands on context with goroutine and http request part.1
 date:   2021-08-19 12:00:00 +0800
 subtitle: 
 tags: [golang]
---
之前的文章有寫了 context.Context 的介紹。不過我想再把 context 的應用寫得更加深入一些，找一些實際使用上的案例來說明，於是我想到了一個需求：

當我在看 [awesome-go](https://pkg.go.dev/context#example_WithValue) 的時候，裡面的 repo 實在是太多，即使他們有做分類，但是在我點進去之前，往往對這個一個 repo 的資訊只有 description ，我不知道這一個 repo 是不是還有在更新？星星數夠不夠多？如果要找到不錯的 repo，星星是一個蠻好的參考標準。因此我在想能不能寫一個簡單的服務來幫助我從 awesome-go 提供的資訊中，得到更多對我有用的訊息，這就是 [githubstarseeker](https://github.com/tingyuchang/gitstarseeker) 的開始。

這一個 project 會分成幾個部分來說明：

- Read source file to find git repositories.
- Using github api to fetch repository information, such like description, starts, issues.
- Make simple goroutine worker pool to fetch mass repositories.
- Store above data to DB.
- Set up cron job to update information.
- provide api or other to show those data

### Read source file to find git repositories

第一個部分是要拿到所有 awesome-go 裏面關於 `owner/repo` 的值，這是因為 github 的 api 要有上述的資料才可以查詢 [GET /repos/{owner}/{repo}](https://docs.github.com/en/rest/reference/repos#get-a-repository)

剛好 awesome-go 的 README 提供了這些資料，所以我們第一步就是要先使用 GET 拿到 README 的 raw data。

一般來說會使用 http.GET() ，這個方法很方便，也很簡單易用，但是我們需要使用 context 來管理的時候，就不是這麼方便了，因為這個方法會直接建立一個新的 http.Request 並執行它，而無法讓我們自行建立 request with context ，所以這邊會用建立 request 再使用 client.Do() 的做法

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

req, err := http.NewRequest("GET", url, nil)
if err != nil {
	log.Print(err)
	return nil
}
req = req.WithContext(ctx)
http.DefaultClient.Do(req)
```

我們設定了 3s 的 timeout ，雖然要建立 http timeout 也可以從 client 去建立 timeout

ex:

```go
client := &http.Client{Timeout: 3*time.Second}
client.Do(req)
```

不過使用 context 的好處是可以對每一個 request 做自己的 timeout ，以彈性來說，會比在 client 做好一些。

接下來是要讓 client 發 request ，get readme 其實只有一個 request ，而且並不需要等待太久，所以可以直接呼叫而不使用 goroutine ，但如果設計一個 context, request 與 goroutine 一起使用的 method ，將來就能直接沿用到 get repo api，因此還是選擇用使用 goroutine 來執行。

執行 http request 的時候會開一個新的 goroutine ，所以我們必須要在原本的 goroutine 裏面等待 http response 返回 ，否則 function 就會直接結束。一般來說有兩種方法來等待 goroutine 完成，一個是 sync.Waitgroup 另一個是 channel block，這裡我們選擇 channel block ，原因是因為 context 的 Done() 是一個 channel，只要把 http response 完成後回傳的資料改用 channel 傳遞，我們就可以使用 select-case 來做 block，既可以監控 context 也可以管理完成的 response 。

HttpDo 的方法是參考 golang 官方的範例，我們需要 context, request 以及一個 func(*http.Response, error) error，這個 func 是用來處理 http response 使用的，任何一個呼叫 HttpDo 的 method 都可以建立自己的 func 來處理 response，HttpDo 只專注在 context 與 http 之間的關係，這個 f func 會回傳一個 error ，代表在 response 處理中發生的 error。

當我們取得這一個 error (也可能是 nil, error is value, you know)的時候，代表 HttpDo 的工作應該要結束了，因為 response 已經被接收處理了，我們在 select-case 中接收這一個訊息，讓堵塞的 goroutine 可以繼續進行。

select-case 的部分會有兩個 channel 要處理，一個是 ctx.Done() ，當 context timeout 或是 cancel func() 被呼叫的時候，就代表這一個 request 已經不重要了，可以直接返回，避免資源的消耗。另一個是 response 的 error return ，前面有提到，當 response 處理完畢返回後，HttpDo 的工作也要結束。

select-case 這邊有點要特別注意的是，當我們 ←ctx.Done() 比 ←c 早發生的時候，代表 ctx 已經 timeout 或是上一層的 cancel func 被呼叫，這時候不能直接 return ，因為我們有宣告 defer close(c) ，如果 return 的話，c 就會被 close 掉，但是執行 http 的 goroutine 可能已經要回傳，來不及取消，如果傳送訊息到已經被 closed 的 channel ，就會造成 `panic: send on closed channel`，所以要在裏面接收 ←c，也不用太擔心會等很久，因為 http request 會知道要取消，所以如果進度是在 http request 那邊，會直接返回 context 的錯誤，如果是在 func response handling 的話，就是撰寫 func 的人要去注意，不應該在 http response 中做太複雜的操作 ex: DB 。 

```go
func HttpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	// i think buffer is not necessary here, but the example used
	c := make(chan error)
	defer close(c)
	// include request with request
	req = req.WithContext(ctx)
	// run http request on goroutine, and pass receive values to f
	go func() {
		c <- f(http.DefaultClient.Do(req))
	}()

	select {
	case <-ctx.Done():
		// context is end, but need waiting f() to return
		// otherwise running f() will send to close channel
		<-c
		return ctx.Err()
	case err := <-c:
		// normal case, receive from f()
		return err
	}
}
```

實際呼叫的例子如下：

```go
func ReadSource(url string) []string {
	// TODO: user could set any type source ex: http or file
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		log.Print(err)
		return nil
	}
	var result []string
	err = service.HttpDo(ctx, req, func(resp *http.Response, err error) error {
		data, err := io.ReadAll(resp.Body)
		if err != nil {
			return err
		}
		repos, err := findGithubRepos(data)
		if err != nil {
			return err
		}
		result = repos
		return nil
	})

	return result
}
```

第一部分的介紹就先到這裡，今天最重要的部分是解釋 HttpDo 的原理，下一個部分我們會把拿到的 2000+ repositories 做大量的查詢，有沒有使用 goroutine 的差別就會很明顯了。
