---
 layout: post
 title: Error Group
 date:   2021-08-21 15:00:00 +0800
 subtitle: goroutine with context and waitgroup
 tags: [golang]
---
之前有提到 Context 可以用來管理 goroutine 的，根據 Context 的結構，可以做水平或是垂直的任務管理，水平的意思是一群平行的任務彼此不互相影響，垂直則是上層的任務會影響下層的任務

![https://i.imgur.com/MrZ7nAM.jpg](https://i.imgur.com/MrZ7nAM.jpg)

Root Ctx 一旦被取消，底下的所有 ctx 都被取消，但如果是 ctx1 被取消，那麼只有 ctx4, ctx5 會被取消，ctx2, ctx3 不會收到影響。

使用 context 管理的 task 可能會這樣這樣寫：

```go
func doSomething(ctx context.Context ...interface{}) error {
	ch := make(chan error)
	go func() {
		/*
			do something
			ch<- err or nil
		*/
		 
	}

	select {
	case <-ctx.Done():
		return ctx.Err()
	case err := <-ch:
		return err
	}
}
```

再配合 waitgroup + error channel 來控制任務的進行

```go
ctxm cancel := context.WithCancel(context.Background())
defer cancel()
ch := make(chan, error)
wg := sync.WaitGroup{}

for i:=0; i < 10; i++ {
	go func() {
		wg.Add(1)
		ch <- doSomething(ctx)
		wg.Done()
		
	}()
}

go func() {
	select {
	case err := <-ch:
		if err != nil {
			cancel()
		}	
	}
}
wg.Wait()
```

Go 也提供了一個更加簡單的 package 來處理這個組合 [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)

> Errgroup provides synchronization, error propagation, and Context cancelation for groups of goroutines working on subtasks of a common task.

這個 package 只有三個 method

- func WithContext(ctx context.Context) (*Group, context.Context)
- func (g *Group) Go(f func() error)
- func (g *Group) Wait() error

先建立 Errgroup 的實體，再使用 Go() 跟 Wait() 即可，errs.Go() 必須傳入一個 func()error，這邊會直接使用 goroutine 來執行，不需要宣告 waitgroup ，只要執行 errs.Wait() ，就會等待所有先前的 errs.Go() 執行完畢，errs.Wait() 會回傳一個 error，邏輯是所有 errs.Go() 中第一個返回 not nil 的 error 。以下是一個簡單的範例

```go
// task simulate a task need to spend duration ms
// if success is false, will return non-nil error
func task(n int, duration time.Duration, success bool) error {
	fmt.Println("start:", n)
	time.Sleep(duration)
	if !success {
		fmt.Println("end:", n)
		return errors.New(fmt.Sprintf("%d: failed", n))
	}
	fmt.Println("end:", n)
	return nil
}

func main() {
	errs := new(errgroup.Group)

	errs.Go(func() error {
		return task(1, 100*time.Millisecond, true)
	})

	errs.Go(func() error {
		return task(2, 5000*time.Millisecond, false)
	})

	errs.Go(func() error {
		return task(3, 500*time.Millisecond, false)
	})

	if err := errs.Wait(); err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println("All task success")

}

/*
start: 3
start: 2
start: 1
end: 1
end: 3 
end: 2 --> wait for task2 
3: failed
*/
```

task1 100ms → success

task2 5000ms → failed

task3 500ms → failed

task1 最快執行完成，task2 最慢， task3 居中，但是是第一個失敗的 task

所以可以看到先等待所有的 task 都完成之後，才會印出 task3 的 error。

另外一個用法則是使用 WithContext() ，邏輯跟上面有一點不同

WithContext()  會根據傳入的 ctx 再建立一個新的 cancelCtx ，保留了 func cancel()

這一個 func cancel() 會在兩個時間點被呼叫

1. 當第一個回傳 non-nil 的 error 的 Go() 完成的時候
2. 所有的 Go() 都完成的時候

所以 WithContext() 回傳的 ctx 別拿去別的地方使用，當所有宣告的 task 都結束之後，這一個 ctx 會被取消。以下是把上面的範例改寫成 context 用的

```go
func test(ctx context.Context, n int, duration time.Duration, timeout time.Duration, success bool) error {
	fmt.Println("start:", n)
	c := make(chan struct{})
	go func() {
		time.Sleep(duration)
		c <- struct{}{}
	}()
	
	select {
	case <-ctx.Done():
		fmt.Println("ctx Done:", n)
		return errors.New(fmt.Sprintf("%d: %v", n, ctx.Err()))
	case <-time.After(timeout):
		return errors.New(fmt.Sprintf("%d: timeout", n))
	case <-c:
		fmt.Println("end:", n)
		if !success {
			return errors.New(fmt.Sprintf("%d: failed", n))
		} else {
			return nil
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	errs, ctx := errgroup.WithContext(ctx)
	timeout := 500 * time.Millisecond
	errs.Go(func() error {
		return test(ctx, 1, 100*time.Millisecond, timeout, true)
	})

	errs.Go(func() error {
		return test(ctx, 2, 5000*time.Millisecond, timeout, false)
	})

	errs.Go(func() error {
		return test(ctx, 3, 400*time.Millisecond, timeout, false)
	})

	go func() {
		time.Sleep(1000 * time.Millisecond)
		cancel()
	}()

	if err := errs.Wait(); err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println("All task success")
}

/*
start: 2
start: 1
start: 3
end: 1
end: 3
ctx Done 2
3: failed
*/
```

task3 的 failed 比 task2 完成的時間早，所以 task2 並沒有完成就收到 ctx 取消的信號，返回 ctx.Err()。

---

### 結論

errorgroup 是一個簡單的 package ，簡單來說它是把 goroutine + context + waitgroup 組合的結果，好處是實現了一個 best practice，大家可以很直覺的使用，如果覺得還有哪邊需要擴充的，也可以自己寫一個 errorgroup ，像是 [kratos](https://github.com/go-kratos/kratos/blob/v1.0.x/pkg/sync/errgroup/errgroup.go) 就增加了 recover() 跟限制了 goroutine 數量的機制。

### Reference

- [https://pkg.go.dev/golang.org/x/sync/errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)
- [https://www.fullstory.com/blog/why-errgroup-withcontext-in-golang-server-handlers/](https://www.fullstory.com/blog/why-errgroup-withcontext-in-golang-server-handlers/)
- [https://github.com/go-kratos/kratos/blob/v1.0.x/pkg/sync/errgroup/errgroup.go](https://github.com/go-kratos/kratos/blob/v1.0.x/pkg/sync/errgroup/errgroup.go)