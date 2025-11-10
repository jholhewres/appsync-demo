# 🎯 Como Testar o Evento de Agent Toggle

## ✅ O que foi implementado?

### 1. **Backend (agent-factory-orchestrator)**
- ✅ Novo tipo GraphQL: `AgentToggle`
- ✅ Nova mutation: `notifyAgentToggle`
- ✅ Nova subscription: `onAgentToggle(sessionId: ID!)` - para visitantes
- ✅ Nova subscription: `onAllAgentToggles` - para managers
- ✅ Resolver AppSync configurado
- ✅ Service implementado para enviar notificação

### 2. **Frontend (demo-appsync)**
- ✅ Subscription `onAgentToggle` configurada (modo visitante)
- ✅ Subscription `onAllAgentToggles` configurada (modo manager)
- ✅ Handler de eventos para mostrar mensagens no chat

---

## 🚀 Como Testar

### **Passo 1: Deploy das alterações**

```bash
cd /home/jhol/HostGator/agent-factory-orchestrator

# Build completo
make build

# Deploy para dev
make deploy -s dev
```

### **Passo 2: Abrir o Demo AppSync**

```bash
# Opção 1: Abrir direto no navegador
open /home/jhol/HostGator/demo-appsync/index.html

# Opção 2: Usar servidor local
cd /home/jhol/HostGator/demo-appsync
python3 -m http.server 8010
# Acesse: http://localhost:8010
```

### **Passo 3: Conectar ao AppSync**

**MODO VISITANTE (Ouvir uma sessão específica):**
1. Preencha os campos:
   - **API URL**: `https://dev-agent-factory-api.hostgator.io`
   - **Agent ID**: (ID do seu agente)
   - **Visitor ID**: `visitor-test-123`
2. Clique em **"🔌 Conectar ao AppSync"**
3. Aguarde as confirmações:
   ```
   ✅ Credenciais obtidas!
   ✅ Conectado ao AppSync!
   ✅ Subscription onMessageReceived registrada
   ✅ Subscription onAgentToggle registrada (agente humano/IA)
   ```

**MODO MANAGER (Ouvir todas as sessões):**
1. Marque o checkbox **"👤 Modo Manager (ouvir todas as sessions)"**
2. Preencha:
   - **API URL**: `https://dev-agent-factory-api.hostgator.io`
3. Clique em **"🔌 Conectar ao AppSync"**
4. Aguarde as confirmações:
   ```
   ✅ Credenciais de manager obtidas!
   ✅ Conectado ao AppSync!
   ✅ Subscription onAllMessagesReceived registrada (modo manager)
   ✅ Subscription onAllAgentToggles registrada (modo manager)
   ```

### **Passo 4: Testar o Toggle do Agente**

Agora você precisa chamar o endpoint para alternar o agente:

```bash
# Exemplo: Agente HUMANO assume (active=false)
curl -X PATCH "https://dev-agent-factory-api.hostgator.io/manager/chats/{chat_id}/agent-active" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {seu_token}" \
  -d '{"active": false}'

# Resposta esperada no demo:
# 👤 {Nome do Usuário} (Agente Humano) assumiu a conversa
```

```bash
# Exemplo: Agente de IA assume (active=true)
curl -X PATCH "https://dev-agent-factory-api.hostgator.io/manager/chats/{chat_id}/agent-active" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {seu_token}" \
  -d '{"active": true}'

# Resposta esperada no demo:
# 🤖 Agente de IA assumiu a conversa
```

---

## 📊 Eventos Esperados

### **Quando Agente Humano assume (active=false):**
```json
{
  "sessionId": "session-123",
  "agentActive": false,
  "userId": 42,
  "userName": "João Silva",
  "timestamp": "2025-11-06T19:30:00Z"
}
```

**Mensagem no chat:**
```
👤 João Silva (Agente Humano) assumiu a conversa
```

### **Quando Agente de IA assume (active=true):**
```json
{
  "sessionId": "session-123",
  "agentActive": true,
  "userId": null,
  "userName": "",
  "timestamp": "2025-11-06T19:35:00Z"
}
```

**Mensagem no chat:**
```
🤖 Agente de IA assumiu a conversa
```

---

## 🔍 Troubleshooting

### Erro: "Cannot query field 'onAgentToggle'"
**Causa:** O schema GraphQL do AppSync não foi atualizado ainda

**Solução:** 
1. Faça deploy novamente: `make deploy -s dev`
2. Aguarde alguns minutos para propagação da infraestrutura
3. Reconecte no demo

### Não recebo o evento no frontend
**Verificações:**
1. ✅ Deploy foi bem-sucedido?
2. ✅ Subscription `onAgentToggle` foi registrada? (veja no log do chat)
3. ✅ `session_id` do chat corresponde ao que você conectou?
4. ✅ O endpoint `PATCH /manager/chats/{chat_id}/agent-active` retornou 200?

### Como ver logs do backend?
```bash
# Ver logs da função agent_chat_toggle
serverless logs -f agent_chat_toggle -s dev --tail

# Ver logs do AppSync
# Acesse: AWS Console > AppSync > agent-factory-orchestrator-dev-chat-api > Logs
```

---

## 🎨 Schema GraphQL Completo

```graphql
type AgentToggle {
    sessionId: ID!
    agentActive: Boolean!
    userId: Int
    userName: String
    timestamp: AWSDateTime!
}

type Mutation {
    notifyAgentToggle(
        sessionId: ID!, 
        agentActive: Boolean!, 
        userId: Int, 
        userName: String
    ): AgentToggle
}

type Subscription {
    # Para visitantes (escuta apenas sua própria sessão)
    onAgentToggle(sessionId: ID!): AgentToggle
        @aws_subscribe(mutations: ["notifyAgentToggle"])
        @aws_api_key
        @aws_iam
    
    # Para managers (escuta todas as sessões)
    onAllAgentToggles: AgentToggle
        @aws_subscribe(mutations: ["notifyAgentToggle"])
        @aws_iam
}
```

---

## 📝 Fluxo Completo

```
1. Frontend Demo conecta ao AppSync
   └─> Registra subscription onAgentToggle ou onAllAgentToggles

2. Manager chama PATCH /manager/chats/{chat_id}/agent-active
   └─> Lambda agent_chat_toggle executa
       └─> Service UpdateAgentActive atualiza chat
           └─> AppSync Adapter envia mutation notifyAgentToggle
               └─> AppSync propaga evento para subscribers
                   └─> Frontend Demo recebe evento em tempo real
                       └─> Exibe mensagem no chat
```

---

## ✅ Checklist de Validação

- [ ] Deploy executado com sucesso
- [ ] Demo AppSync conectado (visitante ou manager)
- [ ] Subscriptions registradas (ver log do chat)
- [ ] Endpoint PATCH retorna 200
- [ ] Evento recebido no frontend
- [ ] Mensagem exibida corretamente no chat

---

## 🎉 Pronto!

Agora você tem um sistema completo de notificação em tempo real para quando agentes humanos assumem ou liberam conversas!

**Benefícios:**
- 🔴 **Visitante** é notificado imediatamente quando humano assume
- 🤖 **Visitante** é notificado quando IA volta a responder
- 👀 **Managers** podem monitorar todas as trocas de agentes em tempo real

