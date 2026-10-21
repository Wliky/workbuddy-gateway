# FORK.md — 与上游的差异

上游：`CangShui/workbuddy-gateway`（本仓库是它的 fork，`Wliky/workbuddy-gateway`）

## 原则：只增不改

本 fork **不修改上游任何现有文件**，只新增下面这些，所以同步上游永远零冲突：

| 路径 | 作用 |
| --- | --- |
| `.github/workflows/build-armv7.yml` | 自建 armv7 镜像 → GHCR（每天查上游 release，已有则跳过） |
| `deploy/Dockerfile` | 多阶段交叉编译 → alpine 运行层（构建阶段固定 BUILDPLATFORM，全程免 QEMU） |
| `deploy/Dockerfile.dockerignore` | 精简构建上下文 |
| `deploy/docker-compose.yml` | 玩客云 CasaOS 应用定义（含 `x-casaos` 段） |
| `deploy/README.md` | 部署 / 账号迁移 / 排错 |
| `FORK.md` | 本文件 |

## 为什么需要这个 fork

上游是纯 Go、零 CGO 单二进制，release 已提供 `workbuddy-gateway-linux-armv7`，
所以**技术上不需要自建二进制**。fork 的唯一目的是把它做成容器镜像，好进 CasaOS 面板，
并让 watchtower 能跟着上游发版自动更新。

## 同步上游

```bash
git remote add upstream https://github.com/CangShui/workbuddy-gateway.git
git fetch upstream --tags
git merge upstream/main        # 只增不改 → 应当永远零冲突
git -c http.proxy=http://127.0.0.1:7890 -c http.version=HTTP/1.1 push origin main
```

若上游改了 `go.mod` 的 Go 版本不用手动跟进：workflow 会从 `go.mod` 读 `X.Y` 并传给
`--build-arg GO_VERSION`。

## 踩坑记录

1. **必须 `-addr 0.0.0.0`**：程序默认 `127.0.0.1`，而且**网页控制台 8316 跟着这个地址走**
   （实测 `-addr 127.0.0.1` 时两个端口都只听回环）。容器里不写这个参数 = 宿主机完全访问不到。
2. **建库只需 arm/v7**：构建阶段 `--platform=$BUILDPLATFORM` + 交叉编译，目标层只有 `COPY`
   没有 `RUN`，所以**不需要 `docker/setup-qemu-action`**；`alpine` 基础镜像也不装包，
   CA 证书与 zoneinfo 从构建阶段 COPY。
3. **凭据字段差异**：本站新增顶层 `edition`（`cn`/`intl`），旧格式（如 wb2api-panel）只有
   `auth.realm`。迁移时要补 `edition`，多余字段 Go 会忽略。
4. **同一账号别被两个网关同时刷新**：refresh token 会轮换，两个网关各自刷新会互踢
   （表现为 12153 / 掉登录）。迁移完成后只保留一个在跑。
5. **GHCR 存在性探测必须带 Accept 头**，否则已存在的 tag 也返回 404，导致每天重复构建。
6. **镜像名必须全小写**：`github.repository_owner` 是 `Wliky`，要在 step 里 `tr` 成小写。
