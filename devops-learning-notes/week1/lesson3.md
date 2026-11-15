# lesson3·网络排障

按「本机 → 网关 → 外网 → DNS → 端口 → HTTP → 抓包」往外查，每一步只回答一个问题。

## 1、本机网络
```bash
ip -br addr          # 接口和地址
ip route             # 默认路由
ip -s link show eth0 # RX/TX 错误和丢包
```
结果：`eth0` 为 UP，地址 `192.168.124.20/24`，默认路由指向 `192.168.124.1`。errors、dropped 都是 0，网卡这一层没有错误计数。

## 2、网关和外网
```bash
ping -c 4 192.168.124.1
ping -c 4 8.8.8.8
ping -c 4 223.5.5.5
```
`ping` 只说明网络层通不通，不能说明 TCP 服务是否正常。

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
sudo ss -tulpn    # 谁在监听
sudo ss -tan      # 当前 TCP 状态
```
`SYN-SENT` 多，通常是对端没回应。`CLOSE-WAIT` 多，通常是本机程序没把连接关掉。

## 5、HTTP：正常和拒绝
服务在跑时：
```bash
curl -v http://127.0.0.1:8000
```
顺序是 `Trying` → 连接建立 → 发出 `GET` → `HTTP/1.0 200 OK` → 响应体 `hello from systemd lab`。

停掉服务再访问：
```bash
sudo systemctl stop hello
curl -v http://127.0.0.1:8000
sudo systemctl start hello
```
报错是 `Connection refused`，`curl: (7)`，耗时约 0 ms。本机发出 SYN 后，内核立刻回 RST，因为这个端口没有进程在听。

`Connection refused` 是对方明确拒绝。`Connection timed out` 是包发出去没有任何回应，常见于防火墙 DROP。这两种要分开。

## 6、抓三次握手
```bash
sudo tcpdump -i lo -nn port 8000
```
抓包开着的时候，另开终端执行 `curl http://127.0.0.1:8000`。前两次抓到 0 个包，是因为抓包期间没有发请求。tcpdump 只记录它运行时经过的包。

第三次对应关系：

| 标志 | 含义 |
|---|---|
| `[S]` | 客户端 SYN |
| `[S.]` | 服务端 SYN-ACK |
| `[.]` | 客户端 ACK，三次握手完成 |
| `[P.]` 78 字节 | HTTP 请求 |
| `[P.]` 92 字节 | 响应头 |
| `[P.]` 23 字节 | 响应体 `hello from systemd lab\n` |
| `[F.]` | 四次挥手，双方关闭连接 |
