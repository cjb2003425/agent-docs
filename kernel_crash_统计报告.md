# Kernel crash 次数与分类统计

**统计结果：198条含crash证据的日志记录；识别并合并4组重复记录（重复5条）后，得到193个不同日志指纹对应的crash事件。**

## 统计口径

- 以每条ROW记录中的一次故障过程为一个事件；同一过程中的多CPU报错、二次Oops、panic调用栈和panic IPC提示不重复计数。
- 每条记录均找到 Internal error/Oops、BUG 或 Kernel panic - not syncing 证据。普通WLAN WARN或WARNING栈不单独计为crash。
- 去重以异常/Oops时刻、机型/版本/地区、进程与调用栈特征共同核对。193是日志指纹去重统计，无法替代设备端实际发生次数；也不能据不同指纹判断是否为同一根因。
- 崩溃位置采用与致命异常对应的PC/调用栈，排除之前的WARN栈；例如ROW_59归入read_endio，ROW_117/157归入__kmalloc，ROW_53归入最终P2P故障。
- 数据来自视频OCR重建。分类对cmp/emp、下划线/空格等明显误读进行了归一化，并以调用栈复核；原始字段保留在逐条JSON中。

## 按崩溃位置分类

| 类别 | 原始记录 | 去重事件 | 去重占比 |
|---|---:|---:|---:|
| Wi-Fi：cmp_bss / BSS比较 | 127 | 125 | 64.8% |
| Wi-Fi：rb_erase / BSS删除/更新 | 33 | 33 | 17.1% |
| 调度/定时器/PSI | 9 | 9 | 4.7% |
| 内存分配/释放/vmap | 5 | 5 | 2.6% |
| BUG/do_undefinstr路径 | 5 | 4 | 2.1% |
| 存储/块设备 | 3 | 3 | 1.6% |
| 网络收包/socket | 2 | 2 | 1.0% |
| Wi-Fi：P2P/消息处理触发panic | 2 | 2 | 1.0% |
| 不可读地址/异常PC | 4 | 2 | 1.0% |
| 内核栈破坏 | 2 | 2 | 1.0% |
| 用户拷贝/地址大小异常 | 1 | 1 | 0.5% |
| 引用计数检查 | 1 | 1 | 0.5% |
| 内存死锁panic | 1 | 1 | 0.5% |
| 文件权限检查 | 1 | 1 | 0.5% |
| RCU链表 | 1 | 1 | 0.5% |
| 视频解码驱动 | 1 | 1 | 0.5% |

Wi-Fi的cmp_bss与rb_erase路径合计158/193（81.9%）；再加P2P/消息处理的2例，Wi-Fi相关崩溃位置合计160/193（82.9%）。rb_erase包括cfg80211_unlink_bss、cfg80211_update_assoc_bss_entry/ch_switch_notify等删除与更新路径。

此分类表示故障出现的位置，不等同于根因定位。日志不足以判定全部故障由同一个驱动缺陷、竞态或内存破坏引起。

## 按异常表现分类

该维度与上表独立，不能把两张表相加。每个事件只归入一个异常表现；ROW_153首段标题缺失，但页表地址0x40及后续NULL异常相互印证。

| 异常表现 | 原始记录 | 去重事件 |
|---|---:|---:|
| 非法地址/页故障 | 128 | 126 |
| NULL指针异常 | 50 | 50 |
| BUG/Oops | 9 | 8 |
| WARN后显式panic | 2 | 2 |
| 不可读地址访问 | 4 | 2 |
| stack-protector栈破坏 | 2 | 2 |
| ttbr地址大小异常 | 1 | 1 |
| 内存死锁panic | 1 | 1 |
| 只读地址写入 | 1 | 1 |

## 重复日志

| 合并的ROW | Oops/致命异常时刻（启动后秒） | 机型 | 判据 |
|---|---|---|---|
| 13, 14 | 525.182730 | 43A40QSV955 | 同机型/版本/地区，故障时间、调用栈及寄存器指纹一致（OCR字符存在差异） |
| 125, 126 | 15035.263476 | 32A40QSV954 | 同机型/版本/地区，故障时间、调用栈及寄存器指纹一致（OCR字符存在差异） |
| 130, 131 | 2942.619239 | 55A60QXVT_0003821 | 同机型/版本/地区，故障时间、调用栈及寄存器指纹一致（OCR字符存在差异） |
| 168, 169, 197 | 29119.359466 | 40Q50SAVT_01556 | 同机型/版本/地区，故障时间、调用栈及寄存器指纹一致（OCR字符存在差异） |

## 按机型统计（去重事件）

| 机型 | 次数 |
|---|---:|
| 55A60QXVT_0003821 | 28 |
| 43A40QSV955 | 25 |
| 32A40QSV954 | 22 |
| 58A60LXVT_05355 | 17 |
| 55A60QXVT440 | 17 |
| 50A60QXVT298 | 13 |
| 40A40QSV838 | 11 |
| 32A40SAVT_01369 | 9 |
| 50A40QSV_02074 | 7 |
| 50A60QXVT_02357 | 7 |
| 40Q40QSV_01366 | 6 |
| 32Q40SUV_01053 | 5 |
| 55A60QXVT_02495 | 5 |
| 43V35RS231 | 5 |
| 43A60QXVT886 | 4 |
| 40A40SAVT_01368 | 4 |
| 40V35RS004 | 3 |
| 40Q50SAVT_01556 | 2 |
| 32Q50SAVT_01554 | 1 |
| 40Q50QEVS_03170 | 1 |
| 50A50QSV_02647 | 1 |

## 按软件版本统计（去重事件）

| 版本 | 次数 |
|---|---:|
| V0000.09.61B.Q0721 | 91 |
| V0000.09.61N.Q0723 | 66 |
| V0000.09.61D.Q0721 | 20 |
| V0000.09.61Q.Q0723 | 11 |
| V0000.09.61B.Q0825 | 3 |
| V0000.09.61B.Q0811 | 1 |
| V0000.09.61W.Q0723 | 1 |

## 核对示例

- cmp_bss：ROW_3，main_thread，rb_find_bss → cfg80211_bss_update → cfg80211_inform_bss_frame_data → MT7663扫描结果上报。
- rb_erase：ROW_95，dhcpcd，cfg80211_unlink_bss；ROW_21等属于关联BSS更新/信道切换路径。
- 内存死锁：ROW_96，显式“System is deadlocked on memory”。
- 栈破坏：ROW_102（__arm64_sys_futex_time32）、ROW_127（do_idle），均有stack-protector panic。
- WARN后panic：ROW_53（P2P状态/信道处理）、ROW_116（mboxRcvAllMsg），均有最终显式panic，不因WARN本身计数。

## 全部记录索引

| ROW | 视频记录头位置 | 崩溃位置类别 | 异常表现 | 归一化PC/目标 | 去重关系 |
|---|---|---|---|---|---|
| ROW_0_COL_11 | 00:00.0 | 调度/定时器/PSI | 非法地址/页故障 | tick_nohz_next_event | 计入 |
| ROW_1_COL_11 | 00:06.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_2_COL_11 | 00:13.0 | 用户拷贝/地址大小异常 | ttbr地址大小异常 | __arch_copy_to_user | 计入 |
| ROW_3_COL_11 | 00:20.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_4_COL_11 | 00:25.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_5_COL_11 | 00:30.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_6_COL_11 | 00:36.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_7_COL_11 | 00:41.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_8_COL_11 | 00:46.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_9_COL_11 | 00:52.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_10_COL_11 | 00:56.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_11_COL_11 | 01:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_12_COL_11 | 01:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_13_COL_11 | 01:12.0 | BUG/do_undefinstr路径 | BUG/Oops | do_undefinstr | 计入 |
| ROW_14_COL_11 | 01:20.0 | BUG/do_undefinstr路径 | BUG/Oops | do_undefinstr | ROW_13_COL_11 |
| ROW_15_COL_11 | 01:23.0 | 调度/定时器/PSI | 非法地址/页故障 | __hrtimer_run_queues | 计入 |
| ROW_16_COL_11 | 01:28.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_17_COL_11 | 01:34.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_18_COL_11 | 01:36.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_19_COL_11 | 01:42.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_20_COL_11 | 01:47.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_21_COL_11 | 01:50.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_22_COL_11 | 01:56.0 | 调度/定时器/PSI | NULL指针异常 | wake_affine_weight | 计入 |
| ROW_23_COL_11 | 02:04.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_24_COL_11 | 02:09.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_25_COL_11 | 02:14.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_26_COL_11 | 02:18.0 | 内存分配/释放/vmap | 非法地址/页故障 | __kmalloc_track_caller | 计入 |
| ROW_27_COL_11 | 02:24.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_28_COL_11 | 02:30.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_29_COL_11 | 02:36.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_30_COL_11 | 02:40.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_31_COL_11 | 02:46.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_32_COL_11 | 02:52.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_33_COL_11 | 02:58.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_34_COL_11 | 03:06.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_35_COL_11 | 03:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_36_COL_11 | 03:16.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_37_COL_11 | 03:24.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_38_COL_11 | 03:26.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_39_COL_11 | 03:32.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_40_COL_11 | 03:38.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_41_COL_11 | 03:44.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_42_COL_11 | 03:52.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_43_COL_11 | 04:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_44_COL_11 | 04:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_45_COL_11 | 04:16.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_46_COL_11 | 04:22.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_47_COL_11 | 04:28.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_48_COL_11 | 04:36.0 | 内存分配/释放/vmap | BUG/Oops | insert_vmap_area | 计入 |
| ROW_49_COL_11 | 04:40.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_50_COL_11 | 04:44.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_51_COL_11 | 04:52.0 | 网络收包/socket | 非法地址/页故障 | eth_type_trans | 计入 |
| ROW_52_COL_11 | 05:02.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_53_COL_11 | 05:10.0 | Wi-Fi：P2P/消息处理触发panic | WARN后显式panic | p2pRoleFsmRunEventChnlGrant | 计入 |
| ROW_54_COL_11 | 05:20.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_55_COL_11 | 05:30.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_56_COL_11 | 05:40.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_57_COL_11 | 05:48.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_58_COL_11 | 05:58.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_59_COL_11 | 06:08.0 | 存储/块设备 | BUG/Oops | read_endio | 计入 |
| ROW_60_COL_11 | 06:16.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_61_COL_11 | 06:22.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_62_COL_11 | 06:32.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_63_COL_11 | 06:38.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_64_COL_11 | 06:48.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_65_COL_11 | 06:56.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_66_COL_11 | 07:10.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_67_COL_11 | 07:18.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_68_COL_11 | 07:24.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_69_COL_11 | 07:33.5 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_70_COL_11 | 07:38.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_71_COL_11 | 07:47.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_72_COL_11 | 07:54.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_73_COL_11 | 08:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_74_COL_11 | 08:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_75_COL_11 | 08:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_76_COL_11 | 08:18.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_77_COL_11 | 08:24.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_78_COL_11 | 08:28.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_79_COL_11 | 08:32.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_80_COL_11 | 08:34.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_81_COL_11 | 08:46.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_82_COL_11 | 08:49.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_83_COL_11 | 08:54.0 | 引用计数检查 | NULL指针异常 | refcount_warn_saturate | 计入 |
| ROW_84_COL_11 | 09:00.0 | 不可读地址/异常PC | 不可读地址访问 | unresolved_PC_address | 计入 |
| ROW_85_COL_11 | 09:06.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_86_COL_11 | 09:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_87_COL_11 | 09:16.0 | 调度/定时器/PSI | 非法地址/页故障 | psi_memstall_tick | 计入 |
| ROW_88_COL_11 | 09:20.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_89_COL_11 | 09:24.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_90_COL_11 | 09:28.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_91_COL_11 | 09:34.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_92_COL_11 | 09:38.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_93_COL_11 | 09:44.0 | 调度/定时器/PSI | NULL指针异常 | check_preempt_wakeup | 计入 |
| ROW_94_COL_11 | 09:50.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_95_COL_11 | 09:56.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_96_COL_11 | 10:02.0 | 内存死锁panic | 内存死锁panic | memory_deadlock | 计入 |
| ROW_97_COL_11 | 10:08.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_98_COL_11 | 10:14.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_99_COL_11 | 10:20.0 | 内存分配/释放/vmap | BUG/Oops | kfree | 计入 |
| ROW_100_COL_11 | 10:24.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_101_COL_11 | 10:32.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_102_COL_11 | 10:36.0 | 内核栈破坏 | stack-protector栈破坏 | __arm64_sys_futex_time32 (stack protector) | 计入 |
| ROW_103_COL_11 | 10:44.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_104_COL_11 | 10:50.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_105_COL_11 | 10:56.0 | BUG/do_undefinstr路径 | BUG/Oops | do_undefinstr | 计入 |
| ROW_106_COL_11 | 11:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_107_COL_11 | 11:06.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_108_COL_11 | 11:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_109_COL_11 | 11:16.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_110_COL_11 | 11:18.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_111_COL_11 | 11:22.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_112_COL_11 | 11:30.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_113_COL_11 | 11:34.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_114_COL_11 | 11:38.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_115_COL_11 | 11:42.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_116_COL_11 | 11:48.0 | Wi-Fi：P2P/消息处理触发panic | WARN后显式panic | mboxRcvAl1Msg | 计入 |
| ROW_117_COL_11 | 11:50.0 | 内存分配/释放/vmap | 非法地址/页故障 | __kmalloc | 计入 |
| ROW_118_COL_11 | 11:56.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_119_COL_11 | 12:02.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_120_COL_11 | 12:10.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_121_COL_11 | 12:17.5 | 网络收包/socket | 非法地址/页故障 | sock_def_readable | 计入 |
| ROW_122_COL_11 | 12:22.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_123_COL_11 | 12:26.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_124_COL_11 | 12:30.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_125_COL_11 | 12:38.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_126_COL_11 | 12:44.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | ROW_125_COL_11 |
| ROW_127_COL_11 | 12:52.0 | 内核栈破坏 | stack-protector栈破坏 | do_idle (stack protector) | 计入 |
| ROW_128_COL_11 | 12:56.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_129_COL_11 | 13:00.0 | 存储/块设备 | BUG/Oops | read_endio | 计入 |
| ROW_130_COL_11 | 13:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_131_COL_11 | 13:08.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | ROW_130_COL_11 |
| ROW_132_COL_11 | 13:14.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_133_COL_11 | 13:16.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_134_COL_11 | 13:20.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_135_COL_11 | 13:24.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_136_COL_11 | 13:28.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_137_COL_11 | 13:33.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_138_COL_11 | 13:39.0 | 调度/定时器/PSI | NULL指针异常 | set_next_entity | 计入 |
| ROW_139_COL_11 | 13:44.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_140_COL_11 | 13:48.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_141_COL_11 | 13:53.5 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_142_COL_11 | 13:56.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_143_COL_11 | 14:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_144_COL_11 | 14:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_145_COL_11 | 14:12.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_146_COL_11 | 14:20.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_147_COL_11 | 14:28.0 | 文件权限检查 | NULL指针异常 | inode_permission | 计入 |
| ROW_148_COL_11 | 14:40.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_149_COL_11 | 14:44.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_150_COL_11 | 14:52.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_151_COL_11 | 15:04.0 | RCU链表 | 非法地址/页故障 | rcu_cblist_dequeue（OCR待核） | 计入 |
| ROW_152_COL_11 | 15:12.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_153_COL_11 | 15:20.0 | 调度/定时器/PSI | NULL指针异常 | set_next_entity | 计入 |
| ROW_154_COL_11 | 15:24.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_155_COL_11 | 15:29.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_156_COL_11 | 15:34.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_157_COL_11 | 15:40.0 | 内存分配/释放/vmap | 非法地址/页故障 | __kmalloc | 计入 |
| ROW_158_COL_11 | 15:52.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_159_COL_11 | 15:54.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_160_COL_11 | 16:02.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_161_COL_11 | 16:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_162_COL_11 | 16:10.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_163_COL_11 | 16:16.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_164_COL_11 | 16:19.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_165_COL_11 | 16:24.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_166_COL_11 | 16:32.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_167_COL_11 | 16:40.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_168_COL_11 | 16:48.0 | 不可读地址/异常PC | 不可读地址访问 | unresolved_PC_address | 计入 |
| ROW_169_COL_11 | 16:56.0 | 不可读地址/异常PC | 不可读地址访问 | unresolved_PC_address | ROW_168_COL_11 |
| ROW_170_COL_11 | 17:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_171_COL_11 | 17:14.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_172_COL_11 | 17:20.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_173_COL_11 | 17:26.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_174_COL_11 | 17:32.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_175_COL_11 | 17:38.0 | Wi-Fi：rb_erase / BSS删除/更新 | 只读地址写入 | rb_erase | 计入 |
| ROW_176_COL_11 | 17:46.0 | 调度/定时器/PSI | 非法地址/页故障 | __schedule | 计入 |
| ROW_177_COL_11 | 17:56.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_178_COL_11 | 18:02.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_179_COL_11 | 18:08.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_180_COL_11 | 18:16.0 | BUG/do_undefinstr路径 | BUG/Oops | do_undefinstr | 计入 |
| ROW_181_COL_11 | 18:22.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_182_COL_11 | 18:30.0 | 调度/定时器/PSI | 非法地址/页故障 | migrate_task_rq_fair | 计入 |
| ROW_183_COL_11 | 18:40.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_184_COL_11 | 18:50.0 | 存储/块设备 | NULL指针异常 | sdhci_send_command | 计入 |
| ROW_185_COL_11 | 19:00.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_186_COL_11 | 19:06.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_187_COL_11 | 19:16.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_188_COL_11 | 19:22.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_189_COL_11 | 19:30.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_190_COL_11 | 19:36.0 | Wi-Fi：cmp_bss / BSS比较 | NULL指针异常 | cmp_bss | 计入 |
| ROW_191_COL_11 | 19:40.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_192_COL_11 | 19:44.0 | BUG/do_undefinstr路径 | BUG/Oops | do_undefinstr | 计入 |
| ROW_193_COL_11 | 19:49.0 | 视频解码驱动 | 非法地址/页故障 | AVi_update_frmlist_data | 计入 |
| ROW_194_COL_11 | 19:54.0 | Wi-Fi：cmp_bss / BSS比较 | 非法地址/页故障 | cmp_bss | 计入 |
| ROW_195_COL_11 | 19:58.0 | Wi-Fi：rb_erase / BSS删除/更新 | 非法地址/页故障 | rb_erase | 计入 |
| ROW_196_COL_11 | 20:04.0 | Wi-Fi：rb_erase / BSS删除/更新 | NULL指针异常 | rb_erase | 计入 |
| ROW_197_COL_11 | 20:08.0 | 不可读地址/异常PC | 不可读地址访问 | unresolved_PC_address | ROW_168_COL_11 |
