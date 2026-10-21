# syntax=docker/dockerfile:1
#
# workbuddy-gateway armv7 镜像（玩客云 / S805 32 位 ARM）
#
# 关键设计：
#   * 构建阶段固定 --platform=$BUILDPLATFORM（跑在 runner 的 x86 上），
#     真正编译靠 GOOS/GOARCH/GOARM 交叉编译 → 全程不需要 QEMU 模拟。
#   * 目标阶段只有 COPY，没有 RUN → arm/v7 层不需要任何模拟执行。
#   * 运行基础镜像用 alpine（保留 /tmp、sh、busybox），但**不装任何包**，
#     CA 证书与 zoneinfo 直接从构建阶段 COPY 过来。
#
# 上游是纯 Go + 零 CGO（依赖仅 skip2/go-qrcode、google/uuid），
# 所以 CGO_ENABLED=0 产出的是完全静态二进制。

ARG GO_VERSION=1.26

FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS build
ARG TARGETOS TARGETARCH TARGETVARIANT
# 只装证书与时区，供最终镜像 COPY；这一步在构建平台执行，与目标架构无关。
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
ENV CGO_ENABLED=0 GOTOOLCHAIN=local
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH GOARM=${TARGETVARIANT#v} \
    go build -trimpath -buildvcs=false -ldflags "-s -w" \
    -o /out/workbuddy-gateway .

FROM alpine:3.23
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /out/workbuddy-gateway /usr/local/bin/workbuddy-gateway

# 程序把 config.json / workbuddy*.json / logs/ 全部落在工作目录 → 一个卷装全部状态
ENV TZ=Asia/Shanghai HOME=/data
WORKDIR /data
VOLUME ["/data"]

# 8317 = 模型 API；8316 = 网页控制台（-webui 时启用）
EXPOSE 8316 8317

ENTRYPOINT ["/usr/local/bin/workbuddy-gateway"]
# 容器内必须监听 0.0.0.0，否则宿主机/CasaOS 访问不到（默认是 127.0.0.1）
CMD ["serve", "-addr", "0.0.0.0", "-webui"]
