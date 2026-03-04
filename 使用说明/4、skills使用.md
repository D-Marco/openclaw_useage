# skills

## 安装 clawhub

```shell
sudo npm i -g clawhub
#或者
pnpm add -g clawhub
```

## 登录 clawhub

```shell
clawhub login
```

然后会在浏览器中去认证

## 安装 skill

```shell
clawhub install <skill-slug>
```

安装的 skills 会在 ~/.openclaw/workspace/skills 目录下

## 更新所有 skills

```shell
clawhub update --all
```

## 查看所有 skills

```shell
openclaw skills list
```

## 内置 skill

如果是使用源码在本地安装的 ，skill 会在源码目录的 skills 目录下。
内置的 skill 有的是 blocked 的，gating 条件不满足，因此不会加载。技能仍然被扫描到，但被判定为“不具备运行条件”。
OpenClaw 会检查 metadata.openclaw.requires：
例如：

```yaml
metadata:
  { "openclaw": { "requires": { "env": ["GEMINI_API_KEY"], "bins": ["uv"] } } }
```

如果：

- GEMINI_API_KEY 没设置
- 或系统里没有 uv 命令
  该技能状态就是 blocked

当我们需要完善这些环境以便让这些 skill 可用，可以在 openclaw 界面中的 skills 中点击安装，但需要 brew，我们可以通过以下命令安装 brew(最好设置 http 代理翻转来安装)

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

然后设置到用户环境变量

```shell
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
```

## 自定义 skill

自定义 skill 可以在 ~/.openclaw/workspace/skills 目录下创建一个新的目录，目录名就是 skill 的 slug，在目录下创建一个 SKILL.md 文件，最小内容如下：

```yaml
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
---
```
