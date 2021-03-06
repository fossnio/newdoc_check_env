# 透過 AD 的群組原則處理北市學校端的新公文系統

## 說明

本專案部署後，群組內的使用者登入後會依序自動處理下列各項

- 檢查 Hicos，版本小於設定版號 (checkHicos.ps1內設定) 就自動安裝最新版
- 檢查 Java 例外網站設定，沒有就補
- 檢查 Chrome 例外網站設定，沒有就補
- 檢查是否安裝 Java 所需憑證，沒有就從設置的的共享檔案抓來裝
- 檢查 Java 是否設置 cache 沒有就設
- 15 秒後啟動新公文元件 (Kdapp.jnlp)
- 檢查桌面有無 新公文元件 的捷徑，沒有就做一個

## 已測試環境

* Windows 10 (1803) 64 位元 企業版
* Windows 7 SP2 32位元 專業版

## 準備

主批次檔 check_newdoc_env.bat 未處理的部分有兩個，請另行處理或事先安裝於部署的映象檔

* 安裝 [Chrome](https://www.google.com.tw/chrome/) (可參考[官方部署指南](https://support.google.com/chrome/a/answer/3115278?hl=zh-Hant))
* 安裝 [Java JRE 8](https://java.com) (如需自動安裝 JAVA 部分請另外下載 JRE 並搭配 check_java.bat 使用 可設定最低版本與安裝檔位置進行自動升級與安裝)

另外需要先準備 trusted.jssecerts、kdapp.jnlp 這兩個檔，請去 [公文系統提供的檔案區](https://sites.google.com/view/newdoc/school/%E6%AA%94%E6%A1%88%E4%B8%8B%E8%BC%89%E8%88%87%E7%92%B0%E5%A2%83%E8%A8%AD%E5%AE%9A) 下載學校用的任一版本，只需取用前述兩個檔案即可。

## 使用 GPO 管理 Chrome

下載 Chrome GPO 樣板 https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip

檔案內有 adm / admx 兩種方式，要使用哪一種方式都可以

* adm: 從 GPO 系統管理範本內新增，新增之後只有這個 GPO 有， 透過 [RSAT](https://support.microsoft.com/zh-tw/help/2693643/remote-server-administration-tools-rsat-for-windows-operating-systems) 管理不須再次安裝 adm 樣板。
* admx: 要自己複製到主機 (你操作 GPO 這台) 的 %systemroot%\PolicyDefinitions , 對應語系資料夾要裝對應的語系樣板檔 adml， 所有的 GPO 都會有新選項，但是換一台操主機就沒了，除非在操作本機再安裝一次。

透過 adm 會安裝在 電腦(使用者)設定 / 原則 / 系統管理範本 / 傳統系統管理範本(ADM) / Google / Google Chrome

透過 admx 的就不會多一層 "傳統系統管理範本" 了

以 adm 為例是這樣安裝

![adm step 1](https://i.imgur.com/mOvAXqk.png)

![adm step 2](https://i.imgur.com/4whI85M.png)

針對行政群組容器設置的群組原則(依人)不能用電腦的系統管理範本，要用使用者的系統管理範本

以下我已使用者的範本為例

![Chrome GPO](https://i.imgur.com/GZzIMbM.png)

新公文系統在 Chrome GPO 上面需要做兩個設定: Flash 與 彈出視窗

都在使用者設定 / 原則 / 系統管理範本 / 傳統系統管理範本(ADM) / Google / Google Chrome / 內容設定 底下

```
啟用 允許在這些網站上執行Flash外掛程式 並新增下列網址
http://signweb.gov.taipei:80
```
```
啟用 允許這些網站的彈出式視窗 並新增下列網址
http://localhost:16888
http://localhost:61161
https://doc.gov.taipei
https://doctest.gov.taipei
https://edoc.gov.taipei
http://scan.taipei.gov.tw 
```

## Java 對外連線 GPO

這部分我自己是沒有設，我內部防火牆是全部關閉，如果你有開啟可能需要設置一下

但要注意一下這個設定只能針對 **電腦** 去做群組原則，無法以 **人** 去做群組原則，所以要另外設置可能不能跟上述對人的群組混用

電腦設定 / Windows 設定 / 安全性設定 / 具有進階安全性的 Windows Defender 防火牆 / 輸入連線

右鍵新增規則 / 程式 / C:\Program Files(x86)\java\jre1.8.0_181\bin\jp2launcher.exe / 允許連線 / 所有 / 給個名字

上面這個要依照版本去設定

## 設置腳本檔與放置共用空間

### 共用空間

找台 nas 或是共用空間放置公文系統的 trusted.jssecerts 、 kdapp.jnlp 和本專案的 checkHicos.ps1 三個檔案

目的是

* 批次檔在檢測環境時需要連線使用 checkHicos.ps1 檔
* 檢測環境如沒有 JAVA 憑證檔 trusted.jssecerts 就可以過來安裝
* 啟動公文時需要 kdapp.jnlp (本專案設置是連線啟用，使用者亦可複製一份回桌面使用)

### 設定 Script  檔內容

本專案主要兩個腳本檔: checkHicos.ps1 和 check_newdoc_env.bat 

checkHicos.ps1 為 powershell 腳本，目的為確認 Hicos 版本並更新

需要設置的部分為 Hicos 最低要求版本，目前設置為 3.0.3 (更改請依照 x.x.x 的三版號格式，缺一不可，記得有雙引號)

```
$min_ver = "3.0.3"
```

check_newdoc_env.bat 部分則要設定共用空間的檔案 UNC 路徑 

**特別注意: 不可是連線磁碟機代號 X:\，應該是 \\\server\path\file 這種 UNC 格式**

```
SET chkHicos=\\nas\share\newdoc\checkHicos.ps1
SET fileJavaCert=\\nas\share\newdoc\trusted.jssecerts
SET fileKdapp=\\nas\share\newdoc\kdapp.jnlp
```

## 設置 GPO 使用登入批次檔

最後就是設置 GPO 的登入時執行的批次檔 check_newdoc_env.bat

![Logon GPO](https://i.imgur.com/Tl1LBu9.png)

設置方式: 群組原則編輯 / 使用者設定 / Windows 設定 / 指令碼 -登入登出 / 登入

先按一下下面的 顯示檔案... 會跳出一個資料夾

把本專案的 check_newdoc_env.bat 丟進去，並選擇本檔案，無須參數

如果要做 JAVA 自動更新的話就把 check_java.bat 也掛進來

這樣就完成了

## 測試

立即更新群組原則 執行 gpupdate

登出 / 登入 應該就有了

## 特別注意

群組原則的設定可針對 組織 (OU) 做設定，像是教務處這樣的組織設定，

**但是請注意 要讓群組原則能夠生效 組織 (OU) 底下必須是使用者或是電腦**

所以如果你的組織像這樣

* 公文人員
  * 處室 A
    * 人員 P1
    * 人員 P2
  * 處室 B
    * 人員 P3
    * 人員 P4

把 公文人員 做群組原則連結的話是無效的

必須對 處室 A / 處室 B 這兩個都做群組原則連結才能生效

又如果組織是

- 行政人員
  - 群組 G1
  - 群組 G2
  - 群組 G3
- 非行政
  - 群組  G4

這種規劃也不能把群組原則放在 行政人員 上面，因為底下不是人員是群組

所以你必須用上面處室 A / B 這種方式去做才行