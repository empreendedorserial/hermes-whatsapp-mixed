# 🤖 Hermes Agent - WhatsApp Plugin

Plugin **`whatsapp-manager`** para o Hermes Agent. Gerencia o comportamento do WhatsApp com assistente pessoal para o dono, atendimento a clientes, classificação inteligente de contatos e histórico cross-session.

---

## 📂 Estrutura do Repositório

```text
/
├── plugin.yaml              # Manifesto do plugin
├── __init__.py              # Entry point
├── whatsapp_manager.py      # Lógica principal do plugin
├── bridge.js                # Ponte WhatsApp (Baileys)
├── package.json             # Dependências da ponte
├── deploy/                  # Docker, configs, scripts
│   ├── docker-compose.yml
│   ├── docker-compose.easypanel.yml
│   ├── setup.sh
│   └── README.md
└── tests/                   # Suite de testes
```

---

## ⚡ Instalação

1. Acesse o **Hermes Dashboard → Plugins**
2. Cole a URL do repositório e clique em **Install/Enable**

---

## 🕹️ Comandos no WhatsApp (Self-Chat)

| Comando | Ação |
|---|---|
| `stop_bot` / `!pausar` | Pausa o bot para todos os clientes |
| `start_bot` / `!retomar` | Reativa o bot |
| `sync contacts` / `sincronize os contatos` | Sincroniza contatos em background (não bloqueia o bot) |
| `update contact <nome> campo=valor` | Atualiza campo específico de um contato |

Silenciamento automático de 10 min quando o dono lê ou responde manualmente um chat de cliente.

---

## ✨ Funcionalidades

### Assistente Pessoal (Self-Chat)

O dono conversa com o bot no próprio número. O bot age como assistente pessoal com acesso a histórico completo.

**Cross-session history:** Quando o dono pergunta sobre uma conversa com outra pessoa ("o que a Isabel falou?"), o bot busca o histórico real no `whatsapp_messages.db` e `state.db` e injeta no contexto antes da chamada ao LLM.

### Classificação de Contatos

Cada contato é classificado automaticamente com perfil, tom, intenção e resumo usando Gemini/OpenAI/OpenRouter. Os dados são salvos em `/opt/data/personal_contacts.json` e sincronizados com o GitHub.

**Campos por contato:**
```json
{
  "name": "Nome completo",
  "relationship": "Cliente|Amigo|AmigoProximo|Parente|Filho|Vendedor",
  "manual_relationship": "definido pelo dono, nunca sobrescrito pelo sync",
  "nickname": "apelido da pessoa",
  "pet_name": "nome do animal de estimação",
  "tone": "tom de atendimento",
  "guidelines": "instruções específicas",
  "notes": "notas livres",
  "summary": "resumo comprimido (1-2 frases) para uso no contexto",
  "full_summary": "histórico cumulativo por período (Jun/25: ..., Jul/25: ...)",
  "last_summarized_at": "timestamp da última sessão processada",
  "last_interaction": "timestamp da última mensagem"
}
```

**Busca de nome:** 6 níveis em cascata:
1. Número/JID exato
2. `name` exato no JSON
3. `nickname` exato no JSON
4. Substring em `name`
5. `sender_name` no `whatsapp_messages.db`
6. `/contacts/search?name=X` no bridge (store do Baileys)

Se não encontrado, o bot pergunta o número ao dono e aplica a atualização quando fornecido.

### Atualização de Contatos em Linguagem Natural

O dono pode dizer: *"atualize as informações da Isabel Alencar, ela é minha filha e o apelido é Bebel"*

O sistema:
1. Extrai o nome via LLM (`_extract_contact_name_via_llm`)
2. Localiza o contato nos 6 níveis de busca
3. Extrai os campos a atualizar via LLM (apenas campos explicitamente mencionados — não sobrescreve `tone`, `summary`, `guidelines`)
4. Garante `manual_relationship = relationship` quando definido pelo dono
5. Salva e sincroniza com GitHub

### Resumo Cumulativo de Histórico (`full_summary`)

A cada sync, sessões novas no `state.db` são processadas incrementalmente:
- Apenas mensagens do **contato** (`role=user`) são incluídas
- O LLM atualiza o `full_summary` adicionando o período novo ao final
- Quando `full_summary` > 600 chars, é comprimido em `summary` de 1-2 frases
- `last_summarized_at` garante que cada sessão é processada uma única vez

### Sync de Contatos (Não-Bloqueante)

```
WHATSAPP_SYNC_INTERVAL_HOURS=24  # padrão, configurável via env
```

- **Não roda no boot** — bot disponível imediatamente após deploy
- Sync via chat roda em thread daemon e notifica o dono quando terminar
- `_sync_running` impede execuções simultâneas
- Sync periódico automático no intervalo configurado

### Transcrição e Descrição de Mídia

- Áudio/voz → transcrição via Gemini
- Imagens → descrição automática
- Histórico atualizado com `[Áudio: "..."]` e `[Imagem: ...]`
- Arquivos deletados do disco imediatamente após processamento

### Resolução de Nome via Bridge

Para contatos sem `sender_name` no DB, consulta:
- `GET /contact/:jid` — nome de um JID específico
- `GET /contacts/search?name=X` — busca no store do Baileys por nome

---

## 🔧 Variáveis de Ambiente

| Variável | Descrição |
|---|---|
| `WHATSAPP_OWNER_NUMBER` | JID do dono (ex: `5511999998888@s.whatsapp.net`) |
| `WHATSAPP_SYNC_INTERVAL_HOURS` | Intervalo do sync automático (padrão: `24`) |
| `WHATSAPP_SILENCE_DURATION_MIN` | Minutos de silêncio após ação manual (padrão: `10`) |
| `WHATSAPP_SYNC_MAX_CLASSIFICATIONS` | Máx de chamadas LLM por sync (padrão: `10`) |
| `WHATSAPP_SYNC_MIN_MESSAGES` | Mín de mensagens para classificar (padrão: `3`) |

---

## 🗄️ Bancos de Dados

| Arquivo | Conteúdo |
|---|---|
| `/opt/data/personal_contacts.json` | Perfis e resumos dos contatos |
| `/opt/data/.hermes/whatsapp_messages.db` | Histórico raw de mensagens (bridge) |
| `/opt/data/.hermes/state.db` | Sessões e mensagens do Hermes |

---

## 🚀 Deploy

```bash
git add -A && git commit -m "..." && git push origin main
```

Depois: Hermes Dashboard → Plugins → Pull → Portainer → Restart container.

---

## 🧪 Testes

```bash
npm install && npm test                              # testes da bridge (JS)
GOOGLE_API_KEY=xxx python3 tests/test_gemini_classification.py  # teste Gemini
python3 tests/plugin_test.py                        # testes do plugin (Python)
```

---

## 📋 Hooks do Plugin

| Hook | Suporte | Função |
|---|---|---|
| `pre_gateway_dispatch` | ✅ | Intercepta mensagens antes do LLM — detecta comandos, sync, NL updates, cross-session |
| `pre_llm_call` | ✅ | Injeta contexto do owner, histórico cross-session |
| `post_llm_call` | ❌ não chamado pelo Hermes | — |

---

*Desenvolvido e mantido por André Alencar / Empreendedor Serial.*
