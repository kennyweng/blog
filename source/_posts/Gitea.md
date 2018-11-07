---
title: Gitea
date: 2018-07-13 14:09:17
tags: Git
---

# **Gitea 介紹**

Gitea 是一個可自行託管的 Git 服務。

<!-- more -->

# **環境要求**

Ubuntu 16.04 LTS

Docker 1.10+

# **安裝 Gitea**

**# 下載映像檔**
```linux
docker pull gitea/gitea:latest
```

**# 建立資料目錄**
```linux
sudo mkdir -p /var/lib/gitea
```

**# 啟動容器**
```linux
docker run -d --name=gitea -p 10022:22 -p 10080:3000 -v /var/lib/gitea:/data gitea/gitea:latest
```
此處10080可改成80


**# 訪問介面**

http://hostname:10080

初次訪問需初始化設定

![](https://i.imgur.com/29ef1Ct.png)

伺服器和其他服務設定

![](https://i.imgur.com/tbThDZS.png)

管理員帳號設定

![](https://i.imgur.com/ZwaY0ze.png)

**# LDAP設定**

登入後，網站管理

![](https://i.imgur.com/nERdmXX.png)

認證來源>新增認證來源

![](https://i.imgur.com/LI5Qtio.png)

即可使用AD帳密登入