---
 layout: post
 title: WebRTC: Know NAT first
 date:   2021-10-22 13:00:00 +0800
 tags: [WebRTC, NAT, STUN, TURN, ICE, SDP, P2P]
---
WebRTC (Web Real-Time Communication) 是一種 Peet-to-Peer 的傳輸技術，P2P 具有低延遲的好處，通常用來作為 VOIP, Streaming, IM等通訊功能的核心技術之一。

WebRTC 有點複雜，想要理解 WebRTC 是怎麼運作的，就一定要先知道什麼是 NAT (Network Address Translation) ，以及 NAT 對於 P2P 的影響。 

P2P 的傳輸是讓兩台裝置直接進行通訊連線 (TCP, UDP ...) 這會比透過一台中心的 Server  要來得快很多（如下圖）。

![https://i.imgur.com/GXuaaZE.jpg](https://i.imgur.com/GXuaaZE.jpg)

![https://i.imgur.com/Tr3lPbX.jpg](https://i.imgur.com/Tr3lPbX.jpg)

但是兩台裝置要怎麼進行連線呢？從 OSI Model 來看，TCP or UDP 等第四層的傳輸協議會基於第三層的 IP Address 來運作，也就是說我們一定要知道對方的 IP 位置才能夠進行溝通跟連線。不過在今日的網路架構下，使用者端的裝置都無法擁有屬於自己的 Public IP ，只有 Private IP (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) ，Private IP 只能使用在內部子網路內，外部無法跟他直接進行溝通。試想一下，如果有一台外部網路的裝置要跟你建立連線，他跟你說他的 IP 位置是 10.0.0.2 ，如果真的對這個位置發送連線請求，一定不會是外部網路的裝置，而是內網中另外一台剛好 IP 也是 10.0.0.2 的裝置。

其實就連我們要連到外部網路也是一樣，你對於一個網頁伺服器 (Web Server) 發出 Http GET req

這個封包也必須包含了你自己的 [IP:Port] ，否則 Web Server 不知道要把 response 送給誰，可是你的 ip 是 Private IP 的話，那麼一樣的道理，Web Server 也會把封包送給自己內網的裝置，而不是外部的請求者。要怎麼解決這個問題呢？就是要透過 NAT 。

## NAT (Network Address Translation)

NAT 是一項解決內外部 IP 位置映射的技術，Router 會使用 NAT 把內網對外部發送的封包進行修改，把原本是 Private [IP:Port] 改成這一個網路的 Public IP 

![https://i.imgur.com/EvCo6Ft.jpg](https://i.imgur.com/EvCo6Ft.jpg)

接收 request 的 Web server 打開封包看，來源地的 [IP:Port] 則是請求方的 Public IP 跟 Router 給的 random port，Web server 的 response 就能傳送回這個 Public [IP:Port] ，當 Router 收到這個封包的時候，會根據 routing table 找出原本的發送者是誰，再把封包的 destination 修改成 Private IP 跟 原本的 port。

但是這跟 P2P 連線有什麼關係呢？原因在於 NAT 雖然能夠提供內部的裝置網路的服務，但畢竟這些裝置並不是真正擁有屬於自己的 Public IP，因此需要 IP 位置進行初始化動作的 TCP, UDP 連線或是協定，沒有辦法利用 Public IP 找到真正要連線的裝置，因此無法建立連線。

試著想看看今天你是一台外部的裝置，打算要對另一個外部網路進行 P2P 連線，這時候你除了要知道對方的 Public IP 之外，還需要知道哪一個 Port 才能對應到想要連線的裝置，即使知道了 IP 跟 Port ，如果對方的 NAT 不支援外部連線，那麼就一點辦法都沒有了。至於 NAT 不支援外部連線的意思是什麼呢？這才是對於 WebRTC 最重要的地方，如果 NAT 是支持 P2P 的類型，我們就可以利用 STUN 來幫助建立連線，但如果不行，就必須要改用 TURN 來連線，關於 STUN 跟 TURN 的部分之後再説，這邊先關注在 NAT 的類型到底是什麼意思。

NAT 有四種：Full-cone NAT, Address restricted NAT, Port restricted NAT 及 Symmetric NAT 

### Full-cone NAT

這一種 NAT 對於內部的裝置對外會給予固定一個對應的 Port ，外部的連線只要從這個一個 Port 進來，Router 就知道要轉給哪一台裝置，所以也稱作 one-to-one NAT。這種 NAT 可以允許沒有紀錄的外部 [IP:Port] 進行傳輸，因此可以滿足 P2P 建立連線的流程。

### Address restricted NAT

這個類型的 NAT 跟 full-cone 類似，但唯一的差別是，他只接受從內部裝置曾經傳輸過的 IP 位置發送過來的封包，舉例來說裝置 B 要傳送一個封包要給裝置 A ，但是在 NAT 的紀錄中裝置 A 沒有發送給裝置 B 的紀錄，因此這一個封包就不會被 Router 處理。可以想見這種類型的 NAT 就無法允許外部主動進行 P2P 連線。

### Port restricted NAT

跟 Address restricted NAT 類似，但是限制上除了 IP 之外，連 Port 也要固定，不允許外部裝置使用沒有傳輸過的 Port 進行連線。Address restricted NAT 跟 Port restricted NAT 某種程度上要進行連線沒有這麼容易，但是仍有一些技術可以解決他們的限制，最後一種 NAT 才是最麻煩的。

### Symmetric NAT

Symmetric NAT 讓內部裝置對應的外部裝置連線都有不同 Port ，舉例來說內部裝置 A 對外部裝置 B 的 Port 與外部裝置 C 的 Port 就不一樣，而且也不允許外部裝置使用沒有傳輸過的 [IP:Port] 進行連線。Symmetric NAT 限制就是除非是由內部的裝置主動跟外部連線，否則外部是無法跟在 Symmetric NAT 內的機器進行連線的，當兩個要進行 P2P 連線的裝置都是 Symmetric NAT 的情況下，可想而知，是無法運作的。那麼如果一邊是 full-cone 一邊是 Symmetric 可以連線嗎？可以的，但是一定要從 Symmetric 這一邊發起連線請求才行。不過很不幸的，現在的 3G/4G 的行動網路都是 Symmetric NAT，所以在這種網路環境下是無法啟動 P2P 的。

### NAT 小結

我想大家可能都還有一個疑問，這樣子為什麼我們不全部都用 full-cone NAT 呢？Symmetric NAT 存在的意義是什麼？我查了一些資料後，得到的答案是：對於網路的使用者來說，Symmetric NAT 並沒有帶來任何好處（如果你覺得可以封鎖 P2P 是好處的話，那就不一樣了），但對於網路架構的提供、維護者（ISP）來說，使用 Symmetric NAT 在建置大型複雜的網路架構是比較容易維護，且不容易出問題，Symmetric NAT 不需要去維護複雜的 routing table ，當內部的裝置要對外發出請求時，只要找到當下可以使用的 Port 就好了，連線一關閉，Port 也釋放出來，對於 Public IP 有限的 ISP 業者來說，只要他們可以比較容易建立多層的子網路，提供更多的裝置網路服務。
以上就是關於 NAT 的說明，要理解 NAT 必須要對 OSI Model 有一定程度的理解，才知道 NAT 運作的原理以及其限制，當知道 NAT 是什麼之後，才能夠明白 WebRTC 要怎麼處理 NAT 帶來的限制，這部分就留給下一篇再來說明。

最後 NAT 的技術也許在未來 IPV6 普及之後就會漸漸被廢止，因為 NAT 原本就是因為 IPV4 的位置不足而發展出來的解決方案，理論上來看在 IPV6 下每一個人都可以被配到非常多的 Public IP，到時候就能直接用 Public IP 連線，而不會有 NAT 的限制存在。