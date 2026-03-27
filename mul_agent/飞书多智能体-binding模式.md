# OpenClaw 飞书多智能体协作 —— Binding 模式

> 适用版本：v2026.1.6+
> **玩法特点**：不同飞书群绑定不同 agent，每个群只有一个 agent 响应，适合按团队/职能划分群组的场景

---

## 一、整体架构

```
系统设计群  →  sys_arch（系统架构师）
                  │ 内部调用 sessions_send
                  ↓
后端开发群  →  be_coder（后端工程师）
                  │ 内部调用 sessions_send
                  ↓
前端开发群  →  fe_coder（前端工程师）
```

**工作方式**：
- 每个飞书群通过 `bindings` 绑定到对应的 agent，消息只在该群与对应 agent 之间流转
- agent 间协作通过 `sessions_send` 内部调用，不经过飞书群聊，用户不可见
- 用户在哪个群 @ 机器人，就由哪个 agent 响应

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
> - `peer.id` 替换为步骤 2 中获取的飞书群 Group ID（格式 `oc_xxxxxxxx`）
> - 群聊中必须 @ 机器人才会触发响应（OpenClaw 默认 `requireMention: true`）

### 步骤 4：为每个智能体配置人格（SOUL.md）

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
- 三类文档全部输出完毕后，**必须调用 `sessions_send` 工具**通知 be_coder 开始工作，参数如下：
  - `sessionKey`: `"agent:be_coder:main"`
  - `message`: 将功能说明文档、后端架构设计文档、前端架构设计文档的完整内容拼接后作为消息体传入
  - 不要只传 `agentId`；当前 `sessions_send` 实现要求使用 `sessionKey` 或 `label`
- **严禁使用 `sessions_spawn`**：该工具用于创建隔离子进程，在飞书群聊中会报错，不能用于跨 agent 通信
- fe_coder 由 be_coder 在完成接口设计文档后通过 `sessions_send` 调用，sys_arch 无需直接调用 fe_coder
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

以上三项全部完成后，**必须调用 `sessions_send` 工具**通知 fe_coder 开始工作，参数如下：
- `sessionKey`: `"agent:fe_coder:main"`
- `message`: 将接口设计文档、功能说明文档、前端架构设计文档的完整内容拼接后作为消息体传入

**严禁使用 `sessions_spawn`**：该工具在飞书群聊中会报错，不能用于跨 agent 通信。
EOF
```

#### fe_coder 人格配置

```bash
cat > ~/.openclaw/workspace-fe_coder/SOUL.md << 'EOF'
你是一位专业的前端工程师，擅长 React/Vue 组件开发、页面布局和交互逻辑。
收到 sys_arch 输出的详细功能说明文档和前端架构设计文档后，严格依据其中规定的技术栈、目录结构、组件规范和状态管理方案逐一实现代码，不得自行更改技术选型。
实现每个页面或组件时，对照功能说明文档中的操作流程，确保交互逻辑与文档保持一致。
优先给出可运行的代码示例，默认使用 React + TypeScript + Ant Design，除非架构文档中另有指定。
遇到需要后端接口的部分，严格对齐 be_coder 输出的接口设计文档中的路径、请求参数和响应结构，并在代码中用 TODO 标注待联调的接口。
组件代码需包含 Props 类型定义、必要注释和基础错误处理。
EOF
```

### 步骤 5：重启服务

```bash
openclaw restart
# 或
sudo systemctl restart openclaw
```

### 步骤 6：验证配置

```bash
openclaw agents list --bindings
```

---

## 三、在飞书群聊中使用

| 使用方式 | 操作 | 实际处理 |
|---------|------|---------|
| 完整规划 + 自动实现 | 在**系统设计群** @ 机器人描述需求 | sys_arch 出三类文档 → 内部调用 be_coder → 内部调用 fe_coder |
| 直接让后端实现 | 在**后端开发群** @ 机器人描述需求 | be_coder 直接响应 |
| 直接让前端实现 | 在**前端开发群** @ 机器人描述需求 | fe_coder 直接响应 |
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
        │ sys_arch 按 SOUL.md 生成三类文档
        │
│ sessions_send(sessionKey="agent:be_coder:main", ...)  ← 内部调用，不经过飞书
        ↓
    be_coder 生成接口设计文档 + init.sql + 业务代码
        │
│ sessions_send(sessionKey="agent:fe_coder:main", ...)  ← 内部调用，不经过飞书
        ↓
    fe_coder 实现前端页面
        │
        ↓ 最终结果回复到飞书群聊（用户可见）
```

---

## 五、观测 Agent 运行状态

### 5.1 飞书群聊里能看到什么

在系统设计群 @ 机器人后，你会依次看到：

```
[系统设计群]
用户:   @机器人 我要做一个博客系统...
机器人: （sys_arch 输出功能说明文档 + 后端架构 + 前端架构）
机器人: （be_coder announce 回复，如"已收到任务，开始实现接口文档..."）
机器人: （be_coder 输出接口设计文档 + init.sql + 业务代码）
机器人: （fe_coder announce 回复，如"已收到任务，开始实现前端页面..."）
机器人: （fe_coder 输出前端代码）
```

> be_coder / fe_coder 的输出会回到**系统设计群**（sessions_send 是内部调用，响应链路绑定到原始会话）。后端开发群 / 前端开发群 是用来直接单独调用对应 agent 的。

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

### 5.3 确认 be_coder 是否被 sys_arch 成功唤起

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

### 5.4 实时监控日志

```bash
# 实时过滤 agent 间调用相关日志
tail -f ~/.openclaw/logs/openclaw.log | grep -E "sessions-send|inter_session|be_coder|fe_coder"
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
    ],
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
    ]
  },
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

---

## 七、部署检查清单

- [ ] 执行 `openclaw agents add sys_arch` / `be_coder` / `fe_coder`
- [ ] 在 `openclaw.json` 的 `agents.list` 中显式声明 `main` agent 条目（内置 agent，无需 `agents add`）
- [ ] 在飞书客户端创建三个群组（系统设计群 / 后端开发群 / 前端开发群）
- [ ] 将机器人加入三个群组（群设置 → 群机器人 → 添加机器人）
- [ ] 获取三个群的 Group ID（客户端 7.60+ 在群设置查看，或调用 `GET /open-apis/im/v1/chats` 接口）
- [ ] 确认飞书应用已开启 `im:message`、`im:message.group_at_msg:readonly`、`im:chat` 权限
- [ ] 修改 `openclaw.json`，将 Group ID 填入 `bindings` 配置
- [ ] 在各工作区创建 `SOUL.md` 人格文件
- [ ] 执行 `openclaw restart` 重载配置
- [ ] 执行 `openclaw agents list --bindings` 验证绑定关系
- [ ] 在系统设计群 @ 机器人发送"你好"测试响应

---

## 八、常见问题

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

**修复**：在 SOUL.md 中明确写明使用 `sessions_send` 工具并提供参数示例，同时禁止使用 `sessions_spawn`。重新写入 SOUL.md 后执行 `openclaw restart`。

**情况二：调用的是 `sessions_send` 但返回 `{"status":"forbidden"}`**

这是 `agentToAgent` 未启用导致的静默失败。检查顶层 `tools` 中是否有：

```json
"agentToAgent": { "enabled": true, "allow": ["sys_arch", "be_coder", "fe_coder"] }
```

`agentToAgent` 必须在**顶层 `tools`** 中，不能放在 per-agent 的 `tools` 里。修复后执行 `openclaw restart`。

### Q: 智能体之间无法互相调用？

**A**: 确认顶层 `tools.agentToAgent.enabled` 为 `true`，且 `allow` 列表中包含了需要互调的所有 agent id。

### Q: 三个智能体会共享对话历史吗？

**A**: 不会。每个智能体的会话独立存储在 `~/.openclaw/agents/<agentId>/sessions/`，互不干扰。
