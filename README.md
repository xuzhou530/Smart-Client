# 全栈技术说明文档 —— 桌面端 AI 对话与文档协作工作台

> 前端：Electron + Vue3（打包为桌面客户端）  
> 后端：Java 21 + Spring Boot 3.x  
> AI：预留统一大模型调用接口（可切换供应商/模型）  
> 目标：一份能让 AI 或团队照着落地的完整全栈规范。

---

## 1. 项目概述

本项目是一个**桌面端 AI 对话与文档协作工作台**，前端打包为可在 Windows / macOS / Linux 安装的桌面应用，后端提供会话管理、文档协作、AI 调用代理等能力。

| 层       | 技术                                                   | 说明                                        |
| -------- | ------------------------------------------------------ | ------------------------------------------- |
| 桌面前端 | Electron 30+                                           | 桌面壳，承载 Vue 渲染进程                   |
| UI 框架  | Vue 3.4（`<script setup>`）+ TypeScript                | 主渲染层                                    |
| 构建     | Vite 5                                                 | 极速 HMR                                    |
| UI 组件  | Element Plus（推荐）/ Naive UI                         | 表单、弹窗、消息                            |
| 状态     | Pinia                                                  | 全局状态                                    |
| 路由     | Vue Router 4                                           | 页面路由                                    |
| 桌面能力 | electronAPI（contextBridge）                           | 文件、通知、窗口、剪贴板、更新              |
| 通信     | Electron 主进程 ↔ 渲染进程（IPC）                      | 安全桥接                                    |
| 后端     | Java 21 + Spring Boot 3.3                              | RESTful API                                 |
| 鉴权     | Sa-Token 1.38+（StpUtil + JWT 整合包，前后端分离模式） | 登录、权限、踢人、同端互斥登录              |
| 持久化   | MyBatis-Plus 3.5.6 + MySQL 8                           | 结构化数据，内置通用 CRUD / 分页 / 代码生成 |
| 缓存     | Redis 7                                                | Token、会话、限流                           |
| AI 网关  | 自封装 `LlmGateway`                                    | 统一适配不同 LLM 供应商                     |
| 构建工具 | Maven 3.9 / Gradle 8                                   | 后端构建                                    |
| 桌面打包 | electron-builder                                       | 生成 exe/dmg/AppImage                       |

---

## 2. 系统总体架构

```
┌──────────────────────────────────────────────────────────┐
│                   桌面客户端 (Electron)                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │ 渲染进程 (Vue3 + Vite + Element Plus)              │  │
│  │  - 对话页 / 文档页 / 设置页                         │  │
│  │  - 通过 axios 调后端 API                            │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │ 主进程 (Node.js)                                   │  │
│  │  - 窗口管理 / 菜单 / 原生通知                       │  │
│  │  - 文件读写 / 剪贴板 / 自动更新                     │  │
│  │  - 通过 preload.js + contextBridge 暴露 API         │  │
│  └────────────────────────────────────────────────────┘  │
│             ↕ IPC（进程间通信）                          │
└─────────────┬────────────────────────────────────────────┘
              │ HTTPS / WSS（SSE 流式）
              ▼
┌──────────────────────────────────────────────────────────┐
│                 Spring Boot 后端 (8080)                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Sa-Token 全局拦截器：登录校验、权限、踢人           │  │
│  │ + 限流 / 审计 / CORS / 统一异常处理                │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐   │
│  │ Auth     │  │ Chat     │  │ Document              │   │
│  │ 登录/注册 │  │ 会话/消息 │  │ 协作文档              │   │
│  └──────────┘  └────┬─────┘  └──────────┬────────────┘   │
│                     │                    │                │
│              ┌──────┴──────┐             │                │
│              │ LlmGateway  │◄────────────┘                │
│              │ (AI 网关)   │                              │
│              └──┬──┬──┬────┘                              │
│                 │  │  │                                   │
│         腾讯混元 OpenAI 自建模型 ...                       │
└─────────────┬─────────┬─────────┬─────────────────────────┘
              ▼         ▼         ▼
         MySQL 8      Redis 7   对象存储(MinIO/S3)
```

**关键设计原则**

1. **前后端分离**：桌面端只当“浏览器 + 原生壳”，业务逻辑全部下沉后端。
2. **AI 调用走后端代理**：API Key 不暴露给客户端，统一在后端鉴权、限流、计费。
3. **流式输出**：AI 回复用 SSE（`text/event-stream`）逐字推送，体验更自然。
4. **多供应商可插拔**：`LlmGateway` 定义统一接口，各厂商实现 `LlmProvider`。

---

## 3. 前端：Electron + Vue3 桌面端

### 3.1 项目结构

```
desktop-client/
├── electron/                    # Electron 主进程
│   ├── main.js                  # 窗口、生命周期
│   ├── preload.js               # contextBridge 暴露 API
│   └── ipc/                     # IPC 处理
│       ├── file.js              # 文件读写
│       ├── notification.js      # 原生通知
│       └── updater.js           # 自动更新
├── src/                         # Vue 渲染进程
│   ├── main.ts                  # 入口
│   ├── App.vue
│   ├── router/
│   ├── stores/                  # Pinia
│   │   ├── user.ts
│   │   ├── chat.ts
│   │   └── doc.ts
│   ├── api/                     # 后端接口封装
│   │   ├── request.ts           # axios 实例 + 拦截器
│   │   ├── chat.ts
│   │   └── doc.ts
│   ├── views/
│   │   ├── ChatView.vue
│   │   ├── DocView.vue
│   │   └── SettingsView.vue
│   ├── components/
│   │   ├── Sidebar.vue
│   │   ├── MessageList.vue
│   │   ├── InputBar.vue
│   │   └── OptionCard.vue
│   └── assets/
├── index.html
├── vite.config.ts
├── electron-builder.yml         # 打包配置
└── package.json
```

### 3.2 关键配置

#### `vite.config.ts`（核心）

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  base: './',                    // 必须，Electron 用 file:// 协议
  server: { port: 5173 },
  build: {
    outDir: 'dist',
    emptyOutDir: true,
    target: 'chrome120'
  }
})
```

#### `electron/main.js`（核心）

```js
import { app, BrowserWindow, ipcMain } from 'electron'
import { join } from 'path'

let win
function createWindow() {
  win = new BrowserWindow({
    width: 1200,
    height: 800,
    minWidth: 900,
    minHeight: 600,
    webPreferences: {
      preload: join(__dirname, 'preload.js'),
      contextIsolation: true,    // 必须 true，安全
      nodeIntegration: false     // 必须 false
    }
  })

  if (process.env.VITE_DEV_SERVER_URL) {
    win.loadURL(process.env.VITE_DEV_SERVER_URL)
  } else {
    win.loadFile(join(__dirname, '../dist/index.html'))
  }
}
app.whenReady().then(createWindow)
```

#### `electron/preload.js`（安全桥接）

```js
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  // 文件
  readFile: (path) => ipcRenderer.invoke('file:read', path),
  writeFile: (path, data) => ipcRenderer.invoke('file:write', path, data),
  // 原生通知
  notify: (title, body) => ipcRenderer.invoke('notify', title, body),
  // 自动更新
  checkUpdate: () => ipcRenderer.invoke('updater:check'),
  // 窗口
  minimize: () => ipcRenderer.invoke('window:minimize')
})
```

#### `src/api/request.ts`（统一请求）

```ts
import axios from 'axios'

const request = axios.create({
  baseURL: import.meta.env.VITE_API_BASE ?? 'http://localhost:8080',
  timeout: 30000
})

request.interceptors.request.use((config) => {
  const token = localStorage.getItem('satoken')
  if (token) config.headers.satoken = token      // Sa-Token 默认请求头字段
  return config
})

request.interceptors.response.use(
  (res) => res.data,
  (err) => {
    if (err.response?.status === 401) {
      // 跳登录
    }
    return Promise.reject(err)
  }
)

export default request
```

### 3.3 桌面专属能力清单

| 能力         | 实现方式           | 用途                    |
| ------------ | ------------------ | ----------------------- |
| 本地文件读写 | `fs` via IPC       | 导入/导出文档、保存草稿 |
| 原生通知     | `Notification`     | 新消息提醒              |
| 系统托盘     | `Tray`             | 后台驻留                |
| 自动更新     | `electron-updater` | 增量升级                |
| 剪贴板       | `clipboard`        | 复制代码/结果           |
| 全局快捷键   | `globalShortcut`   | 唤起窗口                |
| 多窗口       | `BrowserWindow`    | 独立文档窗口            |

### 3.4 开发与打包

```bash
# 安装依赖
npm install

# 开发（同时启动 Vite + Electron）
npm run dev

# 构建渲染层
npm run build

# 打包为安装包（按当前系统）
npm run dist:win      # 生成 .exe
npm run dist:mac      # 生成 .dmg
npm run dist:linux    # 生成 .AppImage
```

`electron-builder.yml` 关键配置：

```yaml
appId: com.your.app
productName: AI工作台
directories:
  output: release
files:
  - dist/**/*
  - electron/**/*
mac:
  target: dmg
win:
  target: nsis
nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true
```

---

## 4. 后端：Java / Spring Boot

### 4.1 项目结构

```
backend/
├── pom.xml
└── src/main/
    ├── java/com/example/aiclient/
    │   ├── AiclientApplication.java
    │   ├── config/              # Sa-Token、MyBatis-Plus、Redis、跨域、SSE 配置
    │   ├── controller/          # REST 接口
    │   │   ├── AuthController.java
    │   │   ├── ChatController.java
    │   │   └── DocController.java
    │   ├── service/             # 业务逻辑
    │   ├── mapper/              # MyBatis-Plus Mapper（继承 BaseMapper）
    │   ├── entity/              # 数据库实体（对应表）
    │   ├── model/               # DTO / VO / Req / Resp
    │   ├── sa/                  # Sa-Token 配置与全局拦截器
    │   │   ├── SaTokenConfig.java
    │   │   └── SaTokenInterceptor.java
    │   ├── llm/                 # AI 网关
    │   │   ├── LlmGateway.java          # 统一接口
    │   │   ├── LlmRequest.java
    │   │   ├── LlmResponse.java
    │   │   └── provider/
    │   │       ├── HunyuanProvider.java
    │   │       ├── OpenAiProvider.java
    │   │       └── LocalProvider.java
    │   └── common/              # 统一响应、异常
    └── resources/
        ├── application.yml
        └── application-dev.yml
```

### 4.2 核心依赖 `pom.xml`

```xml
<properties>
  <sa-token.version>1.38.0</sa-token.version>
  <mybatis-plus.version>3.5.6</mybatis-plus.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Sa-Token 核心（权限、会话、踢人） -->
  <dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-boot3-starter</artifactId>
    <version>${sa-token.version}</version>
  </dependency>
  <!-- Sa-Token 整合 Redis（分布式会话 / 登录态存 Redis） -->
  <dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-redis-jackson</artifactId>
    <version>${sa-token.version}</version>
  </dependency>
  <!-- Sa-Token 整合 JWT（前后端分离下发 token） -->
  <dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-jwt</artifactId>
    <version>${sa-token.version}</version>
  </dependency>

  <!-- MyBatis-Plus -->
  <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>${mybatis-plus.version}</version>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  <dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-core</artifactId>
  </dependency>
</dependencies>
```

> 说明：去掉了 `spring-boot-starter-security` 与 `jjwt-api`，改由 **Sa-Token + sa-token-jwt** 一站式提供登录鉴权；持久化由 **MyBatis-Plus** 承接，无需再引 JPA。

### 4.3 核心配置 `application.yml`

```yaml
server:
  port: 8080

spring:
  application:
    name: aiclient-backend
  datasource:
    url: jdbc:mysql://localhost:3306/aiclient?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: ${DB_USER:root}
    password: ${DB_PASSWORD}   # 通过环境变量注入，不要提交真实口令
    driver-class-name: com.mysql.cj.jdbc.Driver
  data:
    redis:
      host: localhost
      port: 6379

# MyBatis-Plus 配置
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true   # 下划线自动转驼峰
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 开发期打印 SQL，生产关
  global-config:
    db-config:
      id-type: auto                     # 主键自增
      logic-delete-field: deleted        # 逻辑删除字段
      logic-delete-value: 1
      logic-not-delete-value: 0
  mapper-locations: classpath*:/mapper/**/*.xml

# Sa-Token 配置
sa-token:
  token-name: satoken                    # token 名称（请求头字段名）
  timeout: 7200                          # token 有效期 2h（单位：秒）
  active-timeout: -1                     # 无操作超时，-1 不启用
  is-concurrent: true                    # 同账号是否允许并发登录
  is-share: false                        # 同端互斥登录（后登录踢前者）
  token-style: uuid                      # token 风格
  is-log: false                          # 是否打印日志
  is-read-cookie: false                  # 前后端分离项目关闭 cookie
  is-read-header: true                   # 从请求头读取 token
  # 整合 JWT 后，实际下发的是 jwt 格式的 token，需另配 jwt-secret（见 SaTokenConfig）

llm:
  default-provider: hunyuan   # 默认供应商
  providers:
    hunyuan:
      base-url: https://api.hunyuan.cloud.tencent.com
      api-key: ${LLM_API_KEY}
      model: hunyuan-pro
    openai:
      base-url: https://api.openai.com
      api-key: ${OPENAI_API_KEY}
      model: gpt-4o
```

#### Sa-Token 配置类 `SaTokenConfig.java`

```java
@Configuration
public class SaTokenConfig {

    @Bean
    public SaTokenInterceptor saTokenInterceptor() {
        return new SaTokenInterceptor()
                .addInclude("/**")
                .addExclude("/api/auth/login", "/api/auth/register", "/favicon.ico");
    }

    @Bean
    public SaTokenDao saTokenDaoForRedis() {
        // 引入 sa-token-redis-jackson 后自动注入，登录态存 Redis
        return new SaTokenDaoRedisJackson();
    }
}
```

#### Sa-Token + JWT 整合（可选，前后端分离签发 jwt 格式 token）

```java
@Configuration
public class SaJwtConfig {
    @Bean
    public StpLogic getStpLogicJwt() {
        return new StpLogicJwtForStyle();   // 设置为 jwt 风格
    }
}
```

> 在 `application.yml` 同级放置 `sa-token.jwt-secret` 或在配置类指定 `SaJwtUtil.setSecret("your-256-bit-secret-key")`。

### 4.4 AI 网关设计（预留接口，可切换模型）

#### 统一请求/响应

```java
public record LlmRequest(
    String model,
    List<Message> messages,
    boolean stream,
    Double temperature
) {
    public record Message(String role, String content) {}
}

public record LlmResponse(
    String content,
    String model,
    Usage usage
) {}
```

#### 统一接口

```java
public interface LlmGateway {
    LlmResponse chat(LlmRequest request);
    Flux<String> chatStream(LlmRequest request);   // SSE 逐字流
}
```

#### 工厂路由

```java
@Service
public class LlmGatewayFactory {
    private final Map<String, LlmGateway> providers;

    public LlmGateway get(String name) {
        return providers.getOrDefault(name, providers.get("hunyuan"));
    }
}
```

#### Controller（流式输出）

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<SseEmitter> stream(@RequestBody LlmRequest req) {
        SseEmitter emitter = new SseEmitter(60_000L);
        llmFactory.get(req.model() == null ? defaultProvider : req.model())
                  .chatStream(req)
                  .subscribe(
                      chunk -> emitter.send(SseEmitter.event().data(chunk)),
                      emitter::completeWithError,
                      emitter::complete
                  );
        return Flux.just(emitter);
    }
}
```

### 4.6 持久化层：MyBatis-Plus 用法

#### 实体（Entity）

```java
@TableName("chat_session")
public class ChatSession {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long userId;
    private String title;
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createdAt;
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updatedAt;
    // getter/setter
}
```

#### Mapper（继承 BaseMapper，零 SQL 获得 CRUD）

```java
@Mapper
public interface ChatSessionMapper extends BaseMapper<ChatSession> {
    // 需要自定义 SQL 时在此加方法 + 对应 XML
}
```

#### Service 层（继承 IService）

```java
public interface ChatSessionService extends IService<ChatSession> {}

@Service
public class ChatSessionServiceImpl
        extends ServiceImpl<ChatSessionMapper, ChatSession>
        implements ChatSessionService {}

// 调用示例
public List<ChatSession> listByUser(Long userId) {
    return lambdaQuery()
            .eq(ChatSession::getUserId, userId)
            .orderByDesc(ChatSession::getUpdatedAt)
            .list();
}
```

#### 分页（MyBatis-Plus 内置分页插件）

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}

// 使用
Page<ChatMessage> page = page(new Page<>(1, 20),
        lambdaQuery().eq(ChatMessage::getSessionId, sessionId));
```

#### 自动填充时间（MetaObjectHandler）

```java
@Component
public class TimeFillHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject mo) {
        strictInsertFill(mo, "createdAt", LocalDateTime.class, LocalDateTime.now());
        strictInsertFill(mo, "updatedAt", LocalDateTime.class, LocalDateTime.now());
    }
    @Override
    public void updateFill(MetaObject mo) {
        strictUpdateFill(mo, "updatedAt", LocalDateTime.class, LocalDateTime.now());
    }
}
```

#### 代码生成器（一键生成 entity / mapper / service / controller）

```java
public class CodeGenerator {
    public static void main(String[] args) {
        FastAutoGenerator.create("jdbc:mysql://localhost:3306/aiclient", "root", "root")
            .globalConfig(b -> b.author("your-name").outputDir("src/main/java").openDir(false))
            .packageConfig(b -> b.parent("com.example.aiclient").entity("entity").mapper("mapper"))
            .strategyConfig(b -> b.addInclude("user", "chat_session", "chat_message", "document"))
            .execute();
    }
}
```

### 4.7 登录鉴权：Sa-Token 典型流程

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @PostMapping("/login")
    public Result<?> login(@RequestBody LoginReq req) {
        // 1. 校验用户名密码（BCrypt 比对）
        User user = userService.getByUsername(req.username());
        if (user == null || !BCrypt.checkpw(req.password(), user.getPassword())) {
            return Result.fail("用户名或密码错误");
        }
        // 2. Sa-Token 登录，签发 token（整合 jwt 后下发 jwt）
        StpUtil.login(user.getId());
        String token = StpUtil.getTokenValue();
        return Result.ok(Map.of("token", token, "userId", user.getId()));
    }

    @PostMapping("/logout")
    public Result<?> logout() {
        StpUtil.logout();
        return Result.ok();
    }
}

// 获取当前登录用户 ID
Long userId = StpUtil.getLoginIdAsLong();
// 判断是否登录
StpUtil.isLogin();
// 踢人下线
StpUtil.kickout(userId);
// 权限校验（如需要）
StpUtil.checkPermission("doc:edit");
```

### 4.8 主要 RESTful 接口

| 方法   | 路径                               | 说明                                 |
| ------ | ---------------------------------- | ------------------------------------ |
| POST   | `/api/auth/register`               | 注册                                 |
| POST   | `/api/auth/login`                  | 登录，返回 Sa-Token（jwt 格式）token |
| POST   | `/api/auth/logout`                 | 登出                                 |
| GET    | `/api/chat/sessions`               | 会话列表（MyBatis-Plus 分页）        |
| POST   | `/api/chat/sessions`               | 新建会话                             |
| GET    | `/api/chat/sessions/{id}/messages` | 消息历史                             |
| POST   | `/api/chat/stream`                 | AI 流式对话（SSE）                   |
| GET    | `/api/docs`                        | 文档列表                             |
| POST   | `/api/docs`                        | 新建文档                             |
| PUT    | `/api/docs/{id}`                   | 更新文档                             |
| DELETE | `/api/docs/{id}`                   | 删除文档                             |

统一响应体：

```json
{ "code": 0, "message": "success", "data": {} }
```

---

## 5. 数据模型

```sql
-- 用户
CREATE TABLE user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(64) UNIQUE NOT NULL,
  password VARCHAR(128) NOT NULL,
  nickname VARCHAR(64),
  avatar VARCHAR(255),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 会话
CREATE TABLE chat_session (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  title VARCHAR(255),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 消息
CREATE TABLE chat_message (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_id BIGINT NOT NULL,
  role ENUM('user','assistant','system') NOT NULL,
  content TEXT NOT NULL,
  tokens INT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 文档
CREATE TABLE document (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  title VARCHAR(255),
  content LONGTEXT,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## 6. 安全设计

1. **鉴权（Sa-Token + JWT）**：登录后由 Sa-Token 签发 jwt 格式 token，前端存 `localStorage`，每次请求带 `satoken` 请求头；`StpUtil.login/getLoginId/checkPermission` 一行完成登录态与权限校验。
2. **密码**：BCrypt 加盐存储，禁止明文。
3. **同端互斥**：开启 `sa-token.is-share=false`，同账号后登录踢掉前者，避免多点乱写。
4. **踢人 / 强制下线**：`StpUtil.kickout(userId)` 可用于风控、改密重登等场景。
5. **桌面端 API Key 隔离**：所有第三方模型 Key 只在后端配置，客户端无感知。
6. **CORS**：生产环境限定具体域名（Electron 用 `file://` 或自定义协议，需单独放行）。
7. **限流**：基于 Redis 的令牌桶，防止滥用（每用户每分钟 N 次）。
8. **审计日志**：记录关键操作（登录、删除、AI 调用）。

---

## 7. 本地开发流程

### 7.1 环境准备

- JDK 21
- Node.js 20+
- MySQL 8
- Redis 7

### 7.2 启动后端

```bash
cd backend
mvn spring-boot:run
# 或打 jar
mvn clean package -DskipTests
java -jar target/aiclient-backend.jar
```

### 7.3 启动桌面端

```bash
cd desktop-client
npm install
npm run dev          # 同时拉起 Vite + Electron
```

---

## 8. 部署与分发

### 8.1 后端部署

- **容器化**：`Dockerfile` + `docker-compose.yml`（后端 + MySQL + Redis）
- **云部署**：可部署到任意支持 Java 的云主机 / K8s
- **HTTPS**：前置 Nginx 反代 + 证书

### 8.2 桌面端分发

- `electron-builder` 生成对应平台安装包
- 可选接入「自动更新」：把新版 `latest.yml` 与安装包放 CDN，客户端启动时检查

---

## 9. 后续可扩展方向

| 方向       | 说明                                                |
| ---------- | --------------------------------------------------- |
| RAG 知识库 | 接入向量库（Milvus/pgvector），上传文档后做检索增强 |
| 多模态     | 支持图片/文件上传，调用多模态模型                   |
| 协作编辑   | 文档接入 WebSocket + CRDT，实现多人实时协作         |
| 计费/配额  | 基于 token 用量做额度与计费                         |
| 插件系统   | 开放技能/工具调用（Function Calling）               |
| 国际化     | i18n 支持多语言                                     |

---

## 10. 交付验收清单

### 前端（Electron + Vue）

- [ ] 桌面端可正常启动，窗口/托盘/通知正常
- [ ] 登录 → Sa-Token（jwt）存储 → 请求自动带 satoken 头
- [ ] 对话流式输出可用（SSE 逐字显示）
- [ ] 本地文件读写、剪贴板可用
- [ ] 打包出 exe / dmg 可安装运行
- [ ] 侧边栏折叠、响应式适配正常

### 后端（Spring Boot）

- [ ] 登录注册、Sa-Token（jwt）鉴权正常，token 拦截生效
- [ ] 会话/消息/文档 CRUD 正常（MyBatis-Plus 通用 CRUD + 分页）
- [ ] AI 网关可切换供应商（改配置即可）
- [ ] SSE 流式接口可用
- [ ] 限流、异常处理、审计日志到位
- [ ] Docker 一键启动全部依赖

### AI 能力

- [ ] 已预留 `LlmGateway` 统一接口
- [ ] 至少接入一家供应商可跑通（如混元）
- [ ] API Key 仅在后端，客户端无暴露风险

---

> 使用说明：本文档可直接交给 AI 或开发团队落地。  
> 若需**项目目录骨架代码**（含 `pom.xml`、`main.js`、`preload.js`、示例 Controller 等可直接跑的空工程），可在此文档基础上继续生成。
