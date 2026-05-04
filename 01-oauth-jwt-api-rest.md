# Fundações: OAuth, JWT, API REST e Como Funcionam Juntos

## Introdução

Quando você usa seu telefone para acessar um app, existem três conceitos críticos trabalhando juntos para manter você seguro:

1. **API REST** — a forma como o app e o servidor conversam
2. **JWT (JSON Web Token)** — como o servidor confirma quem você é
3. **OAuth 2.0** — o protocolo que garante que essa confirmação é segura

Neste arquivo, vamos aprender cada um desses conceitos do zero, entender por que foram criados, e como eles se relacionam.

---

## Parte 1: O que é uma API REST?

### Conceito Básico

Uma **API REST** (Representational State Transfer) é um conjunto de regras que define como um app e um servidor devem se comunicar. Pense em um restaurante:

- **Você (cliente)** = seu app no celular
- **O garçom** = a API
- **A cozinha** = o servidor
- **Os pratos** = os dados que você quer

Você não entra na cozinha e pega o prato você mesmo. Você pede ao garçom: "Quero um café". O garçom leva seu pedido à cozinha, a cozinha prepara, e o garçom traz de volta. Pronto.

### Como o App e o Servidor Conversam

A API REST usa **requisições HTTP** — mensagens padronizadas que seguem regras bem definidas. Existem vários "tipos" de requisições, chamados de **métodos HTTP**:

- **GET** — Pedir informação (ler dados)
  - Exemplo: "Qual é meu saldo bancário?"
- **POST** — Enviar dados novos (criar algo)
  - Exemplo: "Quero fazer uma transferência de R$ 150"
- **PUT** — Atualizar dados existentes
  - Exemplo: "Mude minha senha"
- **DELETE** — Remover dados
  - Exemplo: "Delete esta transação"

### Exemplo Prático: Seu App Pedindo Saldo

```
Seu App → HTTP GET /api/saldo → Servidor FinanceTrack
Servidor → "Seu saldo é R$ 1.250,00" → Seu App
```

O servidor responde com um **código de status** que explica o que aconteceu:

- **200 OK** — Tudo funcionou! Aqui está a resposta.
- **400 Bad Request** — Seu pedido está errado, não consigo entender.
- **401 Unauthorized** — Você não se autenticou. Quem é você?
- **403 Forbidden** — Você não tem permissão para isso.
- **404 Not Found** — O recurso que você pediu não existe.
- **500 Internal Server Error** — Algo errou no servidor.
- **503 Service Unavailable** — O servidor está temporariamente indisponível.

### Por Que REST Virou Padrão

REST é simples, rápido e funciona em qualquer lugar — web, app mobile, IoT, etc. Todos entendem a mesma linguagem.

---

## Parte 2: O Problema da Segurança — Quem é Você?

Agora surge o primeiro problema: quando seu app faz um pedido ao servidor, **como o servidor sabe que é realmente você fazendo o pedido e não um invasor?**

Imagine o seguinte cenário malvado:

1. Você faz login no app FinanceTrack com seu email e senha
2. Seu app se conecta ao servidor, que valida a senha e diz "OK, você é o João"
3. Seu app recebe dados e começa a exibir na tela
4. **Mas então:** um hacker na mesma rede WiFi consegue "escutar" essa conversa e vê que seu app está pedindo `/api/saldo`
5. O hacker, do seu notebook, faz a mesma requisição sem ter sua senha...

**Sem segurança, o servidor responderia com seu saldo!**

A solução: após você fazer login com a senha, o servidor dá ao seu app um **comprovante de que você já se autenticou**. A cada requisição futura, seu app envia esse comprovante. É como um "cartão de acesso" temporário.

Esse comprovante chamamos de **token**.

---

## Parte 3: JWT — O "Cartão de Acesso" Digital

### O que é JWT?

**JWT** significa **JSON Web Token**. É um formato padrão de "comprovante" que o servidor cria após você fazer login. O servidor assina esse comprovante com uma chave secreta — assim ninguém consegue falsificar.

### A Estrutura de um JWT

Um JWT tem a forma: `xxxxxx.yyyyyy.zzzzzz`

É composto de 3 partes separadas por pontos:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvYW8gZGEgU2lsdmEiLCJpYXQiOjE1MTYyMzkwMjJ9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

#### Parte 1: Header (Cabeçalho)

Descodificando a primeira parte:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Explica: "Este é um JWT, assinado com o algoritmo HS256"

#### Parte 2: Payload (Carga)

Descodificando a segunda parte:

```json
{
  "sub": "1234567890",
  "name": "João da Silva",
  "email": "joao@financetrack.com",
  "iat": 1516239022,
  "exp": 1516325422
}
```

Aqui estão os dados sobre você:

- `sub` — ID único (subject)
- `name` — Seu nome
- `email` — Seu email
- `iat` — "Issued At" (quando foi criado)
- `exp` — "Expiration" (quando expira — válido por quanto tempo?)

#### Parte 3: Signature (Assinatura)

A terceira parte é a **assinatura digital**. O servidor toma as duas primeiras partes, aplica um algoritmo de criptografia secreto (usando uma chave que só ele conhece), e gera essa assinatura.

**Por que funciona?**

Se um hacker interceptar o JWT e tentar mudá-lo (exemplo: trocar `"email": "joao@financetrack.com"` para `"email": "hacker@gmail.com"`), a assinatura não vai mais bater. O servidor recalcula a assinatura da versão modificada e percebe que é diferente — logo, o token é falso.

### Ciclo de Vida de um JWT

```
1. Você digita email + senha no app FinanceTrack
2. Seu app envia para: POST /api/login
   {
     "email": "joao@financetrack.com",
     "password": "senha123"
   }

3. Servidor valida a senha no banco de dados

4. Servidor cria um JWT com seus dados:
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
   eyJzdWIiOiJqb2FvXzEyMyIsIm5hbWUiOiJKb8OjbyIsImV4cCI6MTUxNjMyNTQyMn0.
   SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

5. Servidor envia de volta ao app:
   {
     "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
     "token_type": "Bearer",
     "expires_in": 86400
   }

6. Seu app armazena o JWT

7. Nas próximas requisições, seu app envia:
   GET /api/saldo
   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

8. Servidor recebe o JWT, verifica a assinatura
   - Se válida: processa a requisição
   - Se inválida: retorna 401 Unauthorized
```

### Vantagens do JWT

- ✅ **Stateless** — O servidor não precisa guardar uma tabela de "quem está logado agora" porque o token carrega a informação
- ✅ **Escalável** — Funciona bem em sistemas com múltiplos servidores
- ✅ **Portável** — Funciona em web, app mobile, IoT, etc.
- ✅ **Seguro** — A assinatura garante que ninguém alterou o conteúdo

### Problemas do JWT Simples

Apesar de bom, JWT tem um problema crítico: **uma vez criado, é válido até expirar**. Se alguém roubar seu JWT:

- ❌ Não dá pra "revogar" instantaneamente — o servidor não consegue desinvalidar um token no meio do caminho
- ❌ Se o token é válido por 30 dias, o invasor tem 30 dias de acesso

É aqui que entra o **OAuth 2.0**.

---

## Parte 4: OAuth 2.0 — O Protocolo de Autenticação Segura

### Por Que OAuth Foi Criado?

Imagine este cenário antigo:

```
App FinanceTrack pede: "Me dê sua senha do banco para eu acessar sua conta"
```

Seria como dar sua chave de casa para um vizinho: não é seguro, você não consegue controlá-lo depois, e se ele perder a chave, qualquer um consegue entrar.

O **OAuth 2.0** foi criado para resolver isso. Ele permite que você dê permissão **limitada e temporária** sem compartilhar sua senha com ninguém além do banco.

### O Fluxo OAuth 2.0 (Versão Simplificada)

Imagine que você quer que o app FinanceTrack acesse seus dados no Banco do Brasil:

```
1. Você abre o app FinanceTrack
   ↓
2. Clica no botão "Conectar ao Banco do Brasil"
   ↓
3. App FinanceTrack redireciona você para o site do Banco do Brasil
   ↓
4. Você faz login com seu usuário e senha DO BANCO (não dá pra ninguém)
   ↓
5. Banco pergunta: "Você autoriza FinanceTrack a acessar seus dados?"
   ↓
6. Você clica "SIM"
   ↓
7. Banco volta com um "cartão de acesso temporário" (token)
   ↓
8. App FinanceTrack recebe o token
   ↓
9. App agora consegue pedir dados do Banco usando esse token
```

### Os Papéis no OAuth 2.0

Para entender OAuth, é importante saber que existem **4 atores**:

1. **Resource Owner (Você)** — Dono do recurso (seus dados no banco)
2. **Client (App FinanceTrack)** — A aplicação que quer acessar seus dados
3. **Authorization Server (Banco do Brasil)** — Quem verifica sua identidade e libera permissões
4. **Resource Server (Banco do Brasil)** — Quem guarda seus dados

(Às vezes, os papéis 3 e 4 são o mesmo servidor)

### Os Fluxos OAuth 2.0

Existem vários tipos de fluxo (chamados de "grant types") dependendo da situação. Os principais são:

#### 1. Authorization Code Flow (Fluxo de Código de Autorização)

Este é o **mais seguro** e o recomendado para apps móveis. Funciona assim:

```
[Seu App] ←→ [Banco] ←→ [Você — Usuario]

1. App envia ao Banco: "Eu sou FinanceTrack (client_id: abc123)"

2. Banco redireciona você para tela de login

3. Você faz login e autoriza

4. Banco envia de volta um "authorization_code" temporário
   (válido por 30 segundos, inútil sozinho)

5. App recebe o código E MANDA DE VOLTA:
   - authorization_code
   - client_secret (chave secreta do app)

6. Banco verifica tudo, confia que é realmente o FinanceTrack
   (porque tem a client_secret que só ele conhece)

7. Banco envia de volta:
   - access_token (JWT válido por 5-15 minutos)
   - refresh_token (válido por dias/meses)
```

**Por que é seguro?** Porque o `authorization_code` sozinho é inútil — o invasor não tem a `client_secret`. Apenas o app FinanceTrack (que guarda a chave secreta no servidor) consegue trocar o código por um token.

#### 2. Implicit Flow (Deprecated — Não Use!)

Era usado em aplicações web antigas. Hoje é considerado inseguro.

#### 3. Resource Owner Password Flow

Você dá sua senha direto para o app (não recomendado, porque confia demais no app):

```
Você: "Aqui está minha senha: senha123"
        ↓
App: "Recebi. Deixe-me trocar por um token"
        ↓
Servidor: "OK, aqui está seu token"
```

---

## Parte 5: PKCE — Proteção Extra para Apps Móveis

### Por Que Apps Móveis São Diferentes

Apps móveis têm um problema que apps web não têm: **podem ser descompilados**.

Se você tem um app APK (Android), qualquer pessoa consegue:

```
1. Baixar o app
2. Executar `apktool` para descompilar
3. Ver o código-fonte inteiro
4. Encontrar qualquer segredo (client_secret, chaves, etc.)
```

Se a `client_secret` estiver no código, qualquer um consegue encontrá-la. Isso é um **risco crítico**.

A solução: **PKCE** (Proof Key for Public Clients) — RFC 7636.

### Como PKCE Funciona

Ao invés de usar a `client_secret`, o app gera um desafio único **a cada autenticação**:

```
1. App gera um número aleatório gigante:
   code_verifier = "93qhf82hf923hf923hf923hf923hf2" (128 caracteres)

2. App criptografa esse número com SHA-256:
   code_challenge = SHA-256(code_verifier) = "9fE2...abcD"

3. App envia ao Banco:
   - client_id (código do app, público)
   - code_challenge (criptografado, não precisa de segredo)
   NÃO envia o code_verifier ainda!

4. Usuário faz login e autoriza

5. Banco envia authorization_code

6. App envia ao Banco:
   - authorization_code
   - code_verifier (AH! Agora envia!)

7. Banco recalcula: SHA-256(code_verifier) === code_challenge?

8. Se bater, Banco confia que é realmente o app original
   e envia o token
```

**Por que funciona?**

Mesmo que um invasor intercepte o `authorization_code` e o `code_challenge`, ele não consegue descobrir o `code_verifier` (é criptografia de sentido único).

---

## Parte 6: Ciclo Completo de Autenticação Segura

Vamos juntar tudo: API REST + JWT + OAuth 2.0 + PKCE

### Cenário: Você Faz Login no FinanceTrack

```
[SEU APP NO CELULAR]          [SERVIDOR FINANCETRACK]

1. Você toca "Conectar ao Banco"
                 ↓
2. App gera:
   - code_verifier (aleatório)
   - code_challenge = SHA-256(code_verifier)

3. App faz GET requisição:
   /oauth/authorize?
   client_id=financetrack123&
   code_challenge=9fE2...abcD&
   response_type=code
                 ↓
4.                         Servidor redireciona para tela de login
5.                         Você digita email + senha
6.                         Servidor valida no banco de dados
7.                         Servidor pergunta: "Autorizar?"
                 ←← Você clica SIM ←←
8.                         Servidor gera authorization_code temporário
                           (válido por 30 segundos)
9. App recebe authorization_code
                 ↓
10. App faz POST para /oauth/token:
    {
      "client_id": "financetrack123",
      "code": "authorization_code_aqui",
      "code_verifier": "93qhf82hf923hf923hf923hf923hf2"
    }
                 ↓
11.                        Servidor verifica:
                           - authorization_code é válido?
                           - SHA-256(code_verifier) === challenge armazenado?
                           ✓ Tudo certo!
                 ←←
12. App recebe:
    {
      "access_token": "eyJhbGci...JWT aqui",
      "token_type": "Bearer",
      "expires_in": 900,  // 15 minutos
      "refresh_token": "refresh_token_aqui"
    }
                 ↓
13. App armazena tokens com segurança (Android Keystore ou iOS Keychain)
                 ↓
14. Nas próximas requisições, app envia:
    GET /api/saldo
    Authorization: Bearer eyJhbGci...JWT aqui
                 ↓
15.                        Servidor valida o JWT (assinatura está OK?)
                           ✓ Sim! É você mesmo.
                           Retorna: { "saldo": 1250.00 }
                 ←←
16. App exibe o saldo na tela


[30 minutos depois...]
17. App tenta fazer outra requisição com o mesmo token
    GET /api/extrato
    Authorization: Bearer eyJhbGci...JWT aqui
                 ↓
18.                        Servidor verifica: Token expirou?
                           ✓ Sim, expirou. Retorna: 401 Unauthorized
                 ←←
19. App nota que recebeu 401
    Tem um refresh_token guardado? Sim!
                 ↓
20. App envia:
    POST /oauth/token
    {
      "grant_type": "refresh_token",
      "refresh_token": "refresh_token_aqui"
    }
                 ↓
21.                        Servidor valida o refresh_token
                           ✓ Válido! Gera novo access_token
                 ←←
22. App recebe novo token (válido por mais 15 minutos)
                 ↓
23. Retry automático:
    GET /api/extrato
    Authorization: Bearer [NOVO TOKEN]
                 ↓
24.                        Servidor: ✓ Token válido. Aqui está o extrato.
                 ←←
25. App exibe extrato na tela
```

---

## Parte 7: Como Tudo Se Conecta

### Diagrama Mental

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEGURANÇA EM CAMADAS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CAMADA 1: PROTOCOLO (API REST)                                 │
│  ├─ Define COMO comunicar (GET, POST, URLs, etc.)              │
│  └─ Exemplo: GET /api/saldo → 200 OK                           │
│                                                                  │
│  CAMADA 2: AUTENTICAÇÃO (OAuth 2.0 + PKCE)                      │
│  ├─ Define COMO E QUANDO você prova quem é                     │
│  ├─ Troca credenciais por um token temporário                  │
│  └─ Usa desafio criptográfico para proteger apps móveis        │
│                                                                  │
│  CAMADA 3: COMPROVANTE DE IDENTIDADE (JWT)                      │
│  ├─ Define O QUE você carrega para provar identidade            │
│  ├─ Token assinado, não pode ser falsificado                    │
│  └─ Válido por tempo limitado (15 minutos)                      │
│                                                                  │
│  CAMADA 4: ARMAZENAMENTO SEGURO                                 │
│  ├─ Define ONDE guardar o JWT no celular                        │
│  ├─ Android Keystore ou iOS Keychain                            │
│  └─ Hardware protegido, não em texto puro                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Na Prática: Cada Um Tem Seu Trabalho

| Conceito                 | Trabalho                                                  |
| ------------------------ | --------------------------------------------------------- |
| **API REST**             | Define os "endpoints" (rotas) e como pedir dados          |
| **OAuth 2.0**            | Define como você prova identidade de forma segura         |
| **PKCE**                 | Protege apps móveis contra roubo de segredos              |
| **JWT**                  | Formato do token que você carrega como "cartão de acesso" |
| **Armazenamento Seguro** | Onde guardar o JWT no celular                             |

---

## Parte 8: Resumo Final

### O Mínimo Que Você Precisa Saber

1. **API REST** = linguagem entre app e servidor
2. **OAuth 2.0 + PKCE** = forma segura de provar quem você é sem compartilhar senha
3. **JWT** = o "cartão de acesso" que você carrega depois que se autentica
4. **Tokens curtos + Refresh tokens longos** = segurança com praticidade
5. **Armazenamento seguro** = o token não pode estar em texto puro no celular

### A Fórmula Segura

```
PROTOCOLO SEGURO (OAuth 2.0 + PKCE)
        ↓
TOKEN ASSINADO (JWT)
        ↓
CURTA VALIDADE (15 minutos)
        ↓
ARMAZENAMENTO PROTEGIDO (Keystore/Keychain)
        ↓
= SEGURANÇA
```

### Próximos Passos

No próximo arquivo, você verá como as vulnerabilidades do FinanceTrack violam esses princípios e como o OWASP Mobile Top 10 classifica esses problemas.
