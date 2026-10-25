# ghostlock-b5q

Samsung Galaxy Z Flip5（**SM-F7310**，固件 **F7310ZCS5DXJ2**，内核 5.15 GKI）
Bootloader 锁定状态下的**临时 root（每次开机重取）**——基于社区通用链
[GhostLock / CVE-2026-43499](https://nebusec.ai/research/ionstack-part-3)。

> 不解锁 BL、不清数据、不熔断 Knox（0x0 保持）。root 有效期 = 本次开机，
> 重启后重跑一条命令即可。

## 致谢 / Credits

- **NebuSec** —— [CVE-2026-43499 原始 exploit](https://github.com/NebuSec/CyberMeowfia/tree/main/IonStack/CVE-2026-43499/exploit)
- **[BuSung-dev/Root-My-Galaxy](https://github.com/Buung-dev/Root-My-Galaxy)** 及其
  [payloads 仓库](https://github.com/BuSung-dev/Root-My-Galaxy-Payloads) —— 移植框架、
  PORTING 流程、KernelSU late-load 方案（Apache-2.0）
- **KernelSU** —— late-load 内核模块

本仓库 = 按 PORTING.md 流程产出的 **b5q-F7310ZCS5DXJ2 机型配置** + 一处关键调优
+ 全自动 PC 端编排脚本。

## 实测时间分布（真机两日统计）

| 场景 | 耗时 |
|---|---|
| 最好 | **47 秒**（发射→uid=0） |
| 典型（好内存布局） | **3 – 5 分钟** |
| 坏布局单轮 | 28 分钟封顶后自动重启换布局 |
| 最差记录 | 5 连坏布局 + 第 6 轮命中 ≈ 2h54m（全程无人值守） |

慢的原因是内存布局抽签（每 boot 一次，mm 碰撞数 ≥3 才可继续），属于该 exploit
的固有属性——所有基于 43499 的工具速度特性相同。

## 快速开始

前提：Windows/Linux/macOS + USB 调试（MTP/文件传输模式亦可，adb 可见设备）。
首次部署一次即可，之后每次开机 root 只需第二条命令：

```bash
# 1. 部署（一次）
bash scripts/deploy.sh
# 2. 一键 root（每次开机后）
bash scripts/fast_root.sh
```

成功标志：`[+] ROOT LIVE`，验证：

```bash
adb shell "/data/local/tmp/f731/ksufd -- id"
# uid=0(root) ... context=u:r:ksu:s0
```

## 工作原理

1. `fast_root.sh` 重启手机，等 uptime 70–115s 的"boot 后黄金窗口"（内存布局最稳定）。
2. 设备端 `root_capture_fast.sh` 以 `EXPLOIT_ATTEMPTS=3 EXPLOIT_ATTEMPT_TIMEOUT_SEC=240`
   运行 payload：tracefs KASLR 泄露 → mm 碰撞定位 → fops 劫持 → pipe 物理读写
   → UMH 拉起 root 守护进程 → 立即 `--late-load` 装载 KernelSU。
3. 成功即停；28 分钟无果自动重启换内存布局再抽（最多 8 轮）。

## 本机型调优（相对上游的唯一修改）

`target/b5q-F7310ZCS5DXJ2/target.h`：

```diff
-#define S918_KSNITCH_FULL_COLLISIONS 5
+#define S918_KSNITCH_FULL_COLLISIONS 4
```

实机观测本机 mm 碰撞上限为 3（wanted = N-1），阈值 5 时永不可达；4 之后
好布局 boot 首试即中。二进制已随 `artifacts/` 提供，重编译见上游 PORTING.md。

## 排障

- **adb 找不到设备/频繁掉线**：`adb kill-server && adb start-server`（stale server 高发）。
- **失败**：永远是布局抽签——让脚本自己重启换布局，**不要**中途 kill 正在跑的
  exploit（每 boot 仅一次 stack-writer 机会），**不要**同时跑多个脚本实例。
- **设备端手动单发**（不重启）：
  `adb shell "cd /data/local/tmp/f731 && sh root_capture_fast.sh"`
- 仅适用于 `uname -r`/固件完全匹配的 F7310ZCS5DXJ2；其他固件需按上游 PORTING.md 重提取偏移。

## 免责

仅用于你本人拥有的设备的安全研究。临时 root 会降低设备安全性，请自行评估；
对 Knox/保修/数据的影响与社区同方法一致。

## License

Apache-2.0（随上游）。见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。

---

## English

Temporary root for Samsung Galaxy Z Flip5 (**SM-F7310**, firmware **F7310ZCS5DXJ2**,
kernel 5.15 GKI) with a locked bootloader, via the community-standard
[GhostLock / CVE-2026-43499](https://nebusec.ai/research/ionstack-part-3) chain.
This repo contains the `b5q-F7310ZCS5DXJ2` device profile ported with the
[Root-My-Galaxy](https://github.com/Buung-dev/Root-My-Galaxy) procedure (one-line
collision-threshold tuning, see diff above), prebuilt artifacts, and a fully
automated PC-side driver script: reboot → fire in the post-boot golden window →
fast-fail → auto retry on a fresh memory layout. Root lasts until reboot;
re-run one command after each boot. Typical 3–5 min, best 47 s. For your own
device only. Apache-2.0, see NOTICE for attribution.
