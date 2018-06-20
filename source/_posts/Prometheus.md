---
title: Prometheus
date: 2018-06-19 18:26:03
tags: Prometheus
---

# **Prometheus特點**

多維數據架構 (由 metric 名稱與 key/ value 定義的時間序列)

靈活的查詢語言(PromQL)

支援Local與Remote，不依賴分散式儲存

使用Pull方式取得目標資訊，通過HTTP協議傳輸

支援Pushgateway

支援多種圖形模式與Dashboard

<!-- more -->

# **Prometheus核心元件**

**Prometheus Server：**
主要服務，用來抓取和儲存時序資料 (Time Series Data)

**Client librarys：**
用於對接Prometheus Server，負責檢測應用程序代碼，讓其可以透過HTTP傳送資訊到Prometheus Server

**PushGateway：**
支援Client主動將短期和批量的監控數據推送至此，而Prometheus會定時來抓取數據

**Exporter：**
負責從目標收集數據，並轉換成Prometheus支援的格式，不同的Exporter負責不同的業務， 命名格式為xxx_exporter

**Alertmanager：**
用於告警通知管理

# **Prometheus架構**

![](https://i.imgur.com/bDYKBtw.png)


# **Prometheus VS InfluxDB**

**InfluxDB**：僅僅是一個資料庫，它被動的接受客戶端資料和查詢請求，基於 Push

**Prometheus**：完整的監控系統，能抓取資料、查詢資料、告警等功能，基於 Pull

**Push** 和 **Pull** 主要區別在發起者不同及邏輯架構不同


![](https://i.imgur.com/XSE5VdV.png)


# **PromQL**

在Node Exporter的/metrics接口中返回的監控數據，在Prometheus下稱為一個樣本。採集到的樣本由以下三部分組成：

**指標（metric）**

**時間戳記（timestamp）**

**樣本值（value）**

# **Metric Type**

**Counter（計數器）**

rate(http_requests_total[5m])

**Gauge（儀表板）**

node_memory_MemFree

predict_linear(node_filesystem_free{job="kubernetes-node-exporter"}[1h], 4 * 3600)

**Histogram（直方圖）**

**Summary（摘要）**

# **環境準備**

Ubuntu 16.04 LTS

Docker 1.10+

Kubernetes v1.9.6

Node_exporter v0.15+

**Prometheus v2.0.0**

**Grafana 5.1+**

# **安裝Prometheus**

**Create A Namespace**：

```linux
kubectl create namespace kube-ops
```

**Create Exporter**：

```linux
kubectl create -f node-exporter.yaml
```

node-exporter.yaml

```yaml
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-ops
  labels:
    k8s-app: node-exporter
spec:
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: prom/node-exporter
        name: node-exporter
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http

---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-ops
spec:
  ports:
  - name: http
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```

**Create ServiceAccount、ClusterRole、ClusterRoleBinding**：

```linux
kubectl create -f prometheus-sa.yaml
```

prometheus-sa.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-ops

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: kube-ops
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-ops
```
**Create A Config Map**：

```linux
kubectl create -f prometheus-cm.yaml
```

prometheus-cm.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-ops
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s
    rule_files:
    - /etc/prometheus/rules.yml
    alerting:
      alertmanagers:
        - static_configs:
          - targets: ["localhost:9093"]
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
    - job_name: 'kubernetes-cadvisor'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-node-exporter'
      scheme: http
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__meta_kubernetes_role]
        action: replace
        target_label: kubernetes_role
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:31672'
        target_label: __address__
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name
  rules.yml: |
    groups:
    - name: test-rule
      rules:
      - alert: NodeFilesystemUsage
        expr: (node_filesystem_size{device="rootfs"} - node_filesystem_free{device="rootfs"}) / node_filesystem_size{device="rootfs"} * 100 > 80
        for: 2m
        labels:
          team: node
        annotations:
          summary: "{{$labels.instance}}: High Filesystem usage detected"
          description: "{{$labels.instance}}: Filesystem usage is above 80% (current value is: {{ $value }}"
      - alert: NodeMemoryUsage
        expr: (node_memory_MemTotal - (node_memory_MemFree+node_memory_Buffers+node_memory_Cached )) / node_memory_MemTotal * 100 > 80
        for: 2m
        labels:
          team: node
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 80% (current value is: {{ $value }}"
      - alert: NodeCPUUsage
        expr: (100 - (avg by (instance) (irate(node_cpu{job="kubernetes-node-exporter",mode="idle"}[5m])) * 100)) > 80
        for: 2m
        labels:
          team: node
        annotations:
          summary: "{{$labels.instance}}: High CPU usage detected"
          description: "{{$labels.instance}}: CPU usage is above 80% (current value is: {{ $value }}"

      - alert: test
        expr: (100 - (avg by (instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter",mode="idle"}[5m])) * 100)) > 1
        for: 2m
        labels:
          team: node
        annotations:
          summary: "{{$labels.instance}}: High CPU usage detected"
          description: "{{$labels.instance}}: CPU usage is above 1% (current value is: {{ $value }}"

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: kube-ops
data:
  config.yml: |-
    global:
      resolve_timeout: 5m
    route:
      receiver: webhook
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      group_by: [alertname]
      routes:
      - receiver: webhook
        group_wait: 10s
        match:
          team: node
    receivers:
    - name: webhook
      webhook_configs:
      - url: 'http://apollo/hooks/dingtalk/'
        send_resolved: true
      - url: 'http://apollo/hooks/prome/'
        send_resolved: true
```

**Create A Prometheus Deployment**：

```linux
kubectl create -f prometheus-deploy.yaml
```

prometheus-deploy.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    k8s-app: prometheus
  name: prometheus
  namespace: kube-ops
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:v2.0.0-rc.3
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 1Gi
      - image: quay.io/prometheus/alertmanager:v0.12.0
        name: alertmanager
        args:
        - "-config.file=/etc/alertmanager/config.yml"
        - "-storage.path=/alertmanager"
        ports:
        - containerPort: 9093
          protocol: TCP
          name: http
        volumeMounts:
        - name: alertmanager-config-volume
          mountPath: /etc/alertmanager
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 200m
            memory: 200Mi
      volumes:
      - name: data
        emptyDir: {}
      - configMap:
          name: prometheus-config
        name: config-volume
      - name: alertmanager-config-volume
        configMap:
          name: alertmanager
```

**Exposing Prometheus As A Service**：

```linux
kubectl create -f prometheus-svc.yaml
```

prometheus-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-ops
  labels:
    k8s-app: prometheus
spec:
  selector:
    k8s-app: prometheus
  type: NodePort
  ports:
    - name: web
      port: 9090
      targetPort: http
```

**Create A Grafana Deployment**：

```linux
kubectl create -f grafana-deployment.yaml
```

grafana-deployment.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana
  namespace: kube-ops
  labels:
    app: grafana
    #component: core
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
        #component: core
    spec:
      containers:
      - image: grafana/grafana:latest
        securityContext:
            runAsUser: 0
        name: grafana
        imagePullPolicy: IfNotPresent
        # env:
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 100Mi
        env:
          # The following env variables set up basic auth twith the default admin user and admin password.
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          #   value: Admin
          # does not really work, because of template variables in exported dashboards:
          # - name: GF_DASHBOARDS_JSON_ENABLED
          #   value: "true"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
          # initialDelaySeconds: 30
          # timeoutSeconds: 1
        volumeMounts:
        - name: grafana-persistent-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-persistent-storage
        emptyDir: {}
```

**Exposing the Grafana web UI**：

```linux
kubectl create -f grafana-svc.yaml
```

grafana-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-ops
  labels:
    app: grafana
    #component: core
spec:
  type: NodePort
  ports:
    - port: 3000
  selector:
    app: grafana
    #component: core
```

# **Prometheus UI介面**

# **Grafana設定**