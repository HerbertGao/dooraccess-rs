# 怎么用

程序读一份 INI，再在 8080 端口提供一组 HTTP 接口。Home Assistant 和配套的 HACS 集成走这些接口。

示例里的号码和 IP 都是占位（`192.0.2.0/24` 是文档专用地址）。换成你自己的室内机号、门外机号和地址。

## 配置文件

默认读 `/etc/dooraccess-go/config.ini`，可以用 `--config <路径>` 换。正式安装时，启动脚本传的是 `/etc/dooraccess/config.ini`。

```ini
sip = 01020304@192.0.2.10:18022          ; 本机室内机
iface = br-door                          ; 门禁网卡

[listen]
addr = 0.0.0.0
port = 8080

[station "1"]                            ; 门外机，可以写多个 station "N"
sip = 01020000@192.0.2.20:18022

[hass]
api = http://192.0.2.30:8123             ; Home Assistant 地址
token = <HA Long-Lived Token>            ; 反向通知用的长期令牌

[video]
forward = true
format = flv

[automation]
auto_unlock = false                      ; 出厂：不自动开门
auto_hangup = false                      ; 出厂：不自动挂断
```

`sip` 是这台室内机在门禁网上的号码，形式是 `8位号码@IP:18022`。`iface` 是接门禁网的那块网卡，抓门铃和发开门都走它。

`[station "N"]` 列出允许控制的门外机。视频开始时，请求里的门外机必须在这份名单里。

`[hass]` 用来把门铃、开门、开关状态推回 Home Assistant。`token` 是 HA 的 Long-Lived Access Token。

`[automation]` 只是出厂默认。运行中用下面的 HTTP 接口改过的开关，以状态文件为准。

## 开关记在哪

「自动开门」和「自动挂断」会写到单独的状态文件，不写回 `config.ini`。

默认路径是 `/etc/dooraccess-go/automation.state`。正式安装时，启动脚本用 `--state` 指到 `/etc/dooraccess/automation.state`。这个路径不会从配置文件的位置推出来，要换就显式传 `--state`。

启动时按这个顺序取值：状态文件 > 配置文件里的 `[automation]` > 环境变量。文件不存在（第一次装、或还没拨过开关）就用配置里的出厂值。

## HTTP 接口

监听 `0.0.0.0:8080`。这个端口没有登录。谁能访问它，谁就能开门。需要的话，在路由器上限制来源 IP，只放 Home Assistant 那台机器。

| 方法 / 路径 | 做什么 |
|---|---|
| `GET /info` | 探活。返回品牌、室内机、门外机、视频是否开启 |
| `POST /unlock` | 开门 |
| `POST /auto_unlock` | 开关「有人按铃就自动开门」。正文 `{"on": true}` 或 `{"on": false}`，会写盘 |
| `POST /auto_hangup` | 开关「自动挂断」，同样写盘 |
| `GET /automation` | 看这两个开关现在是开还是关 |
| `POST /video/start` | 开始看门口画面。正文 `{"outdoor":"<门外机 URI>"}`，必须在配置的 station 名单里。返回会话号、流地址和存活时间 |
| `GET /video/<id>/stream.flv` | 拉 FLV 视频流 |
| `POST /video/stop` | 停止这次观看 |
| `GET /playback` | 回放入口，目前是空的 |

日志每一行自己带时间，形如 `dooraccess-rs: YYYY/MM/DD HH:MM:SS.mmm ...`，不依赖系统日志服务打时间戳。
