---
 layout: post
 title: How to make mistakes in go part.1
 date:   2021-08-14 14:00:00 +0800
 subtitle: the best ways to make mistakes
 tags: [golang]
---

前陣子在尋找一個慢跑時候聽的 podcast ，長度希望在 40 ~ 60 mins 左右，最好是英文的！可以練練聽力。有一陣子在聽 NYTimes 的 podcast ，做得非常棒，品質跟討論的深度都很好，不過缺點是有時候都在講美國的政治環境議題，老實說我沒有什麼興趣 （btw 我很喜歡這一篇，在講[美國科技業富豪怎麼避稅](https://www.nytimes.com/2021/06/15/podcasts/the-daily/jeff-bezos-elon-musk-billionaires-taxes.html)的）最後意外發現這個 gopher's podcast  [go time](https://changelog.com/gotime)  先聽完了這一集，覺得蠻有意思的，於是寫了一篇文章是把裡面提到的東西簡單整理一下，可能有些遺漏或是聽錯的，還請多多包涵，有問題的話歡迎來信跟我說。

## 1. nil receiver, accept interface return struct

這是一個蠻經典的問題，我之前也寫過文章 ([Understand Nil](https://tingyuchang.github.io/2021-08-08-understand-nil/)) 研究過，簡單來說：

```go
func main() {
	err := test(false)
	fmt.Println(err) // nil
	if err != nil {
	    fmt.Printf("Type: %T\tValue: %v\n", err, err)
			// Type: *main.myError	Value: <nil>
			panic(err)
	}

	fmt.Println("ture")
}

func test(isError bool) error {
	var err *myError
	if isError {
		err = &myError{value: 1}
	}
    fmt.Println(err) //nil 
	return err
}
```

當回傳值是 error 這個 interface 的時候，對於 golang 來說 Type 跟 Value 都必須是 nil ，才是 nil ，上面的例子因為 err 的 Type 是 *myError，所以對於 interface 的 nil 定義來說，他不是 nil 。

關於這個問題，podcast 中也提到了一個原則：accept interface return struct 。就可以避免這個問題的產生（這個原則希望之後可以寫一篇文章來討論）。簡單來說，從 function 的角度來看，你無法、也不需要知道外部傳入的資料型態是什麼，你只要定義好你所需要的參數抽象即可，如此一來就會讓你的 function 有更多的彈性，反之對於回傳的資料你可以控制，就應該要準確的定義，避免問題的產生。

不過這項原則並不是一定適用於全部 (我認為跟 golang 的 implicit interface 有關)，對於 error 來說 return error interface 的寫法已經變成 gopher 的約定俗成 (還有 io.reader, io.writer...)，這寫法太好用了，連官方、教學、google 等等都這樣寫，所以我們就必須要注意 nil receiver 這個問題。 (完整的範例程式碼我寫在[這邊](https://play.golang.org/p/gyQQqYKiZ61))

## 2. is concurrency faster than sequential?

這個問題，請大家先想想你自己的答案是什麼？

1...

2...

3...

concurrency 應該比較快吧 ← 你的答案是這個嗎？

這感覺不是廢話嘛？有多個 goroutine 去做事當然比一個 goroutine 還要快啊

1 + 2 + .. + 99 + 100 ，我分成四個 goroutine 去做，一個做 25 個，最後在把四個結果彙整，一定比一個 goroutine 做全部的事情還要快吧！ 

其實不一定，可以從兩個觀點來看這件事：

1. Cost ，分成多個 goroutine 需要負擔一定的成本，成本包含記憶空間，以及管理這些 goroutine ，如果沒有處理好，可能會比 sequential 還要慘。podcast 舉了 merge sort 的例子來說明（應該是某一種 divide and conquer  sort ，但講者口音有點重），理論上這種 Divide and Conquer 的演算法很適合用 concurrency 去處理，不過因為需要等待每一個 goroutine 做完事情才會合併，反而不會比 sequential 的做法快。我試著把人家寫好的 merge sort 寫成 concurrency merge sort ，結果是慢了四倍，想說可能是我 concurrency 寫得太爛，就再去找別人寫好的  concurrency merge sort with channel ，結果還是一樣慢.... [code](https://play.golang.org/p/iwOmS8tuK6V) 在這邊，有興趣可以去玩看看。

```go
BenchmarkMergeSort-8               	 1242590	      1003 ns/op
BenchmarkMergeSortConcurrency-8    	  347770	      3250 ns/op
BenchmarkMultiMergeSortWithSem-8   	  192363	      6248 ns/op
```

1. Complex，寫成 concurrency 的程式碼相對比較複雜，代表維護他們的難度是比較高的，這也是會讓人覺得 concurrency 不見得好的一點，特別是對新手來說。

跟其他語言相比， golang 使用 concurrency 的方式十分簡單，但是不代表我們一定要使用 concurrency 的方式去處理我們的問題，sequential 也可以處理得很好！而 Dave Cheney 也說過 [Never start a goroutine without knowing how it will stop](https://dave.cheney.net/tag/goroutines) 。

還有三個 issues ，不過再寫下去有點長了，我會拆成上下兩篇來說明，以上

### Reference

- [https://changelog.com/news/how-to-make-mistakes-in-go-9Oqw](https://changelog.com/news/how-to-make-mistakes-in-go-9Oqw)
- [https://medium.com/@oboturov/golang-time-after-is-not-garbage-collected-4cbc94740082](https://medium.com/@oboturov/golang-time-after-is-not-garbage-collected-4cbc94740082)
- [https://dave.cheney.net/tag/goroutines](https://dave.cheney.net/tag/goroutines)