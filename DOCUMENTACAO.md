# 📚 Documentação - AppSync Real-Time com AWS SigV4

## 🎯 Visão Geral

Esta documentação explica a implementação completa de uma conexão WebSocket com AWS AppSync usando **autenticação AWS Signature Version 4 (SigV4)** no navegador, permitindo que aplicações frontend recebam eventos em tempo real através de GraphQL Subscriptions.

---

## 🏗️ Arquitetura

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Frontend  │ ◄─────► │    Lambda    │ ◄─────► │   AppSync   │
│  (Browser)  │         │  (chat_*)    │         │   GraphQL   │
└─────────────┘         └──────────────┘         └─────────────┘
      │                        │                         │
      │ 1. GET /chat/:id       │                         │
      │───────────────────────►│                         │
      │                        │ 2. AssumeRole (STS)     │
      │                        │────────────────────────►│
      │ 3. Credenciais IAM     │                         │
      │◄───────────────────────│                         │
      │                        │                         │
      │ 4. WebSocket + SigV4   │                         │
      │────────────────────────────────────────────────►│
      │                        │                         │
      │ 5. Subscription ACK    │                         │
      │◄────────────────────────────────────────────────│
      │                        │                         │
      │ 6. POST /message       │                         │
      │───────────────────────►│                         │
      │                        │ 7. sendMessage mutation │
      │                        │────────────────────────►│
      │                        │                         │
      │ 8. Event (real-time)   │                         │
      │◄────────────────────────────────────────────────│
```

---

## 🔐 AWS Signature Version 4 (SigV4)

### O que é SigV4?

AWS Signature Version 4 é o protocolo de autenticação usado pela AWS para validar requisições aos seus serviços. Ele garante:

1. **Autenticidade**: Confirma que a requisição vem de quem diz ser
2. **Integridade**: Garante que a requisição não foi modificada no caminho
3. **Não-repúdio**: O remetente não pode negar que enviou a requisição

### Por que é complexo?

- Requer cálculos criptográficos precisos (SHA-256, HMAC)
- Formato específico e rígido (ordem alfabética de headers, formatação exata)
- Timestamps síncronos (diferença de relógio pode causar rejeição)
- Para credenciais temporárias, o `x-amz-security-token` deve ser incluído

---

## 🔑 Fluxo de Autenticação

### 1. Obtenção de Credenciais Temporárias

**Endpoint**: `GET /agent/{agent_id}/chat/{visitor_id}`

**O que acontece**:
```go
// Lambda chat_create
func (a *Adapter) CreateSessionCredentials(ctx context.Context, sessionID string) (*SessionCredentials, error) {
    // 1. Assume a role AppSyncSessionRole via STS
    input := &sts.AssumeRoleInput{
        RoleArn:         aws.String("arn:aws:iam::...:role/AppSyncSessionRole"),
        RoleSessionName: aws.String("appsync-session-" + sessionID),
        DurationSeconds: aws.Int32(3600), // 1 hora
    }
    
    result, err := a.stsClient.AssumeRole(ctx, input)
    
    // 2. Retorna credenciais temporárias
    return &SessionCredentials{
        AccessKeyId:     result.Credentials.AccessKeyId,
        SecretAccessKey: result.Credentials.SecretAccessKey,
        SessionToken:    result.Credentials.SessionToken,
        Expiration:      result.Credentials.Expiration,
    }, nil
}
```

**Resposta**:
```json
{
  "success": true,
  "data": {
    "session_id": "133c131b-b964-4de5-81f4-0cc8bb682931",
    "web_socket": {
      "endpoint": "https://xxx.appsync-api.us-east-1.amazonaws.com/graphql",
      "region": "us-east-1",
      "credentials": {
        "access_key_id": "ASIA...",
        "secret_access_key": "...",
        "session_token": "IQoJb3JpZ2luX...",
        "expiration": "2025-10-23T15:34:11Z"
      }
    }
  }
}
```

---

## 🔧 Implementação SigV4 no Navegador

### Etapa 1: Gerar Timestamp no Formato ISO8601

```javascript
getAmzDate(date) {
    const pad = (n) => String(n).padStart(2, '0');
    return date.getUTCFullYear() +
        pad(date.getUTCMonth() + 1) +
        pad(date.getUTCDate()) + 'T' +
        pad(date.getUTCHours()) +
        pad(date.getUTCMinutes()) +
        pad(date.getUTCSeconds()) + 'Z';
}
// Resultado: "20251023T151539Z"
```

**⚠️ IMPORTANTE**: 
- Sempre usar UTC
- Formato exato: `YYYYMMDDTHHMMSSZ`
- Sem hífens, sem dois-pontos (exceto no T)

---

### Etapa 2: Criar Canonical Request

O **Canonical Request** é uma representação padronizada da requisição:

```javascript
async createSigV4Authorization(url) {
    // Informações básicas
    const method = 'POST';  // AppSync WebSocket usa POST!
    const canonicalUri = url.pathname;  // Ex: /graphql/connect
    const canonicalQuerystring = '';
    
    // Headers em ORDEM ALFABÉTICA!
    const canonicalHeaders = 
        'host:' + url.host + '\n' +
        'x-amz-date:' + amzDate + '\n' +
        'x-amz-security-token:' + this.credentials.session_token + '\n';
    
    const signedHeaders = 'host;x-amz-date;x-amz-security-token';
    
    // Hash SHA-256 do payload
    const payloadHash = await this.sha256('{}');  // Para conexão: {}
    
    // Montar canonical request
    const canonicalRequest = 
        method + '\n' +
        canonicalUri + '\n' +
        canonicalQuerystring + '\n' +
        canonicalHeaders + '\n' +
        signedHeaders + '\n' +
        payloadHash;
}
```

**Exemplo de Canonical Request**:
```
POST
/graphql/connect

host:xxx.appsync-api.us-east-1.amazonaws.com
x-amz-date:20251023T151539Z
x-amz-security-token:IQoJb3JpZ2luX2VjEI...

host;x-amz-date;x-amz-security-token
44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a
```

**⚠️ CUIDADOS**:
1. Headers devem estar em **ordem alfabética**
2. Cada header termina com `\n`
3. Lista de `signedHeaders` usa `;` como separador
4. Payload hash para conexão é hash de `{}`, não `''`

---

### Etapa 3: Criar String-to-Sign

```javascript
const algorithm = 'AWS4-HMAC-SHA256';
const dateStamp = amzDate.substring(0, 8);  // "20251023"
const credentialScope = dateStamp + '/' + region + '/' + service + '/aws4_request';
const canonicalRequestHash = await this.sha256(canonicalRequest);

const stringToSign = 
    algorithm + '\n' +
    amzDate + '\n' +
    credentialScope + '\n' +
    canonicalRequestHash;
```

**Exemplo**:
```
AWS4-HMAC-SHA256
20251023T151539Z
20251023/us-east-1/appsync/aws4_request
1b09af890fe75f9c9b772eb730a3cc832e8dacda34942c1c6c25eabe54607fea
```

---

### Etapa 4: Calcular Assinatura

A assinatura usa uma **chave derivada** através de múltiplos HMAC-SHA256:

```javascript
async getSignatureKey(key, dateStamp, regionName, serviceName) {
    const kDate = await this.hmacSha256('AWS4' + key, dateStamp);
    const kRegion = await this.hmacSha256(kDate, regionName);
    const kService = await this.hmacSha256(kRegion, serviceName);
    const kSigning = await this.hmacSha256(kService, 'aws4_request');
    return kSigning;
}

async hmacSha256(key, message) {
    const keyData = typeof key === 'string' ? new TextEncoder().encode(key) : key;
    const cryptoKey = await crypto.subtle.importKey(
        'raw',
        keyData,
        { name: 'HMAC', hash: 'SHA-256' },
        false,
        ['sign']
    );
    const signature = await crypto.subtle.sign(
        'HMAC',
        cryptoKey,
        new TextEncoder().encode(message)
    );
    return new Uint8Array(signature);
}
```

**Processo de derivação da chave**:
```
SecretKey = "wJalrXUtnFEMI/K7MDENG+bPxRfiCYEXAMPLEKEY"
    ↓ HMAC("AWS4" + SecretKey, "20251023")
kDate = 0x8e93...
    ↓ HMAC(kDate, "us-east-1")
kRegion = 0x3f7a...
    ↓ HMAC(kRegion, "appsync")
kService = 0x62ef...
    ↓ HMAC(kService, "aws4_request")
kSigning = 0x9a41...
```

**Finalmente, calcular a assinatura**:
```javascript
const signature = await this.hmacSha256Hex(signingKey, stringToSign);
```

---

### Etapa 5: Montar Authorization Header

```javascript
const authorizationHeader = 
    'AWS4-HMAC-SHA256 ' +
    'Credential=' + accessKeyId + '/' + credentialScope + ', ' +
    'SignedHeaders=' + signedHeaders + ', ' +
    'Signature=' + signature;
```

**Exemplo**:
```
AWS4-HMAC-SHA256 Credential=ASIA2IYSJJKSZZSW4WDR/20251023/us-east-1/appsync/aws4_request, SignedHeaders=host;x-amz-date;x-amz-security-token, Signature=7b8ccd98cfe8662f5e2c84dec2640c40d0da1f4aeb45bf249c3df935f0835858
```

---

## 🔌 Conexão WebSocket

### 1. Preparar Header Encodado

```javascript
const header = {
    host: 'xxx.appsync-api.us-east-1.amazonaws.com',  // Host ORIGINAL (appsync-api)
    'x-amz-date': '20251023T151539Z',
    'x-amz-security-token': 'IQoJb3JpZ2luX...',
    Authorization: 'AWS4-HMAC-SHA256 Credential=...'
};

const headerEncoded = btoa(JSON.stringify(header));
const payloadEncoded = btoa(JSON.stringify({}));
```

**⚠️ IMPORTANTE**:
- A assinatura usa `appsync-api.us-east-1.amazonaws.com` (endpoint original)
- O WebSocket conecta em `appsync-realtime-api.us-east-1.amazonaws.com`
- O header deve usar o domínio **original** (`appsync-api`)

---

### 2. Conectar ao WebSocket Realtime

```javascript
const realtimeEndpoint = endpoint.replace('appsync-api', 'appsync-realtime-api');
const wsUrl = `${realtimeEndpoint.replace('https', 'wss')}?header=${headerEncoded}&payload=${payloadEncoded}`;

this.ws = new WebSocket(wsUrl, ['graphql-ws']);
```

**URL Final**:
```
wss://xxx.appsync-realtime-api.us-east-1.amazonaws.com/graphql?header=eyJob3N0Ijoi...&payload=e30=
```

---

### 3. Inicializar Conexão

```javascript
this.ws.onopen = () => {
    // Enviar connection_init
    this.ws.send(JSON.stringify({ type: 'connection_init' }));
};

this.ws.onmessage = async (event) => {
    const message = JSON.parse(event.data);
    
    if (message.type === 'connection_ack') {
        // ✅ Conexão confirmada!
        await this.sendSubscription();
    }
};
```

---

## 📡 GraphQL Subscription

### 1. Preparar Query GraphQL

```javascript
const graphqlData = {
    query: `
        subscription OnMessageReceived($sessionId: ID!) {
            onMessageReceived(sessionId: $sessionId) {
                id
                sessionId
                text
                sender
                createdAt
            }
        }
    `,
    variables: {
        sessionId: this.sessionId
    }
};
```

---

### 2. Assinar com SigV4 (Novamente!)

A subscription também precisa de **autenticação separada**:

```javascript
// IMPORTANTE: Para subscription, usar /graphql (não /connect)
const apiUrl = new URL(endpoint);
apiUrl.pathname = '/graphql';

// Payload hash deve ser do JSON da query!
const dataString = JSON.stringify(graphqlData);
const authHeader = await this.createSigV4AuthorizationWithPayload(apiUrl, dataString);
```

**⚠️ DIFERENÇAS**:
- **Conexão**: assina `/graphql/connect` com payload `{}`
- **Subscription**: assina `/graphql` com payload da query GraphQL

---

### 3. Enviar Subscription

```javascript
const subscriptionMessage = {
    id: 'sub-1761232804034-fr8errqwp',
    type: 'start',
    payload: {
        data: dataString,  // JSON string da query
        extensions: {
            authorization: {
                host: 'xxx.appsync-api.us-east-1.amazonaws.com',
                'x-amz-date': '20251023T151539Z',
                'x-amz-security-token': 'IQoJb3JpZ2luX...',
                Authorization: 'AWS4-HMAC-SHA256 Credential=...'
            }
        }
    }
};

this.ws.send(JSON.stringify(subscriptionMessage));
```

---

### 4. Receber ACK

```javascript
this.ws.onmessage = async (event) => {
    const message = JSON.parse(event.data);
    
    if (message.type === 'start_ack') {
        // ✅ Subscription registrada com sucesso!
        console.log('Ouvindo eventos...');
    }
    
    if (message.type === 'data') {
        // 🎉 Evento recebido!
        const msg = message.payload.data.onMessageReceived;
        console.log('Mensagem:', msg.text);
    }
};
```

---

## 🎯 Fluxo Completo

### 1. Frontend busca credenciais

```bash
GET https://api.example.com/agent/{agent_id}/chat/{visitor_id}
```

**Resposta**: Credenciais IAM temporárias (válidas por 1 hora)

---

### 2. Frontend conecta ao AppSync

1. Gerar assinatura SigV4 para `/graphql/connect`
2. Conectar WebSocket em `appsync-realtime-api`
3. Enviar `connection_init`
4. Receber `connection_ack`

---

### 3. Frontend registra subscription

1. Gerar assinatura SigV4 para `/graphql` com payload da query
2. Enviar mensagem `start` com subscription
3. Receber `start_ack`

---

### 4. Lambda envia mensagem

```bash
POST https://api.example.com/chat/{session_id}/message
Content-Type: application/json

{
  "message": "Olá!"
}
```

**Lambda**:
```go
// 1. Salvar no banco de dados
messageRepo.Create(userMessage)

// 2. Enviar para fila SQS (processamento IA)
sqsAdapter.SendMessage(queueURL, aiMessage)

// 3. Enviar para AppSync (real-time)
appSyncAdapter.SendMessage(ctx, appsync.SendMessageInput{
    SessionID: chat.SessionID,
    Text:      input.Message,
    Sender:    appsync.SenderUser,
})
```

---

### 5. AppSync dispara evento

**Mutation no AppSync**:
```graphql
mutation SendMessage($sessionId: ID!, $text: String!, $sender: String!) {
    sendMessage(sessionId: $sessionId, text: $text, sender: $sender) {
        id
        sessionId
        text
        sender
        createdAt
    }
}
```

**Subscription escuta**:
```graphql
subscription OnMessageReceived($sessionId: ID!) {
    onMessageReceived(sessionId: $sessionId) {
        id
        text
        sender
        createdAt
    }
}
```

---

### 6. Frontend recebe evento

```javascript
{
  "type": "data",
  "payload": {
    "data": {
      "onMessageReceived": {
        "id": "e6330d77-bd92-4f64-9aec-17d6a3110204",
        "sessionId": "133c131b-b964-4de5-81f4-0cc8bb682931",
        "text": "Olá!",
        "sender": "user",
        "createdAt": "2025-10-23T15:20:54.568Z"
      }
    }
  }
}
```

---

## 🔒 Segurança

### Permissões IAM

**AppSyncSessionRole** (assumida pelo Lambda):
```yaml
Policies:
  - PolicyName: AppSyncSubscriptionPolicy
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Action:
            - appsync:GraphQL
            - appsync:Subscribe
            - appsync:Connect
          Resource: !Sub '${AppSyncGraphQLApi.Arn}/*'
```

**Lambda chat_create** (gera credenciais):
```yaml
iamRoleStatements:
  - Effect: "Allow"
    Action:
      - "sts:AssumeRole"
      - "sts:TagSession"
    Resource:
      - !GetAtt AppSyncSessionRole.Arn
```

---

### Credenciais Temporárias

- **Validade**: 1 hora (3600 segundos)
- **Escopo**: Apenas operações GraphQL do AppSync
- **Session ID**: Único por chat/visitante
- **Não reutilizáveis**: Cada sessão gera novas credenciais

---

### Validação de Expiração

```javascript
validateCredentials() {
    const expirationDate = new Date(this.credentials.expiration);
    const now = new Date();
    
    if (expirationDate <= now) {
        throw new Error('Credenciais expiradas! Busque novas credenciais.');
    }
    
    const minutesUntilExpiration = Math.floor((expirationDate - now) / 1000 / 60);
    
    if (minutesUntilExpiration < 5) {
        console.warn('⚠️ Credenciais expiram em menos de 5 minutos!');
    }
}
```

---

## 🚨 Erros Comuns

### 1. `ExpiredTokenException`

**Causa**: Credenciais expiradas ou timestamp incorreto

**Solução**:
- Verificar se credenciais ainda são válidas
- Buscar novas credenciais na API
- Garantir que relógio do sistema está sincronizado

---

### 2. `BadRequestException: Signature does not match`

**Causa**: Assinatura SigV4 incorreta

**Possíveis problemas**:
- ❌ Headers não em ordem alfabética
- ❌ Método errado (deve ser `POST`, não `GET`)
- ❌ Payload hash incorreto
- ❌ `x-amz-security-token` não incluído nos `signedHeaders`
- ❌ Timestamp malformado

**Solução**:
```javascript
// ✅ Correto
const canonicalHeaders = 
    'host:xxx.appsync-api.us-east-1.amazonaws.com\n' +
    'x-amz-date:20251023T151539Z\n' +
    'x-amz-security-token:IQoJb3JpZ2luX...\n';

const signedHeaders = 'host;x-amz-date;x-amz-security-token';
```

---

### 3. `UnauthorizedException: Valid authorization header not provided`

**Causa**: Subscription sem autorização

**Solução**: Incluir assinatura SigV4 completa em `extensions.authorization`:
```javascript
extensions: {
    authorization: {
        host: '...',
        'x-amz-date': '...',
        'x-amz-security-token': '...',
        Authorization: 'AWS4-HMAC-SHA256 Credential=...'
    }
}
```

---

### 4. `HttpNotFoundException (400)`

**Causa**: Caminho WebSocket incorreto

**Solução**:
- Assinatura: `appsync-api.../graphql/connect`
- WebSocket: `appsync-realtime-api.../graphql`
- Não adicionar `/graphql` duas vezes!

---

## 📦 Dependências

### Navegador (Web Crypto API)

```javascript
// ✅ Disponível nativamente no navegador
crypto.subtle.digest('SHA-256', data)
crypto.subtle.importKey(...)
crypto.subtle.sign('HMAC', key, data)
```

**Compatibilidade**:
- ✅ Chrome 37+
- ✅ Firefox 34+
- ✅ Safari 11+
- ✅ Edge 79+

---

### HTTP Client

```html
<!-- Axios para requisições HTTP -->
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
```

---

## 🎨 Interface da Demo

### Estrutura HTML

```html
<div class="container">
    <div class="header">
        <h1>🔌 AppSync Real-Time Demo</h1>
        <span class="status-badge">DESCONECTADO</span>
    </div>
    
    <div class="messages-area">
        <!-- Mensagens do chat aqui -->
    </div>
    
    <div class="input-area">
        <input type="text" class="message-input" placeholder="Digite sua mensagem...">
        <button class="btn-send">Enviar</button>
    </div>
</div>
```

---

### Event Handlers

```javascript
// Conectar
document.getElementById('connectBtn').addEventListener('click', async () => {
    const success = await client.fetchCredentials(apiUrl, agentId, visitorId);
    if (success) {
        await client.connectToAppSync();
    }
});

// Enviar mensagem
document.getElementById('sendMsgBtn').addEventListener('click', async () => {
    await client.sendMessage(text);
});

// Receber evento
client.ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    if (message.type === 'data') {
        displayMessage(message.payload.data.onMessageReceived);
    }
};
```

---

## 📊 Estatísticas

A demo rastreia:
- **Eventos Recebidos**: Contador de mensagens via subscription
- **Mensagens Enviadas**: Contador de mensagens enviadas via API
- **Status da Conexão**: Desconectado / Conectando / Conectado

---

## 🎓 Conceitos Importantes

### 1. WebSocket vs HTTP

| Característica | HTTP | WebSocket |
|----------------|------|-----------|
| Conexão | Stateless | Stateful (persistente) |
| Direção | Cliente → Servidor | Bidirecional |
| Latência | Alta (polling) | Baixa (push) |
| Uso | Requisições pontuais | Comunicação real-time |

---

### 2. GraphQL Subscription

Subscriptions GraphQL são **event-driven**:

```graphql
# 1. Cliente se inscreve
subscription {
    onMessageReceived(sessionId: "123") {
        text
    }
}

# 2. Servidor dispara mutation
mutation {
    sendMessage(sessionId: "123", text: "Olá") {
        id
    }
}

# 3. AppSync envia evento para subscribers
# { "onMessageReceived": { "text": "Olá" } }
```

---

### 3. AWS STS (Security Token Service)

Gera **credenciais temporárias** para acesso controlado:

```
Credenciais Fixas (Lambda)
         ↓
    AssumeRole (STS)
         ↓
Credenciais Temporárias (Frontend)
```

**Vantagens**:
- ✅ Segurança: Credenciais expiram automaticamente
- ✅ Controle: Permissões específicas por role
- ✅ Rastreabilidade: Cada sessão tem credenciais únicas

---

## 🔄 Ciclo de Vida

```
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │ 1. GET /chat/:id (buscar credenciais)
       ↓
┌──────────────┐
│   Lambda     │ 2. AssumeRole (STS)
│ chat_create  │
└──────┬───────┘
       │ 3. Retorna credenciais temporárias
       ↓
┌─────────────┐
│  Frontend   │ 4. Conecta WebSocket + SigV4
└──────┬──────┘    5. Registra subscription
       │
       │ 6. POST /message (enviar mensagem)
       ↓
┌──────────────┐
│   Lambda     │ 7. SendMessage mutation (AppSync)
│ chat_send    │
└──────┬───────┘
       │ 8. AppSync dispara evento
       ↓
┌─────────────┐
│  Frontend   │ 9. Recebe evento via WebSocket
└─────────────┘
```

---

## ✅ Checklist de Implementação

### Backend (Lambda)

- [x] Endpoint para gerar credenciais temporárias
- [x] AssumeRole via STS para AppSyncSessionRole
- [x] Endpoint para enviar mensagens
- [x] Mutation GraphQL para AppSync
- [x] Permissões IAM corretas

### Frontend

- [x] Implementação completa de SigV4
- [x] Validação de credenciais e expiração
- [x] Conexão WebSocket com AppSync
- [x] Subscription GraphQL
- [x] Interface de chat
- [x] Tratamento de erros

### AppSync

- [x] Schema GraphQL com Mutation e Subscription
- [x] Autenticação AWS_IAM
- [x] Role de sessão com permissões corretas
- [x] Resolver para sendMessage

---

## 🚀 Próximos Passos

### Melhorias Possíveis

1. **Renovação Automática de Credenciais**
   ```javascript
   setInterval(() => {
       if (minutesUntilExpiration < 5) {
           refreshCredentials();
       }
   }, 60000);
   ```

2. **Reconexão Automática**
   ```javascript
   ws.onclose = () => {
       setTimeout(() => reconnect(), 5000);
   };
   ```

3. **Queue de Mensagens Offline**
   ```javascript
   if (!ws || ws.readyState !== WebSocket.OPEN) {
       messageQueue.push(message);
   }
   ```

4. **Typing Indicators**
   ```graphql
   subscription OnUserTyping($sessionId: ID!) {
       onUserTyping(sessionId: $sessionId) {
           userId
           isTyping
       }
   }
   ```



