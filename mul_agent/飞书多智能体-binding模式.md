# OpenClaw 飞书多智能体协作 —— Binding 模式

> 适用版本：v2026.1.6+
> **玩法特点**：不同飞书群绑定不同 agent，每个群只有一个 agent 响应，适合按团队/职能划分群组的场景

---

## 一、整体架构

```text
系统设计群  →  sys_arch（总协调者 / supervisor）
后端开发群  →  be_coder（后端执行者）
前端开发群  →  fe_coder（前端执行者）

主协作链路：
用户 → sys_arch → be_coder → sys_arch → fe_coder → sys_arch → 飞书群
```

**工作方式**：

- 每个飞书群通过 `bindings` 绑定到对应 agent，用户在哪个群 @ 机器人，就由哪个 agent 响应
- 主工作流采用中心协调的 supervisor workflow，不让 `be_coder` 与 `fe_coder` 自行形成长期对等主流程
- agent 间协作统一使用 `sessions_send`
- 所有生成的文档和源码必须落盘到各自 workspace，并在需要时由 `sys_arch` 发送附件到飞书群
- agent 之间传递内容时优先发“文件路径 + 摘要 + 下一步说明”，避免粘贴长篇全文

### 1.1 announce 机制与本模式中的实际作用

`sessions_send` 的 A2A 流程里，announce 本质上是一个后置摘要发布步骤：

1. 先把消息投递给目标 agent
2. 目标 agent 开始处理任务
3. 如果目标 session 能解析出有效的外部路由，系统会尝试让目标 agent 再生成一段面向用户的摘要，并发到目标 session 对应的外部聊天渠道

在 OpenClaw 当前源码下，announce 的目标解析依赖目标 session 自身的路由信息。
因此像 `agent:be_coder:main`、`agent:fe_coder:main`、`agent:sys_arch:main` 这类内部 session key，通常不能稳定把阶段摘要自动回到飞书群。

所以在本文档推荐的 binding 模式里：

- announce 可以保留为补充机制
- 但不作为阶段进度可见性的主链路
- 真正可靠的做法是：`be_coder` / `fe_coder` 完成阶段后，显式 `sessions_send` 回传给 `sys_arch`
- 再由 `sys_arch` 统一向飞书群发送阶段进展

### 1.2 文件发送到飞书群聊

OpenClaw 内置 `message` 工具支持 `sendAttachment` 动作，可将本地文件发送到飞书群聊：

```
工具: message
参数:
  action: "sendAttachment"
  media: "/home/lane/.openclaw/workspace-sys_arch/output/功能说明文档.md"
  filename: "功能说明文档.md"
```

飞书底层流程：先调用 `POST /im/v1/files` 上传文件获取 `file_key`，再通过 `POST /im/v1/messages` 以 `msg_type: "file"`
发送到群聊。agent 只需调用 `message` 工具即可，OpenClaw 自动处理上传流程。

> **注意**：需要飞书应用额外开启 `im:resource` 权限来上传文件。

---

## 二、部署步骤

### 步骤 1：创建智能体

```bash
openclaw agents add sys_arch
openclaw agents add be_coder
openclaw agents add fe_coder
```

> **关于 `main` agent**：`main` 是 OpenClaw 的内置默认 agent，无需执行 `agents add`。但由于配置了 `agents.list`
> 后路由查找只在列表中搜索，必须在列表中**显式声明** `main` 条目，否则 binding 指向 `main` 时会被静默回退到列表中的第一个
> agent（sys_arch）。

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

登录 [open.feishu.cn](https://open.feishu.cn) → 进入你的应用 →「API 调试台」→ 搜索「获取群列表」接口 → 直接在界面运行并复制
`chat_id`。

#### 2.4 所需飞书应用权限

确认你的飞书应用在开放平台「权限管理」中已开启：

| 权限                                 | 用途               |
|------------------------------------|------------------|
| `im:message`                       | 接收和发送消息          |
| `im:message.group_at_msg:readonly` | 读取群聊 @ 消息        |
| `im:chat`                          | 获取群信息（含 chat_id） |
| `im:resource`                      | 上传和发送文件附件        |

---

### 步骤 3：修改 openclaw.json

在 `openclaw.json` 中修改 `agents`、`bindings`、`tools` 三个字段：

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "vllm/Qwen3.5-27B"
      },
      "workspace": "/home/lane/.openclaw/workspace",
      "compaction": {
        "mode": "safeguard"
      }
    },
    "list": [
      {
        "id": "sys_arch",
        "name": "系统架构师",
        "workspace": "/home/lane/.openclaw/workspace-sys_arch",
        "agentDir": "/home/lane/.openclaw/agents/sys_arch/agent",
        "tools": {
          "profile": "full"
        }
      },
      {
        "id": "be_coder",
        "name": "后端工程师",
        "workspace": "/home/lane/.openclaw/workspace-be_coder",
        "agentDir": "/home/lane/.openclaw/agents/be_coder/agent",
        "tools": {
          "profile": "coding"
        }
      },
      {
        "id": "fe_coder",
        "name": "前端工程师",
        "workspace": "/home/lane/.openclaw/workspace-fe_coder",
        "agentDir": "/home/lane/.openclaw/agents/fe_coder/agent",
        "tools": {
          "profile": "coding"
        }
      },
      {
        "id": "main",
        "name": "默认助手"
      }
    ]
  },
  "bindings": [
    {
      "comment": "系统设计群 → sys_arch",
      "agentId": "sys_arch",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": {
          "kind": "group",
          "id": "<系统设计群-group-id>"
        }
      }
    },
    {
      "comment": "后端开发群 → be_coder",
      "agentId": "be_coder",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": {
          "kind": "group",
          "id": "<后端开发群-group-id>"
        }
      }
    },
    {
      "comment": "前端开发群 → fe_coder",
      "agentId": "fe_coder",
      "match": {
        "channel": "feishu",
        "accountId": "main",
        "peer": {
          "kind": "group",
          "id": "<前端开发群-group-id>"
        }
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
      "allow": [
        "sys_arch",
        "be_coder",
        "fe_coder"
      ]
    }
  }
}
```

> **重要说明**：
> - `bindings` 是顶层字段，与 `agents` 平级，**不在 `agents` 内部**
> - `agentToAgent` 必须在顶层 `tools` 中配置，per-agent 的 `tools` 不支持该字段（源码 `AgentToolsConfig` Zod schema 用
    `.strict()` 会拒绝未知字段）
> - **`agentToAgent.enabled: true` 是 agent 间互调的开关**，缺少此配置时 `sessions_send` 工具调用会静默返回
    `{"status": "forbidden"}`，LLM 会继续生成文字假装成功，实际 be_coder 未被唤起
> - `allow` 列表中包含所有三个 agent id，使得它们可以**两两互相调用**
> - `peer.id` 替换为步骤 2 中获取的飞书群 Group ID（格式 `oc_xxxxxxxx`）
> - 群聊中必须 @ 机器人才会触发响应（OpenClaw 默认 `requireMention: true`）

### 步骤 4：配置工作区文件

每个 agent 的工作区（workspace）支持以下配置文件，OpenClaw 在每次会话启动时自动加载它们作为系统上下文：

| 文件               | 用途                | 本场景是否需要 | 说明                                |
|------------------|-------------------|:-------:|-----------------------------------|
| **SOUL.md**      | 人格、职责、行为边界        | **必须**  | 定义 agent 的角色、工作流程和输出规范            |
| **AGENTS.md**    | 操作指南、规则、优先级       | **推荐**  | 定义落盘规范、通信协议、安全红线等通用规则             |
| **IDENTITY.md**  | agent 名称、emoji、头像 | **推荐**  | 让飞书群聊中各 agent 的消息更易辨认             |
| **TOOLS.md**     | 环境特定配置笔记          |   可选    | 如需记录特定路径、SSH 地址等环境信息              |
| **USER.md**      | 用户画像              |   可选    | 如需了解团队成员偏好可配置                     |
| **HEARTBEAT.md** | 定时检查清单            |   可选    | 如需 agent 定期主动检查（如编译状态）可配置         |
| **BOOTSTRAP.md** | 首次运行引导            |   不需要   | 面向交互式场景，自动化多 agent 不适用（首次运行后自动删除） |

> **加载机制**：OpenClaw 在 `loadWorkspaceBootstrapFiles()` 中扫描工作区目录，读取上述文件并注入系统提示词。每个文件最大
> 20KB，所有文件总计最大 150KB（可通过 `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` 配置）。

#### 4.1 AGENTS.md（通用模板，三个 agent 相同）

````markdown
# 操作规范

## 最高优先级：所有产出必须落盘

所有生成的文档、SQL、源码和阶段汇报，必须先写入工作区磁盘，再在聊天中报告路径。

- 文档统一存放：`output/{项目名称}/docs/`
- 源码统一存放：`output/{项目名称}/src/`
- SQL 文件存放：`output/{项目名称}/sql/`
- 每次写入后，都要在消息中明确说明文件路径

其中，{项目名称} 需要替换为你取的项目名称

## Agent 间通信规则

- 统一使用 `sessions_send` 做 agent 间协作
- **禁止使用 `sessions_spawn` 做常规派单**
- 传递内容时优先发“文件路径 + 摘要 + 下一步说明”，不要粘贴长篇全文
- 不要依赖 announce 自动把阶段成果发回飞书群

## 阶段成果回传规则

当你完成一个阶段、遇到阻塞、需要确认、或需要下一位 agent 接手时，必须显式 `sessions_send` 回传给 `sys_arch`。

回传内容至少包含：

- 阶段状态：已完成 / 需确认 / 受阻
- 工作摘要
- 产出文件路径
- 下一步建议
- 阻塞项（如有）

## sessions_send 的使用建议

对于“派发长任务”和“阶段性回传”，推荐显式设置：

```json
{
"sessionKey": "<target-session-key>",
"message": "...",
"timeoutSeconds": 0
}
```

````

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

````markdown
你是一位资深系统架构师，具备完整的需求分析、架构设计和多 agent 编排能力。
你可以与 be_coder（后端工程师）和 fe_coder（前端工程师）双向通信，但所有阶段任务的流转由你统一编排。

## 核心职责

收到需求时，按以下顺序执行：

1. 生成三类设计文档：功能说明文档、后端架构设计、前端架构设计，并落盘到工作区
2. 文档生成后，通过 `message` 工具的 `sendAttachment` 动作把文档发送到飞书群
3. 通过 `sessions_send` 派发给 `be_coder` 开始后端阶段
4. 接收 `be_coder` 的阶段回传，读取其产出文件，向飞书群同步进度，并把相应的文件发到飞书群供用户下载
5. 结合后端产出与前端架构设计，再派发给 `fe_coder`
6. 接收 `fe_coder` 的阶段回传，读取其产出文件，向飞书群同步进度,并把相应的文件发到飞书群供用户下载
7. 最终汇总所有产出，给用户明确的完成说明

## 工具使用规范

### 派发给 be_coder

在满足以下条件后再派发给 `be_coder`:

- 已生成功能说明文档、后端架构设计

正文中至少包含：

- 正文中要指明功能说明文档、后端架构设计的真实路径，而非文档内容

### 派发给 fe_coder

在满足以下条件后再派发给 `fe_coder`：

- 已收到 `be_coder` 的接口设计文档
- 已读取相关文档路径
- 已向飞书群同步当前后端进度

正文中至少包含：

- 功能说明文档路径
- 前端架构设计文档路径
- 接口设计文档路径

### 接收阶段回传

当收到 `be_coder` 或 `fe_coder` 的阶段回传时：

1. 读取其报告的文件路径
2. 向飞书群同步“哪个 agent 完成了什么阶段” ， agent 要用中文名称，如 be_coder 对应后端工程师，fe_coder对应前端工程师
3. 必须发送文件供下载

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

````

#### 4.4 SOUL.md —— be_coder（后端工程师）

````markdown
你是一位专业的后端工程师，擅长 API 设计、数据库设计和业务逻辑实现。

## 核心职责

收到 `sys_arch` 的任务消息后，按以下顺序执行：

### 第一步：读取设计文档

从 `sys_arch` 提供的文件路径读取：

- 功能说明文档
- 后端架构设计文档

### 第二步：编写接口设计文档

完成后，**先回传给 sys_arch**。

### 第三步：生成数据库初始化文件

输出数据库初始化 SQL 并落盘

完成后，**回传给 sys_arch**。

### 第四步：实现后端业务代码

严格依据后端架构设计文档中的技术栈、目录结构、分层方案和编码规范逐一实现代码

**完成后，要对源码进行压缩成zip，然后回传给 sys_arch **。

### 第五步：阶段回传规则

每完成一个阶段，就回传一次给 `sys_arch`，内容至少包含：

- 阶段状态
- 工作摘要
- 产出路径
- 下一步建议
- 阻塞项（如有）


````

#### 4.5 SOUL.md —— fe_coder（前端工程师）

````markdown
你是一位专业的前端工程师，擅长 React/Vue 组件开发、页面布局和交互逻辑。
你可以与 sys_arch 通信。

## 核心职责

收到 `sys_arch` 的任务消息后，按以下顺序执行：

### 第一步：读取设计文档

从 `sys_arch` 提供的路径读取：

- 功能说明文档
- 前端架构设计文档
- 接口设计文档

### 第二步：实现前端代码并落盘

严格依据前端架构设计文档规定的技术栈、目录结构、组件规范和状态管理方案逐一实现代码。

默认使用 React + TypeScript + Ant Design，除非架构文档中另有指定。
遇到需要后端接口的部分，严格对齐接口设计文档中的路径、请求参数和响应结构。
组件代码需包含 Props 类型定义、必要注释和基础错误处理。

### 第三步：阶段完成后回传 sys_arch

完成页面开发、接口接入、阶段联调、或遇到阻塞时，都必须显式回传给 `sys_arch`。

完成源码开发后，压缩源码为zip格式，回传给 sys_arch.

回传至少包含：

- 阶段状态
- 工作摘要
- 产出路径
- 待联调项 / 阻塞项


````

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

| 使用方式        | 操作                   | 实际处理                                                                                                                                                                  |
|-------------|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 完整规划 + 自动实现 | 在**系统设计群** @ 机器人描述需求 | `sys_arch` 先产出需求与架构文档，并把后端阶段异步派发给 `be_coder`；`be_coder` 的阶段成果先回传给 `sys_arch`，再由 `sys_arch` 对外同步并派发前端阶段给 `fe_coder`；`fe_coder` 的阶段成果同样先回传 `sys_arch`，最后由 `sys_arch` 汇总 |
| 直接让后端实现     | 在**后端开发群** @ 机器人描述需求 | `be_coder` 直接响应并在自己的工作区落盘产出；如需纳入总流程，应再由 `sys_arch` 统一协调                                                                                                               |
| 直接让前端实现     | 在**前端开发群** @ 机器人描述需求 | `fe_coder` 直接响应并在自己的工作区落盘产出；如需纳入总流程，应再由 `sys_arch` 统一协调                                                                                                               |
| 私聊机器人       | 直接私聊（无需 @）           | `main` 处理兜底会话，不建议作为多 agent 主协作入口                                                                                                                                      |

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


## 五、观测 Agent 运行状态

### 5.1 飞书群聊里能看到什么

在系统设计群 @ 机器人后，你会依次看到：

```
[系统设计群]
用户:   @机器人 我要做一个博客系统...
机器人: [sys_arch] 正在进行需求分析和架构设计...
机器人: [sys_arch] 功能说明文档已生成 [附件: 功能说明文档.md]
机器人: [sys_arch] 后端架构设计已生成 [附件: 后端架构设计.md]
机器人: [sys_arch] 前端架构设计已生成 [附件: 前端架构设计.md]
机器人: [sys_arch] 已派发后端阶段任务给 be_coder，开始实现接口设计
机器人: [sys_arch] 已收到 be_coder 的接口设计阶段成果 [附件: 接口设计文档.md]
机器人: [sys_arch] 已收到 be_coder 的数据库初始化阶段成果 [附件: init.sql]
机器人: [sys_arch] 已收到 be_coder 的后端实现阶段成果
机器人: [sys_arch] 已派发前端阶段任务给 fe_coder
机器人: [sys_arch] 已收到 fe_coder 的前端实现阶段成果
机器人: [sys_arch] 全部开发任务已完成，产出汇总如下：...
```

> - 用户主要看到的是 `sys_arch` 汇总后的阶段消息
> - `be_coder` / `fe_coder` 的阶段结果先回传给 `sys_arch`，再由 `sys_arch` 对外同步
> - 后端开发群 / 前端开发群 仍可用于单独直接调用对应 agent
> - 所有附件文件群成员都可直接下载

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

