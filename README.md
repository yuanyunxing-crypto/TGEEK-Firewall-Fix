# TGEEK Firewall Fix V1.4

统一维护版本（待 OpenWrt 实机验收）。适用于 OpenWrt 24.10.x / firewall4，优先针对 x86/64、Nikki TUN、OAF 6.1.5 / 5.3.2 的已知环境。未替换 OAF 驱动。

## 解决的问题

原生 FW4 `forward` 链在普通定时禁止之前放行 `established/related`；Nikki 的提前放行规则也可能绕过普通限制。此版本使用独立 nftables 表，在 prerouting(-160) 和 forward(-10) 做定时管控。自动读取实际 FW4 规则，支持常见 LAN→WAN 的单一 MAC + 时间/星期 + 任意协议或 TCP/UDP（可带简单地址族）拒绝/丢弃规则。未识别的复杂规则报告而不扩大阻断范围。IPv4/IPv6 回程依据当前邻居表匹配，15 秒轮询及防火墙 reload 同步。保持原生 LuCI 规则，不修改时间、星期或设备地址配置。不清空全局 conntrack，不修改连接标记。

OAF 应用级识别是独立的：已建立流量的 FW4 提前放行并不能证明 V1.2 修改了 OAF 驱动，也不能证明所有 OAF 应用失效。Time Guard 只能保证受支持的定时防火墙阻断路径，不能识别任意应用/PCDN，不能修复 OAF 特征库缺失或加密识别。`oaf-status` 提供诊断，不改变 OAF 配置、不把仅限制视频的设备变成完全断网。Nikki TUN/直连通过前置阻断增强；本机代理回程、特殊策略路由、邻居尚未学习的地址等需实机验证。

## 安装

上传 `tgeek-firewall-fix-v1.4.sh` 到 `/tmp`，SSH 执行：

```sh
cd /tmp
sh tgeek-firewall-fix-v1.4.sh preview
sh tgeek-firewall-fix-v1.4.sh upgrade
```

`preview` 不加载规则；会调用 nft -c 检查。安装前需关闭软件/硬件流量分载并提交 LuCI 配置。依赖 nft、fw4、uci、ubus、jsonfilter、ip、awk、sed、cmp、flock 等；缺少依赖脚本会报错，不自动安装软件。升级默认只启用 Time Guard，不重复迁移 Docker，不关闭 RRDIPT 监控服务。兼容旧 Time Guard 安装，内部表名和备份目录沿用 v13 名称，以免新旧规则叠加。

全新设备如需全部模块（Docker 迁移会短暂停止 Docker，旧 wrtbwmon 清理会停用该统计服务），可明确执行：

```sh
sh tgeek-firewall-fix-v1.4.sh install-all
```

兼容模块有独立预检：旧 Docker 迁移仍只支持原模块明确允许的默认空桥与 host/none 容器；存在自定义网络或端口映射时可能拒绝迁移，定时模块不会因此被卸载。已经关闭 Docker 自动防火墙的设备会跳过重复迁移。

## 管理

```sh
tgeek-firewall-fix version
tgeek-firewall-fix status
tgeek-firewall-fix sync
tgeek-firewall-fix verify
tgeek-firewall-fix oaf-status
tgeek-firewall-fix compat-status
```

正常安装后程序保存在 `/usr/share/tgeek-firewall-fix/install.sh`；服务为 `/etc/init.d/tgeek-time-guard`。旧版状态路径保留：`/root/tgeek-firewall-fix-v1.3/time-guard/`。查看实时计数：

```sh
nft list table inet tgeek_time_guard_v13
```

后续直接在 LuCI 保存并应用定时规则。手动同步后可以通过 `status` 检查异常规则与运行错误。计数在规则/邻居变化重建时可能归零。

## 验收

1. 检查 preview 是否包含所有希望增强的定时规则，确认时间/星期/协议未被扩大。
2. 禁止前让目标设备开始播放，禁止后刷新新内容，核对计数持续增加且不能继续下载新视频。
3. 允许时间恢复后确认正常联网；白名单设备、Docker、Nikki、MosDNS 不受影响。
4. 在 LuCI 修改一个测试设备的时间，保存后检查自动同步；重启后再次验证。
5. 用 `oaf-status` 确认 OAF enable、驱动、接口，并分别测试只启用 OAF 应用规则和启用 FW4 全断网规则的效果。原日志 `meta l4proto ip` 是协议 0，不等同任意 IPv4；如确实需要全协议阻断请在 LuCI 改为任意协议。

## 回滚

```sh
tgeek-firewall-fix rollback
```

仅移除增强模块并恢复已备份的旧手工增强文件（如有），保留当前 LuCI 防火墙规则和 Docker 设置。`uninstall` 同样移除 Time Guard；管理入口和备份保留以便再次安装。若明确需要撤销兼容模块，使用 `compat-rollback`；Docker 模块回滚会恢复旧 firewall/dockerd 文件并覆盖随后修改，务必先另行备份当前配置。

## 测试与限制

已执行 BusyBox ash 语法检查（主程序和内嵌模块），以及模拟 FW4 规则解析测试（电视、平板、星期、IPv4/IPv6 邻居及异常协议跳过）。模拟 nft 检查不等于真实内核验证；生成环境没有 nft 命令/目标路由器，未执行实际防火墙加载、Docker 迁移、重启测试或 OAF 功能验收。正式部署前必须在目标 OpenWrt 上执行 preview 和 verify。此版本不宣称所有复杂 LuCI 规则或所有 OAF 应用均已覆盖。
