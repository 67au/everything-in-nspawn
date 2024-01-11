# Openwrt in nspawn

## 主要用途

运行 openwrt 来透明代理本机，同时也可以单独为其他容器代理，且不限于 docker / podman / nspawn 。此外，也可以作为旁路网关来使用。

## 环境示例

- Raspberry Pi 4 (ArchArm)
- systemd 255 (255.2-2.1-arch)
- openwrt 23.05.2 (armsr/armv8)

## 准备工作

1. 下载 openwrt 镜像([传送门](https://downloads.openwrt.org/))，根据设备的架构来选择 rootfs ，容器使用无需包含内核。
  - openwrt 23.05.2 x86-64
  ```
  https://downloads.openwrt.org/releases/23.05.2/targets/x86/64/openwrt-23.05.2-x86-64-rootfs.tar.gz
  ```
  - openwrt 23.05.2 armsr/armv8
  ```
  https://downloads.openwrt.org/releases/23.05.2/targets/armsr/armv8/openwrt-23.05.2-armsr-armv8-rootfs.tar.gz
  ```
2. 解压并导入 nspawn
  - 解压 
  ```bash
  mkdir openwrt
  bsdtar -xvzf openwrt-23.05.2-armsr-armv8-rootfs.tar.gz -C openwrt
  ```
  - 导入
  ```bash
  sudo machinectl import-fs openwrt openwrt
  ```
  > import 之后，容器会储存在 `/var/lib/machines/openwrt` 。
  > 而如果使用 btrfs ,容器会以 btrfs 子卷形式储存。
3. 初次启动 openwrt 使其初始化生成文件
  ```bash
  sudo systemd-nspawn -D /var/lib/machines/openwrt -b
  ```
  > 使用参数 `-b` 会启动 init ，然后一两分钟后即可连续按三次 `^]` （即 `Ctrl+]` ）杀死容器。
4. 使用 systemd-nspawn 来 chroot openwrt 并删除 `procd-ujail` ， `procd-ujail` 在容器当中无法正常运作
  - 启动
  ```
  sudo systemd-nspawn -D /var/lib/machines/openwrt
  ```
  - 进入到 openwrt 中删除 `procd-ujail`
  ```
BusyBox v1.36.1 (2023-11-14 13:38:11 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 23.05.2, r23630-842932a63d
 -----------------------------------------------------
root@openwrt:~# 
  ```
  ```
  mkdir /var/lock # 可能需要这一步
  opkg remove procd-ujail
  ```
5. 基本准备工作已经完成了

## 容器配置及网络配置

### 配置容器外部网络

**此处只使用** `systemd-networkd` **作为示例**

配置一个网桥，例如我起名为 `br-openwrt` ，配置文件如下
```
# /etc/systemd/network/br-openwrt.netdev

[NetDev]
Name=br-openwrt
Kind=bridge
```
```
cat /etc/systemd/network/br-openwrt.network
[Match]
Name=br-openwrt

[Network]
Address=10.10.0.2/24  # 配置网桥想要使用的网段，宿主使用静态地址，并将 10.10.0.1 留给 openwrt 容器
```
配置完后，重启 `systemd-networkd`
```
sudo systemctl restart systemd-networkd
```

### 配置容器

如果你是第一次配置 nspawn 的文件，那么你需要先创建 `/etc/systemd/nspawn` 这个文件夹

创建配置文件

```nspawn
/etc/systemd/nspawn/openwrt.nspawn     
[Exec]
PrivateUsers=yes

[Network]
#Interface=end0:eth0  # 将本机接口 end0 分配给容器
MACVLAN=end0:eth0  # 使用 MACVLAN 
#IPVLAN=wlan0:eth0  # 使用 IPVLAN
Bridge=br-openwrt  # 与宿主连接的网桥
```
1. 配置建议 openwrt 上 wan 的接口统一改名为 `eth0` ，当然你也可以用别的。
2. 建议使用 MACVLAN ，在容器挂掉的情况下不会影响与宿主的连接。
3. MACVLAN 不能在**无线网络接口**使用
4. `br-openwrt` 接口在容器中为 `host0`

### 配置容器内部网络

1. 先不要直接启动容器，通过 systemd-nspawn 来再次 chroot openwrt ，并根据前面的配置文件来设定容器内网络
  ```
  sudo systemd-nspawn -D /var/lib/machines/openwrt
  ```
2. 进入容器后，修改 `/etc/config/network` ，~~你在宿主直接修改文件也可以的~~，修改以下节点，**不要直接用下面文件**
  ```
  config device    # 配置 lan 的网桥
	  option name 'br-lan'
	  option type 'bridge'
	  list ports 'host0'  # lan 网桥连接 host0 用于与宿主通信

  config interface 'lan'  # 配置 lan 部分
	  option proto 'static'
	  option device 'br-lan'
	  list ipaddr '10.10.0.1/16'  # 前面配置的网段
	  option broadcast '10.10.0.255'

  config interface 'wan'  # 配置 wan 部分，如果该部分不存在，那么直接创建
    option device 'eth0'
    option proto 'dhcp'

  config interface 'wan6'  # 配置 ipv6 ，可选
    option device 'eth0'
    option proto 'dhcpv6'
  ```
3. 因为 wan 部分的防火墙存在，如果你不在宿主转发 openwrt 的端口，是没办法通过 webui 来直接配置 openwrt 的
  - 直接暴力的方法，修改容器 `/etc/config/firewall` 来允许来自 wan 的访问
    ```
    config zone  # 找到这部分
        option name 'wan'
        option input 'ACCEPT'  # 已经改成 ACCEPT
        option output 'ACCEPT'
        option forward 'REJECT'
        option mtu_fix '1'
        option masq6 '1'
        option masq '1'
        list network 'wan'
        list network 'wan6'
    ```
  - 在宿主配置反代，例如 nginx / caddy ，也可以配置端口转发，这里不详细说明了

## 后续工作

配置完后，可以让容器启动，并设置为随开机启动了。

```
sudo machinectl enable --now openwrt
```

启动后，可以通过 wan 口 ip 直接访问 openwrt 了，配置相应的防火墙规则后，记得调整来自 wan 的访问。

## Openwrt 配置

在这里就不详细说明了，配置好透明代理即可

## 透明代理

**此处只使用** `systemd-networkd` **作为示例**

很简单的，直接调整 `br-openwrt` 与其他接口的跃点，使 `br-openwrt` 的权重最高。例如下面配置

```
# /etc/systemd/network/br-openwrt.network
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
重启 `systemd-networkd` ，稍后即透明代理成功

## 注意事项

1. 由于 openwrt 不使用 systemd ，所以在外部无法完全控制容器的。如果不小心以 `machinectl start <容器>` 方式启动容器，
   可能会无法直接使用 `machinectl stop` 和 `machinectl kill` 来杀死容器。
   但是，可以先使用 `machinectl terminate` 再 `machinectl kill` 即可杀死容器。
