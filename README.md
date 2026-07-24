# ServerStatus Refined

> 基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的前端美化版，保留原版全部功能与部署方式，仅修改前端展示层（`web/` 目录），服务端代码、Agent 协议、Docker 构建、API 接口均与上游 100% 兼容。

[![License](https://img.shields.io/badge/license-MIT-4EB1BA.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Hub%20%2B%20GHCR-blue?logo=docker)](https://hub.docker.com/r/aoomee/serverstatus-refined)

## ✨ UI 改动

- 暖色极简配色，全站楷体，分层圆角（大容器 14px / 按钮 8px）
- 卡片 / 表格双布局一键切换
- 亮色 / 暗色主题，自动跟随系统偏好
- 概览卡片重排，告警三分类：离线 / 高载 / 阻断
- 三网延迟可视化药丸柱，排序标记圆点化
- 月流量左入绿 / 右出暖褐，配色协调统一
- 表格 `table-layout:fixed` + 固定列宽，排序切换无跳动
- 增量 DOM 更新 + `requestAnimationFrame` 批量渲染，数据刷新平滑无闪烁

如需恢复原版 UI，直接替换 `web/` 目录为原项目文件即可。

## 🎨 面板告警说明

表格行 / 卡片边框的颜色代表节点健康状态：

| 颜色 | 含义 | 触发条件（任意一项）|
|------|------|-------------------|
| 🔴 红色背景 | **Critical（严重）** | 离线、CPU ≥ 90%、内存 ≥ 90%、硬盘 ≥ 90%、任一网络丢包 ≥ 40%、三网全丢包（被墙） |
| 🟡 黄色背景 | **Warning（警告）** | CPU ≥ 75%、内存 ≥ 80%、硬盘 ≥ 85%、任一网络丢包 30%–40% |
| ⚪ 无背景 | **正常** | 以上均不满足 |

> 丢包率来自客户端对三网探测节点的 `ping` 统计，非延迟值。弹窗图表中的延迟曲线（ms）与颜色判断互不影响。

---

## 一、服务端

```bash
# Docker Compose，本地构建加：--build
ADMIN_TOKEN='your-strong-token' docker compose -f docker-compose-server.yml up -d --build
```

```bash
# Docker Run
wget -qO ~/serverstatus-config.json \
  --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/aoomee/ServerStatus-Refined/contents/server/config.json?ref=main'
mkdir -p ~/serverstatus-data

docker run -d --restart=always --name=serverstatus-server \
  -e ADMIN_TOKEN='your-strong-token' \
  -v ~/serverstatus-config.json:/app/config/config.json \
  -v ~/serverstatus-data:/app/data \
  -p 8080:80 -p 35601:35601 \
  aoomee/serverstatus-refined:server
```

> 镜像同时在 [Docker Hub](https://hub.docker.com/r/aoomee/serverstatus-refined) 和 [GHCR](https://github.com/aoomee/ServerStatus-Refined/pkgs/container/serverstatus-refined) 发布，Docker Hub 拉取 `aoomee/serverstatus-refined:server`，GHCR 拉取 `ghcr.io/aoomee/serverstatus-refined:server`。
> 也有简化版 `docker-compose.yml` 供快速部署：`docker compose up -d` 即可直接拉镜像运行。

启动后访问：

- WebUI：http://127.0.0.1:8080/
- 健康检查：http://127.0.0.1:8080/api/health
- API 描述：http://127.0.0.1:8080/api/schema
- OpenAPI 3.1：http://127.0.0.1:8080/api/openapi.json
- 客户端上报端口：`35601/tcp`

`ADMIN_TOKEN` 不设置时，监控页面仍可读取，管理 API 返回 `503`，WebUI 的"配置"页不能修改数据。

## 二、客户端

```bash
# Docker Compose，本地构建加：--build
SERVER=127.0.0.1 USER=s01 PASSWORD=USER_DEFAULT_PASSWORD \
docker compose -f docker-compose-client.yml up -d --force-recreate
```

```bash
# Docker Run
docker run -d --restart=always --name=serverstatus-client \
  --network=host --pid=host \
  -e SERVER=127.0.0.1 \
  -e USER=s01 \
  -e PASSWORD=USER_DEFAULT_PASSWORD \
  aoomee/serverstatus-refined:client
```

```bash
# Shell Run
wget -qO client-linux.py --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/clients/client-linux.py?ref=master'
nohup python3 client-linux.py SERVER=127.0.0.1 USER=s01 PASSWORD=USER_DEFAULT_PASSWORD >/dev/null 2>&1 &
```

`USER` 是常见的宿主机环境变量名。如果没有显式传递或传递方式不正确，Compose 可能会把系统中的 `$USER` 解析成本机用户名，而不是默认的 `s01`。推荐优先级：

1. 运行命令显式传递 `USER=s01`
2. 修改 `docker-compose-client.yml` 中的 `USER` 默认值
3. Docker 或系统环境中的 `USER`

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `SERVER` | `127.0.0.1` | Go 服务端地址 |
| `USER` | `s01` | 客户端用户名，必须匹配服务端配置 |
| `PORT` | `35601` | Agent TCP 上报端口 |
| `PASSWORD` | `USER_DEFAULT_PASSWORD` | 客户端密码，必须匹配服务端配置 |
| `INTERVAL` | `1` | 状态上报间隔，单位秒 |
| `PROBEPORT` | `80` | 三网 TCP 探测端口 |
| `PROBE_PROTOCOL_PREFER` | `ipv4` | 探测协议偏好，可选 `ipv4`、`ipv6` |
| `PING_PACKET_HISTORY_LEN` | `100` | 丢包历史窗口 |
| `CU` | `cu.tz.cloudcpp.com` | 联通探测地址 |
| `CT` | `ct.tz.cloudcpp.com` | 电信探测地址 |
| `CM` | `cm.tz.cloudcpp.com` | 移动探测地址 |
| `CLIENT` | `psutil` | 客户端实现，可选 `psutil`、`linux` |

## 服务端参数

Docker 镜像中的默认路径为：

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `CONFIG_PATH` | `/app/config/config.json` | 主配置文件 |
| `STATS_PATH` | `/app/data/stats.json` | 月流量与状态持久化文件 |
| `WEB_DIR` | `/app/web` | WebUI 静态文件目录 |
| `HTTP_ADDR` | `:80` | WebUI 与 HTTP API 监听地址 |
| `AGENT_ADDR` | `:35601` | 客户端 TCP 上报监听地址 |
| `ADMIN_TOKEN` | 空 | 管理 API Bearer Token；为空时禁用管理接口 |
| `ADMIN_CORS_ORIGIN` | 空 | 可选的 API CORS Origin |
| `INSECURE_CALLBACK_TLS` | `false` | 是否允许 Watchdog/证书回调使用不可信 TLS 证书 |
| `VERBOSE` | `false` | 输出 Gin HTTP 请求日志 |
| `TZ` | `Asia/Shanghai` | 容器时区 |

对应命令行参数：

```text
--config, -c     config.json 路径
--stats          stats.json 路径
--web-dir, -d    WebUI 目录
--http           HTTP 监听地址
--agent          Agent TCP 监听地址
--verbose, -v    详细请求日志
--version        输出构建版本
```

旧参数 `--bind/-b` 和 `--port/-p` 仍可用于设置 Agent TCP 监听地址。

## HTTP 管理 API

管理接口使用 Bearer Token：

```http
Authorization: Bearer <ADMIN_TOKEN>
```

无需认证：

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/api/health` | 进程、Agent TCP、版本和配置路径状态 |
| `GET` | `/api/schema` | 机器可读的端点与配置集合描述 |
| `GET` | `/api/openapi.json` | 可供 AI Agent 直接导入的 OpenAPI 3.1 文档 |
| `GET` | `/json/stats.json` | WebUI 使用的实时状态快照 |

需要认证：

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET/PUT` | `/api/config` | 读取或整体替换配置 |
| `GET/POST` | `/api/servers` | 查询或新增节点 |
| `PUT/DELETE` | `/api/servers/{username}` | 修改或删除节点 |
| `POST` | `/api/servers/{username}/reset-traffic` | 将当前流量设为本月基线 |
| `GET/POST` | `/api/monitors` | 查询或新增服务监测 |
| `PUT/DELETE` | `/api/monitors/{index-or-name}` | 修改或删除服务监测 |
| `GET/POST` | `/api/sslcerts` | 查询或新增证书监测 |
| `PUT/DELETE` | `/api/sslcerts/{index-or-name}` | 修改或删除证书监测 |
| `GET/POST` | `/api/watchdog` | 查询或新增 Watchdog |
| `PUT/DELETE` | `/api/watchdog/{index-or-name}` | 修改或删除 Watchdog |
| `POST` | `/api/reload` | 从磁盘重新读取配置 |
| `POST` | `/api/restart` | 在进程内重启采集运行时 |

配置修改采用"校验 → 备份 → 持久化 → 原子切换"的顺序。成功后现有 Agent 连接会被关闭，Python 客户端约 3 秒后自动重连并获取新的 `monitors`。`/api/restart` 不退出 Go 进程，因此 Docker 和手动运行方式具有一致语义。

常用调用：

```bash
TOKEN='请替换为 ADMIN_TOKEN'

curl http://127.0.0.1:8080/api/health

curl -H "Authorization: Bearer ${TOKEN}" \
  http://127.0.0.1:8080/api/config

curl -X POST http://127.0.0.1:8080/api/servers \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{"username":"s05","name":"node5","type":"kvm","host":"host5","location":"SG","password":"change-me","monthstart":1}'

curl -X DELETE \
  -H "Authorization: Bearer ${TOKEN}" \
  http://127.0.0.1:8080/api/servers/s05
```

请求体最大为 `1 MiB`。AI Agent 可直接导入 `/api/openapi.json`；轻量客户端也可以先读取 `/api/schema`，再根据返回的集合字段调用 CRUD 接口。

## 配置文件

```json
{
  "servers": [
    {
      "username": "s01",
      "name": "node1",
      "type": "kvm",
      "host": "host1",
      "location": "CN",
      "password": "USER_DEFAULT_PASSWORD",
      "monthstart": 1,
      "disabled": false
    }
  ],
  "monitors": [
    {
      "name": "example",
      "host": "https://example.com",
      "interval": 600,
      "type": "https"
    }
  ],
  "sslcerts": [
    {
      "name": "example",
      "domain": "https://example.com",
      "port": 443,
      "interval": 7200,
      "callback": "https://example.net/push?message="
    }
  ],
  "watchdog": [
    {
      "name": "offline warning",
      "rule": "online4=0&online6=0",
      "interval": 600,
      "callback": "https://example.net/push?message="
    }
  ]
}
```

约束：

- `servers.username` 必须唯一。
- `monthstart` 自动限制在 `1-28`。
- `port` 自动限制在 `1-65535`。
- `interval` 最小为 1 秒；Watchdog 中表示通知冷却时间，不是客户端采集间隔。
- 配置写入前会创建 `config.json.bak-*`，最多保留 10 份。
- Docker 单文件 bind mount 无法被 `rename` 覆盖时，服务端会在完成备份后安全地写回原 inode。

使用 Docker 单文件挂载时，配置备份位于容器 `/app/config` 的可写层；如需长期保留历史版本，建议同时在宿主机备份 `server/config.json`。

### Watchdog 表达式

`rule` 由 Go `expr` 引擎执行，并兼容旧版 Exprtk 的常用写法。Go 服务会把字符串外的单个操作符自动转换：

```text
&  -> &&
|  -> ||
=  -> ==
```

例如以下两种写法等价：

```text
cpu>90&load_1>5&username!='s01'
cpu>90 && load_1>5 && username!='s01'
```

字符串值支持中文、Emoji 和其他 Unicode 字符，例如：

```text
username='节点一号'&name='上海节点'&location='中国 🇨🇳'&type='云主机'
```

字段名必须使用系统定义的英文名称。可用字段包括：`username`、`name`、`type`、`host`、`location`、`load_1`、`load_5`、`load_15`、`cpu`、`memory_total`、`memory_used`、`swap_total`、`swap_used`、`hdd_total`、`hdd_used`、`network_rx`、`network_tx`、`network_in`、`network_out`、`last_network_in`、`last_network_out`、`ping_10010`、`ping_189`、`ping_10086`、`time_10010`、`time_189`、`time_10086`、`tcp_count`、`udp_count`、`process_count`、`thread_count`、`io_read`、`io_write`、`online4`、`online6`。

客户端断开 25 秒后仍未重连，服务端才计算离线规则，避免短暂网络波动触发告警。每个节点、每条规则分别记录冷却时间。

### SSL 证书

证书检查使用 Go `crypto/tls`，不再调用外部 `openssl`。服务端记录到期时间、剩余天数和域名匹配状态，并保留原来的 7/3/1 天通知档位与冷却时间。

回调默认校验 HTTPS 证书。仅在必须兼容自签名回调服务时设置：

```bash
INSECURE_CALLBACK_TLS=true
```

## 源码编译和运行

需要 Go `1.25` 或更高版本：

```bash
cd server
go mod download
go test ./...
go build -trimpath -ldflags='-s -w' -o serverstatus .
```

从 `server/` 目录启动：

```bash
ADMIN_TOKEN='请替换为高强度随机字符串' \
./serverstatus \
  --config=config.json \
  --stats=../web/json/stats.json \
  --web-dir=../web \
  --http=:8080 \
  --agent=:35601
```

访问 http://127.0.0.1:8080/。发送 `SIGHUP` 可以重新读取配置：

```bash
kill -HUP "$(pgrep -x serverstatus)"
```

Systemd 示例位于 `service/status-server.service`。一键脚本 `status.sh` 也已切换到 Go 构建，但 Docker 仍是推荐部署方式。

## 构建和测试

```bash
# Go 单元、协议、API、TLS 和回调测试
cd server
go test ./...
go test -race ./...
go vet ./...

# Docker 镜像
cd ..
docker build -f Dockerfile.server -t aoomee/serverstatus-refined:server .
docker build -f Dockerfile.client -t aoomee/serverstatus-refined:client .

# Compose 配置
docker compose -f docker-compose-server.yml config
docker compose -f docker-compose-client.yml config
```

## 从旧服务端迁移

1. 备份原来的 `config.json` 和 `web/json/stats.json`。
2. 原配置结构和客户端账号无需转换。
3. Docker 挂载目标改为 `/app/config/config.json` 和 `/app/data`。
4. 删除旧的 nginx、`manage_api.py`、`sergate` 启动或监督配置。
5. 启动 Go 服务端后检查 `/api/health`，再观察客户端自动重连。

`stats.json` 会按节点的 `name/type/host/location` 恢复月流量基线。修改这些身份字段会被视为新节点。

## 致谢

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus) — 原项目
- [BotoX/ServerStatus](https://github.com/BotoX/ServerStatus)
- [mojeda/ServerStatus](https://github.com/mojeda/ServerStatus)

## License

与原项目一致，详见 [LICENSE](LICENSE)。
