---
title: COSCUP Attendance
description: 一堆小小問題，湊在一起變成一個幾乎無法處理的災難的故事
date: 2024-08-04T11:29:14+08:00
tags:
categories:
draft: true
---

# 緣起

2023 年 COSCUP 時，製播組有個統計各個議程人數的需求，當時是用 Google 表單，但是因為要手動輸入很多資訊，不太方便。於是會後我閒閒沒事幹就自己開發了一個網站，可以篩選哪一天的哪個議程廳，然後就會篩選出相關議程，就可以數入人數。  
2024 年時資訊組有意整合這個系統到官網的教室狀態，且點人數的工作被轉交到場務組。

# 程式架構

在開始介紹問題之前，先來介紹一下大致上的程式架構。這個網站採用前後端分離，前端是 NextJS 的 SSG 模式，後端是 Golang 的 Gin。網頁打開驗證完 token 後，會先透過 GET `/session.json` 和 `/api/attendance` 兩個 API 獲取必要資訊，根據參數（日期、樓層......）篩選後呈現出議程資訊和人數。當輸入數字後，會透過 POST `/api/attendance` 把議程 id 和人數更新回伺服器，伺服器處理完後就會透過 websocket 對所有人發送更新。

# 洞洞們

## 把執行檔和資料放在同一個 Docker volume

首先是一個跟程式比較沒關係的問題，但確實造成了不小的困擾。我在部屬的時候把 data.db 和 main 都放在同一個資料夾下，然後再一起掛載在 docker volume 裡。這看起來沒什麼問題，但是當更新 docker image 後，因為 volume 裡的 main 會覆蓋掉新版的 main，就導致 image 雖然更新了但是實際上跑的還是舊版程式。  
這個問題不太容易發現，因為 docker image hash 怎麼看都是新版，但是程式就是看起來沒更新，我甚至一度懷疑是 synology（我的程式放在 NAS 上跑）的容器管理器的問題。最後意外發現 `docker exec coscup-attendance /app/main --version` 是舊版才真相白。  
這個問題最麻煩的是會讓程式在本地測試和上正式機時的表現不一致（因為根本不同版本 ==），還以為是鬼打牆了。

## database error

這個問題還沒確定是哪裡出現的，不過肯定是因為兩個 goroutine 同時存取 sqlite 導致出現 `SQLITE_BUSY`。這還需要更多研究，理論上應該無腦枷鎖或是弄個 queue 就可以解決，但是我覺得 sqlute 的 package 不可能連麼基本的問題都沒解決，應該是我的程式哪裡有問題。
另外，這個問題還會和下面的問題連動產生令一個惱人的困擾。

## 如果遠端資源不會變動，就要用 useSWRImmutable

這個算是一個可有可無的問題，原本用 useSWR 是因為想要他去處理 GET `/session.json` suspense 的問題，但是因為 useSWR 會定期重新抓資料更新，所以就會產生很多重複的 request。這個問題其實也不大，最多就是拖慢一點網頁效能，但是因為 GET `/session.json` 其實是從 database 抓一串自串然後回傳，這個地方有機會造成 SQLITE_BUSY，然後可能是伺服器崩潰的造成 websocket 斷線。websocket 一斷線前端會把 input 鎖住，所以打一打就會突然不能用。這個讓使用者體驗變得非常差，

## 不要自製 useWS

大概是去年我覺得別人的輪子太肥，自己造一個輕一點的輪子，結果輕到亂飛。  
這個部份的問題是 websocket 要很久很久才會顯示 connected，我判斷連線的方式是 `isConnected = socket?.readyState === WebSocket.OPEN`。前幾天試圖解決的時候有注意到 `socket.readyState` 其實已經是 `WebSocket.OPEN` 了，但是不知道為什麼還是顯示 disconnected。後來研究發現是我的 useWS 沒寫完善，只要有一個新的訊息進來，react 就會更新 `isConnected` 了。

不要重複造輪子  
不要重複造輪子  
不要重複造輪子

## 即時共編

這是一個在多個 client 之間同步狀態的問題，我現在的解決狀態是假設不會有兩個人同時改同一個值，加上 websocket 就會「看起來」像是即時的。其實這不太算是問題，不果下次如果要在做的話可能會拿掉這個功能，因為為了他引入 websocket 實在是讓前後端都變得很麻煩。  
即時共編還有個問題，就是當 websocket 送來新的更新時要更新狀態，這讓那些 `<input />` 的狀態管理變得有一點麻煩，而且有點影響 UX

# 反思

這些問題每一個其實都不大，最難處理的應該是 database 的部份，但是當我在前一天晚上十一點才發現問題時，每一個要解決都花很久的時間，光是第一個就卡了半小時。  
最後，前端好難，如果可以，我搞後端就好了。
