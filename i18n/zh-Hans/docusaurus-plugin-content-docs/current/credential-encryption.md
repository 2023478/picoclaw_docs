---
id: credential-encryption
title: 凭证加密
---

# 凭证加密

PicoClaw 可以加密 `model_list` 配置中的 `api_key` 值。加密后的密钥以 `enc://<base64>` 字符串形式存储，在启动时自动解密。

## 快速开始

1. 设置密码短语：

```bash
export PICOCLAW_KEY_PASSPHRASE="your-secure-passphrase"
```

2. 运行 onboard 命令加密密钥：

```bash
picoclaw onboard
```

3. 将生成的 `enc://` 值粘贴到配置中：

```json
{
  "model_list": [
    {
      "model_name": "gpt-4",
      "api_key": "enc://c2FsdC4uLm5vbmNlLi4uY2lwaGVydGV4dA=="
    }
  ]
}
```

## 支持的密钥格式

| 格式           | 示例                                       | 说明                          |
|----------------|--------------------------------------------|-------------------------------|
| 明文           | `sk-abc123...`                             | 原始 API 密钥（不推荐）。      |
| 文件引用       | `file:///path/to/keyfile`                  | 启动时从文件读取密钥。          |
| 加密           | `enc://c2FsdC4uLm5vbmNlLi4uY2lwaGVydGV4dA==` | 使用密码短语 + SSH 密钥加密。  |
| 空值           | `""`                                       | 无密钥（用于免密端点）。        |

## 加密设计

PicoClaw 采用**双因子**方案：解密凭证需要同时提供密码短语和 SSH 私钥。

### 密钥派生

```
sshHash   = SHA256(ssh_private_key)
ikm       = HMAC-SHA256(sshHash, passphrase)
aes_key   = HKDF-SHA256(ikm, salt, "picoclaw-credential-v1", 32 bytes)
```

1. 使用 SHA-256 对 SSH 私钥进行哈希。
2. 通过 HMAC-SHA256 将哈希与密码短语组合，生成输入密钥材料（IKM）。
3. 使用 HKDF-SHA256，结合随机盐和信息字符串 `picoclaw-credential-v1`，派生最终的 32 字节 AES 密钥。

### 加密算法

- **算法：** AES-256-GCM
- **传输格式：** `enc://<base64(salt[16] + nonce[12] + ciphertext)>`

盐（16 字节）和随机数（12 字节）前置于密文之前，然后进行 Base64 编码。

## 双因子安全

解密需要**两个**因子同时存在：

| 因子         | 来源                           | 用途               |
|--------------|--------------------------------|--------------------|
| 密码短语     | `PICOCLAW_KEY_PASSPHRASE` 环境变量 | 你所知道的东西。   |
| SSH 私钥     | `~/.ssh/picoclaw_ed25519.key`  | 你所拥有的东西。   |

如果任一因子缺失或不正确，解密将失败，PicoClaw 将无法启动。

## 威胁模型

| 威胁                         | 是否缓解？ | 备注                                                      |
|------------------------------|-----------|-----------------------------------------------------------|
| 配置文件泄露                  | 是        | 加密密钥在没有两个因子的情况下无法使用。                      |
| SSH 密钥被盗（无密码短语）     | 是        | 攻击者仍需要 `PICOCLAW_KEY_PASSPHRASE`。                   |
| 密码短语泄露                  | 是        | 攻击者仍需要 SSH 私钥。                                    |
| 两个因子均被泄露              | 否        | 完全妥协 — 请立即轮换密钥。                                 |
| 暴力破解密码短语              | 部分      | HKDF 减缓派生速度；请使用强密码短语。                        |

## 环境变量

| 变量                       | 必需 | 默认值                            | 说明                          |
|----------------------------|------|-----------------------------------|-------------------------------|
| `PICOCLAW_KEY_PASSPHRASE`  | 是   | —                                 | 用于密钥派生的密码短语。        |
| `PICOCLAW_SSH_KEY_PATH`    | 否   | `~/.ssh/picoclaw_ed25519.key`     | SSH 私钥文件路径。              |

## 迁移

将加密配置迁移到新机器：

1. 将 `~/.picoclaw/config.json` 复制到新机器。
2. 设置相同的 `PICOCLAW_KEY_PASSPHRASE` 环境变量。
3. 将 SSH 私钥（`~/.ssh/picoclaw_ed25519.key`）复制到新机器。

三者必须完全匹配，解密才能成功。
