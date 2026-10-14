# 部署

目标平台为 MikroTik hAP ac lite：MIPS 大端、softfloat、OpenWrt musl、64MB RAM。产物为全静态单文件二进制。

在设备上替换二进制、重启或修改启动项时，按 `dooraccess-go/DEPLOY.md` 执行。本文仅说明部署形态与内存约束。本仓库的 Makefile 不包含远程部署目标。

## 部署形态

正式部署由 procd 托管。二进制位于 `/usr/bin/dooraccess-rs`，开机自启；进程异常退出后由 procd 重新拉起。配置与状态位于 `/etc/dooraccess/`。Go 版本保留在设备上并禁用，作为紧急回滚。安装与回滚见 [package/README.md](../package/README.md)；逐步规程见 `dooraccess-go/DEPLOY.md` 的 Rust 生产部署一节。

临时验证不替换正式二进制。将文件复制到 `/tmp/dooraccess-rs`，以 `setsid` 在后台运行，验证结束后恢复 Go 版本。该设备的 busybox 提供 `setsid`，不提供 `nohup`。步骤见 `DEPLOY.md` 的 Rust 临时验证一节。

## 约束

在 64MB RAM 上，下列操作可能导致进程被终止或设备失联：

- 禁止使用 `sha256sum` 或 `md5sum` 校验二进制。二者会将文件读入内存；已发生过因此断网、需物理重启的情况。校验使用 `ls -l` 的字节数。
- SSH 须附加 `-o ConnectTimeout=8 -o ServerAliveInterval=15`。连接失败后停止，不要重试。
- 不要在单次 SSH 会话中串联多步操作。每步独立连接，完成后断开。
