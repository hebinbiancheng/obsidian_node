---
title: 搞懂 Linux 假死现象——内核级运维排障
date: 2026-07-02
tags: [linux, 运维, 内核, 假死, 排障, soft-lockup, hard-lockup]
category: 知识资料
source: https://mp.weixin.qq.com/s/5NawLVit9L_cSbHMb7i8eA
---

# Linux 假死现象：内核级运维排障

> 原文: [微信文章](https://mp.weixin.qq.com/s/5NawLVit9L_cSbHMb7i8eA)

---

## 一、假死的典型症状

### 命令行与 SSH

- 输入命令无回显，`ls`、`ps`、`top` 等基础命令无响应
- SSH 连接超时或连接后输入无反应
- **区分服务假死 vs 网络故障**：`ping` 目标 IP，能通则网络正常、服务假死；不通则排查网络链路

### 图形界面

- **场景一**：鼠标可移动但无法点击窗口 → 桌面进程（gnome-shell/kwin）用户态卡死
- **场景二**：鼠标完全不动，屏幕静止 → GPU 驱动挂起或内核图形子系统阻塞 → `dmesg` 查显卡日志

### 网络行为

- **能 ping 通但服务全停**：用户态服务进程假死或端口监听异常
- **完全 ping 不通**：内核网络栈阻塞、网卡驱动故障或防火墙规则异常

### 硬件反馈

- 硬盘灯常亮 → 磁盘 I/O 阻塞或文件系统元数据卡死
- 风扇异常飙升 → CPU 死循环或硬件温度失控
- **SysRq 组合键**：有反应 = 内核仍可调度；无反应 = 硬锁定或中断关闭

---

## 二、五种内核假死机制

### 1. 软锁定 (soft lockup)

![soft lockup](assets/k8s存储选型-2.png)

- 中断开启，内核代码长时间死循环占用 CPU
- 调度器无法切换任务
- 通常仅单核卡死，其他核心正常
- 看门狗线程定时巡检，超时打印告警日志
- 常见原因：长时间持自旋锁、关闭抢占代码过久、大规模 RCU 同步

### 2. 硬锁定 (hard lockup)

![hard lockup](assets/k8s存储选型-3.png)

- 内核**关闭所有中断**陷入死循环
- 连 NMI（不可屏蔽中断）都无法响应
- 整机彻底冻结，依靠 NMI 硬件看门狗监测
- 超时后自动生成崩溃日志并重启
- 常见原因：驱动长时间关中断、中断上下文死循环、BIOS/SMM 长期占用 CPU

### 3. D 状态泛滥

![D 状态](assets/k8s存储选型-4.png)

- D 状态 = 不可中断休眠态，**无法被任何信号杀死**
- 进程等待磁盘 IO、网络资源时进入
- 大量进程扎堆 D 状态 → 负载飙升、调度失效 → 系统假死
- 排查：`ps aux | awk '$8~/D/'` 筛选 D 态进程

### 4. 内存耗尽与 OOM 伪假死

![OOM](assets/k8s存储选型-5.png)

- 物理内存用尽 → 内核频繁回收/换页 → 整机卡顿
- 系统仍在运行，只是极慢
- OOM Killer 自动终止高内存进程
- 排查：`vmstat 1` 看 `si/so`（swap in/out）和 `wa`（IO 等待）

### 5. 死锁与活锁

![死锁](assets/k8s存储选型-6.png)

- **ABBA 死锁**：进程 A 持锁 1 等锁 2，进程 B 持锁 2 等锁 1
- **锁顺序反转**：不按约定顺序拿锁
- **用户态/内核态交叉死锁**：用户进程持 pthread_mutex 进内核拿内核锁
- **活锁**：进程不阻塞但一直做无用功

---

## 三、核心成因

### 存储与文件系统

![存储](assets/k8s存储选型-7.png)

**NFS/SMB 挂载超时**：

```bash
# NFS 推荐参数：可中断 + 软挂载 + 超时
mount -t nfs -o intr,soft,timeo=60 192.168.1.100:/shared /mnt/nfs

# SMB
mount -t cifs -o vers=3.0,timeo=30 //192.168.1.101/shared /mnt/smb
```

**磁盘故障**：坏道导致 IO 不断重试 → 进程 D 状态

**文件系统日志卡死**：ext4/XFS 日志回放失败 → 文件系统冻结

### 内存与换页

![内存](assets/k8s存储选型-8.png)

- **overcommit**：内核允许超额申请内存，所有进程真用时系统崩溃
- **直接回收**：`kswapd` 不够用，进程自己同步回收，耗时数百毫秒
- **swap 颠簸**：swap 放机械硬盘 → 随机 IO 极差 → 系统卡成 PPT
- **透明大页 (THP)**：后台内存规整可能暂停进程数毫秒，Redis/Hadoop 建议关闭

### 内核同步与调度

![同步](assets/k8s存储选型-9.png)

- 单核自旋锁 = 抢占禁用，拿不到 CPU 直接死锁
- SCHED_FIFO 高优先级死循环 → 霸占 CPU，SSH/cron 全挂
- `isolcpus` 隔离的 CPU 上任务挂死 → CPU 彻底废掉

### 驱动与硬件

![驱动](assets/k8s存储选型-10.png)

- GPU 驱动（NVIDIA 闭源）休眠唤醒死锁 → 桌面冻住
- 网卡驱动死循环 → 网络反复 reset
- SMI（系统管理中断）长时间停留 → CPU 被固件占据

### 用户空间伪假死

![用户空间](assets/k8s存储选型-11.png)

- **systemd 卡住** → `systemctl restart` 卡死、关机失败
- **D-Bus 冻结** → 窗口管理器、文件管理器全僵，但 X Server 还活着
- **PID 耗尽** → `fork: retry: Resource temporarily unavailable`，SSH 都登不上

---

## 四、排障实操

### CPU 高负载假死

```bash
top -bn1 | head -20              # 找高 CPU 进程
ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | head -20
perf top -p <pid>                # 查看进程热点函数
```

### D 状态进程排查

```bash
ps aux | awk '$8~/D/'            # 列出 D 状态进程
cat /proc/<pid>/wchan            # 查看阻塞在哪
cat /proc/<pid>/stack            # 内核调用栈
```

### 软锁定日志

```bash
dmesg | grep -i "soft lockup\|hung_task"
grep -i "soft lockup" /var/log/messages
```

### OOM 排查

```bash
dmesg | grep -i "out of memory\|oom"
grep -i "killed process" /var/log/messages
free -h && vmstat 1 10
```

### 救命三板斧

```bash
# 1. SysRq 安全重启（按顺序逐个执行）
echo 1 > /proc/sys/kernel/sysrq        # 开启 SysRq
echo s > /proc/sysrq-trigger            # sync 磁盘
echo e > /proc/sysrq-trigger            # kill -15 所有进程
echo i > /proc/sysrq-trigger            # kill -9 所有进程
echo u > /proc/sysrq-trigger            # remount 只读
echo b > /proc/sysrq-trigger            # 重启

# 2. NMI 看门狗（硬锁定时）
echo 0 > /proc/sys/kernel/nmi_watchdog   # 关闭看门狗
echo 1 > /proc/sys/kernel/nmi_watchdog   # 开启

# 3. Magic SysRq 快捷键
Alt + SysRq + (R E I S U B)              # 逐个按，安全重启
```

### 预防措施

```bash
# 关闭 THP（透明大页）
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# NFS 软挂载
mount -t nfs -o soft,timeo=30,intr server:/path /mount

# 限制 PID 上限
echo 65536 > /proc/sys/kernel/pid_max

# 配置 OOM 优先级
echo -1000 > /proc/<pid>/oom_score_adj    # 保护关键进程
```
