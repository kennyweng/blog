---
title: Prometheus (Docker)
date: 2018-03-26 15:25:46
tags: Prometheus
---

# **環境準備**

Ubuntu 16.04 LTS

Docker 1.12+

<!-- more -->

# **前置安裝**

**# Check Version**
```linux
sudo apt-get install -y docker.io
```

**# Check Docker Version**
```linux
sudo docker version
```

# **安裝 mysqld_exporter**

**# 啟動mysqld_exporter容器**
```linux
docker run --name mysqld_exporter -d -p 9104:9104 --restart=always \
  -e DATA_SOURCE_NAME="exporter:exporter@(localhost:3306)/mysql" \
  prom/mysqld-exporter
```

**# 查看容器日誌，確定容器正常運行**
```linux
docker logs -f mysqld_exporter
```

**# 確認exporter有導出數據**

瀏覽器訪問http://localhost:9104

![](https://i.imgur.com/4B8EoJJ.png)


# **安裝prometheus**

**# 創建資料夾**
```linux
mkdir -p /home/docker/prometheus
```

**# 創建Data資料夾**
```linux
mkdir -p /home/docker/prometheus/prometheus-data
```

**# 編輯yml文件**
```linux
vim /home/docker/prometheus/prometheus.yml
```

**# 配置文檔如下**
```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  - job_name: 'mysql'
    scrape_interval: 15s
    static_configs:
      - targets:
        - 'localhost:9104'
        labels:
          instance: db1

```

**# 啟動prometheus容器**
```linux
docker run --name prometheus -d -p 9090:9090 --restart=always \
  -v /home/docker/prometheus/prometheus-data:/prometheus-data \
  -v /home/docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

**# 確認Targets**

瀏覽器訪問http://localhost:9090/targets

![](https://i.imgur.com/2sW4Eiv.png)


# **安裝Grafana**

**# 創建資料夾**
```linux
mkdir -p /home/docker/grafana
```

**# 創建Data資料夾**
```linux
mkdir -p /home/docker/grafana/data
```

**# 啟動grafana容器**
```linux
docker run --name grafana -d -p 3000:3000 --restart=always \
  -e "GF_SECURITY_ADMIN_PASSWORD=admin" \
  -v /home/docker/grafana/data:/var/lib/grafana \
  grafana/grafana
```

**# 登入grafana介面**

瀏覽器訪問http://localhost:3000

帳號admin

密碼admin

**# Add deta source**

![](https://i.imgur.com/tEIjOpA.png)


**# 推薦Dashboards**

3622

2

358

159

![](https://i.imgur.com/z3f3IlX.png)