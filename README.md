# kwrt

面向 AI Agent 的 OpenWrt / Kwrt 软路由管理 CLI。

`kwrt` 是一个单文件、零第三方依赖的 POSIX Shell 工具，把 OpenWrt 常用的
`ubus`、`uci`、`iwinfo`、`logread` 等底层接口包装成结构化 JSON，让 Agent 和脚本
可以稳定地读取设备状态、检查网络，并在明确确认后执行配置变更。

## 特点

- **Agent-friendly**：默认输出 JSON，方便模型和脚本解析。
- **零依赖**：只使用 OpenWrt 常见的系统工具，不要求额外安装运行时。
- **安全确认**：所有写操作都必须显式带 `-y`，否则返回退出码 `3`。
- **覆盖常用运维场景**：系统信息、网络、路由、客户端、Wi-Fi、服务、日志、防火墙、DNS 和软件包。
- **可移植**：目标设备地址、账号和 SSH 密钥由部署环境提供，不写入工具源码。

## 快速开始

### 1. 上传工具

```bash
scp router-cli/kwrt <ROUTER_USER>@<ROUTER_HOST>:/tmp/kwrt
```

### 2. 安装到路由器

```bash
ssh <ROUTER_USER>@<ROUTER_HOST> \
  'mkdir -p /etc/kwrt && mv /tmp/kwrt /etc/kwrt/kwrt \
   && chmod 755 /etc/kwrt/kwrt \
   && ln -sf /etc/kwrt/kwrt /usr/sbin/kwrt'
```

### 3. 验证

```bash
ssh <ROUTER_USER>@<ROUTER_HOST> \
  'sh -n /etc/kwrt/kwrt && kwrt info'
```

临时试用时，也可以只运行上传到 `/tmp` 的文件：

```bash
ssh <ROUTER_USER>@<ROUTER_HOST> 'sh /tmp/kwrt info'
```

## 常用命令

```bash
kwrt info                 # 系统、内存、磁盘、温度和地址
kwrt status               # 运行状态速览
kwrt net                  # 网络接口与流量统计
kwrt clients              # 在线终端与 DHCP 租约
kwrt wifi                 # 无线状态
kwrt services             # 服务运行状态
kwrt logs 100 dnsmasq     # 查看并筛选系统日志
kwrt doctor               # 综合自检
```

默认输出适合 Agent 和脚本处理；人类查看时可以加 `-t`：

```bash
kwrt -t info
kwrt -t status
```

## 写操作安全规则

修改配置、重启服务、安装软件包、刷新 Wi-Fi 和重启设备等操作必须显式确认：

```bash
kwrt uci set <KEY> <VALUE> -y
kwrt uci commit <CONFIG> -y
kwrt svc restart <SERVICE> -y
kwrt reboot -y
```

没有 `-y` 时，工具不会执行写操作，而是向 stderr 输出 `need_confirmation`，并返回退出码 `3`。

建议遵循这个顺序：

```text
读取现状 → 修改 → 提交 → 重启相关服务 → 再次读取验证
```

## 项目结构

```text
router-cli/
├── kwrt          # CLI 主程序
├── AGENTS.md     # 给 AI Agent 的安装、使用与安全约定
└── README.md     # CLI 使用说明
```

`router-cli/AGENTS.md` 是面向 Agent 的可移植安装指南，包含依赖、命令契约、确认规则、故障排查和卸载方式。

## 安全边界

- 不在源码中写入设备地址、账号、密码、API Key 或 SSH 私钥。
- 不监听端口，不运行常驻守护进程。
- 修改网络或防火墙前，必须确认存在备用管理通道或现场接触条件。
- 不要从不匹配的源安装 `kmod-*` 内核模块；自编译固件应使用与内核严格匹配的模块。

## 适用环境

- OpenWrt 21.02 及衍生发行版
- Kwrt、ImmortalWrt 等 OpenWrt 系固件
- POSIX `sh` / BusyBox 环境

具体设备地址、登录账号和密钥由部署环境提供。项目不包含任何真实设备凭据。
