# 配置与接口

守护进程读取 INI 配置，并在 8080 端口提供 HTTP 接口，供 Home Assistant 及配套 HACS 集成调用。

示例中的号码与 IP 为占位值。`192.0.2.0/24` 为文档专用地址，不对应真实设备。

## 配置

默认路径为 `/etc/dooraccess-go/config.ini`，可由 `--config <path>` 覆盖。生产环境的 init 脚本传入 `/etc/dooraccess/config.ini`。

```ini
sip = 01020304@192.0.2.10:18022          ; 室内机
iface = br-door                          ; 门禁网卡

[listen]
addr = 0.0.0.0
port = 8080

[station "1"]                            ; 室外机，可配置多项 station "N"
sip = 01020000@192.0.2.20:18022

[hass]
api = http://192.0.2.30:8123             ; Home Assistant 地址
token = <HA Long-Lived Token>            ; 反向通知用的长期令牌

[video]
forward = true
format = flv

[automation]
auto_unlock = false                      ; 默认关闭自动开锁
auto_hangup = false                      ; 默认关闭自动挂断
```

`sip` 为本机室内机 URI，格式为 `8位号码@IP:18022`。`iface` 为门禁网卡，用于抓帧与出站绑定。

`[station "N"]` 为允许控制的室外机列表。`POST /video/start` 中的室外机 URI 必须位于该列表内。

`[hass]` 用于向 Home Assistant 推送门铃、开锁与自动化状态。`token` 为 Long-Lived Access Token。

`[automation]` 为出厂默认值。运行时经 HTTP 修改的开关以状态文件为准。

## 状态文件

自动开锁与自动挂断写入独立状态文件，不回写 `config.ini`。

默认路径为 `/etc/dooraccess-go/automation.state`。生产环境由 `--state` 指定为 `/etc/dooraccess/automation.state`。状态路径不从配置路径派生；变更时须显式传入 `--state`。

加载优先级为：状态文件 > `[automation]` > 环境变量。状态文件不存在时使用配置中的默认值。

## HTTP

监听 `0.0.0.0:8080`，无认证。能够访问该端口的主机即可发起开锁。如需限制访问，在防火墙上仅放行 Home Assistant 主机。

| 方法 / 路径 | 作用 |
|---|---|
| `GET /info` | 探活。返回品牌、室内机、室外机及视频转发状态 |
| `POST /unlock` | 开锁 |
| `POST /auto_unlock` | 切换门铃自动开锁。请求体为 `{"on": true}` 或 `{"on": false}`，结果写入状态文件 |
| `POST /auto_hangup` | 切换自动挂断，同样写入状态文件 |
| `GET /automation` | 返回当前自动开锁与自动挂断状态 |
| `POST /video/start` | 启动视频会话。请求体为 `{"outdoor":"<室外机 URI>"}`，且须位于 station 列表内。响应包含会话标识、流地址与存活时间 |
| `GET /video/<id>/stream.flv` | 获取 FLV 视频流 |
| `POST /video/stop` | 停止视频会话 |
| `GET /playback` | 回放入口，当前为空实现 |

日志行自带时间戳，格式为 `dooraccess-rs: YYYY/MM/DD HH:MM:SS.mmm <msg>`，不依赖 syslog 补充时间。
