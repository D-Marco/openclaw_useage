
**如果 workspace 里有上百个 Skills，不能让它们“全量加载 + 全量注入 prompt”。**
否则会出现：

* token 成本飙升
* tool 选择准确率下降
* latency 明显增加
* agent 稳定性下降

这种规模必须做 **Skill 管理层（Skill Registry / Router）**。

下面按可落地架构讲。

---

# 一、最推荐方案：Skill 分层 + 动态加载（核心）

这是企业 AI 运维系统常规做法。

## ① 按领域分组（强制）

例如你的摄像头运维：

```
skills/
  camera-basic/
  camera-diagnosis/
  camera-predict/
  network/
  report/
  ops/
```

不要：

```
skills/
  skill1
  skill2
  skill3
  ...100+
```

否则无法管理。

---

## ② 按 Agent 或场景加载

典型设计：

| Agent  | 加载 Skill         |
| ------ | ---------------- |
| 监控助手   | camera-basic     |
| 异常分析助手 | camera-diagnosis |
| 预测助手   | camera-predict   |

这样：

👉 单 Agent 不超过 20 个 skill。

这是经验安全线。

---

# 二、做 Skill Router（强烈推荐）

如果你 Skill 很多，这是必需组件。

## Router 的作用：

根据用户请求：

```
用户问题 → 选相关 skill → 注入 prompt
```

而不是：

```
全部 skills → prompt
```

---

## Router 实现方式三种：

### ① Keyword routing（最简单）

例如：

```
包含“离线” → camera-status skill
包含“遮挡” → diagnose skill
```

优点：

* 实现快
* 成本低

缺点：

* 不智能。

---

### ② Embedding routing（推荐）

流程：

```
skill description embedding
用户 query embedding
做相似度匹配
Top-N skills 注入
```

行业标准方案。

---

### ③ LLM Router（高级）

用小模型判断：

```
需要哪些 tools？
```

适合复杂系统。

但成本略高。

---

# 三、Skill 描述压缩（必须做）

你 workspace 100+ skill 最大风险其实是：

👉 prompt 描述过长。

建议：

## 每个 skill description ≤ 20 token

例如：

```
Diagnose camera abnormal status.
```

不要写：

* API说明
* 使用流程
* 业务背景

这些放文档，不进 prompt。

---

# 四、拆 Skill（反直觉但有效）

很多人 Skill 做太大。

比如：

```
camera-diagnose
```

里面：

* 离线检测
* 遮挡检测
* 偏色检测
* 网络异常

这是错的。

---

应该拆成：

```
camera-offline
camera-occlusion
camera-color
camera-network
```

原因：

👉 Router 更容易选。

---

# 五、Skill 生命周期管理（企业级必须）

如果你上百 Skill：

建议建立：

## Skill Registry

至少包含：

| 字段          | 用途                |
| ----------- | ----------------- |
| name        | 唯一标识              |
| domain      | 领域                |
| owner       | 负责人               |
| version     | 版本                |
| description | prompt描述          |
| status      | active/deprecated |

否则半年就混乱。

---

# 六、避免 Workspace 过载的工程策略

这是经验总结。

## ✔ 不同 workspace 拆分

例如：

```
camera-ops workspace
ai-analysis workspace
report workspace
```

不要一个 workspace 放全部。

---

## ✔ Skill 按客户隔离（如果 ToB）

否则：

* prompt污染
* 数据风险

---

## ✔ 定期清理废弃 skill

否则：

Agent 选择会退化。

---

# 七、现实 token 成本评估（给你量化一下）

粗略经验：

| Skill数量 | prompt增加     |
| ------- | ------------ |
| 10个     | ~200 tokens  |
| 50个     | ~1000 tokens |
| 100个    | 2000+ tokens |

如果每次对话都带：

👉 成本会明显上涨。

---

# 八、结合你业务的最优架构建议（直接可落地）

针对摄像头 AI 运维平台：

## 推荐结构：

```
Skill Router（核心）
    ↓
领域 Skill 包
    ↓
Agent
    ↓
LLM
```

并：

* workspace 不超过 30 skill
* skill 描述极简
* embedding routing

这是成熟 AI 运维系统架构。

---

# 九、一句话总结（重点记住）

**Skill 多不是问题，不做筛选才是问题。**

解决方案：

1. 分组加载
2. Router筛选
3. 描述压缩
4. workspace拆分

---

如果你愿意，我下一步可以直接给：

* 摄像头运维 Skill 架构完整设计
* OpenClaw vs MCP Tool registry 对比
* Skill Router 示例代码（可直接实现）
* 或 AI 运维平台完整 Agent 架构

直接说。
