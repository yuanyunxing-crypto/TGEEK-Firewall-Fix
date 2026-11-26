# TGEEK Firewall Fix V1.4.8 — Stable Guard

2026-11-26 · OpenWrt 24.10.x / FW4 · **实机验收候选版**（独立于 V1.4.7.2 发布，旧包不变）

## 1. 修复范围

TIJIWrt 实机：R1 特征库 565 ID，11:03 文件变成 0 字节，导致 `feature-unavailable`、精准表与 QUIC 表同时消失；恢复文件后两个表自动重建。TGEEK 已验证 V1.4.7.2 全新时间模块安装 `legacy=0`、QUIC 保存、所有应用 565 与指定应用 550 切换；TIJIWrt 已验证迁移 `legacy=2`、560 个指定应用和 QUIC 双栈实际命中。以上是旧版实机结果，不是新版实机验收结果。

- Feature Guard：定期校验活动文件 `/etc/appfilter/feature.cfg`（原生 `/tmp/feature.cfg` 软链接）。校验 ID 唯一、结构边界、大小≤4 MiB、1–4096 ID 及 SHA256。不是 DPI 规则语义验证。
- 可信备份：第一次自动备份要求覆盖已选 ID；两次一致快照间隔至少5秒。新文件保留所有旧 ID 时，允许同样的两次一致检测后自动更新备份；减少/替换 ID 需管理员主动批准。校验不通过不覆盖旧备份。内容寻址文件及 current/previous 指针原子提交，只在内容变化时写入持久盘。历史快照保留，不自动清理。
- 空文件/缺失文件：连续两次检查、间隔至少5秒，且有 SHA256 一致的备份时，原目录临时文件+rename 恢复；恢复权限 644。**不会覆盖非空文件**（包括格式损坏或可能正处于上传的文件），不强制重启 OAF，无法保证原生 DPI 已重新加载。非空异常需管理员修复；日志报告原因。
- QUIC 解耦：特征库异常、待验证或精准 nft 事务失败不会自动删除 QUIC 表。仍需 OAF 开关、运行时限制时段、Connection Guard、过滤模式、有效设备范围、非空已配置 ID 列表、LAN/出口验证、无冲突分载；仅对目标设备外网 UDP443 丢包，25秒租期。此为协议级限制，不等同于单个应用识别。
- 精准过滤：仅用与可信备份一致的活动快照；不拿旧特征库去匹配已变化的活动目录。不扩大设备范围、不整机断网、不主动清空 conntrack。
- Mark：保留 V1.4.7.2 的精确 ID 模式及冲突检查，新增 `mark-audit`。观察到 0x80000FA1（候选 ID 4001）及 0x40000000（仅标志候选），未取得与私有引擎5.3.2一致的定义，**仅诊断，不启用复合标记拦截，也不修改连接标记**。
- Nikki：可用 Lua JSON 解码时一次预筛选受控 IP（含 IPv4-mapped IPv6），失败回退旧遍历；保留3秒轮转扫描预算，增加 `scan_mode/scan_done/scan_total`，`partial=1` 表示本轮未完成。不保证 API 会匹配所有流量。
- 周期全库同步及手动 `luci-sync`：仅在特征文件与备份一致时执行。原 LuCI 保存逻辑保留；更新特征库期间请勿在 LuCI 保存全应用名单，等 `feature-verify` 成功后再操作。
- 继承 LuCI 中文、所有/指定应用模式、V2.1.1 独立星期、Smart Offloading、RRDIPT、Docker/FW4、原 Time Guard。旧内部表名、路径保留，避免重复创建。

## 2. 升级 TIJIWrt 或 TGEEK

建议先在 TIJIWrt 本地管理时升级，通过后再安装 TGEEK。不要删除原 OAF、旧时间补丁或已有 V1.4.7.2 备份。完整安装沿用 V1.4.7.2 流程，可能重载防火墙，网络可能短暂受影响。先留一个 SSH 会话、确认本地访问可用。包不包含 R1 私有特征库，不下载第三方特征库，不替换 OAF 内核。

将 tar.gz 上传到 `/tmp` 后：

```sh
cd /tmp
tar -xzf TGEEK-Firewall-Fix-V1.4.8.tar.gz
sha256sum -c TGEEK-Firewall-Fix-V1.4.8.sha256
sh tgeek-firewall-fix-v1.4.8.sh preview
```

以上均成功后：

```sh
sh tgeek-firewall-fix-v1.4.8.sh upgrade
```

全新设备（原生兼容的 kmod-oaf、appfilter、luci-app-oaf 已安装）用 `install-all` 代替 `upgrade`。无需单独预装 Time Sync；原 V1.4.7.2 的未知旧时间组件拒绝覆盖保护保留。如果没有 OAF，仅安装基础增强，后续安装兼容 OAF 后执行 `luci-install`。需要 fw4/nft/uci/ubus/jsonfilter/ip/awk/sed/cmp/flock/timeout/sha256sum/readlink，Nikki 使用 curl，时间同步需要原生 Lua LuCI 和 nixio。硬件分载应关闭；保存或撤销未提交的 UCI 配置。

## 3. 验证及备份

等待后台采样，执行：

```sh
tgeek-firewall-fix version
tgeek-firewall-fix feature-status
tgeek-firewall-fix feature-verify
tgeek-firewall-fix health
tgeek-firewall-fix mark-audit
```

若当前确认为正确 R1（TIJIWrt 实机 SHA256 `f0544f69b55871d9d9d395b1eef236d3646edf561d38614bce34f65a4fe05500`），且 `feature-verify` 提示尚未建立备份，可手动批准：

```sh
tgeek-firewall-fix feature-backup
tgeek-firewall-fix feature-verify
```

自动备份路径 `/root/tgeek-firewall-fix-v1.3/time-guard/feature-v148/`，`current` 文件内容为已验证的 SHA256，对应 `<SHA256>.cfg`。已有独立备份目录不被覆盖。备份校验成功不代表应用流量一定受阻。

## 4. 日常命令

| 命令（前缀 tgeek-firewall-fix） | 作用 |
|---|---|
| `health` | 简洁查看特征、统计、LuCI、时间、服务；不是完整 nft 自检 |
| `feature-status` | 特征保护状态、现有与备份 SHA、ID 数、最近恢复记录 |
| `feature-verify` | 当前活动文件与可信备份是否一致 |
| `feature-backup` | 手动批准当前有效文件并备份（有意更换特征库时使用） |
| `feature-restore` | 仅空/缺失文件时，从已校验备份恢复，不覆盖非空文件 |
| `mark-audit` | 受控设备非零 conntrack 标记聚合，不暴露目的地址，不修改规则 |
| `app-stats` / `app-stats all` | 增强规则命中统计；后者列全部选中 ID |
| `quic-stats` | IPv4/IPv6 UDP443 命中，表重建计数重新开始 |
| `luci-verify` / `time-verify` | LuCI RPC、时间规则回读 |
| `verify` | 完整检查，可能输出大量规则 |
| `sync` | 触发已有模块同步，可能关闭匹配的受控代理连接 |

故障日志：

```sh
logread -e tgeek-feature-v148 | tail -30
cat /tmp/tgeek-smart-v141/feature.status
cat /tmp/tgeek-v141-smart.error 2>/dev/null
```

`feature-status` 的历史状态可能晚于当前文件变化；以 `feature-verify` 的即时验证为准。`pending` / `blocked` 不代表 OAF 已停止，精准增强在等待可信特征快照；QUIC 仍按自身条件运行。`mark-audit` 的 `high-bit-candidate` 不是已验证的应用归属。

## 5. 特征库安全更新

优先用原生 OAF 官方支持的特征库导入方式。不要把终端输出粘贴回 SSH，也不要使用 `cat > /etc/appfilter/feature.cfg` 或直接覆盖 `/tmp/feature.cfg`。两者指向同一文件，直接重定向会先截断。若手动更新已知兼容的单个 cfg，先上传临时文件、核对 SHA、确认结构后，在 `/etc/appfilter` 内用临时文件+rename 原子切换；合法 ID 缩减需明确 `feature-backup` 批准。未知 11:03 清空程序仍未定位。保护仅减少后果，不替代查明写入者。

自动恢复只修复文件并让增强规则重新构建，不强制重启 OAF；如实测原生 DPI 仍无法识别，在本地管理且确认没有保存/时间同步任务时执行原生 `/etc/init.d/appfilter restart`，然后 `time-verify`、`luci-verify`，并实际测试应用。外部程序不共享本项目锁，原子切换不能完全排除并行写入竞争。不要在生产路由器故意清空特征库测试，建议使用测试虚拟机。

## 6. 退回 V1.4.7.2（不是卸载）

保留旧版安装器，按旧版正常覆盖升级流程重新执行：

```sh
cd /tmp
sh tgeek-firewall-fix-v1.4.7.2.sh preview
sh tgeek-firewall-fix-v1.4.7.2.sh upgrade
tgeek-firewall-fix version
tgeek-firewall-fix time-verify
tgeek-firewall-fix luci-verify
```

这是兼容性回退路径，尚未实机测试。不要用 `rollback` / `uninstall` 代替，它们会移除整个增强层并受到独立星期规则回滚保护。V1.4.8 本次安装保存基础组件的备份 `.../v148-backup-*`；Feature Guard 的备份保留，V1.4.7.2 不会自动使用它。完整升级沿用旧版多阶段事务，非整体原子回滚，若中途失败应保留日志和安装器给出的备份路径。

## 7. 测试边界

详见 TEST.txt。仅完成本地语法和隔离逻辑测试，尚未在两台路由器运行 V1.4.8。真正的 nft 内核装载、procd、真实 Lua JSON 模块、原生 DPI/代理拦截、切换时段及升级/回退需实机确认。未解决的高位标记语义不能凭观察值直接推断；本版不修改精确匹配规则。实际安装完成后先用 health 检查，再用受控设备测试淘宝、腾讯视频、YouTube 等新内容，并确认白名单/非受控设备正常。所有应用模式不是全网断网。
