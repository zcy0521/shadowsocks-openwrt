# Shadowsocks OpenWrt

## 安装Shadowsocks

- 添加openwrt-dist.pub

```shell
wget http://openwrt-dist.sourceforge.net/openwrt-dist.pub
opkg-key add openwrt-dist.pub
```

- 执行`opkg print-architecture | awk '{print $2}'` 并修改`/etc/opkg/customfeeds.conf`

```
src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/x86_64
src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci
```

- 安装shadowsocks

```shell
opkg update
opkg install luci-compat
opkg install shadowsocks-libev luci-app-shadowsocks
opkg install iptables-mod-tproxy ip
```

## 配置 luci-app-shadowsocks

### General Settings

- Global Settings
  - Startup Delay: 5 seconds

- Transparent Proxy
  - Main Server: [SS_SERVER]
  - UDP-Relay Server: Disable
  - Local Port: 1234
  - Override MTU: 1492

### Servers Manage

- SS_SERVER
  - Alias: SS_SERVER
  - TCP Fast Open: True
  - TCP no-delay: True
  - Server Address: 127.0.0.1
  - Server Port: 8388
  - Connection Timeout: 300
  - Encrypt Method: CHACHA20-IETF-POLY1305

### Access Control

- 安装ChinaDNS

```shell
opkg update
opkg install ChinaDNS
```

- 更新`CHNRoute`

```shell
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt
```

- Zone WAN
  - Bypassed IP List: ChinaDNS CHNRoute
  - Bypassed IP: 222.222.222.222, 222.222.202.202
  - Forwarded IP: 8.8.8.8, 8.8.4.4

- Zone LAN
  - Interface: Ethernet Adapter: "eth0"
  - Proxy Type: Normal
  - Self Proxy: Normal
  - Extra arguments: None

## [Kcptun](https://github.com/xtaci/kcptun/releases/)

- 下载[Kcptun](https://github.com/xtaci/kcptun/releases/download/v20210103/kcptun-linux-amd64-20210103.tar.gz)

```shell
scp client_linux_amd64 root@192.168.50.254:/root
cp client_linux_amd64 /usr/bin/kcptun
chmod +x /usr/bin/kcptun
```

- 编辑`/etc/config/kcptun.json`

```json
{
  "localaddr": ":8388",
  "remoteaddr": "[KCP_SERVER_IP]:4000",
  "key": "HelloKcptun!",
  "crypt": "aes-128",
  "mode": "fast3",
  "conn": 1,
  "autoexpire": 300,
  "mtu": 1400,
  "sndwnd": 512,
  "rcvwnd": 4096,
  "datashard": 30,
  "parityshard": 15,
  "dscp": 46,
  "nocomp": true,
  "acknodelay": false,
  "nodelay": 0,
  "interval": 20,
  "resend": 2,
  "nc": 1,
  "sockbuf": 4194304,
  "keepalive": 10,
  "log": "/var/log/kcptun.log"
}
```

- 编辑`/etc/init.d/kcptun`

```
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=01
NAME=kcptun
start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/kcptun -c /etc/config/kcptun.json
    procd_set_param respawn
    procd_set_param pidfile /var/run/kcptun.pid
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

- 运行Kcptun

```shell
chmod +x /etc/init.d/kcptun

/etc/init.d/kcptun enable
/etc/init.d/kcptun start
```

## 访问控制

- 安装[dnsmasq-full](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#enabling-dnssec)

```shell
opkg update
opkg install dnsmasq-full --download-only && opkg remove dnsmasq && opkg install dnsmasq-full --cache .
```

- 编辑`/etc/dnsmasq.conf`

```shell
mkdir /etc/dnsmasq.d
echo "conf-dir=/etc/dnsmasq.d" >> /etc/dnsmasq.conf
```

- 新建文件[`/etc/dnsmasq.d/whitelist.conf`](https://github.com/shadowsocks/luci-app-shadowsocks/wiki/GfwList-Support)

```
# synology
server=/checkip.synology.com/127.0.0.1#5353
ipset=/checkip.synology.com/ss_spec_dst_bp

# qnap
server=/checkip.dyndns.org/127.0.0.1#5353
ipset=/checkip.dyndns.org/ss_spec_dst_bp
```

## DNS防污

### 配置V2Ray作为DNS服务

- 下载 [V2Ray](https://github.com/v2fly/v2ray-core/releases/download/v4.35.0/v2ray-linux-64.zip)

```shell
scp v2ctl v2ray geoip.dat  geosite.dat root@192.168.50.254:/root
cp v2ctl v2ray geoip.dat  geosite.dat /usr/bin/
chmod +x /usr/bin/v2ctl /usr/bin/v2ray
```

- 安装依赖

```shell
opkg update
opkg install ca-certificates iptables-mod-tproxy
```

- 编辑`/etc/config/v2ray.json`

```json
{
  "log": {
    "access": "/var/log/access.log",
    "error": "/var/log/error.log",
    "loglevel": "warning"
  },
  "dns": {
    "servers": [
      {
        "address": "127.0.0.1",
        "port": 5300,
        "domains": [
          "geosite:geolocation-!cn",
          "geosite:tld-!cn",
          "geosite:google",
          "geosite:twitter",
          "geosite:netflix",
          "geosite:spotify"
        ]
      },
      {
        "address": "222.222.222.222",
        "port": 53,
        "domains": [
          "geosite:cn"
        ]
      }
    ]
  },
  "inbounds": [
    {
      "port": 5353,
      "protocol": "dokodemo-door",
      "settings": {
        "address": "127.0.0.1",
        "port": 5300,
        "network": "tcp,udp"
      },
      "tag": "dns-in"
    }
  ],
  "outbounds": [
    {
      "protocol": "dns",
      "tag": "dns-out"
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "inboundTag": "dns-in",
        "outboundTag": "dns-out"
      }
    ]
  }
}
```

- 编辑`/etc/init.d/v2ray`

```
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=01
NAME=v2ray
start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/v2ray -c /etc/config/v2ray.json
    procd_set_param respawn
    procd_set_param pidfile /var/run/v2ray.pid
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

- 运行v2ray

```shell
chmod +x /etc/init.d/v2ray

/etc/init.d/v2ray enable
/etc/init.d/v2ray start
```

### 配置dns-forwarder作为V2Ray上游

- 安装dns-forwarder

```shell
opkg update
opkg install dns-forwarder luci-app-dns-forwarder
```

- 配置dns-forwarder
  - Enable: True
  - Listen Port: 5300
  - Listen Address: 0.0.0.0
  - DNS Server: 8.8.8.8

### 配置系统DNS

- [让dnsmasq-full将dns请求转发给127.0.0.1#5353](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#dnssec-by-dnsmasq)
  1. Select the Network->DHCP and DNS menu entry.
  2. In the "General Settings" tab, enter the address `127.0.0.1#5353` as the only entry in the "DNS Forwardings" dialogue.
  3. In the "Resolv and Host files" tab tick the "Ignore resolve file" checkbox.
  
## OpenWrt作旁路由设置

- LAN (物理设置中取消桥接)

```shell
vi /etc/config/network

config interface 'lan'
	option ifname 'eth0'
	option proto 'static'
	option ipaddr '192.168.50.254'
	option netmask '255.255.255.0'
	option gateway '192.168.50.1'
	list dns '127.0.0.1#5353'
```

- 安装语言包

```shell
opkg update
opkg install luci-i18n-base-zh-cn luci-i18n-base-en
```

- 修改Web页面默认访问端口

```shell
vi /etc/config/uhttpd

config uhttpd main

	# HTTP listen addresses, multiple allowed
	list listen_http	192.168.2.1:8080
	# list listen_http	[::]:80

	# HTTPS listen addresses, multiple allowed
	list listen_https	192.168.2.1:8443
	# list listen_https	[::]:443
```

- 修改SSH默认访问端口

```shell
vi /etc/config/dropbear

config dropbear
	option PasswordAuth 'on'
	option Interface 'lan'
	option Port '8022'
```

- 修复日志时间时区问题

```shell
opkg install zoneinfo-asia

vi /etc/config/system

config system
        option zonename 'Asia/Shanghai'
        option timezone 'CST-8'
```

- 自动重启

```
# Reboot at 4:30am every day
# Note: To avoid infinite reboot loop, wait 70 seconds
# and touch a file in /etc so clock will be set
# properly to 4:31 on reboot before cron starts.
30 4 * * * sleep 70 && touch /etc/banner && reboot
```
