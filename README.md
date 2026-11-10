网络排障速查表（Network Troubleshooting Cheatsheet）
一份给开发者和运维用的速查表：代理、DNS、虚拟网卡、延迟测试这些"知道原理但总记不住命令"的场景。

每条都给出判断依据，不是命令堆砌——重点是"什么现象说明什么问题"。

1. 代理配了但不生效
bash


# 看当前出口 IP
curl -s https://api.ipify.org

# 看环境变量
echo $http_proxy $https_proxy

# 测代理端口通不通（比 ping 有用）
curl -v --max-time 5 http://127.0.0.1:7890
判断

现象	原因
curl 卡在 Trying 1.2.3.4...	路由/防火墙问题，与代理无关
卡在 Connected 之后	代理没转发出去，或 TLS 握手失败
Connection refused	代理没监听这个端口，或只监听 127.0.0.1
返回本地运营商 IP	环境变量没生效
2. WSL2 里的代理
WSL2 是独立虚拟机，127.0.0.1 指它自己。

bash


# NAT 模式：宿主 IP 在 resolv.conf 里
grep nameserver /etc/resolv.conf

export http_proxy=http://$hostip:7890
export https_proxy=http://$hostip:7890
ini


# mirrored 模式（Win11 22H2+）：127.0.0.1 直接可用
# C:\Users\<你>\.wslconfig
[wsl2]
networkingMode=mirrored
改完必须 wsl --shutdown。

不认环境变量的工具：apt、git、npm 各自要单独配。

3. 虚拟机没网（Parallels / VMware）
bash


ip addr                 # 看拿到什么网段
ip route | grep default # 看网关
ping <网关>              # 通不通
curl -v --max-time 5 http://<网关>:7890   # 代理端口开没开
10.211.55.x = Parallels 共享网络，网关是 10.211.55.2（即 Mac）
192.168.x.x = 桥接，走路由器
代理客户端要开 Allow LAN，否则虚拟机的请求会被拒
4. DNS 泄露
bash


cat /etc/resolv.conf
dig +short example.com
sudo tcpdump -i any -n port 53   # 看解析请求发给了谁
两个判断标准

解析请求的目标是不是本地（运营商/路由器）——决定隐私
解析结果的地区是否和出口 IP 匹配——决定能不能访问
5. 延迟怎么测才准
ping 测 ICMP，真实流量走 TCP/UDP，两者不等价。

bash


curl -o /dev/null -s -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer}\n" https://example.com
这三个值才有意义：connect（建连）、ttfb（首字节）、抖动（多次取最大最小值差）。

判据：

建连 > 300ms → 交互会明显迟钝
抖动 > 平均值 2 倍 → 体验不稳定
丢包 > 2% → 视频和语音受影响
6. TUN 模式 vs 系统代理
系统代理	TUN
原理	写系统设置，程序自己读	创建虚拟网卡，接管路由
覆盖	浏览器为主	全部流量
命令行/git/游戏	多数不走	走
风险	低	可能与其他虚拟网卡冲突
判断某个程序走没走：在它内部请求 
https://api.ipify.org
，对比出口 IP。

7. 常见状态码
码	含义	排查方向
301	永久跳转	检查是否重定向链过长
410	永久删除	用于清理旧 URL，比 404 更快出索引
403	禁止访问	目录索引、权限、WAF
444	nginx 断开连接	常用于拒绝未知 Host、扫描器
相关文章
WSL2 怎么走代理：NAT 与 mirrored 两种模式
DNS 泄露怎么自测：三条命令
节点延迟怎么测才准：ping 为什么不算数
TUN 模式和系统代理的区别
Parallels 共享网络下虚拟机连不上代理
