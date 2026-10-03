# SM-S9110 (dm1q) — S9110ZCS8FZI1 临时提权尝试记录

- 日期: 2026-10-03 20:30-20:41
- 设备: Samsung Galaxy S23 (SM-S9110 / dm1q)
- 固件: S9110ZCS8FZI1
- 内核: 5.15.189-android13-8-3251900-abS9110ZCS8FZI1 (KMI android13-5.15, GKI)
- Android: 16 / BP4A.251205.006
- 利用链: CVE-2026-43499 (IonStack / KernelSnitch), RootMyGalaxy payload
- 目标: KernelSU 临时 root (重启即失效, 不写分区)
- 结果: 失败。payload 在内核态进入 pipe physrw 阶段后崩溃, 触发内核 panic, 整机自动重启。

## 1. 失败点定位

日志最后一行: cfi starting pipe physrw
之后完全静默: 无成功行, 无失败行, 无重试行。
源码对应: src/fops.c:489 打印该行; 紧接其后 src/fops.c:338 -> install_pipe_physrw(fd) && install_android_root(fd); 以及 fops.c:510+ 的 fresh physrw pipe 重试逻辑。
结论: 系统在进入基于管道的任意物理内存读写原语 (physrw) 后即刻死于内核态 (panic), 而非 App 层报错。

## 2. 崩溃现场为何缺失

main.c:24-35 的 durable_log_checkpoint 要求 S_ISREG(st.st_mode) 才 fsync; App 用管道接 stdout (mode 010600):
[-] durable log checkpoint skipped stage=fops-page-held mode=010600
导致 panic 现场未落盘。

## 3. 已成功越过的关键阶段 (非结构性不兼容)

- KASLR: slide-kaslr-ok slide=0000000000048000
- controlled mm: group selected base=ffffff8862d18000 mode=0
- p0 物理写: p0 physical write status=0 ok=1
- CFI 假 fops: cfi write ret=35, llseek 覆盖, 回读校验通过
- 死点: cfi starting pipe physrw

## 4. 排除项: Shizuku 中途掉线是否导致失败

假设: 运行中 WiFi 断开 -> Shizuku 掉线 -> 导致失败?
结论: 排除。
1) payload 无外网依赖。全库仅有 AF_UNIX 本地套接字 (pipe.c:920, root.c:155/172, su_daemon.c:143/383/857); slide_app.c 的 AF_INET6 为回环组播。无 getaddrinfo/外网 connect/http。
2) Shizuku 只负责发射。ShizukuController.exec() 用 IShizukuService.newProcess() 起 /system/bin/sh -c true 并注入 LD_PRELOAD; 之后 payload 为独立 shell uid 进程 (pid=8107 uid=2000), 不再回调 Shizuku。
3) 决定性证据: 设备重启了。掉 binder/Shizuku 不会重启手机, 只有内核 panic/lockup 才会。
4) 日志无 [-] 错误行。InstallViewModel.kt:206-207 捕获异常时会 appendLog("[-] ...") 并置 Failed; 无此行说明 App 自身被冻结 (系统级 hang)。
5) 因果大概率相反: 内核崩 -> 重启 -> WiFi 断 & Shizuku 一起没了。

## 5. 与上游支持矩阵对照

- android13-5.15 在 DFRoot 兼容矩阵中标记为 Untested (唯一 Yes = android16-6.12)。
- 公开 feed 中 Galaxy S23 (base) 仅有 5.15.153 变体, 无 5.15.189 条目; 本设备 payload 为自建 FZI1 靶。
- 匹配规则: Build.MODEL + 内核 uname.release 前三段 (5.15.189)。

## 6. 状态与副作用

未发生: 内核持久化写入, KernelSU 持久安装, bootloader 状态改变, KNOX 熔断。payload 全程在内存中, 重启即清空。

## 7. 后续建议

- 盲重试: 具概率性但每次失败 = 一次 panic + 强制重启, 性价比低。
- 正确路线 Step 1: 仪器化诊断 (绕过 stdout, 直接 open(/sdcard/...) + write + fsync), 在 physrw 各子步骤插检查点。
