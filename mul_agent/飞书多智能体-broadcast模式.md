# OpenClaw 飞书多智能体协作 —— Broadcast 模式

> 适用版本：v2026.1.6+
> **玩法特点**：一个飞书群，消息广播给所有 agent。飞书渠道中 **sys_arch 始终是唯一活跃 agent**（由 bindings 决定），be_coder / fe_coder 作为观察者静默记录上下文，由 sys_arch 通过 `sessions_send` 内部调用

---

## 一、整体架构

```
同一个飞书群
   │
   │  @ 机器人 → 消息广播给所有 agent
   │
   ├── sys_arch  → 活跃 agent（真正回复飞书，由 bindings 决定）
   │                  读取消息内容，按需通过 sessions_send 调用 be_coder / fe_coder
   │
   ├── be_coder  → 观察者（静默推理 + 记录上下文，不回复飞书）
   │                  只能通过 sys_arch 的 sessions_send 激活
   │
   └── fe_coder  → 观察者（静默推理 + 记录上下文，不回复飞书）
                       只能通过 be_coder 的 sessions_send 激活
```

> **注意**：飞书渠道的 broadcast 模式中，**`groupChat.mentionPatterns` 完全不生效**（源码 `feishu/bot.ts` 的 broadcast 分支不调用 mentionPatterns 匹配逻辑）。activeAgentId 由 bindings 决定，始终为 sys_arch，无法通过消息内容中的 @关键词 切换活跃 agent。

**工作方式**：
- `broadcast` 配置将群消息广播给所有 agent，所有 agent 都会推理和记录上下文
- 飞书侧 **sys_arch 始终是唯一活跃 agent**，负责回复飞书群
- sys_arch 读取用户消息，识别用户意图，通过 `sessions_send` 内部调用 be_coder / fe_coder
- be_coder / fe_coder 作为观察者已有消息上下文，被调用后能感知前序对话
- 用户无法直接从飞书激活 be_coder / fe_coder，只能通过 sys_arch 的 SOUL.md 中的指令驱动

---

## 二、部署步骤

### 步骤 1：创建三个智能体

```bash
openclaw agents add sys_arch
openclaw agents add be_coder
openclaw agents add fe_coder
```

执行后自动创建以下工作区目录：

- `~/.openclaw/workspace-sys_arch/`
- `~/.openclaw/workspace-be_coder/`
- `~/.openclaw/workspace-fe_coder/`

### 步骤 2：修改 openclaw.json

将 `agents` 部分替换为以下配置，并在顶层加入 `broadcast`：

```json
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
      "tools": {
        "profile": "coding",
        "agentToAgent": {
          "enabled": true,
          "allow": ["be_coder", "fe_coder"]
        }
      }
    },
    {
      "id": "be_coder",
      "name": "后端工程师",
      "tools": {
        "profile": "coding",
        "agentToAgent": {
          "enabled": true,
          "allow": ["sys_arch", "fe_coder"]
        }
      }
    },
    {
      "id": "fe_coder",
      "name": "前端工程师",
      "tools": {
        "profile": "coding",
        "agentToAgent": {
          "enabled": true,
          "allow": ["sys_arch", "be_coder"]
        }
      }
    }
  ],
  "bindings": [
    {
      "comment": "所有飞书消息路由到 sys_arch（broadcast 模式下的唯一活跃 agent）",
      "agentId": "sys_arch",
      "match": {
        "channel": "feishu",
        "accountId": "main"
      }
    }
  ]
},
"broadcast": {
  "<your-feishu-group-id>": ["sys_arch", "be_coder", "fe_coder"]
}
```

> **说明**：
> - `broadcast` 是核心配置，`<your-feishu-group-id>` 替换为飞书群实际 Group ID（格式 `oc_xxxxxxxx`）
> - **飞书 broadcast 模式中 `groupChat.mentionPatterns` 不生效**（源码 `feishu/bot.ts` 的 broadcast 分支不调用该匹配逻辑），活跃 agent 始终由 bindings 决定，即 sys_arch
> - be_coder / fe_coder 在飞书侧是观察者（静默接收消息、记录上下文），不会直接回复飞书群，只能由 sys_arch 通过 `sessions_send` 内部激活
> - `bindings` 中的 sys_arch 是兜底：消息未匹配任何 agent 的 mentionPatterns 时，由 sys_arch 响应
> - 群聊中必须 @ 机器人才会触发响应（OpenClaw 默认 `requireMention: true`）

### 步骤 3：为每个智能体配置人格（SOUL.md）

#### sys_arch 人格配置

```bash
cat > ~/.openclaw/workspace-sys_arch/SOUL.md << 'EOF'
你是一位资深系统架构师，具备完整的需求分析、架构设计和文档编写能力。

## 核心职责

收到需求时，按以下顺序输出完整设计文档，再将实现任务分配给 be_coder 和 fe_coder。

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
- 三类文档全部输出完毕后，将功能说明文档、后端架构设计文档、前端架构设计文档一并传递，调用 be_coder 开始实现；fe_coder 由 be_coder 在完成接口设计文档后调用，无需 sys_arch 直接调用
EOF
```

#### be_coder 人格配置

```bash
cat > ~/.openclaw/workspace-be_coder/SOUL.md << 'EOF'
你是一位专业的后端工程师，擅长服务端开发、API 设计和数据库建模。
收到 sys_arch 输出的详细功能说明文档和后端架构设计文档后，按以下顺序依次输出，不得省略任何一项：

## 一、接口设计文档（供前端使用）

在编写任何代码之前，先输出完整的接口设计文档，供 fe_coder 对接使用。每个接口按以下格式描述：

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

## 二、数据库初始化文件（init.sql）

将功能说明文档中所有表的 DDL 汇总为一个完整的 `init.sql` 文件并写入工作区，要求：
- 文件开头注释项目名称、生成时间
- 按依赖顺序建表（被引用的主表在前，含外键的从表在后）
- 每张表前加注释说明表用途
- 包含所有字段定义、主键、外键、唯一约束、索引
- 每张表建完后插入必要的初始化数据（如字典表、角色表、默认管理员等）
- 文件末尾输出所有表名清单

## 三、后端业务代码

严格依据 sys_arch 后端架构设计文档规定的技术栈（Java + Spring Boot）、分层架构和接口规范实现代码，不得自行更改技术选型。
实现每个功能时，对照功能说明文档中的操作流程、表结构和表间业务逻辑，确保代码逻辑与文档保持一致。
代码需包含必要的注释、错误处理和参数校验（使用 Spring Validation）。

以上三项全部完成后，将接口设计文档、功能说明文档、前端架构设计文档一并传递，调用 fe_coder 开始实现前端页面。
EOF
```

#### fe_coder 人格配置

```bash
cat > ~/.openclaw/workspace-fe_coder/SOUL.md << 'EOF'
你是一位专业的前端工程师，擅长 React/Vue 组件开发、页面布局和交互逻辑。
收到 be_coder 输出的接口设计文档、以及 sys_arch 输出的功能说明文档和前端架构设计文档后，严格依据其中规定的技术栈、目录结构、组件规范和状态管理方案逐一实现代码，不得自行更改技术选型。
实现每个页面或组件时，对照功能说明文档中的操作流程，确保交互逻辑与文档保持一致。
优先给出可运行的代码示例，默认使用 React + TypeScript + Ant Design，除非架构文档中另有指定。
遇到需要后端接口的部分，严格对齐 be_coder 输出的接口设计文档中的路径、请求参数和响应结构，并在代码中用 TODO 标注待联调的接口。
组件代码需包含 Props 类型定义、必要注释和基础错误处理。
EOF
```

### 步骤 4：重启服务

```bash
openclaw restart
# 或
sudo systemctl restart openclaw
```

### 步骤 5：验证配置

```bash
openclaw agents list --bindings
```

---

## 三、在飞书群聊中使用

> **说明**：飞书群聊中始终由 sys_arch 回复。be_coder / fe_coder 只能通过 sys_arch 内部调用触发，不能从飞书直接激活。

| 使用方式 | 飞书发送示例 | 实际处理 |
|---------|------------|---------|
| 完整规划 + 自动实现 | `@机器人 我要做一个XX系统，功能有...` | sys_arch 出三类文档 → 内部调用 be_coder → 内部调用 fe_coder |
| 只要功能说明文档 | `@机器人 详细描述用户登录功能` | sys_arch 输出功能说明+流程+表结构+业务逻辑 |
| 让后端实现（需经 sys_arch 转发） | `@机器人 帮我实现用户登录接口，让后端工程师来做` | sys_arch 收到后通过 sessions_send 调用 be_coder |
| 私聊机器人 | `帮我看看这个需求` | sys_arch（私聊无需 @） |

**使用示例**：

```
@机器人 @sys_arch 我要做一个博客系统，需要以下功能：
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
用户在群聊 @ 机器人
        │
        │ broadcast 广播给所有 agent
        │ activeAgentId = sys_arch（由 bindings 决定，始终如此）
        │
        ├── sys_arch  → 活跃 agent，真正推理并回复飞书
        ├── be_coder  → 观察者，静默推理 + 记录上下文，不回复飞书
        └── fe_coder  → 观察者，静默推理 + 记录上下文，不回复飞书
        │
        │ sys_arch 按 SOUL.md 生成三类文档并回复飞书
        │
        │ sessions_send(agentId="be_coder", ...)  ← 内部调用，不经过飞书
        ↓
    be_coder 生成接口设计文档 + init.sql + 业务代码
        │
        │ sessions_send(agentId="fe_coder", ...)  ← 内部调用，不经过飞书
        ↓
    fe_coder 实现前端页面
        │
        ↓ 最终结果回复到飞书群聊（用户可见）
```

---

## 五、完整 openclaw.json 示例

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
        "id": "sys_arch",
        "name": "系统架构师",
        "tools": {
          "profile": "coding",
          "agentToAgent": { "enabled": true, "allow": ["be_coder", "fe_coder"] }
        }
      },
      {
        "id": "be_coder",
        "name": "后端工程师",
        "tools": {
          "profile": "coding",
          "agentToAgent": { "enabled": true, "allow": ["sys_arch", "fe_coder"] }
        }
      },
      {
        "id": "fe_coder",
        "name": "前端工程师",
        "tools": {
          "profile": "coding",
          "agentToAgent": { "enabled": true, "allow": ["sys_arch", "be_coder"] }
        }
      }
    ],
    "bindings": [
      {
        "comment": "所有飞书消息路由到 sys_arch（broadcast 模式下的唯一活跃 agent）",
        "agentId": "sys_arch",
        "match": {
          "channel": "feishu",
          "accountId": "main"
        }
      }
    ]
  },
  "broadcast": {
    "<your-feishu-group-id>": ["sys_arch", "be_coder", "fe_coder"]
  },
  "tools": { "profile": "coding" },
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

---

## 六、部署检查清单

- [ ] 执行 `openclaw agents add sys_arch` / `be_coder` / `fe_coder`
- [ ] 在飞书开放平台获取目标群的 Group ID（格式 `oc_xxxxxxxx`）
- [ ] 修改 `openclaw.json`，配置 `broadcast`（群 ID 映射到三个 agent）
- [ ] 在各工作区创建 `SOUL.md` 人格文件
- [ ] 执行 `openclaw restart` 重载配置
- [ ] 执行 `openclaw agents list --bindings` 验证绑定关系
- [ ] 在群聊中 `@机器人 你好` 测试响应（由 sys_arch 回复）

---

## 七、常见问题

### Q: @ 了机器人但没有响应？

**A**: 检查两点：
1. `broadcast` 中的群 Group ID 是否填写正确
2. 确认机器人已被加入该群聊，且有消息读取权限

### Q: 为什么无论怎么写消息，都是 sys_arch 在回复？

**A**: 这是飞书 broadcast 模式的设计行为。源码中飞书的 broadcast 分支不调用 `mentionPatterns` 匹配逻辑，activeAgentId 由 bindings 决定，始终是 sys_arch。be_coder / fe_coder 只能由 sys_arch 通过 `sessions_send` 内部激活，不会直接在飞书群里响应。

### Q: 智能体之间无法互相调用？

**A**: 确认双方的 `agentToAgent.allow` 列表中都包含了对方的 `id`，且 `enabled: true`。

### Q: 三个智能体会共享对话历史吗？

**A**: 不会。每个智能体的会话独立存储在 `~/.openclaw/agents/<agentId>/sessions/`，互不干扰。

### Q: broadcast 模式和 binding 模式如何选择？

**A**:
- **broadcast 模式**：团队在同一个群工作，希望灵活切换不同 agent，消息文本中加关键词即可路由
- **binding 模式**：不同职能团队用不同群，每个群只需要一个固定 agent，配置更简单直接
