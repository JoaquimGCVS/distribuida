# Contexto da Atividade: Análise de Vulnerabilidades do FinanceTrack

## Introdução

Este arquivo descreve o **caso real** do app FinanceTrack, a atividade que você precisa resolver, e por que essa atividade importa.

Se você não leu os dois arquivos anteriores, **não precisa** — eles servem como referência. Este arquivo é autocontido e explica tudo que você precisa saber para fazer a atividade.

---

## Parte 1: Conhecendo o FinanceTrack

### O que é?

**FinanceTrack** é um aplicativo de finanças pessoais. Funciona assim:

```
[USUÁRIO]
    ↓
Faz login no app
    ↓
App conecta ao servidor para buscar dados financeiros
    ↓
Exibe saldo, extrato, histórico de transações
    ↓
Usuário pode fazer transferências, pagamentos, saques
    ↓
[SERVIDOR]
```

### Público

Pessoas que querem gerenciar seus gastos pelo celular. Crianças, jovens, adultos, idosos.

### Tipo de Dados

- Email e senha do usuário
- Saldo bancário
- Histórico de transações (transferências, pagamentos)
- Dados pessoais (nome, CPF)

**Tudo isso é informação sensível.**

---

## Parte 2: O Pentest (Teste de Penetração)

### O que é um Pentest?

Um teste de penetração é quando uma empresa contrata especialistas em segurança para "atacar" o próprio app, tentando encontrar vulnerabilidades **antes que hackers reais encontrem**.

É como fazer um exercício de incêndio na sua casa: melhor descobrir problemas agora do que durante um fogo de verdade.

### O Que Encontraram

A empresa de pentest encontrou **5 vulnerabilidades críticas** no FinanceTrack:

```
VULNERABILIDADE 1: Chave secreta exposta no repositório
VULNERABILIDADE 2: Token de acesso sem proteção
VULNERABILIDADE 3: Sem tratamento de falha de rede
VULNERABILIDADE 4: Transação duplicada
VULNERABILIDADE 5: Token de longa duração
```

Mas não é apenas teoria — **2 dessas vulnerabilidades já causaram incidentes reais com danos ao usuário**.

---

## Parte 3: As 5 Vulnerabilidades Detalhadas

### VULNERABILIDADE 1: Chave Secreta Exposta no Repositório

#### O Que Aconteceu

O desenvolvimento do FinanceTrack estava no GitHub (repositório público). Um dos desenvolvedores, por engano, colocou a **chave de API do servidor** (client_secret) diretamente no código:

```java
// App FinanceTrack — Arquivo MainActivity.java
public class MainActivity extends AppCompatActivity {

    private static final String CLIENT_ID = "financetrack_app";
    private static final String CLIENT_SECRET = "[CREDENCIAL_REMOVIDA]"; // ❌ ERRADO!

    // ... resto do código
}
```

#### Como Foi Descoberto

Bots automáticos varriam constantemente o GitHub procurando por padrões que pareçam "segredos" (strings que começam com `sk_`, `pk_`, etc.).

O bot encontrou, salvou em um banco de dados público, e agora está circulando em fóruns de hackers há 3 semanas.

#### Por Que É Perigoso

Qualquer pessoa com essa chave consegue:

```
1. Fazer login falsificado no app FinanceTrack
2. Acessar dados de qualquer usuário
3. Fazer transferências e saques
4. Comprometer a identidade de usuários
```

#### Impacto Real

- ✅ Já causou incidente confirmado
- 🕐 Circulando há 3 semanas
- 💰 Múltiplos usuários potencialmente afetados

#### Por Que a Solução Está no OAuth 2.0 + PKCE

O padrão correto para apps móveis **não usa client_secret no app**. Usa um desafio criptográfico único a cada autenticação (PKCE). Assim, mesmo que alguém descompile o app, não encontra nada útil.

---

### VULNERABILIDADE 2: Token de Acesso Sem Proteção

#### O Que Aconteceu

Após fazer login, o app FinanceTrack recebe um token JWT do servidor. Esse token comprova que você é quem diz ser e permite acessar dados financeiros.

O app, porém, guarda esse token de forma **completamente insegura**:

```java
// ❌ ERRADO — Código do FinanceTrack
SharedPreferences prefs = context.getSharedPreferences("financetrack", MODE_PRIVATE);
String token = response.getJWT();
prefs.edit().putString("token", token).apply(); // Guardado em TEXTO PURO!
```

#### Como Foi Descoberto

Um testador conectou um celular Android com FinanceTrack instalado via USB. Com uma ferramenta chamada **ADB (Android Debug Bridge)**, conseguiu:

```
1. Acessar o armazenamento do app
2. Ler SharedPreferences (é um arquivo XML texto puro)
3. Extrair o token JWT
4. Fazer requisições usando o token roubado
```

**Tempo total: 2 minutos.**

#### Por Que É Perigoso

Uma vez que tem o token:

```
Invasor usa o token para:
    ↓
1. Ver saldo, extrato, histórico
2. Fazer transferências
3. Apagar transações
4. Tudo que o usuário legítimo consegue fazer
```

E tudo isso **sem ter a senha do usuário**.

#### Impacto Real

- ✅ Já causou incidente confirmado
- 💰 Usuários prejudicados
- ⏰ Token válido por 30 dias (próxima vulnerabilidade)

#### Por Que a Solução Está em Armazenamento Seguro

Existem formas seguras de guardar dados sensíveis:

```
❌ SharedPreferences (Android) — texto puro
❌ NSUserDefaults (iOS) — sem criptografia
✅ Android Keystore — criptografia com hardware
✅ iOS Keychain — idem, protegido pelo hardware do celular
```

Com Android Keystore, mesmo com acesso físico, o token não pode ser extraído em texto puro.

---

### VULNERABILIDADE 3: Sem Tratamento de Falha de Rede

#### O Que Aconteceu

O app não trata corretamente erros de comunicação com o servidor.

Cenário:

```
1. Usuário está em metrô (conexão intermitente)
2. Tenta fazer uma transferência: POST /api/transferencia
3. Servidor está temporariamente indisponível (erro 503)
4. App exibe uma tela de erro genérica:
   "Erro ao processar. Tente novamente mais tarde."
5. Usuário clica "OK" e fecha a tela de erro
6. Sem saber: foi o app que cometeu o erro, não o servidor
7. Usuário desinstala app por "estar quebrado"
8. Ou: Usuário pensa que funcionou e faz outra transferência
```

#### Como Foi Descoberto

Testador simulou falha de rede (erro 503 do servidor) e observou o comportamento do app.

#### Por Que É Perigoso (Mais Crítico Que Parece)

A falta de retry adequado causa:

```
1. Usuário não sabe o estado real da operação
   - "Meu dinheiro saiu? Ou não?"

2. Em caso de erro genuíno (servidor down), usuário não consegue tentar novamente
   - Precisa reiniciar app manualmente

3. Em finanças, essa incerteza é CRÍTICA
   - Risco de operação duplicada (próxima vuln)
```

#### Por Que a Solução Está em Retry + Idempotência

Retry automático com backoff exponencial:

```
Tentativa 1: Falha (503)
    ↓ Espera 1 segundo
Tentativa 2: Falha (503)
    ↓ Espera 2 segundos
Tentativa 3: Sucesso! (200)
    ↓ Exibe confirmação
```

Mas só funciona se combinado com idempotência (ver próxima vulnerabilidade).

---

### VULNERABILIDADE 4: Transação Duplicada (Sem Idempotência)

#### O Que Aconteceu

Um usuário estava usando FinanceTrack em conexão 4G instável. Fez uma transferência de **R$ 150,00** para sua mãe.

```
Segundo 1: App envia POST /api/transferencia { "valor": 150 }
    ↓
Segundo 2: Conexão cai
    ↓
Segundo 3: App não recebe resposta do servidor
    ↓
Segundo 4: App tenta novamente (retry automático)
    ↓
Segundo 5: App envia POST /api/transferencia { "valor": 150 }
    ↓
Segundo 6: Conexão volta
    ↓
Segundo 7: Servidor processa AMBAS as requisições
    ↓
Resultado: Duas transferências de R$ 150 cada
    ↓
Débito total: R$ 300 em vez de R$ 150
    ↓
❌ PREJUÍZO: -R$ 150
```

#### Mas Espera... Não Deveria Ser Só Retry que Fix?

**Não.** Retry sem **idempotência** causa duplicação.

A solução é: cada operação precisa de um **identificador único** que o servidor guarda. Se receber a mesma requisição duas vezes (mesmo ID), ignora a segunda.

#### Como Funciona a Solução

```
Tentativa 1:
POST /api/transferencia
{
  "valor": 150,
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"  ← UUID único!
}
Servidor: Processa, guarda o ID

Tentativa 2 (mesma requisição, porque conexão caiu):
POST /api/transferencia
{
  "valor": 150,
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"  ← Mesmo UUID!
}
Servidor: "Já processei isso! Ignora e devolve a resposta anterior"

Resultado: Transação única de R$ 150 ✓
```

#### Impacto Real

- ✅ Já causou incidente confirmado
- 💰 Usuário perdeu R$ 150 reais
- 📊 Problema crítico em operações financeiras

---

### VULNERABILIDADE 5: Token de Longa Duração (30 Dias)

#### O Que Aconteceu

Após fazer login, o app recebe um token JWT válido por **30 dias**.

```
Dia 1: Você faz login, recebe token
    ↓
Dia 2-29: Token continua válido
    ↓
Dia 30: Token expira
    ↓
Você faz novo login
```

#### Por Que É Perigoso

**Se alguém roubar seu token (vulnerabilidade 2), tem 30 dias para usar.**

```
Dia 1: Hacker rouba token
    ↓
Dia 2-29: Hacker consegue acessar sua conta
    ↓
Dia 3: Hacker faz transferência para conta própria
    ↓
Dia 30: Token expira, hacker não consegue mais acessar
    ↓
Resultado: 27 dias de acesso não autorizado = Prejuízo massivo
```

#### Qual É o Padrão Correto?

A **RFC 9700** (padrão OAuth 2.0) recomenda:

```
Access Token: 5 a 15 minutos
    ↓ Invasor consegue no máximo 15 minutos de acesso
    ↓ Minimiza dano

Refresh Token: Dias a meses (revogarável)
    ↓ Usado para renovar access token
    ↓ Pode ser revogado instantaneamente se comprometido
```

Exemplo de fluxo correto:

```
Login bem-sucedido:
    ↓
Recebe:
  - access_token: válido por 15 minutos
  - refresh_token: válido por 30 dias

Após 15 minutos:
    ↓
App usa refresh_token para pedir novo access_token
    ↓
Servidor: "OK, aqui está novo access_token"

Se refresh_token for comprometido:
    ↓
Usuário faz logout
    ↓
Servidor revoga o refresh_token
    ↓
Hacker não consegue mais renovar access_token
```

#### Impacto

- ❌ Não causou incidente ainda, mas é questão de tempo
- 🕐 Risco persistente por 30 dias

---

## Parte 4: Contexto Técnico do Projeto

### Stack Atual do FinanceTrack

```
Frontend (App Android/iOS):
  - HTTP library: http 1.2
  - Problema: Sem retry automático, sem timeout

Autenticação:
  - Bearer token simples (JWT)
  - Problema: Sem PKCE, sem refresh_token

Token:
  - Validade: 30 dias
  - Problema: Muito longo (deveria ser 5-15 minutos)

Armazenamento:
  - SharedPreferences (Android)
  - Problema: Sem criptografia

Operações:
  - POST /api/transferencia
  - Problema: Sem identificador único (idempotency_key)
```

---

## Parte 5: Resumo das 5 Vulnerabilidades

| #   | Vulnerabilidade              | O Que Acontecer?                       | Incidente?   | Severidade |
| --- | ---------------------------- | -------------------------------------- | ------------ | ---------- |
| 1   | Chave secreta no repositório | Qualquer um consegue client_secret     | ✅ Sim       | CRÍTICA    |
| 2   | Token sem proteção           | Token roubado via ADB em 2 minutos     | ✅ Sim       | CRÍTICA    |
| 3   | Sem tratamento de rede       | Usuário não sabe se operação processou | ❌ Não       | MODERADA   |
| 4   | Transação duplicada          | R$ 150 vira R$ 300                     | ✅ Sim       | ALTA       |
| 5   | Token por 30 dias            | Hacker tem 30 dias com acesso          | ❌ Ainda não | ALTA       |

---

## Parte 6: A Atividade Que Você Precisa Resolver

### O Que Pede

**Classifique cada um dos 5 problemas usando OWASP Mobile Top 10 2024 (M1 a M10). Explique por que cada categoria se aplica.**

### Por Que Fazer Isso?

1. **Priorização**: Saber qual categoria ajuda a empresa a priorizar qual problema atacar primeiro
2. **Educação**: Você aprende a reconhecer padrões de vulnerabilidade
3. **Comunicação**: Classificar com OWASP permite falar a "língua" da segurança

### Como Resolver

```
Para cada vulnerabilidade, responda:

1. Qual categoria OWASP? (M1 a M10)
2. Por que essa categoria se aplica?
3. Qual é o risco específico?
4. Como se relaciona com autenticação segura?
```

### Exemplo (Vulnerabilidade Fictícia)

```
VULNERABILIDADE: App salva senha em arquivo texto

RESPOSTA:
Categoria: M5 (Insecure Data Storage)

Por que: Dados sensíveis (senha) guardados sem criptografia

Risco: Qualquer app no celular consegue ler a senha

Relação com autenticação:
Se a senha está legível, OAuth 2.0 não funciona porque
o "segredo" não é mais segredo. Qualquer um consegue
fazer login usando a senha roubada.
```

---

## Parte 7: Por Que Isso Importa (Além da Atividade)

### Você Está Estudando Um Problema Real

Essas vulnerabilidades não são fictícias:

```
✅ Segredos no código: Aconteceu com AWS, GitHub, Twilio, etc.
✅ Token sem proteção: Vulnerabilidade comum em apps Android
✅ Sem retry: Causa de bugs reais em finanças
✅ Duplicação: Problema conhecido em pagamentos
✅ Token longo: Má prática em OAuth até anos recentes
```

### Você Está Aprendendo Segurança Real

A cada vulnerabilidade, você está aprendendo:

```
Vuln 1 → Como proteger segredos? (OAuth 2.0 + PKCE)
Vuln 2 → Onde guardar tokens? (Android Keystore/iOS Keychain)
Vuln 3-4 → Como comunicar com servidor? (Retry + Idempotência)
Vuln 5 → Quanto tempo token deve valer? (5-15 minutos)
```

Essas são habilidades que desenvolvedores de verdade precisam.

---

## Parte 8: Estrutura da Resposta Esperada

No próximo arquivo (`04-respostas-atividade.md`), você verá:

```
Para cada vulnerabilidade:

1. NOME DA VULNERABILIDADE

2. CATEGORIA OWASP (M1-M10)

3. EXPLICAÇÃO DIDÁTICA
   - O que é?
   - Por que se aplica ao FinanceTrack?
   - Qual é o risco?

4. RELAÇÃO COM AUTENTICAÇÃO SEGURA
   - Como OAuth 2.0 preveniria?
   - Como armazenamento correto preveniria?
   - Como PKCE preveniria?

5. CENÁRIO DE ATAQUE
   - Passo a passo: como um invasor exploraria

6. SOLUÇÃO
   - O que fazer para corrigir
```

---

## Resumo Final

### O FinanceTrack Tem 5 Problemas

1. **Chave no código** → Compromete autenticação
2. **Token inseguro** → Roubo local
3. **Sem retry** → Operações incertas
4. **Sem idempotência** → Duplicação de transações
5. **Token longo** → Acesso prolongado se roubado

### Sua Atividade

Classificar cada um com OWASP Mobile Top 10 e explicar por quê.

### Por Que É Importante

Segurança de dados financeiros é crítica. Um bug pode custar milhões.

---

## Próximo Arquivo

O arquivo `04-respostas-atividade.md` contém as respostas completas, didaticamente explicadas.
