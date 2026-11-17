# lesson5进程、CPU 与 Load Average
## 1. 基线
```bash
uptime
top -bn1 | head -20
nproc
cat /proc/loadavg
```
输出：
```text
 04:32:13 up 3 days, 15:36,  2 users,  load average: 0.15, 0.15, 0.10
top - 04:32:13 up 3 days, 15:36,  2 users,  load average: 0.15, 0.15, 0.10
Tasks: 182 total,   1 running, 181 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   5406.7 total,   2254.4 free,    777.7 used,   2735.9 buff/cache
MiB Swap:   6492.0 total,   6492.0 free,      0.0 used.   4629.1 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  35497 zhongj    20   0   10776   6252   4200 R  10.0   0.1   0:00.01 top
      1 root      20   0   25404  16572  11600 S   0.0   0.3   0:13.28 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.61 kthreadd
      3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_wo+
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      7 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      8 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
     10 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
     13 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
     14 root      20   0       0      0      0 S   0.0   0.0   0:00.14 ksoftir+
     15 root      20   0       0      0      0 I   0.0   0.0   0:07.23 rcu_pre+
6
0.15 0.15 0.10 1/301 35504
```
## 2. 制造 CPU 忙的场景
```bash
# 把所有核都打满（按 Ctrl+C 停止）
stress-ng --cpu $(nproc) --timeout 60s
```
另开终端，观察
```bash
uptime
top -bn1 | head -15
```
输出：
```text
 04:54:14 up 3 days, 15:58,  4 users,  load average: 1.12, 0.36, 0.17
top - 04:54:14 up 3 days, 15:58,  4 users,  load average: 1.12, 0.36, 0.17
Tasks: 192 total,   7 running, 185 sleeping,   0 stopped,   0 zombie
%Cpu(s): 98.5 us,  1.5 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
MiB Mem :   5406.7 total,   2150.9 free,    803.8 used,   2815.1 buff/cache     
MiB Swap:   6492.0 total,   6492.0 free,      0.0 used.   4603.0 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  36996 zhongj    20   0   60588   9228   5980 R 100.0   0.2   0:09.77 stress-+
  36997 zhongj    20   0   60588   9228   5980 R 100.0   0.2   0:09.79 stress-+
  36998 zhongj    20   0   60588   9228   5980 R 100.0   0.2   0:09.79 stress-+
  37000 zhongj    20   0   60588   9228   5980 R 100.0   0.2   0:09.79 stress-+
  37001 zhongj    20   0   60588   9228   5980 R 100.0   0.2   0:09.78 stress-+
  36999 zhongj    20   0   60588   9228   5980 R  90.9   0.2   0:09.73 stress-+
  37015 zhongj    20   0   10776   6280   4224 R   9.1   0.1   0:00.01 top
      1 root      20   0   25404  16596  11600 S   0.0   0.3   0:13.46 systemd
```
# 3. 制造 IO 等待的场景
```bash
# 写一个 2G 的大文件，制造磁盘压力
dd if=/dev/zero of=/tmp/bigfile bs=1M count=2048 oflag=direct status=progress
```
同时观察：
```bash
uptime
top -bn1 | head -15
```
输出：
```text
 05:03:39 up 3 days, 16:08,  4 users,  load average: 0.07, 0.27, 0.28
top - 05:03:40 up 3 days, 16:08,  4 users,  load average: 0.07, 0.27, 0.28
Tasks: 184 total,   1 running, 183 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  1.5 sy,  0.0 ni, 98.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
MiB Mem :   5406.7 total,    208.1 free,   2844.4 used,   4766.2 buff/cache     
MiB Swap:   6492.0 total,   6492.0 free,      0.0 used.   2562.3 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      1 root      20   0   25404  16596  11600 S   0.0   0.3   0:13.47 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.62 kthreadd
      3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_wo+
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      7 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      8 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
```


1.Load Average 高，不一定代表 CPU 忙。什么情况下是 CPU 忙？什么情况下是在等 IO？
```text
CPU 忙：表现为 us 和 sy 使用率很高，且 id 使用率很低。
IO 等待：表现为 wa 使用率很高，且 id 使用率很低。
```
2.你在 CPU 压测和 IO 压测时，分别观察到 us 和 wa 有什么不同？
```text
CPU 压测：表现为 us 和 sy 使用率很高，且 id 使用率很低。
IO 压测：表现为 wa 使用率很高，且 id 使用率很高。
```
3.如果面试官问“机器 load 很高，但 CPU 使用率不高，你怎么查？”，你现在能怎么回答？
```text
Load 高但 CPU 使用率不高，优先看 top 里的 wa（iowait）和进程状态。如果 wa 高，或者有大量进程处于 D 状态，就是在等 IO。接着用 iostat、iotop 或 pidstat -d 看是哪个磁盘、哪个进程在做大量读写。
```