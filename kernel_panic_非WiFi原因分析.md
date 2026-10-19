# kernel_panic.wmv — 非 Wi-Fi 故障原因分析

共 118 条记录暂未归入直接Wi-Fi调用栈；其中有的归属仍待确认。这份报告区分已观察到的故障机制与根因假设，不能替代原始日志和厂商符号定位。

## 主要排查方向

### 显示驱动：cache 维护
记录 73 条：ROW 10, 13, 19, 21, 23, 24, 25, 27, 28, 29, 30, 34, 35, 36, 37, 38, 41, 44, 46, 47, 49, 62, 64, 66, 70, 72, 74, 77, 82, 84, 86, 87, 88, 90, 91, 92, 96, 97, 99, 101, 109, 110, 111, 114, 115, 117, 119, 120, 124, 131, 132, 133, 134, 135, 137, 140, 141, 157, 158, 159, 160, 163, 164, 165, 168, 170, 172, 173, 174, 175, 178, 179, 180。

**直接证据：** `__inval_dcache_area → Chip_Inv_Cache_Range`，上层常见 `Mdrv_XC_HSE_Control / SetDSHDRInfo / Hal_DMS_SetMVOP_Task`。

**原因判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**如何确认：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### 显示驱动：Obj/PQ处理
记录 21 条：ROW 8, 12, 14, 26, 33, 43, 50, 65, 67, 71, 73, 83, 95, 107, 116, 123, 143, 161, 166, 169, 171。

**直接证据：** `MDrv_Obj_OP_Process`，调用路径包含 `KDrv_XC_OBJ_Wrapper_Process / KDrv_XC_PQ_SetVACCapability / KDrv_XC_BottomIntHandler`。

**原因判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**如何确认：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### 定时器/调度
记录 4 条：ROW 4, 32, 45, 57。

**直接证据：** 定时器红黑树、hrtimer回调/取消，或 tick_nohz 路径出现非法对象/PC。

**原因判断：** 优先排查定时器对象释放后使用、重复加入/删除、回调与销毁竞态，或其他模块提前破坏指针。`rb_erase` 属于通用树操作，只有调用栈经过 cfg80211 才归入Wi-Fi；定时器树应独立分析。

**如何确认：** 启用 DEBUG_OBJECTS_TIMERS、KASAN/SLUB调试，记录timer地址、回调、激活/取消/释放栈，核对取消与释放的同步。

### 蓝牙驱动：BRK/收发
记录 4 条：ROW 11, 48, 156, 176。

**直接证据：** `Unexpected kernel BRK exception at EL1`，调用栈/返回地址关联 `btmtk_usb_meta_send_data [btmtk_usb_7663]`。

**原因判断：** 确认软件BRK异常，蓝牙USB收发路径关联明确；具体是厂商断言、异常跳转还是代码损坏，需要精确PC和模块反汇编，不能等同Wi-Fi BSS问题。

**如何确认：** 将异常PC与btmtk模块加载基址对齐，定位BRK指令及前置条件，保留HCI/USB状态、URB完成和断连、重置事件。

### 显示/GPU关联异常（需复核）
记录 3 条：ROW 6, 31, 80。

**直接证据：** 未定义指令/异常跳转的调用栈中出现PQ、DRM或Mali路径。

**原因判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**如何确认：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### 内核栈破坏
记录 2 条：ROW 56, 79。

**直接证据：** 日志明确出现 `stack-protector: Kernel stack is corrupted in:`。

**原因判断：** 可以确认栈保护检查失败。报告的函数是检测点，写坏栈的代码可能更早，不能仅凭函数名断定其自身越界。

**如何确认：** 保留精确版本 vmlinux/.config；使用KASAN、栈保护和编译器边界检查定位最早的写入，检查局部数组、拷贝长度、跨线程/IRQ对栈内存的引用。

### 等待队列/进程清理
记录 1 条：ROW 15。

**直接证据：** `__wake_up_common → __wake_up_pollfree → signalfd_cleanup`，发生于进程/信号处理结构清理。

**原因判断：** 等待队列链表或回调对象失效、pollfree/signalfd生命周期竞态是主要方向。当前清理线程是触发上下文，不能据此归因于应用。

**如何确认：** 检查等待项注册、撤销、sighand释放和RCU宽限期；启用DEBUG_LIST/KASAN并收集损坏节点前后指针。

### 网络：netlink队列
记录 1 条：ROW 61。

**直接证据：** 32位ARM的 skb_queue_tail，经 netlink_dump/recvmsg；同一记录夹有另一个线程的Mali栈。

**原因判断：** 队列链表尾指针/对象失效。寄存器r0为0，故障指令为str r4,[r0]的方向可与源画面核对。并行Mali日志不足以证明GPU就是写坏队列的来源。

**如何确认：** 检查sk_buff_head初始化、入队/出队和销毁锁，记录队列prev/next及生命周期；将并发CPU日志按PID与时间排序。

### 显示驱动：UCD断言
记录 1 条：ROW 85。

**直接证据：** kernel BUG at mdrv_ucd.c:1157，PC=MApi_UCD_OP_Process+0x28e4，栈经KDrv_XC_Resume。

**原因判断：** 确认厂商UCD实现主动触发BUG，具体失败条件只能查同一构建mdrv_ucd.c第1157行。优先核对resume时UCD状态、参数/表尺寸、资源初始化和版本兼容。

**如何确认：** 取得对应厂商源码和kdrv_alg模块，读取实际BUG条件及其输入；记录resume前后对象、配置、初始化和释放顺序，避免仅删除BUG掩盖状态异常。

### TCP/socket对象
记录 1 条：ROW 106。

**直接证据：** TCP close/recv路径出现非法地址访问。

**原因判断：** socket/tcp对象指针或生命周期受损方向较强；发送/接收应用线程只是当时上下文，不能证明应用直接写坏内核。

**如何确认：** 定位故障指令基址寄存器，检查sk引用、close与recv竞争、析构后使用；用KASAN/SLUB记录对象分配与释放栈。

### 调谐器驱动：恢复/ioctl
记录 1 条：ROW 108。

**直接证据：** `MI_DEV_TUNER_ResumeEarly [mik]`，经厂商ioctl分发，访问低地址。

**原因判断：** 低地址访问支持对象为空或失效后加字段偏移。可能涉及resume初始化次序、ioctl参数、并发关闭；Binder线程名不是根因证据。

**如何确认：** 反汇编mik.ko，确认基址、索引和写入指令；校验用户拷贝返回值、嵌套指针、compat结构尺寸，检查resume与close/release同步。

### 内存保护单元报警
记录 1 条：ROW 112。

**直接证据：** 显式 `MIUP hit panic`；栈经 `mtk_miup_irq → panic`。

**原因判断：** 内存保护硬件报告某个client访问受保护区间，并被驱动升级为panic。应检查client地址/权限/缓冲区配置；被中断的Binder调用不等于肇事DMA client。

**如何确认：** 用当前SoC的MIUP client表解码CID，记录保护block范围、读写类型、触发master和DMA/IOMMU映射；核对地址截断、边界、保护配置与释放顺序。

### 视频监控线程：对象失效
记录 1 条：ROW 130。

**直接证据：** 32位ARM _MI_VMON_DecMonTask [mik]，之前出现VDEC deinit、buffer remove与cannot find buf_handle。

**原因判断：** 监控线程读取无效对象/字段，解码器关闭与监控任务竞态值得优先排查，但前面的资源释放日志本身不是证明。

**如何确认：** 反汇编mik对应指令并核对r4与FAR，检查VDEC关闭、VMON停止/等待完成、上下文释放和信号量的顺序。

### 内核链表损坏
记录 1 条：ROW 136。

**直接证据：** `__list_del_entry_valid` 等链表一致性检查失败。

**原因判断：** 可确认链表前后关系或节点状态异常；常见候选为重复删除、释放后使用、无锁并发修改或越界覆盖，调用者才决定归属。

**如何确认：** 保留首次WARN前的节点、prev/next值和插入/删除栈；启用DEBUG_LIST/KASAN，核对锁与生命周期。

### 音频tasklet：PC未对齐
记录 1 条：ROW 150。

**直接证据：** SP/PC alignment exception；PC为 PPU_DFSM_Init+0x8d，LR关联tasklet_action_common。

**原因判断：** 确认执行目标未满足ARM64指令对齐。优先检查tasklet回调指针、对象生命周期、越界写和代码/模块有效性，不能将符号最近邻直接解释为音频算法自身错误。

**如何确认：** 记录tasklet对象、func指针和初始化/销毁栈，核对实际ELR低位，使用匹配模块检查回调是否在预期函数入口。

### 存储完成/SLAB：诊断期间再异常
记录 1 条：ROW 155。

**直接证据：** kmem_valid_obj经mem_dump_obj和异常打印路径；更早栈含__slab_free、kmem_cache_free、dm_io_dec_pending、clone_endio和verity_work。

**原因判断：** 当前PC落在异常诊断阶段，不能把kmem_valid_obj当作最初根因。应优先核对原始释放故障，候选为错误释放、重复完成、重复释放或slab元数据损坏；没有证据直接认定存储介质坏。

**如何确认：** 获取首个DataAbort及此前完整日志，对dm-io对象分配、引用和endio逐次编号，启用KASAN/SLUB及双重释放检查。

### 显示驱动：MCNR上下文
记录 1 条：ROW 167。

**直接证据：** MDrv_NR_SetMcnrSwReg+0x658，经MDrv_NR_MappingMcnrContext和scaler线程；FAR为低地址0x90。

**原因判断：** 低地址写入支持空/无效MCNR上下文及字段偏移的方向。仍需用故障指令和寄存器确认，重点检查上下文映射、配置切换、IRQ与释放顺序。

**如何确认：** 反汇编kdrv_alg中的故障store，核对基址与0x90关系，记录MCNR context创建/映射/失效及各线程引用。

## cache 维护现场的具体说明

ROW_10 的日志给出 ESR=0x96000147、CM=1，故障页PTE为0；调用点是显示驱动的缓存维护区间操作。`Code` 中的故障指令可读为 `d50b7e21`，对应对 x1 的 cache clean/invalidate，需以匹配的厂商模块再核验。该类问题首先是访问地址/映射不成立，不能据此推断eMMC故障。

上游5.4.302实现会处理区间起止的部分cache line；vendor模块可能修改过实现，不能直接把其偏移映射到上游源码。参考：[Linux v5.4.302 cache.S](https://github.com/gregkh/linux/blob/v5.4.302/arch/arm64/mm/cache.S)。DMA与CPU的地址、映射及所有权必须按相应API管理，参考：[Linux DMA API](https://www.kernel.org/doc/html/v5.9/core-api/dma-api.html)。

## 每条记录的证据

### ROW_4_COL_11 — 定时器/调度
源正文行 362–441；标题视频位置 00:40.0；异常时间戳 760.820698（fatal）。
PC/检测点（OCR）：`rb_erase+0x304/0x328`。

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
源正文行 548–636；标题视频位置 00:49.0；异常时间戳 525.182730（fatal）。
PC/检测点（OCR）：`do_undefinstr+0x230/0x234`。

```text
CSV 576 / 视频 00:51.0: <2>[ 525.182716][ C3) kernel BUG at arch/arm64/kernel/traps.c:405! . . ” ‘ ‘ . :
CSV 577 / 视频 00:51.0: <O>[ 525.182730][ C3] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 579 / 视频 00:51.0: <4>[ 525.182843][ C3] CPU: 3 PID: 118 Comm: kworker/u8:1 Tainted: P ° §.4.283 #1
CSV 583 / 视频 00:51.0: <4>[ 525.182917][ C3) pe : do_undefinstr+0x230/0x234
CSV 584 / 视频 00:51.0: <4>[ 525.182921][ C3] lr : do_undefinstr+0xb8/0x234
```

**判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**验证：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### ROW_8_COL_11 — 显示驱动：Obj/PQ处理
源正文行 744–828；标题视频位置 01:01.0；异常时间戳 205049.690136（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 782 / 视频 01:03.0: <1>(205049.690102][ 1T551] Unable to handle kernel paging request at virtual address ffffffc079abelSc
CSV 793 / 视频 01:04.0: <0>[(205049.690136)[ T5511) Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 796 / 视频 01:04.0: <4>[(205049.690301][ T1551] CPU: 0 PID: 551 Comm: irg/26-scaler Tainted: P Woo §.4.302 #1
CSV 799 / 视频 01:04.0: <4>[205049.690372][ TS51] pe : MDrv_Obj_OP..Process+0x1b8/0x1120 [kdrv_alg] ¢ : : * - ‘ ¢
CSV 800 / 视频 01:04.0: <4>[205049.690406][ 1551] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_10_COL_11 — 显示驱动：cache 维护
源正文行 920–1006；标题视频位置 01:11.5；异常时间戳 79491.808831（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 954 / 视频 01:11.0: <1>[79491.808791] [ T1617] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 966 / 视频 01:13.0: <O>[79491.808831][ T1617] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 969 / 视频 01:13.0: <4>[(79491.808969] [ T1617] CPU: 1 PID: 1617 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 972 / 视频 01:13.0: <4>[(79491.808986][ T1617] pe : _ inwal_dcache_area+0x28/0x58 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_11_COL_11 — 蓝牙驱动：BRK/收发
源正文行 1009–1086；标题视频位置 01:15.5；异常时间戳 319657.023220（fatal）。
PC/检测点（OCR）：`OxffEFFECOOc5d70be`。

```text
CSV 1051 / 视频 01:20.0: <0>[319657.023220][ T4039] Internal error: ptrace BRK handler: £20003e8 [#1] PREEMPT SMP
CSV 1054 / 视频 01:21.0: <4>[319657.023375][ T4039] CPU: 0 PID: 4039-Comm: hci_sock_reader Tainted: P woo 5.4.302 #1
CSV 1057 / 视频 01:21.0: <4>[319657.023397] [ T4039] pce : OxffEFFECOOc5d70be
CSV 1058 / 视频 01:21.0: <4>[319657.023402][ T4039] lr : btmtk_usb_meta_send_data+0Oxdcc/0x13c0 [btmtk_usb_7663]
```

**判断：** 确认软件BRK异常，蓝牙USB收发路径关联明确；具体是厂商断言、异常跳转还是代码损坏，需要精确PC和模块反汇编，不能等同Wi-Fi BSS问题。

**验证：** 将异常PC与btmtk模块加载基址对齐，定位BRK指令及前置条件，保留HCI/USB状态、URB完成和断连、重置事件。

### ROW_12_COL_11 — 显示驱动：Obj/PQ处理
源正文行 1089–1169；标题视频位置 01:23.0；异常时间戳 242141.500917（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 1123 / 视频 01:24.5: <1>[242141.500882][ T546] Unable to handle kernel paging request at virtual address ffffffc0383ee376
CSV 1134 / 视频 01:24.5: <0>[242141.500917][ 1546] Internal error: Oops: 96000007 [#1] PREEMPT SMP
CSV 1137 / 视频 01:24.5: <4>[242141.501047][ 1546] CPU: 0 PID:* 546 Comm: iraq/26-scaler Tainted: P w.o 5.4.302 #1 * : - ¢ =
CSV 1140 / 视频 01:24.5: <4>[242141.501102][ 1546] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 1141 / 视频 01:24.5: <4>[242141.501136][ 1546) lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_13_COL_11 — 显示驱动：cache 维护
源正文行 1172–1255；标题视频位置 01:27.0；异常时间戳 196743.698748（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 1203 / 视频 01:29.5: <1>[196743.698708][ T1660] Unable to handle kernel paging request at virtual address fffFfffc03537e000
CSV 1214 / 视频 01:29.5: <0>[196743.698748] [ T1660) Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 1217 / 视频 01:29.5: <4>[196743.698927] [ T1660] CPU: 1 PID: 1660 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1 .
CSV 1220 / 视频 01:34.0: <4>[196743.698951][ T1660] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 1221 / 视频 01:34.0: <4>[196743.698955] [ T1660] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_14_COL_11 — 显示驱动：Obj/PQ处理
源正文行 1258–1343；标题视频位置 01:35.5；异常时间戳 689142.499499（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 1297 / 视频 01:37.0: <1>[689142.499454][ 1T523] Unable to handle kernel paging request at virtual address ffffffcOSdde34éb
CSV 1308 / 视频 01:37.0: 4<0>[689142.499499][ 1T523] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 1311 / 视频 01:37.0: <4>[689142.499689][ 1T523] CPU: 0 PID: 523 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 1314 / 视频 01:37.0: <4>[689142.499775][ 1523] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 1315 / 视频 01:37.0: <4>[689142.499810][ 1523] lr : MDrv_Obj_OP Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_15_COL_11 — 等待队列/进程清理
源正文行 1346–1434；标题视频位置 01:39.0；异常时间戳 37986.218756（fatal）。
PC/检测点（OCR）：`__wake_up common+0x60/0x188`。

```text
CSV 1384 / 视频 01:42.5: <1>[37986.218732][ T1282] Unable to handle kernel paging request at virtual address 001218d0ffffEL80
CSV 1394 / 视频 01:42.5: <0>[37986.218756] [ T1282] Internal error:| Oops: 96000004 [#1] PREEMPT SMP ; 5 g : Ie E :
CSV 1397 / 视频 01:46.0: <4>[37986.218879][ T1282] CPU: 3 PID: 1282 Comm: rcS.utopia.linu Tainted: P Woo 5.4.302 #1
CSV 1400 / 视频 01:45.5: <4>[37986.218896][ T1282] pc : __wake_up common+0x60/0x188
CSV 1401 / 视频 01:45.5: <4>[37986.218899][ T1282] lr : __ wake_up_common_lock+0x84/0xd0 ; 5 : 5 : i 7
```

**判断：** 等待队列链表或回调对象失效、pollfree/signalfd生命周期竞态是主要方向。当前清理线程是触发上下文，不能据此归因于应用。

**验证：** 检查等待项注册、撤销、sighand释放和RCU宽限期；启用DEBUG_LIST/KASAN并收集损坏节点前后指针。

### ROW_19_COL_11 — 显示驱动：cache 维护
源正文行 1705–1794；标题视频位置 02:04.5；异常时间戳 111181.028694（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 1743 / 视频 02:07.5: <1>[111181.028649][ T1635] Unable to handle kernel paging request at virtual address ffffffc03337e000
CSV 1754 / 视频 02:07.5: <O>[111181.028694][ T1635] Internal error: Oops: 96000147 [#1] PREEMPT SMP : s * : : * 2?
CSV 1757 / 视频 02:07.5: <4>[111181.028910][ T1635] CPU: 1 PID: 1635 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 1760 / 视频 02:07.5: <4>[(111181.028940][ T1635] pc : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 1761 / 视频 02:07.5: <4>[111181.028944][ T1635] lr : Chip_.Inv_Cache_Range+0x94/0x138 [mdrv_system] ; -
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_21_COL_11 — 显示驱动：cache 维护
源正文行 1904–2000；标题视频位置 02:15.5；异常时间戳 125537.194946（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 1945 / 视频 02:20.0: <1>(125537.194890] [ T1258] Unable to handle kernel paging request at virtual address ffffffc033b7e000 < . . °
CSV 1956 / 视频 02:20.0: <0>[125537.194946][ T1258] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 1959 / 视频 02:20.0: <4>[125537.195145] [ T1258] CPU: 1 PID: 1258 Comm: DMS SetMVOP_Tas Tainted: P wo 5.4.302 #1
CSV 1962 / 视频 02:19.0: <4>[125537.195173][ T1258] pc : __inval_dcache_areat0x28/0x58 [mdrv_system]
CSV 1963 / 视频 02:19.0: <4>[125537.195178][ T1258] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_23_COL_11 — 显示驱动：cache 维护
源正文行 2094–2180；标题视频位置 02:27.0；异常时间戳 85536.076297（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2125 / 视频 02:30.0: <1>[85536.076257][ T1669] Unable to handle kernel paging request at virtual address fffFfffc03937e000
CSV 2136 / 视频 02:30.0: <0>[85536.076297] [ T1669] Internal error: Oops: 96000146 [#1] PREEMPT SMP 7 A ‘ . bs i
CSV 2139 / 视频 02:33.5: <4>[85536.076448] [ T1669] CPU: 1 PID: 1669 Comm: DMS SetMVOP_Tas Tainted: P woo §.4.302 #1
CSV 2142 / 视频 02:33.5: <4>[85536.076473] [ T1669] pe : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 2144 / 视频 02:33.5: <4>[85536.076483] [ T1669] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] . * : : . . >
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_24_COL_11 — 显示驱动：cache 维护
源正文行 2183–2268；标题视频位置 02:36.0；异常时间戳 27592.645320（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2214 / 视频 02:39.5: <1>[27592.645250][ T1625] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 2225 / 视频 02:38.5: <0>[27592.645320][ T1625} Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 2230 / 视频 02:37.5: <4>[27592.645554] [ T1625] CPU: 1 PID: 1625 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 2233 / 视频 02:37.5: <4>[27592.645588][ T1625] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 2234 / 视频 02:39.0: <4>[27592.645593][ T1625] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_25_COL_11 — 显示驱动：cache 维护
源正文行 2271–2353；标题视频位置 02:40.0；异常时间戳 17775.126093（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2296 / 视频 02:42.5: <1>[17775.126052][ T1598] Unable to handle kernel paging request at virtual address ffffffc03437e000
CSV 2307 / 视频 02:42.5: <0>[17775.126093][ T1598] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 2310 / 视频 02:45.5: <4>[17775.126255][ T1598] CPU: 1 PID: 1598 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 2313 / 视频 02:45.5: <4>[(17775.126279] [ T1598] pe : _ inval_dcache_area+0x28/0xS8 [mdrv_system] . : : . . . :
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_26_COL_11 — 显示驱动：Obj/PQ处理
源正文行 2356–2439；标题视频位置 02:47.5；异常时间戳 51465.843032（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 2393 / 视频 02:50.0: <1>[51465.842995][ 1527] Unable to handle kernel paging request at virtual address ffffffc078a96855 : y ‘ :
CSV 2404 / 视频 02:50.0: <O>[51465.843032][ 1527] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 2407 / 视频 02:50.0: <4>([(51465.843188][ 1527] CPU: 0 PID: 527 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 2410 / 视频 02:50.0: <4>[(51465.843264][ 1527] pc : MDrv Obj OP Process+0x1b8/0x1120 [kdrv alg]
CSV 2411 / 视频 02:54.0: <4>[51465.843301][ 1527] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_27_COL_11 — 显示驱动：cache 维护
源正文行 2442–2527；标题视频位置 02:54.0；异常时间戳 558487.665896（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2474 / 视频 02:58.0: <1>[558487.665861][ T1608] Unable to handle kernel paging request at virtual address ffffffc03337e000 K 7 4 7 K
CSV 2485 / 视频 02:59.5: <0>[558487.665896][ T1608] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 2488 / 视频 02:59.5: <4>[558487.666049][ T1608] CPU: 1 PID: 1608 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 2491 / 视频 03:00.0: <4>[558487.666064] [ T1608] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 2492 / 视频 02:59.5: <4>[558487.666069][ T1608] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_28_COL_11 — 显示驱动：cache 维护
源正文行 2530–2623；标题视频位置 03:02.0；异常时间戳 12192.314102（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2572 / 视频 03:05.0: <1>[12192.314054][ T1640] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 2583 / 视频 03:05.0: <0>[12192.314102] [ T1640] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 2586 / 视频 03:07.5: <4>[12192.314259] [ T1640] CPU: 1 PID: 1640 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1 |
CSV 2589 / 视频 03:07.5: <4>[12192.314282][ T1640] pe : __ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 2590 / 视频 03:07.5: <4>[12192.314286][ T1640] lr : Chip_Inv_Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_29_COL_11 — 显示驱动：cache 维护
源正文行 2626–2711；标题视频位置 03:09.5；异常时间戳 14071.841126（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2660 / 视频 03:13.0: <1>[(14071.841089] [ T1660] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 2671 / 视频 03:13.0: <O>[14071.841126][ T1660] Internal error: Oops: 96000147 [#1] PREEMPT SMP €
CSV 2674 / 视频 03:13.0: <4>[14071.841296] [ T1660] CPU: 1 PID: 1660 Comm: DMS SetMVOP_ Tas Tainted: P woo 5.4.302 #1
CSV 2677 / 视频 03:13.0: <4>[14071.841321] [ T1660] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 2678 / 视频 03:13.0: <4>[14071.841325][ T1660] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_30_COL_11 — 显示驱动：cache 维护
源正文行 2714–2799；标题视频位置 03:15.0；异常时间戳 322999.797079（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 2748 / 视频 03:16.5: 4<1>[322999.797048] [ T1636] Unable to handle’ kernel paging request at virtual address ffffffc03537e000
CSV 2759 / 视频 03:20.0: <0>[322999.797079] [ T1636] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 2762 / 视频 03:20.0: <4>[322999.797237][ T1636] CPU: 1 PID: 1636 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 2765 / 视频 03:20.0: <4>[322999.797255][ T1636] pc : __inval_dcache area+0x28/0x58 [mdrv_system]
CSV 2766 / 视频 03:19.5: <4>[322999.797259] [ T1636] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_31_COL_11 — 显示/GPU关联异常（需复核）
源正文行 2802–2885；标题视频位置 03:21.5；异常时间戳 203550.396632（fatal）。
PC/检测点（OCR）：`do_undefinstr+0x1c4/0x4f8`。

```text
CSV 2832 / 视频 03:25.0: <2>[203550.396621] [ CO] kernel BUG at /home/gfkfcmo/DockerWorkspace/M9618V_96EAMP_EUG/MT9618VD/device/mtk/9618/apollo/kernel/mixed/5.4 12/arch/armé4/kernel/traps.c:405 !
CSV 2833 / 视频 03:25.0: <0>[203550.396632) [ CO} Internal error: Oops - BUG: O [#1] PREEMPT SMP . r . . . . .
CSV 2836 / 视频 03:25.0: © <4>[203550.396834)] [ CO] CPU: 0 PID: 17837 Comm: kworker/0:2 Tainted: P woo $.4.302 #1
CSV 2840 / 视频 03:25.0: 6<4>[203550.396914] [ CO) pe : do_undefinstr+0x1c4/0x4f8
CSV 2841 / 视频 03:25.0: <4>[203550.396916] [ ¢O] lr : do_undefinstr+0x26c/0x4f8 : : i ‘ i % i 3
```

**判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**验证：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### ROW_32_COL_11 — 定时器/调度
源正文行 2888–2982；标题视频位置 03:26.0；异常时间戳 560904.863812（fatal）。
PC/检测点（OCR）：`hrtimer_try_to_cancel+0x28/0x1b8`。

```text
CSV 2938 / 视频 03:31.0: <1>[560904.863782] [T23920] Unable to handle kernel paging request at virtual address 003d0ea8d3533200
CSV 2948 / 视频 03:31.0: <0>[560904.863812] [T23920] Internal error: Oops: 96000004 [#1] PREEMPT SMP
CSV 2951 / 视频 03:30.0: <4>[S60904.863958] [T23920] CPU: 3 PID:. 23920 Comm: ARENDER_BI Tainted: P Ww. o $.4.302 #1 :
CSV 2954 / 视频 03:32.5: <4>[560904.863977] [T23920] pc : hrtimer_try_to_cancel+0x28/0x1b8
CSV 2955 / 视频 03:31.5: <4>[560904.863982] [T23920] lr: do_nanosleep+0xbc/0x178
```

**判断：** 优先排查定时器对象释放后使用、重复加入/删除、回调与销毁竞态，或其他模块提前破坏指针。`rb_erase` 属于通用树操作，只有调用栈经过 cfg80211 才归入Wi-Fi；定时器树应独立分析。

**验证：** 启用 DEBUG_OBJECTS_TIMERS、KASAN/SLUB调试，记录timer地址、回调、激活/取消/释放栈，核对取消与释放的同步。

### ROW_33_COL_11 — 显示驱动：Obj/PQ处理
源正文行 2985–3072；标题视频位置 03:32.5；异常时间戳 653933.414134（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 3026 / 视频 03:37.5: <1>[653933.414092][ T524] Unable to handle kernel paging request at virtual address ffffffc06029dcd4
CSV 3037 / 视频 03:37.5: <0>[653933.414134][ 1524] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 3040 / 视频 03:37.5: <4>[653933.414337][ 1T524] CPU: 0 PID: 524 Comm: irg/26-scaler Tainted: P w.o 5.4.302 #1 . . . . .
CSV 3043 / 视频 03:37.5: <4>[653933.414417][ 1524] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 3044 / 视频 03:37.5: <4>[653933.414455][ 1524) lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_34_COL_11 — 显示驱动：cache 维护
源正文行 3075–3158；标题视频位置 03:39.0；异常时间戳 105487.034734（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3105 / 视频 03:40.5: <1>[105487.034698] [ T1618] Unable to handle kernel paging request ‘at virtual address ffffffc038b7e000 i . . . ‘
CSV 3116 / 视频 03:40.5: <0>[105487.034734][ T1618] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 3119 / 视频 03:40.5: <4>[105487.034906] [ T1618] CPU: 1 PID: 1618 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 3122 / 视频 03:40.5: <4>[(105487.034932][ T1618] pc: inval dcache areat+0x28/0x58 [mdrv system]
CSV 3123 / 视频 03:42.0: <4>[(105487.034936] [ T1618] lr: Chip_Inv_Cache_Range+0x94/0x138 (mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_35_COL_11 — 显示驱动：cache 维护
源正文行 3161–3244；标题视频位置 03:43.5；异常时间戳 295610.769221（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3193 / 视频 03:43.5: <1>[295610.769189] [ T1373] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 3204 / 视频 03:46.5: <0>[295610.769221][ T1373] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 3207 / 视频 03:46.5: <4>[295610.769375] [ T1373] CPU: 1 PID: 1373 Comm: DMS_SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 3210 / 视频 03:46.5: <4>[295610.769400][ T1373] pe : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 3211 / 视频 03:46.5: <4>[295610.769404] [ T1373] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_36_COL_11 — 显示驱动：cache 维护
源正文行 3247–3340；标题视频位置 03:48.0；异常时间戳 6609.132280（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3288 / 视频 03:49.0: <1>[ 6609.132244][ T1613) Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 3299 / 视频 03:51.0: <O>[ 6609.132280][ T1613) Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 3302 / 视频 03:51.0: <4>[ 6609.132448][ T1613] CPU: 1 PID: 1613 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 3305 / 视频 03:51.0: <4>[ 6609.132469][ T1613} pc : __inval_dcache_area+0x28/0x58 [mdrv_system] .
CSV 3306 / 视频 03:51.0: <4>[ 6609.132473][ T1613] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_37_COL_11 — 显示驱动：cache 维护
源正文行 3343–3432；标题视频位置 03:53.0；异常时间戳 50550.374329（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3381 / 视频 03:54.5: <1>[50550.374293][ T1623] Unable to handle kernel paging request at virtual address fffFfffc03537e000
CSV 3392 / 视频 03:54.5: <0>[50550.374329][ T1623] Internal error: Oops: 96000147 [#1] PREEMPT SMP 7 . i . . 7 es
CSV 3395 / 视频 03:56.0: <4>[50550.374491] [ T1623] CPU: 1 PID: 1623 Comm: DMS SetMVOP_Tas Tainted: P woo §.4.302 #1
CSV 3398 / 视频 03:56.0: <4>[50550.374518][ T1623] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 3399 / 视频 03:56.0: <4>[(50550.374523][ T1623] lr : Chip_Inv_Cache Range+0x94/0x138 [mdrv_system] : : : :
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_38_COL_11 — 显示驱动：cache 维护
源正文行 3435–3521；标题视频位置 03:58.5；异常时间戳 22351.539694（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3464 / 视频 04:00.0: <1>[(22351.539646] [ T1642] Unable to handle kernel paging request at virtual address ffffffc03437e000
CSV 3475 / 视频 04:01.0: <0>[(22351.539694][ T1642) Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 3480 / 视频 04:01.0: <4>[22351.539865] [ T1642] CPU: 1 PID: 1642 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1 me
CSV 3483 / 视频 04:01.0: <4>[22351.539884][ T1642] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 3484 / 视频 04:01.0: <4>[22351.539888][ T1642] lr : Chip _Inv_ Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_41_COL_11 — 显示驱动：cache 维护
源正文行 3713–3818；标题视频位置 04:15.5；异常时间戳 265286.020479（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 3767 / 视频 04:19.0: <1>[(265286.020442][ T1263] Unable to handle kernel paging request at virtual address ffffffc03337e000
CSV 3778 / 视频 04:20.0: <0>[265286.020479][ T1263] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 3781 / 视频 04:21.0: <4>[265286.020627] [ T1263] CPU: 1 PID: 1263 Comm: DMS_SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 3784 / 视频 04:21.0: <4>[265286.020647] [ T1263] pc : __inval_dceache_area+0x28/0x58 [mdrv_system] . ; : ; :
CSV 3785 / 视频 04:21.0: <4>[(265286.020651] [ T1263] lr : Chip_Inv_Cache_Ranget+0x94/0x138 [mdrv_system] : : : ‘ ° Y :
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_43_COL_11 — 显示驱动：Obj/PQ处理
源正文行 3906–3986；标题视频位置 04:28.5；异常时间戳 611712.385477（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 3940 / 视频 04:31.5: <1>[611712.385443][ 1524] Unable to handle kernel paging request at virtual address ffffffc054c65559
CSV 3951 / 视频 04:31.5: <O0>[611712.385477][ 1T524] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 3954 / 视频 04:34.0: <4>[611712.385607][ 1524] CPU: 0 PID: 524 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1 : ‘ : : :
CSV 3957 / 视频 04:34.0: <4>[611712.385661][ 1524] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 3958 / 视频 04:34.0: <4>[611712.385693][ 1524] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_44_COL_11 — 显示驱动：cache 维护
源正文行 3989–4076；标题视频位置 04:37.0；异常时间戳 65835.912397（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 4022 / 视频 04:38.5: <1>[(65835.912360][ T1615] Unable to handle kernel paging request at virtual address ffffffc03437e000
CSV 4033 / 视频 04:38.5: <0>[65835.912397][ T1615] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 4038 / 视频 04:40.0: <4>[65835.912558][ T1615] CPU: 1 PID: 1615 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 4041 / 视频 04:40.0: <4>[(65835.912577] [ T1615] pc : __ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 4042 / 视频 04:38.5: <4>[65835.912581][ T1615] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] 7 . 7 . i be
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_45_COL_11 — 定时器/调度
源正文行 4079–4180；标题视频位置 04:40.5；异常时间戳 16598.429979（fatal）。
PC/检测点（OCR）：`Oxff£f££377b1019384b`。

```text
CSV 4141 / 视频 04:45.0: <0>[16598.429979] [ C3] Internal error: SP/PC alignment exception: 8a000000 [#1] PREEMPT SMP
CSV 4144 / 视频 04:45.0: <4>[16598.430131] [ C3) CPU: 3 PID: 0 Comm: swapper/3 Tainted: P wo 5.4.302 #1
CSV 4147 / 视频 04:45.0: <4>[16598.430146] [ C3] pe : Oxff£f££377b1019384b
CSV 4148 / 视频 04:45.0: <4>[16598.430157] [ C3] lr : _hrtimer_run_queves+0x118/0x380
```

**判断：** 优先排查定时器对象释放后使用、重复加入/删除、回调与销毁竞态，或其他模块提前破坏指针。`rb_erase` 属于通用树操作，只有调用栈经过 cfg80211 才归入Wi-Fi；定时器树应独立分析。

**验证：** 启用 DEBUG_OBJECTS_TIMERS、KASAN/SLUB调试，记录timer地址、回调、激活/取消/释放栈，核对取消与释放的同步。

### ROW_46_COL_11 — 显示驱动：cache 维护
源正文行 4183–4274；标题视频位置 04:47.0；异常时间戳 150129.479423（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 4220 / 视频 04:49.0: <1>[150129.479383][ T1285] Unable to handle’kernel paging requeSt at virtual address ffffffc03437e000
CSV 4231 / 视频 04:51.5: <0>[150129.479423][ T1285] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 4234 / 视频 04:51.5: <4>[150129.479564][ T1285] CPU: 1 PID: 1285 Comm: DMS_SetMVOP_Tas Tainted: P wo 5.4.302 #1
CSV 4237 / 视频 04:51.5: <4>[150129.479583][ T1285] pe : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 4239 / 视频 04:51.5: <4>[150129.479592][ T1285] lr: Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_47_COL_11 — 显示驱动：cache 维护
源正文行 4277–4360；标题视频位置 04:55.0；异常时间戳 172450.257043（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 4309 / 视频 04:58.0: <1>[172450.257006][ T1276] Unable to handle kernel paging request at virtual address ffffffc033b7e000
CSV 4320 / 视频 05:00.0: <0>(172450.257043][ T1276] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 4323 / 视频 04:58.0: <4>[172450.257280][ T1276] CPU: 1 PID: 1276 Comm: DMS SetMVOP Tas Tainted: P wo 5.4.302 #1
CSV 4326 / 视频 05:00.5: <4>[(172450.257305] [ T1276] pc : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 4327 / 视频 05:00.5: <4>[172450.257309][ T1276] lr : Chip_Inv_Cache_Range+0x94/0x138» [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_48_COL_11 — 蓝牙驱动：BRK/收发
源正文行 4363–4457；标题视频位置 05:02.5；异常时间戳 406035.153420（fatal）。
PC/检测点（OCR）：`OxfffFFfcOOcSabO0be`。

```text
CSV 4422 / 视频 05:05.5: <0>[406035.153420] [T10102] Internal error: ptrace BRK handler: £20003e8 [#1] PREEMPT SMP
CSV 4425 / 视频 05:05.5: <4>[406035.153552] [T10102] CPU: 2 PID: 10102 Comm: h¢i_sock_reader Tainted: P Z woo S.4.302 #1 f ac . i
CSV 4428 / 视频 05:08.5: <4>[406035.153573] [T10102] pe : OxfffFFfcOOcSabO0be
CSV 4429 / 视频 05:08.5: <4>[406035.153578] [T10102] lr : btmtk_usb_meta_send_data+0Oxdcc/0x13c0 [btmtk_usb_7663]
```

**判断：** 确认软件BRK异常，蓝牙USB收发路径关联明确；具体是厂商断言、异常跳转还是代码损坏，需要精确PC和模块反汇编，不能等同Wi-Fi BSS问题。

**验证：** 将异常PC与btmtk模块加载基址对齐，定位BRK指令及前置条件，保留HCI/USB状态、URB完成和断连、重置事件。

### ROW_49_COL_11 — 显示驱动：cache 维护
源正文行 4460–4562；标题视频位置 05:08.5；异常时间戳 215945.860793（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 4508 / 视频 05:14.0: <1>[215945.860753][ T1646] Unable to handle kernel paging request at virtual address ffffffb£ff537e000
CSV 4519 / 视频 05:14.0: <0>[215945.860793] [ T1646] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 4522 / 视频 05:14.0: <4>[215945.860967] [ T1646] CPU: 1 PID: 1646 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 4525 / 视频 05:14.0: <4>[215945.860989][ T1646] pc: __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 4527 / 视频 05:14.0: <4>[215945.860997][ T1646] lr : Chip .Inv_Cache_Ranget0x94/0x138 [mdrv_system] .
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_50_COL_11 — 显示驱动：Obj/PQ处理
源正文行 4565–4644；标题视频位置 05:17.0；异常时间戳 517398.223787（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 4598 / 视频 05:20.0: <1>[517398.223753][ 1T522] Unable to handle kernel paging request at virtual address ffffffc023d753900
CSV 4609 / 视频 05:22.0: <0>[517398.223787][ 1522] Internal error: Oops: 96000007 [#1] PREEMPT SMP
CSV 4612 / 视频 05:22.0: <4>[(517398.223946][ T522] CPU: 0 PID: 522 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1
CSV 4615 / 视频 05:22.0: <4>(517398.224011)[ T7522] pc: MDrv_Obj_OP_Process+0x1b8/0x1120 (kdrv_alg] ; ss
CSV 4616 / 视频 05:22.0: <4>[S17398.224044][ T522] lr : MDrv_Obj_OP..Process+0x154/0x1120 [kdrv_alg] . . . . . . 7
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_56_COL_11 — 内核栈破坏
源正文行 5157–5280；标题视频位置 06:09.5；异常时间戳 252943.300710（fatal）。
PC/检测点（OCR）：`z_erofs_vle_normalaccess_readpages+0x28c/0x28c`。

```text
CSV 5251 / 视频 06:19.5: <0>[252943.300710] [T25951] Kernel panic - not syncing: stack-protector: Kernel stack is ae in: z_erofs_vle_normalaccess_readpages+0x28c/0x28c
CSV 5252 / 视频 06:20.0: <2>[252943.300728] [T25951] [STBCIPC_panic]Caution!! This Only For Kernel Panic
CSV 5254 / 视频 06:19.5: <4>[252943.302751] [T25951] CPU: 1 PID: 25951 Comm: copy netflix_cr Tainted: P ° 5.4.283 #1
```

**判断：** 可以确认栈保护检查失败。报告的函数是检测点，写坏栈的代码可能更早，不能仅凭函数名断定其自身越界。

**验证：** 保留精确版本 vmlinux/.config；使用KASAN、栈保护和编译器边界检查定位最早的写入，检查局部数组、拷贝长度、跨线程/IRQ对栈内存的引用。

### ROW_57_COL_11 — 定时器/调度
源正文行 5283–5384；标题视频位置 06:22.5；异常时间戳 3009.988144（fatal）。
PC/检测点（OCR）：`tick_nohz_next_event+0xb0/0x164`。

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

### ROW_61_COL_11 — 网络：netlink队列
源正文行 5685–5751；标题视频位置 06:57.0；异常时间戳 322570.873536（PC timestamp; fatal header truncated）。
PC/检测点（OCR）：`skb_queue_tail+0x38/0x54`。

```text
CSV 5689 / 视频 06:57.0: <4>[322570.873506][ T2586] CPU: 3 PID: 2586 Comm: UINetworkModule Tainted: P 2 Oo 5.15.189-gbbeaa68710ec-dirty #1 i , 5
CSV 5693 / 视频 07:00.0: <4>[322570.873536][ T2586] pc : [<a0d5a58c>] lr : [<a0d5a578>] psr: 600b0093
```

**判断：** 队列链表尾指针/对象失效。寄存器r0为0，故障指令为str r4,[r0]的方向可与源画面核对。并行Mali日志不足以证明GPU就是写坏队列的来源。

**验证：** 检查sk_buff_head初始化、入队/出队和销毁锁，记录队列prev/next及生命周期；将并发CPU日志按PID与时间排序。

### ROW_62_COL_11 — 显示驱动：cache 维护
源正文行 5754–5842；标题视频位置 07:03.5；异常时间戳 146335.980003（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 5788 / 视频 07:07.0: <1>[146335.979956][ T1266] Unable to handle kernel paging request at virtual address ffffffc033b7e000
CSV 5799 / 视频 07:09.0: <0>[146335.980003][ T1266] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 5802 / 视频 07:09.0: <4>[146335.980193][ T1266] CPU: 1 PID: 1266 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 5805 / 视频 07:09.0: <4>[146335.980216] [ T1266] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system] : 2 i F 3 : |
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_64_COL_11 — 显示驱动：cache 维护
源正文行 5935–6039；标题视频位置 07:25.5；异常时间戳 114092.495797（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 5974 / 视频 07:27.0: <1>[114092.495765][ T1598] Unable to handle kernel paging request at virtual address ffffffbff2b7e000 s . . :
CSV 5985 / 视频 07:27.0: <O>[114092.495797][ T1598] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 5998 / 视频 07:27.0: §=6<4>[114092.495932][ T1598] CPU: 1 PID: 1598 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 6001 / 视频 07:28.5: <4>[114092.495949] [ T1598] pc : _ inval dcache_areat+0x28/0x58 [mdrv_system]
CSV 6003 / 视频 07:28.5: <4>[114092.495956] [ T1598] lr : Chip _Inv_Cache_Ranget+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_65_COL_11 — 显示驱动：Obj/PQ处理
源正文行 6042–6123；标题视频位置 07:30.0；异常时间戳 259312.503691（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 6077 / 视频 07:31.5: <1>[259312.503655][ T526] Unable to handle»kernel paging request at virtual address ffffffc06ce96a99
CSV 6088 / 视频 07:31.5: =<0>[(259312.503691])[ T526]) Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 6091 / 视频 07:31.5: <4>[259312.503841][ 1526) CPU: 0 PID: 526 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1
CSV 6094 / 视频 07:31.5: §=<4>[259312.503902][ T526] pce : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg] e F : x ‘ , i
CSV 6095 / 视频 07:31.5: <4>[259312.503936][ 1526] lr : MDrv_Obj_OP Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_66_COL_11 — 显示驱动：cache 维护
源正文行 6126–6210；标题视频位置 07:34.0；异常时间戳 6422.366047（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 6159 / 视频 07:35.5: <1>[ 6422.366009] [ T1645] Unable to handle kernel paging request at. virtual address ffffffc032b7e000
CSV 6170 / 视频 07:35.5: «<O>[ 6422.366047][ T1645] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 6173 / 视频 07:38.5: <4>[) 6422.366199] [ T1645] CPU: 1 PID: 1645 Comm: DMS _SetMVOP_Tas Tainted: P wo 5.4.302 #1
CSV 6176 / 视频 07:38.5: «© <4>[ 6422.366219][ T1645] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 6177 / 视频 07:38.5: <4>[ 6422.366223][ T1645] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_67_COL_11 — 显示驱动：Obj/PQ处理
源正文行 6213–6295；标题视频位置 07:39.5；异常时间戳 681079.612867（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 6249 / 视频 07:40.5: <1>[681079.612836][ 1524] Unable to handle kernel paging request at virtual address fff£fffcO7£F595c£0
CSV 6260 / 视频 07:40.5: <O0>[(681079.612867][ 1T524] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 6263 / 视频 07:40.5: <4>[681079.613006][ T524] CPU: O PID:, 524 Comm: irg/26-scaler Tainted: P wo 5.4.302 #1 e a . P 2
CSV 6266 / 视频 07:40.5: <4>[681079.613055][ 1T524] pce: MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 6267 / 视频 07:40.5: <4>[681079.613089][ 1524] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_70_COL_11 — 显示驱动：cache 维护
源正文行 6471–6553；标题视频位置 07:49.5；异常时间戳 75668.687696（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 6495 / 视频 07:52.0: <1>[75668.687666] [ T1625] Unable to handle kernel paging request at virtual address ffffffbff2b7e000
CSV 6506 / 视频 07:52.0: <O>[75668.687696] [ T1625] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 6509 / 视频 07:54.0: © <4>[75668.687859] [ T1625] CPU: 1 PID: 1625 Comm: DMS _SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 6512 / 视频 07:54.0: <4>[75668.687882] [ T1625] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 6513 / 视频 07:54.0: <4>[75668.687886] [ T1625] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] 3 ; : : . 4 E
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_71_COL_11 — 显示驱动：Obj/PQ处理
源正文行 6556–6636；标题视频位置 07:55.5；异常时间戳 228429.460298（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 6590 / 视频 07:57.5: <1>[228429.460263][ T523] Unable to handle kernel paging request at virtual address ffffffFc064b77493
CSV 6601 / 视频 07:57.5: <0>[228429.460298][ 1523] Internal error: Oops: 96000005 [#1] PREEMPT SMP z $ ‘ J : i ‘
CSV 6604 / 视频 07:57.5: <4>[228429.460425][ 1523] CPU: O PID: 523 Comm: irg/26-scaler Tainted: P wo 5.4.302 #1
CSV 6607 / 视频 07:57.5: <4>[228429.460480)[ 1523] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 6608 / 视频 07:57.5: <4>[228429.460512][ T1523] lr : MDrv_.Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_72_COL_11 — 显示驱动：cache 维护
源正文行 6639–6722；标题视频位置 07:59.0；异常时间戳 75215.927193（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 6671 / 视频 08:00.5: <1>[75215.927161] [ T1656] Unable to handle kernel paging request at virtual address ffffffc032b7e000
CSV 6682 / 视频 08:00.5: <0>[75215.927193][ T1656] Internal error: Oops: 96000147 [#1] PREEMPT SMP : : ; e ike
CSV 6685 / 视频 08:02.5: <4>[75215.927356] [ T1656] CPU: 1 PID: 1656 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 6688 / 视频 08:02.5: §=9<4>[75215.927377] [ T1656] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 6689 / 视频 08:02.5: <4>[75215.927382] [ T1656) lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] A Z 5 3 z z E
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_73_COL_11 — 显示驱动：Obj/PQ处理
源正文行 6725–6806；标题视频位置 08:05.0；异常时间戳 178917.856872（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 6760 / 视频 08:07.5: <1>[178917.856832][ T524] Unable to handle kernel paging request at virtual address ffffffc027b452ca
CSV 6771 / 视频 08:07.5: <0>[178917.856872][ 1524] Internal error: Oops: 96000007 [#1] PREEMPT SMP
CSV 6774 / 视频 08:07.5: «-<4>[178917.857013][ 1524] CPU: 0 PID: 524 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1
CSV 6777 / 视频 08:07.5: <4>[178917.857068][ 1524] pce : MDrv_.0bj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 6778 / 视频 08:07.5: <4>[178917.857101][ 1524] lr : MDrv_Obj_OP_ Hrocess+0x154/0x1120 [kdrv_alg] : : : % ie: ; i
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_74_COL_11 — 显示驱动：cache 维护
源正文行 6809–6900；标题视频位置 08:09.0；异常时间戳 261197.957102（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 6849 / 视频 08:10.5: <1>[(261197.957059][ T1653] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 6860 / 视频 08:13.0: <0>(261197.957102][ T1653] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 6863 / 视频 08:13.0: <4>[261197.957260] [ T1653] CPU: 1 PID: 1653 Comm: DMS _SetMVOP_Tas Tainted: P woo $.4.302 #1
CSV 6866 / 视频 08:13.0: «6<4>[261197.957282] [ T1653] pec : __inval_dcache_area+0x28/0x58 [mdrv_system] e. ‘ ar .. ° 7
CSV 6867 / 视频 08:13.0: <4>[261197.957286] [ T1653] lr : Chip Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_77_COL_11 — 显示驱动：cache 维护
源正文行 7083–7177；标题视频位置 08:23.0；异常时间戳 97966.765765（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 7121 / 视频 08:25.0: <1>(97966.765726] [ T1622] Unable to handle kernel paging request at virtual address ffffffc032b7e000
CSV 7132 / 视频 08:27.0: <0>[(97966.765765] [ T1622] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 7136 / 视频 08:27.0: <4>[(97966.765919] [ T1622] CPU: 1 PID: 1622 Comm: DMS SetMVOP_Tas Tainted: P woo §.4.302 #1
CSV 7139 / 视频 08:27.0: <4>[97966.765939][ T1622] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 7140 / 视频 08:27.0: <4>[97966.765943][ T1622] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_79_COL_11 — 内核栈破坏
源正文行 7286–7407；标题视频位置 08:34.5；异常时间戳 28959.987779（fatal）。
PC/检测点（OCR）：`scm_recv+0x158/0x158`。

```text
CSV 7382 / 视频 08:41.5: <0>[28959.987779][ 1485] Kernel panic - not syncing: stack-protector: Kernel stack is corrupted in: scm_recv+0x158/0x158
CSV 7383 / 视频 08:41.5: <2>[28959.987797][ 1485] [STBCIPC_panic]Caution!! This Only For Kernel Panic A F x Z P é
CSV 7385 / 视频 08:41.5: <4>[(28959.989818][ 1T485] CPU: 3 PID: 485 Comm: busybox Tainted: P ° 5.4.283 #1
```

**判断：** 可以确认栈保护检查失败。报告的函数是检测点，写坏栈的代码可能更早，不能仅凭函数名断定其自身越界。

**验证：** 保留精确版本 vmlinux/.config；使用KASAN、栈保护和编译器边界检查定位最早的写入，检查局部数组、拷贝长度、跨线程/IRQ对栈内存的引用。

### ROW_80_COL_11 — 显示/GPU关联异常（需复核）
源正文行 7410–7515；标题视频位置 08:43.0；异常时间戳 16606.318313（fatal）。
PC/检测点（OCR）：`do_undefinstr+0x230/0x234`。

```text
CSV 7473 / 视频 08:45.0: <2>[16606.318306] [T12039] kernel BUG at arch/arm6é4/kernel/traps.c:405!
CSV 7474 / 视频 08:45.0: <O0>[(16606.318313] [T12039] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 7480 / 视频 08:47.5: <4>[16606.318550] [T12039] pc : do_undefinstr+0x230/0x234
CSV 7481 / 视频 08:47.5: <4>[16606.318558] [T12039] lr : do_undefinstrt+0xb8/0x234
```

**判断：** 需区分原始异常指令与 do_undefinstr 的二次BUG。IRQ之下的其他栈帧可能只是被中断现场；不将模块出现在列表里当作根因。

**验证：** 用精确模块/vmlinux定位 el1_undef 原始ELR、指令和回调；检查代码/函数指针、IRQ状态、模块和设备关闭时序。

### ROW_82_COL_11 — 显示驱动：cache 维护
源正文行 7614–7697；标题视频位置 08:53.5；异常时间戳 261559.703227（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 7646 / 视频 08:55.5: <1>[261559.703180][ T1626] Unable to handle kernel paging request at virtual address ffffffc032b7e000
CSV 7657 / 视频 08:58.5: <O>[261559.703227][ T1626] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 7660 / 视频 08:58.5: <4>[261559.703396] [ T1626] CPU: 1 PID: 1626 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 7663 / 视频 08:58.5: <4>[261559.703418] [ T1626] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system] 2 fe ‘ r . z E
CSV 7664 / 视频 08:58.5: <4>[261559.703422][ T1626] lr: Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_83_COL_11 — 显示驱动：Obj/PQ处理
源正文行 7700–7781；标题视频位置 08:59.0；异常时间戳 93364.697244（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 7734 / 视频 09:00.5: <1>[(93364.697208][ 1549] Unable to handle kernel paging request at virtual address ffffffc0b293dcd4
CSV 7745 / 视频 09:00.5: <0>[93364.697244][ 1549] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 7748 / 视频 09:00.5: <4>[93364.697413][ 1549] CPU: 0 PID: 549 Comm: irg/26-scaler Tainted: P woo $.4.302 #1
CSV 7751 / 视频 09:00.5: <4>[93364.697481][ 1549] pc : MDrv_Obj_OP_ Process+0x1b8/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_84_COL_11 — 显示驱动：cache 维护
源正文行 7784–7877；标题视频位置 09:01.5；异常时间戳 123157.113585（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 7823 / 视频 09:03.5: <1>[(123157.113548][ T1625] Unable to handle kernel paging request at virtual address ffffffc03ch7e000
CSV 7834 / 视频 09:05.0: <0>(123157.113585][ T1625) Internal error: Oops: 96000146 [#1] PREEMPT SMP
CSV 7837 / 视频 09:05.0: <4>[123157.113785] [ T1625] CPU: 1 PID: 1625 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 7840 / 视频 09:05.0: <4>[123157.113821][ T1625] pc : __inval_dcache_area+0x28/0x58 [mdrv_system] i : rh * ° 7
CSV 7841 / 视频 09:05.0: <4>[123157.113825][ T1625] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_85_COL_11 — 显示驱动：UCD断言
源正文行 7880–7959；标题视频位置 09:07.0；异常时间戳 26744.745629（fatal）。
PC/检测点（OCR）：`MApi_UCD_OP_Process+0x28e4/0x2c60 [kdrv_alg]`。

```text
CSV 7922 / 视频 09:08.5: <2>[26744.745624] [ T1712] kernel BUG at /worktmp/build/4407/RGS631299 hisense_linux_REL_Disty_P1.75.0/code/misdk/mixed/xc_86/kdrv_alg/drv/.//mdrv_ucd.c:1157!
CSV 7923 / 视频 09:08.5: <0>[26744.745629][ T1712] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 7926 / 视频 09:10.0: <4>[26744.745738] [ T1712] CPU: 0 PID: 1712 Comm: kworker/0:1 Tainted: P wo 5.4.302 #1
CSV 7930 / 视频 09:10.0: <4>[26744.745826] [ T1712] pc : MApi_UCD_OP_Process+0x28e4/0x2c60 [kdrv_alg]
CSV 7931 / 视频 09:10.0: <4>[26744.745858][ T1712] lr : g_stCFDInfot+0x6850/0xfffffffffff67850 [kdrv_alg]
```

**判断：** 确认厂商UCD实现主动触发BUG，具体失败条件只能查同一构建mdrv_ucd.c第1157行。优先核对resume时UCD状态、参数/表尺寸、资源初始化和版本兼容。

**验证：** 取得对应厂商源码和kdrv_alg模块，读取实际BUG条件及其输入；记录resume前后对象、配置、初始化和释放顺序，避免仅删除BUG掩盖状态异常。

### ROW_86_COL_11 — 显示驱动：cache 维护
源正文行 7962–8047；标题视频位置 09:11.5；异常时间戳 157627.058144（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 7995 / 视频 09:11.5: <1>[157627.058103] [ T1627] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 8006 / 视频 09:13.0: «0 <0>[157627.058144] [ T1627] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 8009 / 视频 09:13.0: <4>[157627.058295] [ T1627] CPU: 1 PID: 1627.Comm: DMS SetMVOP_Tas Tainted: P °.W O 5.4.302 #1 eo 2 : a *
CSV 8012 / 视频 09:13.0: <4>[157627.058312][ T1627] pe : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 8013 / 视频 09:13.0: <4>[157627.058316] [ T1627] lr : Chip _Inv_Cache_Ranget+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_87_COL_11 — 显示驱动：cache 维护
源正文行 8050–8130；标题视频位置 09:14.0；异常时间戳 389523.277019（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8078 / 视频 09:15.5: «=9<<1>[389523.276965][ T1608] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 8090 / 视频 09:15.5: « <O>[389523.277019][ T1608] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 8093 / 视频 09:17.0: <4>[389523.277214] [ T1608] CPU: 1 PID: 1608 Comm: DMS_SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 8096 / 视频 09:17.0: - <4>[389523.277244][ T1608] pc: __inval_dceache_area+0x28/0x58 [mdrv_system] ; ;
CSV 8097 / 视频 09:17.0: <4>[389523.277248] [ T1608] lr : Chip_Inv_Cache_ Ranget0x94/0x138 [mdrv_system] . z : . :
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_88_COL_11 — 显示驱动：cache 维护
源正文行 8133–8221；标题视频位置 09:19.0；异常时间戳 6558.623799（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8168 / 视频 09:20.0: <1>[ 6558.623749] [ T1606] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 8179 / 视频 09:20.0: <O>[ 6558.623799][ T1606] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 8182 / 视频 09:20.0: <4>[ 6558.623949][ T1606] CPU: 1 PID: 1606 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 8185 / 视频 09:20.0: <4>[ 6558.623970][ T1606] pc : __inval_dcache_area+0x28/0x58 [mdrv_system] : 2 2 x ‘ sf
CSV 8186 / 视频 09:20.0: 6<4>[ 6558.623974][ T1606] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_90_COL_11 — 显示驱动：cache 维护
源正文行 8316–8403；标题视频位置 09:27.0；异常时间戳 9575.204629（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8350 / 视频 09:28.0: <1>[ 9575.204578][ T1602] Unable to handle kernel paging request at virtual address ffffffc03337e000
CSV 8361 / 视频 09:28.0: <O>[ 9575.204629][ T1602] Internal error: Oops: 96000147 [#1] PREEMPT SMP : 3 E E ‘
CSV 8364 / 视频 09:30.5: <4>[ 9575.204833] [ T1602] CPU: 1 PID: 1602 Comm: DMS SetMVOP_Tas Tainted: P woo §.4.302 #1
CSV 8367 / 视频 09:30.5: <4>[) 9575.204857] [ T1602] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 8368 / 视频 09:30.5: <4>[ 9575.204861][ T1602] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] ;
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_91_COL_11 — 显示驱动：cache 维护
源正文行 8406–8489；标题视频位置 09:32.0；异常时间戳 4002.635864（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8433 / 视频 09:32.0: <1>[ 4002.635828][ T1645] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 8444 / 视频 09:33.5: <O>[ 4002.635864][ T1645] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 8447 / 视频 09:33.5: <4>[ 4002.636012][ T1645] CPU: 1 PID: 1645 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 8450 / 视频 09:33.5: <4>[, 4002.636037] [ T1645] pe : __inval_dcache_area+0x28/0x58 [mdrv_‘system] ¥ 2 : . .%
CSV 8451 / 视频 09:33.5: <4>[ 4002.636041][ T1645] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_92_COL_11 — 显示驱动：cache 维护
源正文行 8492–8581；标题视频位置 09:35.0；异常时间戳 145104.247303（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8527 / 视频 09:36.5: <1>[145104.247262] [ T1588] Unable to handle kernel paging request at virtual address ffffffbff4b7e000
CSV 8538 / 视频 09:36.5: | <O>[145104.247303][ T1588] Internal error: Odps : 96000145 [#1] PREEMPT SMP ; ‘ : é il r 5
CSV 8541 / 视频 09:38.5: <4>[145104.247515] [ T1588] CPU: 1 PID: 1588 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 8544 / 视频 09:38.5: <4>[145104.247536][ T1588] pe : _ inval_dcache area+0x28/0x58 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_95_COL_11 — 显示驱动：Obj/PQ处理
源正文行 8778–8858；标题视频位置 09:51.0；异常时间戳 36070.897164（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 8812 / 视频 09:52.5:  <1>[(36070.897133][ 1564] Unable to handle kernel paging request at virtual address ffffffc0af94e376
CSV 8823 / 视频 09:52.5: «<O0>[36070.897164][ 1T564] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 8826 / 视频 09:52.5: <4>[36070.897306][ 1564] CPU: 0 PID: 564 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 8829 / 视频 09:52.5: <4>[36070.897359][ 1564] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 8830 / 视频 09:52.5: 6<<4>[36070.897391][ T564] lr: MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_96_COL_11 — 显示驱动：cache 维护
源正文行 8861–8945；标题视频位置 09:54.0；异常时间戳 2558.464173（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8893 / 视频 09:55.5: <1>[ 2558.464139][ T1628] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 8904 / 视频 09:55.5: <O>[ 2558.464173] [ T1628] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 8907 / 视频 09:57.0: <4>[ 2558.464317][ T1628] CPU: 1 PID: 1628 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 8910 / 视频 09:57.0: <4>[ 2558.464342][ T1628] pc : __ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 8911 / 视频 09:57.0: <4>[ 2558.464346][ T1628] lr : Chip_Inv_Cache_Range+0x94/0x138 {mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_97_COL_11 — 显示驱动：cache 维护
源正文行 8948–9041；标题视频位置 10:00.0；异常时间戳 172821.519987（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 8989 / 视频 10:00.5: <1>[172821.519950][ T1613] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 9000 / 视频 10:00.5: 4<0>[(172821.519987] [ T1613] Internal error: Oops: 96000147 [#1] PREEMPT SMP . |
CSV 9003 / 视频 10:00.5: <4>[172821.520126] [[ T1613] CPU: 1 PID: 1613 Comm: DMS _SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 9006 / 视频 10:00.5: §=<4>[172821.520147] [ T1613] pce: __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 9007 / 视频 10:00.5: <4>[172821.520151] [ T1613] lr : Chip _Inv_Cache_Ranget+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_99_COL_11 — 显示驱动：cache 维护
源正文行 9126–9219；标题视频位置 10:07.5；异常时间戳 374166.702238（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 9164 / 视频 10:09.0: <1>(374166.702202) [ T1632) Unable to handle kernel paging request at virtual address ffffffbff537e000
CSV 9175 / 视频 10:09.0: <0>[374166.702238][ T1632] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 9179 / 视频 10:10.5: <4>[374166.702410] [ T1632] CPU: 1 PID: 1632 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 9182 / 视频 10:10.5: <4>[374166.702432][ T1632] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 9183 / 视频 10:10.5: <4>[374166.702436] [ T1632] lr : Chip _Inv_Cache_Range+0x94/0x138. [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_101_COL_11 — 显示驱动：cache 维护
源正文行 9308–9393；标题视频位置 10:17.0；异常时间戳 164060.229931（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 9342 / 视频 10:17.5: <1>[164060.229888][ T1628] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 9353 / 视频 10:17.5: <0>[164060.229931][ T1628] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 9356 / 视频 10:20.0: <4>[164060.230105] [ T1628] CPU: 1 PID: 1628 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 9359 / 视频 10:20.0: <4>[164060.230131][ T1628] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 9360 / 视频 10:20.0: <4>[164060.230135][ T1628] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_106_COL_11 — TCP/socket对象
源正文行 9787–9876；标题视频位置 10:40.0；异常时间戳 141757.549677（fatal）。
PC/检测点（OCR）：`__tep_close+0x14/0x440 ;`。

```text
CSV 9839 / 视频 10:43.0: <1>[(141757.549642][ T2451] Unable to handle kernel paging request at virtual address 000080ed804alela
CSV 9849 / 视频 10:43.0: <0>[141757.549677] [ T2451] Internal error: Oops: 96000044 [#1] PREEMPT SMP
CSV 9851 / 视频 10:44.0: <4>[141757.549763] [ T2451] CPU: 3 PID: 2451 Comm: HTTPREQUEST_MAN Tainted: P fe) 5.4.283 #1
CSV 9854 / 视频 10:44.0: <4>[141757.549796] [ T2451] pc : __tep_close+0x14/0x440 ;
CSV 9855 / 视频 10:44.0: <4>[141757.549804] [ T2451] lr : tcp_recvmsg+0x448/0x9e0 ; G . . . ; :
```

**判断：** socket/tcp对象指针或生命周期受损方向较强；发送/接收应用线程只是当时上下文，不能证明应用直接写坏内核。

**验证：** 定位故障指令基址寄存器，检查sk引用、close与recv竞争、析构后使用；用KASAN/SLUB记录对象分配与释放栈。

### ROW_107_COL_11 — 显示驱动：Obj/PQ处理
源正文行 9879–9961；标题视频位置 10:44.0；异常时间戳 376507.300581（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 9915 / 视频 10:45.5: <1>[376507.300546][ T523] Unable to handle kernel paging request at virtual address ffffffcOdf7cecfe
CSV 9926 / 视频 10:47.0: ©<0>[376507.300581][ T523] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 9929 / 视频 10:47.0: <4>[376507.300727][ 1T523] CPU: 0 PID: 523 Comm: irq/26-scaler Tainted: P wo 5.4.302 #1
CSV 9932 / 视频 10:47.0: <4>[376507.300779][ 1523] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 9933 / 视频 10:47.0: <4>[376507.300816][ 1523] lr : MDrv_Obj_OP_Processt+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_108_COL_11 — 调谐器驱动：恢复/ioctl
源正文行 9964–10057；标题视频位置 10:49.0；异常时间戳 82127.772860（fatal）。
PC/检测点（OCR）：`MI_DEV_TUNER_ResumeEarly+0xlebc/0x24a0 [mik]`。

```text
CSV 10012 / 视频 10:51.5: <1>[82127.772829] [ T2454] Unable to handle kernel access to user memory outside uaccess routines at virtual address 0000000000000004
CSV 10023 / 视频 10:51.5: <O0>[82127.772860][ T2454] Internal error: Oops: 96000045 [#1] PREEMPT SMP
CSV 10026 / 视频 10:52.5: .<4>[82127.772999] [ T2454] CPU: 2 PID: 2454 Comm: Binder_7 Tainted: P woo 5$.4.302 #1
CSV 10029 / 视频 10:52.5: <4>[82127.773062] [ T2454] pce : MI_DEV_TUNER_ResumeEarly+0xlebc/0x24a0 [mik]
```

**判断：** 低地址访问支持对象为空或失效后加字段偏移。可能涉及resume初始化次序、ioctl参数、并发关闭；Binder线程名不是根因证据。

**验证：** 反汇编mik.ko，确认基址、索引和写入指令；校验用户拷贝返回值、嵌套指针、compat结构尺寸，检查resume与close/release同步。

### ROW_109_COL_11 — 显示驱动：cache 维护
源正文行 10060–10145；标题视频位置 10:54.5；异常时间戳 157594.326702（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10091 / 视频 10:54.5: <1>[157594.326663][ T1603] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 10102 / 视频 10:56.0: <0>[157594.326702] [ T1603] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 10105 / 视频 10:56.0: <4>[157594.326852] [ T1663] CPU: 1 PID: 1603.Comm: DMS _SetMVOP_Tas Tainted: P ‘,W Oo 5.4.302 #1 : : a *
CSV 10108 / 视频 10:56.0: <4>[157594.326874][ T1603] pce : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 10109 / 视频 10:56.0: <4>[157594.326878] [ T1603] lr : Chip _Inv_Cache_Ranget+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_110_COL_11 — 显示驱动：cache 维护
源正文行 10148–10232；标题视频位置 10:58.0；异常时间戳 6575.525835（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10179 / 视频 11:00.0: <1>[ 6575.525797][ T1592] Unable to handle kernel paging request at virtual address ffffffc034b7e000 |
CSV 10190 / 视频 10:59.5: <O>[ 6575.525835][ T1592] Internal error: Oops: 96000146 [#1] PREEMPT SMP
CSV 10195 / 视频 11:01.5: <4>[ 6575.526014] [ T1592] CPU: 1 PID: 1592 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 10198 / 视频 11:01.5: «© <4>[ 6575.526044][ T1592] pc : _ inval_dcache_area+0x28/0xS8 [mdrv_system]
CSV 10199 / 视频 11:01.5: <4>[ 6575.526049][ T1592] lr: Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_111_COL_11 — 显示驱动：cache 维护
源正文行 10235–10319；标题视频位置 11:03.0；异常时间戳 666630.311437（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10266 / 视频 11:05.0: <1>[666630.311410] [ T1262) Unable to handle kernel paging request at virtual address ffffffc033b7e000 ” ‘ . . :
CSV 10277 / 视频 11:05.0: <0>[666630.311437] [ T1262] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 10282 / 视频 11:05.0: <4>[666630.311558] [ T1262] CPU: 1 PID: 1262 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1 :
CSV 10285 / 视频 11:05.0: <4>[666630.311572] [ T1262] pc : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 10286 / 视频 11:05.0: <4>[666630.311576] [ T1262] lr: Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_112_COL_11 — 内存保护单元报警
源正文行 10322–10416；标题视频位置 11:07.0；异常时间戳 45946.768779（fatal）。
PC/检测点（OCR）：`?`。

```text
CSV 10387 / 视频 11:10.0: <0>[45946.768779] [ CO] Kernel panic - not syncing: MIUP hit panic[CID:0x811A read address: 0xC0000000 - OxCO00001F block:24
CSV 10388 / 视频 11:10.0: 4<4>[45946.768786] [ CO] CPU: 0 PID: 1554 Comm: mi_daemon_sys.b Tainted: P Woo §.4.302 #1
```

**判断：** 内存保护硬件报告某个client访问受保护区间，并被驱动升级为panic。应检查client地址/权限/缓冲区配置；被中断的Binder调用不等于肇事DMA client。

**验证：** 用当前SoC的MIUP client表解码CID，记录保护block范围、读写类型、触发master和DMA/IOMMU映射；核对地址截断、边界、保护配置与释放顺序。

### ROW_114_COL_11 — 显示驱动：cache 维护
源正文行 10504–10596；标题视频位置 11:16.5；异常时间戳 90011.559616（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10545 / 视频 11:19.5: <1>[(90011.559575] [ T1644] Unable to handle kernel paging request at virtual address ffffffc03337e000 ig : . 7 :
CSV 10556 / 视频 11:19.5: <O0>[90011.559616][ T1644] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 10559 / 视频 11:19.5:  <4>[90011.559803] [ T1644] CPU: 1 PID: 1644 Comm: DMS SetMVOP Tas Tainted: P Ww oo 5.4.302 #1
CSV 10562 / 视频 11:19.5: <4>[90011.559837] [ T1644] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 10563 / 视频 11:19.5: <4>[90011.559841][ T1644] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_115_COL_11 — 显示驱动：cache 维护
源正文行 10599–10681；标题视频位置 11:21.5；异常时间戳 17045.518537（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10628 / 视频 11:23.0: 9 <1>[17045.518483] [ T1623] Unable to handle kernel paging request at virtual address ffffffc034b7e000 |
CSV 10639 / 视频 11:23.0: <0>[17045.518537] [ T1623] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 10642 / 视频 11:25.5: <4>[17045.518731] [ T1623] CPU: 1 PID: 1623 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 10645 / 视频 11:25.5: <4>[17045.518756] [ T1623] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 10646 / 视频 11:25.5: <4>[17045.518760] [ T1623] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_116_COL_11 — 显示驱动：Obj/PQ处理
源正文行 10684–10769；标题视频位置 11:27.5；异常时间戳 183963.356473（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 10723 / 视频 11:30.0: <1>(183963.356438][ 1T524] Unable to handle kernel paging request at virtual address ffffffc05df56459
CSV 10734 / 视频 11:30.0: <0>[183963.356473][ 1T524] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 10740 / 视频 11:31.5: <4>[183963.356676][ T524] pc: MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 10741 / 视频 11:31.5: <4>[183963.356712][ 1524] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_117_COL_11 — 显示驱动：cache 维护
源正文行 10772–10857；标题视频位置 11:31.5；异常时间戳 420872.599063（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10803 / 视频 11:33.0: <1>[420872.598881][ T1354] Unable to handle kernel paging request at virtual address ffffffc033b7e000
CSV 10814 / 视频 11:35.0: <0>[420872.599063] [ T1354] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 10817 / 视频 11:35.0: <4>[420872.599297] [ T1354] CPU: 1 PID:, 1354 Comm: DMS SetMVOP_Tas Tainted: P , Woo 5.4.302 #1 ‘ : ‘ ZA K
CSV 10820 / 视频 11:35.0: <4>[420872.599341][ T1354] pce : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 10821 / 视频 11:35.0: <4>[420872.599345][ T1354] lr : Chip _Inv_Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_119_COL_11 — 显示驱动：cache 维护
源正文行 10946–11031；标题视频位置 11:44.5；异常时间戳 1559.975117（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 10977 / 视频 11:44.5: «- <i>[ 1559.975079][ T1656] Unable to handle kernel paging request at virtual address fffFffc038b7e000
CSV 10988 / 视频 11:46.0: 4<O>[ 1559.975117][ T1656] Internal error: Oops: 96000146 [#1] PREEMPT SMP
CSV 10991 / 视频 11:46.0: <4>[ 1559.975251][ T1656] CPU: 1 PID: 1656 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 10994 / 视频 11:46.0: <4>[ 1559.975275][ T1656] pe : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 10995 / 视频 11:46.0: <4>[, 1559.975280] [ T1656] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] a ‘ rh * ° 7
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_120_COL_11 — 显示驱动：cache 维护
源正文行 11034–11116；标题视频位置 11:47.5；异常时间戳 149005.472687（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 11064 / 视频 11:49.0: <1>[149005.472641][ T1638] Unable to handle kernel paging request at virtual address ffffffc03237e000
CSV 11075 / 视频 11:49.0: <0>[149005.472687][ T1638] Internal Error: Oops: 96000147 [#1] PREEMPT SMP f F * F ‘ 7
CSV 11079 / 视频 11:50.5: <4>[149005.472890][ T1638] CPU: 1 PID: 1638 Comm: DMS_SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 11082 / 视频 11:50.5: <4>[149005.472916][ T1638] pc: __inval_dcache_area+0x28/0x58 [mdrv_system] ; ;
CSV 11083 / 视频 11:50.5: <4>[149005.472920] [ T1638] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] C a : . :
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_123_COL_11 — 显示驱动：Obj/PQ处理
源正文行 11292–11375；标题视频位置 12:00.5；异常时间戳 82873.694365（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 11329 / 视频 12:01.5: <1>[82873.694326][ 1544] Unable to handle kernel paging request at virtual address ffffffc04221f21le
CSV 11340 / 视频 12:01.5: <0>[82873.694365][ 1544] Internal error: Oops: 96000005 [#1] PREEMPT SMP : ‘ : : i ‘
CSV 11343 / 视频 12:01.5: <4>[82873.694513][ 1544] CPU: 0 PID: 544 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 11346 / 视频 12:01.5: <4>[82873.694580][ 1544] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 11347 / 视频 12:01.5: <4>[82873.694614][ 1544] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_124_COL_11 — 显示驱动：cache 维护
源正文行 11378–11467；标题视频位置 12:03.5；异常时间戳 46688.582509（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 11416 / 视频 12:05.5: <1>[46688.582471][ T1625] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 11427 / 视频 12:07.5: <0>[46688.582509][ T1625] Internal error: Oops: 96000147 [#1] PREEMPT SMP a ;
CSV 11430 / 视频 12:07.5: <4>[46688.582659][ T1625] CPU: 1 PID: 1625 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 11433 / 视频 12:07.5: <4>[46688.582676][ T1625] pc: __inval_dcache_area+0x28/0x58 (mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_130_COL_11 — 视频监控线程：对象失效
源正文行 11975–12053；标题视频位置 12:33.0；异常时间戳 819970.559722（fatal）。
PC/检测点（OCR）：`_MI_VMON_DecMonTask+0x1fc/0x6b8 [mik]`。

```text
CSV 12002 / 视频 12:35.0: <1>[(819970.559688] [T21760] Unable to handle kernel paging request at virtual address e8c6f098
CSV 12005 / 视频 12:35.0: <0>[819970.559722] [T21760] Internal error: Oops: 7 [#1] PREEMPT SMP ARM
CSV 12008 / 视频 12:35.0: =9<4>[819970.560007] [T21760] CPU: 3 PID: 21760 Comm: VMON_DEC_MON Tainted: P Woo 5.15.189-gbbeaa68710ec-dirty #1
CSV 12012 / 视频 12:35.0: <4>[819970.560953] [T21760] pe : [<del5a54c>] lr : [<a01a7048>] psr: 800b0013
```

**判断：** 监控线程读取无效对象/字段，解码器关闭与监控任务竞态值得优先排查，但前面的资源释放日志本身不是证明。

**验证：** 反汇编mik对应指令并核对r4与FAR，检查VDEC关闭、VMON停止/等待完成、上下文释放和信号量的顺序。

### ROW_131_COL_11 — 显示驱动：cache 维护
源正文行 12056–12143；标题视频位置 12:38.0；异常时间戳 140635.568324（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12092 / 视频 12:40.0: <1>[140635.568284][ T1628] Unable to handle kernel paging request at virtual address ffffffc032b7e000
CSV 12103 / 视频 12:40.0: <0>[140635.568324][ T1628] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 12106 / 视频 12:39.5: «=<4>[140635.568477] [ T1628] CPU: 1 PID: 1628 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 12109 / 视频 12:40.0: <4>[140635.568498] [ T1628] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 12110 / 视频 12:39.5: 6<4>[140635.568502][ T1628] lr : Chip Inv_Cache_Ranget0x94/0x138 [mdrv_system] 5 F 2 z ‘ !
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_132_COL_11 — 显示驱动：cache 维护
源正文行 12146–12229；标题视频位置 12:41.0；异常时间戳 13219.528053（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12178 / 视频 12:45.5: <1>[13219.528017][ T1627] Unable to handle kernel paging request at virtual address ffffffcO03cb7e000
CSV 12189 / 视频 12:45.5: <0>[(13219.528053][ T1627] Internal error: Oops: 96000146 [#1] PREEMPT SMP
CSV 12192 / 视频 12:46.5: <4>[13219.528188][ T1627] CPU: 1 PID: 1627 Comm: DMS SetMVOP_Tas Tainted: P Woo 5.4.302 #1
CSV 12195 / 视频 12:46.5: <4>[13219.528205][ T1627] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 12196 / 视频 12:46.0: <4>[13219.528209][ T1627] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_133_COL_11 — 显示驱动：cache 维护
源正文行 12232–12325；标题视频位置 12:47.0；异常时间戳 29086.891414（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12272 / 视频 12:48.5: <1>[29086.891360][ T1591} Unable to handle kernel paging request at virtual address ffffffc03337e000
CSV 12283 / 视频 12:48.5: <0>(29086.891414] [ T1591] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 12288 / 视频 12:48.5: <4>[29086.891610] [ T1591] CPU: 1 PID: 1591 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 12291 / 视频 12:50.0: <4>[29086.891634][ T1591] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 12292 / 视频 12:50.0: <4>[29086.891638][ T1591] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_134_COL_11 — 显示驱动：cache 维护
源正文行 12328–12420；标题视频位置 12:52.5；异常时间戳 17858.927696（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12368 / 视频 12:54.0: <1>[17858.927647][ T1586] Unable to handle kernel paging request at virtual address ffffffc038b7e000
CSV 12379 / 视频 12:53.5: <0>[17858.927696][ T1586] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 12386 / 视频 12:54.0: <4>[17858.927915][ T1586] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
CSV 12387 / 视频 12:53.5: <4>([17858.927915][ T1586] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_135_COL_11 — 显示驱动：cache 维护
源正文行 12423–12513；标题视频位置 12:55.5；异常时间戳 134360.645127（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12460 / 视频 12:56.5: <1>[134360.645083][ T1633] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 12471 / 视频 12:58.0: <O>[134360.645127][ T1633] Internal error: Oops: 96000147 [#1] PREEMPT SMP ;
CSV 12474 / 视频 12:58.0: <4>[134360.645279][ T1633] CPU: 1 PID: 1633 Comm: DMS SetMVOP_ Tas Tainted: P woo 5.4.302 #1
CSV 12477 / 视频 12:58.0: <4>[134360.645298] [ T1633] pce : __ inval. dcache_area+0x28/0x58 [mdrv_system]
CSV 12478 / 视频 12:58.0: <4>[134360.645302][ T1633] lr : Chip _Inv_Cache_Ranget+0x94/0x138 ([mdrv_system] 2 $ Pe : : e ¢
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_136_COL_11 — 内核链表损坏
源正文行 12516–12610；标题视频位置 13:00.5；异常时间戳 87489.161569（fatal）。
PC/检测点（OCR）：`_ list_del_entry valid+0xd0/0xf0`。

```text
CSV 12569 / 视频 13:04.0: <2>[87489.161564][ T2101] kernel BUG at /home/gfkfcmo/DockerWorkspace/M9618V_96EAMP_EUG/MT9618VD/device/mtk/9618/apollo/kernel/mixed/5.4 12/lib/list_debug.c:61!
CSV 12570 / 视频 13:04.0: <0>[87489.161569][ T2101] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
CSV 12573 / 视频 13:04.0: <4>[87489.161696] [ T2101} CPU: 1 PID: 2101 Comm: netflix Tainted: P. wo, 5.4.302 #1, “ ‘ . :
CSV 12576 / 视频 13:04.0: <4>[87489.161712][ T2101] pe : _ list_del_entry valid+0xd0/0xf0
CSV 12577 / 视频 13:04.0: <4>[87489.161715][ T2101] lr : _ list_del_entry valid+0xd0/0xf0
```

**判断：** 可确认链表前后关系或节点状态异常；常见候选为重复删除、释放后使用、无锁并发修改或越界覆盖，调用者才决定归属。

**验证：** 保留首次WARN前的节点、prev/next值和插入/删除栈；启用DEBUG_LIST/KASAN，核对锁与生命周期。

### ROW_137_COL_11 — 显示驱动：cache 维护
源正文行 12613–12699；标题视频位置 13:05.0；异常时间戳 2062.277127（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12648 / 视频 13:07.0: <1>[ 2062.277083][ T1644] Unable to handle kernel paging request at virtual address ffffffoff2b7e000
CSV 12659 / 视频 13:07.0: <O>[ 2062.277127][ T1644] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 12662 / 视频 13:07.0: <4>[ 2062.277281][ T1644] CPU: 1 PID: 1644 Comm: DMS’SetMVOP Tas Tainted: P eNO 5.4.302 $1 g z : i :
CSV 12665 / 视频 13:08.5: <4>[ 2062.277299][ T1644] pe : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 12666 / 视频 13:08.5: <4>[ 2062.277304][ T1644] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_140_COL_11 — 显示驱动：cache 维护
源正文行 12875–12971；标题视频位置 13:19.0；异常时间戳 31397.805734（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 12920 / 视频 13:21.0: <1>[(31397.805688] [ T1644] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 12931 / 视频 13:21.0: <0>[31397.805734][ T1644] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 12937 / 视频 13:22.5: <4>[31397.805933][ T1644] pe : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 12938 / 视频 13:22.5: <4>[31397.805937][ T1644] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_141_COL_11 — 显示驱动：cache 维护
源正文行 12974–13059；标题视频位置 13:24.5；异常时间戳 14198.557446（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 13002 / 视频 13:24.5: <1>[14198.557382][ T1607] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 13013 / 视频 13:26.5: <0>[14198.557446][ T1607] Internal error: Oops: 96000147 [#1] PREEMPT SMP i ; ;
CSV 13020 / 视频 13:26.5: <4>[14198.557612][ T1607] CPU: 1 PID: 1607 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 13023 / 视频 13:26.5: <4>[(14198.557638][ T1607] pc : _ inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 13024 / 视频 13:26.5: <4>[14198.557642][ T1607] lr : Chip _Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_143_COL_11 — 显示驱动：Obj/PQ处理
源正文行 13160–13240；标题视频位置 13:35.0；异常时间戳 464802.951119（fault line; fatal header absent）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 13194 / 视频 13:37.5: <1>[464802.951119][ 1535] Unable to handle kernel paging request at virtual address ffffffcO06c72eacd
CSV 13208 / 视频 13:39.0: <4>[(464802.951280)[ 1T535]) CPU: 0 PID: 535 Comm: irg/26-scaler Tainted: P woo §.4.302 #1
CSV 13211 / 视频 13:40.0: <4>[464802.951336][ 1535] pc : MDrv_Obj_OP»Processt+0x1b8/0x1120 [kdrv_alg]
CSV 13212 / 视频 13:39.5: <4>[464802.951370][ T535] lr: MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg] ;
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_150_COL_11 — 音频tasklet：PC未对齐
源正文行 13817–13918；标题视频位置 14:20.5；异常时间戳 107671.950200（fatal）。
PC/检测点（OCR）：`PPU_DFSM_Init+0x8d/0x2a4 [ndp_alsa]（按截图复核）`。

```text
CSV 13886 / 视频 14:23.5: <0>[107671.950200] [ €2) Internal error: SP/PC alignment exception: 8a000000 [#1] PREEMPT SMP - © . . . :
CSV 13888 / 视频 14:23.5: 4<4>[107671.950295] [ C2] CPU: 2 PID: 20 Comm: ksoftirgd/2 Tainted: P ° 5.4.283 #1
CSV 13892 / 视频 14:23.5: <4>[107671.950486] [ C2) lr : tasklet_action_common+0x104/0x140
```

**判断：** 确认执行目标未满足ARM64指令对齐。优先检查tasklet回调指针、对象生命周期、越界写和代码/模块有效性，不能将符号最近邻直接解释为音频算法自身错误。

**验证：** 记录tasklet对象、func指针和初始化/销毁栈，核对实际ELR低位，使用匹配模块检查回调是否在预期函数入口。

### ROW_155_COL_11 — 存储完成/SLAB：诊断期间再异常
源正文行 14302–14386；标题视频位置 14:52.5；异常时间戳 75266.106337（PC timestamp; fatal header truncated）。
PC/检测点（OCR）：`kmem valid obj+0x84/0xa0`。

```text
CSV 14303 / 视频 14:52.5: <4>[(75266.106304] [T24886] CPU: 0 PID: 24886 Comm: kworker/0:0H Tainted: P woo 5.15.189-gbbeaa68710ec-dirty #1
CSV 14308 / 视频 14:55.0: <4>[75266.106337] [T24886] pc : [<a0359330>] lr : [<a0118e74>] psr: 200£0193
```

**判断：** 当前PC落在异常诊断阶段，不能把kmem_valid_obj当作最初根因。应优先核对原始释放故障，候选为错误释放、重复完成、重复释放或slab元数据损坏；没有证据直接认定存储介质坏。

**验证：** 获取首个DataAbort及此前完整日志，对dm-io对象分配、引用和endio逐次编号，启用KASAN/SLUB及双重释放检查。

### ROW_156_COL_11 — 蓝牙驱动：BRK/收发
源正文行 14389–14469；标题视频位置 15:00.5；异常时间戳 671065.372884（fatal）。
PC/检测点（OCR）：`OxffFFFfc00cSas0be |`。

```text
CSV 14434 / 视频 15:02.5: <0>[671065.372884] [ T4110] Internal error: ptrace BRK handler: £20003e8 [#1] PREEMPT SMP
CSV 14437 / 视频 15:02.5: <4>[671065.373063] [ T4110] CPU: 3 PID:, 4110 Comm: hcj_sock_reader Tainted: P WO 5.4.302 #1 x 3 2 am 4
CSV 14440 / 视频 15:02.5: <4>[671065.373092][ T4110] pe : OxffFFFfc00cSas0be |
CSV 14441 / 视频 15:02.5: <4>[671065.373097] [ T4110] lr : btmtk_usb_meta_send_datat+0xdcc/0x13c0 [btmtk_usb_7663]
```

**判断：** 确认软件BRK异常，蓝牙USB收发路径关联明确；具体是厂商断言、异常跳转还是代码损坏，需要精确PC和模块反汇编，不能等同Wi-Fi BSS问题。

**验证：** 将异常PC与btmtk模块加载基址对齐，定位BRK指令及前置条件，保留HCI/USB状态、URB完成和断连、重置事件。

### ROW_157_COL_11 — 显示驱动：cache 维护
源正文行 14472–14563；标题视频位置 15:05.0；异常时间戳 410183.556381（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 14512 / 视频 15:08.5: <1>[410183.556343][ T1646] Unable to handle kernel paging request at virtual address ffffffc032b7e000
CSV 14523 / 视频 15:11.0: <0>[410183.556381] [ T1646] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 14526 / 视频 15:11.0: <4>[410183.556524] [ T1646] CPU: 1 PID: 1646 Comm: DMS SetMVOP_ Tas Tainted: P woo 5.4.302 #1
CSV 14529 / 视频 15:11.0: <4>[410183.556545][ T1646] pc : __ inval_dcache area+0x28/0x58 [mdrv_system]
CSV 14530 / 视频 15:11.0: <4>[410183.556549] [ T1646] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_158_COL_11 — 显示驱动：cache 维护
源正文行 14566–14654；标题视频位置 15:13.0；异常时间戳 17264.775195（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 14602 / 视频 15:14.5: <1>[17264.775153][ T1398] Unable to handle kernel paging request at virtual address ffffffc033b7e000
CSV 14613 / 视频 15:14.5: <0>[17264.775195][ T1398] Internal error: Oops: 96000147 [#1] PREEMPT SMP , : i Z 2 %
CSV 14616 / 视频 15:18.0: <4>[17264.775367] [ T1398] CPU: 1 PID: 1398 Comm: DMS SetMVOP Tas Tainted: P_ WO -5.4.302 #1
CSV 14619 / 视频 15:18.0: <4>[17264.775394][ T1398] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 14620 / 视频 15:18.0: <4>[17264.775398][ T1398] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] -
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_159_COL_11 — 显示驱动：cache 维护
源正文行 14657–14743；标题视频位置 15:19.5；异常时间戳 35451.509612（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 14688 / 视频 15:20.0: <1>(35451.509571][ T1633] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 14699 / 视频 15:21.5: <0>[35451.509612][ T1633] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 14702 / 视频 15:21.5: <4>[35451.509785][ T1633] CPU: 1 PID: 1633 Comm: DMS*SetMVOP_Tas Tainted: P ‘WoO 5.4.302 #1 : : : " :
CSV 14705 / 视频 15:21.5: <4>[35451.509806][ T1633] pe : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 14706 / 视频 15:21.5: <4>[(35451.509810] [ T1633] lr: Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_160_COL_11 — 显示驱动：cache 维护
源正文行 14746–14829；标题视频位置 15:23.0；异常时间戳 82343.672970（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 14778 / 视频 15:25.0: <1>[82343.672941][ T1646] Unable to handle kernel paging request at virtual address ffffffbFFf537e000
CSV 14789 / 视频 15:25.0: <0>[82343.672970][ T1646] Internal error: Oops: 96000145 [#1] PREEMPT SMP ‘
CSV 14792 / 视频 15:27.0: <4>[82343.673105] [ T1646] CPU: 1 PID: 1646 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 14795 / 视频 15:27.0: <4>[82343.673123][ T1646] pce : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 14796 / 视频 15:27.0: <4>[82343.673127][ T1646] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_161_COL_11 — 显示驱动：Obj/PQ处理
源正文行 14832–14912；标题视频位置 15:28.5；异常时间戳 157435.137921（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 14866 / 视频 15:30.0: <1>[157435.137888][ 1541] Unable to handle kernel paging request at virtual address ffffffc09a80974a
CSV 14877 / 视频 15:30.0: <0>(157435.137921])[ 1541) Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 14880 / 视频 15:30.0: <4>[157435.138041][ 1541] CPU: 0 PID: 541 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1
CSV 14883 / 视频 15:30.0: <4>[(157435.138088][ 1541] pc: MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 14884 / 视频 15:30.0: <4>(157435.138120)[ 1541] lr : MDrv_Obj_OP_Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_163_COL_11 — 显示驱动：cache 维护
源正文行 15002–15101；标题视频位置 15:37.0；异常时间戳 753173.335405（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15050 / 视频 15:40.0: <1>[753173.335355][ T1602] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 15061 / 视频 15:40.0: <0>[753173.335405][ T1602] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 15064 / 视频 15:40.5: <4>[753173.335596] [ T1602] CPU: 1 PID: 1602 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 15067 / 视频 15:40.5: <4>[753173.335616][ T1602] pc : _ inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 15068 / 视频 15:40.5: <4>[753173.335620][ T1602] lr : Chip _Inv_Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_164_COL_11 — 显示驱动：cache 维护
源正文行 15104–15189；标题视频位置 15:42.5；异常时间戳 205068.970212（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15138 / 视频 15:44.0: <1>[205068.970170][ T1269] Unable to handle kernel paging request at virtual address fffFffc033b7e000
CSV 15149 / 视频 15:44.0: <0>(205068.970212] [ T1269) Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 15152 / 视频 15:44.0: <4>[205068.970366][ T1269] CPU: 1 PID: 1269 Comm: DMS SetMVOP_Tas Tainted: P woo $.4.302 #1
CSV 15155 / 视频 15:44.0: <4>[205068.970386][ T1269] pc: __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 15156 / 视频 15:44.0: <4>[205068.970390][ T1269] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_165_COL_11 — 显示驱动：cache 维护
源正文行 15192–15283；标题视频位置 15:45.5；异常时间戳 367208.717108（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15232 / 视频 15:48.5: <1>[(367208.717073] [ T1283] Unable to handle kernel paging request at virtual address ffffffc033b7e000
CSV 15243 / 视频 15:50.5: <0>[(367208.717108] [ T1283] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 15246 / 视频 15:50.5: <4>[367208.717261][ T1283] CPU: 1 PID:* 1283.Comm: DMS _SetMVOP_Tas Tainted: P ‘.W Oo 5.4.302 #1 : + : ‘ :
CSV 15249 / 视频 15:50.5: <4>[367208.717279][ T1283] pc : __ inval_dcache_area+0x28/0x58 [mdrv_system] iS
CSV 15250 / 视频 15:50.5: <4>[367208.717283] [ T1283] lr : Chip_Inv_Cache_Ranget+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_166_COL_11 — 显示驱动：Obj/PQ处理
源正文行 15286–15366；标题视频位置 15:51.5；异常时间戳 289613.288789（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 15320 / 视频 15:52.5: <1>[289613.288754][ 1524] Unable to handle kernel paging request at virtual address ffffffc11454ffad
CSV 15331 / 视频 15:52.5: <0>[289613.288789][ 1T524] Internal error: Oops: 96000005 [#1] PREEMPT SMP |
CSV 15334 / 视频 15:54.5: <4>[289613.288935][ 1524] CPU: 0 PID: 524 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1 c : ‘ . :
CSV 15337 / 视频 15:54.5: <4>[289613.288999][ 1524] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
CSV 15338 / 视频 15:54.5: <4>[289613.289031][ 1524] lr : MDrv_Obj_OP Process+0x154/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_167_COL_11 — 显示驱动：MCNR上下文
源正文行 15369–15452；标题视频位置 15:56.5；异常时间戳 13873.892583（fatal）。
PC/检测点（OCR）：`MDrv_NR_SetMcnrSwRegt0x658/0x21b8 [kdrv_alg]`。

```text
CSV 15399 / 视频 15:56.5: <1>[13873.892549][ 1542] Unable to handle kernel write to read-only memory at virtual address .0000000000000090
CSV 15410 / 视频 15:58.5: <0>[13873.892583][ 1542] Internal error: Oops: 96000045 [#1] PREEMPT SMP
CSV 15413 / 视频 15:58.5: <4>[13873.892717][ 1T542] CPU: 0 PID: 542 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 15416 / 视频 16:00.0: <4>(13873.892777][ 1542] pc : MDrv_NR_SetMcnrSwRegt0x658/0x21b8 [kdrv_alg]
CSV 15417 / 视频 16:00.0: <4>[13873.892811][ 1542] lr : MDrv_NR_SetMcnrSwRegt0x658/0x21b8 [kdrv_alg]
```

**判断：** 低地址写入支持空/无效MCNR上下文及字段偏移的方向。仍需用故障指令和寄存器确认，重点检查上下文映射、配置切换、IRQ与释放顺序。

**验证：** 反汇编kdrv_alg中的故障store，核对基址与0x90关系，记录MCNR context创建/映射/失效及各线程引用。

### ROW_168_COL_11 — 显示驱动：cache 维护
源正文行 15455–15540；标题视频位置 16:00.5；异常时间戳 195769.896996（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15482 / 视频 16:02.0: <1>[(195769.896949] [ T1626] Unable to handle kernel paging request at virtual address ffffffc03437e000
CSV 15493 / 视频 16:02.0: <0>[195769.896996][ T1626] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 15496 / 视频 16:02.0: <4>[195769.897129][ T1626] CPU: 1 PID: 1626 Comm: DMS SetMVOP Tas Tainted: P SEW 20. 5.4.302 #1 5 Z ; i $
CSV 15499 / 视频 16:03.5: <4>[195769.897150][ T1626] pc : __inval_dcache_area+0x28/0x58 [mdrv_system]
CSV 15500 / 视频 16:03.5: <4>[(195769.897154][ T1626] lr : Chip_Inv_Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_169_COL_11 — 显示驱动：Obj/PQ处理
源正文行 15543–15624；标题视频位置 16:06.0；异常时间戳 728584.578195（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 15578 / 视频 16:08.5: <1>(728584.578162][ 1554) Unable to handle kernel paging request at virtual address ffffffc06a917b42
CSV 15589 / 视频 16:08.5: <0>[728584.578195][ 1T554] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 15592 / 视频 16:08.5: <4>[728584.578353][ 1554] CPU: 0 PID: 554 Comm: irg/26-scaler Tainted: P woo 5.4.302 #1
CSV 15595 / 视频 16:08.5:  <4>[728584.578428][ 1554] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg]
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_170_COL_11 — 显示驱动：cache 维护
源正文行 15627–15713；标题视频位置 16:10.5；异常时间戳 198611.940091（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15661 / 视频 16:13.0: <1>(198611.940023] [ T1627] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 15672 / 视频 16:13.0: <0>[(198611.940091][ T1627] Internal error: Oops: 96000147 [#1] PREEMPT SMP ; z é é # uy s
CSV 15675 / 视频 16:16.0: <4>[198611.940289][ T1627] CPU: 1 PID: 1627 Comm: DMS _SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 15678 / 视频 16:16.0: <4>[198611.940315][ T1627] pe : __inval_dcache area+0x28/0x58 [mdrv_system]
CSV 15679 / 视频 16:16.0: <4>[198611.940319][ T1627] lr : Chip_Inv_Cache_Ranget0x94/0x138 [mdrv_system] : ‘ z 5 4
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_171_COL_11 — 显示驱动：Obj/PQ处理
源正文行 15716–15803；标题视频位置 16:18.5；异常时间戳 73272.804987（fatal）。
PC/检测点（OCR）：`MDrv_Obj_OP_Process`。

```text
CSV 15757 / 视频 16:20.0: <1>(73272.804953][ T1556] Unable to handle kernel paging request at virtual address ffffffc04681lec50
CSV 15768 / 视频 16:20.0: 4<O>(73272.804987][ 1556] Internal error: Oops: 96000005 [#1] PREEMPT SMP
CSV 15771 / 视频 16:22.0: <4>[73272.805126][ T556] CPU: 0 PID: 556 Comm: irq/26-scaler Tainted: P woo 5.4.302 #1
CSV 15774 / 视频 16:22.0: <4>[73272.805188][ 1556] pc : MDrv_Obj_OP_Process+0x1b8/0x1120 [kdrv_alg] i ; ; i ;
CSV 15775 / 视频 16:22.0: <4>(73272.805220][ T1556) lr : MDrv_Obj_OP_Process+0x154/0x1120 (kdrv_alg] 5 : MA s . P :
```

**判断：** 多个现场在同一函数偏移出现非法读取，优先怀疑对象/表指针、元素索引或对象生命周期。相同落点说明适合合并排查，尚不足以证明所有现场同一根因。

**验证：** 对 kdrv_alg.ko 的实际故障指令解码，确定基址寄存器与字段偏移；记录对象地址、索引、表长及创建/释放代次；检查线程化IRQ与PQ重配置、关闭、销毁之间的同步。

### ROW_172_COL_11 — 显示驱动：cache 维护
源正文行 15806–15898；标题视频位置 16:22.0；异常时间戳 19541.745804（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15846 / 视频 16:25.0: <1>[19541.745766] [ T1618] Unable to handle kernel paging request at virtual address ffffffc03337e000
CSV 15857 / 视频 16:25.0: <0>[19541.745804][ T1618] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 15860 / 视频 16:25.0: <4>[19541.745974] [ T1618] CPU: 1 PID: 1618 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 15863 / 视频 16:25.0: <4>[19541.745995][ T1618] pe : __inval_dcache_area+0%28/0x58 [mdrv_system] ‘ 2 i z 2 ex ye
CSV 15864 / 视频 16:25.0: <4>[(19541.745999][ T1618] lr : Chip_Inv_Cache Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_173_COL_11 — 显示驱动：cache 维护
源正文行 15901–15986；标题视频位置 16:27.0；异常时间戳 11673.436672（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 15932 / 视频 16:28.0: <1>[11673.436633][ T1642] Unable to handle kernel paging request at virtual address ffffffbff4b7e000
CSV 15943 / 视频 16:28.0: <0>[11673.436672][ T1642] Internal error: Oops: 96000145 [#1] PREEMPT SMP :
CSV 15946 / 视频 16:31.0: <4>[11673.436837] [ T1642] CPU: 1 PID: 1642 Comm: DMS SetMVOP Tas Tainted: P wo 5.4.302 #1
CSV 15949 / 视频 16:31.0: <4>[11673.436858] [ T1642] pc : __ inval_dcache_areat+0x28/0x58 [mdrv_systeq]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_174_COL_11 — 显示驱动：cache 维护
源正文行 15989–16079；标题视频位置 16:33.0；异常时间戳 642036.604512（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 16026 / 视频 16:35.0: <1>[(642036.604473][ T1626] Unable to handle kernel paging request at virtual address ffffffbff2b7e000
CSV 16037 / 视频 16:35.0: <0>[642036.604512][ T1626] Internal error: Oops: 96000145 [#1] PREEMPT SMP
CSV 16040 / 视频 16:35.0: <4>[642036.604693][ T1626] CPU: 1 PID: 1626 Comm: DMS SetMVOP Tas Tainted: P a a O. 5.4.302 #1 ‘ f : i :
CSV 16043 / 视频 16:36.5: <4>[642036.604714][ T1626] pc : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 16044 / 视频 16:36.5: <4>[642036.604718][ T1626] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_175_COL_11 — 显示驱动：cache 维护
源正文行 16082–16168；标题视频位置 16:40.0；异常时间戳 281185.868395（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 16115 / 视频 16:42.5: <1>[281185.868324] [ T1643] Unable to handle kernel paging request at virtual address ffffffc034b7e000
CSV 16126 / 视频 16:42.5: <0>[281185.868395][ T1643] Internal error: Oops: 96000147 [#1] PREEMPT
CSV 16131 / 视频 16:42.5: <4>[281185.868682][ T1643] CPU: 1 PID: 1643 Comm: DMS SetMVOP_Tas Tainted: P wo 5.4.302 #1
CSV 16134 / 视频 16:42.5: <4>[(281185.868721][ T1643] pc : _ inval dcache_areat+0x28/0x58 [mdrv_system]
CSV 16135 / 视频 16:42.5: <4>[281185.868725][ T1643] lr : Chip _Inv_Cache_Ranget0x94/0x138 [mdrv_system] - : i z 2 ex ye
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_176_COL_11 — 蓝牙驱动：BRK/收发
源正文行 16171–16251；标题视频位置 16:44.0；异常时间戳 430204.824869（fatal）。
PC/检测点（OCR）：`OxffFFffc00cSeeOdbe`。

```text
CSV 16216 / 视频 16:46.5: <0>[430204.824869][ T3950] Internal error: ptrace BRK handler: £20003e8 [#1] PREEMPT SMP : : é # Hy :
CSV 16219 / 视频 16:48.5: <4>[430204.825027] [ T3950] CPU: 0 PID: 3950 Comm: hci_sock_reader Tainted: P woo 5.4.302 #1
CSV 16222 / 视频 16:48.5: <4>[430204.825050][ T3950] pe : OxffFFffc00cSeeOdbe
CSV 16223 / 视频 16:48.5: <4>[430204.825055] [ T3950) lr : btmtk_usb_meta_send_data+0xdcc/0x13c0 [btmtk_usb_7663] E ‘ . . r z
```

**判断：** 确认软件BRK异常，蓝牙USB收发路径关联明确；具体是厂商断言、异常跳转还是代码损坏，需要精确PC和模块反汇编，不能等同Wi-Fi BSS问题。

**验证：** 将异常PC与btmtk模块加载基址对齐，定位BRK指令及前置条件，保留HCI/USB状态、URB完成和断连、重置事件。

### ROW_178_COL_11 — 显示驱动：cache 维护
源正文行 16336–16433；标题视频位置 16:55.0；异常时间戳 222344.977122（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 16382 / 视频 16:57.5: <1>[222344.977084][ T1627] Unable to handle kernel paging request at virtual address ffffffc038b7e000
CSV 16393 / 视频 16:59.0: <0>[222344.977122][ T1627] Internal error: Oops: 96000146 [#1] PREEMPT SMP
CSV 16396 / 视频 16:59.0: <4>[222344.977278][ T1627] CPU: 1 PID: 1627 Comm: DMS SetMVOP Tas Tainted: P woo 5.4.302 #1
CSV 16399 / 视频 16:59.0: <4>[222344.977300] [ T1627] pc: __inval. dcache_area+0x28/0x58 [mdrv_system]
CSV 16400 / 视频 17:00.0: <4>[222344.977304][ T1627] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system] : © . : % ©
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_179_COL_11 — 显示驱动：cache 维护
源正文行 16436–16526；标题视频位置 17:01.5；异常时间戳 4858.933904（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 16474 / 视频 17:04.0: <1>[ 4858.933863][ T1671] Unable to handle kernel paging request at virtual address ffffffc03537e000
CSV 16485 / 视频 17:04.0: <O>[ 4858.933904][ T1671] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 16488 / 视频 17:05.5: <4>[ 4858.934065] [ T1671] CPU: 1 PID: 1671 Comm: DMS:SetMVOP_Tas Tainted: P "WO §.4.302 #1 . . .
CSV 16491 / 视频 17:05.5: <4>[ 4858.934085][ T1671] pce : __inval_dcache_areat+0x28/0x58 [mdrv_system]
CSV 16492 / 视频 17:05.5: <4>[ 4858.934089][ T1671] lr : Chip_Inv_Cache_Range+0x94/0x138 [mdrv_system]
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

### ROW_180_COL_11 — 显示驱动：cache 维护
源正文行 16529–16619；标题视频位置 17:08.0；异常时间戳 160640.808002（fatal）。
PC/检测点（OCR）：`__inval_dcache_area`。

```text
CSV 16566 / 视频 17:09.5: <1>[160640.807957] [ T1631] Unable to handle kernel paging request at he address ff£ffffc034b7e000
CSV 16577 / 视频 17:09.5: <0>[160640.808002] [ T1631] Internal error: Oops: 96000147 [#1] PREEMPT SMP
CSV 16580 / 视频 17:09.5: <4>[160640.808192] [ T1631] CPU: 1 PID: 1631 Comm: DMS SetMVOP_Tas Tainted: P woo 5.4.302 #1
CSV 16583 / 视频 17:09.5: <4>[160640.808212][ T1631] pe : _ inval_dcache_areat+0x28/0x58 [mdrv_system] ; |
CSV 16584 / 视频 17:09.5: <4>[160640.808216][ T1631] lr : Chip _Inv_Cache Range+0x94/0x138 [mdrv_system] f z 4 i 3 : %
```

**判断：** cache 维护指令触及无效虚拟映射。优先检查显示/HDR缓冲区的地址、长度、映射存续期，以及释放/重配与DMS线程的竞态。不能据此认定CPU cache硬件坏了，也不能直接认定内存不足。

**验证：** 在 Chip_Inv_Cache_Range 入口记录 CPU VA、物理/DMA地址、长度、buffer对象ID与代次、映射和释放栈；反汇编相同构建的 mdrv_system.ko，确认故障指令操作x0还是x1，核对目标PTE和区间端点。

## 定位还需要的数据

需要每个平台、每个版本对应的 vmlinux、System.map、.config、厂商模块及加载地址，以及原始pstore/ramoops文本。32位ARM与64位ARM、5.4.283/5.4.302/5.15系列不可混用偏移。优先在复现环境开启KASAN、SLUB_DEBUG、DEBUG_LIST及timer/RCU对象调试，并记录首次损坏，而非只分析最终panic。
