# sessions_send 参数修复说明

适用场景：飞书多智能体 binding 模式下，`sys_arch` 需要把任务投递给 `be_coder` / `fe_coder`。

## 问题原因

`sessions_send` 目前要求目标必须通过下面二选一来确定：

- `sessionKey`
- `label`

其中 `agentId` **不能单独作为目标**。`agentId` 只是在使用 `label` 查找时，用来限定查找范围。

所以这类调用会失败：

```json
{
  "agentId": "be_coder",
  "message": "..."
}
```

报错通常就是：

```text
Either sessionKey or label is required
```

## 推荐修复

不要改 `openclaw` 源码，直接把 SOUL.md / 提示词里的调用方式改成显式 `sessionKey`。

同时，`sys_arch` 要跨 agent 给 `be_coder` 发消息，还必须允许跨 agent session 可见性。当前源码默认值不是 `all`。

源码依据：

- [session visibility 默认值在 `tree`](/D:/git_res/lw_openclaw/openclaw/src/agents/tools/sessions-access.ts#L23)
- [跨 agent 且 visibility 不是 `all` 时会直接 forbidden](/D:/git_res/lw_openclaw/openclaw/src/agents/tools/sessions-access.ts#L200)
- [配置说明里也明确写了 `"tree" default`](/D:/git_res/lw_openclaw/openclaw/src/config/schema.help.ts#L661)

因此配置里要显式加上：

```json
"tools": {
  "profile": "coding",
  "sessions": {
    "visibility": "all"
  },
  "agentToAgent": {
    "enabled": true,
    "allow": ["sys_arch", "be_coder", "fe_coder"]
  }
}
```

你当前配置里已经把：

```json
"session": {
  "mainKey": "main",
  "dmScope": "per-channel-peer"
}
```

固定成了 `mainKey = main`，因此各 agent 的主会话 key 可以稳定写成：

- `sys_arch` 主会话：`agent:sys_arch:main`
- `be_coder` 主会话：`agent:be_coder:main`
- `fe_coder` 主会话：`agent:fe_coder:main`

## sys_arch 的 SOUL.md 改法

把原来这类描述：

```text
调用 sessions_send 通知 be_coder，参数：
- agentId: "be_coder"
- message: ...
```

改成：

```text
调用 sessions_send 通知 be_coder，参数必须使用：
- sessionKey: "agent:be_coder:main"
- message: 将功能说明文档、后端架构设计文档、前端架构设计文档的完整内容拼接后传入

严禁只传 agentId，因为 sessions_send 不会把 agentId 当作直接投递目标。
```

推荐直接给模型一个完整示例：

```json
{
  "sessionKey": "agent:be_coder:main",
  "message": "<将三类文档完整拼接后的内容>"
}
```

## be_coder 的 SOUL.md 改法

把原来这类描述：

```text
调用 sessions_send 通知 fe_coder，参数：
- agentId: "fe_coder"
- message: ...
```

改成：

```text
调用 sessions_send 通知 fe_coder，参数必须使用：
- sessionKey: "agent:fe_coder:main"
- message: 将接口设计文档、功能说明文档、前端架构设计文档的完整内容拼接后传入

严禁只传 agentId，因为 sessions_send 不会把 agentId 当作直接投递目标。
```

对应完整示例：

```json
{
  "sessionKey": "agent:fe_coder:main",
  "message": "<将接口文档与相关设计文档完整拼接后的内容>"
}
```

## 如果以后改了 mainKey

如果你后续把配置改成：

```json
"session": {
  "mainKey": "root"
}
```

那主会话 key 也要同步改成：

- `agent:be_coder:root`
- `agent:fe_coder:root`

所以最稳妥的做法是：

- 在配置里显式写出 `session.mainKey`
- 在 SOUL.md 里按这个值写死 `sessionKey`

## 建议你现在实际替换的文本

### sys_arch

```text
三类文档全部输出完毕后，必须调用 sessions_send 工具通知 be_coder 开始工作。
参数必须使用：
- sessionKey: "agent:be_coder:main"
- message: 将功能说明文档、后端架构设计文档、前端架构设计文档的完整内容拼接后作为消息体传入

严禁只传 agentId。sessions_send 的 agentId 不能单独作为投递目标。
```

### be_coder

```text
以上三项全部完成后，必须调用 sessions_send 工具通知 fe_coder 开始工作。
参数必须使用：
- sessionKey: "agent:fe_coder:main"
- message: 将接口设计文档、功能说明文档、前端架构设计文档的完整内容拼接后作为消息体传入

严禁只传 agentId。sessions_send 的 agentId 不能单独作为投递目标。
```
