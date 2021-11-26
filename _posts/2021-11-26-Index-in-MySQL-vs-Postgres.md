---
 layout: post
 title: Index in MySQL vs Postgres
 date:   2021-11-26 14:00:00 +0800
 tags: [postgresql, innodb, mysql, database]
---

## 什麼是 Index

index 把資料做特定的排序，幫助我們用最短的路徑找到我們想要的資料，index 分成兩種：

1. Clustered Index: 儲存了所有資料的 index
2. Non-Clustered Index: 只有部分資料，剩下的資料用指標代替的 index （又稱作 secondary index）

## PostgreSQL

PostgreSQL 只支援 non-clustered index ，在 PostgreSQL 中完整的資料儲存在 Heap，Index 則是透過 Item pointer （tuple） 來取得完整的資料。好處是在獲取小部分資料的情況下，透過 non-clustered index 來找到目標的資料是非常有效率的，比起 Sequential scan 來得好，但是要注意到的是在 Range Query 的時候，就不見得如此，主要是因為 PostgreSQL 是利用 HEAP 的方式在儲存資料，HEAP 是無序的，代表 Index 接近的的兩筆資料，在儲存位置上不見得也在附近，所以使用 index 來取得資料是一種 Random I/O，而 Range Query 大量的 Random I/O 非常昂貴，會造成儲存體的工作效率低下，所以這部分 PostgreSQL 會自己判斷要哪一種方式來進行，在某些情況下，即使有下 Index ，也還是有可能會採用 Sequence Scan 來代替 Index Scan。

下面的範例就是對 index student_id 下 query ，在單筆資料跟多筆資料的情況下的結果：

![https://i.imgur.com/mmt2lBk.jpg](https://i.imgur.com/mmt2lBk.jpg)

## InnoDB (MySQL)

Index 運作機制上跟 PostgreSQL 差不多，主要的差別在於 Clustered Index。

InnoDB 的 Clustered Index 不像其他的 index ，他強制讓 DB 儲存在實體位置的資料，必須根據某一個欄位當作 index 來儲存。這個 column 可以使用 primary key 或是選擇一個 unique & NOT Null 的欄位，或是系統自動產生。這種機制可以避免掉 Random I/O 造成的效率低下。特別是當使用 clustered index 在存取 Range data 的時候，會非常有效率，因為資料的實際儲存己經是根據 clustered index 排序了，這也是為什麼 InnoDB 的 primary key 選擇非常重要的原因。

在 InnoDB，比較精簡的 index 稱作 Secondary index ，跟 PostgresSQL 的 non-clustered index 類似 (幾乎一樣)，但是 secondary index 不是指向實際儲存的位置。而是 primary key，所以在 secondary index 要拿到真正的資料。就必須要再透過 primary key 去找，這也是跟 PostgreSQL 最大的差異。兩者相比，InnoDB 在 index 的機制下單筆資料的取出效率會比較差一點。

## 小結

PostgreSQL 跟 InnoDB(MySQL) 是目前最受歡迎的兩大 RDBMS ，今天討論的部分比較細，沒有要比較兩者的優劣，而是希望能從兩者設計理念上的差異，去了解怎麼 index 工作的原理，幫助我們正確的使用 Index 。其實還有一個部分跟 Index 息息相關，就是 Insert ，不過再講下去感覺又會寫成長篇大論，所以還是先到此告一個段落。

## Reference
[https://severalnines.com/database-blog/postgresql-index-vs-innodb-index-understanding-differences](https://severalnines.com/database-blog/postgresql-index-vs-innodb-index-understanding-differences)
