# 怎么编译，怎么改

日常在自己的电脑上用稳定版 Rust 编译和测试。要出路由器上的程序，才需要 nightly 和一份 OpenWrt 的系统库。

## 本机

```bash
make build
make test
make clippy
bash scripts/check-log-gate.sh
```

`make test` 覆盖协议字节、降级路径这些单测。`make clippy` 要求零警告。日志脚本检查两件事：日志都走带时间戳的那一层，以及写日志时只加一次锁。

仓库的 pre-commit 会跑 `cargo fmt --check`，没格式化不能提交。

## 路由器上的程序

`mips-unknown-linux-musl` 是 Rust 的 Tier 3 目标，rustup 不带现成的标准库。交叉编译要用 nightly，从源码编标准库，再链到 OpenWrt 的 musl 库上。链接脚本用的是本机的 `rust-lld`，所以 Mac（arm64）上也能直接出 MIPS 程序，不用跑 SDK 里的 Linux gcc。

```bash
rustup toolchain install nightly --profile minimal
rustup component add rust-src llvm-tools-preview --toolchain nightly
make fetch-sdk        # 下载 OpenWrt mips_24kc musl 库到 .openwrt-sdk/
make build-mips       # target/mips-unknown-linux-musl/release/dooraccess-rs
make verify-mips      # 检查：MIPS 大端、softfloat、全静态
make dist             # 再拷到 dist/dooraccess-rs-mips
```

`make dist` 不会重新编译。改完代码要先 `make build-mips`。

Mac 上如果 `cargo` 指到 Homebrew 的版本（不认 `+nightly`），把 rustup 的目录放前面：

```bash
PATH="$HOME/.cargo/bin:$PATH" make build-mips
```

发布用的体积优先：`panic=abort`、LTO、`opt-level=z`、去掉符号。具体字节数以编出来之后的 `ls -l` 为准，不要写死在文档里。生产部署用的是 CI 编出来的那一份，不是本机 `make dist` 的产物，见 [package/README.md](../package/README.md)。

## 代码大致怎么分

抓包在单独的线程里做。真正发出去的协议排进一个队列，由一个 worker 串行发送，避免两处同时写网卡。

| 一块事情 | 代码在哪 |
|---|---|
| 号码、帧、抓包 | `codec`、`wire18022`、`listen18022`、`listen6672`、`bpf`、`ffi` |
| HTTP 和控制 | `httpx`、`control`、`info` |
| 启停和自动开门 | `daemon`、`orchestration`、`self_unlock`、`automation_state` |
| 开门 | `unlock`、`wire_sender`、`sender` |
| 视频 | `video/`：预览信令、RTP 重组、FLV、会话超时、多路转发 |
| 日志 | `log` |

门铃走 UDP 6672。开门和视频开关走 TCP 18022。门口画面是 UDP 9880 上的 RTP/H.264，程序把它重组成 FLV，从 HTTP 送出去。

## 改代码时守住的几条

- 外部依赖只有 `libc`，而且只用在抓包和绑定网卡这几处。不要加 nix、pnet、socket2、tokio。要放宽，走 OpenSpec change。
- 这颗 CPU 是 MIPS32。不要用 64 位原子。超时用 `Mutex<Instant>`，计数用 `AtomicU32`。
- 行为变更走 OpenSpec：提案、实现、归档之后才算生效。
- 改动走 PR，并用 Bugbot 再看一遍。
