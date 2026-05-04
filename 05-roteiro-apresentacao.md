# Apresentação: Classificação OWASP Mobile Top 10 - FinanceTrack

## Slide Único - Roteiro de Apresentação

---

## 📊 FinanceTrack: 5 Vulnerabilidades OWASP Mobile

### Introdução (30 segundos)
"O FinanceTrack é um app de finanças que sofreu 5 vulnerabilidades críticas. Vou mostrar como cada uma se classifica no OWASP Mobile Top 10 2024."

---

### 🔴 **Problema 1: Chave Secreta Exposta no Repositório**
- **O quê**: `client_secret` estava em texto puro no código-fonte, em repositório público
- **OWASP**: **M8 — Security Misconfiguration** (Má Configuração)
- **Por quê**: Erro de configuração arquitetural (segredo deveria estar apenas no servidor)
- **Impacto**: Hacker consegue fazer login falsificado como qualquer usuário

---

### 🔴 **Problema 2: Token de Acesso Sem Proteção**
- **O quê**: JWT guardado em SharedPreferences/NSUserDefaults (texto puro, sem criptografia)
- **OWASP**: **M9 — Insecure Data Storage** (Armazenamento Inseguro)
- **Por quê**: Dados sensíveis guardados de forma desprotegida no celular
- **Impacto**: Usuário com celular roubado = hacker acessa conta em 2 minutos

---

### 🟠 **Problema 3: Sem Tratamento de Falha de Rede**
- **O quê**: Quando servidor retorna erro (503), app não tenta novamente automaticamente
- **OWASP**: **M3 — Insecure Communication** (Comunicação Insegura)
- **Por quê**: Comunicação não confiável = usuário clica "tentar novamente" manualmente
- **Impacto**: Abre porta para Problema 4 (transações duplicadas)

---

### 🟠 **Problema 4: Transação Duplicada (Sem Idempotência)**
- **O quê**: Mesma transferência processada 3 vezes (R$ 150 virou R$ 450)
- **OWASP**: **M3 — Insecure Communication** (Comunicação Insegura)
- **Por quê**: Requisições não têm identificador único (sem Idempotency-Key)
- **Impacto**: Prejuízo financeiro direto (débito múltiplo)

---

### 🟡 **Problema 5: Token de Longa Duração (30 Dias)**
- **O quê**: Access token válido por 30 dias (RFC 9700 recomenda 5-15 minutos)
- **OWASP**: **M4 — Insufficient Authentication** (Autenticação Insuficiente)
- **Por quê**: Validade inadequada = se token for roubado, hacker tem 30 dias de acesso
- **Impacto**: Janela de exploração massivamente grande

---

## 📋 Tabela Resumida

| # | Vulnerabilidade | OWASP | Categoria | Severidade |
|---|---|---|---|---|
| 1 | Segredo em repositório | **M8** | Config | 🔴 CRÍTICA |
| 2 | Token sem proteção | **M9** | Storage | 🔴 CRÍTICA |
| 3 | Sem retry | **M3** | Comunicação | 🟠 MODERADA |
| 4 | Transação duplicada | **M3** | Comunicação | 🟠 ALTA |
| 5 | Token 30 dias | **M4** | Autenticação | 🟡 ALTA |

---

## 🔗 Relação com OWASP Mobile Top 10

**Por que essas vulnerabilidades?**

- **M8 (Config)**: Decisões erradas na arquitetura → Segredo exposto
- **M9 (Storage)**: Armazenamento inseguro → Token vulnerável
- **M3 (Comunicação)**: Falta retry e idempotência → Falhas cascata
- **M4 (Autenticação)**: Tokens muito válidos → Janela de exploração

**Conexão entre elas:**
```
M8 (Segredo) → Hacker consegue acesso
      ↓
M9 (Storage) → Hacker rouba token
      ↓
M4 (Auth longa) → Hacker tem 30 dias
      ↓
M3 (Sem idempotência) → Usuário duplica transações
```

---

## 💡 Solução Simples

**Implementar:**
1. OAuth 2.0 + PKCE (resolve M8)
2. Android Keystore + iOS Keychain (resolve M9)
3. Retry automático (resolve M3, parte 1)
4. Idempotency-Key (resolve M3, parte 2)
5. Access token 15 min + Refresh token (resolve M4)

**Resultado**: App seguro ✅

---

## ⏱️ Dicas de Apresentação

- **Tempo total**: ~5-7 minutos
- **Foco**: Explicar POR QUÊ cada problema é OWASP, não como consertar
- **Tom**: Técnico mas acessível (evitar muito jargão)
- **Finalizar**: "Todas essas vulnerabilidades são preveníveis com boas práticas de segurança"

---

**Boa apresentação! 🚀**
