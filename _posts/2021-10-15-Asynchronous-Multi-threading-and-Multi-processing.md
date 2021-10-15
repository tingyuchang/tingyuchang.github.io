---
 layout: post
 title: Asynchronous, Multi-threading and Multi-processing
 date:   2021-10-15 15:00:00 +0800
 tags: [Asynchronous, Multi-threading, Multi-processing]
---
asynchronous, multi-threading 跟 multi-processing 感覺上很像，但是實際上是不同的處理方式。

上次爬山回家車上跟庫伸聊天，我問他覺得後端開發那邊是有趣的，他說以前做過 asynchronous 的部分，他們公司產品結帳付款是交由第三方負責的，因此沒有可預期的處理時間，所以建立了 message queue 來處理後續的工作。那時，我的腦袋就突然浮現了這三個名詞，很久以前就想寫看看這三個的不同，看起來很像的三個名詞，但其實是在講不同的事情！

## Synchronous & Asynchronous

asynchronous 主要是在解決 synchronous 的問題。什麼是 synchronous ? 想像我們操作一個很古老的 web 後台報表系統，你按下某一年的 revenue summary report ，然後整個畫面就卡住了，注重 UX 的話，這個系統可能會給你一個圈圈在哪邊一直轉，讓你感覺系統正在做事，但是真的在做事的是誰呢？browser 嗎? 並不是， browser 只是負責處理 http requestr & response ，幫忙把 request 發出去之後就沒事了，之後等收到 response 之後再做處理。那麼是後台系統的 application layer 在忙嗎？可能是也可能不是，如果 DB 是 NoSQL 的，那可能正在把撈出來的資料做運算，所以要等資料運算完畢才能返回 http response，但是也可能是他也在等待，等待 Database query 的結果。synchronous 就是一直在等待，把事情交給別人之後，就開始等待，即使自己已經沒事了，畫面會卡在那邊不會動，process 停在那邊浪費資源，這就是 synchronous system 會遇到的情況。

asynchronous 讓 process 不再白白地等待工作的完成，把任務交付給下一個之後，asynchronous 的程式會讓 process 繼續執行下一行的程式碼，而不會被堵塞 (blocking)。asynchronous  的實踐方式不見得是 multi-thread ，不同的語言以及 framework 有不同的實現方式。Go 是 multi-threading 的做法，利用 goroutine & channel 來實現 asynchronous 。javascript 是 single thread，利用了 call-back 的 function 讓完成的工作執行這一個 function，達到 asynchronous 的目的。

也就是說 asynchronous 是一種 non-blocking 的工作流程，不需要等待某一項工作完成才能執行下一個工作，跟是不是 multi-thread 並沒有關係。

## Multi-Threading

在多核心時代開始的技術，同時間可以有多個 thread 在同一個 process 裏面執行工作。幫大家複習一下 OS 的課程：Process 是 thread 的容器，一個程式 (code) 被載入到記憶體開始執行後就是一個 process，理論上一個程式可以有多個重複的 processes 同時在執行，且他們之間是互相不干擾的，Process 裡面會有多個 thread ，但是一個 CPU 只能同時跑一個 thread ，所以有多個 CPU 就可以同時跑多個 threads （不要問為什麼你的電腦 CPU 只有兩個核心，但是可以同時跑很多個程式，RTFM）。

因為在同一個 process 內，所以資源是共享的，理論上會比 single-thread 的系統運算更快速，但是有難以管理的問題存在，以及共用資源、變數以及記憶體造成 race-condition or deadlock 的災難。以前在寫 [How to make mistakes in go](https://tingyuchang.github.io/2021-08-14-How-to-make-mistakes-in-go-part1/) 的時候有提到過這一點，使用 multi-thread 不見得會比 single-thread 好，主要在於管理的成本過高，像是 race-condition 的處理，會用到 mutex, semaphore 等 lock 的機制，它們都是非常昂貴的，以及 multi-thread 的程式碼會比較複雜等缺點。

有一些高效能又穩定的系統是採用 single-thread 的，像是 Redis, NodeJS ，而 NGINX, HAProxy 都是 multi-thread。

btw database 使用 multi-thread 跟 single-thread 的差別我還沒有研究過，只是從理論上來看 database 的瓶頸應該不是運算的能力，而是 I/O 才對。

## Multi-Processing

最常拿來跟 Multi-Threading 比較的，就是 Multi-Processing，Multi-Processing 是指利用多個 processes 同時處理工作，可以是同一台機器，也可以是不同台機器，跟 multi-thread 不同的地方在於 multi-process 是各自獨立的 process，彼此的資源不互相干擾，因此不會有 race-condition or deadlock 的問題，如果 Process 之間要溝通，那麼就要走 IPC (Inter-Process Communication) 。舉例來說，假設我們今天要從一堆資料中找到年紀大於 60 的人，Multi-Processing 的做法是開幾個 worker process（ 假設兩個），把名字開頭 A-M 的 users 給 processA 去找，N-Z 的 users  給 processB 去找，最後再把結果合併即可。Multi-Processing 是一種很常見的設計，特別是在分散式系統中，可以很輕易地透過 scaling 來增加處理的能力。

imo，multi-threading 我覺得是在相同資源下（假設多核心不是額外的成本）增加程式的效率（performance），multi-processing 則是增加了程式的可處理能力（capacity）跟部分的效率，不過最終的結果還是取決於程式跟系統是怎麼設計的。

## 小結

asynchronous 是指系統的 process 不會被 block ， 最常使用的情境是處理一個很費時的運算或是不可預測時間的外部請求時，搭配 message queue 來處理之後的工作，可以讓使用者不會被卡在處理時的等待，也可以減少系統資源的消耗。

multi-threading 是在一個 process 中有多個 thread 同時在工作，但老實說 multi-threading 是很不容易的一件事，在大多數的情況下使用 single-thread 就可以把問題處理好，並不一定要用到 multi-threading 不可。

multi-processing 則是有多個同樣的 process 同時工作，彼此獨立不互相影響。