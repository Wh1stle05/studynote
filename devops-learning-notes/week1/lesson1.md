# lesson1·服务器初始化
## 1、记录服务器基线
```bash
lsb_release -a      #查询 Linux 发行版信息，如发行版名称、版本号、代号。
uname -r            #显示当前运行的 内核版本。
df -h               #查看各文件系统的 磁盘空间使用情况，-h 表示人类可读单位（GB/MB）。
free -h             #查看 内存和 swap 使用情况，-h 人类可读。
systemctl --failed  #列出 启动失败的 systemd 服务/单元。
ss -tulpn           #查看当前 网络监听端口及关联进程。
ip -br addr         #以简洁格式显示 网络接口和 IP 地址。
ip route            #显示 路由表，包括默认网关、网段路由、出口网卡等。
```
## 2、创建管理员用户
```bash
sudo adduser <你的用户名>               #创建用户名
sudo usermod -aG sudo <你的用户名>      #赋予用户sudo权限
```
## 3、生成 SSH 密钥并推送到服务器
```bash
ssh-keygen -t ed25519                                            #生成密钥
ssh-copy-id -i ~/.ssh/id_ed25519.pub <你的用户名>@<服务器内网IP>    #将公钥复制到服务器上
```
然后**开一个新终端**测试登录：
```bash
ssh <你的用户名>@<服务器内网IP>
```
能登录后，再进行下一步。
## 4、关闭密码登录和 root 登录
在服务器上创建独立配置文件（比直接改主配置更安全，也方便回滚）：
```bash
sudo vim /etc/ssh/sshd_config.d/99-hardening.conf
```
写入：
```
PasswordAuthentication no 
PermitRootLogin no
```
检查语法并重载：
```bash
sudo sshd -t
sudo systemctl reload ssh
```
**再开一个新终端**确认密钥登录仍然可用。确认成功后再关闭旧会话。

## 5、配置防火墙（先放行 SSH，再启用）
```bash
sudo ufw default deny incoming              #设置防火墙默认策略：默认拒绝所有入站连接。
sudo ufw default allow outgoing             #
sudo ufw allow from <你的内网网段，如 192.168.1.0/24> to any port 22 proto tcp
sudo ufw enable
sudo ufw status verbose
```
**顺序很重要**：如果先 `enable` 再放行 22，就可能把自己锁在外面。如果担心，可以在有显示器或物理接触的情况下操作。

## 6、自动安全更新
```bash
sudo apt update
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```
