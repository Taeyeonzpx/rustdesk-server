# RustDesk Server Program

[![build](https://github.com/Taeyeonzpx/rustdesk-server/actions/workflows/build.yaml/badge.svg)](https://github.com/Taeyeonzpx/rustdesk-server/actions/workflows/build.yaml)

## 本仓库改动（中文说明）

这是 [rustdesk/rustdesk-server](https://github.com/rustdesk/rustdesk-server) 的个人 fork，只做了下面几件事：

### 1. 准备工作：改 `build.yaml`

原版 workflow 会登录 Docker Hub / GHCR 推镜像、发 GitHub Release、打 deb 包，需要配置 secrets。这里全部删掉，只留编译。

**改了什么**

- 删除 Docker Hub、GHCR 的登录和推镜像任务
- 删除 GitHub Release 和 deb 打包任务
- 只保留编译：Linux 四个架构（amd64 / arm64v8 / armv7 / i386），编译结果上传为 Artifact
- 不需要配置任何 secrets，也不会向任何镜像仓库推送
- `Cargo.toml` 里给 `openssl` 开了 `vendored`：cross 的 musl 镜像没有系统 openssl，改成随源码静态编译

### 2. 搬运 [lejianwen/rustdesk-server](https://github.com/lejianwen/rustdesk-server) 的 KeyExchange（commit `9bae6605`）：解决「客户端登录 API 账号后连接超时」

**问题原因**：客户端通过 API 登录后会拿到 token，按 RustDesk 客户端逻辑（`rustdesk/src/client.rs`）token 非空就强制要求走 KeyExchange 加密握手。原版 hbbs 的 TCP 通道不支持 KeyExchange，客户端等不到握手响应就超时。

**改动文件**：`src/rendezvous_server.rs`

- `Sink` 增加加密发送路径，`Inner` 增加 `secure_tcp_pk_b` / `secure_tcp_sk_b`
- 新增 `key_exchange_phase1()` 和 `KeyExchange` 消息分支：TCP 连接收到 KeyExchange 时做 X25519 密钥交换
- 新增 `get_symmetric_key_from_msg()`：从握手消息派生加密通道用的对称密钥
- 接收循环对加密通道先解密再处理

**没有搬的**（与超时无关）：`MUST_LOGIN` 强制登录、`RUSTDESK_API_JWT_KEY` JWT 校验、ws（WebSocket）支持、`OnlineRequest` 在线状态查询。

**怎么用**

1. 修改代码，提交并 push
2. 打开 Actions → `build` → Run workflow（或打 tag：`git tag v1.0.0 && git push origin v1.0.0`）
3. 运行结束后，在页面底部 Artifacts 下载 `binaries-linux-amd64`
4. 解压后得到 `hbbs`、`hbbr`、`rustdesk-utils` 三个文件
5. 在 Linux 服务器上打包镜像（classic，`FROM scratch`）：

```bash
unzip binaries-linux-amd64.zip -d docker-classic/
chmod +x docker-classic/hbbs docker-classic/hbbr

docker build -f docker-classic/Dockerfile -t rustdesk-server:<版本号> docker-classic/
docker compose up -d
```

**说明**

- 镜像只在本地或服务器上构建，不经过 GitHub
- 编译产物是 musl 静态链接，可以直接放进 busybox / scratch 镜像
- 服务器内存不够时，可以在别的机器编译后 `docker save` / `docker load` 传输镜像

### 3. 放宽中继带宽默认值（10 Gb/s）

**改动文件**：`src/relay_server.rs`、`docs/environment-variables.md`

| 变量 | 上游默认 | 本 fork | 说明 |
|---|---|---|---|
| `SINGLE_BANDWIDTH` | 128 Mb/s | **10240 Mb/s** | 单条中继连接上限 |
| `TOTAL_BANDWIDTH` | 1024 Mb/s | **10240 Mb/s** | 所有中继连接合计上限 |
| `LIMIT_SPEED` | 32 Mb/s | 不变 | 只作用于黑名单/被降级的连接 |

- 单条连接实际上不再被单独限速，真实速度由 NAS 的上行带宽和 CPU 决定
- 两个计数器改成 `AtomicU64`：`usize` 在 32 位（i386 / armv7）装不下 10 Gb
- 降级阈值随 `SINGLE_BANDWIDTH` 一起放大（约 6.7 Gb/s 持续 30 分钟才触发），正常使用不会触发
- 想改回保守值：给 hbbr 设环境变量 `SINGLE_BANDWIDTH=128`、`TOTAL_BANDWIDTH=1024`，或在 loopback console 里发 `sb 128`、`tb 1024`

### 4. 已同步上游 `upstream/master`

合并了消息大小上限（`MAX_MESSAGE_KB`，默认 64 KB）、WebSocket 握手/console 回复/relay 写超时、relay waiter 过期修复等安全加固。这些改动和 KeyExchange 走的是不同代码路径，互不影响。

[**Download**](https://github.com/rustdesk/rustdesk-server/releases)

[**Manual**](https://rustdesk.com/docs/en/self-host/)

[**Configuration & environment variables**](docs/environment-variables.md)

[**FAQ**](https://github.com/rustdesk/rustdesk/wiki/FAQ)

[**How to migrate OSS to Pro**](https://rustdesk.com/docs/en/self-host/rustdesk-server-pro/installscript/#convert-from-open-source)

Self-host your own RustDesk server, it is free and open source.

> [!IMPORTANT]
> **Need more features?** [RustDesk Server Pro](https://rustdesk.com/pricing.html) might suit you better.
>
> **Want to develop your own server?** Start with [rustdesk-server-demo](https://github.com/rustdesk/rustdesk-server-demo), a simpler starting point than this repository.

## How to build manually

```bash
cargo build --release
```

Three executables will be generated in target/release.

- hbbs - RustDesk ID/Rendezvous server
- hbbr - RustDesk relay server
- rustdesk-utils - RustDesk CLI utilities

You can find updated binaries on the [Releases](https://github.com/rustdesk/rustdesk-server/releases) page.

## Configuration

`hbbs` and `hbbr` can be configured with command-line flags, environment
variables, or an `.env` / config file. Run `hbbs --help` or `hbbr --help` to see
the available flags.

The most common options:

| Option | Flag | Env var | Applies to | Purpose |
| --- | --- | --- | --- | --- |
| Key | `-k` | `KEY` | hbbs, hbbr | `hbbs` loads/generates one by default |
| Bind address | `-b` | `BIND` | hbbs, hbbr | Local IP address to listen on (default: all interfaces; requires 1.1.17+) |
| Port | `-p` | `PORT` | hbbs, hbbr | Listening port (hbbs `21116`, hbbr `21117`) |
| Relay servers | `-r` | `RELAY-SERVERS` | hbbs | Override when the relay uses a different address or a non-standard port |
| Force relay | — | `ALWAYS_USE_RELAY` | hbbs | `Y` disables direct connections |
| Log level | — | `RUST_LOG` | hbbs, hbbr | e.g. `debug` (default `info`) |

See **[docs/environment-variables.md](docs/environment-variables.md)** for the
full list of variables, the file/flag/env precedence rules, database and relay
bandwidth tuning, Docker image variables, and examples.

## Installation

Please follow this [doc](https://rustdesk.com/docs/en/self-host/rustdesk-server-oss/)
