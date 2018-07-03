---
title: Helm
date: 2018-07-02 14:25:57
tags: Helm
---

# **Helm 介紹**
Helm 是 Kubernetes Chart 的管理工具，Kubernetes Chart 是一套預先組態的 Kubernetes 資源套件。隨著容器化與微服務架構的出現，帶給我們便利的同時，應用被拆分成多個組件，導致服務數量大幅增加，對於 Kubernetes 來說，每個組件有自己的資源文件，並且可以獨立進行佈署與伸縮，而 Helmd 可以簡化這種模式在佈署與管理的複雜性。

Helm 把 Kubernetes 資源打包到一個 chart 中，而 chart 被保存到 chart 倉庫。通過 chart 倉庫可用來儲存和分享 chart 。Helm 使應用發佈可配置，支持版本管理，簡化了 Kubernetes 佈署應用的版本控制、打包、發佈、刪除、更新、退版等操作。

<!-- more -->

# **Helm 架構**

![](https://i.imgur.com/kATeTNG.png)


# **Helm 概念**

**# Chart**

包含 Kubernetes 應用程序資源、佈署檔案的集合，為 Helm 的打包格式

**# Release**

在 Kubernetes 中運行的 Chart 實例，類似 Kubernetes 的 Deployment

**# Repository**

Helm的軟體倉庫，可儲存 Chart 軟體包已供下載，並有 Chart 包的清單檔案可查詢


# **Helm 元件**

**# Helm Client**

一個安裝 Helm CLI 的機器，該機器透過 gRPC 連接 Tiller Server 來對 Repository、Chart 與 Release 等進行管理與操作，如建立、刪除與升級等操作

**# Tiller Server**

主要負責接收來至 Client 的指令，並透過 kube-apiserver 與 Kubernetes 叢集做溝通，根據 Chart 定義的內容，來產生與管理各種對應 API 物件的 Kubernetes 佈署


# **Helm 安裝**

**# 安裝 Helm Client**
```linux
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

**# 創建 Tiller 的 serviceaccount 和 clusterrolebinding**
```linux
kubectl create serviceaccount --namespace kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

**# 或是使用yaml，內容如下**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
**# 將上面內容存成helmsec.yaml**
```linux
vim helmsec.yaml
```

**# 創建對應的 service account 和 role binding**
```linux
kubectl create -f helmsec.yaml
```

**# 安裝 Tiller Server**
```linux
helm init --service-account tiller
```

**# 確認版本**
```linux
helm version
```

**# 舊版升級**
```linux
helm init --upgrade
```