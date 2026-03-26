---
id: credential-encryption
title: Credential Encryption
---

# Credential Encryption

PicoClaw can encrypt `api_key` values in your `model_list` configuration. Encrypted keys are stored as `enc://<base64>` strings and decrypted automatically at startup.

## Quick Start

1. Set your passphrase:

```bash
export PICOCLAW_KEY_PASSPHRASE="your-secure-passphrase"
```

2. Run the onboard command to encrypt your keys:

```bash
picoclaw onboard
```

3. Paste the resulting `enc://` value into your config:

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

## Supported Key Formats

| Format          | Example                                    | Description                          |
|-----------------|--------------------------------------------|--------------------------------------|
| Plaintext       | `sk-abc123...`                             | Raw API key (not recommended).       |
| File reference   | `file:///path/to/keyfile`                  | Read key from a file at startup.     |
| Encrypted       | `enc://c2FsdC4uLm5vbmNlLi4uY2lwaGVydGV4dA==` | Encrypted with passphrase + SSH key. |
| Empty           | `""`                                       | No key (for keyless endpoints).      |

## Cryptographic Design

PicoClaw uses a **two-factor** approach: both a passphrase and your SSH private key are required to decrypt credentials.

### Key Derivation

```
sshHash   = SHA256(ssh_private_key)
ikm       = HMAC-SHA256(sshHash, passphrase)
aes_key   = HKDF-SHA256(ikm, salt, "picoclaw-credential-v1", 32 bytes)
```

1. The SSH private key is hashed with SHA-256.
2. The hash is combined with the passphrase via HMAC-SHA256 to produce input keying material (IKM).
3. HKDF-SHA256 derives the final 32-byte AES key using a random salt and the info string `picoclaw-credential-v1`.

### Encryption

- **Algorithm:** AES-256-GCM
- **Wire format:** `enc://<base64(salt[16] + nonce[12] + ciphertext)>`

The salt (16 bytes) and nonce (12 bytes) are prepended to the ciphertext and then Base64-encoded.

## Two-Factor Security

Decryption requires **both** factors:

| Factor         | Source                         | Purpose                              |
|----------------|--------------------------------|--------------------------------------|
| Passphrase     | `PICOCLAW_KEY_PASSPHRASE` env  | Something you know.                  |
| SSH private key| `~/.ssh/picoclaw_ed25519.key`  | Something you have.                  |

If either factor is missing or incorrect, decryption fails and PicoClaw will not start.

## Threat Model

| Threat                          | Mitigated? | Notes                                                        |
|---------------------------------|------------|--------------------------------------------------------------|
| Config file leaked              | Yes        | Encrypted keys are unusable without both factors.            |
| SSH key stolen (no passphrase)  | Yes        | Attacker still needs `PICOCLAW_KEY_PASSPHRASE`.              |
| Passphrase leaked               | Yes        | Attacker still needs the SSH private key.                    |
| Both factors compromised        | No         | Full compromise — rotate keys immediately.                   |
| Brute-force on passphrase       | Partial    | HKDF slows derivation; use a strong passphrase.              |

## Environment Variables

| Variable                   | Required | Default                            | Description                          |
|----------------------------|----------|------------------------------------|--------------------------------------|
| `PICOCLAW_KEY_PASSPHRASE`  | Yes      | —                                  | Passphrase for key derivation.       |
| `PICOCLAW_SSH_KEY_PATH`    | No       | `~/.ssh/picoclaw_ed25519.key`      | Path to the SSH private key file.    |

## Migration

To migrate an encrypted configuration to a new machine:

1. Copy your `~/.picoclaw/config.json` to the new machine.
2. Set the same `PICOCLAW_KEY_PASSPHRASE` environment variable.
3. Copy the SSH private key (`~/.ssh/picoclaw_ed25519.key`) to the new machine.

All three pieces must match for decryption to succeed.
