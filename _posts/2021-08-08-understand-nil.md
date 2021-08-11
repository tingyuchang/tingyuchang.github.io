---
 layout: post
 title: Understand Nil
 date:   2021-08-08 12:00:00 +0800
 subtitle: nil is not nil
 tags: [golang]
 #categories: golang
---

### 前言

在 Cocup 2021 TW 的一場研討會中，看到了一個問題：

> 請問 slice struct call by value 為什麼判斷 slice == nil 不會報錯，平常的 struct 都要 &struct == nil 才不會報錯

那時候我也不知道怎麼回答這個問題，然後在研究 error 運用的時候，看到了一篇文章

[Why is my nil error value not equal to nil](https://golang.org/doc/faq#nil_error)

感覺我對於 nil 的理解是不足的，所以決定進行 nil 的研究來補強。

我們先來看一段簡單的程式碼:

```go
// MyError implements error interface 
type MyError struct {
	Str string
}

func (e *MyError) Error() string  {
	return fmt.Sprintf("This is my error: %v", e.Str)
}

// test this error
func testError(something bool) error {
	var err *models.MyError

	if  something {
		err = &models.MyError{
			Str: "something happened",
		}
	}

	return err
}

// Then we check the error 

err := testError(true)
if err != nil {
		panic(err)
}
```

看到上面這一段程式碼，應該會覺得很稀鬆平常，是正常寫 error check 的作法

err 如果不是 nil 就執行錯誤處理，err 是 nil 就繼續下執行後續的程式邏輯。

不過有趣的地方來了，要是沒有發生 something 的時候， testError 回傳的數值是 nil 嗎？在呼叫 testError 回傳後的 err ≠ nil 的判斷中，我們竟然會得到：

```go
fatal error: panic while printing panic value
```

err 並不是 nil ，這太奇怪了吧，所以我在 testError 中檢查了回傳值

```go
func testError(something bool) error {
	var err *models.MyError

	// something is false here
	if  something {
		err = &models.MyError{
			Str: "something happened",
		}
	}
	fmt.Println(err)
	// nil
 	return err
}
```

![https://i.imgur.com/Par5dMV.jpg](https://i.imgur.com/Par5dMV.jpg)

err *MyError 的確是 nil ，這是哪邊的問題呢？別急，更奇怪的事情在後面

我們再宣告一個新的 MyError 來確認看看

```go
err := testError(false)
fmt.Println(err) // nil
fmt.Println("err = nil?", err == nil) // err = nil? false
var err2 *models.MyError // 重新宣告一個 MyError
fmt.Println(err2) // nil
fmt.Println("err2 = nil?", err2 == nil) // err2 = nil? true
```

有看到最奇怪的地方了嗎？ err ≠ nil ，但是 err2 == nil ，真是一堆黑人問號????

在解釋這個情況之前，我們先來了解一下什麼是 zero value。

### Zero Value

[spec](https://golang.org/ref/spec#The_zero_value)

> When storage is allocated for a variable, either through a declaration or a call of new, or when a new value is created, either through a composite literal or a call of make, and no explicit initialization is provided, the variable or value is given a default value. Each element of such a variable or value is set to the zero value for its type: false for booleans, 0 for numeric types, "" for strings, and nil for pointers, functions, interfaces, slices, channels, and maps. This initialization is done recursively, so for instance each element of an array of structs will have its fields zeroed if no value is specified.

當我們使用 var or new or make 的時候，只要沒有給予數值，變數就會被給予一個 default value ，這個 default value 我們也稱之為 zero value ，不同類型的變數有不同的 zero value

- bools → false
- numbers → 0
- strings → ""
- pointers → nil, point to nothing
- functions → nil, not initialized
- interfaces → nil, not assigned value, not even a nil pointer
- slices → nil, have no underlying array
- channels → nil, not initialized
- maps → nil, not initialized
- array → 根據存放的變數類型，給與相對應的 zero value
- sturct → 根據宣告的變數類型，給與相對應的 zero value

bool, numbers(int, float ...), string 都有他們自己專屬的 zero value，array 跟 struct 則是會根據存放的變數類型去給予 zero value。接下來我們可以看到 pointers, functions, interfaces, slices, channels, maps 都是給予 nil ，但是仔細想想不會覺得很奇怪嗎？這些類型都不一樣，他們的 nil 是什麼？pointer 的 nil 跟 function 的 nil 是一樣的 nil 嗎？原來 nil 並不是 Golang 的保留自，而是在 builtin.go 中宣告的一個變數（代表你可以宣告自己的 nil 來取代，但別這麼做，這一定會毀滅世界）

```go
// nil is a predeclared identifier representing the zero value for a
// pointer, channel, func, interface, map, or slice type.
var nil Type // Type must be a pointer, channel, func, interface, map, or slice type

// Type is here for the purposes of documentation only. It is a stand-in
// for any Go type, but represents the same type for any given function
// invocation.
type Type int
```

原來 nil 是一個變數，所以當我們做 xxx == nil 的時候，其實是 Golang 會把我們要比較的變數類型，轉換成各自的 nil 定義後，才回傳比對的結果。

舉例來說，slice 的 nil 代表沒有指向 underlying array ：

```go
var s []int
fmt.Println(s) // []
fmt.Printf("Type: %T \tptr: %p\t len: %d\tcap: %d\t\n", s, &s, len(s), cap(s))
// Type: []int     ptr: 0xc0000a6018        len: 0 cap: 0
fmt.Println("s == nil =>", s == nil) // true
```

empty slice 並不是 nil，雖然 len, cap 都是 0 ,但是 empty slice 的 array ptr 是有指向一個 underlying array 的。

```go
s2 := []int{}
fmt.Println(s2) // []
fmt.Printf("Type: %T \tptr: %p\t len: %d\tcap: %d\t\n", s2, &s2, len(s2), cap(s2))
// Type: []int     ptr: 0xc00000c048        len: 0 cap: 0
fmt.Println("s == nil =>", s2 == nil) // false
```

我們再來看一個例子：pointer 

pointer 的 nil 代表其沒有指向任何一個數值，這個應該是最好理解的

```go
type T int 
var a *T
fmt.Println(a) // nil
fmt.Printf("Type: %T \tptr: %p\tvalue: %v\n", a, &a, a)
// Type: *main.T   ptr: 0xc0000b2018       value: <nil>
fmt.Println("a == nil =>", a == nil) // true 

var a2 = (*T)(unsafe.Pointer(uintptr(0x0)))
fmt.Println(a2) // nil
fmt.Printf("Type: %T \tptr: %p\tvalue: %v\n", a2, &a2, a2)
// Type: *main.T   ptr: 0xc000126028       value: <nil>
fmt.Println("a2 == nil =>", a2 == nil) // true
fmt.Println("a == a2 =>", a == a2) // true
```

maps, functions, channels 的 nil 都比較複雜，[Francesc Campoy - Understanding nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)  説先把他們的 nil 當作是一個 nil pointer 就好，這樣比較簡單。

最後則是最讓人困惑的 interface，interface 不是 pointer ，他有兩個元件 (type, value)

當 type, value 都是 nil 的時候，interface 才是 nil。

(nil, nil) ⇒ nil

```go
var s fmt.Stringer // fmt Stringer interface 
s == nil // true, becase s is not assigned type
```

(*int, nil) ⇒ not nil

```go
type Person int 

func (p *Person) String() string {} // implement Stringer

var p *Person
var s fmt.Stringer
s = p
p == nil // true, it's nil because p is pointer
s == nil // false (*Person, nil), not nil
```

那麼讓我們回到前面的 MyError

```go
func testError(something bool) error {
	var err *models.MyError
	// hide not use here ....
	return err
}
```

testError 的回傳是 error ，但是其實我們宣告的是 error(*models.MyError, nil)

從 error 這個 interface 的角度來看，他並不是 nil ，他有一個確定的 type 叫做 *models.MyError

因此 var err2 *models.MyError 是 nil ，但是 err error(*models.MyError, nil) 不是 nil 的原因就在這邊。

所以最好別把自己封裝的 error 當作 interface error 回傳，在使用上一定會讓人感到困惑。


### Reference

- [https://zhuanlan.zhihu.com/p/151140497](https://zhuanlan.zhihu.com/p/151140497)
- [https://www.youtube.com/watch?v=ynoY2xz-F8s](https://www.youtube.com/watch?v=ynoY2xz-F8s)
- [https://medium.com/@ishagirdhar/nil-in-golang-aaa16565a5be](https://medium.com/@ishagirdhar/nil-in-golang-aaa16565a5be)
- [https://golang.org/ref/spec](https://golang.org/ref/spec)
- [https://golang.org/doc/faq#nil_error](https://golang.org/doc/faq#nil_error)