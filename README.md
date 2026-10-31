# ccmon — Claude Code OTel 监控栈管理工具

单文件 bash 工具，把「Claude Code 遥测 → OTel Collector → Prometheus → Grafana（nginx 公网子路径）」
整套环境的安装、配置、恢复封装为一条命令。零依赖（jq/curl/docker 缺 docker 可自动 apt 装）。

## 安装

```bash
sudo ln -sf ~/workspace/claude-monitor/ccmon /usr/local/bin/ccmon   # 已装
```

## 命令

| 命令 | 作用 |
|---|---|
| `ccmon setup` | 引导式安装/配置。**幂等**：重复运行安全，回车=保留现值/默认值，可管道喂答案 |
| `ccmon restart` | 机器重启后恢复：docker → 容器 → nginx → 全链路健康检查（等 Grafana 就绪再检查） |
| `ccmon status` | 只读体检：容器/各端点/公网入口/遥测开关/事件数/存储用量 |
| `ccmon events [N]` | 最近 N 次 API 调用明细表（时间/模型/token 四分类/费用/时长/来源），默认 10 |
| `ccmon start` / `ccmon stop` | 启停采集栈（数据保留） |

## `setup` 引导的参数（括号内为默认值）

安装目录(`~/cc-otel`)、公网域名(`elio.fans`)、子路径(`/dashboard`)、TSDB 本地路径(`~/tsdb`)、
TSDB 卷名 / Grafana 卷名（compose `name:` 钉死，迁移不换卷）、Grafana admin 密码（留空随机生成，
已有则默认保留）、三个镜像版本、host 绑定地址(`127.0.0.1`)、四个端口、匿名角色(`Viewer`)、
HTTPS 与证书路径、是否改 Claude Code settings.json、是否配 nginx。

setup 自动完成：目录创建与属主修正（data→10001、tsdb→65534）、.env/compose/collector/
prometheus/仪表盘模板写入、compose up、Grafana 数据源+仪表盘导入（PUT 幂等）、凭据文件(600)、
nginx location 追加（已有 server 块则插到块尾+备份+`nginx -t` 失败自动回滚；无则生成独立 conf）、
settings.json env 合并（jq，先备份，已就绪则跳过）。

## 状态与文件

- `<INSTALL_DIR>/ccmon.state` — 工具状态（600），restart/status 依据
- `<INSTALL_DIR>/.env` — 全部参数 + Grafana 密码（compose 插值用，600）
- `<INSTALL_DIR>/grafana-admin-credentials.txt` — admin 密码（600）
- 每次修改 settings.json / nginx conf 前自动留 `.bak-ccmon-*` 备份

## 机器重启后的行为

容器带 `restart: unless-stopped`，随 docker 自启（docker.service 已 enable）；
`ccmon restart` 作为确定性恢复入口依然可用（幂等：up -d + nginx reload + 12 项健康检查）。

## 已知边界

- Grafana admin 密码仅在数据卷**首次初始化**时生效；改密码需
  `docker exec -it cc-otel-grafana-1 grafana cli admin reset-admin-password <新密码>` 并同步改 .env
- 安装目录设定后不建议再改（state 的发现路径以首次为准）；确要迁移，手工搬 `~/cc-otel` 后重跑 setup 逐项确认
- 明细文件 `data/cc-events.jsonl` 为 0600/UID 10001，`ccmon events` 内部已用 sudo 读取
- collector 的 prometheus exporter 对 5 分钟无更新的序列会过期（指标显示"暂无"属正常，明细不受影响）
- 域名需已解析到本机；无证书时选 HTTP-only 会生成 80 端口独立 conf
