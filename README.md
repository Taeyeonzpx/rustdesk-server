# RustDesk Server Program

[![build](https://github.com/Taeyeonzpx/rustdesk-server/actions/workflows/build.yaml/badge.svg)](https://github.com/Taeyeonzpx/rustdesk-server/actions/workflows/build.yaml)

## 本仓库改动（中文说明）

这是 [rustdesk/rustdesk-server](https://github.com/rustdesk/rustdesk-server) 的个人 fork，改动有三处：`.github/workflows/build.yaml`、`Cargo.toml` 和 `src/rendezvous_server.rs`。

**改了什么**

- 删除 Docker Hub、GHCR 的登录和推镜像任务
- 删除 GitHub Release 和 deb 打包任务
- 只保留编译：Linux 四个架构（amd64 / arm64v8 / armv7 / i386），编译结果上传为 Artifact
- 不需要配置任何 secrets，也不会向任何镜像仓库推送
- `Cargo.toml` 里给 `openssl` 开了 `vendored`：cross 的 musl 镜像没有系统 openssl，改成随源码静态编译
- `src/rendezvous_server.rs`：参考 [lejianwen/rustdesk-server](https://github.com/lejianwen/rustdesk-server)，修复「客户端登录 API 账号后连接超时」——TCP/WS 连接也会回应注册请求，ws 连接的 sink 会被保存下来用于 punch hole 转发

**怎么用**

1. 修改代码，提交并 push
2. 打开 Actions → `build` → Run workflow（或打 tag：`git tag v1.0.0 && git push origin v1.0.0`）
3. 运行结束后，在页面底部 Artifacts 下载 `binaries-linux-amd64`
4. 解压后得到 `hbbs`、`hbbr`、`rustdesk-utils` 三个文件
5. 在 Linux 服务器上打包镜像：

```bash
unzip binaries-linux-amd64.zip -d docker/rootfs/usr/bin/
chmod +x docker/rootfs/usr/bin/hbbs docker/rootfs/usr/bin/hbbr docker/rootfs/usr/bin/rustdesk-utils

docker build -f docker/Dockerfile --build-arg S6_ARCH=x86_64 -t rustdesk-server-s6:dev docker/
docker compose up -d
```

**说明**

- 镜像只在本地或服务器上构建，不经过 GitHub
- 编译产物是 musl 静态链接，可以直接放进 busybox / scratch 镜像
- 服务器内存不够时，可以在别的机器编译后 `docker save` / `docker load` 传输镜像

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
