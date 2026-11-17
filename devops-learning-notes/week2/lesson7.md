# lesson7 文件系统、Inode 与磁盘 IO
## 1. 磁盘基线
```bash
df -h
df -i
lsblk
iostat -x 1 3 2>/dev/null || (sudo apt install -y sysstat && iostat -x 1 3)
```
输出:
```text
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              1.1G  988K  1.1G   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   23G   11G   11G  51% /
tmpfs                              2.7G     0  2.7G   0% /dev/shm
efivarfs                           128M   37K  128M   1% /sys/firmware/efi/efivars
none                               1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
none                               1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
tmpfs                              2.7G     0  2.7G   0% /tmp
/dev/sda2                          2.0G   97M  1.7G   6% /boot
/dev/sda1                          1.1G  6.4M  1.1G   1% /boot/efi
none                               1.0M     0  1.0M   0% /run/credentials/systemd-networkd.service
none                               1.0M     0  1.0M   0% /run/credentials/getty@tty1.service
tmpfs                              541M  8.0K  541M   1% /run/user/1001
Filesystem                         Inodes  IUsed   IFree IUse% Mounted on
tmpfs                              819200    761  818439    1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv 1540096 128976 1411120    9% /
tmpfs                              692062      1  692061    1% /dev/shm
efivarfs                                0      0       0     - /sys/firmware/efi/efivars
none                                 1024      1    1023    1% /run/credentials/systemd-journald.service
none                                 1024      1    1023    1% /run/credentials/systemd-resolved.service
tmpfs                             1048576     14 1048562    1% /tmp
/dev/sda2                          131072    334  130738    1% /boot
/dev/sda1                               0      0       0     - /boot/efi
none                                 1024      1    1023    1% /run/credentials/systemd-networkd.service
none                                 1024      1    1023    1% /run/credentials/getty@tty1.service
tmpfs                              138412     35  138377    1% /run/user/1001
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   50G  0 disk 
├─sda1                      8:1    0    1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0 46.9G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 23.5G  0 lvm  /
Linux 7.0.0-43-generic (ubuntu1)        11/17/2026      _x86_64_        (6 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.07    0.00    0.32    0.40    0.00   99.22

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s%wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0           167.09   1061.15     0.00   0.00    0.42     6.35  360.52   1450.94     0.00 0.00    8.44     4.02    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.003.11   2.25
loop0            0.01      0.01     0.00   0.00    0.00     1.27    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
sda             49.58   1093.12   118.95  70.58    0.58    22.05  291.00   1451.04    69.5519.29   11.58     4.99    0.00      0.00     0.00   0.00    0.00     0.00    0.12    1.463.40   1.74


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.17    0.00    0.00   99.83

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s%wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
loop0            0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
sda              0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.17    0.00    0.00   99.83

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s%wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             2.00      8.00     0.00   0.00    0.00     4.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
loop0            0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
sda              1.00      8.00     1.00  50.00    1.00     8.00    0.00      0.00     0.00 0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.000.00   0.00
```

## 2. 制造“磁盘空间快满”的场景
```bash
# 写 2.5G 文件（根据你实际剩余空间调整，确保不要写满）
dd if=/dev/zero of=/tmp/spacefile bs=1M count=2560 status=progress
df -h /tmp
```
输出:
```text
2240806912 bytes (2.2 GB, 2.1 GiB) copied, 1.00055 s, 2.2 GB/sdd: IO error: Disk quota exceeded
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           2.7G  2.2G  541M  80% /tmp
```
## 3. 制造 Inode 耗尽的风险
```bash
mkdir -p /tmp/inodetest
# 创建 5000 个小文件
for i in $(seq 1 5000); do touch /tmp/inodetest/file$i; done

df -i /tmp
ls /tmp/inodetest | wc -l
```
输出:
```text
Filesystem      Inodes IUsed   IFree IUse% Mounted on
tmpfs          1048576  5018 1043558    1% /tmp
5000
```
## 4. 观察磁盘 IO 压力
用 fio 或继续用 dd 制造写压力，同时看 iostat：
```bash
# 如果没有 fio，先安装
sudo apt install -y fio

# 另开一个终端持续观察
iostat -x 1

# 当前终端制造随机写压力（30 秒）
fio --name=randwrite --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k --size=512M --runtime=30 --time_based --filename=/tmp/fiotest
```
输出:
```text
vg-cpu:  %user   %nice %system %iowait  %steal   %idle
           3.34    0.00   13.55    0.00    0.00   83.11

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqmw_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
dm-0             0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00   0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
loop0            0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00   0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
sda              0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00   0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00    0.00    0.00   0.00
```
清理测试文件:
```bash
rm -f /tmp/fiotest
```

1.df -h 显示还有空间，但创建文件失败，可能是什么原因？怎么确认？
```text
可能是inode耗尽，需要确认inode数量。也可能是配额限制，可以用 quota 或看系统日志确认。
```
2.磁盘 %util 很高时，通常还要看哪个指标才能判断是不是真的慢？
```text
%util 高 + await（平均等待时间）明显升高 → 磁盘确实忙且响应变慢
只看 iowait 可能被其他因素干扰，await 更能直接反映单次 IO 的延迟
```
3.如果面试官问“磁盘使用率 95%，但服务还正常，需要立刻处理吗？”，你会怎么回答？
```text
先确认是空间满还是 Inode 满（df -h 和 df -i）。如果只是空间使用率高，但服务正常、%util 和 await 都不高，可以先观察增长趋势，规划清理或扩容，不一定要立刻停服处理。如果已经出现写失败、Inode 耗尽，或者 IO 延迟明显升高，就要优先处理。
```

## 小结
```text
df -h 看空间，df -i 看 Inode，两者都要查。
/tmp 是 tmpfs，写它不等于写磁盘。
判断磁盘是否真的慢：%util 高的同时看 await 是否升高。
本次实验验证了空间和 Inode 的查看方法，IO 压力在虚拟机 + tmpfs 环境下不明显。
```