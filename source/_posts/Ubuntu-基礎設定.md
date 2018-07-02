---
title: Ubuntu 網路設定
date: 2018-06-29 16:29:35
tags: Ubuntu
---

# **修改Hostname**

**# 設定 Hostname**
```linux
vim /etc/hostname
```

**# 設定Hosts**
```linux
vim /etc/hosts
```
<!-- more -->

**# 重新啟動**
```linux
./etc/init.d/hostname.sh
```

**# 確認修改成功**
```linux
hostname
```

# **修改IP、DNS**

**# 修改 Ethernet 網路設定、設定DNS**
```linux
sudo vi /etc/network/interfaces


# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto ens160
iface ens160 inet static            # 固定 (靜態) IP
        address 10.140.20.202       # IP 位址
        netmask 255.255.255.0       # 網路遮罩
        gateway 10.140.20.1         # 預設閘道
dns-nameservers 168.95.1.1          # 主要的 DNS 伺服器位址
dns-nameservers 8.8.8.8             # 次要的 DNS 伺服器位址
        # dns-* options are implemented by the resolvconf package, if installe

```

**# 重新啟動網路**
```linux
sudo /etc/init.d/networking restart
```