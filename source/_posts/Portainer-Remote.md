---
title: Portainer-Remote
date: 2018-07-24 11:16:55
tags: Portainer
categories: DevOps
---

# **Portainer 安裝**

**# 安裝Docker**
```linux
sudo apt-get install -y docker.io
```

**# 下載Image**
```linux
docker pull portainer/portainer
```

**# 創建volume**
```linux
docker volume create portainer_data
```

**# 安裝Portainer**
```linux
docker run -d -p 9000:9000 --restart=always --name portainer -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer
```
此處 9000:9000 可自行修改，第一個為Host Port，第二個為Container Port

**# UI設定**
```linux
http://localhost:9000
```

**# 設定密碼**
![](https://i.imgur.com/LPcVhDG.png)

**# 選擇Remote**
![](https://i.imgur.com/2iYwvdk.png)

輸入機器名稱與機器IP，此處 Port:2375 需打開

開啟方式

CentOS 7：
```linux
vim /usr/lib/systemd/system/docker.service

[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock


systemctl daemon-reload

systemctl restart docker
```

Ubuntu 16.04：
```linux
vim /etc/default/docker

DOCKER_OPTS="-H tcp://0.0.0.0:2375"

service docker restart
```

查詢是否成功開啟可用此指令
```linux
netstat -tulp
```