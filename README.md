# ServerStatus Refined

> 基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的前端美化版，保留原版全部功能与部署方式，UI 全面升级。

**Refined 版仅修改前端展示层（`web/` 目录），服务端代码、Agent 协议、Docker 构建、API 接口均与上游 100% 兼容。**

- 黑白极简配色，全站楷体，字号圆角统一
- 卡片 / 表格双布局一键切换
- 亮色 / 暗色主题，跟随系统偏好
- 概览卡片重排，告警三分类：离线 / 高载 / 阻断
- 详情页简化，色彩体系统一（绿 `#16a34a` / 蓝 `#155dfc`）

---

## 一、服务端
### 方式一：Docker Run

```bash
wget -qO ~/serverstatus-config.json \
  --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/server/config.json?ref=master'
mkdir -p ~/serverstatus-data

docker run -d --restart=always --name=serverstatus-server \
  -e ADMIN_TOKEN='your-strong-token' \
  -v ~/serverstatus-config.json:/app/config/config.json \
  -v ~/serverstatus-data:/app/data \
  -p 16888:80 -p 35601:35601 \
  ghcr.io/aoomee/serverstatus-refined:latest
```

### 方式二：Docker Compose

`docker-compose.yml`：

```yaml
version: '3'

services:
  serverstatus:
    image: ghcr.io/aoomee/serverstatus-refined:latest
    build:
      context: .
      dockerfile: Dockerfile.server
    container_name: serverstatus-server
    restart: unless-stopped
    ports:
      - "16888:80"       # 外网端口，可以改
      - "35601:35601"    # Agent 上报端口，不能改
    volumes:
      - ./server/config.json:/app/config/config.json
      - ./web/json:/app/data
    environment:
      - ADMIN_TOKEN=your-password-here   # 把这里改成你的密码
```

```bash
git clone https://github.com/aoomee/ServerStatus-Refined.git
cd ServerStatus-Refined
docker compose up -d
```

客户端部署见下方 [二、客户端](#二客户端)。

### 方式三：二进制/源码编译

需要 Go `1.25` 或更高版本：

```bash
cd server
go mod download
go build -trimpath -ldflags='-s -w' -o serverstatus .
```

从 `server/` 目录启动：

```bash
ADMIN_TOKEN='your-strong-token' \
./serverstatus \
  --config=config.json \
  --stats=../web/json/stats.json \
  --web-dir=../web \
  --http=:16888 \
  --agent=:35601
```

访问 http://127.0.0.1:16888/。Systemd 示例位于 `service/status-server.service`。


启动后访问：

- WebUI：http://127.0.0.1:8080/
- 健康检查：http://127.0.0.1:8080/api/health
- API 描述：http://127.0.0.1:8080/api/schema
- OpenAPI 3.1：http://127.0.0.1:8080/api/openapi.json
- 客户端上报端口：`35601/tcp`

`ADMIN_TOKEN` 不设置时，监控页面仍可读取，管理 API 返回 `503`，WebUI 的「配置」页不能修改数据。

## 二、客户端

### 方式一：Docker Run

```bash
docker run -d --restart=always --name=serverstatus-client \
  --network=host --pid=host \
  -e SERVER=127.0.0.1 \
  -e USER=s01 \
  -e PASSWORD=USER_DEFAULT_PASSWORD \
  cppla/serverstatus:client
```

### 方式二：Docker Compose

`docker-compose-client.yml`：

```yaml
version: '3'

services:
  serverstatus-client:
    image: cppla/serverstatus:client
    container_name: serverstatus-client
    restart: unless-stopped
    network_mode: host
    pid: host
    environment:
      - SERVER=127.0.0.1                # 改成服务端 IP
      - USER=s01                        # 改成服务端配置的用户名
      - PASSWORD=USER_DEFAULT_PASSWORD  # 改成服务端配置的密码
```

```bash
SERVER=127.0.0.1 USER=s01 PASSWORD=USER_DEFAULT_PASSWORD \
docker compose -f docker-compose-client.yml up -d --force-recreate
```

### 方式三：Shell 脚本

```bash
wget -qO client-linux.py --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/clients/client-linux.py?ref=master'
nohup python3 client-linux.py SERVER=127.0.0.1 USER=s01 PASSWORD=USER_DEFAULT_PASSWORD >/dev/null 2>&1 &
```

`USER` 是常见的宿主机环境变量名。如果没有显式传递或传递方式错误，Compose 可能会把系统中的 `$USER` 解析成本机用户名，而不是默认的 `s01`。推荐优先级：

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

配置修改采用「校验 → 备份 → 持久化 → 原子切换」的顺序。成功后现有 Agent 连接会被关闭，Python 客户端约 3 秒后自动重连并获取新的 `monitors`。`/api/restart` 不退出 Go 进程，因此 Docker 和手动运行方式具有一致语义。

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

## Refined 版 UI 改动说明

本版仅修改 `web/` 目录下的三个文件（`index.html`、`css/app.css`、`js/app.js`），服务端逻辑和构建流程不变：

- **配色**：全局黑白极简风格，绿色 `#16a34a` / 蓝色 `#155dfc` 统一
- **字体**：全站楷体
- **布局**：右上角一键切换卡片视图 / 表格视图
- **主题**：右上角一键切换亮色 / 暗色，自动跟随系统
- **概览**：卡片顺序调整为 在线主机 → 本月上行 → 本月下行 → 证书风险 → 活跃告警
- **告警**：三类分别为 离线（Agent 失联）/ 高载（CPU / 内存 / 硬盘超限）/ 阻断（在线但三网全丢包）
- **详情**：点击主机卡片仅显示机器 ID，页面更克制

如需恢复原版 UI，直接替换 `web/` 目录为原项目文件即可。

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
docker build -f Dockerfile.server -t serverstatus:server .
docker build -f Dockerfile.client -t serverstatus:client .

# Compose 配置
docker compose -f docker-compose.yml config
docker compose -f docker-compose-client.yml config
```

CI 还会检查 Go 格式、Python 客户端、Shell 脚本、WebUI JavaScript、服务端/客户端 Compose 文件和两个 Docker 镜像。

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
