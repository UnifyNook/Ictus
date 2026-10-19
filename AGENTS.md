## 项目概览

ARTEX 是 AI 自主渗透测试系统：Go 单体后端（`net/http` + PostgreSQL）内嵌 Next.js 静态前端，多 agent 探索能力由 [Norma](https://github.com/UnifyNook/Norma) SDK（agentcore / tool / permission / harness / memory / transcript）提供。核心是**双图架构**：全局共享的资产图（companies / assets / scope，跨任务真值库）与每任务独立的探索图（goal / intent / fact / finding / hint 血缘链）经 anchors 相连；planner 事件驱动派意图、worker 领意图执行并写回，形成探索闭环。

## 常用命令

- 开发：`./dev.sh`（后端 :8787 + 流量代理 :8788 + 前端 :5173）
- 后端测试：`go test ./...`（集成测试需可达 PostgreSQL，DSN 取 `ARTEX_PG_DSN` 或 config.json，不可用自动 Skip）
- 前端：`cd web && npm run dev`；静态导出 `npm run build:static`；检查 `npm run check`（Biome）
- 完整构建（内嵌前端单二进制）：`./build.sh`（等价于 `build:static` → 同步 `web/out/` 到 `server/webui/dist/` → `go build -tags embedui ./cmd/artex`）

## 全局要点

- **前端改动要进入单二进制**，必须重新静态导出并同步 `server/webui/dist/`，再带 `-tags embedui` 编译；不带该 tag 时 webui 路由返回 404 提示页。
- **数据库迁移是「重启即迁移」**：`db/schema.sql` 随 go:embed 每次启动幂等重跑，改表结构必须保持幂等（细则见 db/AGENTS.md）。
- `docs/`（跨模块设计文档）与 `参考项目/`（上游源码比对用）均在 .gitignore 中，不入库；`参考项目/` 除非用户明确要求，不要修改，复制代码前必须适配本项目 API 与配置。
- **涉及到实际运行功能的提交需同步写入CHANGELOG.md**：在commit的时候，当提交的更改涉及项目功能更改的时候请同步更新CHANGELOG.md,由于此文件非常大，请你只阅读头部200行

## ⚠ 必读：分层规则

**修改某个目录下的代码前，必须先阅读该目录下的 `AGENTS.md`（如有）。** 根文件只包含全局规则和概览，子目录的 `AGENTS.md` 承载该包特有的约定、模式和检查清单（如数据库幂等迁移、通知渠道无状态约束、前端静态导出限制）。跳过子目录规则会违反项目约定。

### 子规则索引

- [server/AGENTS.md](server/AGENTS.md)：HTTP API、Manager/Engine 生命周期、SSE、JWT 鉴权、webui 嵌入
- [agent/AGENTS.md](agent/AGENTS.md)：planner / worker / mainagent、双图工具集、prompt 与工具目录机制
- [db/AGENTS.md](db/AGENTS.md)：双图 schema、幂等迁移、Store 组织、事务与 advisory lock
- [web/AGENTS.md](web/AGENTS.md)：静态导出约束、API 客户端、类型与 Biome 约定
- [traffic/AGENTS.md](traffic/AGENTS.md)：MITM 记录代理、SQLite 本地索引与 blob 存储
- [notify/AGENTS.md](notify/AGENTS.md)：通知渠道适配层（无状态叶子包）
- [skills/AGENTS.md](skills/AGENTS.md)：agent 技能包目录结构与加载方式

### 支撑包速览（无专属 AGENTS.md，改动前直接读包内代码）

| 包             | 职责                                                                                                                              |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `cmd/artex`    | 进程入口：flag 解析 → selfupdate.Bootstrap → Manager/Server 装配；退出码 75 是与 start.sh 约定的换装重启信号                      |
| `config`       | 配置加载：路径取 `ARTEX_CONFIG` > CWD > 二进制旁；DSN 取 `ARTEX_PG_DSN` > `database.dsn` > 字段拼装，缺失即启动失败（无默认兜底） |
| `guard`        | Norma 工具 hook 入口：内存审计环 + 成功/失败归因，拦截判定委托 intercept                                                          |
| `intercept`    | 工具拦截规则引擎：allow / deny / ask 三态，ask 进 pending 队列等审批，可注入 LLM 兜底裁判                                         |
| `enrich`       | 非 AI 异步补全：DNS 解析与 HTTP 探测经记录代理写回资产图                                                                          |
| `mcphttp`      | 远程 MCP 客户端：Streamable HTTP 与 legacy SSE，适配为 Norma 工具                                                                 |
| `evidence`     | 漏洞证据内容寻址存储（sha256 + staging 原子落盘，区别于可清理的 traffic）                                                         |
| `llmpool`      | LLM 失败转移链：多 profile 优先级、熔断退避、安全窗口内切换                                                                       |
| `llmrec`       | LLM 调用记录：usage 恒写 PG，完整 wire body 按开关记录                                                                            |
| `sidequestion` | `/btw` 旁路提问：上下文快照 + 单次回答，不建 agent 会话                                                                           |
| `report`       | 报告导出：任务级 Markdown、单漏洞 zip、CSV                                                                                        |
| `selfupdate`   | 一键更新：下载校验、暂存换装、连崩自动回滚（三次启动协议）                                                                        |
