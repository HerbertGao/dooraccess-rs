# 构建与开发

本机构建与测试使用 stable。MIPS 交叉构建需要 nightly、`rust-src`，以及 OpenWrt mips_24kc musl sysroot。

## 本机

```bash
make build
make test
make clippy
bash scripts/check-log-gate.sh
```

`make test` 包含协议字节与降级路径测试。`make clippy` 要求无警告。`scripts/check-log-gate.sh` 检查日志均经带时间戳的日志层写出，且写日志时只加锁一次。

pre-commit 钩子执行 `cargo fmt --check`。未格式化的变更不能提交。

## MIPS 交叉构建

`mips-unknown-linux-musl` 为 Tier 3 目标，rustup 不提供预编译的 std。交叉构建使用 nightly，以 `-Z build-std` 从 `rust-src` 编译 std，并链接 OpenWrt musl sysroot。`scripts/link-mips.sh` 使用宿主机的 `rust-lld` 与 SDK 静态库完成全静态、非 PIE 链接，因此可在 macOS arm64 上直接生成 MIPS 二进制。

```bash
rustup toolchain install nightly --profile minimal
rustup component add rust-src llvm-tools-preview --toolchain nightly
make fetch-sdk        # 将 OpenWrt mips_24kc musl sysroot 下载到 .openwrt-sdk/
make build-mips       # target/mips-unknown-linux-musl/release/dooraccess-rs
make verify-mips      # 校验 MIPS 大端、softfloat、全静态
make dist             # 复制到 dist/dooraccess-rs-mips
```

`make dist` 不触发 `build-mips`。修改源码后须先执行 `make build-mips`。

若 `cargo` 解析到 Homebrew 版本且不支持 `+nightly`，将 rustup 的目录置于 `PATH` 之前：

```bash
PATH="$HOME/.cargo/bin:$PATH" make build-mips
```

release profile 为 `panic=abort`、LTO、`opt-level=z`、strip。产物字节数以构建后的 `ls -l` 为准。生产部署使用 CI 产物，而非本机 `make dist` 的输出。见 [package/README.md](../package/README.md)。

## 模块

PF_PACKET 监听位于独立线程。出站报文经单个 worker 与任务队列串行发送，以避免并发写网卡。

| 范围 | 模块 |
|---|---|
| 编码、帧与抓包 | `codec`、`wire18022`、`listen18022`、`listen6672`、`bpf`、`ffi` |
| HTTP 与控制面 | `httpx`、`control`、`info` |
| 进程编排与自动开锁 | `daemon`、`orchestration`、`self_unlock`、`automation_state` |
| 开锁 | `unlock`、`wire_sender`、`sender` |
| 视频 | `video/`：预览信令、RTP 重组、FLV、会话超时、多路转发 |
| 日志 | `log` |

门铃事件经 UDP 6672 接收。开锁与视频信令经 TCP 18022 发送。门口视频为 UDP 9880 上的 RTP/H.264，重组为 FLV 后经 HTTP 输出。

## 约束

- 外部依赖仅 `libc`，且仅用于 PF_PACKET 与 `SO_BINDTODEVICE`。不引入 nix、pnet、socket2、tokio。放宽该限制须经 OpenSpec change。
- 目标 CPU 为 MIPS32，禁止使用 64 位原子。超时使用 `Mutex<Instant>`，计数使用 `AtomicU32`。
- 行为变更经 OpenSpec 管理：提案、实现，归档后生效。
- 代码变更经 PR 提交，并由 Bugbot 复审。
