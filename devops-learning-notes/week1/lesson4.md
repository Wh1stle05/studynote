# lesson4·Docker 与数据库

## 1、安装并确认 Docker 可用
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```
本地没有镜像时会先从 Docker Hub 拉取。输出里能看到 `Hello from Docker!`，以及客户端连上 daemon、拉取镜像、创建容器、把输出流回终端这几步，说明安装可用。

## 2、免 sudo
```bash
sudo usermod -aG docker $USER
```
重新登录后生效。加入 `docker` 组等同于 root：容器可以挂载宿主机任意目录。这台虚拟机只做实验，可以接受。

## 3、端口映射
```bash
docker run -d --name web -p 8080:80 nginx
curl -v http://127.0.0.1:8080
docker ps
docker logs web
```
`-p 8080:80` 是宿主机端口在前、容器端口在后：访问本机 8080，转到容器里的 80。

`docker ps` 的 `PORTS` 列写成 `0.0.0.0:8080->80/tcp` 时，表示宿主机所有地址的 8080 都映射进容器 80。没写绑定地址时，默认是 `0.0.0.0`，局域网也能访问。

`docker logs web` 能看到访问记录，排障时先看它。

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
`127.0.0.1:5432:5432` 只绑本机。不需要对外的服务不要绑 `0.0.0.0`。

```bash
cd ~/lab/compose
docker compose up -d
docker compose ps
```
实际状态：
```text
NAME           IMAGE         COMMAND                  SERVICE   STATUS          PORTS
compose-db-1   postgres:16   "docker-entrypoint.s…"   db        Up              127.0.0.1:5432->5432/tcp
```
`127.0.0.1:5432->5432/tcp` 表示只有本机能连这个端口。

写入一行数据：
```bash
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1);"
```
`$(docker compose ps -q db)` 取出正在运行的 db 容器 ID。容器还没起来时这个替换是空的，整条命令会变成 `docker exec -it psql ...`，于是报 `No such container: psql`。先 `docker compose ps` 确认是 Up，再 exec。

刚 `up` 完立刻连，还可能看到：
```text
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```
这是 Postgres 还在初始化，socket 还没建好。等几秒再查。

## 5、数据卷会不会留住数据
只删容器、保留卷：
```bash
docker compose down
docker compose up -d
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "SELECT * FROM t;"
```
结果仍是 `id = 1`。`down` 删的是容器和网络，`pgdata` 还在，新容器挂上同一卷，数据还在。

连卷一起删：
```bash
docker compose down -v
docker compose up -d
docker exec -it $(docker compose ps -q db) psql -U postgres -d labdb -c "SELECT * FROM t;"
```
`down -v` 的输出里有 `Volume compose_pgdata Removed`，随后又 `Volume compose_pgdata Created`。再查：
```text
ERROR:  relation "t" does not exist
```
卷被删掉后，Postgres 重新初始化数据目录，表和数据都没了。

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
```text
192.168.124.1 dev eth0 lladdr 5c:a7:21:c4:a3:dd STALE
```
MAC 前缀 `5c:a7:21` 是 Hyper-V 虚拟交换机常用网段。再加上第三天 ping 到的 TTL=128，`192.168.124.1` 是笔记本上的 Hyper-V 虚拟交换机，不是路由器。虚拟机的流量先到笔记本的虚拟交换机，再由笔记本转出去。
