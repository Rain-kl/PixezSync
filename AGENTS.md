# AGENTS.md — 项目AI助手工作操作手册

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.


## Git 提交规范

遵循 Conventional Commits：`<type>(<scope>): <subject>`（例：`feat(auth): support email login`）。

## 务必阅读匹配的 Skill

| Skill | 何时使用 |
| :--- | :--- |
| `new-api` | 添加或修改自定义业务 API、Handler、服务层逻辑、自定义路由注册 |
| `new-async-task` | 添加或修改 Asynq 任务、定时任务、TaskHandler、任务元数据 |
| `new-setting` | 添加或修改系统/业务/公开设置、`/admin/system` 参数或 `/admin/settings` 图形化设置 |
| `database-migration` | 数据库表结构变更、goose SQL 迁移（PG/SQLite/ClickHouse）、seed 数据 |
| `file-upload` | 业务上传文件、Worker 程序化摄取、`upload.Ingest` 策略选型、文件访问与 `w_uploads` / 统计排查 |
| `cache-framework` | 新增或修改业务缓存（RAM/Redis/DB 三层读路径）、缓存失效、多节点 pub/sub 同步、评估高频读是否应接入缓存 |
| `push-notification` | 系统通知推送事件、统一触发器投递、带消息推送的业务功能 |
| `release-guide` | 根据自上一正式版本 Tag 以来的提交整理 Version Bump 提交信息以触发双语 Release |
| `shadcn` | 添加、修改或组合 shadcn/ui 组件 |


## 严格遵循事项 (Guardrails)

- 切勿删除 `frontend/node_modules`
- 保持 `internal/util/` 绝对纯净且不引入任何框架。禁止从 `internal/util/` 及其子包中导入 Gin、GORM、sessions 等 HTTP/Web/数据库相关框架包（例如，Web 会话选项已收敛至 `internal/apps/oauth/session.go`）。
- 编写测试用例时，禁止使用硬编码的相对路径（如 `"uploads/test_cache"`）在源码目录下创建临时测试目录，必须统一使用 Go 内置的 `t.TempDir()` 以避免污染源码目录。
- 所有 HTTP 路由仅在 `internal/router/router.go` 中注册。
- 当 API Handler 发生变化时，更新 Swagger 文档（运行 `make swagger`）。
- 在完成代码开发后必须运行 `make code-check`, 并修复报错。
- 需要缓存或文件管理能力时，必须复用现有平台实现，禁止在业务包中自行创建缓存目录、直接管理缓存文件或重复封装存储后端。
- 文件摄取必须通过 `upload.Ingest`（`upload.PolicyCreate` / `PolicyDedupNewRecord` / `PolicyResolveExisting`）；删除必须通过 `upload.Remove` 或 `upload.RemoveOwned`。禁止业务模块直接调用 `repository.CreateUpload` / `repository.SoftDeleteUpload`，禁止 `db.Create(&model.Upload{})` 旁路写 `w_uploads`。
- 禁止在 `init()` 中注册跨模块集成（任务 Handler、推送内置事件、域事件监听器、任务完成钩子）。统一通过 `internal/bootstrap` 在 `internal/cmd` 入口显式装配。
- `internal/router/router.go` 的 `Serve()` 仅负责 HTTP 路由与中间件，禁止在其中执行 `SyncEvents`、`InitLogWriter` 等进程级运行时初始化。
- 核心业务模块（如 `oauth`、`user`）禁止直接 `import` `internal/apps/admin/push` 或 `custom_events` 触发通知；应通过 `internal/listener` 发射域事件，由 push 模块在 bootstrap 阶段订阅。
- 编写依赖任务注册或推送事件同步的测试时，必须在测试 setup 中显式调用 `bootstrap.RegisterTasks()`、`bootstrap.RegisterPushDomainEvents()` 等，不得依赖 `init()` 副作用。
- API 错误响应必须通过 `response.Abort*` 中断请求，由 `ErrorHandlerMiddleware` 统一写出 JSON；禁止 `c.JSON(http.StatusOK, response.Err(...))` 及 Handler 直接 `c.JSON(status, response.Err(...))`。
- 严格禁止在任何更新、标记删除或数据导入流程中覆盖或修改已存在收藏记录的原始元数据（如 `Title`、`UserName`、`PageCount` 等字段）以及 `illust_json` 和 `novel_json`，只有在记录首次新建入库时方可写入。

## 项目介绍

### 技术栈

- 后端：Go 1.25+、Gin、GORM、PostgreSQL、可选 ClickHouse、Redis、Asynq、Cobra、Viper、Swaggo、OpenTelemetry、Zap、AWS SDK v2、Snowflake IDs。
- 前端：Next.js App Router、TypeScript、Tailwind CSS、pnpm、shadcn/ui。

## 开发要求

### 后端规则

命名规范：

- Go 包和文件使用小写蛇形命名（lowercase snake case）：如 `auth_source`、`postgres_logger.go`。
- 导出的 Go 标识符使用 PascalCase；未导出的标识符使用 camelCase。
- 请求/响应结构体使用 camelCase 并带有后缀，例如 `listUsersRequest` 和 `listUsersResponse`。
- 错误消息常量是 camelCase 字符串 `const`值，而不是包级别的 `error` 值。
- YAML 配置键使用小写蛇形命名（lowercase snake case）。

Handler 规范：

- Handler 命名为 动词 + 名词，例如 `ListUsers`。
- 使用 `ShouldBindQuery` 或 `ShouldBindJSON` 进行绑定。
- 每个 HTTP API 都需要有完整的 Swagger 注释；在 API 变更后运行 `make swagger`。

#### API 响应信封（统一格式）

所有 JSON API 响应的外层结构**必须**为：

```json
{ "error_msg": "", "data": ... }
```

- 成功时：`error_msg` 为空字符串，`data` 承载业务载荷。
- 失败时：`data` 为 `null`，`error_msg` 为用户可见的错误说明。
- 分页响应在 `data` 下使用 `{ "total": 0, "results": [] }`。

#### 成功响应（唯一写法）

成功时**始终**使用 HTTP `200`，由 Handler 直接写出 JSON：

```go
import (
    "net/http"
    "github.com/Rain-kl/Wavelet/internal/common/response"
    "github.com/gin-gonic/gin"
)

// 有数据
c.JSON(http.StatusOK, response.OK(data))

// 无数据（data 为 null）
c.JSON(http.StatusOK, response.OKNil())
```

#### 失败响应（中断请求，禁止直接写错误 JSON）

失败时**禁止**在 Handler / 中间件中直接调用 `c.JSON(..., response.Err(msg))`，也**禁止**用 HTTP `200` 携带非空 `error_msg` 表示失败。

统一通过 `internal/common/response` 的 **Abort 系列函数**中断请求。这些函数会将 `*response.APIError` 挂载到 Gin 的 `c.Errors` 链并 `c.Abort()`；请求结束后由全局 `response.ErrorHandlerMiddleware()`（在 `internal/router/middlewares.go` 中注册）统一写出 JSON，并记录到 OpenTelemetry Trace/Jaeger。

- **Logics 分工**：`logics.go` 只接受 `context.Context`，返回 `(result, error)`，严禁依赖 `*gin.Context` 或调用 `c.JSON`/`Abort*`。
- **错误日志**：底层错误在 Handler/Logic 边界用 `pkg/logger` 打印日志，禁止使用 `_ = ...` 静默吞掉关键错误。

**标准 Handler 模板：**

```go
func CreateWidget(c *gin.Context) {
    var req createWidgetRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.AbortBadRequest(c, errBindParamsFailed)
        return
    }

    widget, err := createWidgetLogic(c.Request.Context(), req)
    if err != nil {
        // 底层错误已记录日志时，向用户返回安全文案
        response.AbortBadRequest(c, err.Error()) // 或按语义选用 AbortConflict / AbortInternal 等
        return
    }

    c.JSON(http.StatusOK, response.OK(widget))
}
```

**中间件**与 Handler 遵循同一规则。参考 `oauth.LoginRequired()` → `AbortUnauthorized`，`admin.LoginAdminRequired()` → `AbortNotFound`，`cap.VerifyMiddleware` → `AbortUnauthorized`。

#### 错误消息定义

- 面向用户的错误文案定义为模块内 **camelCase 字符串常量**（放在 `errs.go`），例如 `errBindParamsFailed = "参数绑定失败"`。
- Handler / 中间件向 Abort 函数传入这些常量或经校验的安全字符串；**禁止**将数据库驱动错误、堆栈信息等内部细节直接暴露给客户端。
- `response.Err(msg)` 仅供 `ErrorHandlerMiddleware` 内部构造 JSON，**业务代码不得直接用于 `c.JSON`**。

#### `logics.go` 与 Handler 的分工

- `logics.go` 接受 `context.Context`，返回 `(result, error)` 或带状态的业务结果结构体（参考 `internal/apps/user/logics.go` 的 `LoginEmailVerificationResult`）。
- `logics.go` **不得**依赖 `*gin.Context`，**不得**调用 `response.Abort*` 或 `c.JSON`。
- Handler 负责：绑定参数 → 调用 logic → 将 logic 错误/状态映射为对应的 `Abort*` 或 `response.OK`。

#### 日志与内部错误

- 数据库、Redis、第三方 API、文件 I/O 等**运行时错误**：在 Handler 或 logic 边界用 `pkg/logger` 记录（带 `ctx`），再向用户返回安全的 `AbortInternal` 或语义匹配的业务错误常量。
- 任何关键错误在被吞掉、转换为通用响应，或由后台 worker 忽略之前，都必须通过 `pkg/logger` 打印日志。
- 禁止用 `_ = ...` 静默丢弃重要错误。如果某个错误因为 best-effort 操作或确认无害而需要忽略，必须添加简短注释说明原因。
- 避免重复刷日志：在真正处理或抑制错误的边界记录一次，然后 `Abort*` 或成功返回。

#### 禁止写法（反模式）

```go
// ❌ 禁止：HTTP 200 表示失败
c.JSON(http.StatusOK, response.Err("密码错误"))

// ❌ 禁止：Handler 直接写错误 JSON，绕过 ErrorHandlerMiddleware 与 OTel 记录
c.JSON(http.StatusBadRequest, response.Err("参数错误"))

// ❌ 禁止：gin.H 手写错误体
c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error_msg": "...", "data": nil})

// ❌ 禁止：logics.go 中中断 HTTP 请求
func doSomething(c *gin.Context) { response.AbortBadRequest(c, "...") }
```

#### Swagger 注释约定

- `@Success 200` 的 `data` 使用具体类型或 `response.Any`。
- 对每个可能返回的 Abort 状态码声明 `@Failure`，例如 `@Failure 400 {object} response.Any "参数错误"`、`@Failure 401 {object} response.Any "未登录"`。

路由与模块：

- 仅在 `internal/router/router.go` 中作为统一高层入口进行路由分发委派，不允许在 `router.go` 中直接挂载业务 Handler。
- 关于所有的路由归属划分、接口开发隔离防线以及详细的注册和开发步骤，请直接阅读并严格遵循 [new-api](file:///Users/ryan/DEV/Go/Wavelet/.claude/skills/new-api/SKILL.md) 技能。

应用装配与跨模块集成：

- 新增跨模块副作用（任务注册、推送订阅、后台监听器）时，在 `internal/bootstrap/bootstrap.go` 增加 `Register*` 函数，并在对应 `internal/cmd/*.go` 入口调用；参考现有 `RegisterAPI` / `RegisterWorker` / `RegisterAll` 分工。
- `bootstrap.Init` 必须在 `RegisterPushDomainEvents()` 之后调用（API/`all` 模式），以确保 `SyncEvents` 能同步内置推送事件元数据。
- Handler 与业务逻辑分离：HTTP Handler 负责绑定与响应；可复用逻辑放入 `logics.go`（接受 `context.Context`，不依赖 `*gin.Context`），便于 Worker 与单元测试复用。参考 `internal/apps/user/logics.go`。

中间件：

- 全局中间件属于路由设置：`gin.Recovery()`、`otelgin.Middleware()`、日志中间件 and session 中间件。
- 对于登录路由组，使用 `oauth.LoginRequired()`。
- 对于管理路由组，使用 `admin.LoginAdminRequired()`。

配置管理：

- 运行时代码从 `config.Config` 中读取配置，绝对不要直接从 `os.Getenv()` 中读取。
- 当添加配置时，同时更新 `config.example.yaml` and `internal/config/model.go`。

数据库操作：

- 简单查询可以直接从 model 层使用 GORM。
- 管理员代码应首选 `db.DB(ctx)` 以获得链路追踪感知的 DB 访问。
- 不要在 Handler 中放置复杂的 SQL；将其移至 `internal/model/` 或模块内的业务服务层（如 `internal/apps/<module>/service.go` 或 `logics.go`）。
- 在 `internal/db/migrator/goose/` 下使用 goose SQL 迁移；不要添加基于 GORM AutoMigrate 的 Schema 升级。
- 不要创建物理数据库外键。改为关系字段添加显式索引。
- 数据库默认值必须与 Go 模型零值（`nil`、`0`、`false`、`""`）匹配，以避免意外的插入。

### 前端规则

在进行任何 Next.js 工作之前，请在 `node_modules/next/dist/docs/` 中找到并阅读相关文档。您的训练数据已过时 —— 这些文档是唯一的真理来源。

请直接查看并参考项目提供的示例和 Demo 代码：[frontend/app/(main)/admin/demo](file:///Users/ryan/DEV/Go/Wavelet/frontend/app/(main)/admin/demo)。

样式规范：

- shadcn/ui 基础组件应该使用它们的 `variant` 系统和全局 CSS 变量。当组件的变体（variant）应该拥有某种外观时，不要在业务 `className` 中硬编码颜色、背景或阴影。
- 如果现有的变体不足以满足需求，请扩展 shadcn/ui 组件的变体，而不是硬编码一次性的颜色。

页面标题栏规范 (新人开发与重构必读)：

- **容器与对齐机制**：
    - 标题容器统一使用 `flex items-center gap-2`。如果右侧有操作按钮（如“新增”、“刷新”），请使用 `justify-between` 布局让操作区与标题双向分布。
    - 为了确保所有页面在进入/切换时，顶部的呼吸感和视觉高度完全一致，页面最外层容器**必须**统一使用 `py-6 px-1` 或 `py-6` 进行上边距对齐。
- **图标标准**：图标作为视觉辅助点缀，**必须**直接嵌套在标题容器中，直接使用 Lucide 图标组件，样式大小限制为 `size-5 text-primary`。**严禁**为图标包裹任何背景小卡片、圆角边框或额外的修饰容器。
- **标题文字标准**：标题文字使用且仅使用 `h1 className="text-2xl font-semibold tracking-tight"`。不要自行定义字号、字量（如使用 `font-bold`）或添加任何渐变色，保持整个系统的字形规范化。
- **Tabs 模块化与文件拆分规范**：凡是带有多个 Tab 页切换的复杂页面，**禁止**将所有 Tab 的渲染逻辑堆积在同一个主文件内。每个 Tab 的具体渲染内容必须单独拆分为独立的 React 组件文件（如 `tabs/events-tab.tsx`）。主页面文件应该仅用于导入子组件、注册 Tabs 触发器以及管理 Tab 的切换激活状态。这有利于防止单文件过大（避免单文件行数超过 600 行限制），并大幅度提高代码的可读性与编译维护效率。
- **扁平化结构与避免冗余中间件**：为了消除无意义的“中间代理文件”，所有作为路由物理入口的 Tabs 状态维护、骨架及外层布局代码，**必须**直接定义在 Next.js 的 `app/` 页面文件（即 `page.tsx`）中。禁止在 `page.tsx` 中仅写一个单纯的 `<AnotherComponent />` 转发，而在外部新建一个同名中转容器。
- **复杂度驱动的组件拆分规范**：组件的拆分不应局限于“跨页面复用”。当一个路由页面的复杂度变高时（如渲染逻辑膨胀、存在大型嵌套弹窗或多层状态管理，如单文件代码行数超过 600 行），必须主动将其拆分为子组件以维持单文件的高可读性与低耦合度。拆分时遵循就近原则：特定于该路由且不复用的子组件应放置在最邻近该路由的特征目录（Feature Folder，如 `components/` 局部文件夹）中；只有真正具备跨页面复用价值的通用业务/基础 UI 组件才应存放在全局 `components/` 共享目录下。
    - **最佳实践标杆案例（数据管理 `/admin/database`）**：
      该页面由于整合了“运行状态概览”、“物理表网格浏览器”、“磁盘缓存管理”和“SQL 交互控台”多个复杂大区块，重构前单文件接近 1000 行。
      重构后，主页面 `page.tsx` 仅做高级页面骨架与排版排布，维护全局刷新机制与终端视图切换；而“数据表浏览器 (`table-browser.tsx`)”、“缓存管理 (`cache-manager.tsx`)”与“SQL 终端 (`sql-console.tsx`)”等独立高状态密度区块均被抽离为局部子组件，存放在 `frontend/app/(main)/admin/database/components/`。这保证了代码结构层次清晰、单文件小巧好维护。所有复杂页面的新开发和重构必须遵循此模式。

页面宽度：

- 页面根容器必须支持全宽。使用 `w-full`。
- 不要硬编码页面级的最大宽度，如 `max-w-6xl` 或 `max-w-4xl`；主布局（main layout）拥有正常/全宽的限制。

组件放置：

- 跨页面的业务组件属于 `frontend/components/common/`。
- shadcn/ui 原生组件（primitives）属于 `frontend/components/ui/`。
- 特定于路由/页面的组件放在最邻近的特征（feature）目录中。

服务类（Services）：

- 前端 API 访问通过服务类和导出的 `services` 对象进行。
- 新增服务结构如下：

```text
frontend/lib/services/<service-name>/
  types.ts
  <service-name>.service.ts
  index.ts
```

- 服务类继承 `BaseService`，定义 `basePath`，并暴露有类型的静态方法。
- 在 `frontend/lib/services/index.ts` 中注册新服务。

