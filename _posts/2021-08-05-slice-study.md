---
 layout: post
 title: Slice Study
 date:   2021-08-05 16:30:25 +0800
 # categories: golang
 subtitle: powerful, and convenient interface to sequences of data.
 # cover-img: /assets/img/path.jpg
 # thumbnail-img: /assets/img/thumb.png
 # share-img: /assets/img/path.jpg
 # gh-repo: daattali/beautiful-jekyll
 # gh-badge: [star, fork, follow]
 tags: [golang]
---

在 Golang 中，有 Array and Slice 這兩種 type。
Array 類似其他語言的定義，但是在 Golang 中 Array type 必須定義長度以及元素的類型

```go
var a [4]int
a[0] = 1
i := a[0] // i == 1
```

Array 的長度是固定的，因此，不同長度的 Array 是不一樣的類型

```go
var a [4]int
var b [5]int
reflect.TypeOf(a) == reflect.TypeOf(b) // false
```

Array 的定義其實不是這麼好使用，像是在呼叫一個 method 的時候，必須傳入符合其定義的 type ，否則 complier 是不會讓它通過的

```go
func sum(arr [10]int) int {}

sum([]int{1,2,3}) 
// cannot use []int{...} (type []int) as type [10]int in argument to sum
```

從以上的範例可以想像，如果我們要計算一個 array 的總和，但限制一定要傳入 10 個元素的陣列，這在使用上是非常不方便的。

因此 Golang 提供了另一個在使用上比較方便的類型：**Slice**

Slice 是一個 struct ，可以看到在 source code 中的定義：

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

array unsafe.Pointer 是一個指向 underlying array 內元素的指標，len 是 slice 的長度，cap 是從目前 underlying array index 到 array 底的長度，有點難以理解，所以用程式碼說明：

```go
a := [10]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
// underlying array is [1,2,3,4,5,6,7,8,9]
b := a[3:6] // [3,4,5] len: 3 cap: 7
c := a[3:] // [3,4,5,6,7,8,9] len: 7 cap: 7
```

b 跟 c 都指向同一個 underlying array，b = [3,4,5] c = [3,4,5,6,7,8,9]，len 分別是 3 跟 7 應該沒有問題，而兩者的 cap 都是 7 ，這是因為兩個 slice 都指向 underlying array a 的 index 3，而 a 的長度是 10，所以 10 - 3 = 7。

---

Slice 的說明先到這邊，因為我比較想談的是後面的部分，如果對 Slice 還有基本定義或是使用上的疑慮，可以看官方的[介紹](https://blog.golang.org/slices-intro)。

再來談談一些 slice 應用上的可能會有的疑惑

```go
a := [10]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
b := a[3:6] // [3,4,5] len: 3 cap: 7
c := a[3:] // [3,4,5,6,7,8,9] len: 7 cap: 7

b[3] = 20 // panic: runtime error: index out of range [3] with length 3
```

雖然 b 的 cap 是 7 ，但是 len 只有 3 因此如果直接 assign [3] 會造成 panic error。那麼我們要怎麼擴充 b 呢？有三個方法： append, copy and re-slice (if capacity is enough)

```go
// append
b = append(b, item)

//copy

b2 := make([]int, len(b)+1)
copy(b2. b)
b2[len(b)] = item

//re-slice, attention! not over the capacity of b
b = [:len(b)+1]
b[len(b)] = item
```

append 比較簡單也常用。copy 要注意到 underlying array 是不同的，在某些情況下，使用 copy 來 insert 會比 append 來得有效率 ([slice tricks](https://github.com/golang/go/wiki/SliceTricks) 中有提到，之後會再說明)。re-slice 也很直覺易懂，不過要注意到 capacity 是否足夠，否則是會產生 runtime error 的。

值得一提的是使用 append or re-slice 要注意到 b 的 underlying array 也會被修改這一點

```go
b = append(b, 20)

// b = [3,4,5,20]

// c = [3,4,5,20,7,8,9]

// a = [0,1,2,3,4,5,20.7,8,9]
```

在 underlying array 有足夠的 capacity 下，會將新的元素放進去，因此 underlying array 中的元素就會被置換，連帶影響到其他指向這個 array 的 slices，想是 c[3] 就變成了 20。

但如果 underlying array 沒有足夠的 capacity 呢？ 請大家再看一段程式碼：

```go
c = append(c, 20)
// [3,4,5,6,7,8,9,20]
b = b[:cap(b)]
// [3,4,5,6,7,8,9]
```

為什麼 b 跟 c 的行為不一樣了呢？這是因為 c 的 append 發現 c 的 capacity 不夠了，因此觸發了

growslice 這一個 func ，重新產生了一組新的 underlying array 給他，這個時候 b, c 兩者的 underlying array 已經不同了。

其中運作的原理，可以看看 [growslice](https://github.com/golang/go/blob/4bb0847b088eb3eb6122a18a87e1ca7756281dcc/src/runtime/slice.go#L162) 這一段 source code 

---

```go
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
```

當 slice 的 cap 不足需要擴大的時候，就會呼叫 growslice 這一個 func 處理，會先根據 old. cap (old.cap) 以及 exp.cap 來計算出 new.cap。處理的邏輯如下: (old.cap ⇒ 舊的 exp.cap ⇒ 期望的 new.cap ⇒ 最後計算出來新的)

1. 如果 exp.cap 大於兩倍的 old.cap，new.cap = exp.cap
2. 如果 exp.cap 小於兩倍的 old.cap，而且 old.cap < 1024，那麼 new.cap 就等於兩倍的 old.cap
3. 如果 exp.cap 小於兩倍的 old.cap，但是 old.cap ≥ 1024，那麼就會跑一個 for loop 讓 new.cap = 1.25 old.cap ，直到 new.cap ≥ exp.cap
4. 最後，如果 old.cap ≤ 0, new.cap = exp.cap

計算完新的 cap 之後，就會把目前 array 內的元素複製到新的 array 中

接下來談談 slice 中傳遞資料的問題

## Call by Value

```go
func main() {
	a := [10]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}

	b := a[3:6]
	c := a[3:]
	double(b)
	fmt.Println(b) // [6 8 10]
	fmt.Println(c) // [6 8 10 6 7 8 9]
}
func double(x []int) {
	for i, v := range x {
		x[i] = v * 2
	}
}
```

在 Golang 的世界裡面只要沒有用到指標，就都是 call by value ，slice 也不例外，但是為什麼上面的程式碼 double 卻會影響到 c 呢？這是因為在傳遞 b 給 double 的時候，的確是複製了一份 b 的值，但是 b 的 slice (struct) 只有 ptr, len 以及 cap ，並沒有真正的持有元素，而複製出來的 slice 也指向了同樣的 underlying array ，所以在 double 裡面修改了元素，就會影響到 c。 

要利用這個特性時，必須要注意到改變 slice 的長度時 (append, re-slice or copy) 都會讓 slice 的 underlying array 變成新的，因此可能就會發生在新的 slice 中修改，但是其他地方的 slice 因為兩者的 underlying array 不一樣了，造成修改是無效的，要怎麼有效的處理這個情況發生呢？可以參考以下的思考：

1. 避免改變 slice ，當 slice 被當作參數傳遞之後，不嘗試做任何 len or cap 的修改，不讓 realloc 的情況發生，也許有人會認為只要熟悉 slice 的原理，小心操作就不會有這個問題產生，但是你要怎麼保證其他人不會踩到這個陷阱呢？請思考以下的程式碼：

    ```go
    // 這個例子是要證明操作 append 的先後順序，其實對外面的 xs 影響極大
    // 先進行 append 的話，造成 alloc new underlying array，之後再對 xs2 進行的操作
    // 就跟外面的 xs 無關了。反過來，如果先進行操作，再進行 append ，那麼
    //  xs2[0] = 1
    //  xs2[1] = 2
    // 這兩個都會影響到原本的 xs ，因為這時候 xs & xs2 都還是同一個 underlying array
    func main() {
    	xs := make([]int, 1, 3)
    	func(xs2 []int) {
    	    xs2 = xs2[1:3]
    	    // xs2 = append(xs2, 3)
    		xs2[0] = 1
    	    xs2[1] = 2
    	    xs2 = append(xs2, 3)
    		fmt.Printf("Inside: %v Addr of slice: %p\n", xs2, &xs2)		
    	}(xs)
    	xs = xs[:cap(xs)]
    	fmt.Printf("Outside: %v Addr of slice: %p\n", xs, &xs)
    }
    ```

    看過了上面的例子就知道，這樣的錯誤要 debug 是極為困難的。

    函式的撰寫者無法限制 capacity ，因此即使他知道可能會有 realloc slice 的情況產生了，也無法阻止。

2. 會需要改變 slice 長度的操作，不要依賴既有的 slice，建立一個新的 slice ，以及使用新的 slice 當作返回結果，感覺對於記憶體的使用上比較沒有效率，但是比起誤用造成的問題，應該是兩權相害取其輕，否則就需要更嚴格的規範使用的情境。

    ```go
    func Insert(x []interface{}, index int, items ...interface{}) []interface{} {
    	return append(x[:index], append(items, x[index:]...)...)
    }

    func InsertByCopy(x []interface{}, index int, item interface{}) []interface{} {
    	s := append(x, 0)
    	copy(s[index+1:], s[index:])
    	s[index] = item
    	return s
    }
    ```

3. 如果是要處理 Slice of Structs  的時候，可以嘗試使用 Slice of Pointers ，也就是只傳遞指標，而不是數據本身，如此一來即使 underlying array 的元素被複製了，也還是指向相同的數據，不過這個作法有好也有壞，稍後的章節會深入討論。

## Slice of pointers vs Slice of structs

先來複習一下幾個 slice 使用上的情境

**Example 1**

```go
// xs 被當作參數傳遞進去，裡面的 xs2 跟外面的 xs 其實是不同的
// 不過因為 underlying array 一樣，所以 xs2[0] = 1 
// 也讓外面的 xs 受到影響
func main() {
	xs := make([]int, 1, 3)

	func(xs2 []int) {
		xs2[0] = 1
		fmt.Printf("Inside: %v Addr of slice: %p\n", xs2, &xs2)
		// Inside: [1] Addr of slice: 0xc0000b6030
	}(xs)

	fmt.Printf("Outside: %v Addr of slice: %p\n", xs, &xs)
	// Outside: [1] Addr of slice: 0xc0000b6018
}
```

**Example 2**

```go
// 這個例子就可以很明顯的知道，當 append 啟動的時候，
// 因為增加三個元素後，已經超過原本 slice 的 capacity
// 所以 xs2 會呼叫 growslice 建立新的 underlying array ，並把當前的元素複製過去
// 這個時候 xs 與 xs2 已經是參考到不同的 underlying array
func main() {
	xs := make([]int, 1, 3)

	func(xs2 []int) {
	    xs2 = append(xs2, 1,2,3)
	    xs2[0] = 99
			fmt.Printf("Inside: %v Addr of slice: %p\n", xs2, &xs2)
			// Inside: [99 1 2 3] Addr of slice: 0xc00010c018
	}(xs)

	fmt.Printf("Outside: %v Addr of slice: %p\n", xs, &xs)
	// Outside: [0] Addr of slice: 0xc00010c000

}
```

example 1 說明了只要 underlying array 是一樣的，即使 slice 是不一樣的，也不會有什麼問題

example2 則是改變了 underlying array ，因此讓兩個 slices 彼此的行為脫鉤，不再互相影響

以下來看看 struct 在 slice 中要注意的地方

```go
type Object struct {
	Value int
}

func main() {
    xo := []Object{
            Object{0}, Object{1}, Object{2}, Object{3}, 
        }
    obj1 := xo[0]
    obj1.Value = 21
    fmt.Println(xo) // [{0} {1} {2} {3}]

		fmt.Printf("Inside: %v Addr of slice: %p\n", obj1, &obj1)
    fmt.Printf("Inside: %v Addr of slice: %p\n", xo[0], &xo[0])
}
```

這個問題很簡單，因為 Golang 是 call by value ，所以 obj1 := xo[0] 是複製 xo[0] 的資料給 obj1，所以兩個的 ptr 也不同，改成以下的寫法即可

```go
// 使用指標
obj2 := &xo[1]
obj2.Value = 22
fmt.Println(xo) // [{0} {22} {2} {3}]

// 用 index 也可以
xo[2].Value = 23
fmt.Println(xo) // [{0} {22} {23} {3}]
```

在 slice 中使用 struct 要特別注意的地方大概是這樣。接著我們回到這一個章節的主題：**Slice of Pointers**

當 slice 改變時候，裡面的 struct 也會被複製一份到新的 underlying array ，所以既使用了 &xo[0] 的指標也沒用，因為新的 slice 裡面的 struct 跟舊的是不同的

```go
xo := []Object{
		Object{0}, Object{1}, Object{2}, Object{3},
}

obj1 := &xo[0]
xo = append(xo, Object{4})
obj1.Value = 21

fmt.Println(xo) // [{0} {1} {2} {3} {4}]
fmt.Printf("Inside: %v Addr of slice: %p\n", obj1, &obj1)
// Inside: &{21} Addr of slice: 0xc000102018
fmt.Printf("Inside: %v Addr of slice: %p\n", xo[0], &xo[0])
// Inside: {0} Addr of slice: 0xc000116000
```

要怎麼解決這個問題呢？答案就是利用 Slice of Pointers，指標被複製了也沒關係，它們還是指向同一個 struct。

```go
xop := []*Object{
		&Object{0}, &Object{1}, &Object{2}, &Object{3},
}

obj2 := xop[0]
xop = append(xop, &Object{4})
obj2.Value = 21

fmt.Printf("Inside: %v Addr of slice: %p\n", obj2, obj2)
// Inside: &{21} Addr of slice: 0xc000018050
fmt.Printf("Inside: %v Addr of slice: %p\n", xop[0], xop[0])
// Inside: &{21} Addr of slice: 0xc000018050
```

[Kubernetes](https://github.com/kubernetes/apimachinery/blob/a644435e2c133d990b624858d9c01985d7f59adf/pkg/runtime/conversion.go#L56) 的 API 也運用了同樣的操作 ，不是直接拿 in []string ，而是指標，避免 in []string 被修改，而沒有反應在複製的參數中。

```go
func Convert_Slice_string_To_string(in *[]string, out *string, s conversion.Scope) error {
	if len(*in) == 0 {
		*out = ""
		return nil
	}
	*out = (*in)[0]
	return nil
}
```

最後要注意使用在 struct slice 中，使用 Slice of Pointers 會比 Slice of Structs 的效率以及空間的使用上都會比較多一些，原因也很簡單，Slice of Pointers 比 Slice of Structs 還多了指標的建立以及儲存，所以如果你對於 Slice 內資料的一致性沒有這麼大的需求，其實改用 Slice of Struct ，以及要存取的時候，利用 index 來操作 struct 就好了，沒有必要一定要建立整個 Slice of Pointers。

恩... 可以看到其實差別還蠻大的

```go
type Object struct {
	Value int
}

func BenchmarkWithPointer(b *testing.B) {
	b.ReportAllocs()

	for i := 0; i < b.N; i++ {
		xo := make([]*Object, 100)
		for i:=0; i<100; i++ {
			xo = append(xo, &Object{i})
		}
	}
}

func BenchmarkWithoutPointer(b *testing.B) {
	b.ReportAllocs()

	for i := 0; i < b.N; i++ {
		xo := make([]Object, 100)
		for i:=0; i<100; i++ {
			xo = append(xo, Object{i})
		}
	}
}

// BenchmarkWithPointer-8      507992	   2068 ns/op	  2592 B/op	 101 allocs/op
// BenchmarkWithoutPointer-8   3619908  370.8 ns/op	  1792 B/op	   1 allocs/op
```

BTW: 這裡提到的使用 Slice of Struct 比較好跟 [Struct Method 使用 Pointer 比較好](https://golang.org/doc/faq#methods_on_values_or_pointers)是不同的概念

### Insert: Append vs Copy

上面有提到在 slice 中 insert 元素，有的時候 copy 會比 append 來得有效率，這其實是出自 [SliceTricks](https://github.com/golang/go/wiki/SliceTricks) 中的一段解釋:

```go
a = append(a[:i], append(b, a[i:]...)...)

// The above one-line way copies a[i:] twice and
// allocates at least once.
// The following verbose way only copies elements
// in a[i:] once and allocates at most once.
// But, as of Go toolchain 1.16, due to lacking of
// optimizations to avoid elements clearing in the
// "make" call, the verbose way is not always faster.
//
// Future compiler optimizations might implement
// both in the most efficient ways.
//
// Assume element type is int.
func Insert(s []int, k int, vs ...int) []int {
	if n := len(s) + len(vs); n <= cap(s) {
		s2 := s[:n]
		copy(s2[k+len(vs):], s[k:])
		copy(s2[k:], vs)
		return s2
	}
	s2 := make([]int, len(s) + len(vs))
	copy(s2, s[:k])
	copy(s2[k:], vs)
	copy(s2[k+len(vs):], s[k:])
	return s2
}

a = Insert(a, i, b...)
```

append 觸發了兩次的 realloc ，而 copy  只有在 cap 不足，重新宣告 s2 的時候做了一次，如果 capacity 是夠了，那麼連一次的 realloc 都沒做

```go
BenchmarkInsertByAppend-8   	 3905568	       295.0 ns/op	     672 B/op	       2 allocs/op
BenchmarkInsertByCopy-8     	 7681284	       156.8 ns/op	     352 B/op	       1 allocs/op
```

### 再來一些範例

```go
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // Conversion from string to path.
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
		// /usr/bin
}
```

```go
type path []byte

func (p path) ToUpper() {
    for i, b := range p {
        if 'a' <= b && b <= 'z' {
            p[i] = b + 'A' - 'a'
        }
    }
}

func main() {
    pathName := path("/usr/bin/tso")
    pathName.ToUpper()
    fmt.Printf("%s\n", pathName)
		// /USR/BIN/TSO
}
```

上面兩個 method 的目的不同 TruncateAtFinalSlash 是要改變 path 的長度，找到最後一個 '/' 後，讓 path 縮短成為 /usr/bin，而 ToUpper 則是每一個字都改成大寫。

TruncateAtFinalSlash 使用的是指標，在呼叫的時候，雖然是 copy 指標的數值，不過指標本身仍然會指向同一個 slice struct ，進而去改變這個 slice 的 metadata。

ToUpper 也同樣複製了 slice ，但是因為指向的是同一個 underlying array ，所以透過 index of slice 去修改 underlying array 的元素，達成原本的目的。

btw 其實 ToUpper 也可以用 *path 來執行，不過要寫成 *p 

```go
func (p *path) ToUpper() {
    for i, b := range *p {
        if 'a' <= b && b <= 'z' {
            (*p)[i] = b + 'A' - 'a' //  string 也是 byte 
        }
    }
}
```

### 關於 Slice 的 nil

empty slice 不見得是 nil ，雖然 len = 0 ，但是 cap 以及 ptr 都可能指向一個不為空的 underlying array， nil slice 是指沒有指向任何一個 underlying array 的 slice 。

### 關於 String

String 是一個 read only 的 byte slice 。

---

## 結論

1. 使用 slice 的時候，必須充分理解跟 underlying array 之間的關係，slice 不持有元素，只持有指標
2. Append, copy 都可能會改變 slice 的 capacity 導致 realloc ，re-slice 只能改變 length of slice 
3. 一旦發生了 slice realloc ，就會把現有的 underlying array 的資料複製到新的 underlying array 
4. 在 slice 中使用 pointer  ([]*Object)是個好方法，但是要注意在 struct of slice 上其效能的消耗
5. 直接傳遞 slice 的指標 (*[]Object) 也是不錯，可以參考 k8s 的做法
6. Copy 處理得好，會比 Append 來得有效率 


### Reference

- [https://blog.golang.org/slices-intro](https://blog.golang.org/slices-intro)
- [https://blog.golang.org/slices](https://blog.golang.org/slices)
- [https://github.com/golang/go/wiki/SliceTricks](https://github.com/golang/go/wiki/SliceTricks)
- [https://medium.com/swlh/golang-tips-why-pointers-to-slices-are-useful-and-how-ignoring-them-can-lead-to-tricky-bugs-cac90f72e77b](https://medium.com/swlh/golang-tips-why-pointers-to-slices-are-useful-and-how-ignoring-them-can-lead-to-tricky-bugs-cac90f72e77b)
- [https://medium.com/@opto_ej/there-are-other-nuances-one-should-consider-c798f12be15c](https://medium.com/@opto_ej/there-are-other-nuances-one-should-consider-c798f12be15c)
- [https://philpearl.github.io/post/bad_go_slice_of_pointers/](https://philpearl.github.io/post/bad_go_slice_of_pointers/)