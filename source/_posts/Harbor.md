---
title: Harbor
date: 2018-06-20 15:01:51
tags: Harbor
---

# **環境要求**

Ubuntu 16.04 LTS

Python 2.7+

Docker 1.10+

Docker-compose 1.6.0+

<!-- more -->

****

# **環境前置安裝**


**# Check Version**
```linux
ubuntu@ubuntu-xenial:~$ uname -a

ubuntu@ubuntu-xenial:~$ cat /etc/lsb-release
```


**# Install Python2.x**
```linux
ubuntu@ubuntu-xenial:~$ sudo apt-get update

ubuntu@ubuntu-xenial:~$ sudo apt-get install -y python
```  
  

**# Check Python Version**
```linux
ubuntu@ubuntu-xenial:~$ python --version
```  
  

**# Install Docker**
```linux
ubuntu@ubuntu-xenial:~$ sudo apt-get install -y docker.io
```
  
  

**# Check Docker Version**
```linux
ubuntu@ubuntu-xenial:~$ sudo docker version
```
  
  
  

**# Install Docker Compose**
```linux
ubuntu@ubuntu-xenial:~$ sudo curl -L   "https://github.com/docker/compose/releases/download/1.11.2/docker-compose-$(uname -s)-$(uname -m)"   -o /usr/local/bin/docker-compose
```
  
  

**# 設定權限**
```linux
ubuntu@ubuntu-xenial:~$ sudo chmod +x /usr/local/bin/docker-compose
```
  
  

**# Check Docker Compose Version**
```linux
ubuntu@ubuntu-xenial:~$ docker-compose --version
```
  
  

**# /etc/ssl/openssl.cnf內的[v3_ca]加入倉庫IP**
```linux
ubuntu@ubuntu-xenial:~$ sudo vim /etc/ssl/openssl.cnf  
[ v3_ca ]  
subjectAltName=IP:xx.xx.xx.xx
```
****

# **安裝 Harbor**


**# 下載版本v1.1.1**
```linux
ubuntu@ubuntu-xenial:~$ wget https://github.com/vmware/harbor/releases/download/v1.1.1/harbor-online-installer-v1.1.1.tgz
```
  
  

**# 解壓縮**
```linux
ubuntu@ubuntu-xenial:~$ tar zxvf harbor-online-installer-v1.1.1.tgz
```
  
  
**# Create Directory for Certificate and Change Directory**
```linux
ubuntu@ubuntu-xenial:~$ mkdir cert

ubuntu@ubuntu-xenial:~$ cd cert
```
  
  

**# Create Certificate**

**# Input Common Name only at this time**
```linux
ubuntu@ubuntu-xenial:~/cert$ openssl req -sha256 -x509 -days 365 -nodes -newkey rsa:4096 -keyout registry.08online.xsg.key -out registry.08online.xsg.crt
```
  
```linux
Country Name (2 letter code) \[AU\]:

State or Province Name (full name) \[Some-State\]:

Locality Name (eg, city) \[\]:

Organization Name (eg, company) \[Internet Widgits Pty Ltd\]:

Organizational Unit Name (eg, section) \[\]:

Common Name (e.g. server FQDN or YOUR name) \[\]:IP

Email Address \[\]:
```
  
  

**# Change Directory and Modify harbor.cfg**
```linux
ubuntu@ubuntu-xenial:~$ cd harbor

ubuntu@ubuntu-xenial:~/harbor$ vim harbor.cfg
```
  
  
```yaml
< hostname = reg.mydomain.com

\> hostname = registry.08online.xsg

  

< ui\_url\_protocol = http

\> ui\_url\_protocol = https

  

< ssl_cert = /data/cert/server.crt

\> ssl_cert = /home/ubuntu/cert/registry.08online.xsg.crt

  

< ssl\_cert\_key = /data/cert/server.key

\> ssl\_cert\_key = /home/ubuntu/cert/registry.08online.xsg.key
```
  
  

**# Harbor has been installed**
```linux
ubuntu@ubuntu-xenial:~/harbor$ sudo ./install.sh
```
  
  

**# Check Containers for Harbor**
```linux
ubuntu@ubuntu-xenial:~/harbor$ sudo docker-compose top
```
  
  

**#WebUI([https://](about:blank)IP)**

帳號：admin

密碼：xxxxx

  
****

# 安裝 Certificate 

**# 修改憑證，需用公司憑證取代**

```linux
ubuntu@ubuntu-xenial:~/harbor$ vim /home/ubuntu/cert/registry.08online.xsg.crt
  
ubuntu@ubuntu-xenial:~/harbor$ vim /home/ubuntu/cert/registry.08online.xsg.key
```
  
**# 修改Docker login需要之憑證**

```linux
ubuntu@ubuntu-xenial:~/harbor$ mkdir /etc/docker/certs.d/registry.08online.xsg
  
ubuntu@ubuntu-xenial:~/harbor$ vim /etc/docker/certs.d/registry.08online.xsg/ca.crt
  
ubuntu@ubuntu-xenial:~/harbor$ vim /etc/docker/certs.d/registry.08online.xsg/client.cert

ubuntu@ubuntu-xenial:~/harbor$ vim /etc/docker/certs.d/registry.08online.xsg/client.key
```

**# 測試登入**
```linux
ubuntu@ubuntu-xenial:~/harbor$ docker login registry.08online.xsg
```
帳號：admin

密碼：xxxxxx

  
****

# **LDAP設定**

  
**# 修改設定檔**

```linux
ubuntu@ubuntu-xenial:~/harbor$ vim haobor.cfg

  
  

< auth_mode = db_auth

> auth_mode = ldap_auth

  

< ldap_url = ldaps://ldap.mydomain.com

> ldap_url = ldap://xxx.xxx.xxx:xxx

  

< #ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

> ldap_searchdn = CN=xxxxxxxx,OU=PublicID,OU=Account,DC=xxx,DC=xxx

  

< #ldap_search_pwd = password

> ldap_search_pwd = xxxxxxxx

  

< ldap_basedn = ou=people,dc=mydomain,dc=com

> ldap_basedn = dc=xxx,dc=xxx

  

< ldap_uid = uid

> ldap_uid = sAMAccountName
```
  
  

**# 重啟服務並強制清除data目錄下資料**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker-compose down -v

ubuntu@ubuntu-xenial:~/harbor$ rm -rf /data

ubuntu@ubuntu-xenial:~/harbor$ ./prepare

ubuntu@ubuntu-xenial:~/harbor$ docker-compose up -d
```
  
  

**# 先使用管理者帳號登入**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker login registry.08online.xsg
```
帳號：admin

密碼：xxxxxx

  
  
  
  

**# 登入Web-UI調整設定**

Configuration > Authentication


測試連線成功後，即可使用AD帳號登入

****



# **Push、Pull**

  
  

**# 登入**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker login registry.08online.xsg
```
  
  

**# 幫需上傳的image加上tag**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker tag redis:latest registry.08online.xsg/redis:latest
```
  
  

**# 上傳**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker push registry.08online.xsg/library/redis:latest
```
  
  

**# 下載**

```linux
ubuntu@ubuntu-xenial:~/harbor$ docker pull registry.08online.xsg/library/redis:latest
```
****
