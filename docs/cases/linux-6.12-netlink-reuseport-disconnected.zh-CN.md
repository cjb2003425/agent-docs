# Linux 5.4 升级到 6.12.17 后，DHCP 成功但 UI 显示 Disconnected

| 字段 | 内容 |
|---|---|
| 类型 | 已解决故障案例 |
| 状态 | 用户已确认板端修复成功；未记录团队审核结果 |
| Jira 编号 | 未提供，暂未关联 |
| 平台 | Novatek NT72690D，ARM32 / armv7l，电视 Linux 平台 |
| 版本变化 | 内核从 5.4 升级到 6.12；现场运行版本为 6.12.17 |
| 组件 | `libkidock.so`、`rtnlk_init()`、Netlink 网络状态监听器、内核 socket 选项处理 |
| 适用范围 | 旧用户态对 Netlink 设置 `SO_REUSEPORT`，且目标内核拒绝该操作的系统 |

本案例可用于 Jira 排障知识沉淀，但没有实际 Jira 编号，也没有自动写入 Hindsight。

## 现象

升级内核后，UI 在引导或网络设置页面显示 `Disconnected`，手动 Retry 和重启仍失败。板端有线接口实际上已获得 IPv4 地址、默认路由和 DNS；修正必要的 DNS/DHCP 防火墙规则后，域名解析也能成功，但 UI 状态仍未恢复。

## 稳定日志特征

```text
network-netcfg: rtnetlnk_hooks.cpp:910:rtnlk_init():
SOCKIDX_RTNETLINK set setsockopt of reuseaddr failed. errorcode = 95
```

随后网络服务上报的状态仍包含空的 MAC/IP/DNS 字段以及 `Connecting`；UI 最终显示 `Disconnected`。

日志中的 `reuseaddr` 名称具有误导性：对现场二进制的反汇编确认，实际传给 `setsockopt()` 的选项值是 **15，即 `SO_REUSEPORT`**。

## 事件序列与证据

1. 旧用户态库在 `rtnlk_init()` 中创建 `AF_NETLINK` / `NETLINK_ROUTE` socket。
2. 初始化代码调用 `setsockopt(fd, SOL_SOCKET, 15, &one, sizeof(one))`。
3. 目标内核返回 `EOPNOTSUPP`（95），库将该错误作为初始化失败处理。
4. 检查 `/proc/net/netlink`，未发现 chassis 对路由通知组建立预期订阅。
5. 实际 Retry 抓包中，DHCP 的 Discover、Offer、Request、ACK 完整成功，并下发地址、网关和 DNS。
6. 内核网络配置成功，但状态监听器未正常建立，用户态没有正确更新并上报网络状态。
7. 应用内核兼容补丁并更新后，用户确认 UI 网络问题已修复。

现场反汇编关键点：

```asm
mov r2, #15
bl  setsockopt@plt
```

反汇编只能确认最终选项值，不能证明原始 C/C++ 代码究竟直接使用 `SO_REUSEPORT`，还是错误地对多个选项名做了按位或。

## 根因

Linux 5.4 的 `SO_REUSEPORT` 设置分支直接赋值 `sk_reuseport`。Linux 6.12.17 的实现新增了对非 INET socket 的检查：启用该选项时，若 `!sk_is_inet(sk)`，则返回 `-EOPNOTSUPP`。

Netlink 不属于 INET socket，因此旧库过去能够通过的初始化调用，在现场的新内核中被拒绝。这是用户态与内核行为变化之间的兼容问题，不能仅归因于 ARM64/ARM32 切换。

本案例确认的是 **6.12.17 的具体实现**，不应将其泛化为所有 6.12 分支；厂商补丁和 stable 回补也会影响行为。

## 已验证的修复方案：内核侧兼容

现场没有可修改的 UI/用户态库源码，因此在 `net/core/sock.c` 的 `sk_setsockopt()` 中增加 Netlink 专用兼容分支。

补丁：[netlink-reuseport-compat.patch](../../patches/netlink-reuseport-compat.patch)

该补丁对 `AF_NETLINK` 的启用请求返回成功，但**不将 `sk_reuseport` 设置为 true**；其他非 INET 协议仍保留原有拒绝逻辑。

这是一项面向旧电视用户态的产品兼容措施，不是完整恢复 Linux 5.4 的所有行为，也不表示 Netlink 真正支持 TCP/UDP 意义上的端口复用。后续若有代码依赖 `getsockopt(SO_REUSEPORT)` 回读为 1，仍需单独评估。

在内核源码根目录执行：

```sh
git apply --check /path/to/netlink-reuseport-compat.patch
git apply /path/to/netlink-reuseport-compat.patch
```

然后使用产品现有流程重新编译、更新内核镜像并重启。该修改位于内核通用网络代码中，单独替换驱动 `.ko` 不会生效。若目标厂商内核已经改动相关分支，应人工核对上下文后移植。

## 有用户态源码时的替代方案

检查以下文件中 `rtnlk_init()` 附近的 Netlink socket 初始化：

```text
cbb/chassis/framework/chassis/dock/docker/src/network/rtnetlnk_hooks.cpp
```

去掉 Netlink socket 上不必要的 `SO_REUSEPORT` 设置，或将该选项不受支持视为非致命错误并继续必要的 `bind()` 和通知组订阅。若确实需要 `SO_REUSEADDR`，应单独使用该选项。

不要将 `SO_REUSEADDR | SO_REUSEPORT` 当作选项组合：`setsockopt()` 的 `optname` 不是位掩码。

此方案在本次现场未执行，不能作为已验证结果记录。

## 验证结果与后续回归

已完成：

- 补丁通过原版 Linux 6.12.17 源码的 `git apply --check`。
- 用户应用内核兼容修复后明确反馈问题已解决。

验证边界：内核由用户自行构建、部署，未记录修复后固件的 commit、独立测试报告或完整自动回归结果。

建议纳入产品回归，以下项目尚无本次完整执行记录：

1. 冷启动与恢复出厂后的引导联网。
2. Retry、网线拔插、DHCP 续租后 UI 状态更新。
3. 启动日志不再出现上述 `errorcode = 95`。
4. 路由通知订阅正常，地址变化后网络服务上报更新。
5. DNS、HTTP/HTTPS 连通性分别验证，并符合当前 EULA 防火墙策略。
6. TCP/UDP 的 `SO_REUSEPORT` 及其他非 INET socket 的原有处理不受影响。

## 排障中未解决根因的尝试

- 补齐 DNS、放行 DHCP/DNS 后，基础网络可用，但 UI 仍显示失败。
- 调整网络配置文件权限、执行临时修复脚本以及重启，均未解决此次 UI 状态问题。
- 未抓到外网请求不能单独证明应用没有发送请求：出站防火墙可能在网卡抓包点之前丢弃数据。
- 早期持久化日志混有历史启动记录，后续通过本次启动的线程和事件关联才定位到监听器初始化错误。

因此，临时的 `fix-network.sh` 不属于本案例的正式根因修复，不应作为该补丁的必需依赖。权限、DNS 和防火墙配置应按产品策略分别核对。

## 参考源码

- [Linux 5.4：SO_REUSEPORT 分支](https://github.com/torvalds/linux/blob/v5.4/net/core/sock.c#L717)
- [Linux 6.12.17：非 INET socket 检查](https://github.com/gregkh/linux/blob/v6.12.17/net/core/sock.c#L1228)
