# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JRebel License Active Server - 用 Go 编写的 JRebel/XRebel 许可证激活服务器。提供模拟的许可证验证 API 端点。纯 Go 标准库实现，无外部依赖（go.mod 中无 require）。Go 版本要求 >= 1.19。

## Build & Run

```bash
go build ./                          # 本地构建
./jrebel-license-active-server       # 默认端口 12345
./jrebel-license-active-server --port=5555 --exportSchema=https --exportHost=jrebel.domain.com

# 跨平台编译（CI 中也构建 arm64）
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ./
CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build ./
```

Docker: `docker build -t jrebel-license-active-server .` / `docker compose up`

注意：项目无测试文件、无 lint 配置。

## Architecture

所有代码在 `main` 包中，单一 package 平铺结构。

### 请求处理流程

1. `main.go` — 入口点，`init()` 解析命令行参数到全局 `config`，`main()` 注册路由并启动 `http.ListenAndServe`
2. `jrebelhandler.go` — 所有 HTTP handler 实现。请求体通过 `getHttpBodyParameter()` 统一解析为 `url.Values`（将 body 当作 query string 解析）。首页使用内嵌的 Go template（`indexTemplateHtml` 常量）
3. `jrebelstruct.go` — JSON 响应模板。通过 `init()` 将 JSON 常量反序列化为全局 struct 变量（`jRebelLeases`, `jrebelLeases1`, `jrebelValidate`），handler 在这些 struct 上修改字段后序列化返回

### 签名机制

两套 RSA 私钥在 `jrebelprivatekey.go` 中硬编码：
- `privateKey` — 用于 SHA1 签名（租约签名，`signWithSha1`）
- `asmPrivateKey` — 用于 MD5 签名（ping/ticket 等 RPC 接口，`signWithMd5`）

签名流程：`jrebelsign.go` 中 `toLeaseCreateJson()` 拼接参数字符串 → SHA1 签名 → Base64 编码

### 工具文件

- `byteutil.go` — Base64 编解码
- `uuid.go` — 自实现 UUID v4 生成（不依赖外部库）
- `util.go` — 生成随机 serverRandomness 字符串
- `logger.go` — 自实现日志系统（ILogger 接口），支持 Console/File/Multi 三种模式，文件日志按日期+大小自动轮转

### API Endpoints

| 路径 | 用途 |
|------|------|
| `/` | 首页，显示激活地址（Go template 渲染） |
| `/jrebel/leases`, `/agent/leases` | 租约请求（共用 `jrebelLeasesHandler`） |
| `/jrebel/leases/1`, `/agent/leases/1` | 租约验证（共用 `jrebelLeases1Handler`） |
| `/jrebel/validate-connection` | 连接验证 |
| `/rpc/ping.action` | Ping（XML + MD5 签名） |
| `/rpc/obtainTicket.action` | 获取票据（XML + MD5 签名） |
| `/rpc/releaseTicket.action` | 释放票据（XML + MD5 签名） |

### 配置参数

全局 `config` 在 `main.go` 中通过 `flag` 解析，所有参数均有默认值：
- `--port` 默认 12345
- `--offlineDays` 默认 30（超过 180 会导致无效）
- `--exportSchema` / `--exportHost` 控制首页显示的 URL
- `--jrebelWorkMode` 0:自动 1:强制离线 2:强制在线 3:oldGuid
- `--logFile` / `--logPath` / `--level` 日志相关

## CI/CD

- `.github/workflows/go.yml` — push master 或 tag 时构建 5 个平台二进制，tag 时发布 GitHub Release
- `.github/workflows/docker-build-push.yml` — tag 时构建并推送多架构 Docker 镜像到 DockerHub
