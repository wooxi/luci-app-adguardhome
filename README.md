# luci-app-adguardhome

AdGuardHome 的 OpenWrt LuCI 管理界面。

**本仓库是 [rufengsuixing/luci-app-adguardhome](https://github.com/rufengsuixing/luci-app-adguardhome) 的 fork**，针对 **OpenWrt 25.12+ / apk 包管理器** 做了适配。

原版最后更新于 2022 年（opkg 时代），在 OpenWrt 25.12 上存在包管理器不兼容、架构映射缺失等问题。本 fork 保持原版的功能与界面，只做必要的兼容性修改。

---

## 与上游的差异

| 项目 | 上游（2022） | 本 fork |
|---|---|---|
| 包管理器 | `opkg` | `apk` |
| 架构检测 | `opkg info kernel` | `uname -m` |
| x86_64 支持 | ❌ 缺失（走 fallback 报错） | ✅ 已补 |
| 配置路径默认值 | `/etc/AdGuardHome.yaml` | `/etc/adguardhome/adguardhome.yaml` |
| 工作目录默认值 | `/usr/bin/AdGuardHome` | `/etc/adguardhome` |
| 二进制路径默认值 | `/usr/bin/AdGuardHome/AdGuardHome` | 同上游（目录结构） |
| yaml 编辑器宽度 | 半宽显示 | 100% 宽 + 70vh 高 |
| 中文翻译 | 未编译 | 已编译 `.lmo` |

功能、界面、重定向逻辑均与上游一致。

---

## 安装

### 前置条件

- OpenWrt 25.12 或更新版本（apk）
- 约 40 MB 磁盘空间
- 已安装 `lua` 与 `luci-compat`（Lua 界面兼容层）

```sh
apk add lua luci-compat
```

### 安装本插件

```sh
# 1. 下载（或自行构建）
wget -O /tmp/luci-app-adguardhome.apk \
  https://github.com/wooxi/luci-app-adguardhome/releases/latest/download/luci-app-adguardhome.apk

# 2. 安装（未签名，需 allow-untrusted）
apk add --allow-untrusted /tmp/luci-app-adguardhome.apk

# 3. 安装 AdGuardHome 本体（如果尚未安装）
apk add adguardhome

# 4. 清缓存
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache
/etc/init.d/uhttpd restart
```

### 手动部署（无 apk 包时）

仓库的 `root/` 和 `luasrc/` 可直接覆盖到系统：

```sh
# luasrc → /usr/lib/lua/luci/
# root/   → /
```

---

## 目录与路径约定

| 内容 | 默认路径 |
|---|---|
| UCI 配置 | `/etc/config/AdGuardHome` |
| AdGuardHome 配置 | `/etc/adguardhome/adguardhome.yaml` |
| 工作目录 | `/etc/adguardhome` |
| 二进制 | `/usr/bin/AdGuardHome/AdGuardHome` |
| 运行日志 | `/var/log/AdGuardHome.log` |
| Web 端口 | `3000` |
| DNS 端口 | `53`（由重定向模式决定） |

**注意**：二进制路径必须是 `/usr/bin/AdGuardHome/AdGuardHome`（目录 + 文件），因为 init 脚本会对 `${binpath%/*}` 执行 `mkdir -p`。如果把 `binpath` 指向单文件，会报 `mkdir: Not a directory`。

---

## 重定向模式

界面「基础设置 → 重定向」提供四种模式，由插件自动配置，不需要手工改 dnsmasq 或防火墙：

| 模式 | 行为 | 适用场景 |
|---|---|---|
| `none` | 不做任何重定向 | 手动管理端口时 |
| `redirect` | iptables 把 53 端口重定向到 ADG | 保留 dnsmasq 在 53，ADG 用其他端口 |
| `dnsmasq-upstream` | ADG 监听其他端口，dnsmasq 保留 53 并转发给它 | 想保留 dnsmasq 的本地域名解析 |
| `exchange` | ADG 抢 53，dnsmasq 自动换到 1745 | ADG 完全接管 DNS |

切换模式后重启服务即生效：

```sh
uci set AdGuardHome.AdGuardHome.redirect='exchange'
uci commit AdGuardHome
/etc/init.d/AdGuardHome restart
```

`exchange` 模式下的端口分配：

```text
AdGuardHome  → :53
dnsmasq      → :1745（只做 DHCP）
```

---

## 功能

- **核心管理**：LuCI 内下载/更新 AdGuardHome 核心，支持自定义下载链接
- **upx 压缩**：压缩核心节省空间（用内存换磁盘）
- **重定向**：四种模式，见上表
- **路径自定义**：可改二进制、配置、工作目录、日志路径
- **gfwlist**：删除/添加/指定上游 DNS
- **密码修改**：网页内改 ADG 登录密码
- **日志查看**：倒序/正序、备份、每 3 秒刷新、本地时区转换
- **yaml 编辑器**：内置 CodeMirror，支持语法高亮和折叠
- **模板配置**：无配置文件时可一键套用模板
- **升级保留**：系统升级时保留指定文件
- **开机等待**：网络就绪后再启动（3 分钟超时）
- **关机备份**：关机时备份工作目录
- **计划任务**：自动更新核心、截断日志、更新 gfwlist

---

## 已知问题

- **db 数据库不支持 mmap 文件系统**（如 jffs2）。本插件检测到 jffs2 会自动把 db 软链到 `/tmp`，重启后丢失查询历史。x86_64 的 ext4 无此问题。
- **AdGuardHome 不支持 ipset**，因此无法完全替代 dnsmasq（只能作为上游或独占 53）。
- **大量 127.0.0.1 查询**：通常是 ddns 插件引起，可注释 `/etc/hotplug.d/iface/95-ddns`。
- **Lua 界面依赖 luci-compat**：OpenWrt 25.12 默认不装 Lua，必须手动安装 `lua` 和 `luci-compat`。

---

## 与上游的同步

本 fork 只做兼容性修改，功能逻辑不改。上游如有更新，可以这样合并：

```sh
git remote add upstream https://github.com/rufengsuixing/luci-app-adguardhome.git
git fetch upstream
git merge upstream/master
```

冲突通常集中在 `update_core.sh`（包管理器相关）和 `config/AdGuardHome`（路径默认值）。

---

## 许可

与原版一致：Apache License 2.0

## 致谢

- 原始项目：[rufengsuixing/luci-app-adguardhome](https://github.com/rufengsuixing/luci-app-adguardhome)
- 核心：[AdGuardTeam/AdGuardHome](https://github.com/AdguardTeam/AdGuardHome)
