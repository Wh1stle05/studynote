# lesson6 内存与 Swap
## 1. 先看当前内存基线
```bash
free -h
cat /proc/meminfo | head -20
vmstat 1 5
```
输出：
```text
               total        used        free      shared  buff/cache   available
Mem:           5.3Gi       539Mi       4.7Gi       4.0Mi       244Mi       4.8Gi
Swap:          6.3Gi          0B       6.3Gi
MemTotal:        5536496 kB
MemFree:         4961276 kB
MemAvailable:    4983224 kB
Buffers:            5992 kB
Cached:           228860 kB
SwapCached:            0 kB
Active:           352264 kB
Inactive:          10124 kB
Active(anon):     139024 kB
Inactive(anon):        0 kB
Active(file):     213240 kB
Inactive(file):    10124 kB
Unevictable:       12688 kB
Mlocked:            9616 kB
SwapTotal:       6647800 kB
SwapFree:        6647800 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0 4960832   5992 244244    0    0   805    34  460    0  0  0 100  0  0  0
 0  0      0 4960652   5992 244352    0    0     0     0  607  278  0  0 100  0  0  0
 0  0      0 4960652   5992 244352    0    0     0     0  481  252  0  0 100  0  0  0

```
## 2. 制造“内存占用高但无压力”的场景
```bash
# 创建一个 2G 文件并反复读取，让它进入 page cache
dd if=/dev/zero of=/tmp/cachefile bs=1M count=2048 status=progress
# 反复读，把内容推进缓存
for i in {1..5}; do cat /tmp/cachefile > /dev/null; done
```
观察输出
```text
vmstat 1 3
               total        used        free      shared  buff/cache   available
Mem:           5.3Gi       2.5Gi       2.7Gi       2.0Gi       2.2Gi       2.8Gi
Swap:          6.3Gi          0B       6.3Gi
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  0      0 2859688   6004 2345312    0    0   774    33  456    0  0  0 100  0  0  0
 0  0      0 2859544   6004 2345312    0    0     0     0  582  282  0  0 100  0  0  0
 0  0      0 2859544   6004 2345312    0    0     0     0  474  242  0  0 100  0  0  0
```

## 3. 制造真正的内存压力
```bash
# 分配 4G 内存，持续 30 秒（根据你的实际内存调整，不要超过可用内存太多）
stress-ng --vm 1 --vm-bytes 4G --timeout 30s --vm-keep
```
另开终端观察：
```bash
free -h
vmstat 1 5
top -bn1 | head -15
```
输出：
```text
zhongj@ubuntu1:~$ free -h
vmstat 1 5
top -bn1 | head -15
               total        used        free      shared  buff/cache   available
Mem:           5.3Gi       5.2Gi       125Mi       2.0Gi       2.2Gi       112Mi
Swap:          6.3Gi       446Mi       5.9Gi
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  2 491248 131204   2824 2327208    0  357   495   377  402    0  0  0 100  0  0  0
 0  2 736036 136688   2832 2327176    4 244500     4 244520 16597 4850  0 15 65 20  0  0
 0  2 991180 136552   2832 2321404   24 254096    24 254096 13757 2941  0 16 64 20  0  0
 0  2 1244324 167656   2832 2305632    0 252268     0 252268 17491 2776  0 14 66 19  0  0
 0  2 1486260 226472   2832 2285284 6228 241828  6228 241828 21385 2203  0 14 66 20  0  0
top - 07:35:59 up 22 min,  2 users,  load average: 0.41, 0.19, 0.11
Tasks: 155 total,   1 running, 154 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  7.9 sy,  0.0 ni, 60.5 id, 27.6 wa,  0.0 hi,  3.9 si,  0.0 st 
MiB Mem :   5406.7 total,    233.9 free,   5200.8 used,   2246.5 buff/cache     
MiB Swap:   6492.0 total,   5008.3 free,   1483.7 used.    205.9 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2284 zhongj    20   0 4254868   2.6g    608 D  18.2  49.9   0:04.09 stress-+
   2293 zhongj    20   0   10776   5864   3816 R   9.1   0.1   0:00.01 top
      1 root      20   0   25236  13096  10608 S   0.0   0.2   0:01.35 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kthreadd
      3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_wo+
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
```
## 4.清理
```bash
rm -f /tmp/cachefile /tmp/bigfile
```

1.free -h 里 used 很高，但 available 还比较多，这说明什么？
```text
说明内存还有剩余，但是被缓存占用，导致available还比较多。
```
2.什么情况下你会认为“真正有内存压力”？需要看到哪些证据？
```text
当used很高，且available很少时，说明内存压力很大。
available 接近 0
Swap 开始被使用（Swap used 增加）
vmstat 的 si/so 非 0
进程出现 D 状态，或系统响应明显变慢
wa 升高（因为换页要写磁盘）
```
3.如果面试官问“服务器内存占用 90%，是不是需要立刻加内存？”，你现在会怎么回答？
```text
内存占用 90% 不一定需要立刻加内存。先看 available 和 Swap。如果 available 还比较多，且 Swap 几乎没用，那大部分是缓存，属于正常现象。如果 available 很低、Swap 持续增长、si/so 不为 0，或者已经出现 OOM killer 日志，才说明真正有压力，需要进一步分析是哪个进程在吃内存，再决定是优化还是加资源。
```


## 小结
```text
used ≠ 内存压力，要看 available。
缓存（buff/cache）是可回收的，占用高不等于有问题。
真正的内存压力标志：available 接近 0 + Swap 被使用 + si/so 非 0 + 进程 D 状态或系统变卡。
本次实验成功制造了从“无压力缓存”到“开始 swap”的对比。
```