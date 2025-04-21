# OpenWrt旁路由配置指南

本文档提供了将OpenWrt配置为旁路由及安装配置Tailscale的完整步骤。

## 1. 网络配置

### 基本网络设置

1. 登录OpenWrt终端（通过SSH或控制台）

2. 编辑网络配置文件：
   ```bash
   vi /etc/config/network
   ```

3. 配置LAN接口（确保与主路由在同一网段）：
   ```
   config interface 'lan'
     option type 'bridge'
     option ifname 'eth0'
     option proto 'static'
     option ipaddr '192.168.31.31'  # 与主路由同网段但不同IP
     option netmask '255.255.255.0'
     option gateway '192.168.31.1'  # 主路由IP
     option dns '114.114.114.114 8.8.8.8'  # DNS服务器
   ```

4. 保存并退出（按ESC，然后输入:wq）

5. 重启网络服务：
   ```bash
   /etc/init.d/network restart
   ```

### 禁用DHCP服务

1. 在Web界面中：
   - 导航到"Interfaces" > "lan"
   - 切换到"DHCP Server"标签
   - 勾选"Ignore interface"复选框
   - 保存设置

2. 或者通过终端：
   ```bash
   uci set dhcp.lan.ignore='1'
   uci commit dhcp
   /etc/init.d/dnsmasq restart
   ```

## 2. 配置软件源

### 设置中国镜像源

1. 备份当前源配置：
   ```bash
   cp /etc/opkg/distfeeds.conf /etc/opkg/distfeeds.conf.bak
   ```

2. 修改为国内镜像：
   ```bash
   # 清华源
   sed -i 's_downloads.openwrt.org_mirrors.tuna.tsinghua.edu.cn/openwrt_' /etc/opkg/distfeeds.conf
   
   # 如果清华源连接问题，可使用阿里云源
   sed -i 's_downloads.openwrt.org_mirrors.aliyun.com/openwrt_' /etc/opkg/distfeeds.conf
   ```

3. 更新软件包列表：
   ```bash
   opkg update
   ```

## 3. 安装和配置Tailscale

### 安装Tailscale

1. 安装必要依赖：
   ```bash
   opkg install libustream-openssl ca-bundle kmod-tun iptables-mod-extra
   opkg install ca-certificates
   ```

2. 安装Tailscale：
   ```bash
   opkg install tailscale
   ```

### 配置Tailscale

1. 启动Tailscale服务：
   ```bash
   service tailscale start
   ```

2. 设置并启用Tailscale：
   ```bash
   tailscale up --advertise-routes=192.168.31.0/24 --accept-routes
   ```
   
   注意：以上命令会通告您的本地网络(192.168.31.0/24)到Tailscale网络，根据您的实际网段调整。

3. 设置开机自启：
   ```bash
   service tailscale enable
   ```

4. 查看Tailscale状态：
   ```bash
   tailscale status
   ```

## 4. 故障排除

### DNS问题

如果遇到DNS解析问题（无法ping域名），确保设置了正确的DNS服务器：

```bash
# 编辑网络配置
vi /etc/config/network

# 在lan接口下添加
option dns '114.114.114.114 8.8.8.8'

# 重启网络
/etc/init.d/network restart
```

### Tailscale连接问题

1. 检查IP转发是否启用：
   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   ```
   应该显示"1"，如果显示"0"，执行：
   ```bash
   echo 1 > /proc/sys/net/ipv4/ip_forward
   echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
   sysctl -p
   ```

2. 如果使用了`--netfilter-mode=off`但遇到连接问题，重置配置：
   ```bash
   tailscale down
   tailscale up --reset --advertise-routes=192.168.31.0/24 --accept-routes
   ```

3. 检查防火墙配置：
   - 确保OpenWrt允许tailscale接口间的转发
   - 如使用nftables (fw4)，可查看规则：
     ```bash
     nft list ruleset
     ```

## 5. 中文界面设置

如果需要中文界面，安装相应语言包：

```bash
# 查找可用中文包
opkg list | grep luci-i18n | grep zh

# 安装中文包
opkg install luci-i18n-base-zh-cn
```

## 附录：常用命令

- 检查网络连接：`ping 192.168.31.1`
- 查看IP配置：`ip addr`
- 查看路由表：`ip route`
- Tailscale连接测试：`tailscale ping 对方IP`
- Tailscale网络检查：`tailscale netcheck` 