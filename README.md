# dooraccess-rs

安居宝可视门禁的 Rust 守护进程。运行于 OpenWrt，将门铃事件、开锁和门口视频接入 Home Assistant，用于替换原闭源门禁程序。仅支持安居宝。

部署平台为 MikroTik hAP ac lite（RB952Ui-5ac2nD）。安居宝室外机的上行网线提供 24V passive PoE；该机型 ether1 接受 10–28V passive PoE，可由该网线供电，无需另接电源。CPU 为 QCA9533（MIPS 24Kc，大端，无 FPU），内存 64MB。发布产物为全静态、softfloat 的单文件二进制，体积约 570KB。

## 功能

- 接收门铃事件并通知 Home Assistant
- 开锁；可在门铃触发时自动开锁
- 将门口视频转发为 FLV，供 Home Assistant 拉取
- 运行时切换自动开锁与自动挂断；状态持久化，重启后保留

## 文档

| 内容 | 位置 |
|---|---|
| 配置与 HTTP 接口 | [docs/usage.md](docs/usage.md) |
| 部署 | [docs/deploy.md](docs/deploy.md) |
| 构建与开发约束 | [docs/develop.md](docs/develop.md) |

在 hAP ac lite 上替换二进制、重启或修改启动项时，按 `dooraccess-go/DEPLOY.md` 执行。安装与回滚概览见 [package/README.md](package/README.md)。

## 配置示例

配置为 INI。下列号码与地址均为占位值。

```ini
sip = 01020304@192.0.2.10:18022          ; 室内机
iface = br-door                          ; 门禁网卡

[listen]
addr = 0.0.0.0
port = 8080

[station "1"]                            ; 室外机，可配置多项
sip = 01020000@192.0.2.20:18022

[hass]
api = http://192.0.2.30:8123
token = <HA Long-Lived Token>

[video]
forward = true
format = flv

[automation]
auto_unlock = false
auto_hangup = false
```

字段说明见 [docs/usage.md](docs/usage.md)。

## 构建

```bash
make build
make test
```

MIPS 交叉构建见 [docs/develop.md](docs/develop.md)。

## 状态

v1.0.0 起已在 hAP ac lite 上以 procd 服务运行。门铃自动开锁已完成物理验证，Home Assistant 可接收视频。v1.0.1 修复约 16 秒后画面冻结的问题；该改动的长时间视频尚未在真机复测。
