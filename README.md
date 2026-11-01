# ccmon — Claude Code 运行监控栈管理工具

本项目通过**两条互补通道**监测 Claude Code 的运行状态：

| 通道 | 用的工具 | 监测 Claude 的什么 | 在哪看 |
|---|---|---|---|
| **指标**（聚合趋势） | Claude Code 内置 OpenTelemetry 遥测（env 开关写入 `~/.claude/settings.json`）→ OTel Collector → Prometheus → Grafana | 每次 LLM/API 调用的量化指标：模型、时间、token 四分类（input / output / cache 读 / cache 写）、估算费用、耗时、请求来源 | Grafana 面板 `https://elio.fans/dashboard`（趋势与聚合）；`ccmon events [N]`（最近 N 次调用明细表） |
| **会话日志**（逐轮明细） | Claude Code hooks（UserPromptSubmit / PostToolUse / Stop / SubagentStop / SessionEnd）→ `toollog/ccmon-hook.py` → SQLite → 内置 HTTP 服务 | 会话与每轮命令（起止/耗时/最终输出）、每次工具调用（工具名 + 完整参数 + 截断响应，含子代理）、每次模型推理（model、思考过程、输出文本、请求的工具、token 用量） | `https://elio.fans/sessions`（会话 → 轮次 → 融合时间线三级视图，参数用 JSON 树展示） |

两条通道互相独立、互不依赖：指标通道回答「用了多少、花了多少、快不快」，
日志通道回答「具体每轮做了什么、怎么做的」。两个公网入口均在 `elio.fans` 登录网关之后。

单文件 bash 工具 `ccmon` 把这两套环境的安装、配置、恢复封装为一条命令。
零依赖（jq/curl/docker/python3；缺 docker 可自动 apt 装）。

## 安装

```bash
sudo ln -sf ~/workspace/claude-monitor/ccmon /usr/local/bin/ccmon   # 已装
```

## 命令

| 命令 | 作用 |
|---|---|
| `ccmon setup` | 引导式安装/配置。**幂等**：重复运行安全，回车=保留现值/默认值，可管道喂答案 |
| `ccmon reset-dashboard` | 恢复出厂仪表盘模板：默认否确认，先备份本地 JSON 再以 `overwrite` 覆盖导入（旧版可在 Grafana Versions 历史找回） |
| `ccmon restart` | 机器重启后恢复：docker → 容器 → nginx → 会话日志服务 → 全链路健康检查（等 Grafana 就绪再检查） |
| `ccmon status` | 只读体检：容器/各端点/公网入口/遥测开关/事件数/会话日志/存储用量 |
| `ccmon events [N]` | 最近 N 次 API 调用明细表（时间/模型/token 四分类/费用/时长/来源），默认 10 |
| `ccmon start` / `ccmon stop` | 启停采集栈与会话日志服务（数据保留） |

## `setup` 引导的参数（括号内为默认值）

安装目录(`~/cc-otel`)、公网域名(`elio.fans`)、子路径(`/dashboard`)、TSDB 本地路径(`~/tsdb`)、
TSDB 卷名 / Grafana 卷名（compose `name:` 钉死，迁移不换卷）、Grafana admin 密码（留空随机生成，
已有则默认保留）、三个镜像版本、host 绑定地址(`127.0.0.1`)、四个端口、匿名角色(`Viewer`)、
HTTPS 与证书路径、是否改 Claude Code settings.json、是否配 nginx；
会话日志：是否启用(`y`)、页面端口(`7780`)、公网子路径(`/sessions`)、保留天数(`30`，0=永久)、
响应截断字符数(`65536`)。

setup 自动完成：目录创建与属主修正（data→10001、tsdb→65534）、.env/compose/collector/
prometheus 写入、仪表盘模板**仅缺失时写入**（已存在不覆盖）、compose up、Grafana 数据源创建/更新
（PUT/POST，幂等）+仪表盘**仅缺失时导入**（Grafana 里已存在的仪表盘定制不覆盖）、凭据文件(600)、
nginx location 追加（已有 server 块则插到块尾+备份+`nginx -t` 失败自动回滚；无则生成独立 conf）、
settings.json env 合并（jq，先备份，已就绪则跳过）；
会话日志（启用时）：`<INSTALL_DIR>/toollog/` 三件套写入（hook 脚本/HTTP 服务/页面）、SQLite
建库建表（WAL）、settings.json hooks **追加式**合并（jq；cc-bridge/botmux 等既有 hooks 原样保留，
幂等，先备份）、systemd --user 服务 `ccmon-sessions.service`（enable --now，Linger 随开机自启）、
nginx `/sessions` 入口追加。此前已启用而本次答 n：可选摘除 hooks + 停用服务（数据库保留）。

## 会话/工具调用日志（toollog）

`~/.claude/settings.json` 注入五个 hook（UserPromptSubmit / PostToolUse / Stop / SubagentStop /
SessionEnd，均 `timeout:10`，追加不覆盖，matcher 省略=匹配所有工具），事件由
`toollog/ccmon-hook.py`（python3 stdlib，任何异常静默退出，
绝不打断 Claude）写入 `toollog/ccmon-sessions.db`（SQLite WAL；`sessions` / `turns` / `tool_calls`
/ `llm_calls` 四表；tool_input 全量、tool_response 截断；子代理调用按 transcript 路径差异探测）。

**LLM 推理调用**（`llm_calls`）：PostToolUse/Stop/SubagentStop/SessionEnd 时从 transcript 尾部
"收割" assistant 消息——流式分片按 `message.id` 合并成一次调用，含 model、stop_reason、
thinking、输出文本、请求的工具名、input/output/cache token；按 (session_id, mid) 去重幂等，
只收录能归属到轮次的消息（防 -c/--resume 把旧会话历史误记到新会话）。Stop 的退避重试与
SessionEnd 兜底覆盖"最终响应晚于 Stop 落盘"的竞态。

`toollog/ccmon-sessions-server.py`（127.0.0.1:7780，只读）+ `ccmon-sessions.html` 提供三级视图：
会话列表 → 每轮用户命令卡片（开始/结束/耗时/调用数/推理数/最终输出）→ **融合时间线**（工具调用
与 ✦ 模型推理按时间穿插；参数/响应用 VSCode 风格 JSON 树展示：key/类型着色、可折叠、大节点
懒展开、长字符串可展开；推理行含思考过程与 token 用量）。最终输出在 Stop 时从 transcript 尾部
提取（带时间下界过滤 + 小退避重试 + 下一轮补账，规避 hook 与 transcript 落盘的竞态）。
服务每天按保留天数清理旧行并 VACUUM。

## 状态与文件

- `<INSTALL_DIR>/ccmon.state` — 工具状态（600），restart/status 依据
- `<INSTALL_DIR>/.env` — 全部参数 + Grafana 密码（compose 插值用，600）
- `<INSTALL_DIR>/grafana-admin-credentials.txt` — admin 密码（600）
- `<INSTALL_DIR>/toollog/`（700）— `ccmon-hook.py` / `ccmon-sessions-server.py` /
  `ccmon-sessions.html` / `ccmon-sessions.db`（600）+ `.err.log`（hook 旁路错误日志）
- `~/.config/systemd/user/ccmon-sessions.service` — 会话日志服务 unit
- 每次修改 settings.json / nginx conf 前自动留 `.bak-ccmon-*` 备份
- `ccmon reset-dashboard` 重建模板前留 `grafana-dashboard.json.bak-ccmon-<时间戳>`（同目录）

## 机器重启后的行为

容器带 `restart: unless-stopped`，随 docker 自启（docker.service 已 enable）；
`ccmon-sessions.service` 为 systemd --user 单元（Linger 已开，随开机自启）；
`ccmon restart` 作为确定性恢复入口依然可用（幂等：up -d + nginx reload + 会话日志服务 + 健康检查）。

## 已知边界

- Grafana admin 密码仅在数据卷**首次初始化**时生效；改密码需
  `docker exec -it cc-otel-grafana-1 grafana cli admin reset-admin-password <新密码>` 并同步改 .env
- 安装目录设定后不建议再改（state 的发现路径以首次为准）；确要迁移，手工搬 `~/cc-otel` 后重跑 setup 逐项确认
- 明细文件 `data/cc-events.jsonl` 为 0600/UID 10001，`ccmon events` 内部已用 sudo 读取
- collector 的 prometheus exporter 对 5 分钟无更新的序列会过期（指标显示"暂无"属正常，明细不受影响）
- 域名需已解析到本机；无证书时选 HTTP-only 会生成 80 端口独立 conf
- setup 不覆盖已有仪表盘（本地 `grafana-dashboard.json` 与 Grafana 侧均保留）；恢复出厂模板需显式
  运行 `ccmon reset-dashboard`。本地 `.bak-ccmon-*` 只是重建前的模板文件快照，不是 Grafana 当前
  线上版本的导出，远端旧版恢复以 Grafana Versions 历史为准
- 会话日志公网入口（`/sessions`）依赖 **elio.fans 全站登录网关**（nginx `auth_request`，ccmon 配置时
  只加 location 不加认证）；tool_input/响应里可能含代码、路径等敏感内容，网关账号勿外传，
  也可改答仅本机（SSH 隧道 `ssh -L 7780:127.0.0.1:7780`）访问
- hooks 命令里写死了 toollog 绝对路径；若迁移安装目录，需重跑 setup 重新注入 hooks
- 「进行中/已结束」为启发式（15 分钟内无活动视为已结束）；会话真实结束依赖 Stop/SessionEnd 事件
- 已开的 Claude Code 会话不会加载新 hooks，从新会话起才被记录
