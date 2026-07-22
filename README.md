# ServerStatus 美化版

基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的轻量服务器探针与云监控面板，保留原版全部功能，前端 UI 全面优化。

在线演示：https://tz.cloudcpp.com（原版）

## 效果预览

- 黑白极简风格，对齐哪吒探针配色体系
- 全站楷体，统一字号与圆角
- 卡片/表格双视图一键切换
- 亮色/暗色主题切换
- 概览卡片：在线主机、本月上行、本月下行、证书风险、活跃告警
- 告警三分类：离线 / 高载 / 阻断
- 详情页仅显示机器 ID，简约克制
- 全局绿 `#16a34a` / 蓝 `#155dfc` 色彩统一

## 与原版的区别

| 项目 | 原版 | 美化版 |
|------|------|--------|
| UI 风格 | 基础功能样式 | 哪吒探针风格，黑白+楷体 |
| 布局切换 | 无 | 卡片/表格一键切换 |
| 主题切换 | 无（或手动 CSS） | 亮/暗一键切换，跟随系统 |
| 概览排序 | 证书风险在第二 | 上行第二，下行第三 |
| 告警展示 | 离线/异常/被墙 | 离线/高载/阻断，语义明确 |
| 颜色体系 | 多套杂色 | 全局统一色彩变量 |

**服务端代码、Agent 协议、Docker 部署、API 接口完全不变，与上游 100% 兼容。**

## 快速开始

### 服务端（Docker）

```bash
# 克隆项目
git clone https://github.com/你的用户名/ServerStatus.git
cd ServerStatus

# 启动服务
ADMIN_TOKEN='你的密码' docker compose -f docker-compose-server.yml up -d --build
```

启动后访问 `http://你的服务器IP:8080`

### 客户端部署

在需要监控的机器上运行 Agent：

```bash
# Docker 方式
SERVER=你的服务端IP USER=s01 PASSWORD=USER_DEFAULT_PASSWORD \
docker compose -f docker-compose-client.yml up -d

# 脚本方式
wget -qO client-linux.py --header='Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/cppla/ServerStatus/contents/clients/client-linux.py?ref=master'
nohup python3 client-linux.py SERVER=你的IP USER=s01 PASSWORD=USER_DEFAULT_PASSWORD >/dev/null 2>&1 &
```

更多部署选项见 [原项目文档](https://github.com/cppla/ServerStatus)。

## 配置说明

`server/config.json` 中预设了 4 个示例节点（`s01` ~ `s04`），部署客户端后即可看到数据。

管理 API 需要设置 `ADMIN_TOKEN` 环境变量，用于 Web 配置管理和 Watchdog 告警。

## 项目结构

```
├── server/          # Go 服务端源码
│   └── config.json  # 节点/监测/证书/告警配置
├── web/             # 前端（本文修改范围）
│   ├── index.html   # 页面模板
│   ├── css/app.css  # 样式（美化）
│   ├── js/app.js    # 逻辑（美化）
│   └── json/        # 运行时数据目录
├── clients/         # Agent 客户端脚本
├── service/         # Systemd 服务文件
├── Dockerfile.server
├── Dockerfile.client
├── docker-compose-server.yml
└── docker-compose-client.yml
```

## 致谢

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus) — 原项目
- [BotoX/ServerStatus](https://github.com/BotoX/ServerStatus)
- [nezhahq/nezha](https://github.com/nezhahq/nezha) — 哪吒探针（配色参考）

## License

与原项目一致，详见 [LICENSE](LICENSE)。
