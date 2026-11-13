# kwrt —— Kwrt 软路由 AI 管理 CLI

一个装在 OpenWrt 系路由器上的零依赖命令行工具，把 `ubus` / `uci` / `iwinfo` / `logread`
这些底层接口包装成**结构化 JSON**，让 AI（或脚本）不用记参数就能管理路由器。
人类使用时加 `-t` 看表格。

- 设备：OpenWrt 系软路由（具体型号由目标环境提供）
- 路由器地址：`<ROUTER_HOST>`
- 安装位置：`/etc/kwrt/kwrt`，通过软链接暴露为 `/usr/sbin/kwrt`
  （放在 `/etc` 下是为了 `sysupgrade` 刷机时能随配置保留）

## 一、连接方式

如果目标环境已经配置 SSH 别名，可直接：

```bash
ssh router kwrt info          # JSON
ssh router kwrt -t info       # 人类可读
ssh router                    # 进交互 shell
```

本机 `~/.ssh/config` 里的条目：

```
Host router kwrt
    HostName <ROUTER_HOST>
    User root
    IdentityFile <ROUTER_KEY_PATH>
```

> 你之后修改 root 密码不会影响这条免密通道（走的是密钥认证）。

## 二、命令速查

### 只读（随时可用）

| 命令 | 说明 |
|---|---|
| `kwrt info` | 固件、型号、内核、运行时长、负载、内存、磁盘、温度、LAN/WAN、终端数 |
| `kwrt status` | 运行状态速览 |
| `kwrt net` | 所有网络接口：协议、存活、设备、IPv4、收发字节 |
| `kwrt route` | 路由表 |
| `kwrt clients` | 在线终端（DHCP 租约 + 无线关联合并，含信号强度、连接方式） |
| `kwrt dhcp` | 仅 DHCP 租约与剩余时间 |
| `kwrt wifi` | 射频：SSID、信道、频段、加密方式、终端数 |
| `kwrt services` | 全部服务及运行/开机自启状态 |
| `kwrt logs [行数] [过滤词]` | 系统日志，例：`kwrt logs 50 openclash` |
| `kwrt firewall` | 防火墙区域与规则/转发数量 |
| `kwrt dns <域名>` | 解析测试 |
| `kwrt ping <主机> [次数]` | 连通性与 RTT |
| `kwrt doctor` | 自检：软件源、DNS、WAN、磁盘、内存、SSH、时间、温度、依赖 |
| `kwrt uci show <配置>` | 导出配置为 JSON，例：`kwrt uci show network` |
| `kwrt uci get <键>` | 读单个配置项 |
| `kwrt pkg update` | 更新软件源索引 |
| `kwrt pkg search <关键词>` | 搜索软件包 |
| `kwrt pkg installed` | 已安装列表 |
| `kwrt pkg info <包名>` | 包详情 |
| `kwrt ubus <对象> <方法> [参数]` | 直通 ubus，例：`kwrt ubus system board` |

### 写操作（必须 `-y`）

```bash
kwrt uci set network.lan.ipaddr=<NEW_LAN_IP> -y   # 也支持 kwrt uci set <键> <值>
kwrt uci commit network -y                      # 提交生效
kwrt uci revert network -y                      # 丢弃未提交改动
kwrt uci delete <键> -y
kwrt svc restart dnsmasq -y
kwrt svc enable|disable|start|stop <服务> -y
kwrt wifi reload -y
kwrt pkg install|remove <包名> -y
kwrt reboot -y
```

## 三、给 AI 用的方式

AI 侧只需要一条命令拿全量 JSON，再自己筛选：

```bash
# 谁在线、信号怎么样
ssh router kwrt clients | jq -r '.clients[] | "\(.ip) \(.hostname) \(.medium) \(.signal_dbm // "-")"'

# WAN 有没有掉
ssh router kwrt status | jq '.wan_up'

# 查某个服务为什么没起来
ssh router kwrt services | jq '.services[] | select(.running==false)'
ssh router kwrt logs 100 dnsmasq
```

改动类操作建议流程：`uci get` 看现状 → `uci set` → `uci commit` → `svc restart` → `logs` 验证。

## 四、安全设计

- 只读命令无门槛；**所有写操作**（改配置、重启服务、装包、重启设备）在未加 `-y` 时直接拒绝，
  返回 `{"error":"need_confirmation",...}` 且退出码为 `3`，不会误触发。
- `kwrt reboot` 有 2 秒延迟缓冲，期间可 Ctrl-C 中断。
- 输出默认 JSON，错误以 JSON 写 stderr，stdout 保持干净便于管道处理。

## 五、本次顺带修复的问题

**opkg 软件源全部 404**：部分固件的 `/etc/opkg/distfeeds.conf` 可能指向旧版本路径，
而上游只有 `releases/24.10.0/`。已改为官方 24.10.0 路径，现在 `opkg update` 正常，
可用包约 8990 个。原文件备份在 `/etc/opkg/distfeeds.conf.bak.20261114`。

如果某个厂商专有源失效，应先确认新地址，不要盲目替换为不明来源的软件源。

## 六、注意事项

1. **不要从官方源装 kmod 内核模块**。Kwrt 用的是自编译内核，官方 kmod 版本不匹配，
   装错可能导致网络异常。纯用户态软件包（jq、curl、tcpdump 等）没问题。
2. 未实测的命令：`kwrt reboot -y`、`kwrt wifi reload -y`（会断网/断无线，未在你的网络里验证），
   其余命令均已实测通过。
3. 刷机（sysupgrade）时勾选"保留配置"，`/etc/kwrt/kwrt` 会保留；
   但软链接 `/usr/sbin/kwrt` 可能丢失，刷完补一条：
   `ln -sf /etc/kwrt/kwrt /usr/sbin/kwrt`
4. 升级本 CLI：在本机 `scp router-cli/kwrt router:/etc/kwrt/kwrt` 即可。

## 七、卸载

```bash
ssh router 'rm -f /usr/sbin/kwrt; rm -rf /etc/kwrt'
```
