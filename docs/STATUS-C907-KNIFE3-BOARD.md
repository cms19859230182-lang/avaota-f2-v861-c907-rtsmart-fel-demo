# STATUS · 刀3｜Win FEL 板上冷链灌装 C907 单核 rt-smart

- 时间：2026-11-07 19:03–19:05（Asia/Shanghai）
- 机位：win（PowerShell + Win 原生 xfel.exe，USB 直通 FEL；未用 WSL xfel）
- 结论：**注入链全绿，C907 单核 rt-smart 起 banner 并进 msh 交互提示符**
- MARKER=C907-KNIFE3-BOARD-OK

## 1. 工具与输入（sha 核验）

| 项 | 值 |
|---|---|
| xfel.exe | `C:\Users\seteiro\Tools\xfel\xfel-windows-v1.4.0\xfel.exe`（1547092 B） |
| init_dram_efex.bin | `C:\Users\seteiro\v861-official-try\syterkit-bins\init_dram_efex.bin`（29568 B） |
| rtthread.bin | 同上目录（91092 B），sha256=`a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe` ✅ 与任务要求一致 |
| c907_boot_efex.bin（launcher） | 8624 B，sha256=`538aed5c920a0be328cef2da56b2085286c5de815939be54ea07b9774daea529` ✅ 一致 |
| 串口 | COM6 / 115200 / 8N1（DTR=RTS=false，被动监听） |

本目录 `bins/` 内三份同 sha 副本已 `sha256sum` 复核，与 Win 侧一致。
未使用 bcache／ret-v* 类 launcher。

## 2. FEL 连通性（step 2）

```
xfel version   rc=0
AWUSBFEX ID=0x00191800(V881) dflag=0x44 dlength=0x08 scratchpad=0x0011fc00
```
无 `usb bulk send error`，无 `No FEL device` → 继续。

## 3. 冷链五步（step 3，逐步记录 exit code）

| # | 命令 | rc | 输出摘要 |
|---|---|---|---|
| 1 | `xfel write 0x100000 init_dram_efex.bin` | 0 | 100% 28.875 KB, 919.880 KB/s |
| 2 | `xfel exec 0x100000` | 0 | （无输出，预期） |
| 3 | `xfel write 0x40000000 rtthread.bin` | 0 | 100% 88.957 KB, 923.348 KB/s |
| 4 | `xfel write 0x100000 c907_boot_efex.bin` | 0 | 100% 8.422 KB, 886.711 KB/s |
| 5 | `xfel exec 0x100000` | 0 | （无输出，预期） |

五步**全 rc=0**，无 bulk 失败，未触发任何自创恢复动作。

## 4. 串口证据（step 4，监听 >90 s，实采 1784+ B）

监听窗口 400 s，exec 后持续听 100 s；COM6 文本：

```
[  164.148259][I] clk-sun252iw1: pll periph0 has been enabled, skip enable
[  164.156545][I] axp333: Found AXP333 PMU
[  164.162652][I] ZQ value = 0x606
[  164.166863][I] DRAM_VCC set to 1500 mv
[  164.174038][I] trefi:7.8ms
[  164.178650][I] [AUTO DEBUG] single rank and full DQ!
[  164.191698][I] [AUTO DEBUG] rank 0 row = 13
[  164.197145][I] [AUTO DEBUG] rank 0 bank = 8
[  164.202690][I] [AUTO DEBUG] rank 0 page size = 2 KB
[  164.208966][I] DRAM BOOT DRIVER INFO: V0.22
[  164.214396][I] DRAM CLK = 1056 MHz
[  164.218973][I] DRAM Type = 3 (2:DDR2, 3:DDR3, 6:LPDDR2, 7:LPDDR3)
[  164.226504][I] DRAMC ZQ value: 0x7b6bfb
[  164.231557][I] DRAM ODT off.
[  164.242344][I] trefi: 7.8us
[  164.246167][I] DRAM Size = 128 MB
[  164.250653][I] dram_tpr11:0x460000
[  164.255222][I] dram_tpr12:0x55
[  164.259417][I] dram_tpr13:0x34046501
[  164.265301][I] DRAM simple test OK.
[  165.555677][I] c907_boot: starting the C907 (rv32) at 0x40000000, this core sleeps afterwards
STEP4-BOARD-CONSOLE-READY
STEP4-SOFT-TICK-ON cpu_hz=1008000000 cycles_per_tick=1008000

 \ | /
- RT -     Thread Operating System
 / | \     5.4.0 build Oct 31 2026 18:36:20
 2006 - 2026 Copyright by RT-Thread team
STEP4-APP-INIT-CREATE
STEP4-APP-INIT-CREATED tid=0x4005a5fc main
STEP4-PROBE-R3: ENV-INIT-CREATE
STEP4-PROBE-R3: APP-INIT-CREATE
msh >STEP4-MAIN-ENTER-A
STEP4-MAIN-ENTER-B tick=0
STEP4-MAIN-ENTER
STEP4-SOFT-TICK-ON tick=0 prio=21
STEP4-MSH-SCHED found=1 shell=tshell main_prio=21
```

## 5. 判读

- `init_dram_efex.bin` 起效：DRAM 自检 OK，128 MB，DDR3 @1056 MHz，single rank full DQ。
- `c907_boot_efex.bin` 成功拉起 C907（rv32）于 0x40000000，宿主核随后休眠 —— 与预期 launcher 语义一致。
- **C907 单核 rt-smart 起 banner**：`RT-Thread Operating System 5.4.0 build Oct 31 2026 18:36:20`。
- 调度器活：SOFT-TICK-ON（cpu_hz=1008000000，cycles_per_tick=1008000）、main 线程 tid=0x4005a5fc、priority 21。
- **msh 交互提示符出现**（`msh >`），tshell 已注册（found=1），即壳层可交互。
- 结论：单核 rt-smart 在 V861 板 B 上 **banner + msh 双达成**，最低通过条件满足。

## 6. 产物

| 文件 | 说明 |
|---|---|
| `uart-com6.log` | COM6 原始串口日志（1787 B，含 banner 与 msh） |
| `xfel-coldchain.log` | xfel 各步 rc + 输出 + 串口全文（带时间戳） |
| `run-c907-knife3-board.ps1` | 本次执行脚本（可复跑） |
| `STATUS-C907-KNIFE3-BOARD.md` | 本文件 |

## 7. 边界

未改任何 .c/.h，未重新编译，未 git commit／push。串口仅被动监听（DTR/RTS 保持 false），未发送任何字符。

MARKER=C907-KNIFE3-BOARD-OK
