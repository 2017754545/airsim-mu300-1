# 7.3.1 内核下的蓝牙：现状、依赖与驱动缺陷

结论先说：**内核支持 BlueZ，缺的是 Unisoc 的传输层驱动 `sprdbt_tty`**。7.3.1 镜像里没有
这个模块，也没有可替代的 H4 传输，所以 `btattach`、`bluetoothd` 装了也用不起来。

本文记录判断依据、重编方法，以及审读驱动源码时发现的 5 个缺陷。文中不含任何设备
标识（BD_ADDR 等）。

## 1. 判断依据（实机只读检查）

内核配置（`/proc/config.gz`，7.3.1）：

```
CONFIG_BT=y              CONFIG_BT_BREDR=y         CONFIG_BT_LE=y
CONFIG_BT_RFCOMM=y       CONFIG_BT_RFCOMM_TTY=y    CONFIG_BT_BNEP=y
CONFIG_BT_HIDP=m         CONFIG_BT_HCIUART=y       CONFIG_BT_HCIUART_H4=y
CONFIG_RFKILL=y
```

`HCIUART_H4` 正好对上厂商脚本 `bluetooth-start` 里的
`btattach -B /dev/ttyBT0 -P h4`。

但传输层是空的：

| 检查项 | 结果 |
|---|---|
| `/dev/ttyBT0` | 不存在 |
| `/sys/class/bluetooth` | 空 |
| `lsmod` | 没有 `sprdbt_tty` |
| 设备树 | `sprd-marlin3/sprd-mtty`：`compatible="sprd,mtty"`、`sprd,name="ttyBT"` 存在 |
| 驱动绑定 | `/sys/bus/platform/devices/sprd-marlin3:sprd-mtty` 没有 driver 链接 |
| `rfkill` | 只有 `phy0`（wlan），没有 bluetooth 条目 |

也就是说：设备树节点在、内核框架在、**中间那层没加载**。

## 2. 为什么不能直接拿旧模块

6.18.54 与 5.4.254 的社区包都带了 `sprdbt_tty.ko`（`depends=wcn_bsp`），但那两个包能开
蓝牙的原因只是"随包带了与内核严格匹配的模块"，不是内核配置更强。`vermagic` 是
`6.18.54` / `5.4.254-…`，`insmod` 到 7.3.1 上会被直接拒绝 —— 和当初音频模块一样。

所以要么给 7.3.1 重编，要么退回 6.18.54（代价是丢掉 7.3.1 上的音频改造）。

## 3. 重编方法（不随本仓库分发源码）

驱动源码属于 Unisoc vendor 内核（`modules/sprdbt_tty`，`tty-pcie/` 下
`tty.c` / `rfkill.c` / `lpm.c` / `alignment/sitm.c`，Kbuild 里
`BSP_BOARD_UNISOC_WCN_SOCKET := pcie`）。**由于许可证原因，本仓库不包含这些文件**，
下面只记录步骤与结论，方便有 vendor 源码的人复现。

```sh
make -C <kernel-src> M=$PWD ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     KBUILD_EXTRA_SYMBOLS=<wcn_bsp build dir>/Module.symvers modules
```

`KBUILD_EXTRA_SYMBOLS` 是必须的：不传时 modpost 报 6 个未定义符号 ——
`get_wcn_bus_ops` / `start_marlin` / `stop_marlin` / `marlin_set_wakeup` /
`marlin_set_sleep` / `wcn_reset_notifier_list`，它们都由 `wcn_bsp` 以
`EXPORT_SYMBOL_GPL` 导出。

值得注意的是：**总线操作不是通过导出符号拿的**，而是 `get_wcn_bus_ops()` 返回一张
操作表，所以 `kallsyms` 里看不到 `mtty_*` / `sprdwcn_bus_list_alloc` 并不代表
`wcn_bsp` 缺东西。只要这 6 个符号在，模块就能链接成。

产物 `sprdbt_tty.ko` 的 `vermagic` 必须与设备一致（`7.3.1 SMP preempt mod_unload
aarch64`），且 `depends=wcn_bsp`；装载顺序是 `wcn_bsp` → `sprdbt_tty`。

## 4. 审读源码发现的 5 个缺陷

这些是**驱动自身**的问题，与 7.3.1 无关，旧内核上同样存在。前两个会直接导致蓝牙
"起不来"，后三个是异常路径下的泄漏/越界。

| # | 位置 | 问题 | 建议 |
|---|---|---|---|
| 1 | `rfkill.c` | `rfkill_init_sw_state(bt_rfk, true)` 让适配器**默认处于软阻塞**。OpenWrt 上 busybox 没有 `rfkill`，软件源里也没有单独的 `rfkill` 包，于是永远解不开，BlueZ 一直认为适配器被关 | 初始化为 `false`，或在首次 `btattach` 前显式 unblock |
| 2 | `tty.c` `mtty_write()` | `dma_alloc_coherent` 失败时直接 `return -ENOMEM`，**漏了 `up(&sem_id)`**，也没把 list 还回去。每失败一次少一个额度，累计 63 次后写路径永久阻塞 | 失败分支补 `sprdwcn_bus_list_free()` + `up()` |
| 3 | `tty.c` `mtty_close()` | 先 `mtty_dma_buf_free()` 再 `sprdwcn_bus_chn_deinit()`；中间若有 RX 回调进来，会往已释放的 DMA 缓冲写（use-after-free） | 改成 置状态 → 摘通道 → 再释放缓冲 |
| 4 | `tty.c` `mtty_tx_cb()` | 每个 mbuf 都用 `head->phy` 去 `dma_free_coherent`，`num > 1` 时释放的是错地址 | 改用 `pos->phy` |
| 5 | `alignment/sitm.c` `sitm_ini()` | `kmalloc` 后不判空就 `memset`；`kfifo_alloc` 失败只打印，留下半初始化的 `rd` | 补判空 + 失败回收并返回 `-ENOMEM` |

另外几处只在极端情况下触发，记录但不建议贸然改：`tty_register_driver()` 排在锁/链表
初始化之前；`mtty_open()` 不检查 `start_marlin()`/`chn_init()`/`dma_buf_alloc()` 的
返回值；`mtty_close()` 从不 `stop_marlin()`（厂商设计）；`bluesleep_init()` 的返回值
被忽略，而 `bt_wakeup_host()` 会无条件 `__pm_relax()`。

## 5. 与 WiFi 共存的冲突核对

蓝牙和 WiFi 共用一颗 Marlin3，走同一条 WCN 总线，所以这是最容易出事的地方。核对结果：

- **通道不撞**：WiFi 用 TX 4/5、RX 7/9/11；蓝牙用 TX0=1、RX=2。
  `BT_TX_CHANNEL1=4` 虽然和 WiFi 的 TX 命令通道同号，但代码里从未 `chn_init` 它；
- **GNSS 不抢**：`platform/gnss/*` 没有注册 mchn，且 gnss 模块未加载；
- **上电不复位**：`start_marlin(MARLIN_BLUETOOTH)` 在芯片已为 WiFi 上电时走
  "marlin have open" 分支，只置位并通知，不会复位芯片，不会打断 WiFi AP。真正上电
  发生在 `open /dev/ttyBT0` 时。

## 6. 状态

- 已按上述修补重编通过（modpost 干净，导出符号全部解析），产物未随本仓库分发；
- **未在实机装载**：装载会碰 WCN 电源与 WiFi 共存，需要设备主人在场时进行，并且要先
  有 `--disable` 的回退路径；
- 首次装载建议观察：`dmesg | grep -i mtty` 是否出现 open/power on；`/dev/ttyBT0` 是否
  出现；`/sys/class/rfkill` 是否多出 `type=bluetooth` 的条目；
- 若 `btattach` 报 command timeout，通常是 PSKey/RF 参数没写进去（`mu300-bt-init`
  那一步），而不是驱动没起来。

## 7. 对上游的建议

蓝牙不属于本仓库的职责范围（README 明确系统/内核保持原样），所以这里只留结论。真要
推进，建议路径是：把第 4 节的 5 个缺陷作为 bug report 提给 Unisoc vendor 内核维护方，
或在一个独立仓库里存放"针对某个内核版本的 `sprdbt_tty` 重编脚本 + 补丁"。
