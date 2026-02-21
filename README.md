# 🌤️ Telegram Weather Chatbot — N8N

Chatbot no Telegram que informa a **temperatura atual** de qualquer cidade do Brasil.  
Construído com **N8N**, **OpenWeather API** e **LLM via OpenRouter** para processamento de linguagem natural.

---

## 📐 Arquitetura do Workflow

O workflow possui **12 nós** organizados em 3 fases:

```
┌─────────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   ENTRADA (NLP)     │    │   CONSULTA API   │    │   SAÍDA (NLP)       │
│                     │    │                  │    │                     │
│ Telegram Trigger    │───▸│ Consulta         │───▸│ LLM Resposta        │
│ Captura Mensagem    │    │ OpenWeather      │    │ Sucesso / Erro      │
│ LLM Extrai Cidade   │    │ Resposta Válida? │    │ Formata Resposta    │
│ Processa Resposta   │    │                  │    │ Envia Telegram      │
└─────────────────────┘    └──────────────────┘    └─────────────────────┘
```

### Fluxo detalhado

| # | Nó | Tipo | Descrição |
|---|-----|------|-----------|
| 1 | **Telegram Trigger** | Trigger | Recebe mensagens de texto do Telegram |
| 2 | **Captura Mensagem** | Set | Extrai `originalText` e `chatId` da mensagem |
| 3 | **LLM Extrai Cidade** | HTTP Request | Chama OpenRouter (Qwen) para extrair cidade/estado do texto natural |
| 4 | **Processa Resposta LLM** | Code | Parseia resposta do LLM; fallback determinístico se LLM falhar |
| 5 | **Consulta OpenWeather** | HTTP Request | Consulta API de clima com a cidade extraída |
| 6 | **Resposta Válida?** | IF | Verifica se `cod == 200` (sucesso) |
| 7 | **LLM Resposta Sucesso** | HTTP Request | LLM gera resposta educada com dados de clima |
| 8 | **Formata Sucesso** | Code | Parseia LLM; fallback: `🌤️ A temperatura em X é de Y°C.` |
| 9 | **Envia Temperatura** | Telegram | Envia mensagem de sucesso ao usuário |
| 10 | **LLM Resposta Erro** | HTTP Request | LLM gera mensagem de erro educada |
| 11 | **Formata Erro** | Code | Parseia LLM; fallback: `❌ Cidade não encontrada...` |
| 12 | **Envia Erro** | Telegram | Envia mensagem de erro ao usuário |

### LLM e Fallback

- **Modelo primário:** `qwen/qwen3-vl-235b-a22b-thinking`
- **Modelo fallback:** `qwen/qwen3-next-80b-a3b-instruct:free`
- **Rota OpenRouter:** `fallback` (tenta primário, cai pro segundo se falhar)
- **Fallback determinístico:** cada nó Code tem lógica de fallback caso ambos os modelos falhem — o bot **nunca** para de responder

---

## 🏗️ Infraestrutura (Docker Compose)

O projeto sobe **4 containers**:

| Serviço | Imagem | Função |
|---------|--------|--------|
| **n8n** | `docker.n8n.io/n8nio/n8n:latest` | Motor do workflow, porta `5678` |
| **postgres** | `postgres:16-alpine` | Banco de dados do N8N (persistência de workflows e execuções) |
| **redis** | `redis:7-alpine` | Fila de execução (modo queue) |
| **tunnel** | `cloudflare/cloudflared:latest` | Túnel HTTPS gratuito para webhooks do Telegram |

### Por que Cloudflare Tunnel?

O Telegram exige HTTPS para webhooks. O túnel Cloudflare gera uma URL pública HTTPS gratuita que redireciona para o N8N local — sem precisar de domínio, certificado SSL ou porta aberta no roteador.

---

## 🔑 Obtendo as Credenciais

### 1. Bot do Telegram (TELEGRAM_BOT_TOKEN)

1. Abra o Telegram e procure **@BotFather**
2. Envie `/newbot` e siga as instruções
3. Escolha um nome e username (deve terminar em `bot`)
4. **Copie o token** fornecido e guarde em local seguro

### 2. API Key do OpenWeather (OPENWEATHER_API_KEY)

1. Crie conta em [openweathermap.org/users/sign_up](https://home.openweathermap.org/users/sign_up)
2. Confirme o email
3. Acesse [API Keys](https://home.openweathermap.org/api_keys) e copie sua chave
4. ⚠️ **A chave demora até 2 horas para ativar** após a criação

### 3. API Key do OpenRouter (OPENROUTER_API_KEY)

1. Crie conta em [openrouter.ai](https://openrouter.ai/)
2. Acesse [Settings → Keys](https://openrouter.ai/settings/keys) e crie uma nova chave
3. Copie a chave (começa com `sk-or-v1-...`)
4. Para usar modelos gratuitos, não é necessário adicionar créditos

---

## 🚀 Instalação e Execução

### Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) e [Docker Compose](https://docs.docker.com/compose/install/) instalados

### Passo a passo

**1. Clone o repositório**

```bash
git clone https://github.com/SEU_USUARIO/chatbot-telegram.git
cd chatbot-telegram
```

**2. Configure as variáveis de ambiente**

```bash
cp .env.example .env
```

Edite o arquivo `.env` com suas credenciais reais:

```env
OPENWEATHER_API_KEY=sua_chave_openweather
OPENROUTER_API_KEY=sk-or-v1-sua_chave_openrouter
```

**3. Suba os containers**

```bash
docker compose up -d
```

**4. Obtenha a URL do túnel Cloudflare**

```bash
docker logs n8n-tunnel 2>&1 | findstr "trycloudflare.com"
```

Copie a URL HTTPS gerada (ex: `https://xxx-xxx.trycloudflare.com`).

**5. Configure a WEBHOOK_URL no `.env`**

```env
WEBHOOK_URL=https://xxx-xxx.trycloudflare.com/
```

Recrie o N8N para aplicar:

```bash
docker compose up -d --force-recreate --no-deps n8n
```

**6. Importe o workflow no N8N**

1. Acesse `http://localhost:5678` no navegador
2. Crie uma conta local (primeira vez)
3. Clique em **Add workflow** → **⋯** → **Import from File**
4. Selecione `workflow-chatbot-telegram.json`

**7. Configure o `TELEGRAM_BOT_TOKEN` no N8N**

O token do bot do Telegram (`TELEGRAM_BOT_TOKEN`) é configurado diretamente na interface do N8N como credencial:

1. No workflow importado, clique no nó **Telegram Trigger**
2. Em **Credential to connect with**, clique em **Create New Credential**
3. No campo **Access Token**, cole o seu `TELEGRAM_BOT_TOKEN` (o token obtido do BotFather, no formato `123456:ABC-DEF...`)
4. Clique em **Save**
5. Repita para os nós **Envia Temperatura** e **Envia Erro** (selecione a mesma credencial já criada)

> ⚠️ **Importante:** O `TELEGRAM_BOT_TOKEN` **não** é uma variável de ambiente no `.env` — ele é armazenado de forma segura e criptografada pelo N8N (usando a `N8N_ENCRYPTION_KEY`). Nunca suba este token no repositório.

**8. Ative o workflow**

Clique no botão **Active** no canto superior direito do workflow.

---

## 🧪 Testando o Chatbot

Abra o Telegram e envie mensagens para seu bot:

| Mensagem | Resposta esperada |
|----------|-------------------|
| `São Paulo` | 🌤️ Temperatura atual de São Paulo |
| `Bom dia, qual o clima em BH?` | 🌤️ Temperatura de Belo Horizonte (LLM entende apelidos) |
| `Colatina ES` | 🌤️ Temperatura de Colatina,ES |
| `sampa` | 🌤️ Temperatura de São Paulo (LLM converte gíria) |
| `xyzabc123` | ❌ Mensagem educada de erro |

### Verificando logs

```bash
docker logs n8n --tail 50
```

---

## 📁 Estrutura do Projeto

```
Telegram Weather Chatbot/
├── workflow-chatbot-telegram.json   # Workflow N8N exportado (sem credenciais)
├── docker-compose.yml               # Infraestrutura: N8N + PostgreSQL + Redis + Tunnel
├── .env.example                     # Variáveis de ambiente (template)
├── .env                             # Variáveis de ambiente (NÃO versionar)
├── .gitignore                       # Ignora .env
└── README.md                        # Este arquivo
```

---

## ⚠️ Notas Importantes

- **Nunca suba o `.env` com credenciais reais no repositório** — ele já está no `.gitignore`
- O arquivo `workflow-chatbot-telegram.json` **não contém tokens ou chaves** — as credenciais são referenciadas via `$env.OPENWEATHER_API_KEY`, `$env.OPENROUTER_API_KEY`, e credencial do Telegram configurada no N8N
- A cada reinício do contêiner `tunnel`, uma **nova URL Cloudflare** é gerada — atualize `WEBHOOK_URL` no `.env` e recrie o N8N
- O modelo LLM é **opcional** — se as chaves do OpenRouter estiverem ausentes ou o modelo falhar, o fallback determinístico responde normalmente

---

## 📋 Variáveis de Ambiente

| Variável | Obrigatória | Onde configurar | Descrição |
|----------|:-----------:|:---------------:|-----------|
| `TELEGRAM_BOT_TOKEN` | ✅ | UI do N8N | Token do bot obtido via @BotFather no Telegram |
| `OPENWEATHER_API_KEY` | ✅ | `.env` | Chave da API OpenWeather |
| `OPENROUTER_API_KEY` | ⭐ | `.env` | Chave da API OpenRouter (para LLM) |
| `WEBHOOK_URL` | ✅ | `.env` | URL HTTPS do túnel Cloudflare |
| `N8N_ENCRYPTION_KEY` | ✅ | `.env` | Chave para criptografar credenciais do N8N |
| `POSTGRES_USER` | ❌ | `.env` | Usuário do PostgreSQL (padrão: `n8n`) |
| `POSTGRES_PASSWORD` | ❌ | `.env` | Senha do PostgreSQL (padrão: `n8n_password`) |
| `POSTGRES_DB` | ❌ | `.env` | Nome do banco (padrão: `n8n`) |

> ⭐ A `OPENROUTER_API_KEY` é necessária para a funcionalidade LLM. Sem ela, o bot funciona com respostas determinísticas (fallback).
