---
 layout: post
 title: L4 vs L7 load balancing
 date:   2021-09-27 14:00:00 +0800
 tags: [nginx, load balancing]
---
OIS model 有七層，L4 是傳輸層（Transport Layer），L7 則是應用層（Application Layer），常見的 Load Balancing 機制在這兩層有不同的工作方式。

這邊不會提到太多網路通訊相關的部分，但是至少要知道 L4 把資料切成一個一個的 packet 封裝了兩端的 ports，這一層使用的通訊協定是 TCP/UDP。L7 則是我們常見的 HTTP, telnet, FTP 等。

以 HTTP 來說 L4 這一層無法知道一個 request 的 url path, header, body 等資訊，因此如果要對 `/location` or    `/*.jpg`  or header 裏面的資訊做邏輯處理，L4 這一層是辦不到的，必須要到 L7 才有辦法。另外 TLS 的部分理論上來說是歸類在 L6 （Presentation Layer） 不過實際上大家都是在 L7 做，所以有加密的話 L4 看到的也只是加密後的內容，無法進行解密。

### 工作原理

*L4 Load Balancing*
![Imgur](https://i.imgur.com/h5IhRmN.png)

假設我們要發一個 HTTP GET request 到一台 L4  的 load balancer。可以看到 L4 不會去處理封包的內容，也不會跟 client 端建立 TCP 連線，單純的把封包作轉發，到後端真正處理的機器上，由被轉發的機器跟 client 端建立連線。

*L7 Load Balancing*
![Imgur](https://i.imgur.com/MUwM1U0.png)

同樣假設我們要發一個 HTTP GET request 到一台 L7  的 load balancer。跟 L4 不同的地方在於 L7 的 Load Balancer 會去解析封包，跟 client 進行 TCP 連線，再根據設定的邏輯（L7 比 L4 能處理的邏輯更多）跟後端要處理的機器建立 TCP 連線，這時候的封包跟原本 client 傳來的是不一樣的，同理當 L7 Load Balancer 收到 response 的封包時，也會先解析後才會回傳給 client ，不能直接轉傳的是因為兩者是不同的 TCP 連線。

### 小結

L7 能做的事情比 L4 多很多，在硬體上，能處理 L7 的 switch ，幾乎也一定能處理 L4 。這兩者的價格也是天差地遠。不過因為我也不是硬體的專業，所以要討論的並不是 swich，而是 Nginx/traefik/HAProxy 等軟體的 solution，這些 opensorce 也都能處理 L4/L7 的 load balancing ，但如果不明白兩者工作方式上的差異就無法真正理解在設定 conf 的時候應該調用哪些參數，舉例來說，為什麼在 stream {} 裏面不能使用 location 呢？

```
# nginx.conf
stream {
    upstream backend {
        server 172.17.0.4:1001;
        server 172.17.0.5:1002;
        server 172.17.0.6:1003;
        server 172.17.0.7:1004;
    }

    server {
        listen 80;
        proxy_pass backend;
				# location / {
        #     proxy_pass http://backend;
        # }
 
    }
}
```

L7 load balancer 看起來很強大，但是一定很吃效能，如果是簡單的情況下，是不是使用 L4 會比較有效呢？

此外還有 TLS 在 L4/L7 的情況下要怎麼處理？

諸如此類的問題，有時還有衍伸更多的問題需要討論，如果只是 c&p 範例的設定檔來修改，也許在簡單的問題上能夠 work ，但遇到更複雜的問題時，我覺得就必須要回到基本的網路/系統/演算法/資料結構上去理解問題的本質，才能夠真的找到問題的最佳解答。

原本是因為聽了這一集的 [podcast](https://changelog.com/shipit/19) 想去研究一下 traefik ，不過我連 nginx 可能都沒這麼了解，哪要怎麼知道 traefik vs nginx 的差異呢？一旦開始研究下去，又一堆問題跑出來，nginx 最有名的應用是 reverse proxy ，這一塊又跟網路通訊的知識息息相關，所以今天決定先從基本的 OSI Model 開始說起，將來再把這一串研究的結果寫下來做紀錄。