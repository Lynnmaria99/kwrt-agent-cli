# kwrt · Agent 安装与使用指引

> 面向 AI Agent 的可移植说明。本文**不含任何具体设备地址、账号或凭据**，部署时由环境提供。
> 若你需要在某台路由器上安装或使用这个工具，按本文操作即可。

## 这是什么

`kwrt` 是装在 **OpenWrt 系软路由**上的单文件命令行工具（POSIX sh，无第三方依赖）。
它把 `ubus`、`uci`、`iwinfo`、`logread` 包装成结构化 JSON，让 Agent 不必记底层参数
就能查询和修改路由器。

## 前置条件

| 条件 | 说明 |
|---|---|
| 系统 | OpenWrt 21.02+ 及衍生版（Kwrt / ImmortalWrt 等）；已在 Kwrt 24.10、kernel 6.6.x 上验证 |
| 依赖命令 | `ubus` `uci` `iwinfo` `jsonfilter` `nft` `logread` — OpenWrt 默认自带，无需安装 |
| 访问方式 | 能 SSH 登录路由器；凭据由部署者提供 |
| 写权限 | 路由器 `/etc` 可写 |

Shell 说明：`/bin/sh` 通常是 busybox ash；部分精简固件删掉了 `hostname`、`free` 等 applet，
本工具全部通过 `/proc` 与 `ubus` 取值，不依赖这些命令。

## 安装

三步，把 `kwrt` 文件放到路由器上：

```bash
# 1) 上传
scp kwrt <ROUTER_USER>@<ROUTER_HOST>:/tmp/kwrt

# 2) 安装（放在 /etc 下，sysupgrade 勾选"保留配置"时可留存）
ssh <ROUTER_USER>@<ROUTER_HOST> \
  'mkdir -p /etc/kwrt && mv /tmp/kwrt /etc/kwrt/kwrt \
   && chmod 755 /etc/kwrt/kwrt && ln -sf /etc/kwrt/kwrt /usr/sbin/kwrt'

# 3) 验证
ssh <ROUTER_USER>@<ROUTER_HOST> 'sh -n /etc/kwrt/kwrt && kwrt info'
```

预期结果：第 3 步输出一段合法 JSON，包含
`hostname` `model` `kernel` `uptime_seconds` `load` `memory_mb` `disk_mb`
`temperature_c` `lan_ip` `wan_ip` `wifi_clients` `dhcp_leases`。

如果只想临时试用、不想改动系统，可以直接运行上传后的
`sh /tmp/kwrt info`（此时入口不是 `kwrt`，子命令相同）。

## 输出契约（Agent 必须知道的）

| 约定 | 说明 |
|---|---|
| 成功输出 | stdout 是**单个 JSON 对象**，可直接 `\| jq` 解析 |
| 人类可读 | 加 `-t` 输出表格/对齐文本，仅用于展示，不要用于解析 |
| 错误 | stderr 输出 `{"error":"...","detail":"..."}`，退出码非 0 |
| 未确认的写操作 | 退出码 **`3`**，错误体含 `need_confirmation` |
| 无守护进程 | 不监听端口、不常驻，每条命令执行完即退出 |

## 命令

只读（随时可用）：

```
info      status    net       route     clients   dhcp      wifi
services  logs      firewall  dns       ping      doctor
uci show <配置>      uci get <键>
pkg update | search <词> | installed | info <包>
ubus <对象> <方法> [JSON参数]        # 直通任意 ubus 接口
```

写操作（**必须带 `-y`**，否则被拒绝）：

```
uci set <键> <值> | uci set <键>=<值> | uci delete <键>
uci commit <配置> | uci revert <配置>
svc <start|stop|restart|enable|disable> <服务>
wifi reload          pkg install|remove <包>          reboot
```

`doctor` 会自检软件源、DNS、WAN、磁盘、内存、SSH、时间、温度、依赖命令，
是接手一台陌生路由时的推荐起点。

## Agent 使用建议

1. **先看再动**：`kwrt doctor` → `kwrt info` → `kwrt status`，再决定改什么。
2. **改配置固定四步**，每步都要复核：
   ```
   kwrt uci get <键>
   kwrt uci set <键> <新值> -y
   kwrt uci commit <配置> -y
   kwrt svc restart <服务> -y
   ```
   改完用对应的只读命令验证（如改网络后 `kwrt net`、改无线后 `kwrt wifi`）。
3. **用 jq 取字段，不要 grep 文本输出**，例如
   `kwrt clients | jq -r '.clients[] | "\(.ip)\t\(.hostname)"'`。
4. **一次取全量再筛选**，避免高频轮询路由器（设备资源有限）。
5. **改动前先备份**：`uci export <配置> > /tmp/<配置>.bak`，
   出错用 `kwrt uci revert <配置> -y` 或还原备份。
6. `wifi reload` 和 `reboot` 会短暂断网；执行前确认自己有其它可用通道或现场接触。

## 硬约束

- **禁止从软件源安装 `kmod-*` 内核模块**。这类固件多为自编译内核，
  与官方 kmod 版本不匹配，装上可能导致网络不可用。纯用户态包（jq、tcpdump 等）安全。
- 修改 `network` / `firewall` 前，必须确认存在物理接触或备用管理通道，
  否则一次配置错误就可能失去远程访问。
- 不要在路由器上手改 `/etc/kwrt/kwrt`；应改本地源码再上传，避免与源码脱节。

## 故障排查

| 现象 | 处理 |
|---|---|
| `kwrt: not found` | 入口软链丢失：`ln -sf /etc/kwrt/kwrt /usr/sbin/kwrt` |
| `opkg update` 全部 404 | 发行版路径写错（例如 `releases/24.10/` 应为 `releases/24.10.0/`），检查 `/etc/opkg/distfeeds.conf` |
| SSH 连不上 | 确认 `dropbear` 在运行；若配置改坏需现场恢复 |
| `doctor` 报磁盘红 | overlay 剩余不足 10%，清理 `/overlay` 或卸载大包 |
| 无线终端读数为 0 | `iwinfo` 无权限或无射频：确认以 root 执行 |

## 升级与卸载

```bash
# 升级：覆盖主文件即可，软链无需重建
scp kwrt <ROUTER_USER>@<ROUTER_HOST>:/etc/kwrt/kwrt

# 卸载
ssh <ROUTER_USER>@<ROUTER_HOST> 'rm -f /usr/sbin/kwrt; rm -rf /etc/kwrt'
```

刷机（`sysupgrade`）后 `/etc/kwrt/kwrt` 通常保留，但 `/usr/sbin/kwrt` 软链可能丢失，
刷完补一条 `ln -sf /etc/kwrt/kwrt /usr/sbin/kwrt` 即可。

## 设计约定（修改本工具时遵守）

- 只读命令不加确认；任何写操作都调用 `require_yes`，未确认时退出码 3。
- stdout 只放 JSON，错误只走 stderr，保证 `| jq` 管道不被污染。
- 数值字段用 `jnum`（支持负号与小数点，非法值输出 `null`），
  字符串字段用 `jstr`（空值输出 `null`），避免拼出非法 JSON。
- 新增命令必须同时提供 `-t` 文本模式与 JSON 模式，并在文档中同步。
