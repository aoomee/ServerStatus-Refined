# ServerStatus Refined

基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的前端美化版本，保留原版全部功能，界面全面重构。

## 预览

- 黑白极简配色，降低视觉干扰
- 全站楷体，字号圆角统一
- 卡片与表格双布局，一键切换
- 亮色 / 暗色主题，跟随系统偏好
- 概览卡片重排，信息层级更清晰
- 告警分三类：离线 / 高载 / 阻断，语义明确
- 详情页仅显示机器 ID，干净利落
- 全局绿 `#16a34a` / 蓝 `#155dfc` 色彩统一

## 与原版对比

| 项目 | 原版 | Refined |
|------|------|---------|
| UI 风格 | 基础功能样式 | 极简黑白 + 楷体 |
| 布局切换 | 无 | 卡片 / 表格一键切换 |
| 主题切换 | 无 | 亮 / 暗一键切换，跟随系统 |
| 概览排序 | 证书风险在前 | 流量在前，信息流更自然 |
| 告警展示 | 离线 / 异常 / 被墙 | 离线 / 高载 / 阻断 |
| 颜色体系 | 多套色值混杂 | 全局统一色彩变量 |

**服务端代码、Agent 协议、Docker 部署、API 接口完全不变，与上游 100% 兼容。**

## 快速开始

### 服务端部署

```bash
git clone https://github.com/aoomee/ServerStatus-Refined.git
cd ServerStatus-Refined

ADMIN_TOKEN='你的密码' docker compose -f docker-compose-server.yml up -d --build
```

启动后访问 `http://你的服务器IP:8080`

### 客户端部署

在需要监控的机器上运行 Agent：

```bash
SERVER=你的服务端IP USER=s01 PASSWORD=USER_DEFAULT_PASSWORD \
docker compose -f docker-compose-client.yml up -d
```

或使用脚本方式：

```bash
wget -qO client-linux.py --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/clients/client-linux.py?ref=master'
nohup python3 client-linux.py SERVER=你的IP USER=s01 PASSWORD=USER_DEFAULT_PASSWORD >/dev/null 2>&1 &
```

更多说明见 [原项目文档](https://github.com/cppla/ServerStatus)。

## 配置

`server/config.json` 预设了 4 个示例节点（`s01` ~ `s04`），部署客户端后即可看到数据。

管理 API 需设置 `ADMIN_TOKEN` 环境变量，用于 Web 配置管理和 Watchdog 告警。

## 项目结构

```
├── server/                 # Go 服务端
│   └── config.json         # 节点、监测、证书、告警配置
├── web/                    # 前端（本版主要修改范围）
│   ├── index.html
│   ├── css/app.css
│   ├── js/app.js
│   └── json/               # 运行时数据目录
├── clients/                # Agent 脚本
├── service/                # Systemd 服务文件
├── Dockerfile.server
├── Dockerfile.client
├── docker-compose-server.yml
└── docker-compose-client.yml
```

## 致谢

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus) — 原项目
- [BotoX/ServerStatus](https://github.com/BotoX/ServerStatus)

## License

与原项目一致，详见 [LICENSE](LICENSE)。
