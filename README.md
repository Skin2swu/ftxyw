# 使用openwrt路由过校园网多设备检测（非破解）

由于lz在贴吧的校园网相关的教程已失效，这里补上新的。

### 关于校园网检测的手段（主要手段）
1.基于 IPv4 数据包包头内的 TTL 字段的检测（固定TTL）  
2.基于 HTTP 数据包请求头内的 User-Agent 字段的检测(UA2F)  
3.DPI (Deep Packet Inspection) 深度包检测技术）（极耗费学校设备性能,如果学校使用这种方法那真给学校了）  
5.基于 IPv4 数据包包头内的 Identification 字段的检测（rkp-ipid 设置 ipid）  
6.基于网络协议栈时钟偏移的检测技术（防时钟偏移检测）  
7.Flash Cookie 检测技术（iptables 拒绝 AC 进行 Flash 检测 不常用）

本校已经不是原先的ua与ttl和时间戳检测。

详细我并没有去测因为太费时间了，一次就要封10分钟，个个组合起来花的时间就多了。关于设备选择看个人了，lz用的j3160工控板＋路由器组合，j3160作为反制设备路由器做整个宿舍的无线ap和交换机并为j3160减轻一些压力，（j3160整套下来为100米，路由器型号为cr6606 45米）如果是选购单路由器注意一下路由器的性能
## 1.构建openwrt
首先需要一个可以刷写的路由器或拿一块如j3160或j1900这类的工控板做软路由。进入在线编译网站填写自己的设备型号来获取openwrt刷机包，关于如何刷入得根据设备自行寻找教程

例如：
[https://openwrt.ai/](https://openwrt.ai/)
[https://firmware-selector.immortalwrt.org/](https://firmware-selector.immortalwrt.org/)
[https://github.com/wukongdaily/ImmortalWrt-ImageBuilder](https://github.com/wukongdaily/ImmortalWrt-ImageBuilder)
https://firmware-selector.openwrt.org/

这边推荐kwrt也就是第一个网站因为方便构建，如有其他需求可以用下面3个分别是immortalwrt（比较纯净的版本）和一个自由度更好的immortalwrt项目最后一个是官方版本

## 2.根据自己的设备架构下载ua3f插件
UA3F的项目：[https://github.com/SunBK201/UA3F](https://github.com/SunBK201/UA3F)

这边是ua3f官方的安装教程： [https://sunbk201public.notion.site/UA3F-2a21f32cbb4b80669e04ec1f053d0333](https://sunbk201public.notion.site/UA3F-2a21f32cbb4b80669e04ec1f053d0333)

### 安装后进入ua3f页面，启用以下功能

1.`MitM设置`中的 `HTTPS MitM`（记得生成ca证书）

2.`Desync 设置`中的 `TCP 分片乱序发射`

3.`L3重写` 中的 `固定TTL` `删除TCP时间戳` `固定IPID`

`L3 重写 eBPF 卸载` 这一项能打开尽量打开能减轻路由器负担

（记得在软件包安装处安装提示的依赖，和要关闭的选项如防火墙的流量卸载）

最后保存应用即可，可以重启一下路由器确保生效

## 3.关于自动认证
由于宿舍区的认证界面由不知名第三方承包如发包的自动认证方式无法使用，需要每日手动认证一遍，注意一下安装插件时不要使用校园网否则直接10分钟了，请在安装完成后再接入校园网。

# 不使用UA3F的方法
 如果路由器性能使用ua3f太吃力了可以试试以下方法(虽然单独使用ipid也比较吃性能）
 ## 1.添加在线编译包时的插件
 在在线编辑步骤加入插件包
 这是我自己使用的软件包供大家参考可以直接复制,核心为`kmod-rkp-ipid` `iptables-mod-ipopt` `kmod-ipt-ipopt`(ipid修改与修改数据包TTL值的插件）如果lz漏写了导致有些不生效直接用lz的即可
这些软件包，大家可以根据自身情况删减或添加软件包
```
luci-theme-argon luci-app-taskplan luci-app-ttyd kmod-rkp-ipid iptables-mod-filter iptables-mod-ipopt iptables-mod-u32 iptables-nft kmod-ipt-ipopt ipset iptables-mod-conntrack-extra 
```
刷写进设备后关于改写ua可以用ua mask插件来代替ua2f（如果学校还在用的话）
项目链接：[https://github.com/Zesuy/UA-Mask](https://github.com/Zesuy/UA-Mask)
## 2.防检测配置
UA改写直接启用UAmask即可不用额外配置
进入防火墙选择自定义规则添加以下规则
```
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53

# 通过 rkp-ipid 设置 IPID

iptables -t mangle -N IPID_MOD
iptables -t mangle -A FORWARD -j IPID_MOD
iptables -t mangle -A OUTPUT -j IPID_MOD
iptables -t mangle -A IPID_MOD -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 127.0.0.0/8 -j RETURN

#由于本校局域网是172，所以我将这一条注释掉了

iptables -t mangle -A IPID_MOD -d 10.0.0.0/8 -j RETURN
# iptables -t mangle -A IPID_MOD -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A IPID_MOD -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A IPID_MOD -d 255.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -j MARK --set-xmark 0x10/0x10

# 防时钟偏移检测

iptables -t nat -N ntp_force_local
iptables -t nat -I PREROUTING -p udp --dport 123 -j ntp_force_local
iptables -t nat -A ntp_force_local -d 0.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 127.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 192.168.0.0/16 -j RETURN
iptables -t nat -A ntp_force_local -s 192.168.0.0/16 -j DNAT --to-destination 192.168.1.1

# 通过 iptables 修改 TTL 值 数字为需要的修改的ttl值

iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64

# iptables 拒绝 AC 进行 Flash 检测

iptables -I FORWARD -p tcp --sport 80 --tcp-flags ACK ACK -m string --algo bm --string " src=\"http://1.1.1." -j DROP
```
进入 OpenWRT 系统设置, 勾选 Enable NTP client（启用 NTP 客户端）和 Provide NTP server（作为 NTP 服务器提供服务）

NTP server candidates（候选 NTP 服务器）四个框框分别填写:
```
ntp1.aliyun.com
time1.cloud.tencent.com
stdtime.gov.hk
pool.ntp.org
```
在防火墙常规设置处将流量卸载关闭`选择‘无’`(如果并没有影响ipid的话可以不关闭）

最后保存应用重启一下路由器即可
### 由于我并没有去专门测试本校网络（UA3F方法实测是生效的）此方法可能不生效
