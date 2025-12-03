# 📊 Agente de Cotação de Dólar - Documentação Completa

## 📋 Índice
- [Visão Geral](#visão-geral)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Arquitetura do Sistema](#arquitetura-do-sistema)
- [Fluxos de Funcionamento](#fluxos-de-funcionamento)
- [Configuração e Instalação](#configuração-e-instalação)
- [Estrutura do Banco de Dados](#estrutura-do-banco-de-dados)
- [Como Usar](#como-usar)
- [Detalhes Técnicos](#detalhes-técnicos)

---

## 🎯 Visão Geral

Agente automatizado de cotação de dólar integrado com WhatsApp que permite aos clientes:
- Solicitar cotações em tempo real
- Fechar negócios com valores dinâmicos
- Sistema de expiração de 10 segundos para garantir preços justos
- Validação automática de tempo e valores
- Registro automático em Google Sheets

### Principais Funcionalidades
✅ Cotação em tempo real via API
✅ Sistema de expiração de 10 segundos
✅ Valores dinâmicos (2k, 100k, 50000, etc)
✅ Validação de tempo automática
✅ Registro no Google Sheets
✅ Gerenciamento de leads no Supabase
✅ Conversação com IA para dúvidas gerais

---

## 🛠️ Tecnologias Utilizadas

### **n8n** - Orquestrador de Workflows
- **Versão**: Requer n8n v1.x ou superior
- **Função**: Automação visual de todo o fluxo de negócio
- **Nós utilizados**:
  - Webhook (receber mensagens WhatsApp)
  - HTTP Request (APIs externas)
  - Code (JavaScript para lógica customizada)
  - Supabase (operações de banco de dados)
  - Google Sheets (registro de vendas)
  - LangChain Agent (IA conversacional)
  - Date & Time (cálculos de tempo)
  - Switch/Filter (roteamento de fluxo)

### **Supabase** - Backend as a Service
- **Função**: Banco de dados PostgreSQL hospedado + Gerenciamento de timer
- **Tabelas criadas**:
  - `Leads`: Cadastro de clientes
  - `n8n_cotacao`: **Controle de cotações com timestamp (timer de 10 segundos)**
- **Recursos usados**:
  - Tabelas relacionais
  - Queries com filtros
  - Operações CRUD
  - **Armazenamento de timestamp para validação de expiração**

### **PostgreSQL** (Externo - para IA)
- **Função**: Apenas para memória conversacional da IA
- **Tabela**: `n8n_chat_histories` (gerenciada automaticamente pelo LangChain)
- **Uso específico**: Postgres Chat Memory (histórico de conversas por telefone)
- **Nota**: NÃO usado para timer ou cotações

### **API Coopfy**
- **Endpoint**: `https://coopfy.com/api/usdt/price?spread=0.00498`
- **Função**: Cotação em tempo real do dólar
- **Response**: `{ data: { dollarOTC: 6.15 } }`

### **WhatsApp (via UAZApi)**
- **Gateway**: UAZApi
- **Endpoint**: `https://andradek.uazapi.com/send/text`
- **Autenticação**: Token via header
- **Função**: Enviar/receber mensagens

### **Google Sheets API**
- **Função**: Registro de vendas fechadas
- **Planilha**: "Dolar Comprado"
- **Colunas**: Data, Preço Dolar, Valor Compra, Telefones

### **Groq LLM**
- **Modelo**: `openai/gpt-oss-120b`
- **Função**: IA conversacional para perguntas gerais
- **Framework**: LangChain
- **Memory**: Postgres Chat Memory (histórico por telefone)

---

## 🏗️ Arquitetura do Sistema

### Diagrama de Fluxo Principal

```
┌─────────────────────────────────────────────────────────────────┐
│                         WEBHOOK (WhatsApp)                       │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              FILTER (Verifica telefones autorizados)            │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│        VARIAVEIS (Extrai dados: telefone, mensagem, etc)        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│         GET MANY ROWS (Busca lead no Supabase: Leads)          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│    SWITCH4 (Primeiro contato ou já cadastrado?)                 │
├─────────────────────┬───────────────────────────────────────────┤
│  Primeiro Contato   │           Já Cadastrado                   │
└──────────┬──────────┴──────────┬────────────────────────────────┘
           │                     │
           ▼                     ▼
    ┌─────────────┐      ┌──────────────┐
    │ CREATE ROW  │      │   SWITCH5    │
    │  (Supabase) │      │  (Análise de │
    └──────┬──────┘      │   Mensagem)  │
           │             └───┬──────┬───┬┘
           │                 │      │   │
           └─────────────────┘      │   │
                                   │   │
        ┌──────────────────────────┘   │
        │                              │
        ▼                              ▼
┌───────────────┐              ┌──────────────┐
│   "cotacao"   │              │  "fechar"    │
│               │              │              │
│ BuscarCotacao3│              │BuscarCotacao4│
│      ↓        │              │      ↓       │
│ Get many rows3│              │  Date & Time │
│      ↓        │              │      ↓       │
│   SWITCH6     │              │   SWITCH7    │
│      ↓        │              │   ↙     ↘   │
│ Create/Update │              │Expirado  OK  │
│      ↓        │              │   ↓      ↓   │
│ SalvarCotação │              │ Erro  Extrai │
│      ↓        │              │       Valor  │
│EnviarCotacao  │              │         ↓    │
└───────────────┘              │    BuscarAPI │
                               │         ↓    │
                               │   Append     │
                               │   Sheets     │
                               │         ↓    │
                               │   Confirma   │
                               └──────────────┘
```

---

## 🔄 Fluxos de Funcionamento

### 1️⃣ **Fluxo: Cliente Solicita Cotação**

**Trigger**: Cliente envia "cotacao" ou "cotação"

```
1. Webhook → Recebe mensagem do WhatsApp
2. Filter1 → Verifica se telefone está autorizado
3. Variaveis → Extrai telefone, mensagem, timestamp
4. Get many rows2 → Busca lead na tabela "Leads"
5. Switch4 → Verifica se é primeiro contato
   - Primeiro: Create a row2 (cadastra novo lead)
   - Já existe: Segue direto
6. Switch5 → Identifica palavra "cotacao"
7. BuscarCotacao3 → Chama API Coopfy
8. Get many rows3 → Busca cotação anterior no Supabase
9. Switch6 → Verifica se já solicitou antes
   - Primeira vez: Create a row3 (insere registro)
   - Já solicitou: SalvarCotação3 (atualiza timestamp)
10. SalvarCotação2 → Atualiza horário da solicitação
11. EnviarCotacao → Envia WhatsApp com:
```

**Mensagem enviada:**
```
💵 *Cotação do Dólar*

Valor: *R$ 6.15*

⏰ Válida por 10 segundos

Para fechar, responda:
*fechar [valor]*

Exemplo: fechar 100k
```

---

### 2️⃣ **Fluxo: Cliente Fecha Negócio**

**Trigger**: Cliente envia "fechar 2k" (ou qualquer valor)

```
1. Webhook → Recebe mensagem
2. Filter1 → Valida telefone
3. Variaveis → Extrai dados
4. Switch5 → Identifica palavra "fechar"
5. BuscarCotacao4 → Busca timestamp da solicitação no Supabase
6. Date & Time → Calcula diferença em segundos
7. Switch7 → Verifica se passou de 10 segundos

   ❌ SE EXPIRADO (> 10s):
   → EnviarErro → "⏱️ Essa cotação já expirou!"

   ✅ SE VÁLIDO (<= 10s):
   → ExtrairValor → Code Node que:
      • Extrai "2k" da mensagem
      • Converte para 2000
      • Suporta: k, m, números diretos
   → BuscarCotacao5 → Busca cotação atual da API
   → Append row in sheet1 → Salva no Google Sheets:
      • Data: 2025-12-03 14:30
      • Preço Dolar: 6.15
      • Valor Compra: 2000
      • Telefones
   → EnviaMensagem → Confirma com:
```

**Mensagem de confirmação:**
```
✅ *Compra Fechada!*

Valor: *2k* (R$ 2.000)
Cotação: *R$ 6.15*
Total em dólares: *$325.20*

🎉 Obrigado pela preferência!
```

---

### 3️⃣ **Fluxo: Conversa Normal (IA)**

**Trigger**: Cliente faz pergunta que não é "cotacao" nem "fechar"

```
1. Switch5 → "Conversa Normal"
2. Agente Cotação → LangChain Agent com:
   - Model: Groq (openai/gpt-oss-120b)
   - Memory: Postgres Chat Memory (por telefone)
   - Tool: HTTP Request (pode consultar API)
   - Prompt: Respostas concisas (2-3 linhas)
3. EnviaMensagem3 → Envia resposta da IA
```

**Exemplos de perguntas:**
- "Qual o horário de atendimento?"
- "Aceitam PIX?"
- "Como funciona a compra?"

---

## ⚙️ Configuração e Instalação

### Pré-requisitos
- [x] n8n instalado (self-hosted ou cloud)
- [x] Conta Supabase (free tier funciona)
- [x] Conta Google Cloud (para Sheets API)
- [x] Conta Groq (para LLM)
- [x] UAZApi ou similar para WhatsApp

---

### Passo 1: Configurar Supabase

#### 1.1 Criar tabela `Leads`
```sql
CREATE TABLE IF NOT EXISTS "Leads" (
    "id" BIGSERIAL PRIMARY KEY,
    "Nome" VARCHAR(255),
    "Telefone" VARCHAR(20) UNIQUE NOT NULL,
    "created_at" TIMESTAMPTZ DEFAULT NOW()
);
```

#### 1.2 Criar tabela `n8n_cotacao` (TIMER DE EXPIRAÇÃO)
```sql
CREATE TABLE IF NOT EXISTS "n8n_cotacao" (
    "id" BIGSERIAL PRIMARY KEY,
    "Telefone" VARCHAR(20) UNIQUE NOT NULL,
    "horarioSolicitacao" TIMESTAMPTZ NOT NULL, -- Momento que cliente pediu cotação
    "created_at" TIMESTAMPTZ DEFAULT NOW(),
    "updated_at" TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cotacao_telefone ON n8n_cotacao("Telefone");

-- Esta tabela é usada APENAS para controlar o timer de 10 segundos
-- O n8n busca o horarioSolicitacao, calcula a diferença com $now
-- Se passou mais de 10 segundos, cotação expirou
```

#### 1.3 Obter credenciais Supabase
1. Acesse seu projeto no Supabase
2. Vá em **Settings > API**
3. Copie:
   - Project URL
   - Service Role Key (anon public)

---

### Passo 2: Configurar n8n

#### 2.1 Importar workflow
1. Abra n8n
2. Clique em **Import from File**
3. Selecione `AgenteCotacaoDolar (1).json`

#### 2.2 Configurar credenciais

**Supabase:**
- Nome: `Supabase account`
- URL: `https://seu-projeto.supabase.co`
- API Key: `sua-service-role-key`

**Google Sheets:**
- Nome: `Google Sheets account`
- Autenticação: OAuth2
- Seguir wizard de autorização

**Groq:**
- Nome: `Groq account`
- API Key: Obter em https://console.groq.com

**PostgreSQL (APENAS para memória da IA):**
- Host: `db.seu-projeto-postgres.supabase.co` (ou servidor externo)
- Database: `postgres`
- User: `postgres`
- Password: obtido no provedor PostgreSQL
- **Uso**: Somente para `n8n_chat_histories` (LangChain Memory)
- **Nota**: Timer/cotações usam Supabase, NÃO este PostgreSQL

**UAZApi (WhatsApp):**
- Token: configurado no header dos nós HTTP Request

---

### Passo 3: Configurar Google Sheets

1. Criar planilha "Dolar Comprado"
2. Adicionar colunas:
   - Data
   - Telefone Comprador
   - Telefone Vendedor
   - Valor Compra
   - Preço Dolar
3. Copiar ID da planilha da URL
4. Atualizar no nó `Append row in sheet1`

---

### Passo 4: Configurar Webhook WhatsApp

#### 4.1 Obter URL do webhook
1. No n8n, abra o workflow
2. Clique no nó **Webhook**
3. Copie a **Production URL**
   - Exemplo: `https://seu-n8n.com/webhook/f872b772-...`

#### 4.2 Configurar no UAZApi
1. Acesse painel UAZApi
2. Vá em **Webhooks**
3. Cole a URL copiada
4. Selecione eventos: **Messages**

---

### Passo 5: Ajustar telefones autorizados

No nó **Filter1**, edite os telefones permitidos:

```javascript
// Linha 31 e 41
"rightValue": "5511958988854"  // Substitua pelo seu telefone
```

---

## 📊 Estrutura do Banco de Dados

### Tabela: `Leads`
```
┌──────────────────────────────────────┐
│ Leads                                │
├──────────────┬──────────┬────────────┤
│ Campo        │ Tipo     │ Descrição  │
├──────────────┼──────────┼────────────┤
│ id           │ BIGSERIAL│ PK auto    │
│ Nome         │ VARCHAR  │ Nome lead  │
│ Telefone     │ VARCHAR  │ Único      │
│ created_at   │ TIMESTAMP│ Data       │
└──────────────┴──────────┴────────────┘
```

### Tabela: `n8n_cotacao` (Supabase - TIMER)
```
┌─────────────────────────────────────────────────────────────┐
│ n8n_cotacao (SUPABASE)                                      │
├────────────────────┬──────────┬─────────────────────────────┤
│ Campo              │ Tipo     │ Descrição                   │
├────────────────────┼──────────┼─────────────────────────────┤
│ id                 │ BIGSERIAL│ PK auto                     │
│ Telefone           │ VARCHAR  │ Único (chave)               │
│ horarioSolicitacao │ TIMESTAMP│ Quando pediu cotação        │
│ created_at         │ TIMESTAMP│ Criação                     │
│ updated_at         │ TIMESTAMP│ Atualizado                  │
└────────────────────┴──────────┴─────────────────────────────┘

Índice: idx_cotacao_telefone (Telefone)

FUNÇÃO: Controlar timer de 10 segundos
- Salva timestamp quando cliente pede "cotacao"
- n8n busca e calcula: $now - horarioSolicitacao
- Se > 10 segundos: EXPIRADO
- Se <= 10 segundos: PODE FECHAR
```

### Tabela: `n8n_chat_histories` (PostgreSQL - MEMÓRIA IA)
```
┌─────────────────────────────────────────────────────────────┐
│ n8n_chat_histories (POSTGRES EXTERNO)                       │
├────────────────────┬──────────┬─────────────────────────────┤
│ Campo              │ Tipo     │ Descrição                   │
├────────────────────┼──────────┼─────────────────────────────┤
│ session_id         │ VARCHAR  │ Telefone do cliente         │
│ message            │ TEXT     │ Mensagem (user/assistant)   │
│ created_at         │ TIMESTAMP│ Timestamp                   │
└────────────────────┴──────────┴─────────────────────────────┘

FUNÇÃO: Histórico de conversas com IA
- Gerenciada automaticamente pelo LangChain
- Usado pelo nó "Postgres Chat Memory"
- NÃO tem relação com timer/cotações
```

### Relacionamentos
```
┌─────────────────────────────────────────────────────────────┐
│                     SEPARAÇÃO DE BANCOS                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SUPABASE (Timer + Leads)          POSTGRES (Memória IA)   │
│  ┌─────────────────┐               ┌──────────────────┐    │
│  │ Leads           │               │ n8n_chat_        │    │
│  │ - Telefone (PK) │               │   histories      │    │
│  └────────┬────────┘               │ - session_id     │    │
│           │                        │ - message        │    │
│           │ 1:1                    └──────────────────┘    │
│  ┌────────▼────────┐                                       │
│  │ n8n_cotacao     │               Usado por:              │
│  │ - Telefone (FK) │               • Postgres Chat Memory │
│  │ - horarioSolic. │◄──── TIMER    • Agente Cotação       │
│  └─────────────────┘                                       │
│                                                              │
│  Função: Validar 10 segundos                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎮 Como Usar

### Para o Cliente (WhatsApp)

#### 1. Solicitar Cotação
```
Cliente: cotacao
```
**Resposta:**
```
💵 *Cotação do Dólar*

Valor: *R$ 6.15*

⏰ Válida por 10 segundos

Para fechar, responda:
*fechar [valor]*

Exemplo: fechar 100k
```

#### 2. Fechar Negócio (dentro de 10 segundos)
```
Cliente: fechar 2k
```
**Resposta:**
```
✅ *Compra Fechada!*

Valor: *2k* (R$ 2.000)
Cotação: *R$ 6.15*
Total em dólares: *$325.20*

🎉 Obrigado pela preferência!
```

#### 3. Formatos aceitos
```
fechar 2k      → R$ 2.000
fechar 100k    → R$ 100.000
fechar 2.5k    → R$ 2.500
fechar 50000   → R$ 50.000
fechar 1m      → R$ 1.000.000
```

#### 4. Perguntas gerais
```
Cliente: Qual o horário de atendimento?
Bot: Horário de atendimento: 9h às 18h
```

---

## 🔧 Detalhes Técnicos

### Nó: ExtrairValor (Code Node)

**Função**: Extrai e converte valores da mensagem

```javascript
const variaveis = $('Variaveis').first().json;
const mensagem = variaveis.Mensagem.toLowerCase();

// Regex para extrair valor
const matchValor = mensagem.match(/fechar\s+([0-9.]+[km]?)/i);

// Conversão
if (valorTexto.endsWith('k')) {
  valorNumerico = parseFloat(valorTexto) * 1000;
} else if (valorTexto.endsWith('m')) {
  valorNumerico = parseFloat(valorTexto) * 1000000;
}

return [{
  json: {
    valorTexto: "2k",
    valorNumerico: 2000,
    ...variaveis
  }
}];
```

---

### Nó: Date & Time (Validação de Timer)

**Função**: Calcula diferença de tempo entre solicitação e fechamento

**Configuração:**
- Start Date: `{{ $json.horarioSolicitacao }}` (vem do Supabase: n8n_cotacao)
- End Date: `{{ $now }}` (momento atual do n8n)
- Units: `second`

**Fluxo:**
1. Cliente pede "cotacao" → Supabase salva timestamp em `horarioSolicitacao`
2. Cliente diz "fechar 2k" → n8n busca `horarioSolicitacao` do Supabase
3. Date & Time calcula: `$now - horarioSolicitacao`
4. Switch7 verifica se passou 10 segundos

**Output:**
```json
{
  "timeDifference": {
    "seconds": 8  // Se 8s: VÁLIDO | Se 12s: EXPIRADO
  }
}
```

**Observação:** O Supabase é usado apenas para ARMAZENAR o timestamp. O cálculo de diferença é feito pelo n8n com o nó Date & Time.

---

### Nó: Switch7 (Validação de Expiração)

**Condições:**

| Condição | Operação | Valor | Saída |
|----------|----------|-------|-------|
| `timeDifference.seconds` | `>` | 10 | Expirado |
| `timeDifference.seconds` | `<=` | 10 | Fechado |

---

### Nó: Agente Cotação (LangChain)

**System Prompt:**
```
Você é um assistente de cotação de dólar. IMPORTANTE:

1. Seja EXTREMAMENTE conciso - máximo 2-3 linhas
2. Use apenas informações diretas, sem enrolação
3. Para cotações: informe apenas o valor em formato "Dólar: R$ X.XX"
4. Para outras perguntas: responda de forma direta em no máximo 2 frases
5. NUNCA repita informações ou faça textos longos

Exemplos de respostas boas:
- "Dólar agora: R$ 6.15"
- "Sim, aceitamos pagamento via PIX"
- "Horário de atendimento: 9h às 18h"
```

**Tools disponíveis:**
- HTTP Request: Pode consultar API de cotação

**Memory:**
- Postgres Chat Memory
- Session por telefone
- Context window: 50 mensagens

---

## 📈 Métricas e Monitoramento

### Dados registrados no Google Sheets

Cada venda fechada registra:
- **Data**: Timestamp da compra
- **Preço Dolar**: Cotação no momento
- **Valor Compra**: Valor em reais (numérico)
- **Telefone Vendedor**: Quem recebeu a mensagem
- **Telefone Comprador**: Cliente

### Exemplo de registro:
```
| Data              | Preço Dolar | Valor Compra | Telefone Vendedor | Telefone Comprador |
|-------------------|-------------|--------------|-------------------|--------------------|
| 2025-12-03 14:30  | 6.15        | 2000         | 553173591394      | 5511958988854      |
| 2025-12-03 14:35  | 6.16        | 100000       | 553173591394      | 5511958988854      |
```

---

## ⚠️ Tratamento de Erros

### Erro: Cotação Expirada
**Mensagem:**
```
⏱️ Essa cotação já expirou!

Digite *cotacao* para receber o valor atualizado do dólar. 💵
```

### Erro: Formato Inválido
**Quando**: Cliente digita "fechar" sem valor

**Code Node retorna:**
```json
{
  "erro": true,
  "valorTexto": "0",
  "valorNumerico": 0
}
```

### Erro: Telefone não autorizado
**Quando**: Filter1 bloqueia telefone não listado

**Resultado**: Workflow para silenciosamente

---

## 🔒 Segurança

### Credenciais protegidas
- Todas as credenciais armazenadas no n8n (criptografadas)
- Tokens não expostos no workflow JSON exportado

### Telefones autorizados
- Lista whitelist no nó Filter1
- Apenas telefones específicos podem usar

### Validação de tempo
- Sistema de expiração previne manipulação de preço
- Timestamp salvo no banco (imutável)

---

## 🚀 Melhorias Futuras

### Sugestões de implementação:

1. **Dashboard de Analytics**
   - Total vendido no dia/semana/mês
   - Ticket médio
   - Gráfico de conversão

2. **Notificações para Vendedor**
   - Telegram/Email quando venda fecha
   - Resumo diário de vendas

3. **Múltiplas moedas**
   - Euro, Libra, Peso
   - Seleção via menu interativo

4. **Histórico de cotações**
   - Salvar todas as cotações consultadas
   - Gráfico de variação

5. **Limite de tentativas**
   - Prevenir spam
   - Cooldown entre solicitações

6. **Confirmação antes de fechar**
   - Botões interativos do WhatsApp
   - "Confirmar" ou "Cancelar"

---

## 📞 Suporte e Manutenção

### Logs do n8n
- Acessar: **Executions** no menu do n8n
- Ver: Cada execução do workflow
- Debug: Inspecionar dados entre nós

### Verificar tabelas

**Supabase (Timer + Leads):**
```sql
-- Ver todas as cotações ativas (timer)
SELECT
  "Telefone",
  "horarioSolicitacao",
  EXTRACT(EPOCH FROM (NOW() - "horarioSolicitacao")) as segundos_passados
FROM n8n_cotacao
ORDER BY "horarioSolicitacao" DESC
LIMIT 10;

-- Ver leads cadastrados
SELECT * FROM "Leads"
ORDER BY created_at DESC;
```

**PostgreSQL (Memória IA):**
```sql
-- Ver histórico de conversas
SELECT session_id, message, created_at
FROM n8n_chat_histories
WHERE session_id = '5511958988854'  -- telefone do cliente
ORDER BY created_at DESC
LIMIT 20;
```

### Testar API manualmente
```bash
curl https://coopfy.com/api/usdt/price?spread=0.00498
```

---

## 📝 Licença e Créditos

**Desenvolvido com:**
- n8n (Apache 2.0)
- Supabase (Apache 2.0)
- LangChain (MIT)
- Groq API

**Autor**: Desenvolvido para automação de vendas de câmbio

**Versão**: 2.0 (Dezembro 2025)

---

## 🆘 Troubleshooting

### Problema: Cotação não envia
**Solução:** Verificar credenciais UAZApi no nó EnviarCotacao

### Problema: Sempre diz "expirado"
**Solução:**
1. Verificar se o Supabase está salvando `horarioSolicitacao` corretamente
2. Testar query: `SELECT NOW(), horarioSolicitacao FROM n8n_cotacao`
3. Verificar timezone do Supabase (deve usar TIMESTAMPTZ)
4. Confirmar que o nó Date & Time está lendo do Supabase corretamente

### Problema: Google Sheets não registra
**Solução:** Reautorizar OAuth2 do Google

### Problema: IA não responde
**Solução:** Verificar créditos da conta Groq

---

**Pronto para uso! 🎉**
