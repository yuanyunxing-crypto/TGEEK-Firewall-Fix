# TGEEK Firewall Fix V1.4.4 — OAF Feature-Driven App Precision Guard

OpenWrt 24.10.x / FW4 · 实机验收候选版（RC） · 2026-11-26

## 1. 更新目标

以 V1.4.3 为基线进行原位升级。保留已经验证有效的通用 Mark Guard，以及 Time Guard、Smart Offloading、Docker/FW4 和 RRDIPT Guard。移除旧版为 YouTube、腾讯视频、优酷分别维护的域名名单。本版没有任何固定应用 ID/域名规则，不新增第二套应用配置，不改变 OAF 原始特征库或驱动。

OAF 是唯一应用策略来源：已提交的 `appfilter.rule.app_list`、全局启用状态、用户模式、指定设备、白名单以及内核实际运行状态。当前活动特征文件 `/tmp/feature.cfg` 决定有效的 ID、应用名称和可解析的特征。每轮重新读取，文件 SHA256 发生变化时重建增强规则并重新开始统计，防止同一 ID 在特征库更新后被赋予不同名称。这里的同步是读取当前特征库，不是下载、更新或修改特征库。

## 2. 应用管控机制

### App Mark Guard

- 只对当前受控 MAC/邻居地址、禁止应用 ID 与当前特征库的交集生效，排除 OAF 白名单。
- 使用与 V1.4.3 相同的保守兼容条件：OAF 驱动 5.3.2、可读取规则集且未发现明显的其他连接标记写入者；只匹配完整应用 ID，绝不猜测高位标志、掩码或修改 ct mark。发现 mwan3/CONNMARK 等冲突或未知驱动时停用 Mark Guard，报告降级。检查并不能证明不存在其他内核模块写 mark。
- 外网转发及符合路由条件的早期链持续拦截已识别的新旧应用连接；TCP 在转发链尝试 RST，早期链及 UDP 丢弃。每个应用设置独立 nftables 命名计数器。
- 保留已有连接的识别标记，不执行 `conntrack -F` 或按设备清空连接。不创建整机 INPUT/OUTPUT 断流链，不默认阻断 UDP/443。
- 跳过路由器本地目的地址和 DNS/DoT 端口，限于 WAN/Nikki 外网路径；不会主动断开 LuCI、SSH、同网段 NAS。经外网或代理访问的管理业务不保证等同本地管理；OAF 原生误分类及原有设备级 Time Guard 规则仍可能影响业务。

### Proxy App Guard

使用 Nikki 本机 Mihomo 控制 API（127.0.0.1；UCI `nikki.mixin.api_port`，默认9090，`nikki.mixin.api_secret`）。不改变 Mihomo 规则或密钥，不重启代理。需要 curl、jsonfilter。

1. 源 IP 必须精确对应当前非白名单受控设备，并验证目标路由属于已知 WAN/Nikki，不处理路由器自身、局域网、DNS/DoT 或无法验证的目标。
2. 若原始五元组与 Conntrack 中唯一记录一致，并且 Mark Guard 兼容检查通过，优先采用当前特征库中的完整 OAF 应用 ID。若该 ID 属于允许应用，不再用域名推断覆盖它。
3. 若没有可靠的已识别 ID，只从当前特征库解析可安全转换的主机名特征。仅接受简单字面主机名、TCP/UDP 与数字端口/端口范围条件；同时要求该规则没有 URI、载荷、字典或其他附加限制。点分字面量按域名标签边界匹配，裸字面量仅匹配倒数第二个域名标签。不是 OAF 原生 DPI/正则引擎的完整复刻。若同一主机名匹配不同应用（包括允许应用或包含额外条件的其他应用），则跳过，防止扩大拦截。复杂正则、通配符、仅载荷特征等由 OAF 自身识别，本模块不直接翻译。
4. 匹配到禁止应用后，使用单个会话 UUID 的 DELETE API 关闭它。若 Conntrack 五元组也一致，则增加仅该五元组的短期精确阻断。轮询有时间预算，并轮转扫描，避免繁忙设备的连接列表后部持续得不到检查。

本机制仅增强特征库已有识别依据的应用。若特征库只有 `3049 Youtube:[tcp;;;youtube;;]`，不意味着 `googlevideo.com` 或 QUIC 视频连接能够被识别；本版不额外加入这些域名，也不把 TCP 特征扩大为 UDP。若 OAF 本身没有标记，且特征库无可验证匹配，则可能继续播放。应由 OAF 特征库提供覆盖，而非 TGEEK 逐个应用打补丁。代理 API 轮询不是接入前 REJECT，仍可能有短连接或少量传输。不能保证所有应用所有数据流均被拦截，也不能清除客户端已有缓存。

### 时间与安全

只支持当前已验证配置结构的网关模式（`work_mode=0`），实际 `/proc/sys/oaf/enable=1` 且已提交配置启用时才增强。未知模式、无法读取特征库或目标范围时不盲目断流。OAF 关闭或允许时间开始时移除精准表。集合元素25秒自动过期，正常轮询约每轮完成后等待5秒；25秒是失效保护租期，不是整机断流时间。

## 3. 应用统计

```sh
tgeek-firewall-fix app-stats      # 只显示有命中记录的选中应用
tgeek-firewall-fix app-stats all  # 包括未命中应用
```

字段：

| 字段 | 含义 |
|---|---|
| ID / NAME | 当前活动特征库的应用 ID、名称 |
| MARK_PACKETS | TGEEK 精准标记规则拦截包数 |
| MARK_BYTES | 上述规则拦截的包字节数 |
| PROXY_MATCHED | 代理匹配事件次数（可能重复） |
| PROXY_CLOSED | 成功关闭的代理会话次数 |
| PROXY_FAILED | API 关闭失败次数 |
| TUPLES_ADDED | 五元组阻断元素添加成功次数（不是唯一连接总数） |

统计存在 `/tmp`，不频繁写入闪存。升级、重启和活动特征库内容改变时重新开始统计；正常 OAF 关闭或规则布局替换时尽力归档旧计数。异常退出、手动删表、服务停止/重新配置导致的未归档计数可能丢失。未启用 OAF 时显示最近一次活动策略的应用名单。原生 OAF 自己的拦截不计入本统计；代理关闭次数和 Mark 包数是不同量纲，不相加。0命中不代表失败，有命中不代表全部流量被阻断。

`app-status` 中 `requested_ids` 是 UCI 有效数字 ID 数，`feature_ids` 为特征库交集，`missing_ids` 为不在库中的 ID 数，`ids` 为已启用 Mark Guard ID 数；`host_rules` 为全库解析出的候选主机特征条目数（含仅用于歧义检查的条目）。`via_mark` / `via_feature` 为本轮匹配来源；`ambiguous` 为主机归属不唯一跳过数；`unverified` 为元数据/目标路由无法验证或保留端口的跳过数；`partial=1` 表示轮询达到时间预算。`matched/closed/failed/tuples` 均为最近一轮。

## 4. 原功能保留

- Time Guard：独立 `tgeek_time_guard_v13`，按 LuCI 防火墙定时设备规则处理新旧连接，约15秒同步；它是设备级 WAN 阻断，与 OAF 应用过滤不同。
- Smart Offloading：独立 `tgeek_smart_flow_v141`，排除受控设备；全设备/未知模式或不安全环境保守关闭加速。不要同时开启 LuCI 原生流量分载。硬件分载不支持。
- RRDIPT Guard：停止/禁用 wrtbwmon，检查双栈已知链、悬空启动链接、残留进程，复发时自动清理。未知规则不删除，旧 wrtbwmon 统计停止。每12轮检查（实际间隔随负载变化）。
- Docker/FW4：原兼容修复及备份保留，`upgrade` 不重复迁移，`install-all` 才调用旧兼容模块。
- 为兼容升级和旧版清理，精准表仍名为 `tgeek_app_precision_v143`，运行状态目录仍为 `/tmp/tgeek-smart-v141`。不是版本号显示错误。

## 5. 安装及覆盖升级

建议先在 TGEEK 做单机验收，使用未受管控的 LAN 管理设备。保存/撤销 LuCI 未提交配置，确认硬件分载关闭，备份配置。升级会重载防火墙，可能产生短暂影响；不需重启路由器。

```sh
cd /tmp
tar -xzvf TGEEK-Firewall-Fix-V1.4.4.tar.gz
sha256sum -c TGEEK-Firewall-Fix-V1.4.4.sha256
sh tgeek-firewall-fix-v1.4.4.sh preview
```

预检通过后，现有 V1.4–V1.4.3：

```sh
sh tgeek-firewall-fix-v1.4.4.sh upgrade
```

全新设备改用：

```sh
sh tgeek-firewall-fix-v1.4.4.sh install-all
```

依赖 fw4、nft、uci、ubus、jsonfilter、ip、awk、sed、cmp、flock、timeout、sha256sum；代理增强还需要 curl。无需安装额外 conntrack CLI。安装前在目标机执行 nft -c 布局预检，失败则拒绝升级。原生软件分载开启时转换为 TGEEK 管理的 smart 模式；关闭时保留 off。新装默认开启精准增强，已使用 V1.4.3 并显式关闭过增强的设备保留该选择，可手动执行 `connection-on`。

## 6. 验证

```sh
tgeek-firewall-fix version
tgeek-firewall-fix verify
tgeek-firewall-fix app-verify
tgeek-firewall-fix app-status
tgeek-firewall-fix app-stats
tgeek-firewall-fix smart-status
tgeek-firewall-fix rrd-status
logread -e tgeek-precision | tail -30
cat /tmp/tgeek-v141-smart.error 2>/dev/null
```

期望版本 `TGEEK Firewall Fix V1.4.4`。`app-verify` 报告驱动/Mark 冲突、缺失 ID、API 不可用、心跳过期等问题；不要将降级视作完整保护。

实机验收：使用测试 Mac，先开启视频或其他应用，再开启 OAF 或等待计划时间；查看新旧连接与统计、是否继续下载而非仅缓存播放；检查白名单、允许应用、LuCI/SSH、DNS 和 NAS 本地访问；关闭 OAF 确认增强规则释放。针对无法确认的应用，按 `app-stats all` 查 ID/名称与命中情况。特征库不能识别的流量应反馈给 OAF 特征库维护者。

## 7. 管理与恢复

| 命令（前缀 tgeek-firewall-fix） | 用途 |
|---|---|
| app-status / cut-status | 运行状态和当前 nftables 表 |
| app-verify | 精准模块自检 |
| app-stats / app-stats all | 应用级统计 |
| connection-on / connection-off | 开关精准增强，不改 OAF |
| sync | 手动同步；守护锁占用时稍后重试 |
| flow-smart / flow-off | 管理自有软件分载 |
| rrd-fix / rrd-status / rrd-rollback | RRD 修复、检查及恢复 |
| compat-status / compat-install / compat-rollback | Docker 兼容模块 |
| rollback / uninstall | 移除增强及恢复原生分载（不是自动退回旧版） |

意外误伤时优先 `tgeek-firewall-fix connection-off`。若需立即解除，可使用另一台管理设备执行 `/etc/init.d/tgeek-firewall-guard stop`，再执行 `nft delete table inet tgeek_app_precision_v143`（表可能已不存在）。这也会暂停智能分载和 RRD 自动检查；不会移除原生 OAF 或 Time Guard。不要清空整个 nftables 或 Conntrack。升级备份 `/root/tgeek-firewall-fix-v1.3/time-guard/v144-backup-*`，历史备份保留。退回 V1.4.3 可使用原包 `upgrade`，仍需检查原先的配置及开启状态。

## 8. 开发验证和边界

本地 BusyBox ash 语法通过；模拟测试通过：任意新增 ID、特征库同步、协议/端口约束、共用域名、载荷条件不扩大、允许应用标记优先、白名单、局域网/DNS 排除、双栈、精确五元组、计数汇总、OAF 关闭及 Mark 冲突降级。RRDIPT 原回归通过（卸载后残留、双栈、悬空链接、复发、未知引用保护）。模拟 nft 和 API 不等于实际内核加载或实机网络测试。开发容器未安装 nftables，实际语法/内核能力由路由器 preview 验证。YouTube 等所有未覆盖数据流、吞吐及管理连接行为尚需实机验证。

参考：OAF 特征格式 https://doc.openappfilter.com/en/download.html；OAF https://github.com/destan19/OpenAppFilter；Mihomo https://wiki.metacubex.one/en/api/；nftables 命名计数器 https://wiki.nftables.org/wiki-nftables/index.php/Counters 。

一、补充 YouTube 特征库
修改前：
3049 Youtube:[tcp;;;youtube;;]


修改后：
3049 Youtube:[tcp;;;youtube;;,tcp;;;googlevideo.com;;,tcp;;;ytimg.com;;,tcp;;;youtu.be;;]


这种修改符合 OAF 官方特征库的域名匹配格式。
OpenAppFilter

二、一键备份并修改
在 TGEEK 执行以下完整命令，包含原始文件备份及规则校验：
```
(
    set -e

    FILE=/etc/appfilter/feature.cfg
    BACKUP=/root/oaf-feature-backup
    NEW="${FILE}.new"

    mkdir -p "$BACKUP"

    echo "===== 检查原始特征 ====="

    [ "$(grep -c '^3049 ' "$FILE")" = "1" ] || {
        echo "应用 ID 异常，停止修改"
        exit 1
    }

    grep -q '^3049 Youtube:\[tcp;;;youtube;;\][[:space:]]*$' "$FILE" || {
        echo "原始规则已变化，停止修改"
        exit 1
    }

    echo "===== 备份原始特征库 ====="

    BAK="$BACKUP/feature.cfg.$(date +%Y%m%d-%H%M%S)"
    cp -p "$FILE" "$BAK"

    echo "备份位置: $BAK"

    echo "===== 更新 YouTube 特征 ====="

    cp -p "$FILE" "$NEW"

    sed 's/^3049 Youtube:.*/3049 Youtube:[tcp;;;youtube;;,tcp;;;googlevideo.com;;,tcp;;;ytimg.com;;,tcp;;;youtu.be;;]/' \
        "$FILE" > "$NEW"

    grep -q '^3049 Youtube:.*googlevideo' "$NEW"

    [ "$(wc -l < "$FILE")" = "$(wc -l < "$NEW")" ]

    mv -f "$NEW" "$FILE"

    echo "===== 修改结果 ====="
    grep '^3049 ' "$FILE"

    echo "特征库修改成功"
)
```
