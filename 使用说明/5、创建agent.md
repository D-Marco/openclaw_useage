# 创建 agent

## 确认 OpenClaw 运行环境正常

```shell
openclaw doctor
```

需要保证

```text
✓ Gateway running
✓ Model provider configured
✓ Workspace detected
✓ Skills registry ok
```

## 创建 Workspace（Agent容器）

OpenClaw 所有 agent 都在 workspace 里。

```shell
mkdir openclaw-workspace
cd openclaw-workspace
openclaw init
```
