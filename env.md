# bocker 项目环境要求

>参考文件：
https://www.codercto.com/a/41843.html
https://juejin.im/post/5bfd5f2ff265da61764a94ff


## vagrant 操作

todo： 如果懂vagrant，使用vagrant操作会方便的多

## centos 系统

- 源码中脚本要求使用的命令

```
yum install -y -q autoconf automake btrfs-progs docker gettext-devel git libcgroup-tools libtool python-pip jq
```
- 文件挂载系统btrfs
此项目中仅支持 btrfs 文件挂载
```
fallocate -l 10G ~/btrfs.img
mkdir /var/bocker
mkfs.btrfs ~/btrfs.img
mount -o loop ~/btrfs.img /var/bocker
```

- 安装linux-utils 一个linux的工具 unshare 工具
```
git clone https://github.com/karelzak/util-linux.git
cd util-linux
git checkout tags/v2.25.2
./autogen.sh
./configure --without-ncurses --without-python
make
mv unshare /usr/bin/unshare
```

- 创建网卡和网络转发
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables --flush
iptables -t nat -A POSTROUTING -o bridge0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp0s31f6 -j MASQUERADE
ip link add bridge0 type bridge
ip addr add 10.0.0.1/24 dev bridge0
ip link set bridge0 up

1 首先需要创建一块虚拟网卡 bridge0 ，然后配置bridge0网卡的nat地址转换，这里bridge相当于docker中的 docker0 ，bridge0相当于在网络中的交换机二层设备，他可以连接不同的网络设备，当请求到达Bridge设备时，可以通过报文的mac地址进行广播和转发，所以所有的容器虚拟网卡需要在bridge下，这也是连接 namespace 中的网络设备和 宿主机网络 的方式，这里下变会有讲解。(如果需要实现 overlay 等，需要换用更高级的转换工具，如用ovs来做类 vxlan，gre 协议转换)

2 开启内核转发和配置iptables MASQUERADE ，这是为了用MASQUERADE规则将容器的ip转换为宿主机出口网卡的ip，在linux namespace中，请求宿主机外部地址时，将namespace中的原地址换成宿主机作为原地址，这样就可以在namespace中进行地址正常转换了。

