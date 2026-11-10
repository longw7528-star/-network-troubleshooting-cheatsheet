# 网络排障速查表

代理、DNS、虚拟网卡、延迟测试的排查命令与判断依据。

## 1. 代理配了不生效

    curl -s https://api.ipify.org
    curl -v --max-time 5 http://127.0.0.1:7890

- 卡在 Trying → 路由或防火墙问题，与代理无关
- 卡在 Connected 之后 → 代理没转发出去
- Connection refused → 没监听这个端口，或只监听 127.0.0.1

## 2. WSL2 里的代理

    grep nameserver /etc/resolv.conf

NAT 模式下宿主 IP 就是 resolv.conf 里的 nameserver；mirrored 模式（Win11 22H2+）直接用 127.0.0.1。改完必须 wsl --shutdown。

## 3. 虚拟机没网

    ip addr
    ip route | grep default

10.211.55.x 是 Parallels 共享网络，网关即宿主。代理客户端要开 Allow LAN，否则虚拟机的请求会被拒。

## 4. DNS 泄露

    sudo tcpdump -i any -n port 53

看两件事：解析请求发给了谁、解析结果的地区是否和出口 IP 匹配。

## 5. 延迟怎么测才准

    curl -o /dev/null -s -w "connect:%{time_connect} ttfb:%{time_starttransfer}\n" https://example.com

ping 测的是 ICMP，真实流量走 TCP，两者不等价。判据：建连超过 300ms 该换节点，抖动超过平均值 2 倍该换节点。

## 6. TUN 模式 vs 系统代理

系统代理只对认系统设置的程序生效；TUN 建虚拟网卡接管全部流量。
判断某个程序走没走：在它内部请求 https://api.ipify.org，对比出口 IP。

## 相关文章

- [WSL2 怎么走代理：NAT 与 mirrored 两种模式](https://ncjsub.com/knowledge/wsl2-network-proxy.html)
- [DNS 泄露怎么自测：三条命令](https://ncjsub.com/knowledge/dns-leak-test.html)
- [节点延迟怎么测才准：ping 为什么不算数](https://ncjsub.com/knowledge/latency-test-method.html)
- [TUN 模式和系统代理的区别](https://ncjsub.com/knowledge/tun-vs-system-proxy.html)
- [Parallels 共享网络下虚拟机连不上代理](https://ncjsub.com/knowledge/parallels-mac-network.html)
