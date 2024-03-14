# ArchWrt in nspawn

## 主要用途

和容器运行 openwrt 用途一致，但是可以使用 systemd 

## 环境示例

- Raspberry Pi 4 (ArchArm)
- systemd 255 (255.4-2-arch)
- ArchArm (container)

## 准备工作

1. 部署容器环境，可以按照 ArchWiki 上设置 [点击查看](https://wiki.archlinuxcn.org/wiki/Systemd-nspawn#%E7%94%A8%E4%BE%8B)
  > 其实不止可以用 Arch ，也可以用 Debian / Ubuntu / openSUSE 等等发行版本来搭建
2. 然后将容器导入到 `/var/lib/machines` 中，如果该目录是使用 btrfs 将会创建子卷，更方便设置快照。~~没那么容易滚挂~~
  ```bash
  sudo machinectl import-fs <ContainerPath> <ContainerName> # 自己填路径和名字
  ```
> 整体来说，准备工作比 openwrt 要方便

## 容器配置及网络配置

### 配置容器外部网络

**此处只使用** `systemd-networkd` **作为示例**

配置一个网桥，例如起名为 `br-wrt` ，配置文件如下

```
# /etc/systemd/network/br-wrt.netdev

[NetDev]
Name=br-wrt
Kind=bridge
```

```
# /etc/systemd/network/br-wrt.network
[Match]
Name=br-wrt

[Network]
# 设置 DHCP 可以方便配置 ipv4/ipv6 双栈上网
DHCP=yes
```

配置完后，重启 `systemd-networkd`

```
sudo systemctl restart systemd-networkd
```

### 配置容器

如果是第一次配置 nspawn 的文件，那么需要先创建 `/etc/systemd/nspawn` 这个文件夹

创建配置文件

```nspawn
/etc/systemd/nspawn/archwrt.nspawn     
[Exec]
PrivateUsers=yes

[Network]
# 将本机接口 end0 分配给容器
#Interface=end0:eth0
# 使用 MACVLAN 
MACVLAN=end0:eth0
# 使用 IPVLAN  
#IPVLAN=wlan0:wl0
# 与宿主连接的网桥
Bridge=br-wrt
```

1. 建议使用 MACVLAN ，在容器挂掉的情况下不会影响与宿主的连接。
2. MACVLAN 不能在**无线网络接口**使用
3. 网桥 `br-wrt` 接口在容器中为 `host0`
4. 如果要使用 dae 代理的话，需要额外的配置，后面的详细说 [跳转](#)
5. 如果宿主更新 systemd 至 256 ，可以直接将 wifi 接口移到容器内 [点击查看](https://github.com/systemd/systemd/issues/7873) 或者备用方式 [点击查看](https://github.com/systemd/systemd/issues/7873#issuecomment-451494943)
6. 如果 wifi 用来宿主上网的话，可以容器连上 wifi 但关闭 DHCP ，然后容器开启 DHCP ，宿主通过容器上网，想要处理访问宿主问题可以设置端口转发来实现 DMZ

### 配置容器内部网络

```bash
sudo machinectl start archwrt
sudo machinectl shell root@archwrt
```
> 最好就是设置一个**普通用户**来配置。至少对于 Arch 来说，方便 makepkg 。

1. 配置与宿主连接的网桥 host0 ，路径一定是下面提到的，原因详见 `/usr/lib/systemd/network/80-container-host0.network` 这个文件，需要你覆盖这个文件的配置
```
# /etc/systemd/network/80-container-host0.network

[Match]
Kind=veth
Name=host0
Virtualization=container

[Network]
# ipv4 网段
Address=10.10.0.1/24
# ipv6 网段
Address=fd67:6767:6767::/64
MulticastDNS=yes
# 开启 NAT 伪装
IPMasquerade=both
IPv6SendRA=yes

[IPv6SendRA]
Managed=yes
OtherInformation=yes
```

2. 配置对外有线接口 `eth0` 的连接

```
# /etc/systemd/network/eth0.network

[Match]
Name=eth0

[Network]
DHCP=yes
IPv6AcceptRA=yes

[DHCP]
# 这里配置有线的权重比无线高(数值小)
RouteMetric=10
```

3. 配置对外无线接口 `wl0` 的连接

```
# /etc/systemd/network/wl0.network

[Match]
Name=wl0

[Network]
DHCP=yes
IPv6AcceptRA=yes

[DHCP]
RouteMetric=200
```

4. (**可选**) 容器内配置 MACVLAN (没错！就是套娃设置，私有网络的话 nspawn 会自动给特权) 

  > 在用来开 MACVLAN 的接口配置文件上加上下面这部分

```
# /etc/systemd/network/eth0.network

[Network]
MACVLAN=eth0mv0
```
  > 下面是配置 `eth0mv0`
```
# /etc/systemd/network/eth0mv0.netdev

[NetDev]
Name=eth0mv0
Kind=macvlan

[MACVLAN]
Mode=vepa
```

```
# /etc/systemd/network/eth0mv0.network

[Match]
Name=eth0mv0

[Network]
DHCP=both
IPv6AcceptRA=yes

[DHCP]
RouteMetric=300
```

## (可选) 设置默认 dns 解析

使用 systemd-resolved 的目的是获取上游的 dns ，如果你有计划设置自定义 dns 的话，可以忽略

```
sudo systemctl enable --now systemd-resolved
```

> 将监听在 127.0.0.53#53 上

## 配置 DHCP Server

使用的是 dnsmasq 

1. 在 `/etc/dnsmasq.conf` 上添加这行，启用配置文件夹 `/etc/dnsmasq.d/`

```
conf-dir=/etc/dnsmasq.d/,*.conf
```

```
# /etc/dnsmasq.d/default.conf

port=53
interface=host0
# 上游无污染 dns ，后面会提到用途
# server=127.0.0.1#6053

# 使用 systemd-resolved 作为上游，如果上面配置 dns 无法工作将可以 fallback 到这里
server=127.0.0.53

# 配置 dhcp server
bind-interfaces
dhcp-option=3,0.0.0.0
dhcp-option=6,0.0.0.0
# ipv4 网段，与 host0 接口静态 ip 有关
dhcp-range=10.10.0.100,10.10.0.200,255.255.255.0,12h
# ipv6 网段，与 host0 接口静态 ip 有关
dhcp-range=::f,::fff,constructor:host0,ra-names,slaac
# ipv6 有关
enable-ra
dhcp-authoritative

# local
local=/lan/
domain=lan
expand-hosts
```

```
# /etc/dnsmasq.d/static.conf

# 绑定 MAC (可选)
dhcp-host=ff:ff:ff:ff:ff:ff,10.10.0.2
```

最后启用 dnsmasq

```
sudo systemctl enable --now dnsmasq
```

## 配置防火墙

使用的是 nftables ，详见 [nftables.conf](./nftables.conf)

在配置格式上模拟了 openwrt 的 nftables 输出，行为与 openwrt 一致，分成 lan 与 wan 两个区

复制粘贴在 `/etc/nftables.conf` 上，然后修改下文件开始的接口配置，本人设定如下

```
# 默认 lan 
define DEV_LAN = host0
define LAN = { $DEV_LAN }

# 设置三种 wan : wl0(WLAN), eth0(ETH), eth0mv0(ETH MACVLAN)
# 按自己需要来配置需要的
define DEV_WWAN = wl0
define DEV_WAN = eth0
define DEV_MVWAN0 = eth0mv0
define WAN = { $DEV_WAN, $DEV_WWAN, $DEV_MVWAN0 }
```

> 或者你也可以使用其他便捷防火墙，例如红帽的 firewalld

## 设置开机启动

配置完后，可以让容器启动，并设置为随开机启动了。

```
sudo machinectl enable --now archwrt
```

## 透明代理

### 透明代理宿主

**此处只使用** `systemd-networkd` **作为示例**

很简单的，直接调整 `br-wrt` 与其他接口的跃点，使 `br-wrt` 的权重最高。例如下面配置

```
# /etc/systemd/network/br-wrt.network
# 在该文件添加这个小节

[DHCP]  
RouteMetric=5  # 这个数值要比其文件中小，保持权重最高
```

```
# /etc/systemd/network/70-en.network
# 另一个接口添加这一小节

[DHCP]
RouteMetric=20
```

重启 `systemd-networkd` ，稍后透明代理成功

### 容器配置透明代理

方案是 dae + sing-box ，dae 只处理国内分流与透明代理， sing-box 处理代理协议与 dns ，安装教程这里无需多讲

#### sing-box 配置

详见 [点击查看](./sing-box.json)

1. 将 127.0.0.1:1080 设置为 socks5 地址，处理 dae 转发过来的出国流量，然后通过 hy2 协议转发出去
2. 将 127.0.0.1:6053 设置为无污染 dns 的地址
3. 自行根据文档配置节点和功能

#### dae 配置

详见 [点击查看](./dae.dae)

1. 复制粘贴到 `/etc/dae/config.dae` 上即可
2. 如果是使用 firewalld 必须配置 `auto_config_firewall_rule: true` 这个选项，在重启 firewalld 后必须重启 dae ，详见 [点击查看](https://github.com/daeuniverse/dae/pull/420)
3. 承接 2 那点，使用 nftables 的话，需要运行以下命令(已经添加到提供的 [`nftables.conf`](./nftables.conf) 中了)
```
nft 'insert rule inet filter input mark & 0x08000000 == 0x08000000 accept'
```
4. 如果是在 arm 架构之类的嵌入式设备上使用，必须手动编译内核启用 BTF 详见 [点击查看](https://github.com/daeuniverse/dae/blob/main/docs/en/user-guide/kernel-upgrade.md)
5. 承接 3 那点，如果使用运行 archlinuxarm 的 rpi 3b/4 ，可以使用我编译的内核，详见 [点击查看](https://github.com/67au/linux-rpi-btf)

#### 启用代理

```
sudo systemctl enable --now sing-box
sudo systemctl enable --now dae
```

## 结语

ArchWrt ，启动！

```
                  -`                     67au@archwrt
                 .o+`                    ------------
                `ooo/                    OS: Arch Linux ARM aarch64
               `+oooo:                   Kernel: 6.6.21-2-rpi-btf
              `+oooooo:                  Uptime: 10 hours, 56 mins
              -+oooooo+:                 Packages: 191 (pacman)
            `/:-:++oooo+:                Shell: zsh 5.9
           `/++++/+++++++:               Terminal: xterm-256color
          `/++++++++++++++:              CPU: ARMv8 rev 3 (v8l) (4) @ 1.80 GHz
         `/+++ooooooooooooo/`            Memory: 325.94 MiB / 3.70 GiB (9%)
        ./ooosssso++osssssso+`           Swap: 0 B / 4.00 GiB (0%)
       .oossssso-````/ossssss+`          Disk (/): 18.82 GiB / 29.20 GiB (64%) - btrfs
      -osssssso.      :ssssssso.         Local IP (eth0): 10.0.0.188/24 *
     :osssssss/        osssso+++.        Locale: C
    /ossssssss/        +ssssooo/-
  `/ossssso+/:-        -:/+osssso+-      
 `+sso+:-`                 `.-/+oso:     
`++:.                           `-/+/
.`                                 `/
```