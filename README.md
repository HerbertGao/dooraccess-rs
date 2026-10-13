# dooraccess-rs

安居宝可视门禁的 Rust 程序。它跑在家里的小路由器上，把门外的门铃接到 Home Assistant：有人按铃会通知你，也可以按设置自动开门，并把门口画面转成视频流。

它只做安居宝这一家，用来替换原来的闭源门禁程序。目标机器是 MikroTik hAP ac lite，一台 64MB 内存的 MIPS 路由器。编出来是一个大约 570KB 的单文件，不依赖别的运行库。

## 能做什么

- 听门铃，通知 Home Assistant
- 开门；有人按铃时也可以自动开门
- 把门口画面转成 FLV，给 HA 拉流
- 运行中开关「自动开门」和「自动挂断」，设置写在磁盘上，重启还在

## 先看哪份

| 你要做的事 | 文档 |
|---|---|
| 写配置、调接口 | [docs/usage.md](docs/usage.md) |
| 装到路由器上 | [docs/deploy.md](docs/deploy.md) |
| 编译，或改代码 | [docs/develop.md](docs/develop.md) |

装到那台 64MB 路由器上时，逐步命令以 `dooraccess-go/DEPLOY.md` 为准，不要自己发明步骤。安装和回滚的概览在 [package/README.md](package/README.md)。

## 配置长什么样

程序读一份 INI。下面的号码和地址都是占位，换成你自己的：

```ini
sip = 01020304@192.0.2.10:18022          ; 室内机
iface = br-door                          ; 门禁网卡

[listen]
addr = 0.0.0.0
port = 8080

[station "1"]                            ; 门外机，可以写多个
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

字段说明在 [docs/usage.md](docs/usage.md)。

## 在自己电脑上试一下

```bash
make build
make test
```

要出路由器上的程序，见 [docs/develop.md](docs/develop.md)。

## 现在到哪了

v1.0.0 起已在 hAP 上作为正式服务运行。按铃自动开门在真机上开过门，Home Assistant 能收到视频。v1.0.1 修了画面大约 16 秒后冻住的问题；这次改动的长时间画面还没在真机上复测。
