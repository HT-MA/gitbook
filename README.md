# k3s install

## K3s Install

### update

```shell
yum upodate -y
```

### install tools

```shell
yum install -y vim net-tools wget
```

### set hostname

```shell
hostnamectl set-hostname master
hostnamectl set-hostname node01
hostnamectl set-hostname node02
```

### set static network

```shell
[root@matser ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=edc3484d-b159-497e-a29b-b0a625d26418
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.43.135
NETMASK=255.255.255.0
GATEWAY=192.168.43.2
DNS1=114.114.114.114
DNS2=8.8.8.8
```

### Install k3s

#### master

```shell
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cnINSTALL_K3S_VERSION=v1.26.4+k3s1 sh -s - --with-node-id --bind-address 0.0.0.0
```

#### node
