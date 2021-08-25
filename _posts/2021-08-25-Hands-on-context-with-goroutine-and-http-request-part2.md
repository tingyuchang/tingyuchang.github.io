# Hands on context with goroutine and http request part.2

---
 layout: post
 title: Hands on context with goroutine and http request part.2
 date:   2021-08-25 13:00:00 +0800
 tags: [golang]
---
[上一期](https://tingyuchang.github.io/2021-08-19-Hands-on-context-with-goroutine-and-http-request-part1md/)主要再解釋怎麼設計一個運用 context 管理的 goroutine request ，今天比較簡單，分成兩個部分，一個是說明 github 的 api ，另一個部分是上次的 httpDo 後來在研究 concurrency 的問題時，發現有一個 bug ，在這邊修正一下。

- Read source file to find git repositories.
- Using GitHub api to fetch repository information, such like description, starts, issues.
- Make simple goroutine worker pool to fetch mass repositories.
- Store above data to DB.
- Set up cron job to update information.
- provide api or other to show those data

### Using GitHub api to fetch repository information, such like description, starts, issues.

[GitHub API](https://docs.github.com/en/rest) 的文件在這邊，內容說明得很清楚，主要是使用下面這隻 API

```go
GET /repos/{owner}/{repo}
```

比較值得注意的地方是，因為我們的 repositories 有兩千多個，根據 GitHub API 的限制，沒有 token 的 request 一小時只能請求 60 個，所以請記得去申請 personal token 來使用，有 token 的話，一小時有 5000 個請求額度，應該是很夠用了，不過記得測試 code 的時候一次跑一些就好，不然一下子用掉 2000+ ，5000 很快就見底了。

想要知道目前的 ip 還能用幾個，可以打這支 API 看看，有帶 token 跟沒帶是不同的，可以試看看

```go
https://api.github.com/rate_limit
```

GET repo 的工作相對簡單，建立 request 並填上需要的參數，再呼叫 HttpDo 後，讀取 response 處理後回傳即可：

```go
	req, err := http.NewRequest("GET", GITHUB_API_URL+"repos/"+userRepo, nil)
	req.Header.Set("Accept", viper.Get("http.githubheaderaccept").(string))
	req.Header.Set("Authorization", fmt.Sprintf("token %v", viper.Get("http.githubheaderauthorization")))
	if err != nil {
		return Repository{}, err
	}
	repo := Repository{}
	err = service.HttpDo(ctx, req, func(resp *http.Response, err error) error {
		if err != nil {
			return err
		}
		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			return err
		}

		err = json.Unmarshal(body, &repo)
		if err != nil {
			return err
		}
		return nil
	})
	return repo, err
```

看到這邊我突然想起之前有看到 Dave Cheney 寫過一篇[文章](https://dave.cheney.net/high-performance-json.html)提到原生套件 encoding/json 的 decoding  太慢的問題，之後也許有機會可做看看實驗。

最後想提一個 concurrency blocking bug

```go
func HttpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	// to more safety, give 1 buffer
	c := make(chan error, 1)
	defer close(c)
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

c := make(chan error, 1) 之前有提到是不需要 buffer 的，因為我們有在 ctx timeout 的 case 中讀取

這麼做的原因是因為如果當 c <- f(http.DefaultClient.Do(req)) & <-ctx.Done() 同時滿足條件的時候，select-case 有隨機性，因此我們不知道哪一個 case  會先被執行，但如果是 <-ctx.Done() 的話，直接 return 會造成 c 這個 channel 的關閉，因此 f() 無法傳遞信號給 c ，產生了 block，因此之前的解法是在 case <-ctx.Done(): 中讀取 ←c，以避免 block 的問題。

ctx 的取消，可能是因為 timeout 或是其 cancel func 被呼叫，理論上我們希望的是立即返回錯誤，繼續執行後續的程式，我們會在裡面等 c 的返回後（只要還在等待 response 會有錯誤返回，因為 request 裏面也有 ctx）再來返回 error 。如果沒有等待 c 的返回，就一定要替 c 加上 buffer ，否則就會有很高的機率產生 panic block，為了避免有無法預期的 blocking bug，除了等待 c 的返回之外我覺得最重要的還是要替 c 加上一個 buffer ，這也提醒了我，未來在 select-case 的處理上，當無法 100% 確認可以處理 case 的 channel 的生命週期時，應該還是要加上 buffer 會比較安全一些。

### Reference

- [https://songlh.github.io/paper/go-study.pdf](https://songlh.github.io/paper/go-study.pdf)