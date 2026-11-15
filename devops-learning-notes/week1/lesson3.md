# lesson3·网络排障

按「本机 → 网关 → 外网 → DNS → 端口 → HTTP → 抓包」往外查，每一步只回答一个问题。

## 1、本机网络
```bash
ip -br addr
ip route
ip -s link show eth0
```
输出（`ip route`）：
```text
default via 192.168.124.1 dev eth0 proto static
192.168.124.0/24 dev eth0 proto kernel scope link src 192.168.124.20
```
`ip -br addr`：确认接口为 UP、地址是否为 `192.168.124.20/24`。`ip route`：默认路由是否指向 `192.168.124.1`。`ip -s link show eth0`：RX/TX 的 errors、dropped 是否为 0；若持续增长，可能是网卡或虚拟交换机问题。

## 2、网关和外网
```bash
ping -c 4 192.168.124.1
ping -c 4 8.8.8.8
ping -c 4 223.5.5.5
```
输出（`ping 192.168.124.1`，节选）：
```text
64 bytes from 192.168.124.1: icmp_seq=1 ttl=128 time=3.64 ms
...
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 2.755/4.844/8.140/2.358 ms
```
输出（`ping 8.8.8.8`，节选）：
```text
64 bytes from 8.8.8.8: icmp_seq=1 ttl=106 time=183 ms
...
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 182.268/182.741/182.993/0.335 ms
```
输出（`ping 223.5.5.5.`，错误示例）：
```text
ping: 223.5.5.5.: Name or service not known
```
输出（`ping 223.5.5.5`，节选）：
```text
64 bytes from 223.5.5.5: icmp_seq=1 ttl=51 time=18.5 ms
...
rtt min/avg/max/mdev = 17.605/18.010/18.458/0.349 ms
```

| 目标 | 丢包 | 平均延迟 | TTL | 说明 |
|---|---|---|---|---|
| 192.168.124.1 | 0% | 4.84 ms | 128 | 同网段，链路正常 |
| 8.8.8.8 | 0% | 182.7 ms | 106 | 能通，跨境链路，延迟高 |
| 223.5.5.5 | 0% | 18.0 ms | 51 | 阿里 DNS，国内节点，延迟低 |


## 3、DNS
```bash
dig baidu.com +short
dig @8.8.8.8 baidu.com +short
resolvectl status
```

`+short` 下不同 DNS 返回的 IP 顺序不一样，这是多条 A 记录轮换，顺序本身没有含义。

## 4、端口
```bash
sudo ss -tulpn
sudo ss -tan
```
`ss -tulpn` 看谁在监听哪个端口。`ss -tan` 看 TCP 连接状态：`SYN-SENT` 多通常是对端没回应；`CLOSE-WAIT` 多通常是本机程序没把连接关掉。

## 5、HTTP：正常和拒绝
服务在跑时：
```bash
curl -v http://127.0.0.1:8000
```
关键输出（正常）：
```text
* Connected to 127.0.0.1 (127.0.0.1) port 8000
...
> GET / HTTP/1.1
...
< HTTP/1.0 200 OK
hello from systemd lab
```
停掉服务再访问：
```bash
sudo systemctl stop hello
curl -v http://127.0.0.1:8000
sudo systemctl start hello
```
关键输出（拒绝）：
```text
* connect to 127.0.0.1 port 8000 from 127.0.0.1 port xxxxx failed: Connection refused
curl: (7) Failed to connect to 127.0.0.1 port 8000 after 0 ms: Could not connect to server
```
正常时顺序是连上 TCP → 发 `GET` → `200 OK` → 正文。停服务后内核立刻回 RST，所以是 `Connection refused` 且耗时约 0 ms。`Connection refused`（没人监听）和 `Connection timed out`（常见防火墙 DROP）要分开记。

## 6、抓三次握手
```bash
sudo tcpdump -i lo -nn port 8000
```
另开终端执行 `curl http://127.0.0.1:8000`。前两次抓到 0 个包，是因为抓包期间没有发请求；tcpdump 只记录它运行时的流量。

第三次关键输出（节选）：
```text
IP 127.0.0.1.xxxxx > 127.0.0.1.8000: Flags [S], ...
IP 127.0.0.1.8000 > 127.0.0.1.xxxxx: Flags [S.], ...
IP 127.0.0.1.xxxxx > 127.0.0.1.8000: Flags [.], ...
IP 127.0.0.1.xxxxx > 127.0.0.1.8000: Flags [P.], seq 1:79, ...
...
```

| 标志 | 含义 |
|---|---|
| `[S]` | 客户端 SYN |
| `[S.]` | 服务端 SYN-ACK |
| `[.]` | 客户端 ACK，三次握手完成 |
| `[P.]` 78 字节 | HTTP 请求 |
| `[P.]` 92 字节 | 响应头 |
| `[P.]` 23 字节 | 响应体 `hello from systemd lab\n` |
| `[F.]` | 四次挥手，双方关闭连接 |

`-i lo` 只盯回环口。`-nn` 不解析主机名和端口名，输出更直观。