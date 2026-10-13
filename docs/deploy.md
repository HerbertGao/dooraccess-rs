# 怎么装到路由器上

目标机器是 MikroTik hAP ac lite：64MB 内存，MIPS 大端，没有浮点。程序是单个静态文件，拷上去就能跑。

真机上替换程序、重启、改启动项，逐步命令以 `dooraccess-go/DEPLOY.md` 为准。本文只说明有哪两种装法，以及哪几件事会把机器打挂。本仓库的 `Makefile` 不会自动连到路由器。

## 两种装法

正式安装由 OpenWrt 的 procd 托管：程序在 `/usr/bin/dooraccess-rs`，开机自启，崩了会拉起来。配置和开关状态放在 `/etc/dooraccess/`。原来的 Go 程序留在机器上，停用，当作紧急退回。安装、退回的步骤概览在 [package/README.md](../package/README.md)，完整步骤在 `dooraccess-go/DEPLOY.md` 的 Rust 生产部署一节。

临时验证不替换正式程序：把二进制拷到 `/tmp/dooraccess-rs`，用 `setsid` 前台拉到后台跑，测完再把 Go 程序拉起来。这台机器的 busybox 有 `setsid`，没有 `nohup`。逐步命令在 `DEPLOY.md` 的 Rust 临时验证一节。

## 不要做的事

这台机器内存很紧，下面几件事已经出过事：

- 不要用 `sha256sum` 或 `md5sum` 核对二进制。它们会把整个文件读进内存，曾经把机器打到断网，只能物理重启。核对用 `ls -l` 看字节数。
- SSH 加上 `-o ConnectTimeout=8 -o ServerAliveInterval=15`。一次失败就停，不要连着重试。多次连上去会再占内存。
- 不要把很多步串成一次很长的远程命令。做完一步就断开。
