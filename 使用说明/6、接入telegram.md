# 接入telegram



# 一、创建 Telegram Bot（获取 token）

1. 打开 Telegram
2. 搜索 **@BotFather**
3. 发送命令：

```
/newbot
```

然后按提示输入：

* Bot 名称
* Bot username（必须以 `bot` 结尾，例如 `lane_ai_bot`）

完成后 BotFather 会返回：

```
123456789:ABCDEFxxxxxxxxxxxx
```

这个就是 **Bot Token**，必须保存好。 ([OpenClaw - 开源个人 AI 助手][1])

---

# 二、配置 OpenClaw Telegram Channel

编辑配置文件：

```
~/.openclaw/openclaw.json
```

加入 Telegram 配置：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "123456789:ABCDEFxxxxxxxxxxxx",
      "dmPolicy": "pairing",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

关键字段说明：

| 字段                    | 作用                  |
| --------------------- | ------------------- |
| enabled               | 启用 telegram channel |
| botToken              | BotFather 给你的 token |
| dmPolicy              | DM 默认需要配对           |
| groups.requireMention | 群里必须 @bot 才回复       |

OpenClaw 通过 Telegram Bot API 与 Telegram 通信。 ([OpenClaw][2])

---

# 三、启动 OpenClaw Gateway

执行：

```
openclaw gateway
```

或：

```
openclaw gateway start
```

Gateway 是所有聊天渠道的入口。 ([OpenClaw AI][3])

---

# 四、给机器人发消息

1. 在 Telegram 搜索你的 bot
2. 发送任意消息

因为默认开启 **pairing 模式**，OpenClaw 不会立即回复。 ([OpenClaw - 开源个人 AI 助手][1])

---

# 五、批准配对

查看配对请求：

```
openclaw pairing list telegram
```

你会看到类似：

```
CODE: 493021
USER: 123456789
```

批准：

```
openclaw pairing approve telegram 493021
```

之后 Telegram 就可以正常聊天。

---

# 六、验证是否成功

Telegram 给 bot 发消息：

```
hello
```

如果配置正确：

OpenClaw 会回复。

---

# 七、推荐的安全配置（建议）

限制只有你能用：

```json
{
  "channels": {
    "telegram": {
      "botToken": "...",
      "ownerIds": ["你的telegram用户ID"]
    }
  }
}
```

用户 ID 可以通过：

* `@userinfobot`
* `@getidsbot`

获取。 ([OpenClaw AI][3])
