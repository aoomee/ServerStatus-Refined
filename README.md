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

- WebUI：http://127.0.0.1:16888/
- 健康检查：http://127.0.0.1:16888/api/health
- API 描述：http://127.0.0.1:16888/api/schema
- OpenAPI 3.1：http://127.0.0.1:16888/api/openapi.json
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

## 管理 API 与配置

服务端通过 HTTP API 管理节点、监测、证书和告警规则。配置文件示例及 Watchdog 表达式语法详见 [原项目文档](https://github.com/cppla/ServerStatus)。

常用 API 调用：

```bash
TOKEN='你的 ADMIN_TOKEN'

# 查看所有节点
curl -H "Authorization: Bearer ${TOKEN}" http://127.0.0.1:16888/api/servers

# 新增节点
curl -X POST http://127.0.0.1:16888/api/servers \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{"username":"s01","name":"我的VPS","type":"kvm","host":"vps01","location":"US","password":"pw123","monthstart":1}'

# 删除节点
curl -X DELETE -H "Authorization: Bearer ${TOKEN}" \
  http://127.0.0.1:16888/api/servers/s01
```

更多接口及服务端参数说明见 [原项目 README](https://github.com/cppla/ServerStatus#http-%E7%AE%A1%E7%90%86-api)。

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

## 致谢

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus) — 原项目
- [BotoX/ServerStatus](https://github.com/BotoX/ServerStatus)
- [mojeda/ServerStatus](https://github.com/mojeda/ServerStatus)

## License

与原项目一致，详见 [LICENSE](LICENSE)。
