# ServerStatus Refined

> 基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的前端美化版，保留原版全部功能与部署方式，UI 全面升级。

**Refined 版仅修改前端展示层（`web/` 目录），服务端代码、Agent 协议、Docker 构建、API 接口均与上游 100% 兼容。**

---

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

---

## 🚀 服务端

### 方式一：Docker Run

```bash
wget -qO ~/serverstatus-config.json \
  'https://raw.githubusercontent.com/aoomee/ServerStatus-Refined/main/server/config.json'
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
services:
  serverstatus:
    image: ghcr.io/aoomee/serverstatus-refined:latest
    container_name: serverstatus-server
    restart: unless-stopped
    ports:
      - "16888:80"       # WebUI 端口
      - "35601:35601"    # Agent 上报端口
    volumes:
      - ./data:/app/data
    environment:
      - ADMIN_TOKEN=VpsTest2026!
```

```bash
git clone https://github.com/aoomee/ServerStatus-Refined.git
cd ServerStatus-Refined
docker compose up -d
```

### 方式三：二进制 / 源码编译

需要 Go `1.25+`：

```bash
cd server
go build -trimpath -ldflags='-s -w' -o serverstatus .
```

启动：

```bash
ADMIN_TOKEN='your-strong-token' \
./serverstatus \
  --config=config.json \
  --stats=../web/json/stats.json \
  --web-dir=../web \
  --http=:16888 \
  --agent=:35601
```

启动后访问：

| 地址 | 说明 |
| --- | --- |
| `http://127.0.0.1:16888/` | WebUI |
| `http://127.0.0.1:16888/api/health` | 健康检查 |
| `http://127.0.0.1:16888/api/schema` | API 描述 |
| `35601/tcp` | Agent 上报端口 |

`ADMIN_TOKEN` 不设置时，监控页面仍可读取，管理 API 返回 `503`，配置页不可修改。

---

## 📡 客户端

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

```yaml
services:
  serverstatus-client:
    image: cppla/serverstatus:client
    container_name: serverstatus-client
    restart: unless-stopped
    network_mode: host
    pid: host
    environment:
      - SERVER=127.0.0.1                # 改成服务端 IP
      - USER=s01                        # 改成配置的用户名
      - PASSWORD=USER_DEFAULT_PASSWORD  # 改成配置的密码
```

### 方式三：Shell 脚本

```bash
wget -qO client-linux.py --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/clients/client-linux.py?ref=master'
nohup python3 client-linux.py SERVER=127.0.0.1 USER=s01 PASSWORD=USER_DEFAULT_PASSWORD >/dev/null 2>&1 &
```

> `USER` 是常见的宿主机环境变量名，Compose 可能误解析为系统 `$USER`。推荐显式传递 `USER=s01`。

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `SERVER` | `127.0.0.1` | 服务端地址 |
| `USER` | `s01` | 客户端用户名，须匹配服务端配置 |
| `PORT` | `35601` | Agent 上报端口 |
| `PASSWORD` | `USER_DEFAULT_PASSWORD` | 客户端密码，须匹配服务端配置 |
| `INTERVAL` | `1` | 上报间隔（秒） |
| `PROBEPORT` | `80` | 三网探测端口 |
| `PROBE_PROTOCOL_PREFER` | `ipv4` | 探测协议偏好 |
| `CU` / `CT` / `CM` | cloudcpp.com | 三网探测地址 |
| `CLIENT` | `psutil` | 客户端实现，可选 `psutil`、`linux` |

---

## 🔧 管理 API

```bash
TOKEN='your ADMIN_TOKEN'

# 查看节点
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

更多接口见 [原项目 README](https://github.com/cppla/ServerStatus#http-管理-api)。

---

## 致谢

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus) — 原项目
- [BotoX/ServerStatus](https://github.com/BotoX/ServerStatus)
- [mojeda/ServerStatus](https://github.com/mojeda/ServerStatus)

## License

与原项目一致，详见 [LICENSE](LICENSE)。
