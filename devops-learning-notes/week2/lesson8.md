# lesson8 综合排障练习
## 1. 制造环境故障
```bash
# 终端1：打满 CPU
stress-ng --cpu $(nproc) --timeout 180s &

# 终端2：吃掉大部分内存并开始 swap
stress-ng --vm 1 --vm-bytes 4G --timeout 180s --vm-keep &
```
## 2. 观察系统状态
1. 整体印象
```bash
uptime
free -h
df -h /
```
状态输出：
```text
 05:15:19 up 22:02,  4 users,  load average: 5.89, 2.20, 0.85
               total        used        free      shared  buff/cache   available
Mem:           5.3Gi       5.0Gi       204Mi       434Mi       777Mi       317Mi
Swap:          6.3Gi       137Mi       6.2Gi
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   23G   12G   11G  51% /
```
2. CPU 层
```bash
top -bn1 | head -15
mpstat 1 3 2>/dev/null || (sudo apt install -y sysstat && mpstat 1 3)
```
状态输出：
```text
top - 05:15:50 up 22:02,  4 users,  load average: 6.33, 2.66, 1.05
Tasks: 171 total,   8 running, 163 sleeping,   0 stopped,   0 zombie
%Cpu(s):100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   5406.7 total,    198.6 free,   5091.0 used,    781.1 buff/cache
MiB Swap:   6492.0 total,   6354.6 free,    137.4 used.    315.8 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  11560 zhongj    20   0   60588   8388   5136 R 100.0   0.2   1:49.55 stress-+
  11561 zhongj    20   0   60588   8388   5136 R  90.9   0.2   1:48.11 stress-+
  11563 zhongj    20   0   60588   8388   5136 R  90.9   0.2   1:48.48 stress-+
  11564 zhongj    20   0   60588   8388   5136 R  90.9   0.2   2:08.06 stress-+
  11571 zhongj    20   0 4254868   4.0g   2240 R  90.9  75.8   2:09.14 stress-+
  11562 zhongj    20   0   60588   8388   5136 R  45.5   0.2   1:54.15 stress-+
  11565 zhongj    20   0   60588   8388   5136 R  45.5   0.2   1:49.81 stress-+
  11589 zhongj    20   0   10776   5868   3816 R   9.1   0.1   0:00.01 top
Linux 7.0.0-43-generic (ubuntu1)        11/18/2026      _x86_64_        (6 CPU)

05:15:50 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:15:51 AM  all  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
05:15:52 AM  all   99.83    0.00    0.17    0.00    0.00    0.00    0.00    0.00    0.00    0.00
05:15:53 AM  all  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:     all   99.94    0.00    0.06    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```
3. 内存层
```bash
vmstat 1 5
```
状态输出：
```text
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 8  0 140684 203344  24172 775872   14   33    30    39  263    0  0  0 100  0  0  0
 7  0 140684 203344  24172 775912    0    0     0    20 6202  601 100  0  0  0  0  0
 7  0 140684 203344  24172 775912    0    0     0     0 6196  587 100  0  0  0  0  0
 7  0 140684 203344  24172 775912    0    0     0     0 6185  595 100  0  0  0  0  0
 7  0 140684 203344  24172 775912    0    0     0     0 6097  484 100  0  0  0  0  0
```
4. IO 层
```bash
iostat -x 1 3
```
状态输出：
```text
Linux 7.0.0-43-generic (ubuntu1)        11/18/2026      _x86_64_        (6 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.29    0.00    0.06    0.01    0.00   99.64

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             3.93     30.40     0.00   0.00    0.43     7.74   10.05     45.17     0.00   0.00    7.84     4.49    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.08   0.55
loop0            0.00      0.00     0.00   0.00    0.00     1.27    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
sda              1.22     31.14     2.74  69.25    0.58    25.56    8.09     45.17     1.96  19.47   10.56     5.58    0.00      0.00     0.00   0.00    0.00     0.00    0.04    1.15    0.09   0.06


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          99.83    0.00    0.17    0.00    0.00    0.00

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
loop0            0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
sda              0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          99.67    0.00    0.33    0.00    0.00    0.00

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             0.00      0.00     0.00   0.00    0.00     0.00   29.00    164.00     0.00   0.00    1.79     5.66    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.05   0.90
loop0            0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
sda              0.00      0.00     0.00   0.00    0.00     0.00   28.00    164.00     1.00   3.45    1.21     5.86    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.03   0.70
```
5. 进程层
```bash
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
```
状态输出：
```text
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
zhongj     11642  100  0.0   9764  4700 pts/2    R+   05:18   0:00 ps aux --sort=-%cpu
zhongj     11639 98.8 75.8 4254868 4197072 pts/1 R    05:18   0:02 stress-ng-vm
zhongj     11630 97.6  0.1  60588  8976 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11628 90.2  0.1  60588  8928 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11631 90.0  0.1  60588  8976 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11632 88.8  0.1  60588  8976 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11633 87.5  0.1  60588  8976 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11629 87.3  0.1  60588  8928 pts/0    R    05:18   0:04 stress-ng-cpu
zhongj     11636  0.3  0.2  60556 12628 pts/1    SL   05:18   0:00 stress-ng --vm 1 --vm-bytes 4G --timeout 180s --vm-keep
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
zhongj     11639 99.2 75.8 4254868 4197072 pts/1 R    05:18   0:02 stress-ng-vm
root        1350  0.0  1.0 789816 60592 ?        Ssl  Nov17   1:04 /usr/local/bin/mihomo -d /etc/mihomo
root        1532  0.0  0.5 2202748 28992 ?       Ssl  Nov17   0:10 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root        1329  0.0  0.5 126120 28444 ?        Ssl  Nov17   0:00 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
root        1333  0.2  0.4 2090136 22564 ?       Ssl  Nov17   2:50 /usr/bin/containerd
root        1177  0.0  0.3  46744 21888 ?        Ss   Nov17   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
zhongj      1322  0.0  0.3  33228 18664 ?        Ss   Nov17   0:17 /usr/bin/python3 /home/zhongj/lab/hello/server.py
root         580  0.0  0.2  50548 14872 ?        S<s  Nov17   0:00 /usr/lib/systemd/systemd-journald
root           1  0.0  0.2  25528 14812 ?        Ss   Nov17   0:04 /usr/lib/systemd/systemd --switched-root --system --deserialize=55
```

现象:Load 5.89，available 317Mi，Swap 已使用 137Mi，系统响应变慢。
CPU：us ≈ 100%，多个用户态进程占满 CPU。结论：CPU 饱和。
内存：available 很低，Swap 开始使用。结论：内存压力存在，但尚未严重到大量换页。
IO：%util 低，wa 接近 0。结论：磁盘不是瓶颈。
根因：人为启动的 stress-ng 同时消耗 CPU 和内存。
修复：kill 对应 PID 后系统恢复。

## 小结：
```text
综合排障要按 CPU → 内存 → IO → 进程 的顺序收集证据，避免一上来就找进程。
本次场景是 CPU 饱和 + 轻度内存压力，IO 正常。
修复时先确认是哪个进程，再决定是 kill、限流还是扩容。
```