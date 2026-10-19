# 非Wi-Fi直接崩溃路径的原因分析

## 结论与范围修正

对原先按PC归为“其他”的33个去重事件逐条重读完整栈后，可以确认若干直接故障机制，但不能仅由视频OCR确认最初写坏对象的模块或唯一根因。

上一版统计中的ROW_51（MT7663收包→eth_type_trans）、ROW_99（BSS释放→kfree）、ROW_105（MT7663 RSN解析→do_undefinstr）应归入Wi-Fi关联调用路径。按该口径Wi-Fi关联事件至少163/193，原来“其他”33例中剩30例。ROW_26仍含无线事件通知链，保留为关联待定；剩余记录也不能据PC位置证明与Wi-Fi完全无关。ROW_83最终致命PC应为ip_mc_inc_group，前面的refcount_warn_saturate是WARN点。

以下分析使用日志中的5.4.283版本与上游源码对照，但厂商补丁、配置和编译方式会改变偏移、字段和行号；最终需使用同一次固件的vmlinux、System.map、.config和模块核对。

## 能确认得较清楚的机制

1. **ROW_59/129：dm-bufio读完成状态断言。** 同一read_endio与dm-bufio.c:1024，在下载核对的5.4.283源码中是`BUG_ON(!test_bit(B_READING, &b->state))`。读完成时状态位不在预期状态；重复end_io、提前清状态、对象复用/UAF、内存覆盖均为候选，当前不能唯一选择其一。
2. **ROW_96：系统级OOM无法继续处理。** 明文panic由out_of_memory路径触发。对应上游分支在没有合适可杀进程时退出。日志slab_reclaimable=107236页、slab_unreclaimable=31493页（4KiB页约419MiB和123MiB），free=271260kB中free_cma=240668kB，普通slab分配不能将CMA页一概视为可用。优先核查slab占用、回收/GFP约束和watermark，而非仅因出现Mali notifier便判GPU泄漏。
3. **ROW_102/127：内核栈canary被改写。** 能确认栈保护检测失败；检测函数futex_time32/do_idle不是已证明的越界写位置。
4. **ROW_138/153：NULL调度实体模式。** x1=0、故障指令和FAR=0x40相互吻合，同一直接机制出现两次。CFS为何返回/接收空实体需检查rq/实体生命周期。
5. **ROW_84/168：中断回调的取指权限异常。** IABT且PC固定0xffffff802a81e800、LR在handle_percpu_devid_irq；优先核对IRQ handler指针与目标页PTE。不能把异常下面的Binder/音频调用栈直接当作犯错模块。
6. **ROW_193：AV1帧列表地址计算后写入低地址。** 指令和寄存器算出的0x300c与FAR一致，能确定非法地址写入；缺少厂商模块反汇编，尚不能区分基址失效、索引错误或更早覆盖。
7. **ROW_184：SDHCI处理中低地址指针解引用。** x19=0x30与字节加载偏移0xd解释FAR=0x3d；具体指针对应cmd、mrq还是厂商额外参数必须由厂商汇编确认。

## 可能的共同故障方向

栈canary变化、slab内部坏指针、vmap树断言、RCU链表异常、调度实体NULL和IRQ异常跳转，共同支持“内核对象/控制数据被更早破坏”作为优先调查方向。它们不证明都来自同一个缺陷，更不能仅据这些模式判硬件DDR故障。

复核原画面还确认ROW_59前有SSID len 153、ROW_117前有SSID len 185，并带有超出正常文本的控制字符。Linux定义的SSID最大长度为32。这是值得优先追查的长度/对象生命周期/日志参数疑点，但不是已经证明的溢出写或Wi-Fi根因；需要检查打印参数及内存拷贝的真实长度。

ROW_0/153中pstore缺/dev/mmcblk0p24出现在初次Oops后，属于故障日志持久化失败，不能用来解释最初crash。

## 逐条分析（30例，已去重）

| ROW | 视频记录头位置 | 调查方向 | 已观察到的直接机制 | 原因判断/候选 | 置信度 | 下一步 |
|---|---|---|---|---|---|---|
| 0 | 00:00.0 | 定时器/idle | tick_nohz_next_event向非正常地址写入，随后idle task触发panic。 | 对象基址或每CPU tick相关数据失效/遭覆盖；第二次Oops属于同一次故障扩散。 | 高：异常机制；低：最初写坏者 | 用匹配vmlinux解析+0xb0，追踪x19对应对象；pstore缺分区发生在Oops后，另修日志持久化。 |
| 2 | 00:13.0 | Binder/uaccess | Binder兼容ioctl的copy_to_user期间出现ttbr address size fault，多个线程几乎同时失败。 | 页表/地址翻译状态或uaccess路径异常；给用户指针错误本身不应直接导致内核Oops。 | 中 | 核对原始ESR/FAR/TTBR、页表及compat Binder参数；匹配厂商uaccess补丁。 |
| 13 | 01:12.0 | PQ/DRM未定义指令 | kernel-mode do_undefinstr触发traps.c:405；异常返回链指向PQ HLINE中断。 | 真正触发点应沿el1_undef下方PQ/DRM路径查找，可能非法指令、错误跳转、模块生命周期或代码破坏。 | 高：内核态未定义指令；低：成因 | 解码原始异常现场ELR/指令，检查PLAT_PQ_Mpfw_IRQ_DP_HLINE_Routine及中断注册/卸载。 |
| 15 | 01:23.0 | hrtimer | 定时器中断遍历队列时访问失效内核地址，多CPU同时报错。 | timer/task对象释放、队列损坏或更早的内存破坏；Binder是被中断的上下文，不能直接判为Binder原因。 | 中 | DEBUG_OBJECTS_TIMERS、KASAN；跟踪timer init/start/cancel/free时序。 |
| 22 | 01:56.0 | 调度唤醒 | futex_wake进入wake_affine_weight后访问0x17。 | task/runqueue/调度域数据异常，可能对象生命周期或内存覆盖。 | 中 | 反汇编+0x84与寄存器对照，检查被唤醒task、CPU/rq及退出/迁移并发。 |
| 26 | 02:18.0 | slab/无线网络通知关联 | __kmalloc_track_caller在netlink skb扩容时解引用异常地址。 | slab freelist或分配器对象异常，比一般分配失败更吻合；调用链含wireless_nlevent_flush，仍与无线通知场景关联。 | 中 | 检查slab freelist、相关skb分配释放与无线事件通知；不能将其视为已排除Wi-Fi的故障。 |
| 48 | 04:36.0 | PQ/vmalloc | PQ_OS_MemAlloc→vmalloc→insert_vmap_area触发BUG。 | vmap区间或树元数据不一致，候选为区间重叠、重复插入、释放/插入竞态或内存破坏；不是仅凭分配栈就能判内存不足。 | 中高 | 厂商mm/vmalloc.c:477对应断言；转储新旧va_start/va_end、rb树和分配/释放调用栈。 |
| 59 | 06:08.0 | dm-bufio读完成状态 | read_endio触发drivers/md/dm-bufio.c:1024。与上游5.4.283的B_READING断言匹配。 | 读完成时buffer已不处于B_READING；候选为重复完成、过早清状态、对象复用/UAF或状态被覆盖。 | 高：断言条件；中低：触发来源 | 记录dm_buffer地址/状态、bio提交和end_io次数、释放栈；结合MMC与device-mapper链核对。 |
| 83 | 08:54.0 | IPv4组播对象/引用计数 | 先refcount_warn_saturate，后在ip_mc_inc_group发生0x2c1地址异常。 | 组播成员对象/引用计数或相关链表已不一致；需区分重复释放、对象复用和其他内存覆盖。 | 高：实际崩溃位置；中：对象损坏方向 | 以致命Oops的ip_mc_inc_group为准，不把之前WARN位置误当最终PC；检查join/leave与设备销毁并发。 |
| 84 | 09:00.0 | per-CPU IRQ异常执行 | EC=0x21 IABT，PC=0xffffff802a81e800，LR在handle_percpu_devid_irq。 | CPU试图在不可执行/权限异常页取指；IRQ handler指针失效或该地址页权限异常为主要候选。 | 高：取指异常；中：handler/页属性方向 | 用System.map/vmlinux解析目标PC，检查IRQ action->handler、模块卸载、页表PTE及执行权限。不能因被中断的Binder栈就归因Binder。 |
| 87 | 09:16.0 | PSI/中断 | psi_memstall_tick访问0x0000000100010020。 | 压力统计或传入对象指针不正常，疑似更早对象破坏；需核实厂商中断调用路径。 | 中 | 反汇编+0x70及调用点，核对参数、对象生命周期和每CPU数据。 |
| 93 | 09:44.0 | 调度/超时唤醒 | 超时处理唤醒线程后在check_preempt_wakeup访问0x150。 | sched_entity/rq或task数据异常，空指针字段访问/对象生命周期问题为候选。 | 中 | 核对对应寄存器和字段偏移；跟踪超时任务退出、唤醒、迁移。 |
| 96 | 10:02.0 | OOM内存分配约束 | out_of_memory调用panic，明文System is deadlocked on memory；上游对应无合适可杀进程的系统级OOM分支。 | 普通可用内存/回收能力不足，可能slab占用、CMA保留、watermark或不可回收资源问题；不能仅据Mali OOM notifier断定GPU泄漏。 | 高：直接panic分支；中：资源方向 | 抓slabtop/slabinfo、GFP/order、zoneinfo、CMA、min_free_kbytes和OOM候选。日志free约271260kB，其中free_cma约240668kB，不能把全部free视为本次slab可用。 |
| 102 | 10:36.0 | 内核栈破坏 | stack-protector在__arm64_sys_futex_time32返回检查发现canary不匹配。 | 此前发生了栈数据覆盖；可能越界写、野指针或其他写坏，检查点不等同于写坏点。 | 高：栈破坏；低：首次写入位置 | KASAN/栈保护调试，捕获canary和栈周边；检查系统调用及中断嵌套路径。 |
| 117 | 11:50.0 | dm-verity预读/slab | dm_bufio_prefetch→submit_io→bio_alloc_bioset→__kmalloc，访问低地址0x6f6c。 | 更符合slab freelist/对象元数据坏指针，不是简单OOM；之前RCU WARN及异常SSID是关联线索，尚无因果证明。 | 中高 | SLUB_DEBUG/KASAN记录分配释放；同时追查RCU头和异常SSID长度的原始产生位置。 |
| 121 | 12:17.5 | TCP/socket生命周期 | TCP收包进入sock_def_readable时发生非法地址访问。 | socket或sk_wq/waitqueue指针损坏/生命周期问题为候选。 | 中 | 对应+0x24汇编确认具体字段；检查close/接收/回调与RCU释放顺序。 |
| 127 | 12:52.0 | idle内核栈破坏 | stack-protector指出do_idle中的canary遭破坏。 | idle任务或中断栈附近发生覆盖；返回do_idle时才被发现。 | 高：栈破坏；低：首次写入位置 | 检查IRQ嵌套栈、跨CPU写入、DMA范围、调试版栈保护与KASAN。 |
| 129 | 13:00.0 | dm-bufio读完成状态 | 与ROW_59相同：read_endio、dm-bufio.c:1024、同一条BUG条件。 | 可与ROW_59作为同类缺陷簇排查，但属于两个不同故障事件。 | 高：同类断言；中低：底层来源 | 合并比对两例dm_buffer/bio状态和完成次数，不直接归因为eMMC物理故障。 |
| 138 | 13:39.0 | 调度实体NULL | set_next_entity+0x10，x1=0，指令b9404028对应ldr w8,[x1,#0x40]，故障地址0x40。 | 传入的调度实体指针为NULL，与上游第二参数sched_entity*吻合；为何会传空尚需rq/CFS状态。 | 高：NULL读取；中：参数对象映射 | 匹配vmlinux确认se->on_rq偏移；检查pick_next_task_fair、rq树与任务退出/组调度。 |
| 147 | 14:28.0 | VFS路径对象 | openat路径在inode_permission访问0x10。 | inode/关联元数据缺失或失效，普通文件权限拒绝不应触发Oops。 | 中 | 解析+0x88对应字段；跟踪dentry/inode生命周期和文件系统损坏/并发。 |
| 151 | 15:04.0 | RCU回调链表 | rcu_cblist_dequeue访问0xfb0000e0，处于RCU softirq。 | RCU回调头或next指针损坏；候选为同一rcu_head重复入队、过早释放或内存覆盖。 | 中高 | CONFIG_DEBUG_OBJECTS_RCU_HEAD和KASAN；记录call_rcu/kfree_rcu的对象和调用栈。 |
| 153 | 15:20.0 | 调度实体NULL | 与ROW_138相同PC/指令/x1=0和0x40，之后多个CPU出现二次Oops和idle panic。 | 同一NULL sched_entity故障模式；后续panic是扩散结果，不能多算。 | 高：同一直接机制；中：具体CFS对象来源 | 与ROW_138合并定位同一代码分支；pstore分区缺失仅影响事后落盘。 |
| 157 | 15:40.0 | 显示缩放器/slab | ndp_video帧缓冲配置路径中__kmalloc解引用0x0000000500000000。 | 内存分配器内部/对象指针异常，可能freelist破坏；调用者是显示缩放路径，不能仅凭PC判通用OOM。 | 中高 | 跟踪VPL_SCLR_FB_Alloc、OS_SHL_MemAlloc和dma/framebuffer边界；KASAN/SLUB验证早期破坏者。 |
| 168 | 16:48.0 | per-CPU IRQ异常执行 | 与ROW_84同PC和IABT/LR模式；ROW_169/197是本例重复日志。 | IRQ回调目标或页执行权限异常；下面音频poll栈只是被中断的上下文，不足以证明音频驱动是根因。 | 高：直接异常；中：回调/页属性 | 比较两次IRQ注册信息及PTE，确认目标地址属于代码还是数据/已释放内存。 |
| 176 | 17:46.0 | 调度队列 | __schedule中访问低地址0xbd4c。 | task/rq/调度实体数据不一致，存在坏指针；不能仅凭调度函数名判调度器自身bug。 | 中 | 用精确+0x238汇编与寄存器查出是哪一个结构指针，再追踪其来源。 |
| 180 | 18:16.0 | DRM PRIME未定义指令 | do_undefinstr的下层现场在drm_prime_add_buf_handle，用户weston经PRIME ioctl进入。 | PRIME handle/buffer导入路径触发内核态未定义指令；需查原始异常PC、指令及模块ABI/生命周期。 | 高：异常机制；中低：根因 | 检查ndp_drm与DRM PRIME对象、模块和内核构建一致性，不能将BUG处理函数当原始犯错函数。 |
| 182 | 18:30.0 | 任务迁移/唤醒 | hrtimer唤醒线程，migrate_task_rq_fair访问20fd58694b15479c。 | 明显非正常指针，倾向task/调度实体字段覆盖或失效。 | 中高 | 解码+0x24确定被解引用字段；关联task释放、迁移并发与内存破坏。 |
| 184 | 18:50.0 | SDHCI命令完成/生命周期 | sdhci_send_command，x19=0x30，39403668=ldrb w8,[x19,#0xd]，0x30+0xd=FAR 0x3d。 | 命令处理相关指针已成为低地址0x30；可能错误参数、失效对象或cmd/mrq完成状态问题。厂商函数使用x2，不能套上游两参数签名。 | 高：坏指针解引用；中低：指针身份 | 用厂商sdhci_send_command反汇编映射x2/x19，检查finish_command后下一条命令、超时/重置/IRQ并发。 |
| 192 | 19:44.0 | ADC/timer未定义指令 | do_undefinstr下方del_timer_sync→schedule_timeout→msleep→ndp_preinit ADC线程。 | timer/调用代码或跳转目标异常，需查原始异常指令与timer对象；不够证据直接判定del_timer_sync实现bug。 | 高：异常机制；低：原始破坏点 | DEBUG_OBJECTS_TIMERS、精确vmlinux及ADC线程退出/休眠时序。 |
| 193 | 19:49.0 | AV1帧列表坏地址写入 | AV1_update_frmlist_data中x9=0x173c，b918d128=str w8,[x9,#0x18d0]，目标地址0x300c与FAR一致。 | AV1帧列表地址计算使用了不合法的小基址；候选为空/失效基址、索引越界或状态破坏。 | 高：写坏地址机制；中：根因范围 | 对齐ndp_codec.ko+0x114，追溯x9的计算、frame list基址与index及reset/flush并发，保留触发AV1码流。 |

## 建议的排查顺序

1. 收集每个固件匹配的vmlinux/System.map/.config及ndp_codec、ndp_video、ndp_pq、ndp_drm、wlan模块。用PC/LR偏移定位指令和字段；OCR中的十六进制值必须对原始panic再核对。
2. 调试固件优先启用KASAN、SLUB_DEBUG/alloc-free跟踪，捕获第一次越界/UAF，而非只分析受害函数。带性能和内存开销的选项先用于复现版本。
3. 对RCU/timer分别启用DEBUG_OBJECTS_RCU_HEAD、DEBUG_OBJECTS_TIMERS，定位重复回调入队和释放时序。
4. 对dm-bufio记录buffer/bio地址、状态转移和提交/完成计数；对OOM抓slabinfo、zoneinfo、CMA及触发分配GFP/order。
5. 对AV1与SDHCI分别保留触发码流/帧列表状态，以及命令完成、超时、重置与IRQ序列。对IRQ错误PC检查handler注册/卸载和页权限。

## 源码依据

- [dm-bufio read_endio（5.4.283）](https://github.com/gregkh/linux/blob/v5.4.283/drivers/md/dm-bufio.c#L1020)
- [OOM无可杀进程分支](https://github.com/gregkh/linux/blob/v5.4.283/mm/oom_kill.c)
- [内核栈保护失败处理](https://github.com/gregkh/linux/blob/v5.4.283/kernel/panic.c#L728)
- [set_next_entity](https://github.com/gregkh/linux/blob/v5.4.283/kernel/sched/fair.c#L4212)
- [IRQ per-CPU handler调用](https://github.com/gregkh/linux/blob/v5.4.283/kernel/irq/chip.c#L923)
- [内核态未定义指令处理](https://github.com/gregkh/linux/blob/v5.4.283/arch/arm64/kernel/traps.c#L396)
- [SSID长度上限](https://github.com/gregkh/linux/blob/v5.4.283/include/linux/ieee80211.h)

本报告为视频证据与上游源码结合的初步诊断。没有对应厂商源码/符号与原始dump，不能把候选原因提升为已确认根因。
