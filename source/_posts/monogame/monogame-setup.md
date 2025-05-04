---
title: 安裝並發佈一個 MonoGame 空白專案
date: 2025-01-24 14:51:46
updated: 2025-01-24 14:51:46
categories: MonoGame
tags:
---

本文主要用來記錄如何安裝 MonoGame 並發佈一個空白專案。

<!-- more -->
### 安裝 Visual Studio 2022
1. 只開發 Windows 版本的話工作負載選擇「.NET 桌面開發」即可。
2. 如果要使用 DirectX 版本的話需要另外安裝 [DirectX End-User Runtimes (June 2010)](https://www.microsoft.com/en-us/download/details.aspx?id=8109) 因目前還不支援 DirectX 12。
3. MonoGame 3.8.2 版本需要 .NET 8 SDK 以上的版本，如果已經安裝 Visual Studio 2022 的話就不必另外安裝。

### 安裝延伸模組
1. 啟動 Visual Studio 以後選擇「不使用程式碼繼續」。
2. 在上方功能列「延伸模組」→「管理延伸模組」，搜尋 MonoGame 找到「MonoGame Framework C# project templates」安裝後並重新啟動 Visual Studio‧

### 建立專案
1. 啟動 Visual Studio 選擇「建立新的專案」，找到「MonoGame Cross-Platform Desktop Application」這是使用 OpenGL 的版本，如果要用 DirectX 則選擇「MonoGame Windows Desktop Application」。
2. 專案建立好以後先執行一次建置，建置的過程中會幫我們安裝好 mgcb 跟 mgcb-editor，建置完成以後可以從檔案總管中雙擊「Content.mgcb」打開 mgcb-editor。
3. 如果 mgcb-editor 沒有成功開啟，可以手動在專案的根目錄下執行「dotnet tool restore」，相關設定在 .config 資料夾內的 dotnet-tools.json 內。

### 發佈專案
1. 右鍵選擇專案打開「屬性」，「建置」→「發佈」可以找到「發佈原生 AOT」，官方建議可以開啟。
2. 右鍵選擇專案打開「發佈」或從上方功能列「建置」→「發佈選取範圍」，目標選擇「資料夾」，特定目標選擇「資料夾」，資料夾位置不必修改，點選完成。
3. 可以更改「目標執行階段」為「win-x64」或其他目標，之後點擊「發佈」就可以完成整個流程了。

***
參考資料
- *[https://docs.monogame.net/articles/getting_started/index.html](https://docs.monogame.net/articles/getting_started/index.html)*