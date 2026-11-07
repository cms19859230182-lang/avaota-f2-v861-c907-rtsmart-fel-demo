# Avaota F2 (V861) · C907 单核 rt-smart FEL 冷链验证包（档 C）

本仓为 **Avaota F2 / Allwinner V861** 开发板的 **C907 单核 rt-smart** 启动验证包：
以 **FEL 冷链（五步）** 把 RT-Thread 5.4.0 送上 C907 大核，并以 **COM6 115200 串口日志**
作为板上成功证据（banner + msh）。

> 本包主题：**C907 单核 rt-smart FEL 冷链 ＋ 串口证据**。
> 不是 HOST／RISC-V 共存演示（那部分见姊妹仓
> `avaota-f2-v861-coexist-host-demo`，两者 bin 勿混用）。

- 板上实测：MARKER=C907-KNIFE3-BOARD-OK（2026-11-07 19:03–19:05）
- 详细逐条记录见 `docs/STATUS-C907-KNIFE3-BOARD.md`

---

## 1. 硬件与软件基线

| 项 | 值 |
|----|----|
| 板型 | Avaota F2（主控 Allwinner **V861**） |
| 目标核 | **C907 大核，单核**（本次只跑 C907，不启 A55/HOST 共存） |
| 上位机串口 | **COM6 @ 115200 8N1**（DTR/RTS=false，被动监听） |
| 烧录/引导方式 | **FEL**（USB，`xfel` v1.4.0） |
| FEL 设备 ID | `AWUSBFEX ID=0x00191800(V881)` |
| DRAM | DDR3 128 MB，single rank full DQ，DRAM CLK 1056 MHz |
| RTOS | **RT-Thread 5.4.0** |
| 验证日期 | 2026-11-07 |

板上实测 banner：

```
 \ | /
- RT -     Thread Operating System
 / | \     5.4.0 build Oct 31 2026 18:36:20
 2006 - 2026 Copyright by RT-Thread team
msh >
```

## 2. FEL 冷链五步

在 PC 端（Linux）依次执行。`xfel` 为本仓验证所用工具，`<port>` 视实际 USB 节点而定。

本次实测的**真实五步**（全部 rc=0，见 `docs/STATUS-C907-KNIFE3-BOARD.md` §3）：

```bash
# 0) 接通 FEL（板子上电进 FEL 模式），确认设备可见
xfel version

# 1) 载入 DRAM 初始化镜像到 0x100000
xfel write 0x100000 bins/init_dram_efex.bin
# 2) 执行之（DRAM 初始化，无输出属预期）
xfel exec  0x100000

# 3) 载入 RT-Thread 固件到 0x40000000
xfel write 0x40000000 bins/rtthread.bin

# 4) 载入 C907 launcher 到 0x100000
xfel write 0x100000 bins/c907_boot_efex.bin

# 5) 执行 launcher → 拉起 C907(rv32) 并跳转，宿主核随后休眠
xfel exec  0x100000
```

```
     ┌──────────┐  ①write/exec    ┌───────────────────────┐
PC ──┤  xfel    ├────────────────►│ init_dram_efex.bin    │
     └────┬─────┘   @0x100000     │ → DRAM 128MB 自检 OK  │
          │                       └───────────┬───────────┘
          │ ②write rtthread.bin @0x40000000   │
          │ ③write c907_boot_efex.bin @0x100000
          │ ④exec 0x100000（launcher）         │
          │                                   ▼
     ┌────┴────────────────────────────────────────────┐
     │ c907_boot: starting the C907 (rv32) @0x40000000 │
     │ → RT-Thread 5.4.0 banner → msh >                 │
     └───────────────────────┬─────────────────────────┘
                             │ UART COM6 @115200
                             ▼
                     logs/uart-com6.log
```

> 五步即：**①载入 DRAM 镜像 → ②执行 DRAM 初始化 → ③载入 RT-Thread 固件 →
> ④载入 C907 launcher → ⑤执行 launcher 拉起 C907**。

## 3. 成功判据

串口 **COM6 @ 115200** 上同时出现：

1. **RT-Thread 5.4.0 banner**（见上），且
2. **`msh >` 提示符出现、tshell 已注册**（日志 `STEP4-MSH-SCHED found=1 shell=tshell`）。

两者齐全即判定板上验证通过。原始日志见 `logs/uart-com6.log`。

日志中的自检标记（本次实测全部出现）：

| 标记 | 含义 |
|------|------|
| `c907_boot: starting the C907 (rv32) at 0x40000000` | launcher 拉起 C907 |
| `STEP4-BOARD-CONSOLE-READY` | 板级控制台就绪 |
| `STEP4-SOFT-TICK-ON cpu_hz=1008000000 cycles_per_tick=1008000` | 软 tick 生效 |
| `STEP4-APP-INIT-CREATED tid=0x4005a5fc` | main 线程创建 |
| `STEP4-MSH-SCHED found=1 shell=tshell priority=21` | msh 壳层可调度 |

## 4. 本仓内容

```
bins/
  rtthread.bin           RT-Thread 5.4.0 固件（C907 单核）
  c907_boot_efex.bin     C907 boot 辅助镜像（FEL 载入）
  init_dram_efex.bin     DRAM 初始化镜像
logs/
  uart-com6.log          COM6 115200 板上启动日志（banner + msh 证据）
docs/                    对照笔记、地址表、踩坑记录
  STATUS-C907-KNIFE3-BOARD.md   本次板上实测逐条记录（rc / 串口 / 判读）
  STATUS-C907-KNIFE3-READY.md   刀3 就绪前置状态
  NOTES-C907-KNIFE3-UPLOAD-PACK.md  上包配置笔记
  NOTES-C907-L2-VS-STILLZERO.md     L2 与 stillzero 对照笔记
SHA256SUMS              bins/ 的校验和
```

校验：

```bash
sha256sum -c SHA256SUMS
```

> `SHA256SUMS` 只覆盖 `bins/`（三份二进制为可复现主证据）；
> `logs/uart-com6.log` 为实测原始记录，另可由仓库历史校验。

预期 sha256：

| 文件 | SHA256 |
|------|--------|
| `bins/rtthread.bin` | `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe` |
| `bins/c907_boot_efex.bin` | `538aed5c920a0be328cef2da56b2085286c5de815939be54ea07b9774daea529` |
| `bins/init_dram_efex.bin` | `0493812f91497d5627caa846a4742fc4b8940b893ecedad1c731434ac7e76eaf` |

## 5. 已知局限（务必阅读）

- **未实现 STEP4-HEARTBEAT**：本包只到「banner + msh」，无心跳上报链路（日志中无
  `STEP4-HEARTBEAT` 行）。
- **未做 rootfs**：无文件系统／无用户态，纯内核 msh。
- **仅 C907 单核**：A55/HOST 侧未启用，不存在 HOST↔C907 共存（那是姊妹仓的内容）。
- FEL 地址与具体板卡丝印可能需微调，本仓以本次实测为准。
- 串口为**被动监听**（DTR/RTS=false），未向板端发送任何字符；`msh` 可交互性以
  提示符出现 + tshell 注册为证，未做完整命令回环测试。
- 内存布局与 efex 辅助镜像属实验验证用途，不作产品发布依据。

## 6. 姊妹仓

- `avaota-f2-v861-coexist-host-demo` —— HOST（A55）与 C907 **共存** HOST 侧演示。
  本仓是其 **C907 单核 FEL 冷链** 的前置/对照验证，两者 payload 不通用。

## 7. 许可

随仓文件为验证用途，bins 中 RT-Thread 部分遵循 RT-Thread 许可证（Apache-2.0）；
其余脚本/笔记见各文件头部说明。
