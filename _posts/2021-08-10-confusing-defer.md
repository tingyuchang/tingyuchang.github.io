---
 layout: post
 title: Confusing Defer 
 date:   2021-08-10 12:00:00 +0800
 categories: golang
---
### TL;DR

不要亂用 Defer ，老老實實地拿來做釋放資源之類的用途就好

golang 有個特別的關鍵字：**defer**

> Go's defer statement schedules a function call (the deferred function) to be run immediately before the function executing the defer returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return. The canonical examples are unlocking a mutex or closing a file.

defer 會延遲接在後面的 function call，在所屬的 function return 前執行，defer 可以呼叫多次，多個 defer 執行的順序是 LIFO，最適合使用的時機是需要釋放某些資源的場合 ex:  db.close(), file.close(), mutex.unlock()，以及 recovery 。

不用 defer 也可以達成一樣的目的，不過使用 defer 有兩個好處：

1. 開啟了某個資源，就可以在下一行執行 defer close，這樣子比較不會忘記（其實大家都很常忘記）
2. 資源的開跟關的程式碼都在附近，會很容易找到

提醒大家別濫用 defer，先來看看幾個程式碼：

```go
// n = 1, return 3
func test1(n int)(sum int){
	defer func() {
		sum++ // 3
	}()
	sum = n+1 // 2
	return
}
```

sum 不需要宣告的原因是因為在 return value 那邊已經先宣告了，因此 defer 會執行同一個 sum ，讓回傳的 sum = 3  

```go
// n = 1, return 2
func test2(n int)int{
	defer func() {
		n++
	}()
	n++
	return n
}
```

在 return 的時候，已經複製了一份 n 的數據，所以 defer func 的修改不影響回傳值，這一個測試跟 test3 其實是類似的

```go
// n = 1, return 2
func test3(n int)int{
	defer func() {
		n++
	}()
	n++
	x := n
	return x
}
```

x 是複製 n 的資料，所以不會再被 +1 

```go
// n = 1, return 2
func test4(n int)(sum int){
	defer func() {
		sum++
	}()
	sum = 1+n
	return 1 // sum = 1
}
```

最後的一行其實是 sum 被 assign 了 1 的數據，所以 defer 裡面其實是 1+1

```go
// n = 1, return 2
func test5(n int) int {
	defer func(n int) {
		n++
	}(n)
	n++
	return n
}
```

在宣告 defer 的當下，傳入的參數 n 已經被複製一份傳進去 func 裡面了，所以裡面的 n 跟外面的 n 是不一樣的。

有些程式碼的思考不是這麼直覺，所以建議還是不要在 defer 裡面執行邏輯處理比較好 。

最後，可能會看到某些網路上的舊文章說 defer 有效能上的疑慮，如果在意效能請不要常用。在 [Golang 1.14](https://golang.org/doc/go1.14#runtime) 之後，已經優化了 defer 的效能，大部分的情況下，defer 跟直接呼叫 func 是沒有差異的。

### Reference

- [https://golang.org/doc/effective_go#defer](https://golang.org/doc/effective_go#defer)