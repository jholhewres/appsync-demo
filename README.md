# 🚀 AppSync Real-Time Demo

Demo de conexão WebSocket com AWS AppSync usando autenticação SigV4 para receber eventos em tempo real.

## 📋 Quick Start

### 1. Abrir a Demo

```bash
# Abrir diretamente no navegador
open demo-appsync/index.html

# OU usar um servidor local
python3 -m http.server 8010
# Acesse: http://localhost:8010
```

### 2. Conectar

1. **Preencha os campos**:
   - **API URL**: `https://api.example.com`
   - **Agent ID**: `6426d565-ec6a-4cd3-bbc1-a1610df607aa`
   - **Visitor ID**: `visitor-1234` (ou qualquer ID único)

2. **Clique em "🔌 Conectar ao AppSync"**

3. **Aguarde a conexão** (você verá):
   ```
   ✅ Credenciais obtidas com sucesso!
   ✅ WebSocket conectado!
   ✅ Conexão confirmada pelo AppSync
   ✅ Subscription registrada com sucesso!
   ```

### 3. Testar Eventos

1. **Clique em "📤 Enviar Mensagem Teste"**
2. **Digite uma mensagem** (ex: "Olá!")
3. **Observe o evento sendo recebido em tempo real** 🎉

---

## 🏗️ Arquitetura Simplificada

```
Frontend ──GET /chat/:id──► Lambda ──AssumeRole──► STS
   │                           │
   │   ◄─ Credenciais IAM ─────┘
   │
   ├──WebSocket + SigV4──────────► AppSync GraphQL
   │                                     │
   │   ◄─── Subscription ACK ────────────┘
   │
   ├──POST /message──────────► Lambda ──Mutation──► AppSync
   │                                                   │
   │   ◄───────── Event (real-time) ──────────────────┘
```

---

## 🔑 Como Funciona?

### 1. Buscar Credenciais
```javascript
GET https://api.example.com/agent/{agent_id}/chat/{visitor_id}

// Resposta:
{
  "has_last_chat_closed": true,  // ✨ NOVO: indica se o último chat foi encerrado
  "session_id": "...",
  "web_socket": {
    "endpoint": "https://xxx.appsync-api.us-east-1.amazonaws.com/graphql",
    "credentials": {
      "access_key_id": "ASIA...",
      "secret_access_key": "...",
      "session_token": "...",
      "expiration": "2025-10-23T15:34:11Z"
    }
  },
  "messages": []
}
```

### 2. Gerar Assinatura SigV4

A assinatura AWS Signature Version 4 garante autenticidade e integridade:

```javascript
// 1. Criar canonical request (POST /graphql/connect)
// 2. Criar string-to-sign (AWS4-HMAC-SHA256)
// 3. Calcular assinatura (derivação de chave HMAC-SHA256)
// 4. Montar Authorization header
```

### 3. Conectar WebSocket

```javascript
wss://xxx.appsync-realtime-api.us-east-1.amazonaws.com/graphql
  ?header=eyJob3N0Ij...  // Header com assinatura SigV4
  &payload=e30=          // Payload vazio encodado
```

### 4. Registrar Subscription

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

### 5. Receber Eventos

```javascript
{
  "type": "data",
  "payload": {
    "data": {
      "onMessageReceived": {
        "id": "...",
        "text": "Olá!",
        "sender": "user",
        "createdAt": "2025-10-23T15:20:54.568Z"
      }
    }
  }
}
```

---

## 🔐 Segurança

### Credenciais Temporárias (STS)

- **Validade**: 1 hora
- **Escopo**: Apenas GraphQL do AppSync
- **Única por sessão**: Cada chat tem credenciais próprias

### Autenticação SigV4

- ✅ Garante autenticidade da origem
- ✅ Previne modificação da requisição
- ✅ Protege contra replay attacks
- ✅ Usa criptografia HMAC-SHA256

---

## 🚨 Troubleshooting

### Erro: `CORS blocked`

**Solução**: Abra o arquivo diretamente (`file://`) ou adicione sua origem no `serverless/custom.yml`:

```yaml
cors:
  origins:
    - http://localhost:8010  # Adicione sua origem aqui
```

### Erro: `ExpiredTokenException`

**Causa**: Credenciais expiradas (1 hora de validade)

**Solução**: Clique em "🔌 Conectar ao AppSync" novamente para buscar novas credenciais

### Erro: `BadRequestException: Signature does not match`

**Causa**: Assinatura SigV4 incorreta

**Verificar**:
- ✅ Método é `POST` (não `GET`)
- ✅ Headers em ordem alfabética
- ✅ Payload hash correto (`{}` para conexão)
- ✅ `x-amz-security-token` incluído

### Erro: `UnauthorizedException`

**Causa**: Subscription sem autorização

**Solução**: A subscription também precisa de assinatura SigV4 completa em `extensions.authorization`

---

## 📁 Estrutura de Arquivos

```
demo-appsync/
├── index.html              # Demo completa (HTML + CSS + JS)
├── README.md              # Este arquivo
└── DOCUMENTACAO.md        # Documentação técnica completa
```

---

## 📚 Documentação Completa

Para entender em detalhes como funciona a assinatura SigV4, fluxo completo, erros comuns e muito mais, veja:

👉 **[DOCUMENTACAO.md](./DOCUMENTACAO.md)**

A documentação completa inclui:
- 🔐 Explicação detalhada da assinatura SigV4
- 🏗️ Arquitetura e fluxo completo
- 🔧 Implementação passo a passo
- 🚨 Erros comuns e soluções
- 📊 Diagramas e exemplos
- 💡 Próximos passos

---

## ✅ Features

- [x] Autenticação SigV4 completa no navegador
- [x] WebSocket persistente com AppSync
- [x] GraphQL Subscriptions em tempo real
- [x] Credenciais temporárias via STS
- [x] Interface de chat responsiva
- [x] Validação de expiração de credenciais
- [x] Tratamento completo de erros
- [x] Estatísticas de eventos e mensagens
- [x] Logs detalhados para debugging
- [x] ✨ Detecção de chat anterior encerrado (`has_last_chat_closed`)
- [x] ✨ Mensagens contextuais para retorno de visitantes

---

## ✨ Novo: Detecção de Chat Anterior Encerrado

### Campo `has_last_chat_closed`

O campo `has_last_chat_closed` indica se o visitante teve um chat anterior que foi formalmente encerrado.

**Valores possíveis:**
- `true` - O último chat do visitante foi encerrado (status = "closed")
- `false` - O último chat não foi encerrado ou não existe chat anterior

**Comportamento no Demo:**

Quando o visitante se conecta, o sistema automaticamente:

1. **Se `has_last_chat_closed = true`:**
   ```
   💬 Bem-vindo de volta! Seu último chat foi encerrado. 
   Vamos iniciar uma nova conversa!
   ```
   - Ideal para mostrar mensagem de boas-vindas
   - Frontend pode limpar contexto anterior
   - Iniciar conversa "do zero"

2. **Se `has_last_chat_closed = false`:**
   ```
   👋 Continuando a conversa anterior...
   ```
   - Chat anterior ainda estava aberto/ativo
   - Manter contexto da conversa
   - Carregar histórico de mensagens

**Exemplo de Uso no Código:**

```javascript
const response = await axios.get(`${apiUrl}/agent/${agentId}/chat/${visitorId}`);
const data = response.data.data;

if (data.has_last_chat_closed) {
  // Mostrar boas-vindas para nova conversa
  showWelcomeMessage();
  clearPreviousContext();
} else {
  // Continuar conversa anterior
  loadChatHistory();
  showContinuationMessage();
}
```

---

## 🎯 Casos de Uso

### 1. Chat em Tempo Real
- Mensagens instantâneas entre visitante e atendente
- Indicadores de digitação
- Status de entrega

### 2. Notificações
- Alertas de novos tickets
- Atualizações de status
- Eventos do sistema

### 3. Dashboard Live
- Métricas em tempo real
- Atualizações de dados
- Sincronização multi-usuário

---

## 🔗 Endpoints

### Desenvolvimento
- **API**: `https://dev-api.example.com`
- **AppSync**: Auto-configurado via credenciais

### Produção
- **API**: `https://api.example.com`
- **AppSync**: Auto-configurado via credenciais

---

## 🛠️ Stack Tecnológica

### Frontend
- **Vanilla JavaScript** (sem frameworks)
- **Web Crypto API** (assinatura SigV4)
- **WebSocket** (conexão real-time)
- **Axios** (HTTP client)

### Backend
- **AWS Lambda** (serverless functions)
- **AWS AppSync** (GraphQL + WebSocket)
- **AWS STS** (credenciais temporárias)
- **Go** (linguagem do backend)

---

## 📊 Status

- ✅ **Desenvolvimento**: Completo
- ✅ **Testes**: Funcionando
- ✅ **Documentação**: Completa
- 🚀 **Produção**: Pronto para deploy

