---
 layout: post
 title: About Database
 date:   2021-10-01 23:00:00 +0800
 tags: [database]
---
database 的知識廣泛又深入，理解真正的問題是什麼才能找到正確的方式來應對

1. 讀取很慢？
2. 寫入很慢？
3. 連線數太多造成 query or insert 很慢？

### 讀取資料的問題

第一步應該是先檢視 query 本身是不是有效率的，Query 的欄位是否有建立正確的 index？index 的數量是否過大或是有不必要的 index 佔用了大量的 RAM ？SELECT 的欄位數量是否過度浪費 (SELECT * 是個不好的做法)？data schema 是否不適合造成 Query 的效度不佳等。可以透過改善 query 方式、建立適當的 index、改良 data schema、增加 DB 的硬體效能、善用 cache 減少 query 的數量等方式來優化。

但是如果上述的方式都做過了，問題可能就回到比較單純的部分，比如說要 query 的資料量太大了，裡面有好幾個 billion 的 data 或是資料量一般，反而是遇到高流量問題之類的。

有一個很經典的問題是這樣的：  
Q: how to query billions data fast?   
A: not to query billions data.

query billions data 本來就會慢，一般常用的 database engine 都是使用 B-tree (or B+tree)，index query 的 big-O 是 logn，是一種很有效率的資料結構，但是當資料量大到一個量級的時候，還是會感覺到變慢，因此這時候建議的解決方式就是 Partition。

### Partition

partition 是在同一個 database 裏面把資料切割成不同的 table，切割的方法分成兩種 Horizontal and Vertical。

![https://i.imgur.com/JmsqZNU.png](https://i.imgur.com/JmsqZNU.png)

我們常用的是 Horizontal，可以有效的把巨量資料的 table 做切分，Vertical 主要用在某一個欄位的資料量非常龐大時，把過度龐大的欄位從主要的 table 分割。

假設我們的 table 有 1 billion 的資料，我們可以使用適合的 partition type 做分割，舉例來說如果是使用者的交易紀錄，那麼也許可以根據交易的時間做 partition （不過應該比較常用 pk 做 partition），ex transations_2019, transations_2020, transations_2021 之類的方式，如此一來 index 的數量會下降，可以大幅度地縮短 query 的時間，也符合 `not to query billions data` 的精神 (而且還有不少的優點)。

使用 partition 也有一些限制，比如說:

1. 更新 partition key 造成必須移動資料到另一個 partition table 的時候會比較複雜
2. query 的方式沒有考慮到 partition 的規則，還是執行 full tables  scan 
3. 改變 data schema 要考慮的比較多

### Replication

有時候我們遇到 query 過慢的問題不是因為數量太大，而是連線數量太多，DB server 無法負擔。這邊有個前提是不要讓一個 request 就能夠直接開啟 db connection ，應該是要讓 connection pool 來管理會比較安全，不過 connection pool  的數量也是有極限的，主要是取決於 DB Server 的硬體規格，所以其實可以用錢解決這個問題，我覺得也沒有不好，某一些商業模式的利潤非常的高，對於他們來說，用錢解決是一種又快又有效率的做法。但如果今天要用技術的手段來處理，就需要搞清楚需求的環境以及每一種方法的優缺點。

如果遇到的情境是讀取的需求大於寫入，以及可以允許一點點的寫入延遲（資料不同步，比較專業的說法是 eventual consistency），那麼 Replication 就是一個很好的做法。Relication 會有多台的 DB server ，由其中的一台當作 master ，其他台作為 slave (or backup)，這邊不會提到 multi-master 的 replication ，要處理很麻煩的衝突問題實在是很麻煩，就先當作沒這個東西吧。

![https://i.imgur.com/nwmw4cY.png](https://i.imgur.com/nwmw4cY.png)

replication 讓 master 負責寫入資料、處理同步，slave 只負責讀取，因此在讀取的需求大於寫入的情境下，就可可以複製多台的 slave 來分擔大量讀取的連線需求，是一種很有效率的方法。

使用 replication 的限制：

1. eventual consistency
2. 寫入很慢（要讓全部的 server 都同步需要一些時間，雖然有 async 但還是會比一台的速度慢）
3. implement replication 的步驟比較複雜

### Log Structured-Merge tree

提到最後一個方法 Sharding 之前，還有一個情況可以考慮：database 的選擇。

常見的 database 有 MySQL, MariaDB, Postgres, MongoDB, SQLite, MSSQL 等等，可以參考[這邊](https://db-engines.com/en/ranking)。 可以發現大多數的主流 DB 都是 B-tree 的資料結構。B-tree 的類型特性讀取快速，寫入比較慢，也是因為過去的需求讀取是大於寫入的，不過隨著海量數據的時代演進，對於大量、快速資料寫入的需求增加， Google 在 BigTable 上開始使用了 LSM tree 的資料結構，其特性剛好跟 B-tree 相反，寫入非常快 (O(1))，但是讀取比較慢(O(n))，所以仔細思考對於 DB 的需求，理解 DB 的特性，選擇正確的 database engine ，搞不好就能有效的解決問題了。

LSM tree DB 有：Cassandra, LevelDB, RocksDB, influxDB 等

### Sharding

最後當我們必須要面對高流量寫入的場景時，不管是 partition, replication 都無法處理真正的高流量寫入，雖然說還是有其他的方法可以解決，但是 DBMS 的觀點來看，還有 Sharding 這個方法可以使用，不過會擺在最後的關係是因為 Sharding 的缺點很明顯，因此在使用上的限制跟需要注意的地方很多，一定要非常熟悉才建議使用 Sharding 作為 production 的 solution。

![https://i.imgur.com/uqCjh9E.png](https://i.imgur.com/uqCjh9E.png)

Sharding 依賴 consistent hash 演算法來決定資料要存在哪一台的 DB Server，很輕易的做到了 DB server 的 scaling，在大量資料寫入下，可以有多台 server 同時負擔工作。

不過這樣的做法其實有幾個問題：

1. consistent hash 保存了 hash key 跟 db server 對應的邏輯，這部分的實作一旦有問題就會造成嚴重的後果。
2. 無法保證 ACID，所以 transaction, rollback 基本上都是無法使用的
3. 最重要的是，consistent hash 的實作必須要知道 query 的內容，才有辦法設計出 sharding 的 model 

ps. Sharding 的部分其實沒有這麼簡單，不過要解釋 Sharding 的話，篇幅又會變太長，所以有興趣的話，還是請自行 Google Sharding 的工作原理。  

### 小結

我覺得 database 並沒有所謂的 GOAT，每一個解決方案都有其對應的場景跟缺點，有些問題還沒出現的，也不用急著去實作處理，應該要先認清楚目前的情境，選擇最適合的解決方案。database 的知識非常的廣，且不深入研究的話，並不知道怎麼做出正確的選擇，這一篇文章想做個總整理，幫助我把這些知識做好順序以及判斷的依據，有些地方寫錯或是解釋有問題的，如果有發現的話，非常希望可以跟我說，讓我可以糾正，感謝！
