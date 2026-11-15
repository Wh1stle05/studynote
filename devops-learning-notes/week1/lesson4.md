# lesson4·Docker 与数据库

## 1、安装并确认 Docker 可用
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```
输出（`hello-world` 节选）：
```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
`apt` 装引擎和 Compose 插件；`systemctl enable --now` 让 dockerd 开机自启并立刻运行。`run --rm` 跑完就删容器。能打出 `Hello from Docker!`，说明客户端、daemon、拉镜像、建容器这条链是通的。

## 2、免 sudo
```bash
sudo usermod -aG docker $USER
```
无终端输出；重新登录后，`docker` 命令可不用 `sudo`。加入 `docker` 组等同于 root：容器可以挂载宿主机任意目录。这台虚拟机只做实验，可以接受。

## 3、端口映射
```bash
docker run -d --name web -p 8080:80 nginx
curl -v http://127.0.0.1:8080
docker ps
docker logs web
```
`-p 8080:80` 是宿主机端口在前、容器端口在后：访问本机 8080，转到容器里的 80。`docker ps` 里 `PORTS` 若写成 `0.0.0.0:8080->80/tcp`，表示所有宿主机地址的 8080 都映射进去；未写绑定地址时默认 `0.0.0.0`。`docker logs web` 能看到访问记录，排障时先看它。

实验完清理：
```bash
docker stop web && docker rm web
```
`stop` 停进程，`rm` 删容器。这个容器没有挂数据卷，删掉后里面的改动也没了。

## 4、用 Compose 起 PostgreSQL
`~/lab/compose/docker-compose.yml`：
```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: labpass
      POSTGRES_DB: labdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

volumes:
  pgdata:
```
```bash
cd ~/lab/compose
docker compose up -d
docker compose ps
```
输出（`docker compose ps`）：
```text
NAME           IMAGE         COMMAND                  SERVICE   CREATED          STATUS          PORTS
compose-db-1   postgres:16   "docker-entrypoint.s…"   db        51 seconds ago   Up 49 seconds   127.0.0.1:5432->5432/tcp
```
`127.0.0.1:5432:5432` 只绑本机；`127.0.0.1:5432->5432/tcp` 表示只有本机能连这个端口。

写入一行数据：
```bash
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1);"
```
输出：
```text
CREATE TABLE
INSERT 0 1
```
`$(docker compose ps -q db)` 取正在运行的 db 容器 ID。若容器还没起来，替换为空，命令变成 `docker exec -it psql ...`，会报 `No such container: psql`。先 `docker compose ps` 确认 Up 再 exec。

刚 `up` 完立刻连，还可能看到：
```text
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```
Postgres 还在初始化，等几秒再查。

## 5、数据卷会不会留住数据
只删容器、保留卷：
```bash
docker compose down
docker compose up -d
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "SELECT * FROM t;"
```
输出（`down` / `up` 过程节选）：
```text
[+] Running 2/2
 ✔ Container compose-db-1   Removed
 ✔ Network compose_default  Removed
[+] Running 2/2
 ✔ Network compose_default  Created
 ✔ Container compose-db-1   Started
```
输出（`SELECT`）：
```text
 id
----
  1
(1 row)
```
`down` 删容器和网络，命名卷 `pgdata` 还在，新容器挂同一卷，数据仍在。

连卷一起删：
```bash
docker compose down -v
docker compose up -d
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "SELECT * FROM t;"
```
输出（`down -v` / `up` 节选）：
```text
 ✔ Container compose-db-1   Removed
 ✔ Volume compose_pgdata    Removed
 ✔ Network compose_default  Removed
...
 ✔ Volume compose_pgdata    Created
 ✔ Container compose-db-1   Started
```
输出（再查）：
```text
ERROR:  relation "t" does not exist
LINE 1: SELECT * FROM t;
                      ^
```

| 操作 | 数据 | 原因 |
|---|---|---|
| `docker compose down` 再 `up` | 保留 | 容器没了，volume 还在 |
| `docker compose down -v` 再 `up` | 丢失 | volume 被删，数据库重新初始化 |

容器本身不保存数据。要留下来的东西放 volume。删 volume 等于删数据。

## 6、容器和宿主机怎么连上
宿主机访问容器，靠的是端口映射：本机连 `127.0.0.1:5432`，Docker 转到容器里的 5432。

同一个 Compose 项目里的容器走 Docker 网络，用服务名互相访问，不需要把端口公布到宿主机。数据库这次公布到宿主机，只是为了在虚拟机上用客户端连接，并且限制在 `127.0.0.1`。

## 7、顺手确认网关是谁
```bash
ip neigh show 192.168.124.1
```
输出：
```text
192.168.124.1 dev eth0 lladdr 5c:a7:21:c4:a3:dd STALE
```
MAC 前缀 `5c:a7:21` 常见于 Hyper-V 虚拟交换机。结合 lesson3 ping 网关 TTL=128，可判断 `192.168.124.1` 是笔记本上的 Hyper-V 虚拟交换机，不是家用路由器。