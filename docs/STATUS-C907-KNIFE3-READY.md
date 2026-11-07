# STATUS · 刀3｜C907 单核 rt-smart 冷链命令卡

- MARKER: C907-KNIFE3-READY-OK
- Time: 2026-11-07 Asia/Shanghai
- 机位: box（rsync 副本 job `v861-c907-knife3-ready-20261107`）
- 范围：**纯 HOST 只读核对 + 冷链命令卡**。未上板、未写板、未冷链执行、未 xfel、未 push、未 git lease、未改闸。
- 对照基线: `/workspace/shared/v881-flash-20261027/STATUS-STEP4-UART-MSH.md`（上板实证）、
  `/workspace/shared/v881-flash-20261027/NOTES-C907-RTSMART-SINGLE-GAP.md` §3「刀 3」，
  `/workspace/shared/v861-c907-knife2-host-20261107/STATUS-C907-HOST-KNIFE2.md`（sha256 核对）

---

## 1. 环境概览

| 项 | 值 |
|---|---|
| 串口 | COM6 / 115200 8N1（刀0已核） |
| payload 绝对路径 | `/workspace/shared/v881-flash-20261027/step4-fel-rtt/rtthread.bin` |
| payload sha256 | `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe` |
| payload size | 91092 B（0x163d4） |
| 返回型 launcher | `c907_boot_efex-bcache.bin`（见 §5） |
| 冷链来源 | 实盘 `/workspace/shared/v881-flash-20261027/STATUS-STEP4-UART-MSH.md`（2026-10-31 19:56，COM6） |

---

## 2. 冷链步骤顺序（每步一命令，来源文件路径）

**顺序严格对应** `STATUS-STEP4-UART-MSH.md` 与 `NOTES-C907-RTSMART-SINGLE-GAP.md` §3 刀3 所述实测通过的步骤。每行为独立 `xfel` 命令，来源文件为 box 实盘绝对路径。

1. `xfel write 0x100000 /workspace/shared/v881-flash-20261027/syterkit-bins/init_dram_efex.bin`
2. `xfel exec 0x100000`
3. `xfel write 0x40000000 /workspace/shared/v881-flash-20261027/step4-fel-rtt/rtthread.bin`
4. `xfel write 0x100000 /workspace/shared/v881-flash-20261027/syterkit-bins/c907_boot_efex.bin`
5. `xfel exec 0x100000`
6. **等待串口输出**：出现 `RT-Thread 5.4.0` banner、`STEP4-HEARTBEAT n=...` 与 `msh >`

> **说明**：
> - 第 1 步 `init_dram` 写+exec rc=0，DRAM 读回 `0x40000000` = `0x01234567`（已在刀2 基线验证）。
> - 第 3 步 payload `rtthread.bin` sha256 = `a8d0f5a8…`，与刀2 权威产物完全一致（`step4-fel-rtt/rtthread.bin` 与 `bsp-v861/rtthread.bin` 字节相同）。
> - 第 4 步 launcher 为返回型 `c907_boot_efex.bin`（非 `ret-v*` 变体），ENTRY 仍 `0x40000000`。
> - 若第 5 步执行后串口**无任何输出**，请参见「失败分支」§5。

---

## 3. 关键参数复核

| 参数 | 值 | 来源 |
|---|---|---|
| UART_COM | COM6 | `STATUS-STEP4-UART-MSH.md` 实测 |
| Baudrate | 115200 | `STATUS-STEP4-UART-MSH.md` 实测 |
| Data bits | 8 | `STATUS-STEP4-UART-MSH.md` 实测 |
| Parity | N | `STATUS-STEP4-UART-MSH.md` 实测 |
| Stop bits | 1 | `STATUS-STEP4-UART-MSH.md` 实测 |
| Payload address | 0x40000000 | `NOTES-C907-STEP4-HOWTO.md` §1.2；`STATUS-STEP4-UART-MSH.md` 实测 |
| init_dram address | 0x100000 | `NOTES-C907-STEP4-HOWTO.md` §113；`STATUS-STEP4-FEL-SCRIPT.md` 冷链 |
| c907_boot address | 0x100000 | `NOTES-C907-STEP4-HOWTO.md` §119；`run_uart_rtt.ps1` 冷链计划 |
| 成功 banner | `RT-Thread 5.4.0` + `STEP4-HEARTBEAT n=...` + `msh >` | `STATUS-STEP4-UART-MSH.md` 尾部日志 |

---

## 4. 成功判据（按历史 STATUS 原文抄）

- 串口在冷链第 5 步（`xfel exec 0x100000`）后 **90 秒内**出现下列三者之一即视为**部分成功**，三者皆出为**全成功**：
  1. `RT-Thread 5.4.0` banner（含 `copyright by RT-Thread team`）
  2. `STEP4-HEARTBEAT n=...` 行（每约 1s  once via volatile loop counter）
  3. `msh >` 提示符
- **全成功判定**：banner + heartbeat + msh 三者全部出现，且串口保持可读状态（无卡死、无垃圾字符）。
- **最低通过**：banner 出现即视为达成「C907 单核 rt-smart 起动第一半」。

---

## 5. 失败分支（usb bulk／离 FEL → 停写，要少爷重进，勿自创恢复术）

| 现象 | 处置 |
|---|---|
| `xfel version` 返回 `usb bulk send error` 或 `No FEL` | **立即停止**，不再执行后续 write/exec。确认 FEL 会话是否在之前 `exec` 后已死（需重插/重新进 FEL），**不要**自行推测 bin 有问题导致卡死。 |
| 冷链任意一步 `xfel write` 返回 rc≠0 | **立即停止**，不执行后续步骤。记录 rc 码，等少爷重进 FEL 会话后再试。 |
| 串口超时 90s 未出现任何 banner/心跳/msh | **立即停止**，不重试。可能原因：launcher 版本不匹配、DRAM 初始化不兼容、板级时钟树错误。**勿**自行改寄存器、自行修改 bin 或自行编译。等少爷介入。 |
| 串口出现乱码或非 RT-Thread 输出 | **立即停止**，不再分析。可能为 FEL 会话残留或时序错误。 |
| `xfel exec 0x100000` 后立刻返回 FEL（`xfel version` 仍 V881） | **视为「返回型 launcher 执行后 FEL 立即回」—— 这在返回型流程中是预期行为，但若随后串口仍无输出，则视为失败分支（C907 无实质起动）。** |

> **核心纪律**：一次只改一处；没有 L0（串口字节）不往下分析；缺串口/缺 FEL/缺 bin 时如实写缺口，不绕路。

---

## 6. 交叉核对（不可跳过）

- [x] payload `rtthread.bin` sha256 = `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe`（刀2 `/workspace/shared/v861-c907-knife2-host-20261107/STATUS-C907-HOST-KNIFE2.md` §1.1 权威产物）
- [x] payload 与 `step4-fel-rtt/rtthread.bin` 完全相同（同一 sha，见 §1.1 说明）
- [x] 链接地址 `text@0x40000000`、Entry `0x40000040` 对得上 `STATUS-STEP4-HOST-BIN.md`（本刀未复用该文件作为交差，仅作核对）
- [x] `RT_CPUS_NR=1`、`RT_PIC_RISCV_CLIC=1` 在 `rtconfig.h` 中已核实（见 `STATUS-C907-HOST-KNIFE2.md` §2）
- [x] UART 基址 `0x02500000`、PH9/PH10 mux5 在 `drv_uart_poll.c` 中已核实（见 `STATUS-C907-HOST-KNIFE2.md` §3.1）

---

## 7. 本刀范围声明

- **只做**本 STATUS 文件的 write；**未**修改任何工程源码、**未**编译、**未**上板、**未**冷链执行、**未** git push、**未**触碰 coexist bins。
- 冷链命令卡仅为「可照抄」记录，**不代表已执行**。实际上板请在 win 机器上依 `run_uart_rtt.ps1` 与 `syterkit-bins` 目录进行。
- **未**产出新 bin，因此**没有**新 sha 需要登记；继续以上板基线 `a8d0f5a8…` 为准。

MARKER=C907-KNIFE3-READY-OK