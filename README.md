# luci-app-timecontrol

> 基于 [sirpdboy/luci-app-timecontrol](https://github.com/sirpdboy/luci-app-timecontrol)
> 与 [kavass168/luci-app-timecontrol-25.12](https://github.com/kavass168/luci-app-timecontrol-25.12)
> 的适配版，修复了原版在 OpenWrt/ImmortalWrt 上两个导致**界面按钮点了没反应、
> 模式/强度设置不生效**的问题。

## 这是什么

nftables 版网络时间控制 LuCI 插件 —— 按时段 + 设备（MAC/IP）控制上网，支持
多时段、黑/白名单、流量感知计时。

本仓库在 25.12 适配版基础上修了两处上游 bug，其余功能、界面、脚本与上游一致。

## 修复的问题

### ① ACL 无效写法 → 前端所有按钮点了没反应

原版 `root/usr/share/rpcd/acl.d/luci-app-timecontrol.json` 把可执行命令全写进
`ubus.file`：

```json
"ubus": { "file": ["exec", "list", "stat", "read"], ... }
```

但 ubus 的 `file` 对象只负责 **文件读写/stat**，**执行命令**走的是独立的
`cgi-io` + `file.exec` 通道。这个写法等于把"执行"权限挂在了错误的对象上，
ACL 校验全部失败 —— 前端 `fs.exec_direct()` 调 `/usr/bin/timecontrol`、
`/etc/init.d/timecontrol` 全被拒，所以保存/重启/查流量/清日志点了都没反应。

修正版把可执行命令按 **绝对路径 + 通配** 显式列入 `file`：
`"/usr/bin/timecontrol *": ["exec"]`、`"/etc/init.d/timecontrol *": ["exec"]`
等，并补上 `cgi-io: ["exec"]`，前端按钮恢复正常。

### ② 配置节匿名 → 模式/强度设置不生效

原版 `/etc/config/timecontrol` 的全局节是**匿名**的：

```
config timecontrol
    option enabled '0'
```

后端脚本读 `@timecontrol[0]`（第一个匿名节），而前端 JS 读写的是命名节
`timecontrol.timecontrol`。两边指向的**不是同一节**，所以你在界面里改的
`control_mode`（黑/白名单）、流量阈值等**根本没写进后端读的那个节**，等于改了个
寂寞。

修正：把全局节显式命名为 `timecontrol`，前后端统一。并加了
`95-timecontrol-section` uci-defaults，对**已升级的旧配置**（还留着匿名节）
自动 `uci rename`，避免老用户升级后失效。

## 安装

### 方式一：编进固件（推荐）

把 `luci-app-timecontrol/` 整个目录放进 OpenWrt/ImmortalWrt 源码树的
`package/` 下（或 feeds 里），`make menuconfig` 勾选
`luci-app-timecontrol` 后正常编译即可。

依赖：`bc`、`nftables`、`conntrack`（内核需启用 nftables）。

### 方式二：已有固件上装 ipk

```
opkg install luci-app-timecontrol_*.ipk
```

装完重启 LuCI（或 `rm -f /tmp/luci-indexcache*`）即可在「网络 → 时间控制」看到。

## 与上游的差异

| 文件 | 改动 |
|---|---|
| `root/usr/share/rpcd/acl.d/luci-app-timecontrol.json` | 修正 ACL：可执行命令移到 `file` 显式路径 + 补 `cgi-io` |
| `root/etc/config/timecontrol` | 全局节 `config timecontrol` → `config timecontrol 'timecontrol'` |
| `root/etc/uci-defaults/95-timecontrol-section` | 新增：老配置匿名节自动改名 |

其余文件与 `kavass168/luci-app-timecontrol-25.12` 保持一致。

## 鸣谢

- [sirpdboy/luci-app-timecontrol](https://github.com/sirpdboy/luci-app-timecontrol) —— 原始项目
- [kavass168/luci-app-timecontrol-25.12](https://github.com/kavass168/luci-app-timecontrol-25.12) —— 25.12/apk 适配

## 许可

Apache-2.0（沿用上游）
