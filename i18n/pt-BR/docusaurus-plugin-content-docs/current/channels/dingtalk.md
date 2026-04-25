---
id: dingtalk
title: DingTalk
---

# DingTalk

O DingTalk e a plataforma de comunicacao corporativa da Alibaba, amplamente usada em ambientes de trabalho chineses. O PicoClaw usa o SDK do **Stream Mode** do DingTalk, que mantem uma conexao WebSocket persistente sem precisar de IP publico nem configuracao de webhook.

## Configuracao

### 1. Criar um App Interno

1. Acesse a [Plataforma Aberta do DingTalk](https://open.dingtalk.com/)
2. Clique em **Application Development** -> **Enterprise Internal Development** -> **Create Application**
3. Preencha o nome e a descricao do app

### 2. Obter Credenciais

1. Va em **Credentials & Basic Info** nas configuracoes do seu app
2. Copie o **Client ID** (AppKey) e o **Client Secret** (AppSecret)

### 3. Ativar o Recurso de Robo

1. Va em **App Features** -> **Robot**
2. Ative o recurso de robo
3. O robo funciona tanto em **chats de grupo** quanto em **chats privados**

### 4. Configurar Permissoes

Em **Permissions & Scopes**, garanta que as seguintes permissoes estejam concedidas:

- Receber mensagens
- Enviar mensagens

### 5. Configurar o PicoClaw

#### 1. Configuracao pelo WebUI

Recomendamos priorizar a configuracao pelo WebUI, pois e mais rapida e conveniente.

![WebUI DingTalk Connection Interface](/img/channels/webui_dingtalk.png)

Preencha, nesta ordem, o Client ID (`YOUR_CLIENT_ID`) e o Client Secret (`YOUR_CLIENT_SECRET`), depois clique em **Salvar**.

#### 2. Arquivos de Configuracao

Edite `~/.picoclaw/.security.yml`:

```yaml
dingtalk:
  settings:
    client_secret: YOUR_CLIENT_SECRET
```

Edite `~/.picoclaw/config.json`:

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

### 6. Executar

```bash
picoclaw gateway
```

## Referencia de Campos

| Campo | Tipo | Obrigatorio | Descricao |
| --- | --- | --- | --- |
| `client_id` | string | Sim | Client ID do app DingTalk (AppKey) |
| `client_secret` | string | Sim | Client Secret do app DingTalk (AppSecret) |
| `allow_from` | array | Nao | Lista branca de IDs de usuarios do DingTalk (vazio = permitir todos) |
| `group_trigger` | object | Nao | Configuracoes de acionamento em chat de grupo (veja [Campos Comuns dos Canais](../#common-channel-fields)) |
| `reasoning_channel_id` | string | Nao | Direcionar a saida de raciocinio para um chat separado |

## Como Funciona

### Stream Mode

O Stream Mode do DingTalk usa uma conexao WebSocket persistente mantida pelo SDK:

- **Nao precisa de IP publico**: o SDK se conecta de forma outbound aos servidores do DingTalk
- **Reconexao automatica**: o SDK lida com desconexoes e reconecta automaticamente
- **Entrega em tempo real**: as mensagens sao entregues instantaneamente pelo canal WebSocket

### Tratamento de Mensagens

- **Chats privados**: As mensagens sao recebidas diretamente
- **Chats de grupo**: O bot responde quando e @mencionado (configuravel via `group_trigger`)
- **Session Webhook**: Cada mensagem recebida traz uma URL `sessionWebhook` para respostas diretas
- **Tamanho maximo da mensagem**: 20.000 caracteres por mensagem (respostas mais longas sao truncadas automaticamente)

### Tratamento de Mencoes em Grupos

Quando o bot e @mencionado em um chat de grupo, o PicoClaw remove automaticamente as tags `@mention` iniciais da mensagem antes de passa-la ao agente. Isso garante que o agente receba o texto de entrada limpo, sem o prefixo `@BotName`. O bot usa o campo `IsInAtList` do DingTalk para detectar com confianca se foi mencionado, em vez de analisar o texto manualmente.

### Grupo vs. Chat Privado

| Recurso | Chat Privado | Chat de Grupo |
| --- | --- | --- |
| Gatilho | Qualquer mensagem | @mencao por padrao |
| Resposta | Resposta direta | Resposta via session webhook |
| Contexto | Sessao por usuario | Sessao por grupo |
