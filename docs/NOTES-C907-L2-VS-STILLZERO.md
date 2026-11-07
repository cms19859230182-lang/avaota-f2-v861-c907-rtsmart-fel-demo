# NOTES · C907 L2 历史证据 vs STILL-ZERO 对账

- 时区：Asia/Shanghai；机位：box（rsync 副本）。
- 本刀**只写笔记**：不改工程、不上板、不 push、不执行 xfel、不 web_search。
- 只读已知路径（勿乱扫全盘）：`../v881-flash-20261027/NOTES-C907-RTSMART-SINGLE-GAP.md`、`../v861-c907-knife2-host-20261107/STATUS-C907-HOST-KNIFE2.md`、`../v861-c907-knife0-uart-20261107/STATUS-C907-UART-KNIFE0.md`、`../v881-flash-20261027/STATUS-STEP4-UART-MSH.md`、`STATUS-STEP4-FEL-RTT.md`、`STATUS-SYTERKIT-C907.md`、`STATUS-SYTERKIT-C907-STOCK-RECHECK.md`、`STATUS-SYTERKIT-C907-RET.md`。

---

## 1. 两条证据链对照表

| 对比项 | Chain A：stock SyterKit 冷链 | Chain B：rtthread.bin（刀2/上板基线） |
|---|---|---|
|  payload | stock `c907_boot_efex.bin`（官方永 WFI）+ `payload-a2.bin`（172B 哨兵载体） | 内核 `rtthread.bin`（RT-Thread 5.4.0） |
|  payload sha | `c907_boot_efex.bin`: `538aed5c…`；`payload-a2.bin`: `bcbeb24b…`；`c907_boot_efex-bcache.bin`（返回型）: `efbba80b…` | `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe` |
|  载入地址 | `payload-a2` → `0x40000000`；`c907_boot` → `0x100000` | `rtthread.bin` → `0x40000000`；`c907_boot` → `0x100000` |
|  Entry 点 | `c907_boot` 官方入口（E907 后永 WFI） | `rtthread.elf` Entry `0x40000040`，`.start` LOAD/VMA `0x40000000` |
|  FEL 版本/通道 | `AWUSBFEX ID=0x00191800(V881)`，version rc=0 | 同上 `ID=0x00191800(V881)`，version rc=0 |
|  DRAM 签名 | `read32 0x40000000`=`0x01234567` 通过 | 同冷链（init_dram 首先跑，rc=0） |
|  哨兵期望 | 写 `0x40010000=deadbeef`、`0x40010040=cafebabe`；期 C907 跑后被改 | 无专用哨兵；以 UART banner/msh 为判据 |
|  哨兵实际（读回） | **仍 `deadbeef`/`cafebabe`，未被改**；`0x0010E000/E010` 全 0（无 `0xC907A11C`） | 无哨兵机制；改为串口输出判定 |
|  串口结果 | **C907-UART-NO-PORT**：本机仅 COM1 主板口，无 AvaotaUART/CH340；0 字节、无监听 | **COM6（CH340，115200 8N1）**：RT-Thread 5.4.0 banner、`msh >`、软 tick（GOT_MSH=True, GOT_TICK=True, GOT_HB=False） |
|  串口可得性 | 2026-10-27 无可用板载调试口 | 2026-11-07 刀0 证实 COM6 CH340 在位、可打开（pass1/pass2 共 18s，0 字节因对端未上电/未共地，适配器本身正常） |
|  xfel 后状态 | 官方永 WFI → `usb bulk send error`（LEFT_FEL，不可读回）；返回型 bcache → FEL 存活但 C907 无执行痕迹 | `c907_boot exec` 后 UART 抓到 `c907_boot: starting the C907 (rv32) at 0x40000000` → RT-Thread banner；FEL 协议后续失联属预期（核已接管） |
|  日期/标记 | 2026-10-27 16:28 `C907-PARTIAL`；16:38 `C907-STILL-ZERO`；17:43 `LEFT_FEL` | 2026-10-31 19:56:35 `MSH-OK`（L2：banner+tick+msh） |
|  核心结论 | 冷链四步 rc=0、DRAM 签名 OK，**但 C907 核至今无「跑起来」实证** → `C907-STILL-ZERO` | **L2 已达成**：banner + 软 tick + msh，`a8d0f5a8…` bin 在板上真跑过 |

---

## 2. 为何刀2 说 L2，刀1 仍在查 stock STILL-ZERO

两条链不是同一件事：

- **Chain A（stock）**：2026-10-27 用官方 `c907_boot_efex.bin` 走完 init_dram → payload@0x40000000 → c907_boot write/exec，四步 rc=0、DRAM 签名 `0x01234567` 通过。**但**这只证明「FEL 通道通 + DRAM 能被初始化 + payload 被写入」，**不等于「C907 执行了 payload」**。返回型 bcache 读回槽位后，`0x40010000`/`0x40010040` 仍是 `deadbeef`/`cafebabe`，`0x0010E000/E010` 全 0，无 `0xC907A11C`；加之当时**无可用串口**（C907-UART-NO-PORT），无法通过 console 抓取 C907 侧任何输出。故结论为 `C907-STILL-ZERO` —— 冷链「形式通、实质未验」。

- **Chain B（rtthread.bin）**：2026-10-31 用同一套 FEL 冷链（init_dram → 写 `rtthread.bin` @0x40000000 → c907_boot write/exec），区别是 payload 换成了真正的 RT-Thread 内核（sha `a8d0f5a8…`，text@0x40000000，Entry 0x40000040），且此时能通过 **COM6 CH340 @115200** 抓串口。UART 尾部出现 `c907_boot: starting the C907 (rv32) at 0x40000000` 后紧跟 RT-Thread 5.4.0 banner、`STEP4-SOFT-TICK-ON`、`msh >` 与 `STEP4-MSH-SCHED` —— 这是 L2（banner+msh+软 tick）的直接证据。刀2（2026-11-07 18:45，纯 HOST）复核确认该 bin 链接地址、单核配置（`RT_CPUS_NR=1`）、CLIC、UART 真写寄存器均一致，与上板基线互相印证，无矛盾。

**一句话**：刀2 的 `a8d0f5a8…` L2 是「换 payload 成 RT-Thread + 有串口」后的正向证据；刀1 的 stock `C907-STILL-ZERO` 是「原装 payload + 无串口」下的负向/未决证据，二者不冲突，但 Chain B 是唯一能证明「C907 真跑了 rt-smart 内核」的链。

---

## 3. 刀3 建议：复用哪条 payload / 哪份冷链

**复用 Chain B：**
- **payload**：`rtthread.bin`，sha256 `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe`，size 91092 B，text@0x40000000，Entry@0x40000040，`RT_CPUS_NR=1`。两份字节相同副本（构建树 `step4-dt/bsp-v861/rtthread.bin` 与共享目录 `step4-fel-rtt/rtthread.bin` sha 完全一致，无漂移）。
- **冷链脚本/序列**：复用 `../v881-flash-20261027/step4-fel-rtt/STATUS-STEP4-UART-MSH.md` 中记录的冷链：`xfel version` → `init_dram_efex.bin` write+exec @0x100000 → `read32 0x40000000` 校 `0x01234567` → `rtthread.bin` write @0x40000000 → `c907_boot` write+exec @0x100000 → **COM6 @115200 8N1 抓 banner/msh**。目录下现有 `run_live.ps1` / `run_uart_rtt.ps1` / `run_live_rtt.ps1` 等 PS1 跑车器，未发现名为独立「冷链脚本」的文件，以 STATUS-STEP4-UART-MSH.md 记录的命令序列为准（勿编造脚本名）。
- **验收口**：刀0 已证实 COM6 CH340 在位，115200 8N1 可打开，可作 L2（banner+msh）验收口；上板前需确认 C907 目标板已上电、CH340 TX/RX 交叉、共地。

**stock STILL-ZERO 对刀3 的定性：`C907-STILL-ZERO` 是 blocker（若被误读为「C907 已能跑」），不是可复用 payload。** 它不能作为刀3 的基线，原因：(1) 官方 `c907_boot_efex.bin` 让 E907 永 WFI，xfel 协议断，槽位不可读回，无法经 xfel 验核；(2) 邮箱哨兵全程未变，无 C907 执行痕迹；(3) 当时无串口，缺 console 真值。它只能作为**反面对照/负对照**：说明「FEL 四步 rc=0 + DRAM 签名 OK」并不等于「C907 执行 payload」，刀3 不能只看 xfel rc=0 就宣判成功，必须以 COM6 串口 banner/msh 为 L2 判据。

---

## 4. 缺口清单（仅写缺，勿编）

1. **可核对的独立「冷链脚本」文件** —— 目录下仅 `run_live.ps1`/`run_uart_rtt.ps1`/`run_live_rtt.ps1` 等 PS1 跑车器，未找到单一可复用的「冷链脚本」；以 STATUS-STEP4-UART-MSH.md 命令序列为准，脚本化需另行整理。
2. **rtthread.bin 用户态 rootfs** —— 刀3 若追 L3（用户态/msh 交互），仍缺 `smart-rootfs`/`smart-image` 产出；刀3 以内核态 L2（banner+tick+msh）为阶段目标可暂不阻塞。
3. **RT_PIC_RISCV_CLIC_MAX_NR=128 与 `SUN252IW1_CLIC_NUM_SOURCES=208` 不一致** —— 刀2 列为待定，未改；对 rt-smart 单核 L2 非阻塞，但需记录。
4. **`v861_ccu_pio_enable()` 仍空实现**（PIO BGR/总线门控待确认） —— 当前 UART 靠直写 `PH_CFG1` 绕过；若上板 UART 失效需补。
5. **UART 时钟源 24 MHz 估值** —— 波特率 divisor 按 24 MHz 算；新板/新时钟树时需重算。现网板 COM6 已收到完整 banner，可认为实证。
6. **刀1 stock 取证结果** —— 刀1 正在 win 取证 stock，本刀未等待、未复述；若刀1 后续产出与本对账冲突，需以刀1 为准重新核对。

---

末行：MARKER=C907-L2-VS-STILLZERO-OK
