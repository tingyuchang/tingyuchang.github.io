---
 layout: post
 title: Docker for go-openbbsmiddleware
 date:   2021-09-05 16:00:00 +0800
 tags: [docker, pttneverdie]
---
這一篇跟 Go 本身沒有太大的關係，是在 [PttNeverDie](https://hackmd.io/@twbbs/guide) 的  opensource 專案中，關於使用 Docker 啟動開發環境的部分。

建議搭配[投影片](https://docs.google.com/presentation/d/1YzltLLyiq11xW9DJHsdOlsDDGNGw1oGwhrBBIHfQiE8/edit?usp=sharing)一起看，會比較知道在幹嘛。

### For non  backend developer

主要的 services 有：

- go-pttbbs: 使用 Go 改寫的 c-pttbbs ，能直接操作 BBS 的檔案系統，以及 share memory 的部分，也提供 Restful API 給 go-openbbsmiddleware。在 docker 環境下因為使用了 imageptt ，所以也可以用 telnet 或是 pttweb 來連線。
- go-openbbsmiddleware: 主要提供 Restful API 給 App, Web 等 Frontend services。
- postfix: mail system
- redis: cache
- mingo: DB，未來有考慮轉到 DB 不過目前是當作中介資料使用

![https://imgur.com/UOXxlEl](https://imgur.com/UOXxlEl.jpg
)

如果是非後端開發人員，可以直接使用 docker-compose 啟動，拿到完整的服務

```bash
./scripts/start.sh #init env
docker-compose --env-file docker_compose.env -f docker-compose.yaml up -d #start
docker-compose --env-file docker_compose.env -f docker-compose.yaml down -v #end
```

原本的開發環境建置需要修改的路徑、啟動 script 的步驟太繁瑣，為了簡化並且留下文件記錄，才開始進行 docker 一鍵啟動的構想，目前也還沒完全完成，不過已經可以讓非開發人員比較容易上手了。

### For backend developer

對後端開發人員來說，架構圖我認為可以修改如下：

![https://imgur.com/POtQ6gt](https://imgur.com/POtQ6gt.jpg)

兩個主要的專案：go-pttbbs, go-openbbsmiddleware 都改成本機啟動的方式

go-pttbbs 的部分比較簡單，只要設定好原本的檔案系統，修改啟動的 ini 參數，就可以順利跑起來了。

go-openbbsmiddleware 的部分比較麻煩，沒有 template files, DB connection, go-pttbbs 等服務的話，直接啟動會因為 panic 而中止，所以我們要先把這些服務建立起來，才能夠跑 go-openbbsmiddleware。

主要步驟如下：(BBSHOME 是你的當前目錄)

1. 先 clone demo repo 建置全新的 BBS 環境 `./scripts/docker_initbbs.sh {BBSHOME} pttofficialapps/go-pttbbs:latest`
2. 建立 passwd `./scripts/docker_initpasswd.sh {BBSHOME} pttofficialapps/go-pttbbs:latest {N_USER}`
3. 修改 go-pttbbs 的 ini file ，找一個 ini template 修改即可，要把 BBSHOME 改成當前的目錄
4. build go-pttbbs, and run with ini file 
5. 修改 docker_compose.env 裏面的 BBSHOME ，改成當前目錄
6. `docker-compose --env-file docker_compose.env -f docker-compose-dev.yaml up -d`，啟動 dev 版本的 docker-compose ，這樣子 DB 就會啟動了
7. 進入 go-openbbsmiddleware ，修改 types/00-config.go 裏面的 template 路徑（這邊未來應該會改寫成吃 ini file 的方式）
8. build go-openbbsmiddleware, and run with ini file (可以直接使用 development.ini)

啟動之後可以試看看 port 3457 → go-openbbsmiddleware, 3456 → go-pttbbs

### Reference

- [Demo Repository](https://github.com/tingyuchang/demo-bbs-docker)
- [Slide](https://docs.google.com/presentation/d/1YzltLLyiq11xW9DJHsdOlsDDGNGw1oGwhrBBIHfQiE8/edit?usp=sharing)