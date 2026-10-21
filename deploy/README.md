# armv7 镜像与玩客云部署

上游 `CangShui/workbuddy-gateway` 是纯 Go、零 CGO 的单二进制，官方 release 有
`workbuddy-gateway-linux-armv7` 但**没有容器镜像**。这里自建 armv7 镜像，供玩客云
（OneCloud / S805，armv7l）以 CasaOS 应用方式运行。

## 镜像

| 项 | 值 |
| --- | --- |
| 镜像 | `ghcr.io/wliky/workbuddy-gateway` |
| tag | `:armv7`（固定别名）、`:v1.14.21`（跟上游 release tag）、`:latest` |
| 平台 | `linux/arm/v7`（GOARM=7） |
| 基础 | `alpine:3.23` + 静态二进制（约 8.3 MiB 二进制） |
| 端口 | `8317` 模型 API，`8316` 网页控制台 |

构建流程见 `.github/workflows/build-armv7.yml`：每天 04:00 UTC 查上游最新 release，
**已有同 tag 镜像则跳过**；手动 dispatch 可指定 `tag` 或 `force` 重建。

```bash
# 手动重建（本地唯一能往 api.github.com 发带 body POST 的通道）
curl -s --ssl-no-revoke -X POST \
  -H "Authorization: Bearer $PAT" -H 'Content-Type: application/json' \
  -d '{"ref":"main","inputs":{"force":true}}' \
  https://api.github.com/repos/Wliky/workbuddy-gateway/actions/workflows/build-armv7.yml/dispatches
```

## 部署（CasaOS）

```bash
mkdir -p /DATA/AppData/workbuddy-gateway && chmod 700 /DATA/AppData/workbuddy-gateway
curl -s -X POST -H 'Content-Type: application/yaml' \
  --data-binary @docker-compose.yml http://127.0.0.1/v2/app_management/compose
```

`Content-Type` 必须是 `application/yaml`，别的写法会被 CasaOS 拒绝。

卸载：`curl -s -X DELETE http://127.0.0.1/v2/app_management/compose/workbuddy-gateway`
（只删应用，不动 `/DATA/AppData/workbuddy-gateway`）。

## 数据与账号

全部状态都落在 `/DATA/AppData/workbuddy-gateway`（容器内 `/data`）：

| 文件 | 说明 |
| --- | --- |
| `config.json` | 网关配置（`gateway.apiKeyEnabled` / `apiKey` / `adminKey` / `webPort`） |
| `workbuddy*.json` | 账号凭据，**多个文件 = 多账号池**，5 秒热加载 |
| `logs/` | 运行日志 |
| `wb-models-cache.json`、`workbuddy-status.json` | 模型目录缓存、运行状态快照 |

加账号两种方式：

1. **网页**（推荐）：`http://<设备IP>:8316/ui/` → 账号池 → 添加账号 → 选站点、填凭据名 →
   获取登录接口 → 手机扫码。扫码后自动写凭据并热加载，不用重启。
   国内站会话 5 分钟、国际站 15 分钟。
2. **命令行**：`docker exec -it workbuddy-gateway /usr/local/bin/workbuddy-gateway login`。

凭据字段（与 WorkBuddy 官方客户端同源）：

```json
{
  "auth": { "accessToken": "...", "refreshToken": "...", "expiresAt": 1795096800, "domain": "..." },
  "account": { "uid": "...", "enterpriseId": "", "nickname": "..." },
  "edition": "cn"
}
```

> 注意 `edition`（`cn` / `intl`）是本站点新增字段。从别的面板迁来的凭据如果只有
> `auth.realm`，需要把 `realm` 的值复制到顶层 `edition`，否则会被当成国内站。

## 排错

| 现象 | 原因 |
| --- | --- |
| 宿主机访问 8316/8317 不通 | 少了 `-addr 0.0.0.0`（默认只监听 127.0.0.1，且网页控制台跟着这个地址走） |
| 容器起来立刻退出 | 挂载目录不可写，或工作目录不是 `/data` |
| 日志里 upstream 全部失败 | 出站被墙/证书问题，可加 `-proxy` 或检查 CA（镜像已内置 ca-certificates） |
| 账号频繁掉线 | 同一个账号的凭据被两个网关同时刷新（refresh token 轮换互踢）→ 只保留一个在跑 |
