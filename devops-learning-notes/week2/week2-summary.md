# Week 2 总结：CPU / 内存 / IO 排障

## 本周目标

拿到「机器很慢」或「负载很高」的现象后，能按 **系统层指标 → CPU → 内存 → IO → 进程** 的顺序收集证据、缩小范围，并说出每类指标代表什么、不能代表什么。

## 实验记录

| 课时 | 主题 | 结论 |
|---|---|---|
| lesson5 | 进程、CPU 与 Load | CPU 压测时 `us≈100%`、`load` 上升；区分 load 高来自算力忙还是 IO 等待 |
| lesson6 | 内存与 Swap | page cache 让 `used` 高但 `available` 仍充足；`stress-ng --vm` 后出现 Swap、`si/so`、`wa` 升高 |
| lesson7 | 空间、Inode、磁盘 IO | `df -h` / `df -i` 各看一类耗尽；`/tmp` 在 tmpfs 上；`%util` 要结合 `await` 判断 |
| lesson8 | 综合排障 | 同时打满 CPU + 内存，`top`/`vmstat`/`iostat`/`ps` 串起来定位到 `stress-ng` |

## 关键结论

**Load Average**

- 含义是「正在跑 + 在等 CPU 的进程」数的滑动平均（1/5/15 分钟），不是 CPU 使用率。
- `load` 明显高于 CPU 核数时，要看是 **算力不够**（`us`/`sy` 高、`id` 低）还是 **在等 IO**（`wa` 高、进程 `D` 状态多）。
- lesson5 对比：纯 CPU 压测时 `load≈1.12`（6 核机）、`us≈98.5%`；IO 压测在本机环境下 `wa` 不明显，但理论上 load 也可由 IO 等待推高。

**CPU**

- 看 `top` 的 `%Cpu(s)` 或 `mpstat`：`us`（用户态）、`sy`（内核态）、`id`（空闲）、`wa`（等 IO）、`st`（被虚拟机偷走）。
- CPU 饱和：多个进程 `R` 状态、`us` 接近 100%、`idle` 接近 0（lesson5、lesson8 的 `stress-ng --cpu`）。

**内存**

- `free -h` 里 **不要只看 `used`**，要看 **`available`** 和 **Swap 使用量**。
- `buff/cache` 是可回收的 page cache，占满不等于 OOM（lesson6：读大文件后 `used` 升高，`available` 仍约 2.8Gi，Swap 为 0）。
- 真正压力：`available` 很低 + Swap `used` 增加 + `vmstat` 的 `si`/`so` 非 0 + 进程 `D` 或系统变卡 + `wa` 升高（lesson6：`swpd` 到百万级 kB，`so` 持续很大）。

**磁盘与 IO**

- 空间：`df -h`；Inode：`df -i`。创建失败可能是 **空间满、Inode 满、配额**（lesson7：`dd` 到 `/tmp` 触发 quota，和根分区 51% 不是一回事）。
- `/tmp` 在本机是 **tmpfs**，写它主要占 **内存**，不等于在压真实磁盘（lesson7 小结）。
- 是否磁盘慢：`iostat -x` 看 **`%util` 高且 `await`（`r_await`/`w_await`）明显升高**；单看 `%iowait` 不够准确。

**综合排障顺序（lesson8 验证）**

1. `uptime` + `free -h` + `df -h /`：整体负载、内存余量、根分区空间。
2. CPU：`top` / `mpstat`。
3. 内存：`vmstat 1`（尤其 `si`/`so`/`free`）。
4. IO：`iostat -x 1`。
5. 进程：`ps aux --sort=-%cpu` 与 `-%mem`，对上可疑 PID 再 `kill` 或限流。

本次综合场景结论：**CPU 饱和（根因）+ 轻度内存压力（Swap 已用 137Mi）+ IO 不是瓶颈（`%util` 低、`wa≈0`）**。

## 指标对照表（排障时快速扫一眼）

| 现象 | 优先怀疑 | 建议再看 |
|---|---|---|
| load 高，`us` 高，`id`≈0 | CPU 忙 | `top`/`ps` 找高 `%CPU` 进程 |
| load 高，`us` 不高，`wa` 高 | IO 等待 | `iostat -x`、`iotop`、`pidstat -d` |
| `used` 很高，Swap=0，`available` 还行 | 多半是 cache | `MemAvailable`，不必急着加内存 |
| `available` 很低，Swap 涨，`si/so`>0 | 内存压力 / 换页 | `ps` 按内存排序，查泄漏或大对象 |
| 写文件失败，`df -h` 还有空间 | Inode / 配额 / 挂载点 | `df -i`、日志、`quota` |
| 磁盘「忙」 | 真实 IO 瓶颈 | `%util` + `await`，别只看 `iowait` |

## 综合命令清单

系统全景：

```bash
uptime
free -h
df -h /
df -i /
```

CPU：

```bash
top -bn1 | head -20
mpstat 1 3
nproc
cat /proc/loadavg
```

内存：

```bash
vmstat 1 5
cat /proc/meminfo | head -20
```

IO：

```bash
iostat -x 1 3
lsblk
```

进程：

```bash
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
```

lesson8 故障时典型片段（便于复习对照）：

```text
load average: 5.89, 2.20, 0.85          # 6 核机，负载明显高于核数
available 317Mi，Swap used 137Mi          # 内存紧，已开始用 Swap
%Cpu(s): 100.0 us, 0.0 wa, 0.0 id         # CPU 打满，不是在等磁盘
iostat: %util 很低                        # IO 不是本次瓶颈
stress-ng-cpu / stress-ng-vm 占满列表     # 进程层对上根因
```

## 面试三句话

1. **Load 高但 CPU 不高？** 先看 `wa` 和不可中断的 `D` 状态进程，再用 `iostat`/`iotop` 找磁盘或具体进程读写。
2. **内存占用 90% 要加内存吗？** 先看 `available` 和 Swap；缓存多、Swap 几乎不用通常不用立刻加；`available` 低、Swap 持续增长、`si/so` 活跃或 OOM 才要分析进程再决定优化或扩容。
3. **磁盘使用率 95% 要立刻处理吗？** 同时看 `df -i`、IO 延迟和是否已写失败；空间高但 `await` 正常可先规划清理；Inode 满或 IO 延迟飙高要优先处理。

## 环境与局限

- 实验机在 **Hyper-V 虚拟机**（约 6 CPU / 5.3Gi 内存），`st`、IO 表现和物理机、云盘会有差别。
- lesson7 的 fio 写在 `/tmp`（tmpfs）上，**压测 IO 在磁盘上不明显**；要看真实盘延迟，应对 **持久化挂载点**（如 `/` 对应块设备）做压测。
- lesson7 大文件写 `/tmp` 曾触发 **Disk quota exceeded**，说明「有空间」和「能写成功」中间还可能有配额层。

