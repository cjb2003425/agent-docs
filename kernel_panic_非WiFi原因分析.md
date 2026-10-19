# kernel_panic.wmv — NT72690 非Wi-Fi原因分析

仅保留NT72690平台中未归入直接Wi-Fi调用栈的 **8条记录**。这些分类不代表已经绝对排除上游内存破坏的其他来源。根因尚需匹配的vmlinux、模块和原始日志验证。

## 保留记录概览

| 原ROW | 检测点/路径 | 方向 |
|---|---|---|
| 4 | `rb_erase+0x304/0x328` | 定时器/调度 |
| 6 | `do_undefinstr+0x230/0x234` | 显示/GPU关联异常（需复核） |
| 56 | `z_erofs_vle_normalaccess_readpages+0x28c/0x28c` | 内核栈破坏 |
| 57 | `tick_nohz_next_event+0xb0/0x164` | 定时器/调度 |
| 79 | `scm_recv+0x158/0x158` | 内核栈破坏 |
| 80 | `do_undefinstr+0x230/0x234` | 显示/GPU关联异常（需复核） |
| 106 | `__tep_close+0x14/0x440 ;` | TCP/socket对象 |
| 150 | `PPU_DFSM_Init+0x8d/0x2a4 [ndp_alsa]（按截图复核）` | 音频tasklet：PC未对齐 |

## 逐条证据与原因判断

### ROW_4_COL_11 — 定时器/调度
源CSV正文行 362–441；标题视频位置 00:40.0；异常时间戳 760.820698。

**直接证据：** 定时器红黑树、hrtimer回调/取消，或 tick_nohz 路径出现非法对象/PC。

```text
CSV 392 / 视频 00:40.5: <1>[ 760.820658] [ CO) Unable to handle kernel paging request at virtual address 004fffc0146eb7c8
CSV 402 / 视频 00:40.5: <O0>[ 760.820698] [ CO] Internal error: Oops: 96000004 [#1] PREEMPT SMP
CSV 404 / 视频 00:40.5: <4>[ 760.820788] [ CO] CPU: O PID: 0 Comm: swapper/0 Tainted: P ° 5.4.283 #1
CSV 407 / 视频 00:40.5: <4>[ 760.820821][ CO] pe : rb_erase+0x304/0x328
CSV 408 / 视频 00:40.5: <4>[ 760.820830][ cO] lr : timerqueue_del+0x44/0x68
```

**判断：** 优先排查定时器对象释放后使用、重复加入/删除、回调与销毁竞态，或其他模块提前破坏指针。`rb_erase` 属于通用树操作，只有调用栈经过 cfg80211 才归入Wi-Fi；定时器树应独立分析。

**验证：** 启用 DEBUG_OBJECTS_TIMERS、KASAN/SLUB调试，记录timer地址、回调、激活/取消/释放栈，核对取消与释放的同步。

### ROW_6_COL_11 — 显示/GPU关联异常（需复核）
源CSV正文行 548–636；标题视频位置 00:49.0；异常时间戳 525.182730。

**直接证据：** 未定义指令/异常跳转的调用栈中出现PQ、DRM或Mali路径。

```text
CSV 576 / 视频 00:51.0: <2>[ 525.182716][ C3) kernel BUG at arch/arm64/kernel/traps.c:405! . . ” ‘ ‘ . :
CSV 577 / 视频 00:51.0: <O>[ 525.182730][ C3] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 579 / 视频 00:51.0: <4>[ 525.182843][ C3] CPU: 3 PID: 118 Comm: kworker/u8:1 Tainted: P ° §.4.283 #1
CSV 583 / 视频 00:51.0: <4>[ 525.182917][ C3) pe : do_undefinstr+0x230/0x234
CSV 584 / 视频 00:51.0: <4>[ 525.182921][ C3] lr : do_undefinstr+0xb8/0x234
```

**判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**验证：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### ROW_56_COL_11 — 内核栈破坏
源CSV正文行 5157–5280；标题视频位置 06:09.5；异常时间戳 252943.300710。

**直接证据：** 日志明确出现 `stack-protector: Kernel stack is corrupted in:`。

```text
CSV 5251 / 视频 06:19.5: <0>[252943.300710] [T25951] Kernel panic - not syncing: stack-protector: Kernel stack is ae in: z_erofs_vle_normalaccess_readpages+0x28c/0x28c
CSV 5252 / 视频 06:20.0: <2>[252943.300728] [T25951] [STBCIPC_panic]Caution!! This Only For Kernel Panic
CSV 5254 / 视频 06:19.5: <4>[252943.302751] [T25951] CPU: 1 PID: 25951 Comm: copy netflix_cr Tainted: P ° 5.4.283 #1
```

**判断：** 可以确认栈保护检查失败。报告的函数是检测点，写坏栈的代码可能更早，不能仅凭函数名断定其自身越界。

**验证：** 保留精确版本 vmlinux/.config；使用KASAN、栈保护和编译器边界检查定位最早的写入，检查局部数组、拷贝长度、跨线程/IRQ对栈内存的引用。

### ROW_57_COL_11 — 定时器/调度
源CSV正文行 5283–5384；标题视频位置 06:22.5；异常时间戳 3009.988144。

**直接证据：** 定时器红黑树、hrtimer回调/取消，或 tick_nohz 路径出现非法对象/PC。

```text
CSV 5284 / 视频 06:22.5: <2>[ 3009.965595][ TO] [STBCIPC_panic]Caution!! This Only For Kernel Panic
CSV 5286 / 视频 06:22.5: <4>[ 3009.965672] [ TO) CPU: 1 PID: O Comm: swapper/1 Tainted: P D ° 5.4.283 #1
CSV 5289 / 视频 06:22.5: <4>[ 3009.965700] [ TO) pe : tick_nohz_next_event+0xb0/0x164
CSV 5319 / 视频 06:26.5: <O>[ 3009.988144][ TO] Internal error: Oops: 96000044 [#3] PREEMPT SMP
CSV 5321 / 视频 06:26.5: <4>[ 3009.988200] [ TO] CPU: 2 PID: O Comm: swapper/2 Tainted: P D ° 5.4.283 #1
CSV 5324 / 视频 06:26.5: <4>[ 3009.988220] [ TO) pe : tick_nohz_next_event+0xb0/0x164
CSV 5352 / 视频 06:30.0: <0>[ 3010.010736][ TO] Internal error: Oops: 96000044 [#4] PREEMPT SMP
CSV 5354 / 视频 06:30.0: <4>[ 3010.010786) [ TO] CPU: 0 PID: 0 Comm: swappet/0 Tainted: P . D ° . 5. #1, : bcs af . i
```

**判断：** 优先排查定时器对象释放后使用、重复加入/删除、回调与销毁竞态，或其他模块提前破坏指针。`rb_erase` 属于通用树操作，只有调用栈经过 cfg80211 才归入Wi-Fi；定时器树应独立分析。

**验证：** 启用 DEBUG_OBJECTS_TIMERS、KASAN/SLUB调试，记录timer地址、回调、激活/取消/释放栈，核对取消与释放的同步。

### ROW_79_COL_11 — 内核栈破坏
源CSV正文行 7286–7407；标题视频位置 08:34.5；异常时间戳 28959.987779。

**直接证据：** 日志明确出现 `stack-protector: Kernel stack is corrupted in:`。

```text
CSV 7382 / 视频 08:41.5: <0>[28959.987779][ 1485] Kernel panic - not syncing: stack-protector: Kernel stack is corrupted in: scm_recv+0x158/0x158
CSV 7383 / 视频 08:41.5: <2>[28959.987797][ 1485] [STBCIPC_panic]Caution!! This Only For Kernel Panic A F x Z P é
CSV 7385 / 视频 08:41.5: <4>[(28959.989818][ 1T485] CPU: 3 PID: 485 Comm: busybox Tainted: P ° 5.4.283 #1
```

**判断：** 可以确认栈保护检查失败。报告的函数是检测点，写坏栈的代码可能更早，不能仅凭函数名断定其自身越界。

**验证：** 保留精确版本 vmlinux/.config；使用KASAN、栈保护和编译器边界检查定位最早的写入，检查局部数组、拷贝长度、跨线程/IRQ对栈内存的引用。

### ROW_80_COL_11 — 显示/GPU关联异常（需复核）
源CSV正文行 7410–7515；标题视频位置 08:43.0；异常时间戳 16606.318313。

**直接证据：** 未定义指令/异常跳转的调用栈中出现PQ、DRM或Mali路径。

```text
CSV 7473 / 视频 08:45.0: <2>[16606.318306] [T12039] kernel BUG at arch/arm6é4/kernel/traps.c:405!
CSV 7474 / 视频 08:45.0: <O0>[(16606.318313] [T12039] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 7480 / 视频 08:47.5: <4>[16606.318550] [T12039] pc : do_undefinstr+0x230/0x234
CSV 7481 / 视频 08:47.5: <4>[16606.318558] [T12039] lr : do_undefinstrt+0xb8/0x234
```

**判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**验证：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### ROW_106_COL_11 — TCP/socket对象
源CSV正文行 9787–9876；标题视频位置 10:40.0；异常时间戳 141757.549677。

**直接证据：** TCP close/recv路径出现非法地址访问。

```text
CSV 9839 / 视频 10:43.0: <1>[(141757.549642][ T2451] Unable to handle kernel paging request at virtual address 000080ed804alela
CSV 9849 / 视频 10:43.0: <0>[141757.549677] [ T2451] Internal error: Oops: 96000044 [#1] PREEMPT SMP
CSV 9851 / 视频 10:44.0: <4>[141757.549763] [ T2451] CPU: 3 PID: 2451 Comm: HTTPREQUEST_MAN Tainted: P fe) 5.4.283 #1
CSV 9854 / 视频 10:44.0: <4>[141757.549796] [ T2451] pc : __tep_close+0x14/0x440 ;
CSV 9855 / 视频 10:44.0: <4>[141757.549804] [ T2451] lr : tcp_recvmsg+0x448/0x9e0 ; G . . . ; :
```

**判断：** socket/tcp对象指针或生命周期受损方向较强；发送/接收应用线程只是当时上下文，不能证明应用直接写坏内核。

**验证：** 定位故障指令基址寄存器，检查sk引用、close与recv竞争、析构后使用；用KASAN/SLUB记录对象分配与释放栈。

### ROW_150_COL_11 — 音频tasklet：PC未对齐
源CSV正文行 13817–13918；标题视频位置 14:20.5；异常时间戳 107671.950200。

**直接证据：** SP/PC alignment exception；PC为 PPU_DFSM_Init+0x8d，LR关联tasklet_action_common。

```text
CSV 13886 / 视频 14:23.5: <0>[107671.950200] [ €2) Internal error: SP/PC alignment exception: 8a000000 [#1] PREEMPT SMP - © . . . :
CSV 13888 / 视频 14:23.5: 4<4>[107671.950295] [ C2] CPU: 2 PID: 20 Comm: ksoftirgd/2 Tainted: P ° 5.4.283 #1
CSV 13892 / 视频 14:23.5: <4>[107671.950486] [ C2) lr : tasklet_action_common+0x104/0x140
```

**判断：** 确认执行目标未满足ARM64指令对齐。优先检查tasklet回调指针、对象生命周期、越界写和代码/模块有效性，不能将符号最近邻直接解释为音频算法自身错误。

**验证：** 记录tasklet对象、func指针和初始化/销毁栈，核对实际ELR低位，使用匹配模块检查回调是否在预期函数入口。

## 定位所需资料

使用NT72690实际固件对应的vmlinux、System.map、.config和厂商模块。日志中该平台为Linux 5.4.283；应以厂商构建为准。优先在复现环境中开启KASAN、SLUB_DEBUG及timer对象调试，定位首次异常写入或生命周期违规。正文为视频OCR重建，地址及函数偏移需复核。
