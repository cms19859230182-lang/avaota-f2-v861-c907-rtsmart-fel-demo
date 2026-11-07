# TASK · 打并推个人仓：C907 单核 rt-smart 板上验证包（档 C）

机位 box。Harness=pi。目标：新建或更新公开个人仓，把刀3 板上 banner+msh 证据打成可复现包并 push。

## 已知事实
- 板上已出 RT-Thread 5.4.0 banner + msh（2026-11-07）；uart：本目录 logs/uart-com6.log
- bins/ 三件 sha：
  - rtthread.bin a8d0f5a81ad83fdedb0db3229232073f0e7e50b22054caa9de9245476e3707fe
  - c907_boot_efex.bin 538aed5c920a0be328cef2da56b2085286c5de815939be54ea07b9774daea529
  - init_dram_efex.bin 0493812f91497d5627caa846a4742fc4b8940b893ecedad1c731434ac7e76eaf
- 对照笔记在 docs/
- 旧档 B 仓：https://github.com/cms19859230182-lang/avaota-f2-v861-coexist-host-demo （共存 HOST，勿混）
- 本包主题：**C907 单核 rt-smart FEL 冷链＋串口证据**，不是共存

## 做
1. 在本目录整理仓库：README（中文为主）写清：板型 Avaota F2／V861、冷链五步、COM6 115200、成功判据 banner+msh、局限（无 STEP4-HEARTBEAT／未做 rootfs）
2. 纳入 bins/＋logs/＋docs/；写 SHA256SUMS
3. 用已有 GH_TOKEN 推到 cms19859230182-lang 下新仓，建议名 `avaota-f2-v861-c907-rtsmart-fel-demo`（若重名则加后缀）；public
4. 交付本目录 DONE-C907-DEMO-PACK-C.md：仓 URL、tip sha、MARKER=C907-DEMO-PACK-C-OK

## 禁止
勿推 RT-Thread/rt-thread 上游；勿改 #11979；勿上板；勿把共存 bin 当本包主 payload。
