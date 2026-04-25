---
id: dingtalk
title: 钉钉
---

# 钉钉

钉钉是阿里巴巴的企业通信平台，在中国职场中使用非常广泛。PicoClaw 使用钉钉的 **Stream 模式** SDK，通过持久化 WebSocket 连接通信，无需公网 IP，也不需要配置 Webhook。

## 配置流程

### 1. 创建企业内部应用

1. 前往 [钉钉开放平台](https://open.dingtalk.com/)
2. 点击 **应用开发** -> **企业内部开发** -> **创建应用**
3. 填写应用名称和描述

### 2. 获取凭据

1. 进入应用设置中的 **凭证与基础信息**
2. 复制 **Client ID**（AppKey）和 **Client Secret**（AppSecret）

### 3. 开启机器人能力

1. 进入 **应用功能** -> **机器人**
2. 开启机器人功能
3. 机器人支持 **群聊** 和 **单聊**

### 4. 配置权限

在 **权限与范围** 中，确保已经授予以下权限：

- 接收消息
- 发送消息

### 5. 配置 PicoClaw

#### 1. WebUI 配置

优先推荐使用 WebUI 配置，方便快捷。

![WebUI DingTalk Connection Interface](/img/channels/webui_dingtalk.png)

依次填入 Client ID（`YOUR_CLIENT_ID`）和 Client Secret（`YOUR_CLIENT_SECRET`），然后点击 **保存** 即可。

#### 2. 配置文件

修改 `~/.picoclaw/.security.yml`：

```yaml
dingtalk:
  settings:
    client_secret: YOUR_CLIENT_SECRET
```

修改 `~/.picoclaw/config.json`：

```json
{
  "channels": {
      "enabled": true,
      "type": "dingtalk",
      "reasoning_channel_id": "",
      "group_trigger": {},
      "typing": {},
      "placeholder": {
        "enabled": false
      },
      "settings": {
        "client_id": "YOUR_CLIENT_ID"
      }
  }
}
```

### 6. 运行

```bash
picoclaw gateway
```

## 字段参考

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `client_id` | string | 是 | 钉钉应用 Client ID（AppKey） |
| `client_secret` | string | 是 | 钉钉应用 Client Secret（AppSecret） |
| `allow_from` | array | 否 | 钉钉用户 ID 白名单（空数组 = 允许所有用户） |
| `group_trigger` | object | 否 | 群聊触发设置（见 [通用通道字段](../#common-channel-fields)） |
| `reasoning_channel_id` | string | 否 | 将推理过程输出到单独的聊天 |

## 工作原理

### Stream 模式

钉钉 Stream 模式使用 SDK 维护持久化 WebSocket 连接：

- **无需公网 IP**：SDK 主动向钉钉服务器发起出站连接
- **自动重连**：SDK 会自动处理断线与重连
- **实时投递**：消息通过 WebSocket 通道即时送达

### 消息处理

- **单聊**：消息直接接收
- **群聊**：默认只有在 @ 机器人时才会触发（可通过 `group_trigger` 配置）
- **Session Webhook**：每条进入的消息都携带 `sessionWebhook` URL，可用于直接回复
- **最大消息长度**：单条消息最多 20,000 个字符，超长回复会自动截断

### 群聊中的 @ 提及处理

当机器人在群聊中被 @ 提及时，PicoClaw 会自动去掉消息开头的 `@mention` 标签，再将处理后的文本传给 Agent。这样 Agent 接收到的是干净的输入文本，而不会带有 `@BotName` 前缀。机器人通过钉钉的 `IsInAtList` 字段可靠判断自己是否被提及，而不是手动解析文本。

### 群聊与单聊

| 特性 | 单聊 | 群聊 |
| --- | --- | --- |
| 触发方式 | 任意消息 | 默认需要 @ 提及 |
| 回复方式 | 直接回复 | 通过 session webhook 回复 |
| 上下文 | 按用户维持会话 | 按群组维持会话 |
