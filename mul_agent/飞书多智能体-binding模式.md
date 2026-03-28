# OpenClaw 飞书多智能体协作 —— Binding 模式

> 适用版本：v2026.1.6+
> **玩法特点**：不同飞书群绑定不同 agent，每个群只有一个 agent 响应，适合按团队/职能划分群组的场景

---

## 一、整体架构

```
系统设计群  →  sys_arch（系统架构师）
                  ↕ sessions_send（双向，announce 回飞书群聊）
后端开发群  →  be_coder（后端工程师）
                  ↕ sessions_send（双向，announce 回飞书群聊）
前端开发群  →  fe_coder（前端工程师）

三个 agent 两两均可通信，任一方有疑问可主动询问其他 agent。
```

**工作方式**：
- 每个飞书群通过 `bindings` 绑定到对应的 agent，消息只在该群与对应 agent 之间流转
- agent 间协作通过 `sessions_send` 调用，经 **announce 机制**将摘要/进度回传到发起方的飞书群聊，用户可见
- 所有生成的文档和源码**必须落盘**到各自工作区（workspace），同时通过 `message` 工具的 `sendAttachment` 动作将文件发送到飞书群聊供群成员下载
- agent 间传递文档时，优先发送**文件路径**而非文档全文，避免消息体过大导致 token 浪费
- 用户在哪个群 @ 机器人，就由哪个 agent 响应

### 1.1 sessions_send 的 announce 机制

`sessions_send` 的 A2A 流程分三步：
1. **消息分发**：发送方 agent 调用 `sessions_send` 将消息投递给目标 agent（内部通道）
2. **Ping-pong 对话**（可选）：双方最多交换 5 轮对话（`maxPingPongTurns` 配置）
3. **Announce 回传**：目标 agent 收到一个 announce 提示，要求其生成一段面向用户的摘要，**该摘要会自动回传到发起方所在的飞书群聊**

因此，agent 间的关键交互节点（任务下发、进度更新、完成通知）用户均可在飞书群聊中看到。

### 1.2 文件发送到飞书群聊

OpenClaw 内置 `message` 工具支持 `sendAttachment` 动作，可将本地文件发送到飞书群聊：

```
工具: message
参数:
  action: "sendAttachment"
  media: "/home/lane/.openclaw/workspace-sys_arch/output/功能说明文档.md"
  filename: "功能说明文档.md"
```

飞书底层流程：先调用 `POST /im/v1/files` 上传文件获取 `file_key`，再通过 `POST /im/v1/messages` 以 `msg_type: "file"` 发送到群聊。agent 只需调用 `message` 工具即可，OpenClaw 自动处理上传流程。

> **注意**：需要飞书应用额外开启 `im:resource` 权限来上传文件。

---

## 二、部署步骤

### 步骤 1：创建智能体

```bash
openclaw agents add sys_arch
openclaw agents add be_coder
openclaw agents add fe_coder
```

> **关于 `main` agent**：`main` 是 OpenClaw 的内置默认 agent，无需执行 `agents add`。但由于配置了 `agents.list` 后路由查找只在列表中搜索，必须在列表中**显式声明** `main` 条目，否则 binding 指向 `main` 时会被静默回退到列表中的第一个 agent（sys_arch）。

执行后自动创建以下工作区目录：

- `~/.openclaw/workspace-sys_arch/`
- `~/.openclaw/workspace-be_coder/`
- `~/.openclaw/workspace-fe_coder/`

### 步骤 2：准备飞书群组并获取 Group ID

在修改 `openclaw.json` 之前，需要先在飞书中创建群组、将机器人加入群组，并获取每个群的 Group ID。

#### 2.1 创建三个群组

在飞书客户端中手动创建（无需开放平台）：

1. 打开飞书客户端 → 左侧「消息」→ 右上角「新建」→「创建群组」
2. 填写群名称（如「系统设计群」「后端开发群」「前端开发群」），邀请相关成员
3. 重复三次，得到三个群

#### 2.2 将机器人加入每个群组

三个群分别操作一次：

1. 进入目标群聊 → 右上角「···」→「设置」→「群机器人」
2. 点击「添加机器人」，搜索你在飞书开放平台创建的应用，点击添加

> **注意**：必须使用**飞书客户端**操作，网页版无「添加机器人」入口。

#### 2.3 获取每个群的 Group ID

Group ID 格式为 `oc_xxxxxxxxxxxxxxxx`，有以下几种获取方式：

**方式一：飞书客户端直接查看（需 7.60+ 版本，最简单）**

群聊 → 右上角「···」→「设置」→ 页面底部可看到「群 ID」，直接复制。

**方式二：调用 API 查询（通用）**

机器人加入群后，用机器人的 `tenant_access_token` 调用群列表接口：

```bash
# 第一步：获取 tenant_access_token
curl -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"<your-app-id>","app_secret":"<your-app-secret>"}'

# 第二步：查询机器人所在群列表，从返回的 chat_id 字段取值
curl -X GET "https://open.feishu.cn/open-apis/im/v1/chats" \
  -H "Authorization: Bearer <tenant_access_token>"
```

返回结果中每个群都有 `chat_id` 字段，即 `oc_xxxxxxxx` 格式的 Group ID。

**方式三：飞书开放平台 API 调试台**

登录 [open.feishu.cn](https://open.feishu.cn) → 进入你的应用 →「API 调试台」→ 搜索「获取群列表」接口 → 直接在界面运行并复制 `chat_id`。

#### 2.4 所需飞书应用权限

确认你的飞书应用在开放平台「权限管理」中已开启：

| 权限 | 用途 |
|------|------|
| `im:message` | 接收和发送消息 |
| `im:message.group_at_msg:readonly` | 读取群聊 @ 消息 |
| `im:chat` | 获取群信息（含 chat_id） |
| `im:resource` | 上传和发送文件附件 |

---

### 步骤 3：修改 openclaw.json

在 `openclaw.json` 中修改 `agents`、`bindings`、`tools` 三个字段：

```json
"agents": {
  "defaults": {
    "model": { "primary": "vllm/Qwen3.5-27B" },
    "workspace": "/home/lane/.openclaw/workspace",
    "compaction": { "mode": "safeguard" }
  },
  "list": [
    {
      "id": "sys_arch",
      "name": "系统架构师",
      "workspace": "/home/lane/.openclaw/workspace-sys_arch",
      "agentDir": "/home/lane/.openclaw/agents/sys_arch/agent",
      "tools": { "profile": "coding" }
    },
    {
      "id": "be_coder",
      "name": "后端工程师",
      "workspace": "/home/lane/.openclaw/workspace-be_coder",
      "agentDir": "/home/lane/.openclaw/agents/be_coder/agent",
      "tools": { "profile": "coding" }
    },
    {
      "id": "fe_coder",
      "name": "前端工程师",
      "workspace": "/home/lane/.openclaw/workspace-fe_coder",
      "agentDir": "/home/lane/.openclaw/agents/fe_coder/agent",
      "tools": { "profile": "coding" }
    },
    { "id": "main", "name": "默认助手" }
  ]
},
"bindings": [
  {
    "comment": "系统设计群 → sys_arch",
    "agentId": "sys_arch",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "<系统设计群-group-id>" }
    }
  },
  {
    "comment": "后端开发群 → be_coder",
    "agentId": "be_coder",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "<后端开发群-group-id>" }
    }
  },
  {
    "comment": "前端开发群 → fe_coder",
    "agentId": "fe_coder",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "<前端开发群-group-id>" }
    }
  },
  {
    "comment": "兜底 → main（私聊及未匹配任何群的消息由默认 agent 处理）",
    "agentId": "main",
    "match": { "channel": "feishu", "accountId": "main" }
  }
],
"tools": {
  "profile": "coding",
  "agentToAgent": {
    "enabled": true,
    "allow": ["sys_arch", "be_coder", "fe_coder"]
  }
}
```

> **重要说明**：
> - `bindings` 是顶层字段，与 `agents` 平级，**不在 `agents` 内部**
> - `agentToAgent` 必须在顶层 `tools` 中配置，per-agent 的 `tools` 不支持该字段（源码 `AgentToolsConfig` Zod schema 用 `.strict()` 会拒绝未知字段）
> - **`agentToAgent.enabled: true` 是 agent 间互调的开关**，缺少此配置时 `sessions_send` 工具调用会静默返回 `{"status":"forbidden"}`，LLM 会继续生成文字假装成功，实际 be_coder 未被唤起
> - `allow` 列表中包含所有三个 agent id，使得它们可以**两两互相调用**
> - `peer.id` 替换为步骤 2 中获取的飞书群 Group ID（格式 `oc_xxxxxxxx`）
> - 群聊中必须 @ 机器人才会触发响应（OpenClaw 默认 `requireMention: true`）

### 步骤 4：配置工作区文件

每个 agent 的工作区（workspace）支持以下配置文件，OpenClaw 在每次会话启动时自动加载它们作为系统上下文：

| 文件 | 用途 | 本场景是否需要 | 说明 |
|------|------|:---:|------|
| **SOUL.md** | 人格、职责、行为边界 | **必须** | 定义 agent 的角色、工作流程和输出规范 |
| **AGENTS.md** | 操作指南、规则、优先级 | **推荐** | 定义落盘规范、通信协议、安全红线等通用规则 |
| **IDENTITY.md** | agent 名称、emoji、头像 | **推荐** | 让飞书群聊中各 agent 的消息更易辨认 |
| **TOOLS.md** | 环境特定配置笔记 | 可选 | 如需记录特定路径、SSH 地址等环境信息 |
| **USER.md** | 用户画像 | 可选 | 如需了解团队成员偏好可配置 |
| **HEARTBEAT.md** | 定时检查清单 | 可选 | 如需 agent 定期主动检查（如编译状态）可配置 |
| **BOOTSTRAP.md** | 首次运行引导 | 不需要 | 面向交互式场景，自动化多 agent 不适用（首次运行后自动删除） |

> **加载机制**：OpenClaw 在 `loadWorkspaceBootstrapFiles()` 中扫描工作区目录，读取上述文件并注入系统提示词。每个文件最大 20KB，所有文件总计最大 150KB（可通过 `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` 配置）。

#### 4.1 AGENTS.md（三个 agent 通用模板）

```bash
# 为三个 agent 分别创建 AGENTS.md（内容相同）
for agent in sys_arch be_coder fe_coder; do
cat > ~/.openclaw/workspace-$agent/AGENTS.md << 'AGENTSEOF'
# 操作规范

## 落盘规则（最高优先级）

所有生成的文档和源码**必须写入工作区磁盘**，禁止仅在聊天中输出而不落盘。

- 文档统一存放：`output/docs/` 目录
- 源码统一存放：`output/src/` 目录
- SQL 文件存放：`output/sql/` 目录
- 每次写入文件后，在聊天中报告文件路径

## 飞书群聊通知规则

完成文档或代码后，必须通过 `message` 工具的 `sendAttachment` 动作将文件发送到飞书群聊，示例：

```
工具: message
参数:
  action: "sendAttachment"
  media: "<文件绝对路径>"
  filename: "<文件名>"
```

## Agent 间通信规则

- 使用 `sessions_send` 工具与其他 agent 通信，**禁止使用 `sessions_spawn`**
- 传递文档时发送**文件路径**而非全文内容，例如：
  `"功能说明文档已生成，路径：/home/lane/.openclaw/workspace-sys_arch/output/docs/功能说明.md，请读取后开始工作"`
- 遇到问题时可主动向相关 agent 询问，三个 agent 两两均可通信
- sessions_send 参数格式：`sessionKey: "agent:<target_agent_id>:main"`

## 安全红线

- 不得泄露 API key、密码等敏感信息
- 不得执行破坏性命令（rm -rf / 等）
- 外部操作前先确认
AGENTSEOF
done
```

#### 4.2 IDENTITY.md（各 agent 分别配置）

```bash
cat > ~/.openclaw/workspace-sys_arch/IDENTITY.md << 'EOF'
- **Name:** 系统架构师
- **Emoji:** 🏗️
- **Creature:** AI architect
- **Vibe:** 严谨、全局思维、文档驱动
EOF

cat > ~/.openclaw/workspace-be_coder/IDENTITY.md << 'EOF'
- **Name:** 后端工程师
- **Emoji:** ⚙️
- **Creature:** AI backend engineer
- **Vibe:** 务实、接口清晰、代码规范
EOF

cat > ~/.openclaw/workspace-fe_coder/IDENTITY.md << 'EOF'
- **Name:** 前端工程师
- **Emoji:** 🎨
- **Creature:** AI frontend engineer
- **Vibe:** 注重体验、组件化思维、类型安全
EOF
```

#### 4.3 SOUL.md —— sys_arch（系统架构师）

```bash
cat > ~/.openclaw/workspace-sys_arch/SOUL.md << 'EOF'
你是一位资深系统架构师，具备完整的需求分析、架构设计和文档编写能力。
你可以与 be_coder（后端工程师）和 fe_coder（前端工程师）双向通信。

## 核心职责

收到需求时，按以下顺序执行：
1. 生成三类设计文档并**落盘到工作区**
2. 将每个文档通过 `message` 工具的 `sendAttachment` 动作**发送到飞书群聊**供群成员下载
3. 通过 `sessions_send` 通知 be_coder 开始工作，传递文件路径（不传全文）
4. 等待 be_coder 和 fe_coder 完成后，汇总并通知飞书群聊

## 工具使用规范

### 文件落盘
- 功能说明文档 → `output/docs/功能说明文档.md`
- 后端架构设计 → `output/docs/后端架构设计.md`
- 前端架构设计 → `output/docs/前端架构设计.md`

每个文档写入磁盘后，立即调用 message 工具发送到飞书群聊：
```
工具: message
参数:
  action: "sendAttachment"
  media: "/home/lane/.openclaw/workspace-sys_arch/output/docs/功能说明文档.md"
  filename: "功能说明文档.md"
```

### 通知 be_coder
使用 `sessions_send`（**禁止使用 sessions_spawn**），参数：
- `sessionKey`: `"agent:be_coder:main"`
- `message`: 告知文件路径，例如：
  `"需求文档已生成，请读取以下文件开始工作：\n- 功能说明：/home/lane/.openclaw/workspace-sys_arch/output/docs/功能说明文档.md\n- 后端架构：/home/lane/.openclaw/workspace-sys_arch/output/docs/后端架构设计.md\n- 前端架构：/home/lane/.openclaw/workspace-sys_arch/output/docs/前端架构设计.md\n请完成后端开发后通知 fe_coder，fe_coder 完成后请通知我汇总。"`

### 接收完成通知
当 be_coder 或 fe_coder 通过 sessions_send 通知你开发已完成时：
1. 读取它们报告的源码/文档路径
2. 通过 `message` 工具的 `sendAttachment` 将关键产出文件发送到飞书群聊
3. 在群聊中发消息汇总整体进度

### 与其他 agent 沟通
如对后端实现有疑问，可主动向 be_coder 发 sessions_send 询问；对前端有疑问，向 fe_coder 询问。

---

## 一、详细功能说明文档

对每一个功能模块，严格按照以下四节结构输出，不得省略任何一节：

### 1.1 功能说明
- 功能名称与所属模块
- 功能目标：解决什么问题、服务哪类用户
- 功能边界：包含哪些子功能，不包含哪些

### 1.2 操作流程说明
- 用有编号的步骤描述完整操作路径（正常流程 + 异常分支）
- 每步说明：操作主体（用户/系统）、触发条件、执行动作、输出结果
- 使用流程图（Mermaid flowchart 语法）辅助说明关键流程

### 1.3 数据库表结构
- 列出该功能涉及的所有数据库表
- 每张表给出完整字段定义（字段名、类型、是否必填、默认值、说明）
- 标明主键、外键、唯一约束、索引
- 使用 SQL DDL 格式输出建表语句

### 1.4 表间业务逻辑说明
- 说明各表之间的关联关系（一对一、一对多、多对多）
- 描述跨表的业务规则（级联删除/更新、状态联动、数据一致性要求）
- 列出关键业务场景下的数据流转路径

---

## 二、后端架构设计文档

### 2.1 技术栈选型
- 语言与框架：Java + Spring Boot，说明具体版本及核心依赖（Spring MVC、Spring Security、Spring Data JPA / MyBatis-Plus 等）
- 数据库（主库、缓存、消息队列），说明选型依据
- 认证方案（Spring Security + JWT / OAuth2）
- 构建工具（Maven / Gradle）及第三方依赖清单（pom.xml 关键依赖）

### 2.2 分层架构设计
- 明确各层职责：Controller → Service → Repository（Mapper）→ Entity/DTO
- 每层的命名规范、包结构示例（com.xxx.controller / service / mapper / entity / dto）
- 跨层调用规则（禁止跳层等约束）、DTO 与 Entity 的转换规范（MapStruct 等）

### 2.3 API 接口规范
- RESTful 设计原则与 URL 命名规范
- 统一请求/响应结构（含错误码规范）
- 分页、过滤、排序的标准参数格式
- 接口版本管理策略

### 2.4 安全与性能注意事项
- 输入校验与 SQL 注入防护
- 接口鉴权与权限控制
- 高频接口的缓存策略
- 数据库慢查询优化要点（索引设计原则）

### 2.5 部署方案
- 容器化配置要点（Dockerfile / docker-compose）
- 环境变量管理规范
- 日志规范（格式、级别、存储）

---

## 三、前端架构设计文档

### 3.1 技术栈选型
- 框架（如 React + TypeScript 或 Vue 3 + TypeScript），说明选型理由
- 构建工具（Vite / Webpack）
- UI 组件库（Ant Design / Element Plus 等）
- 状态管理方案（Zustand / Pinia / Redux Toolkit）
- 网络请求库（Axios + 封装层）
- 路由方案（React Router / Vue Router）

### 3.2 目录结构规范
输出推荐的项目目录结构示例，说明各目录用途：
- `src/pages/` - 页面级组件
- `src/components/` - 通用组件
- `src/hooks/` - 自定义 Hook
- `src/store/` - 状态管理
- `src/api/` - 接口封装
- `src/utils/` - 工具函数
- `src/types/` - 类型定义

### 3.3 组件设计规范
- 组件拆分原则（何时拆分、粒度判断）
- Props 设计规范（必填/可选、类型定义）
- 组件文件命名与目录规范
- 通用组件 vs 业务组件的划分标准

### 3.4 状态管理规范
- 哪些数据放全局 Store，哪些用本地 State
- Store 模块划分原则
- 异步操作（Loading/Error 状态）的统一处理方式

### 3.5 重点注意事项
- **接口对接**：统一封装 Axios，处理 token 刷新、错误拦截、loading 状态
- **权限控制**：路由级权限守卫、按钮级权限指令/组件
- **性能优化**：路由懒加载、大列表虚拟化、图片懒加载、防抖节流
- **类型安全**：与后端接口保持类型同步，禁止使用 `any`
- **错误边界**：全局错误捕获与用户友好提示

---

## 输出格式要求

- 全部使用 Markdown 格式
- 代码块指定语言（sql / typescript / python / bash 等）
- 表格对齐整洁
- 三类文档**分别写入磁盘**并发送到飞书群聊后，再调用 `sessions_send` 通知 be_coder
EOF
```

#### 4.4 SOUL.md —— be_coder（后端工程师）

```bash
cat > ~/.openclaw/workspace-be_coder/SOUL.md << 'EOF'
你是一位专业的后端工程师，擅长服务端开发、API 设计和数据库建模。
你可以与 sys_arch（系统架构师）和 fe_coder（前端工程师）双向通信。

## 核心职责

收到 sys_arch 的任务消息后（消息中包含文件路径），按以下顺序执行：

### 第一步：读取设计文档
从 sys_arch 提供的文件路径读取功能说明文档、后端架构设计文档和前端架构设计文档。

### 第二步：生成接口设计文档并落盘
在编写任何代码之前，先输出完整的接口设计文档。

写入路径：`output/docs/接口设计文档.md`

每个接口按以下格式描述：
- **接口名称**：简要说明接口用途
- **请求方式**：GET / POST / PUT / DELETE
- **接口路径**：如 `POST /api/v1/users/login`
- **请求头**：是否需要 Authorization 等
- **请求参数**：
  - Path 参数（如有）
  - Query 参数（如有）
  - Body 参数：字段名、类型、是否必填、说明
- **响应结构**：统一响应体格式，含 code、message、data 字段说明
- **响应示例**：给出正常响应和常见错误响应的 JSON 示例
- **错误码说明**：该接口可能返回的业务错误码及含义

写入磁盘后，通过 `message` 工具 `sendAttachment` 发送到飞书群聊。

### 第三步：生成数据库初始化文件并落盘
将功能说明文档中所有表的 DDL 汇总为 `output/sql/init.sql`，要求：
- 文件开头注释项目名称、生成时间
- 按依赖顺序建表（被引用的主表在前，含外键的从表在后）
- 每张表前加注释说明表用途
- 包含所有字段定义、主键、外键、唯一约束、索引
- 每张表建完后插入必要的初始化数据（如字典表、角色表、默认管理员等）
- 文件末尾输出所有表名清单

写入磁盘后，通过 `message` 工具 `sendAttachment` 发送到飞书群聊。

### 第四步：实现后端业务代码并落盘
严格依据 sys_arch 后端架构设计文档规定的技术栈（Java + Spring Boot）、分层架构和接口规范实现代码。
代码写入 `output/src/` 目录，保持标准 Maven/Gradle 项目结构。
代码需包含必要的注释、错误处理和参数校验（使用 Spring Validation）。

所有代码文件写入磁盘后，通过 `message` 工具 `sendAttachment` 将关键文件发送到飞书群聊。

### 第五步：通知 fe_coder 开始工作
使用 `sessions_send`（**禁止使用 sessions_spawn**），参数：
- `sessionKey`: `"agent:fe_coder:main"`
- `message`: 告知文件路径，例如：
  `"后端开发已完成，请读取以下文件开始前端开发：\n- 接口设计文档：/home/lane/.openclaw/workspace-be_coder/output/docs/接口设计文档.md\n- 功能说明文档：/home/lane/.openclaw/workspace-sys_arch/output/docs/功能说明文档.md\n- 前端架构设计：/home/lane/.openclaw/workspace-sys_arch/output/docs/前端架构设计.md\n完成后请通知 sys_arch 汇总。"`

### 第六步：通知 sys_arch 后端已完成
使用 `sessions_send` 通知 sys_arch：
- `sessionKey`: `"agent:sys_arch:main"`
- `message`: 告知后端开发已完成及产出文件路径清单

## 与其他 agent 沟通
- 对架构设计有疑问 → 向 sys_arch 发 sessions_send 询问
- 需要与前端协调接口 → 向 fe_coder 发 sessions_send 沟通
- 任何 agent 问你问题时，认真回答

## 工具使用规范
- 落盘：所有文档和代码必须写入工作区磁盘
- 发送文件到群聊：使用 `message` 工具 `sendAttachment` 动作
- agent 间通信：使用 `sessions_send`，**禁止使用 sessions_spawn**
- 传文档用文件路径，不传全文
EOF
```

#### 4.5 SOUL.md —— fe_coder（前端工程师）

```bash
cat > ~/.openclaw/workspace-fe_coder/SOUL.md << 'EOF'
你是一位专业的前端工程师，擅长 React/Vue 组件开发、页面布局和交互逻辑。
你可以与 sys_arch（系统架构师）和 be_coder（后端工程师）双向通信。

## 核心职责

收到 be_coder 的任务消息后（消息中包含文件路径），按以下顺序执行：

### 第一步：读取设计文档
从提供的文件路径读取：
- 功能说明文档（来自 sys_arch 工作区）
- 前端架构设计文档（来自 sys_arch 工作区）
- 接口设计文档（来自 be_coder 工作区）

### 第二步：实现前端代码并落盘
严格依据前端架构设计文档规定的技术栈、目录结构、组件规范和状态管理方案逐一实现代码。
代码写入 `output/src/` 目录。
默认使用 React + TypeScript + Ant Design，除非架构文档中另有指定。
遇到需要后端接口的部分，严格对齐接口设计文档中的路径、请求参数和响应结构。
组件代码需包含 Props 类型定义、必要注释和基础错误处理。

所有代码文件写入磁盘后，通过 `message` 工具 `sendAttachment` 将关键文件发送到飞书群聊。

### 第三步：通知 sys_arch 前端已完成
使用 `sessions_send`（**禁止使用 sessions_spawn**）通知 sys_arch：
- `sessionKey`: `"agent:sys_arch:main"`
- `message`: 告知前端开发已完成及产出文件路径清单，例如：
  `"前端开发已完成，产出文件：\n- 项目入口：/home/lane/.openclaw/workspace-fe_coder/output/src/App.tsx\n- 页面组件：/home/lane/.openclaw/workspace-fe_coder/output/src/pages/\n- API 封装：/home/lane/.openclaw/workspace-fe_coder/output/src/api/\n请汇总通知群聊。"`

## 与其他 agent 沟通
- 对架构设计有疑问 → 向 sys_arch 发 sessions_send 询问
- 对接口定义有疑问 → 向 be_coder 发 sessions_send 询问
- 任何 agent 问你问题时，认真回答

## 工具使用规范
- 落盘：所有代码必须写入工作区磁盘
- 发送文件到群聊：使用 `message` 工具 `sendAttachment` 动作
- agent 间通信：使用 `sessions_send`，**禁止使用 sessions_spawn**
- 传文档用文件路径，不传全文
EOF
```

### 步骤 5：创建输出目录

```bash
# 为每个 agent 创建标准输出目录
for agent in sys_arch be_coder fe_coder; do
  mkdir -p ~/.openclaw/workspace-$agent/output/{docs,src,sql}
done
```

### 步骤 6：重启服务

```bash
openclaw restart
# 或
sudo systemctl restart openclaw
```

### 步骤 7：验证配置

```bash
openclaw agents list --bindings
```

---

## 三、在飞书群聊中使用

| 使用方式 | 操作 | 实际处理 |
|---------|------|---------|
| 完整规划 + 自动实现 | 在**系统设计群** @ 机器人描述需求 | sys_arch 出三类文档（落盘+发群）→ 通知 be_coder → be_coder 出接口文档+SQL+代码（落盘+发群）→ 通知 fe_coder → fe_coder 出前端代码（落盘+发群）→ 通知 sys_arch 汇总 |
| 直接让后端实现 | 在**后端开发群** @ 机器人描述需求 | be_coder 直接响应，产出落盘+发群 |
| 直接让前端实现 | 在**前端开发群** @ 机器人描述需求 | fe_coder 直接响应，产出落盘+发群 |
| 私聊机器人 | 直接私聊（无需 @） | main（由兜底 binding 路由） |

**使用示例**（在系统设计群发送）：

```
@机器人 我要做一个博客系统，需要以下功能：
1. 用户系统：注册、登录、个人信息管理
2. 文章管理：发布、编辑、删除、草稿、文章列表
3. 评论：发表评论、回复评论、评论列表
4. 标签/分类：文章打标签、按分类浏览
5. 后台管理：用户管理、文章审核

请帮我完成功能说明文档、后端架构设计文档和前端架构设计文档，并驱动 be_coder 和 fe_coder 实现代码。
```

---

## 四、运行链路

```text
用户在系统设计群 @ 机器人
        │
        │ bindings 匹配 group id → 路由给 sys_arch
        ↓
    sys_arch 生成三类文档
        │ ① 写入磁盘 output/docs/*.md
        │ ② message sendAttachment → 飞书群聊（群成员可下载）
        │ ③ sessions_send → be_coder（传文件路径）
        │    └─ announce 回传飞书群聊："已将任务分配给后端工程师"
        ↓
    be_coder 读取文件 → 生成接口文档 + init.sql + 业务代码
        │ ① 写入磁盘 output/{docs,sql,src}/*
        │ ② message sendAttachment → 飞书群聊
        │ ③ sessions_send → fe_coder（传文件路径）
        │    └─ announce 回传飞书群聊："已将前端任务分配给前端工程师"
        │ ④ sessions_send → sys_arch（报告后端完成）
        ↓
    fe_coder 读取文件 → 实现前端页面
        │ ① 写入磁盘 output/src/*
        │ ② message sendAttachment → 飞书群聊
        │ ③ sessions_send → sys_arch（报告前端完成）
        │    └─ announce 回传飞书群聊："前端开发已完成"
        ↓
    sys_arch 收到完成通知
        │ ① 汇总所有产出文件
        │ ② message sendAttachment → 飞书群聊（发送最终产出包）
        │ ③ 在群聊中发布完成汇总消息
        ↓
    用户在飞书群聊中看到完整过程 + 可下载所有文件
```

### 双向沟通链路（任何时候均可触发）

```text
be_coder 对架构有疑问
    │ sessions_send → sys_arch："关于用户表的 xxx 字段，是否需要..."
    │    └─ announce 回飞书群聊
    │ sys_arch 回复
    │    └─ announce 回飞书群聊
    ↓
fe_coder 对接口有疑问
    │ sessions_send → be_coder："登录接口的 token 返回格式是..."
    │    └─ announce 回飞书群聊
    │ be_coder 回复
    │    └─ announce 回飞书群聊
```

---

## 五、观测 Agent 运行状态

### 5.1 飞书群聊里能看到什么

在系统设计群 @ 机器人后，你会依次看到：

```
[系统设计群]
用户:   @机器人 我要做一个博客系统...
机器人: 🏗️ [sys_arch] 正在进行需求分析和架构设计...
机器人: 🏗️ [sys_arch] 功能说明文档已生成 [附件: 功能说明文档.md]
机器人: 🏗️ [sys_arch] 后端架构设计已生成 [附件: 后端架构设计.md]
机器人: 🏗️ [sys_arch] 前端架构设计已生成 [附件: 前端架构设计.md]
机器人: ⚙️ [be_coder announce] 已收到任务，开始实现后端...
机器人: ⚙️ [be_coder] 接口设计文档已完成 [附件: 接口设计文档.md]
机器人: ⚙️ [be_coder] 数据库初始化文件已完成 [附件: init.sql]
机器人: ⚙️ [be_coder] 后端业务代码已完成 [附件: ...]
机器人: 🎨 [fe_coder announce] 已收到任务，开始实现前端...
机器人: 🎨 [fe_coder] 前端页面代码已完成 [附件: ...]
机器人: 🏗️ [sys_arch] 全部开发任务已完成！产出汇总：...
```

> - sys_arch 的直接输出在**系统设计群**显示
> - be_coder / fe_coder 的 announce 消息也回到**系统设计群**（sessions_send 的 announce 机制绑定到原始会话的飞书群）
> - 后端开发群 / 前端开发群 用于直接单独调用对应 agent
> - 所有附件文件群成员均可直接点击下载

---

### 5.2 查看每个 Agent 的完整消息记录

每个 agent 的收发消息以 JSONL 格式记录在本地，无需 daemon 在线即可直接读取。

#### 文件位置

```
~/.openclaw/agents/sys_arch/sessions/sessions.json       ← 会话索引
~/.openclaw/agents/sys_arch/sessions/<sessionId>.jsonl   ← 实际消息
~/.openclaw/agents/be_coder/sessions/<sessionId>.jsonl
~/.openclaw/agents/fe_coder/sessions/<sessionId>.jsonl
```

#### 第一步：查看会话索引，找到 sessionId

```bash
# 列出 sys_arch 的所有会话（含 token 用量、时间等元数据）
openclaw sessions --agent sys_arch --json

# 或直接查看索引文件
cat ~/.openclaw/agents/sys_arch/sessions/sessions.json | python3 -m json.tool
```

#### 第二步：读取消息内容

```bash
# 将 <sessionId> 替换为上一步找到的实际 ID
cat ~/.openclaw/agents/sys_arch/sessions/<sessionId>.jsonl | \
  python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        msg = obj.get('message', {})
        role = msg.get('role', '?')
        content = msg.get('content', '')
        if isinstance(content, list):
            content = ' '.join(
                c.get('text', '') for c in content if isinstance(c, dict)
            )
        print(f'[{role}]')
        print(str(content)[:500])
        print('---')
    except:
        pass
"
```

#### 一次查看三个 agent 的最新会话

```bash
for agent in sys_arch be_coder fe_coder; do
  echo "========== $agent =========="
  latest=$(ls -t ~/.openclaw/agents/$agent/sessions/*.jsonl 2>/dev/null | head -1)
  if [ -z "$latest" ]; then
    echo "（暂无会话记录）"
  else
    echo "文件：$latest"
    wc -l "$latest"
  fi
  echo ""
done
```

---

### 5.3 确认 agent 间是否成功通信

sessions_send 产生的内部消息携带 `inter_session` 来源标记，可以直接在文件里检索：

```bash
# 检查 be_coder 是否收到过来自 sessions_send 的消息
grep -l "inter_session" ~/.openclaw/agents/be_coder/sessions/*.jsonl

# 查看该消息的来源详情
grep "inter_session" ~/.openclaw/agents/be_coder/sessions/*.jsonl | \
  python3 -c "
import sys, json
for line in sys.stdin:
    parts = line.split(':', 1)
    if len(parts) < 2: continue
    try:
        obj = json.loads(parts[1])
        prov = obj.get('inputProvenance') or obj.get('message', {}).get('inputProvenance')
        if prov:
            print(json.dumps(prov, ensure_ascii=False, indent=2))
            print('---')
    except:
        pass
"
```

成功唤起时输出类似：

```json
{
  "kind": "inter_session",
  "sourceSessionKey": "agent:sys_arch:feishu:group:oc_xxxxxxxx",
  "sourceTool": "sessions_send"
}
```

---

### 5.4 检查文件是否落盘

```bash
# 查看各 agent 的产出文件
for agent in sys_arch be_coder fe_coder; do
  echo "========== $agent output =========="
  find ~/.openclaw/workspace-$agent/output -type f 2>/dev/null || echo "（暂无产出）"
  echo ""
done
```

---

### 5.5 实时监控日志

```bash
# 实时过滤 agent 间调用相关日志
tail -f ~/.openclaw/logs/openclaw.log | grep -E "sessions-send|inter_session|be_coder|fe_coder|sendAttachment"
```

---

## 六、完整 openclaw.json 示例

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "main": {
          "appId": "<your-app-id>",
          "appSecret": "<your-app-secret>",
          "name": "My AI assistant"
        }
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "vllm": {
        "baseUrl": "http://<your-vllm-host>/v1",
        "apiKey": "dummy",
        "api": "openai-completions",
        "models": [
          {
            "id": "Qwen3.5-27B",
            "name": "Qwen3.5-27B (vLLM)",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 131072,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "vllm/Qwen3.5-27B" },
      "workspace": "/home/lane/.openclaw/workspace",
      "compaction": { "mode": "safeguard" }
    },
    "list": [
      {
        "id": "sys_arch", "name": "系统架构师",
        "workspace": "/home/lane/.openclaw/workspace-sys_arch",
        "agentDir": "/home/lane/.openclaw/agents/sys_arch/agent",
        "tools": { "profile": "coding" }
      },
      {
        "id": "be_coder", "name": "后端工程师",
        "workspace": "/home/lane/.openclaw/workspace-be_coder",
        "agentDir": "/home/lane/.openclaw/agents/be_coder/agent",
        "tools": { "profile": "coding" }
      },
      {
        "id": "fe_coder", "name": "前端工程师",
        "workspace": "/home/lane/.openclaw/workspace-fe_coder",
        "agentDir": "/home/lane/.openclaw/agents/fe_coder/agent",
        "tools": { "profile": "coding" }
      },
      { "id": "main", "name": "默认助手" }
    ]
  },
  "bindings": [
    {
      "comment": "系统设计群 → sys_arch",
      "agentId": "sys_arch",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": { "kind": "group", "id": "<系统设计群-group-id>" }
      }
    },
    {
      "comment": "后端开发群 → be_coder",
      "agentId": "be_coder",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": { "kind": "group", "id": "<后端开发群-group-id>" }
      }
    },
    {
      "comment": "前端开发群 → fe_coder",
      "agentId": "fe_coder",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": { "kind": "group", "id": "<前端开发群-group-id>" }
      }
    },
    {
      "comment": "兜底 → main（私聊及未匹配任何群的消息由默认 agent 处理）",
      "agentId": "main",
      "match": {
        "channel": "feishu",
        "accountId": "main"
      }
    }
  ],
  "tools": {
    "profile": "coding",
    "agentToAgent": {
      "enabled": true,
      "allow": ["sys_arch", "be_coder", "fe_coder"]
    }
  },
  "commands": { "native": "auto", "nativeSkills": "auto", "restart": true, "ownerDisplay": "raw" },
  "session": { "dmScope": "per-channel-peer" },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": { "mode": "token", "token": "<your-gateway-token>" },
    "tailscale": { "mode": "off", "resetOnExit": false },
    "nodes": {
      "denyCommands": ["camera.snap", "camera.clip", "screen.record", "contacts.add", "calendar.add", "reminders.add", "sms.send"]
    }
  }
}
```

> **注意**：上面的完整示例中 `bindings` 已经放在顶层（与 `agents` 平级），这是正确的位置。

---

## 七、工作区目录结构一览

部署完成后，各 agent 工作区的文件结构如下：

```
~/.openclaw/workspace-sys_arch/
├── SOUL.md              ← 人格与职责定义
├── AGENTS.md            ← 操作规范（落盘、通信、安全）
├── IDENTITY.md          ← 名称与 emoji（🏗️ 系统架构师）
├── output/
│   └── docs/
│       ├── 功能说明文档.md      ← 运行时生成
│       ├── 后端架构设计.md      ← 运行时生成
│       └── 前端架构设计.md      ← 运行时生成

~/.openclaw/workspace-be_coder/
├── SOUL.md
├── AGENTS.md
├── IDENTITY.md          ← ⚙️ 后端工程师
├── output/
│   ├── docs/
│   │   └── 接口设计文档.md      ← 运行时生成
│   ├── sql/
│   │   └── init.sql             ← 运行时生成
│   └── src/
│       └── (Java Spring Boot 项目结构)

~/.openclaw/workspace-fe_coder/
├── SOUL.md
├── AGENTS.md
├── IDENTITY.md          ← 🎨 前端工程师
├── output/
│   └── src/
│       └── (React/Vue 项目结构)
```

---

## 八、部署检查清单

- [ ] 执行 `openclaw agents add sys_arch` / `be_coder` / `fe_coder`
- [ ] 在 `openclaw.json` 的 `agents.list` 中显式声明 `main` agent 条目（内置 agent，无需 `agents add`）
- [ ] 在飞书客户端创建三个群组（系统设计群 / 后端开发群 / 前端开发群）
- [ ] 将机器人加入三个群组（群设置 → 群机器人 → 添加机器人）
- [ ] 获取三个群的 Group ID（客户端 7.60+ 在群设置查看，或调用 `GET /open-apis/im/v1/chats` 接口）
- [ ] 确认飞书应用已开启 `im:message`、`im:message.group_at_msg:readonly`、`im:chat`、`im:resource` 权限
- [ ] 修改 `openclaw.json`，将 Group ID 填入 `bindings` 配置（bindings 在顶层，不在 agents 内部）
- [ ] 为各工作区创建 `SOUL.md`、`AGENTS.md`、`IDENTITY.md`
- [ ] 为各工作区创建 `output/{docs,src,sql}` 目录
- [ ] 执行 `openclaw restart` 重载配置
- [ ] 执行 `openclaw agents list --bindings` 验证绑定关系
- [ ] 在系统设计群 @ 机器人发送"你好"测试响应
- [ ] 验证文件落盘：检查 `output/` 目录是否有产出
- [ ] 验证飞书附件：检查群聊中是否收到可下载文件

---

## 九、常见问题

### Q: @ 了机器人但没有响应？

**A**: 检查两点：
1. 群的 Group ID 是否填写正确，可在飞书开放平台群管理页面查询
2. 确认机器人已被加入该群聊，且有消息读取权限

### Q: 如何获取飞书群的 Group ID？

**A**: Group ID 格式为 `oc_xxxxxxxxxxxxxxxx`，三种方式：
1. **客户端查看**（需 7.60+）：群聊 → 右上角「···」→「设置」→ 页面底部复制群 ID
2. **API 接口**：获取 `tenant_access_token` 后调用 `GET https://open.feishu.cn/open-apis/im/v1/chats`，从返回的 `chat_id` 字段取值
3. **开放平台调试台**：登录 [open.feishu.cn](https://open.feishu.cn) → 你的应用 →「API 调试台」→ 搜索「获取群列表」直接运行

### Q: sys_arch 回复说"已分配给 be_coder"，但 be_coder 的 .jsonl 文件里没有收到任何消息？

查看 sys_arch 的 `.jsonl` 文件，确认实际调用的工具名称：

**情况一：调用的是 `sessions_spawn`（最常见）**

`sessions_spawn` 是创建隔离子进程的工具，不是跨 agent 通信工具。在飞书群聊中调用会报错：

```
"Feishu current-conversation binding is only available in direct messages or topic conversations."
```

LLM 收到此错误后不会中止，而是继续生成文字假装成功。

**修复**：在 SOUL.md 和 AGENTS.md 中都明确写明使用 `sessions_send` 工具并提供参数示例，同时**禁止使用 `sessions_spawn`**。重新写入后执行 `openclaw restart`。

**情况二：调用的是 `sessions_send` 但返回 `{"status":"forbidden"}`**

这是 `agentToAgent` 未启用导致的静默失败。检查顶层 `tools` 中是否有：

```json
"agentToAgent": { "enabled": true, "allow": ["sys_arch", "be_coder", "fe_coder"] }
```

`agentToAgent` 必须在**顶层 `tools`** 中，不能放在 per-agent 的 `tools` 里。修复后执行 `openclaw restart`。

### Q: 智能体之间无法互相调用？

**A**: 确认三点：
1. 顶层 `tools.agentToAgent.enabled` 为 `true`
2. `allow` 列表中包含了**所有三个** agent id（`sys_arch`、`be_coder`、`fe_coder`）
3. SOUL.md 中明确告知 agent 可以向哪些 agent 发送消息

### Q: 文件没有落盘，只在聊天中输出？

**A**: 检查 AGENTS.md 和 SOUL.md 是否明确指示了落盘规则。LLM 倾向于直接在聊天中输出内容，需要在提示词中**强调"必须先写入磁盘文件，再在聊天中报告路径"**。确保 `output/` 目录已预先创建。

### Q: 飞书群聊没有收到文件附件？

**A**: 检查以下几点：
1. 飞书应用是否开启了 `im:resource` 权限
2. SOUL.md 中是否有明确的 `message sendAttachment` 调用示例
3. 文件路径是否正确（绝对路径）
4. 查看日志 `grep sendAttachment ~/.openclaw/logs/openclaw.log`

### Q: agent 间通信时消息太长导致超时？

**A**: 这就是为什么推荐传文件路径而非全文。在 SOUL.md 和 AGENTS.md 中强调：
- sessions_send 的 message 只放摘要和文件路径
- 接收方通过读取文件获取完整内容
- 这样既节省 token 又避免超时

### Q: 三个智能体会共享对话历史吗？

**A**: 不会。每个智能体的会话独立存储在 `~/.openclaw/agents/<agentId>/sessions/`，互不干扰。但它们可以通过 sessions_send 交换信息，通过读取对方工作区的文件获取产出。

### Q: BOOTSTRAP.md、HEARTBEAT.md、TOOLS.md、USER.md 是否需要配置？

**A**:
- **BOOTSTRAP.md**：不需要。这是首次运行的交互式引导脚本，运行一次后自动删除。自动化多 agent 场景不需要
- **HEARTBEAT.md**：可选。如果需要 agent 定期主动检查某些状态（如编译是否通过、测试是否通过），可以配置。留空则不消耗 API 调用
- **TOOLS.md**：可选。如果需要记录特定的 SSH 地址、数据库连接等环境信息，可以配置
- **USER.md**：可选。如果需要 agent 了解团队成员的偏好（如编码风格偏好），可以配置
