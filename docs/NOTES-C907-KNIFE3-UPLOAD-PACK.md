# NOTES · C907 刀3 成功后个人仓上传包清单（只写笔记）

- 时区：Asia/Shanghai；机位：box（rsync 副本 `v861-c907-knife3-success-pack-notes-20261107`）。
- 本刀**只写笔记**：不上板、不 push、不执行 xfel、不改任何 .c/.h/.S、不 web_search。
- 仅读已知路径：`../v861-c907-knife3-ready-20261107/STATUS-C907-KNIFE3-READY.md`、`../v861-c907-l2-vs-stillzero-20261107/NOTES-C907-L2-VS-STILLZERO.md`、`../v881-flash-20261027/step5-hetero/demo-pack-b/`（档B 结构参考）。

---

## 1. 背景与触发条件

刀3（Chain B：`rtthread.bin` 作 payload，`c907_boot_efex.bin` 作返回型 launcher）在 Win 机位上板灌装后，串口若在冷链第 5 步（`xfel exec 0x100000`）后 90 s 内出现 `RT-Thread 5.4.0` banner（全成功需再有 `STEP4-HEARTBEAT n=...` 与 `msh >`），即视为 L2 达成。

**触发条件**：banner 一出 → 立即把下列文件打进个人仓（本地先打包，暂不 push）。

---

## 2. banner 一出要打进个人仓的文件清单

按档 B `demo-pack-b/` 结构（README／NOTICE／LAYOUT／PLAN／DONE／SHA256SUMS／bins／docs／logs／scripts）组织，个人仓放在 `c907-knife3-pack/`（或同名目录）。

### 2.1 根目录文件

| 路径 | 用途 | 来源/依据 |
|---|---|---|
| `c907-knife3-pack/README.md` | 中文说明（≤50 行，见 §3 大纲） | 本刀写大纲；上板后补 |
| `c907-knife3-pack/NOTICE` | GPL-2.0 + libdram 闭源 stub 声明 | 参考 `demo-pack-b/NOTICE` |
| `c907-knife3-pack/LAYOUT.md` | 加载地址表（禁踩区） | §4 |
| `c907-knife3-pack/PLAN.md` | 刀3 待办 / 下一刀 | §5 |
| `c907-knife3-pack/DONE-C907-KNIFE3-PACK.md` | 打包完成标记 + SHA 核对结果 | 上板后补 |
| `c907-knife3-pack/SHA256SUMS.txt` | 本包全部 bin 的 SHA256 | §2.2 |

### 2.2 bins/（产物，带 SHA）

| 路径 | 字节 | SHA256 | 说明 |
|---|---|---|---|
| `bins/rtthread.bin` | 91092 | `a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe` | rt-smart 内核 payload，text@0x40000000，Entry@0x40000040，`RT_CPUS_NR=1` |
| `bins/c907_boot_efex.bin` | 8624 | `538aed5c920a0be328cef2da56b2085286c5de815939be54ea07b9774daea529` | 返回型 launcher（非 bcache／ret-v*） |
| `bins/init_dram_efex.bin` | 29568 | `0493812f91497d5627caa846a4742fc4b8940b893ecedad1c731434ac7e76eaf` | DRAM 初始化（冷链第 1 步） |
| `bins/SHA256SUMS.rtthread.txt` | — | — | 原样复制自构建产出目录 |
| `bins/SHA256SUMS.c907-boot.txt` | — | — | 原样复制自构建产出目录 |

stock 只读参考副本（来源 `../stock-fullcopy/bins/`，未改动，仅核对）：`c907_boot_efex.bin`、`init_dram_efex.bin`（同 `demo-pack-b` 的 stock 副本做法）。

### 2.3 串口证据（banner 出后补入）

| 路径 | 用途 |
|---|---|
| `logs/uart-knife3-board-*.log` | COM6 @115200 8N1 被动监听日志，记录 `RT-Thread 5.4.0` banner、`STEP4-HEARTBEAT`、`msh >` 出现时刻 |
| `STATUS-C907-KNIFE3-BOARD.md` | Win 板上刀3 的每步 rc 与判定（OK / NOFEL / NOBANNER） |
| `cold-chain-card.md` | 可照抄的 6 步 xfel 冷链（init_dram → payload → launcher → exec），来源 `STATUS-C907-KNIFE3-READY.md` §2 |

### 2.4 docs / scripts / logs（可留空占位，参考 demo-pack-b）

- `docs/`：放冷链 diff／patch（若有），demo-pack-b 为空占位即可
- `scripts/`：放 xfel 冷链脚本（PS1，如 `run_uart_rtt.ps1` 的引用），demo-pack-b 为空占位即可
- `logs/`：本刀无新日志，空占位

---

## 3. README 大纲（`c907-knife3-pack/README.md`）

结构仿照 `demo-pack-b/README.md`（≤50 行）：

1. **标题**：档 C · C907 单核 rt-smart 刀3 上板产物包（HOST 证据 + 串口日志）
2. **一句话说明**：打包刀3 首次跑通 `rtthread.bin` 上板后的 bin、SHA 与串口证据；不是上游 PR，不是 RT-Thread BSP 完成。
3. **边界（读前必看）**：HOST-only 打包 + 串口证据；未做 git push／gh／上传；真板 banner 已采（COM6），未做 E907／SMP。
4. **V861 vs V881**：板名 V861 / Avaota F2；`xfel version` 实测 `AWUSBFEX ID=0x00191800(V881)`。
5. **地址表（禁踩区）**：`0x100000` init_dram／launcher；`0x40000000` rtthread.bin（禁踩，与 SMP 保留区重叠）；UART 基址 `0x02500000`、PH9/PH10 mux5。
6. **包内 bin 与 SHA256**：表格列 rtthread.bin / c907_boot_efex.bin / init_dram_efex.bin 及字节与 SHA。
7. **串口判据**：banner `RT-Thread 5.4.0` + `STEP4-HEARTBEAT n=...` + `msh >`（COM6 115200 8N1），最低通过＝banner 出。
8. **下游来源与下一刀**：引用 `STATUS-C907-KNIFE3-READY.md`、`STATUS-C907-KNIFE3-BOARD.md`、`L2-VS-STILLZERO` 结论；下一刀 L3（用户态 rootfs / finsh 交互）。
9. **致谢**：SyterKit 作者 BernardXiong（YuzukiHD）与上游维护者；许可见 `NOTICE`。

---

## 4. LAYOUT（加载地址表，禁踩区）

| 用途 | 地址 | 产物 | 说明 |
|---|---|---|---|
| DRAM 初始化 | `0x00100000` | `bins/init_dram_efex.bin` | 冷链第 1 步 write+exec，DRAM 签名 `0x40000000`=`0x01234567` |
| payload（内核） | `0x40000000` | `bins/rtthread.bin` | sha `a8d0f5a8…`，text@0x40000000，Entry@0x40000040 |
| 返回型 launcher | `0x00100000` | `bins/c907_boot_efex.bin` | 冷链第 4 步 write+exec，ENTRY `0x40000000` |
| UART 基址 | `0x02500000` | — | PH9/PH10 mux5，COM6 115200 8N1 |
| 禁踩 | `0x40000000` | — | SMP 保留区，rtthread.bin 落点，勿越界覆盖 |

---

## 5. PLAN（刀3 打包后的下一刀，参考 demo-pack-b PLAN.md 风格）

- [x] P-1 打包 bin + SHA + 串口证据（本 NOTES 大纲；banner 出后落盘）
- [ ] P-2 冷链命令卡落盘 `cold-chain-card.md`（可照抄，勿重复造脚本名）
- [ ] P-3 L2 全成功确认：banner + `STEP4-HEARTBEAT` + `msh` 三者皆出（`STATUS-C907-KNIFE3-BOARD.md` MARKER=OK）
- [ ] P-4（之后，不在本包）L3：`smart-rootfs`/`smart-image` 用户态 + finsh 交互；`RT_PIC_RISCV_CLIC_MAX_NR=128` 与 `SUN252IW1_CLIC_NUM_SOURCES=208` 不一致待定；`v861_ccu_pio_enable()` 空实现待确认。

---

## 6. 缺口清单（仅写缺，勿编）

1. 串口证据日志 `logs/uart-knife3-board-*.log` 与 `STATUS-C907-KNIFE3-BOARD.md` —— 等 Win 板上刀3 跑通 banner 后补入。
2. 独立「冷链脚本」文件 —— 目录下仅 PS1 跑车器，以 `STATUS-C907-KNIFE3-READY.md` §2 命令序列为准，脚本化需另行整理。
3. rtthread.bin 用户态 rootfs —— L3 所需，本刀以 L2 为阶段目标可暂不阻塞。
4. `RT_PIC_RISCV_CLIC_MAX_NR=128` 与 `SUN252IW1_CLIC_NUM_SOURCES=208` 不一致；`v861_ccu_pio_enable()` 空实现 —— 均非 L2 阻塞，需记录。

---

末行：MARKER=C907-KNIFE3-UPLOAD-PACK-OK
