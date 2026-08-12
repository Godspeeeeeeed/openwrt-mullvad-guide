OpenWrt 反向分流 + Mullvad WireGuard 实战手册

作者：@Godspeeeeeeed_

适用：GL.iNet MT6000 Flint2/ OpenWrt 25.12.5  + Mullvad
实测：B站直连 0.15s，Google 隧道 0.4s，重启全自动恢复，定时更新IP表
时间：2026-08-12

说明：该文档由我与DeepSeek pro V4 连续对话20小时的内容总结而来，比较菜，烧了2亿 Token，只能保证文档中的所有代码经本人实测或DeepSeek反复搜索开源社区核实，不代表唯一解，请酌情使用。我是网络小白，完成此次实战前我只认得IP、子网掩码、MAC、DNS，内容准确性和真实性本人没有能力核实，文档本身也是DeepSeek提炼生成的，我只做了文本语义提炼优化，如有错误之处或新的点子请提出，以推进开源精神的细水长流。

0. 方案总览
┌─ DNS 层（反向分流）──────────────────────────────┐
│  dnsmasq(53)                                     │
│  ├─ 默认 → 223.5.5.5 / 119.29.29.29（国内域名） │
│  └─ gfwlist(几千条) → 100.64.0.23:53（被墙域名，隧道加密）│
├─ 流量层（IP 分流）───────────────────────────────┤
│  国内IP(nft集合1.2万条) → fwmark 0x100 → table100 │
│      → 宽带直连（快，0.15s）                      │
│  其他 → 默认路由 → Mullvad 隧道（安全，0.4s）     │
├─ 维护 ────────────────────────────────────────────┤
│  cron 每周一 03:00 三源合并自动更新 IP 表          │
└───────────────────────────────────────────────────┘
一句话优势：国内域名绝不发给 Mullvad（域名级精确分流）→ 国内快、国外安全、全自动维护、重启自恢复。

这套方案在干嘛：让路由器当"聪明分拣员"——国内网站走宽带直连（快），国外走 Mullvad 加密隧道（安全），DNS 分域解析防污染，设备端零设置。

1. 为什么用反向分流而不是 AGH（核心优势）
三个原因：

pbr反复失败：pbr 启动会先执行 Resetting routing（重置整张路由表）再装 nft 规则，在 VPN/多 WAN 复杂环境极易失败 → 路由表被清空 → 全网断（社区同款失败案例多，本质是"启动即动全局路由"的设计缺陷）
AGH 不够极致：所有 DNS 模式都做不到"国内域名绝不发给 Mullvad"的域名级精确分流
反向分流是社区公认方案：基于 dnsmasq 官方标准功能 server=/域名/上游，有官方文档（man page），使用人数多，出问题容易搜到现成解法
AGH 三种模式的硬伤（精炼）：

load_balance：随机选上游 → 国内查询可能落到 Mullvad（慢 + 返回国外节点）
parallel：并发把查询发给所有上游 → 国内查询量翻倍，易触发阿里/腾讯免费版 20QPS 限速
fastest_addr：需要等全部上游（含隧道 300ms+）返回后测速 → 国内解析延迟必增，最影响国内访问体验
三种模式都绕不开"Mullvad 在列表里"的问题，AGH 无法实现域名级精确分流——这正是反向分流的根本优势。

结论：AGH 不是不能用，是不够好用。它的短板是VPN DNS 解析速度把内外网拖成同一水平。对外网需求不高的人，AGH/全隧道是给自己找罪受，不如需要时手动开 VPN 客户端。

⚠️ 隐私缺陷警告（高隐私需求人群必读）
本方案国内 DNS 是明文 UDP（223.5.5.5），国内流量（DNS + 访问目标 IP）走宽带直连，运营商可见你查询的国内域名、访问的国内网站（HTTPS 内容加密，但访问行为可见）。
请勿用本方案从事任何违规、违法或需隐匿行踪的操作。 追求绝对隐私时（敏感内容、BT 等），临时开启设备级 VPN 客户端或改全隧道模式。
一句话：本方案是"国内便利 + 国外安全"的平衡方案，不是匿名方案。

公共 DNS 限速背景（多源核实）：

阿里 223.5.5.5/223.6.6.6（官方公告 2024-09-30 起）：DoH/DoT 单 IP 20QPS，UDP/TCP 2000bps
腾讯 119.29.29.29（V2EX/网易证实）：免费版单域名 20QPS
结论：家庭查询量远低于 20QPS，基本不触发；只有 AGH 并发 DoH 才易撞限速——反向分流天然规避
1.1 AGH 分流的妥协玩法（流派对比）
除了反向分流，还有一条"不用维护列表"的路线——ChinaDNS / smartdns / mosdns 的"上游对比递归分流"，它们是 AGH 统一分流的妥协变体：

收到查询 → 同时发「国内DNS」和「可信DNS(走隧道)」
 ├─ 国内DNS返回【国内IP】→ 采用（CDN优化）✓
 └─ 国内DNS返回【国外IP】→ 判定污染 → 丢弃, 用可信DNS真实结果 ✓
工具	特点
ChinaDNS（老牌）	双上游对比，经典防污染
smartdns	多上游测速 + 分组判断
mosdns（当前主流）	plugin 编排 + 分流 + 配合 OpenClash/PassWall
与反向分流的对比：

维度	反向分流	ChinaDNS 式对比
列表	维护 gfwlist	零维护
精确性	域名级精确	依赖上游行为，污染判断有边缘情况
资源	单上游查询	每查询发双上游（流量翻倍）
可控性	完全可控	逻辑黑盒难排查
成熟度	dnsmasq 标准功能	需装 smartdns/mosdns 服务

2. 适用性
✅ 路由器下所有有线/无线设备全生效（分流发生在路由器上）
❌ 例外：设备自装 VPN 客户端（强烈建议卸载）、残留虚拟网卡（ipconfig 见 utun/wintun → 禁用/设备管理器删除）

3. ⚠️ 先装工具 + 预下载数据（防断网无法推进）
apk update && apk add curl tcpdump bind-dig netcat strace
工具	用途
curl	下载（自带 wget 缺 TLS，报 Operation not permitted 就用它）
tcpdump	抓包定位"发没发出、有没有回包"
dig	DNS 排查（可指定端口、+short）
netcat	测端口连通
strace	查进程沙箱行为
趁网络可用，把后续数据全部预下载（Mullvad 断网也能继续）：

mkdir -p /root/tools && cd /root/tools
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o gfwlist.txt \
  https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt
mkdir -p /etc/china && cd /etc/china
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o a.txt \
  https://raw.githubusercontent.com/gaoyifan/china-operator-ip/ip-lists/china.txt
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o b.txt \
  https://raw.githubusercontent.com/mayaxcn/china-ip-list/master/chnroute.txt
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o c.txt \
  https://raw.githubusercontent.com/ipverse/country-ip-blocks/master/country/cn/ipv4-aggregated.txt
cat a.txt b.txt c.txt | grep -E '^[0-9.]+/[0-9]+$' | sort -u > /etc/china/china_ip_list.txt
wc -l /etc/china/china_ip_list.txt    # 应 1.2 万+ 条
--resolve IP 失效时：IP=$(dig +short raw.githubusercontent.com @100.64.0.23 | head -1) 后重新带 --resolve 下载。

4. 阶段一：Mullvad 隧道
LuCI 网络→接口→添加 WireGuard，SSH 配置：

uci set network.Mullvad=interface
uci set network.Mullvad.proto='wireguard'
uci set network.Mullvad.private_key='[你的私钥]'
uci set network.Mullvad.addresses='[你的隧道地址/32]'
uci set network.Mullvad.route_allowed_ips='1'
uci set network.@wireguard_Mullvad[0]=wireguard_Mullvad
uci set network.@wireguard_Mullvad[0].public_key='[服务器公钥]'
uci set network.@wireguard_Mullvad[0].endpoint_host='[Mullvad服务器IP]'
uci set network.@wireguard_Mullvad[0].endpoint_port='[冷门端口，见第7节]'
uci set network.@wireguard_Mullvad[0].allowed_ips='0.0.0.0/0'
uci commit network
ifup Mullvad
防火墙（不加 Mullvad 会无 NAT → 局域网全断）：

uci add_list firewall.@zone[1].network='Mullvad'
uci commit firewall
/etc/init.d/firewall restart
nft list ruleset | grep masquerade   # 应有两条
验证：wg show | grep handshake（"几秒前"=通）、ping -c3 1.1.1.1。

5. 阶段二：DNS 反向分流
uci set dhcp.@dnsmasq[0].noresolv='1'
uci set dhcp.@dnsmasq[0].localservice='0'
uci set dhcp.@dnsmasq[0].nonwildcard='0'
uci set dhcp.@dnsmasq[0].filter_aaaa='1'      # 见IPv6说明
uci set dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'
uci delete dhcp.@dnsmasq[0].server            # 强烈建议 delete+add_list！
uci add_list dhcp.@dnsmasq[0].server='223.5.5.5'
uci add_list dhcp.@dnsmasq[0].server='119.29.29.29'
uci commit dhcp
IPv6 说明：Mullvad 支持 IPv6（生成密钥时可选）。filter_aaaa='1' 是主动禁用，因 IPv6 地址可追踪，不符合此方案的外网访问绝对隐私性，故禁用，各位可在生成密钥时按需勾选。

沙箱处理（OpenWrt 25.x 默认把进程关进独立 netns，dnsmasq 看不到隧道）：

sed -i 's/procd_add_jail dnsmasq ubus log/procd_add_jail dnsmasq ubus log network/' /etc/init.d/dnsmasq
/etc/init.d/dnsmasq stop; sleep 1; /etc/init.d/dnsmasq start   # 强烈建议 stop+start！
gfwlist 生成（-p 53 是关键，默认 5353 无人监听）：

sh /root/tools/gfwlist2dnsmasq.sh -o /etc/dnsmasq.d/gfwlist.conf -d 100.64.0.23 -p 53
工具下载：curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -o /root/tools/gfwlist2dnsmasq.sh ...；离线时改脚本 BASE_URL='file:///root/tools/gfwlist.txt'。

验证：nslookup www.google.com 127.0.0.1（142.x 真实IP）、nslookup www.baidu.com（国内IP）。

6. 阶段三：流量分流（国内直连）
原理：几千条 IP 不能用 ip rule 逐条（线性拖慢→崩），用 nft 集合（区间树，2 万条微秒级）；用独立表 inet chnroute（不碰 fw4，坏只影响分流、兜底走隧道）。

/etc/hotplug.d/iface/90-chnroute.sh：

#!/bin/sh
[ "$ACTION" = "ifup" ] && [ "$INTERFACE" = "wan" ] || exit 0
[ -s /etc/china/china_ip_list.txt ] || exit 1
i=0; while [ $i -lt 20 ] && ! ip route | grep -q "default.*pppoe-wan"; do sleep 1; i=$((i+1)); done
WAN_GW=$(ip route | awk '/default.*pppoe-wan/{print $3; exit}')
[ -n "$WAN_GW" ] || exit 1
ip route replace default via "$WAN_GW" dev pppoe-wan table 100
nft list table inet chnroute >/dev/null 2>&1 || nft add table inet chnroute
nft list set inet chnroute cn_ipv4 >/dev/null 2>&1 || nft add set inet chnroute cn_ipv4 { type ipv4_addr\; flags interval\; auto-merge\; }
nft flush set inet chnroute cn_ipv4
awk '{printf "add element inet chnroute cn_ipv4 { %s }\n", $1}' /etc/china/china_ip_list.txt | nft -f -
nft delete chain inet chnroute prerouting 2>/dev/null
nft add chain inet chnroute prerouting { type filter hook prerouting priority mangle\; policy accept\; }
nft add rule inet chnroute prerouting ip daddr @cn_ipv4 meta mark set 0x100
ip rule del pref 100 2>/dev/null
ip rule add fwmark 0x100 table 100 pref 100
ip route flush cache
手动测试带 ACTION=ifup INTERFACE=wan
强烈建议禁用 uidrange 0-0 lookup（会劫持 root→SSH 断）
/root/chnroute-update.sh（三源合并+防空）：

#!/bin/sh
mkdir -p /tmp/chnroute && cd /tmp/chnroute
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o a.txt https://raw.githubusercontent.com/gaoyifan/china-operator-ip/ip-lists/china.txt
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o b.txt https://raw.githubusercontent.com/mayaxcn/china-ip-list/master/chnroute.txt
curl -4 --resolve raw.githubusercontent.com:443:185.199.110.133 -s -o c.txt https://raw.githubusercontent.com/ipverse/country-ip-blocks/master/country/cn/ipv4-aggregated.txt
ALL=$(cat a.txt b.txt c.txt 2>/dev/null | grep -cE '^[0-9.]+/[0-9]+$')
[ "$ALL" -gt 5000 ] || { logger "chnroute: download failed, skip"; exit 1; }
cat a.txt b.txt c.txt 2>/dev/null | grep -E '^[0-9.]+/[0-9]+$' | sort -u > /tmp/chnroute.new
[ "$(wc -l < /tmp/chnroute.new)" -gt 5000 ] || exit 1
cp /tmp/chnroute.new /etc/china/china_ip_list.txt
nft list table inet chnroute >/dev/null 2>&1 || nft add table inet chnroute
nft list set inet chnroute cn_ipv4 >/dev/null 2>&1 || nft add set inet chnroute cn_ipv4 { type ipv4_addr\; flags interval\; auto-merge\; }
nft flush set inet chnroute cn_ipv4
awk '{printf "add element inet chnroute cn_ipv4 { %s }\n", $1}' /etc/china/china_ip_list.txt | nft -f -
ip route flush cache
三重保险：总数<5000 不更新；临时文件验证后替换；灌入失败兜底走隧道。

执行 + 自动更新：

chmod +x /etc/hotplug.d/iface/90-chnroute.sh /root/chnroute-update.sh
sh /root/chnroute-update.sh
ACTION=ifup INTERFACE=wan sh /etc/hotplug.d/iface/90-chnroute.sh
echo "0 3 * * 1 /root/chnroute-update.sh" > /etc/crontabs/root
/etc/init.d/cron enable && /etc/init.d/cron start
验证（用电脑测，本机不走分流钩子测不准）：

ip rule | grep 100
curl.exe -sI --connect-timeout 5 https://www.bilibili.com -o NUL -w "b站:%{http_code} 耗时:%{time_total}"
7. 端口选择
Mullvad 官方支持 UDP 端口：53、853、443、51820 + 4000–33433 自定义范围。
避开：53/80/443/853/51820/7777（常用/特殊数字服务端口，GFW 重点监控）。
推荐：4000–33433 内冷门随机数（如 17453、29134）。
⚠️ 端口需要是 Mullvad 服务器监听的（官方范围内），不在范围的（如 797）会连不上。

自动测试端口：

for p in 4615 6817 8349 31806 17453 29134; do
  uci set network.@wireguard_Mullvad[0].endpoint_port="$p"
  uci commit network
  ifdown Mullvad; sleep 2; ifup Mullvad; sleep 4
  echo "=== 端口 $p ==="; wg show | grep -E "handshake|transfer"
done
被封特征：握手停滞"几分钟前"、transfer 只发不收（148B sent/0B received）、ICMP 通。

8. 手动例外（两边都命中不了时）
场景：某网站既不在国内 IP 表，又不在 gfwlist。

先诊断：

nslookup 域名 127.0.0.1
nft get element inet chnroute cn_ipv4 { 该IP }
grep "域名" /etc/dnsmasq.d/gfwlist.conf
A. 流量强制走隧道（已核实 wg 接口支持 dev 路由）：

ip route add 1.2.3.4/32 dev Mullvad
uci add network route; uci set network.@route[-1].interface='Mullvad'; uci set network.@route[-1].target='1.2.3.4'; uci set network.@route[-1].netmask='255.255.255.255'; uci commit network
B. 流量强制走直连：

ip route add 1.2.3.4/32 via 10.21.0.1 dev pppoe-wan
uci add network route; uci set network.@route[-1].interface='wan'; uci set network.@route[-1].target='1.2.3.4'; uci set network.@route[-1].netmask='255.255.255.255'; uci set network.@route[-1].gateway='10.21.0.1'; uci commit network
⚠️ netmask 强烈建议写 255.255.255.255（不写默认 /24）

C. DNS 指定上游（域名级）：

echo "server=/example.com/100.64.0.23#53" >> /etc/dnsmasq.d/manual.conf
echo "server=/example.com/223.5.5.5" >> /etc/dnsmasq.d/manual.conf
/etc/init.d/dnsmasq restart
D. hosts 直写 IP：

echo "1.2.3.4 example.com" >> /etc/hosts
/etc/init.d/dnsmasq restart
9. 排查思路
分层定位：ping 223.5.5.5（路由层）vs nslookup 域名（DNS 层）
对照实验：本机 vs 电脑（转发差异）；shell vs 进程（沙箱差异）；直连 vs 隧道（路径差异）
抓包：tcpdump -i any -nn "host 100.64.0.23" -c 20
查实际配置：/var/etc/dnsmasq.conf.cfg01411c（uci 值 ≠ 实际加载）
语法检查：dnsmasq --test、sh -x 脚本
netns 对比：ls -l /proc/<pid>/ns/ vs readlink /proc/self/ns/net
10. OpenWrt 命令环境速查
① 工具能力速查：

命令	缺什么/坑	正确替代
wget(uclient)	缺 TLS 库，https 报 Operation not permitted	curl
nslookup(busybox)	不支持端口语法	dig
base64	busybox 默认未启用	apk add coreutils-base64
crontab -l	不支持（报 can't open 'root'）	cat /etc/crontabs/root
ip rule	fwmark 语法需简写	ip rule add fwmark 0x100 table 100 pref 100
PowerShell curl	是 Invoke-WebRequest 别名	curl.exe
② "无返回" = 成功还是失败：

情况	含义
ip route add/uci commit/nft add 无输出	成功（静默）
sh -x 执行无任何行	失败：首行条件未满足直接退出（如 $ACTION 未设置）
ip rule | grep 100 无结果	失败：规则没加上
ps | grep crond 无进程	失败：服务没启动
nslookup Connection refused	53 没人监听
nslookup REFUSED	dnsmasq 在跑但没上游（server 配置丢失）
通用判断：命令; echo "exit=$?"（0=成功，非0=失败）	
③ 先核验再调用四步法：

which curl dig tcpdump nc base64
/etc/init.d/dnsmasq status; ps | grep dnsmasq
uci show dhcp.@dnsmasq[0].server; grep "^server" /var/etc/dnsmasq.conf.cfg01411c
dnsmasq --test; sh -x 脚本 2>&1 | tail -20
11. 完整踩坑清单
11.1 uci set 改 server → 全部 REFUSED
症状：所有域名 REFUSED，像断网但路由正常
原因：server 是列表型配置，set 变单值脚本遍历不到

uci delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server='223.5.5.5'
uci add_list dhcp.@dnsmasq[0].server='119.29.29.29'
uci commit dhcp
/etc/init.d/dnsmasq stop; sleep 1; /etc/init.d/dnsmasq start
11.2 gfwlist 默认端口 → 超时
症状：被墙域名 connection timed out
原因：工具默认 -p 5353，Mullvad DNS 监听 53

sh /root/tools/gfwlist2dnsmasq.sh -o /etc/dnsmasq.d/gfwlist.conf -d 100.64.0.23 -p 53
11.3 ujail 沙箱 → 进程走不了隧道
症状：dnsmasq 查隧道 DNS 失败（shell 手动查却通）

sed -i 's/procd_add_jail dnsmasq ubus log/procd_add_jail dnsmasq ubus log network/' /etc/init.d/dnsmasq
/etc/init.d/dnsmasq stop; sleep 1; /etc/init.d/dnsmasq start
11.4 localservice=1 → 53 不监听
症状：nslookup Connection refused，但进程活着（DHCP 正常）

uci set dhcp.@dnsmasq[0].localservice='0'
uci commit dhcp
/etc/init.d/dnsmasq restart
11.5 IPv6 记录 → 国外超时
原因：IPv6 流量不走隧道（主动禁用防追踪）

uci set dhcp.@dnsmasq[0].filter_aaaa='1'
uci commit dhcp
/etc/init.d/dnsmasq restart
11.6 uidrange 劫持 → SSH 全断
原因：root 流量被劫持进空路由表

ip rule del pref 200 2>/dev/null
ip rule del pref 100 2>/dev/null
ip route flush table 100
ip route flush cache
11.7 fw4 include 挂表 → nft 全空断网
原因：include 引用不存在的集合 → fw4 整体加载失败

rm -f /usr/share/nftables.d/ruleset-post/10-chnroute-mark.nft
/etc/init.d/firewall restart
11.8 Mullvad 端口封 → 握手停滞
症状：握手停"几分钟前"、transfer 只发不收

for p in 53 853 443 51820 17453 29134; do
  uci set network.@wireguard_Mullvad[0].endpoint_port="$p"
  uci commit network; ifdown Mullvad; sleep 2; ifup Mullvad; sleep 4
  echo "=== $p ==="; wg show | grep -E "handshake|transfer"
done
11.9 hotplug 手动执行 → "没生效"
原因：首行检查 $ACTION=ifup，手动执行时为空直接退出

ACTION=ifup INTERFACE=wan DEVICE=pppoe-wan sh /etc/hotplug.d/iface/90-chnroute.sh
11.10 路由器本机测速 → 看着没分流
原因：本机流量走 OUTPUT 不走 prerouting 分流钩子。用电脑测：

curl.exe -sI --connect-timeout 5 https://www.bilibili.com -o NUL -w "b站:%{http_code} 耗时:%{time_total}"
11.11 uci route 不写 netmask → 变成 /24
症状：路由覆盖过大（1.2.3.4 变成 1.2.3.0/24）

uci set network.@route[-1].netmask='255.255.255.255'
12. 回滚（恢复全隧道）
nft delete table inet chnroute
ip rule del pref 100
rm /etc/hotplug.d/iface/90-chnroute.sh /etc/dnsmasq.d/gfwlist.conf
/etc/init.d/dnsmasq restart
13. 最终架构（呼应开头）
DNS: 默认→223.5.5.5/119.29.29.29(国内) + gfwlist→100.64.0.23:53(隧道加密)
流量: 国内IP(nft集合1.2万条) →fwmark 0x100→ table100 → pppoe-wan 直连
      其他 → 默认路由 → Mullvad(冷门UDP端口)
维护: cron 每周一03:00 三源合并自动更新
实战验证：GL.iNet MT6000 / OpenWrt 25.12.5 /  2026-08-12


作者：@Godspeeeeeeed_