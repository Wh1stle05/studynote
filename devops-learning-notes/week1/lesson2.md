# lesson2·systemd 托管服务

## 1、写一个只监听本机的 HTTP 服务
```bash
mkdir -p ~/lab/hello && cd ~/lab/hello
python3 server.py
```
另开终端：
```bash
curl http://127.0.0.1:8000
```
输出：
```text
hello from systemd lab
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
`python3 server.py` 在前台占用终端。`curl` 能返回正文，说明脚本本身没问题，再 `Ctrl+C` 停掉手动进程。

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
`tee` 把单元文件写到 `/etc/systemd/system/`。`enable --now` 同时设置开机自启并立刻启动。`systemctl status` 里应看到 `active (running)`。

改过 unit 文件之后必须 `daemon-reload`，否则 systemd 还在用旧配置。

## 3、故意写错路径，看退出码
把 `ExecStart` 里的文件名改成不存在的 `erver.py`，然后：
```bash
sudo systemctl daemon-reload
sudo systemctl restart hello
systemctl status hello
sudo journalctl -u hello --no-pager | tail -n 20
```
输出（`systemctl status hello`）：
```text
× hello.service - Hello lab service
     Loaded: loaded (/etc/systemd/system/hello.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Wed 2026-11-11 11:22:59 UTC; 6s ago
   Duration: 32ms
    Process: 4325 ExecStart=/usr/bin/python3 /home/zhongj/lab/hello/erver.py (code=exited, status=2)
   Main PID: 4325 (code=exited, status=2)
```
输出（`journalctl` 末尾）：
```text
Hint: You are currently not seeing messages from other users and the system.
      Users in groups 'adm', 'systemd-journal' can see all messages.
...
Nov 11 11:22:58 ubuntu1 python3[4325]: /usr/bin/python3: can't open file '/home/zhongj/lab/hello/erver.py': [Errno 2] No such file or directory
```
`Process` 行里的 `status=2` 就是退出码。`Result: exit-code` 表示主进程以非零状态结束。journal 里 `can't open file` 说明 Python 找不到脚本

`Restart=on-failure` 会不停拉起失败的进程，所以 journal 里同一条错误会连着出现好几次。改配置前先停掉：
```bash
sudo systemctl stop hello
sudo systemctl reset-failed hello
```

journal 开头的 Hint 表示当前用户不在 `adm` / `systemd-journal` 组，只能看到自己的日志。看这个单元的完整日志用 `sudo journalctl -u hello`。

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
systemctl status hello
curl http://127.0.0.1:8000
```
输出（`curl`）：
```text
hello from systemd lab
```
路径改回 `server.py` 并 `daemon-reload` 后，`status` 应为 `active (running)`，`curl` 与手动运行时一致。

## 5、复盘
- **现象**：`hello.service` 为 `failed`，`Result: exit-code`，退出码 2。
- **证据**：journal 里 `can't open file '.../erver.py'`。
- **根因**：unit 里路径少写了一个 `s`，这是故意制造的。
- **修复**：改回 `server.py`，`daemon-reload` 后再启动。