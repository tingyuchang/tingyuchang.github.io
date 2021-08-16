---
 layout: post
 title: How to make mistakes in go part.2
 date:   2021-08-16 22:00:00 +0800
 subtitle: the best ways to make mistakes
 tags: [golang]
---

這是一篇聽完 podcast 的心得分享，因為上集有點長，所以拆成兩個部分，這是下集

[上集在這裡](https://tingyuchang.github.io/2021-08-14-How-to-make-mistakes-in-go-part1/)

## 3. Generic

這是個常見的主題，贊成或是反對兩邊的人大概是一半一半，Podcast 中的討論其實也沒有說一定對這件事給出一個答案，我想這件事就大家自己判斷吧～

這樣寫感覺有點偷懶，所以列一下什麼是 generic

> Generic programming means writing functions and data structures where some types are left to be specified later

**Advantage:**

- 方便
- 減少重複的程式碼

**DisAdvantage:**

- 效能降低
- 難以閱讀

## 4. "time.After" causes memory leak

這是一個很容易犯的小錯誤，相信大家都看過類似的 code

```go
for {
	select{
	case <- ch:
		// do something
	case <- time.After(5*time.Minute):
		// timout handle
	}
}
```

正常使用下，time.After 是會被 GC 回收的

> The underlying Timer is not recovered by the garbage collector until the timer fires.

但是要等設定的時間到了（fires）之後才會被 GC 回收，所以這樣會有什麼問題呢？

如果你的 ← ch 是一個非常頻繁被使用的 channel ，比如說一秒有 60k 訊息通過這個 channel ，則會在第一個 timer 被GC 回收之前，產生 18M timer 出來（60K * 5mins ）把你的 memory 搞爆

所以如果你的程式符合這個情境，請快點去修改吧！

改法如下：

```go
	duration := 5 * time.Minute
	timer := time.NewTimer(duration)
	defer timer.Stop()
	for {
		timer.Reset(duration)
		select {
		case <- timer.C:
			//
		}
	}
```

## 5. make everything public

 如果有人只是不知道大寫是 public ，小寫是 private ，那就沒事了，因為比較大的問題是你把所有的 property, func variable 都寫成了 public。盡量不公開，或是只公開你不得已、非得要公開的東西，才是比較好的做法，因為那會讓其他人對你的 package 的依賴、耦合性降到最低。

---

最後，這個主題是因為有一位講者寫了一片文章，[The Top 10 Most Common Mistakes I’ve Seen in Go Projects](https://itnext.io/the-top-10-most-common-mistakes-ive-seen-in-go-projects-4b79d4f6cd65) 因為這一篇文章的反應不錯，所以寫了一本書 100 Go Mistakes: How to Avoid Them ，有興趣的人可以去試閱看看。

### Reference

- [https://changelog.com/news/how-to-make-mistakes-in-go-9Oqw](https://changelog.com/news/how-to-make-mistakes-in-go-9Oqw)
- [https://medium.com/@oboturov/golang-time-after-is-not-garbage-collected-4cbc94740082](https://medium.com/@oboturov/golang-time-after-is-not-garbage-collected-4cbc94740082)
- [https://www.manning.com/books/100-go-mistakes-how-to-avoid-them](https://www.manning.com/books/100-go-mistakes-how-to-avoid-them)