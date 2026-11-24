# Smart-Client 智能工作台

聚合「大模型 + 智能体 + 工作流编排 + 知识库 + 技能 + MCP 服务」的一体化 AI 应用平台。

用户可以在一个工作台里接入多家模型供应商，用画布编排工作流 / Chatflow，给智能体挂上知识库、技能与 MCP 工具，
再通过 Web 页面、API、Webhook、MCP 等入口把应用发布出去，并在监控页看到真实的调用量、耗时与费用。

---

## 一、功能总览

| 模块 | 能力 |
|---|---|
| 对话 | 多会话流式对话（SSE）、提示词增强、消息反馈、停止生成、引用来源展示 |
| 工作室（应用） | 工作流 / Chatflow 应用列表，创建空白应用，编排 → 测试运行 → 发布 |
| 工作流编排 | 画布编排节点：开始 / 知识检索 / LLM / Agent 协作 / ANSWER / 条件分支 / HTTP 请求 / 代码执行 / MCP 工具 |
| 智能体 | 智能体配置（提示词、模型、温度、历史条数、技能、知识库、MCP 工具）、草稿 / 发布 / 下线、导入导出 |
| 智能体生成（Agent Forge） | 一句话需求 → 澄清问答 → 方案生成 → 确认 → 发布为智能体，全流程 SSE 推送 |
| 知识库 | 创建向导、流水线编排、多格式文档入库（PDF/Word/Excel/TXT/MD/CSV/JSON/HTML）、分段与向量化、召回测试、外部 API |
| 技能 | 技能列表 / 详情、SKILL.md 描述与文件树管理、zip 包导入、测试运行、发布到能力广场 |
| MCP 服务 | 接入 MCP 服务（http / sse / stdio）、工具清单同步与按需启用、单工具调试、调用日志 |
| 模型接入 | 模型 CRUD、供应商凭据、每千 token 单价、启用 / 发布 / 下线 |
| 模型供应商市场 | 17 家预置供应商：市场浏览 → 安装 → 填凭证 → 探测校验 → 拉取模型 → 设为默认 → 发布 |
| 集成 | 触发器（定时 / Webhook）、Agent 策略（护栏 / 限流 / 模型路由）、扩展、自定义端点、工作空间成员 |
| 能力广场 / 我的能力 | 聚合已发布的智能体、技能、知识库、MCP 服务 |
| 运维监控 | 应用日志（会话级明细）、监测页（调用量、失败率、token、费用）、标注、每日统计汇总 |
| 插件市场 | 同步 Dify 插件市场条目、白名单管理、安装映射到原生能力 |
| 其他 | 登录注册、审计日志、字体缩放、安全传输（可开关）、Electron 桌面壳 |

---

## 二、技术栈

**后端**
- Java 21 / Spring Boot 3.3.5
- MyBatis-Plus 3.5.6（逻辑删除、自动填充时间）
- MySQL 8 + Redis（登录态与限流计数）
- Sa-Token 1.38（JWT 模式，请求头 `satoken`）
- WebFlux WebClient（调模型 / MCP，连接池复用）
- PDFBox 2.0.31 + POI-OOXML 5.2.5（文档解析）
- 统一响应体 `{ code, message, data }`

**前端**
- Vue 3.4 + TypeScript + Vite 5
- Pinia（状态）、Vue Router 4（Hash 路由）
- TailwindCSS 3 + SCSS（自定义 design token）
- Element Plus（图标）
- Electron 45（可选打包桌面端）

---

## 三、快速开始

前置：**JDK 21、Maven、Node 18+、MySQL 8、Redis**。

**无需设置任何环境变量**：所有配置都有默认值；密钥类（JWT 签名、API Key 加密密钥）留空时会自动生成并持久化到
数据库 `t_system_secret`（数据库不可用时落盘到 `./data/`）；数据库不存在时由 `createDatabaseIfNotExist=true` 自动创建。

### 方式一：一键启动（推荐）

```bash
./start.sh            # 启动前后端，两端就绪后打印访问地址
./start.sh stop       # 停止
./start.sh restart    # 重启
./start.sh status     # 查看运行状态
./start.sh logs       # 实时跟踪前后端日志（Ctrl+C 只退出跟踪）
```

脚本会自动完成：环境预检（Java/Maven/Node/MySQL/Redis）、端口冲突检查、前端依赖安装（首次）、
后端健康等待、启动失败回滚与日志定位提示。换端口用 `BACKEND_PORT=8081 ./start.sh`。

### 方式二：手动启动

```bash
# 后端：http://localhost:8080
cd backend && mvn spring-boot:run

# 前端：http://localhost:5173
cd frontend && npm install && npm run dev
```

### 访问与账号

- 前端地址：<http://localhost:5173>，后端健康检查：`GET /api/health`
- 首次使用在登录页点「立即注册」注册账号即可（注册后自动登录）
- 前端通过 `frontend/.env` 的 `VITE_API_BASE` 指向后端地址

### 默认连接配置

| 项 | 默认值 | 覆盖变量 |
|---|---|---|
| MySQL | `localhost:3306/smart-client-db`，root/root | `MYSQL_HOST` `MYSQL_PORT` `MYSQL_DATABASE` `MYSQL_USER` `MYSQL_PASSWORD` |
| Redis | `localhost:6379`，无密码 | `REDIS_HOST` `REDIS_PORT` `REDIS_PASSWORD` |
| 后端端口 | 8080 | `APP_PORT`（刻意不用 `SERVER_PORT`，避免被容器/IDE 意外注入） |

启动时 `spring.sql.init` 自动执行：
- `db/schema.sql` — 建全部业务表（含 `t_system_secret` 密钥表）
- `db/providers.sql` — 灌入 17 家供应商种子（`INSERT IGNORE` + 唯一键，幂等）

---

## 四、目录结构

```
Smart-Client/
├── backend/                      # Spring Boot 后端
│   └── src/main/java/xz/sc/smartclient/
│       ├── common/               # 统一响应、异常、限流、审计、安全编解码与迁移器
│       ├── config/               # Sa-Token / MyBatis-Plus / 安全传输 / 插件市场配置
│       ├── controller/           # 各模块 REST 接口
│       ├── entity/ mapper/       # 实体与 MyBatis-Plus Mapper
│       ├── service/              # 业务服务（含知识库入库、MCP 管理器等）
│       ├── llm/                  # AgentRuntime、网关工厂、RAG、Embedding、MCP 客户端
│       │   └── provider/         # 内置供应商适配（qwen/hunyuan/openai/动态/本地兜底）
│       ├── providers/            # 模型供应商模块（市场/凭证/校验/拉模型/协议适配）
│       ├── workflow/             # 编排执行器 SPI 与各节点实现
│       └── ops/                  # 策略引擎、统计汇总、触发器调度、插件同步
├── frontend/                     # Vue 3 前端
│   └── src/
│       ├── api/                  # 各模块接口封装
│       ├── components/           # 画布、面板、抽屉等复用组件
│       ├── views/                # 页面（agent/app/kb/skill/tools/providers/...）
│       ├── stores/               # Pinia
│       ├── styles/               # 全局样式与 design token
│       └── router/ config/       # 路由与侧栏菜单配置
├── doc/                          # 设计文档与 demo 数据
├── scripts/                      # 辅助脚本
├── start.sh                      # 一键启动脚本
└── README.md
```

---

## 五、核心设计

### 1. 智能体运行时统一装配

`AgentRuntime.prepare(...)` 是唯一装配入口，按固定顺序组装一次对话所需的全部上下文：

1. **策略校验**：`PolicyEngine` 在装配最前面执行护栏与限流，命中直接拒绝，不消耗模型额度
2. **系统提示词**：智能体提示词 + 绑定的技能提示词
3. **RAG**：检索绑定知识库的相关片段注入上下文（无命中时明确告警，不伪造参考资料）
4. **模型选择**：显式 modelId > 智能体绑定模型 > 路由策略 > 前端指定供应商 > 默认模型
5. **MCP 工具**：加载绑定的 MCP 服务工具及调用绑定
6. **消息装配**：按智能体配置的历史条数截断，避免长会话无限膨胀

模型不可用时会**明确报错**（禁用 / 下线 / 配置不完整 / 脱敏密钥占位符），不做静默回退，
避免「切换没生效」难以排查。仅在完全未配置真实供应商时回退本地演示模式，并在回复中显式提示。

### 2. 工作流节点执行器 SPI

```
NodeExecutor (type() + execute(NodeContext) → NodeResult)
├── McpNodeExecutor      MCP_TOOL  连接池复用、重试、连接/调用耗时分离
├── LlmNodeExecutor      LLM       支持温度、系统提示词、变量模板渲染
└── AgentNodeExecutor    AGENT     调用另一个智能体，独立装配其提示词/技能/RAG/MCP
```

- 其余节点（知识检索 / 代码 / 分支 / 开始 / 结束）不改变既有链路，只做「跳过」推进
- 变量引用统一走 `VariableResolver`，支持 `{{start.query}}`、`{{mcpResult.data}}` 等路径
- 编排运行以 SSE 推送 `start / step(running|success|failed|skipped) / failed / done / run-cancelled / error`
- AGENT 节点做归属隔离与自调用检测，禁止透过编排借用他人智能体的提示词与资源
- 编排运行同样落会话日志，监控页能看到编排链路的真实用量、失败率与实际模型

### 3. Agent 策略真正生效

集成页配置的三类策略在运行时由 `PolicyEngine` 执行（此前只有增删改查、运行时不读取）：

| 类型 | 配置项 | 行为 |
|---|---|---|
| guardrail | `blockKeywords` / `maxInputChars` | 命中关键词或超长 → `400` 直接拦截 |
| limit | `maxRequestsPerMinute` | 按用户+智能体维度限流 → 超限 `429` |
| routing | `modelId` | 未显式指定模型时改用该模型 |

`t_agent_policy` 无 `agent_id` 列，策略与智能体的关联通过 config 里的 `agentIds` 数组表达，缺省表示对该用户全部智能体生效。

### 4. 模型供应商：配置驱动

- 只实现 `OpenAiProtocolClient` / `AnthropicProtocolClient` 两个协议适配器，由 `provider.protocol` 列驱动分发，**禁止按厂商写 Adapter**
- 供应商、凭证字段（JSON Schema）、模型全部来自数据库：**新增一家供应商 = 插一条 SQL，Java/前端零改动**
- 前端按 `credential_schema` 动态渲染 form 控件，后端同源做必填校验（双保险）
- **先校验后落库**：安装前先打最小代价探测（GET /models → POST /chat/completions → 超时兜底），失败不产生脏数据
- 同一 workspace + 供应商校验 5 秒 1 次（Redis INCR + EXPIRE）
- 拉取模型用独立线程池异步执行，失败只标记状态，不影响凭证已落库

### 5. 密钥与安全

| 机制 | 说明 |
|---|---|
| 字段加密 | API Key 等字段 AES-256-GCM 加密入库（随机 12B IV，base64(iv+密文)），带 `enc:v1:` 前缀 |
| 脱敏回显 | 出参只回显 `sk-****abcd`，原样回传不会覆盖真实密钥（按 `****` 判定） |
| 密钥托管 | 未注入环境变量时按「数据库 `t_system_secret` → `./data/*.key` → 自动生成」解析，启动后写入全局配置 |
| 日志脱敏 | 不打印 Authorization / API Key；审计日志记录操作人与目标资源 ID |
| 归属隔离 | 智能体 / 技能 / 知识库 / 模型 / MCP 服务全部按用户与 scope 过滤，越权按「不存在」处理 |
| 安全传输 | `TRANSPORT_SECURE_ENABLED=true` 时前后端原始数据按「压缩 → 加密 → Base64」传输 |

### 6. 知识库真实检索链路

- 入库：文件 → 文本抽取（PDF/Word/Excel/TXT/MD/CSV/JSON/HTML）→ 分段 → 调 embedding 接口向量化 → 写切片，SSE 推送进度
- 检索：query 向量化后用余弦相似度打分
- **没有可用向量模型时返回 null 并自动退回词法检索**，同时在结果里标注真实模式，不把词法检索伪装成向量检索
- 入库任务状态持久化在 `t_kb_document`，页面关闭后任务继续，重新进入可续看进度

---

## 六、主要接口

统一前缀 `/api`，认证方式为请求头 `satoken`（登录后返回），响应体统一 `{ code, message, data }`。

| 模块 | 前缀 | 说明 |
|---|---|---|
| 认证 | `/api/auth` | 注册 / 登录 / 登出 / 当前用户 |
| 对话 | `/api/chat` | 会话 CRUD、消息列表、流式对话 `/stream`、提示词增强、停止、反馈 |
| 智能体 | `/api/agents` | CRUD、发布 / 下线、复制、导出、从文件 / URL 导入 |
| 编排 | `/api/workflows` | 画布读写、运行（SSE）、停止、可用变量、发布前检查 |
| 应用 | `/api/apps` | 访问入口、发布 / 下线、API Key、Web 配置、频道开关、触发器 |
| 公开入口 | `/api/public/apps/{id}`、`/api/v1/apps/{id}/chat`、`/api/mcp/app/{id}` | 免登录 Web / API / MCP 入口，Webhook 回调 |
| 应用运维 | `/api/apps/{appId}/logs`、`/monitor`、`/annotations`、`/agent` | 日志、监测、标注、应用专属对话 |
| 智能体生成 | `/api/agent-forge` | 模板、创建运行、事件流、澄清、方案确认、取消、发布 |
| 知识库 | `/api/knowledge-bases`、`/api/kb` | CRUD、文档、检索、上传、入库进度、流水线、召回测试、统计、外部 API |
| 外部知识库 | `/api/v1/datasets` | 对外开放的数据集检索接口 |
| 技能 | `/api/skills` | CRUD、发布 / 下线、zip 上传、文件树读写 |
| MCP 服务 | `/api/mcp/services` | CRUD、工具清单、同步、单工具调用与启用、测试、导入 |
| MCP 节点 | `/api/mcp` | 编排节点选工具时的 schema 与调试 |
| 模型接入 | `/api/models` | CRUD、启用、凭据、每千 token 单价、发布 / 下线 |
| 供应商市场 | `/api/providers` | 市场、详情、预校验、安装、已安装、换凭证、同步 / 添加模型、默认模型 |
| 能力 | `/api/capabilities`、`/api/studio/apps/count` | 能力广场聚合、我的能力、应用计数 |
| 集成 | `/api/integrations` | 触发器、策略、扩展、自定义端点、成员 |
| 触发器目录 | `/api/triggers` | 可用触发器类型目录 |
| 插件市场 | `/api/plugin-market` | 条目、同步、安装、白名单、图标 |
| 测试 | `/api/test/chat`、`/api/mcp/{id}/invoke`、`/api/skills/{id}/run`、`/api/kb/{id}/search` | 各能力调试入口 |
| 文档 | `/api/docs` | 文档协作 |

### 新增一家供应商（只改 SQL）

```sql
INSERT IGNORE INTO provider
(name, display_name, provider_group, icon, protocol, default_base_url,
 credential_schema, model_fetch_mode, official_url, doc_url, description, notes, sort, enabled)
VALUES
('xmodel', '某某模型', 'AGGREGATOR', 'X', 'OPENAI', 'https://api.xmodel.com/v1',
 '{"type":"object","required":["apiKey"],
   "properties":{"apiKey":{"type":"string","format":"password","title":"API Key"},
                 "baseUrl":{"type":"string","title":"接入地址",
                           "default":"https://api.xmodel.com/v1"}}}',
 'REMOTE', 'https://xmodel.com', 'https://xmodel.com/docs', '一句话简介', '接入须知（选填）', 2, 1);
```

刷新市场页即出现该供应商（无需重启、无需改任何 Java/前端代码）。重复执行不报错。

---

## 七、数据表一览

表名统一 `t_` 前缀（供应商模块为 `provider*`），所有表与字段均带注释，逻辑删除字段 `deleted`。

| 分组 | 表 |
|---|---|
| 系统 | `t_system_secret`、`t_user`、`t_workspace_member` |
| 对话 | `t_chat_session`、`t_chat_message`、`t_document` |
| 模型 | `t_llm_model`、`provider`、`provider_credential`、`provider_model`、`provider_default_model` |
| 智能体 | `t_agent`、`t_agent_version`、`t_agent_file`、`t_app_config`、`t_agent_policy`、`t_agent_tool_binding` |
| 编排 | `t_workflow`、`t_mcp_tool_cache`、`t_mcp_invoke_log` |
| 知识库 | `t_knowledge_base`、`t_knowledge_doc`、`t_kb_document`、`t_kb_template` |
| 技能 | `t_skill`、`t_skill_file` |
| MCP | `t_mcp_server`、`t_mcp_service` |
| 运维 | `t_conversation_log`、`t_stat_daily`、`t_annotation` |
| 集成 | `t_trigger`、`t_extension`、`t_api_endpoint` |
| 插件市场 | `t_plugin_market_item`、`t_plugin_market_whitelist`、`t_installed_plugin` |

---

## 八、定时任务

| 任务 | 频率 | 说明 |
|---|---|---|
| `StatAggregator` | 每天 00:10 | 汇总前一日会话日志到 `t_stat_daily`，供监控页看历史趋势 |
| `TriggerScheduler` | 启动 20s 后，每 30s 轮询 | 执行到期的定时触发器 |
| `PluginSyncScheduler` | 每天 03:00 | 同步 Dify 插件市场条目 |

---

## 九、生产部署注意事项

需要显式注入的变量：

| 变量 | 作用 | 不注入的后果 |
|---|---|---|
| `PROVIDER_AES_KEY` | 加密 `t_llm_model.api_key` 等字段 | 密钥与密文同库，整库导出即可解密全部 Key |
| `SA_JWT_SECRET` | 登录令牌签名 | 同上：密钥托管在库中，且多实例需共用同一份存储 |
| `TRANSPORT_SECRET_KEY` | 安全传输编解码（仅 `TRANSPORT_SECURE_ENABLED=true` 时需要） | 开启该功能时会启动失败 |
| `APP_SECURITY_STRICT` | 设为 `true` 时上述密钥缺失直接启动失败 | 默认 `false`，只告警不阻断 |
| `APP_CORS_ALLOWED_ORIGINS` | 允许的跨域来源 | 默认 `*`，仅适合开发环境 |

**切换密钥来源后不要删除旧的密钥记录 / 文件**，否则历史密文与已签发令牌将无法恢复。

模型凭据（可选，用于自动播种公共模型）：`QWEN_BASE_URL` `QWEN_API_KEY` `QWEN_MODEL`
`HUNYUAN_BASE_URL` `LLM_API_KEY` `OPENAI_BASE_URL` `OPENAI_API_KEY` `OPENAI_MODEL`。

---

## 十、已知取舍

1. **未引入 openai-java SDK**：其依赖与 Boot 3.3.5 存在版本耦合，协议层直接用统一 WebClient（连接池 / 超时 / 禁用重定向）实现，等价覆盖「官方 SDK + WebClient」诉求。
2. **手动添加模型接口为规格外扩展**：Anthropic（无 `/models`）、火山方舟（`ep-` 接入点）、Ollama（MANUAL）均依赖它完成闭环。
3. **发布到能力广场**：`t_llm_model` 中仅存脱敏 Key（避免明文二次入库），真实聊天调用由凭证体系在调用层解密接入；若密钥被写成脱敏占位符，调用时会明确报错而不是返回 401 让用户猜。
4. **本地演示兜底**：完全未配置真实供应商时回退 `LocalProvider`，只在回复正文里显式声明「本地演示模式」，不会伪装成真实模型输出。

---

## 十一、开发约定

- 前端类型校验：`cd frontend && npx vue-tsc --noEmit -p tsconfig.json`（必须带 `-p`）
- 前端构建：`npm run build`（先类型校验再 vite build）
- 后端编译：`cd backend && mvn -o compile -DskipTests`
- 后端测试：`cd backend && mvn -o test`
- 风格：复用项目现有组件与 design token（`--primary #2f6bff` 等），不新增第三方依赖、不硬覆盖色值
