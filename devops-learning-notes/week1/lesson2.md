# lesson2·systemd 托管服务

## 1、写一个只监听本机的 HTTP 服务
```bash
mkdir -p ~/lab/hello && cd ~/lab/hello
```
`server.py` 只绑 `127.0.0.1:8000`，外面访问不到：
```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"hello from systemd lab\n")

HTTPServer(("127.0.0.1", 8000), Handler).serve_forever()
```
先手动跑起来，另开终端 `curl http://127.0.0.1:8000`，能返回内容后再 `Ctrl+C`。

## 2、写成 systemd 单元
```bash
sudo tee /etc/systemd/system/hello.service > /dev/null <<'EOF'
[Unit]
Description=Hello lab service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/zhongj/lab/hello/server.py
Restart=on-failure
User=zhongj

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now hello
systemctl status hello
```
改过 unit 文件之后必须 `daemon-reload`，否则 systemd 还在用旧配置。

## 3、故意写错路径，看退出码
把 `ExecStart` 里的文件名改成不存在的 `erver.py`，然后：
```bash
sudo systemctl daemon-reload
sudo systemctl restart hello
systemctl status hello
sudo journalctl -u hello --no-pager | tail -n 20
```
这次的证据：
```text
Process: 4325 ExecStart=/usr/bin/python3 /home/zhongj/lab/hello/erver.py (code=exited, status=2)
can't open file '/home/zhongj/lab/hello/erver.py': [Errno 2] No such file or directory
```
`status=2` 就是退出码。Python 找不到脚本时以 2 退出。systemd 没有坏，它只是按 `ExecStart` 去执行。

`Restart=on-failure` 会不停拉起失败的进程，所以 journal 里同一条错误会连着出现好几次。改配置前先停掉：
```bash
sudo systemctl stop hello
sudo systemctl reset-failed hello
```

journal 提示当前用户不在 `adm` / `systemd-journal` 组，只能看到自己的日志。看这个单元的完整日志用 `sudo journalctl -u hello`。

常见退出码：

| 退出码 | 含义 |
|---|---|
| `status=2`，文件不存在 | 脚本路径写错，Python 找不到文件 |
| `status=1`，带 Traceback | 代码跑起来后抛异常，例如端口已被占用 |
| `status=127` | 命令不存在 |
| `203/EXEC` | systemd 执行不了 `ExecStart`，例如没权限或程序路径不对 |


## 4、改回正确路径并确认
```bash
sudo systemctl daemon-reload
sudo systemctl start hello
systemctl status hello          # active (running)
curl http://127.0.0.1:8000      # hello from systemd lab
```

## 5、复盘
- **现象**：`hello.service` 为 `failed`，`Result: exit-code`，退出码 2。
- **证据**：journal 里 `can't open file '.../erver.py'`。
- **根因**：unit 里路径少写了一个 `s`，这是故意制造的。
- **修复**：改回 `server.py`，`daemon-reload` 后再启动。
