---
title: ELK Stack
date: 2019-01-09 10:53:20
tags:
categories:
---

# **ELK 介紹**

ELK是由 ElasticSearch、Logstash、Kibana 三個 open-source 軟體組成。運作流程為，Logstash 從 Server 端收集了 Log 並將之處理後，將其推送至 ElasticSearch 進行儲存，再透過 Kibana 視覺化界面呈現出來，可在其介面上搜尋並分析 Log。

<!-- more -->

**# ElasticSearch**

分布式搜尋引擎，可用於日誌的檢索

**# Logstash**

數據收集處理引擎，可用於收集日誌或是資訊，並根據設定的格式，轉換成該資料欄位

**# Kibana**

是一款可為 ElasticSearch 提供圖表分析與搜尋的 Web 介面


# **Beats 介紹**

Beats 是 open-source 的數據發送者，可安裝在 Server 端上，將各種類型的數據發送到 ElasticSearch ，Beats 可以直接發送數據到 ElasticSearch 或通過 Logstash 發送到 ElasticSearch ，為何這種使用方式是因為 Logstash 若安裝在 Server 端上，會使其 CPU 及記憶體消耗過多造成效能下降，而 Beats 所佔系統的效能幾乎可以忽略不計。目前 Beats 包含四種：

**# Packetbeat**

搜集網路流量數據

**# Topbeat**

搜集系統、進程和文件系統級別的 CPU 和記憶體使用情況等數據

**# Filebeat**

搜集文件數據

**# Winlogbeat**

搜集 Windows 事件日誌數據


# **ELK 架構**

若系統架構不大，可使用 Filebeat + Logstash 並行的架構

![](https://i.imgur.com/wHUNcDx.png)

此架構讓 Filebeat 單純用來接收 Log ，讓其他較複雜的交給 Logstash 來處理
\
\
\
若系統架構龐大， Log 不能有所遺失，那上述的架構便不符需求了， Logstash 的效能並不好，忙碌時容易造成 Log 遺失，即使 Filebeat 有重送的功能，若 Server 端剛好執行登錄檔輪替( logrotate ) ， Log 還沒被送到 Logstash 就被壓縮起來了。為避免此狀況，我們可以在中介加入 Redis 。

![](https://i.imgur.com/07qelkC.png)

此架構先把 Log 都先送到 Redis ，等 Logstash 有空就可以從 Redis 將資料抓出來歸檔，當然中介不一定要使用 Redis ， kafka 或是 RabbitMQ 也行，這邊選擇 Redis 是因為  Redis 的數據全部 in memory ，所以速度較快。


# **ELK 安裝建置**

**# 機器規格**

| 項目 | 規格 |
| ---- | ---- |
| CPU | 12 Core |
| RAM | 24G |
| System Disk | 50G HDD |
| Data Disk | 100G HDD |
| OS | Ubuntu 16.04 LTS |


**# 更新並安裝 Docker**
```linux
sudo apt-get update
sudo apt-get install -y docker.io
```

**# 建立 Redis**
```linux
docker run -d --name elk-redis -p 6379:6379 -v /data/redis:/data --restart=always redis:3.2.4 redis-server  --maxmemory 4gb --requirepass QFkXXBZkLD6MgcEL1y8l
```

| 指令 | 說明 |
| ---- | ---- |
| -d | 在背景執行 |
| --name | Container 的名稱 |
| -p | 本機與 Container 對應的 port |
| -v | 本機與 Container 對應的資料路徑 |
| --restart=always | 機器重啟後 Container 自動重啟（預設是關閉） |
| redis:3.2.4 | Redis Container 的映像檔名稱與版本 |
| redis-server | 建立 redis 的 server（另有 redis-cli 的客端版本） |
| --maxmemory 4gb | 設定 Redis Container 佔用記憶體的最大容量 |
| --requirepass | 客端 VM 將 Log 資料傳送到 Redis 時驗證用的 key |
\
**# 建立 Elasticsearch**
```linux
docker run -d --name elk-elasticsearch -p 9200:9200 -v /data/elasticsearch:/usr/share/elasticsearch/data --restart=always -e ES_JAVA_OPTS:-Xmx6g -e ES_JAVA_OPTS:-Xms6g elasticsearch:5.3.1
```

| 指令 | 說明 |
| ---- | ---- |
| -d | 在背景執行 |
| --name | Container 的名稱 |
| -p | 本機與 Container 對應的 port |
| -v | 本機與 Container 對應的資料路徑 |
| --restart=always | 機器重啟後 Container 自動重啟（預設是關閉） |
| -e ES_JAVA_OPTS:-Xms | 指定Elasticsearch 占用記憶體的初始值 |
| -e ES_JAVA_OPTS:-Xmx | 指定Elasticsearch 占用記憶體的最大值 |
| elasticsearch:5.3.1 | Elasticsearch 的映像檔名稱與版本 |
\
**# 建立 Logstash**

建立 Logstash 資料夾
```linux
mkdir logstash
```

進入 logstash 資料夾，建立 conf.d 資料夾
```linux
cd logstash 
mkdir conf.d
```

建立 Dockerfile
```linux
vim Dockerfile

FROM  logstash:5.3.1
COPY conf.d /etc/logstash/conf.d
CMD ["-f", "/etc/logstash/conf.d"]
```

進入 conf.d 資料夾，建立 logstash.conf
```linux
vim logstash.conf
```
```linux
input {
    redis {
        host => "redis"
        port => 6379
        data_type => "list"
        key => "08log"
        password => "QFkXXBZkLD6MgcEL1y8l"
    }
}
output {
    elasticsearch {
        hosts => ["elasticsearch"]
        index => "%{[@metadata][env]}_%{[type]}-%{+YYYY.MM.dd}"
    }
}

```

建立 weblog.conf
```linux
vim weblog.conf
```
```linux
filter {
    if [type] == "weblog" {
        grok {
            match => ["message", "%{TIMESTAMP_ISO8601:[@metadata][timestamp]} %{NUMBER:threadid} %{LOGLEVEL:loglevel} %{NOTSPACE:logger} %{GREEDYDATA:message}"]
            overwrite => [ "message" ]
        }

        date {
            match => [ "[@metadata][timestamp]", "YYYY-MM-dd HH:mm:ss.SSS" ]
            timezone => "UTC"
        }

        mutate {
            convert => {
                "threadid" => "integer"
            }
            add_field => {
                "hostname" => "%{[beat][hostname]}"
                "webtype" => "%{[fields][webtype]]}"
                "[@metadata][env]" => "%{[fields][env]]}"
            }
            remove_field => ["beat", "fields"]
        }
    }
}

```

回到 logstash 資料夾，建立鏡像
```linux
docker build -t elk-logstash .
```
![](https://i.imgur.com/DSGV9WD.png)

開始安裝 Logstash
```linux
docker run -d --name elk-logstash -p 5000:5000 --link elk-redis:redis --link elk-elasticsearch:elasticsearch --restart=always elk-logstash:latest
```

| 指令 | 說明 |
| ---- | ---- |
| -d | 在背景執行 |
| --name | Container 的名稱 |
| -p | 本機與 Container 對應的 port |
| --link | 其它的 Container 做連結 |
| --restart=always | 機器重啟後 Container 自動重啟（預設是關閉） |
| -e ES_JAVA_OPTS:-Xms | 指定Elasticsearch 占用記憶體的初始值 |
| -e ES_JAVA_OPTS:-Xmx | 指定Elasticsearch 占用記憶體的最大值 |
| elk-logstash:latest | 自建 logstash 映像檔名稱與版本 |
\
**# 建立 Kibana**
```linux
docker run -d --name elk-kibana -p 80:5601 --link elk-elasticsearch:elasticsearch --restart=always kibana:5.3.1
```

| 指令 | 說明 |
| ---- | ---- |
| -d | 在背景執行 |
| --name | Container 的名稱 |
| -p | 本機與 Container 對應的 port |
| --link | 其它的 Container 做連結 |
| --restart=always | 機器重啟後 Container 自動重啟（預設是關閉） |
| -e ES_JAVA_OPTS:-Xms | 指定Elasticsearch 占用記憶體的初始值 |
| -e ES_JAVA_OPTS:-Xmx | 指定Elasticsearch 占用記憶體的最大值 |
| kibana:5.3.1 | kibana 的映像檔名稱與版本 |
\
**# 建立 Grafana**
```linux
docker run -d -p 3000:3000 --restart=always -v /data/grafana:/var/lib/docker/grafana --name=grafana grafana/grafana
```

檢查以上 Container 是否建立成功
```linux
docker ps -a
```
![](https://i.imgur.com/pBdsPm8.png)


瀏覽器輸入機器 IP 即可進入初始頁面

![](https://i.imgur.com/gmratx8.png)


**# 建立 Filebeat**

至此下載 Filebeat 5.3.1\
https://www.elastic.co/downloads/past-releases/filebeat-5-3-1

設定 filebeat.yaml

![](https://i.imgur.com/vHaebKO.png)

| 項目 | 說明 |
| ---- | ---- |
| env | Kibana Fields 的名稱 |

![](https://i.imgur.com/CTrSsas.png)

| 項目 | 說明 |
| ---- | ---- |
| hosts | ELK 機器 IP |
| key | logstash.conf 內 input 底下 key 值 |
| password | logstash.conf 內 input 底下的 password 值 |

其餘 Web 、 Server 的 Log 格式請依需求設定

將 filebeat-5.3.1-linux-x86_64 放在 D:\Service\ 

以系統管理員開啟 Windows Power Shell，輸入以下指令執行安裝 filebeat 服務
```linux
cd D:
cd Service\filebeat-5.3.1-windows-x86_64
.\install-service-filebeat
```
檢查 filebeat Service 是否成功執行

**# Kibana 介面設定**

至 Dev Tools 輸入
```linux
GET _cat/indices?v
```
![](https://i.imgur.com/htWkCuk.png)

可看到 index 的值為 weblog ，將前面 weblog- 的複製，貼上到 Management → Index Patterns → Index 
![](https://i.imgur.com/ASDxsnZ.png)

點擊 Discover 就可以看到匯入的 web Log 資料
![](https://i.imgur.com/bNIpwAx.png)


**# 刪除功能**

若因匯入格式不正確的 Log 而出現 Elasticsearch 在初始化 index 導致 Kibana 無法正常使用時，請輸入下列指令刪除目前 Kibana 內目前的 Index
```linux
curl -XDELETE http://localhost:9200/*
```