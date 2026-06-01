---
name: hermes-env-vars
description: Variáveis de ambiente do Hermes Agent — WhatsApp, E-mail, Google OAuth
category: devops
---

# Hermes Agent — Variáveis de Ambiente

Guia de referência rápida para encontrar e configurar variáveis de ambiente no Portainer.

---

## Como identificar variáveis disponíveis no container

```bash
# Lista todas variáveis (filtrado por categoria)
python3 -c "import os; [print(k) for k,v in os.environ.items()]"

# Filtrar por plataforma
python3 -c "import os; [print(k) for k,v in os.environ.items() if 'WHATSAPP' in k.upper()]"
python3 -c "import os; [print(k) for k,v in os.environ.items() if 'EMAIL' in k.upper() or 'IMAP' in k.upper() or 'SMTP' in k.upper()]"
python3 -c "import os; [print(k) for k,v in os.environ.items() if 'GOOGLE' in k.upper()]"
```

---

## WhatsApp

**Localização:** `/opt/data/.hermes/platforms/whatsapp/bridge.log`

**Variáveis esperadas (Portainer):**
- `WHATSAPP_ENABLED=true`
- `WHATSAPP_OWNER_NUMBER=5586981612061`
- `WHATSAPP_MODE=mixed`
- `WHATSAPP_BRIDGE_PORT` (opcional)

**Verificação:**
```bash
python3 -c "import os; print([k for k in os.environ.keys() if 'WHATSAPP' in k.upper()])"
```

---

## E-mail (Gmail API / Google Workspace)

**ATENÇÃO:** O sistema de email NÃO usa IMAP/SMTP — usa a **Google Gmail API** via OAuth2, através do script `support_agent.py`.

**Arquitetura:**
- Script: `~/.hermes/scripts/support_agent.py`
- API: Google Gmail API (Google Workspace)
- Autenticação: OAuth2 com web client credentials
-调度: cron job ( watchdog pattern )

**O que foi confirmado:**
- Variáveis `EMAIL_*` NÃO existem no ambiente container (não é IMAP/SMTP)
- O sistema de email usa `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` (mesmas variáveis do Gemini)
- Não é o adapter `email.py` do Hermes — é um sistema separado

**Variáveis esperadas (Portainer):**
- `GOOGLE_CLIENT_ID` — ID do cliente OAuth (mesma variável para Gmail API e Gemini)
- `GOOGLE_CLIENT_SECRET` — Secret do cliente OAuth (mesma variável para Gmail API e Gemini)

**Observação:** O `support_agent.py` foi diseñado para usar OAuth2 web client credentials. a mesma `GOOGLE_CLIENT_ID` e `GOOGLE_CLIENT_SECRET` que alimentam o Gemini STT também alimentam o sistema de email.

**Verificação:**
```bash
python3 -c "import os; print([k for k in os.environ.keys() if any(x in k for x in ['GOOGLE','CLIENT'])])"
```

---

## Google OAuth ( Gemini / Google AI )

**Essas variáveis são compartilhadas com o sistema de email Gmail API.**

**Variáveis esperadas (Portainer):**
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`

**Usado por:**
- **Gmail API** — sistema de auto-resposta de suporte (support_agent.py)
- **Gemini** — transcrição de áudio/STT e vision

**Status:** ⚠️ Configurado, mas billing do Gemini pode estar bloqueado (429 RESOURCE_EXHAUSTED).

---

## Resumo

| Plataforma | Variáveis | Método | Status |
|------------|-----------|--------|--------|
| WhatsApp | `WHATSAPP_*` | Baileys | ✅ Configurado |
| E-mail | `GOOGLE_CLIENT_*` | Gmail API (OAuth2) via support_agent.py | ✅ Configurado |
| Gemini STT/Vision | `GOOGLE_CLIENT_*` | Google OAuth API | ⚠️ Configurado, billing bloqueado |

---

## Verificação rápida após restart

```bash
# WhatsApp
ps aux | grep bridge | grep -v grep

# Todas plataformas ativas
python3 -c "import os; print(sorted([k for k in os.environ.keys() if any(x in k for x in ['WHATSAPP','EMAIL','GOOGLE','IMAP','SMTP'])]))"
```
