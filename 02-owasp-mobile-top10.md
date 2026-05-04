# OWASP Mobile Top 10 2024: Segurança em Aplicações Móveis

## Introdução

O OWASP (Open Web Application Security Project) é uma organização internacional que estuda segurança de software. A cada alguns anos, eles atualizam uma lista dos **10 principais problemas de segurança** encontrados em apps móveis.

Essa lista ajuda desenvolvedores e testadores a saber **onde procurar problemas** e como priorizá-los.

Este arquivo explica cada uma das 10 categorias do OWASP Mobile Top 10 2024 (M1 a M10), e como elas se relacionam com os conceitos de autenticação que você aprendeu no arquivo anterior.

---

## Contexto: Por Que Apps Móveis São Mais Vulneráveis?

### Diferenças Importantes

Apps móveis enfrentam desafios únicos que apps web não têm:

| Desafio           | App Web                      | App Móvel                        |
| ----------------- | ---------------------------- | -------------------------------- |
| **Descompilação** | Difícil (código em servidor) | Fácil (APK/IPA no celular)       |
| **Acesso Físico** | Improvável                   | Possível (celular roubado)       |
| **Conexões**      | WiFi confiável               | 4G, WiFi público, sem conexão    |
| **Armazenamento** | Banco de dados em servidor   | SharedPreferences/NSUserDefaults |
| **Usuários**      | Experientes                  | Desde crianças até idosos        |

Por isso, o OWASP tem uma lista separada para mobile.

---

## OWASP Mobile Top 10 2024 (M1 a M10)

### M1 — Improper Credential Usage (Uso Inadequado de Credenciais)

**O que é:** O app guarda, transmite ou valida credenciais (senhas, tokens, chaves) de forma insegura.

**Exemplos:**

- ❌ App envia senha em texto puro pela rede (sem HTTPS)
- ❌ App salva senha em um arquivo texto no celular
- ❌ App aceita qualquer certificado SSL (não valida a identidade do servidor)
- ❌ App não valida o servidor corretamente

**Por que é perigoso:**

- Interceptação: invasor na rede captura a senha/token
- Roubo local: qualquer app no celular consegue ler
- Man-in-the-middle: invasor finge ser o servidor

**Como prevenir:**

- ✅ Use HTTPS em TODAS as requisições
- ✅ Valide certificados SSL corretamente
- ✅ Nunca transmita senhas, use tokens (OAuth 2.0)
- ✅ Armazene tokens em Android Keystore / iOS Keychain

**Relação com Autenticação:**

- Violação da camada de **Segurança em Camadas** do OAuth 2.0
- O token não é mais um "comprovante seguro" se for transmitido sem HTTPS

---

### M2 — Inadequate Supply Chain Security (Segurança Inadequada na Cadeia de Suprimentos)

**O que é:** Problemas de segurança em dependências externas, bibliotecas terceirizadas, ou componentes que o app usa.

**Exemplos:**

- ❌ Seu app usa uma biblioteca HTTP desatualizada que tem vuln conhecida
- ❌ Seu app usa uma SDK de um serviço terceiro que foi comprometida
- ❌ Você baixa uma dependência de um repositório não confiável

**Por que é perigoso:**

- Código malicioso entra sem você saber
- Vulnerabilidades conhecidas não são corrigidas
- Você não pode revisar todo o código

**Como prevenir:**

- ✅ Mantenha bibliotecas atualizadas
- ✅ Use apenas repositórios confiáveis (Maven Central, CocoaPods oficial)
- ✅ Revise as dependências regularmente
- ✅ Use ferramentas como Snyk ou WhiteSource

**Relação com Autenticação:**

- Se a biblioteca HTTP tiver falhas, a comunicação OAuth não é segura
- Exemplo: biblioteca com HTTPS quebrada = credenciais expostas

---

### M3 — Insecure Communication (Comunicação Insegura)

**O que é:** Dados sensíveis são transmitidos sem proteção adequada, ou a comunicação não trata corretamente erros e exceções.

**Exemplos:**

- ❌ HTTP ao invés de HTTPS
- ❌ SSL/TLS mal configurado (versão antiga, algoritmo fraco)
- ❌ Certificados inválidos aceitos
- ❌ Dados sensíveis em logs ou cache
- ❌ Sem tratamento de falhas de rede (retry, timeout)
- ❌ Requisições idênticas ressinadas (sem nonce ou timestamp único)

**Por que é perigoso:**

- Invasor na mesma rede consegue capturar dados
- App não sabe se a operação foi processada (gera duplicatas)
- App não trata indisponibilidade do servidor

**Como prevenir:**

- ✅ Use HTTPS em TUDO
- ✅ Use TLS 1.2 ou 1.3
- ✅ Valide certificados
- ✅ Implemente retry automático com backoff exponencial
- ✅ Use Idempotency-Key em operações críticas
- ✅ Defina timeouts apropriados

**Relação com Autenticação:**

- Mesmo com OAuth 2.0 correto, se HTTPS falhar, o token é roubado
- Se sem retry, operação financeira fica em estado incerto

**Cenário Real do FinanceTrack:**

- Token transmitido por HTTP = M3
- Transação não retentada = M3 (sem idempotência)

---

### M4 — Insufficient Authentication (Autenticação Insuficiente)

**O que é:** Fraqueza no processo de autenticação ou na gestão de sessão/token.

**Exemplos:**

- ❌ Senha fraca aceita (não validar força)
- ❌ Sem autenticação multifator (2FA)
- ❌ Token válido por tempo muito longo (30 dias)
- ❌ Sem revogar sessão ao deslogar
- ❌ Sem re-autenticar para operações sensíveis
- ❌ OAuth sem PKCE em app móvel

**Por que é perigoso:**

- Senha fraca = invasor consegue adivinhar
- Token longo = invasor tem muito tempo com acesso
- Sem 2FA = invadir é fácil

**Como prevenir:**

- ✅ Enforce senhas fortes (mínimo 12 caracteres, complexidade)
- ✅ Implemente 2FA (autenticador, SMS, email)
- ✅ Access token curto (5-15 minutos)
- ✅ Refresh token revogarável
- ✅ Re-autentique para operações sensíveis
- ✅ Use OAuth 2.0 + PKCE em apps móveis

**Relação com Autenticação:**

- Violação direta dos princípios OAuth 2.0
- Token de 30 dias = M4 (deveria ser 5-15 minutos)

**Cenário Real do FinanceTrack:**

- Token válido por 30 dias = M4

---

### M5 — Insecure Data Storage (Armazenamento Inseguro de Dados)

**O que é:** Dados sensíveis guardados de forma não segura no celular.

**Exemplos:**

- ❌ Token JWT em SharedPreferences (Android)
- ❌ Senha em arquivo texto
- ❌ Dados em NSUserDefaults sem criptografia (iOS)
- ❌ Informação sensível em logs
- ❌ Backups não criptografados

**Por que é perigoso:**

- Qualquer app instalado consegue ler (permissões insuficientes)
- Backup via ADB (Android Debug Bridge) = dados expostos
- Celular roubado = acesso aos dados

**Como prevenir:**

- ✅ Android: Use Android Keystore
- ✅ iOS: Use iOS Keychain
- ✅ Criptografe dados sensíveis
- ✅ Não guarde senhas, use tokens
- ✅ Controle permissões de acesso

**Relação com Autenticação:**

- JWT guardado inseguramente = qualquer um consegue roubar
- Exemplo: token em SharedPreferences = qualquer app lê
- Violação da camada 4 (armazenamento seguro) da fórmula segura

**Cenário Real do FinanceTrack:**

- Token em armazenamento simples (sem criptografia) = M5

---

### M6 — Inadequate Privacy Controls (Controles de Privacidade Inadequados)

**O que é:** App não respeita privacidade do usuário ou coleta/compartilha dados além do necessário.

**Exemplos:**

- ❌ App coleta localização sem permissão
- ❌ App compartilha dados com terceiros sem consentimento
- ❌ App não oferece forma de deletar dados
- ❌ App não respeita "Do Not Track"
- ❌ Transparência insuficiente sobre coleta

**Por que é perigoso:**

- Violação de direitos do usuário
- LGPD/GDPR: multas pesadas
- Abuso de dados

**Como prevenir:**

- ✅ Peça permissão antes de usar câmera, localização, contatos
- ✅ Transparência: avise o que coleta
- ✅ Minimize coleta: pegue só o necessário
- ✅ Ofereça opção de deletar dados
- ✅ Não venda dados

---

### M7 — Client-side Injection (Injeção no Lado do Cliente)

**O que é:** App não valida entrada do usuário ou dados do servidor, permitindo injeção de código malicioso.

**Exemplos:**

- ❌ WebView renderiza HTML sem sanitizar
- ❌ SQL Injection em banco local
- ❌ Intent Injection (Android)
- ❌ Comando shell sem validação

**Por que é perigoso:**

- Código malicioso executado no contexto do app
- Acesso a dados sensíveis
- Roubo de token

**Como prevenir:**

- ✅ Valide SEMPRE entrada do usuário
- ✅ Sanitize HTML antes de renderizar
- ✅ Use prepared statements para SQL
- ✅ Valide dados vindos do servidor

---

### M8 — Security Misconfiguration (Má Configuração de Segurança)

**O que é:** Qualquer coisa relacionada a **configuração incorreta** de segurança no app ou no servidor.

**Exemplos:**

- ❌ API key no código-fonte do app
- ❌ Debug ativado em produção
- ❌ Servidor sem HTTPS
- ❌ Permissões do Android/iOS muito amplas
- ❌ Certificado autossinado em produção
- ❌ Senha padrão não alterada
- ❌ Bibliotecas desatualizadas com bugs conhecidos

**Por que é perigoso:**

- Segredos extraídos por descompilação
- Modo debug permite bypass de segurança
- Permissões amplas = acesso a tudo

**Como prevenir:**

- ✅ Nunca coloque segredos no app (use servidor)
- ✅ Desative debug em produção
- ✅ Use HTTPS
- ✅ Princípio de menor privilégio (peça permissões mínimas)
- ✅ Mantenha dependências atualizadas
- ✅ Revise configurações regularmente

**Relação com Autenticação:**

- API key no código = M8 (configuração errada)
- Client_secret no app móvel = M8 (deveria usar PKCE, não segredo)

**Cenário Real do FinanceTrack:**

- Client_secret no código-fonte = M8

---

### M9 — Insecure Data Storage (Armazenamento Inseguro de Dados) — Versão Expandida

Já foi explicado em M5, mas merece destaque especial:

**Especificamente para tokens e credenciais:**

- ❌ Token em SharedPreferences/NSUserDefaults
- ❌ Senha em arquivo texto
- ❌ Chave de criptografia hardcoded
- ❌ Dados sensíveis em cache HTTP

**Diferença entre M5 e M9:**

Não existe diferença significativa nas versões 2024 do OWASP Mobile Top 10 — é principalmente semântica. O importante é: **dados sensíveis não armazenados com segurança = problema crítico**.

**Como prevenir (para tokens especificamente):**

```
❌ ERRADO:
SharedPreferences prefs = context.getSharedPreferences("app", MODE_PRIVATE);
prefs.edit().putString("token", jwt).apply();

✅ CORRETO (Android):
KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
keyStore.load(null);
// ... usar keyStore para guardar token criptografado
```

```
✅ CORRETO (iOS):
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "token",
    kSecValueData as String: tokenData
]
SecItemAdd(query as CFDictionary, nil)
```

**Cenário Real do FinanceTrack:**

- Token sem criptografia no disco = M9

---

### M10 — Extraneous Functionality (Funcionalidade Estranha/Desnecessária)

**O que é:** App expõe funcionalidades destinadas apenas ao desenvolvimento, teste ou administrador para o usuário final.

**Exemplos:**

- ❌ Botão "debug" oculto que desativa segurança
- ❌ API interna exposta para cliente
- ❌ Modo "super user" sem autenticação
- ❌ Endpoint administrativo acessível via app
- ❌ Console de erro exibindo stack trace

**Por que é perigoso:**

- Invasor consegue bypass de segurança
- Exposição de lógica interna
- Facilita ataque

**Como prevenir:**

- ✅ Remova funcionalidades debug antes de produção
- ✅ Não exponha APIs internas
- ✅ Controle acesso a funcionalidades administrativas
- ✅ Não mostre stack traces para usuário final

---

## Tabela Resumida: OWASP Mobile Top 10 2024

| #       | Categoria                   | Descrição Resumida                                 | Impacto | Relação com Autenticação                |
| ------- | --------------------------- | -------------------------------------------------- | ------- | --------------------------------------- |
| **M1**  | Improper Credential Usage   | Credenciais mal usadas, transmitidas inseguramente | Alto    | Tokens roubados                         |
| **M2**  | Inadequate Supply Chain     | Bibliotecas/dependências comprometidas             | Crítico | Código malicioso em SDK de auth         |
| **M3**  | Insecure Communication      | HTTPS fraco, sem retry, sem idempotência           | Alto    | Tokens capturados, operações duplicadas |
| **M4**  | Insufficient Authentication | Autenticação fraca, sessão longa                   | Crítico | Acesso não autorizado prolongado        |
| **M5**  | Insecure Data Storage       | Dados sensíveis guardados inseguramente            | Crítico | Tokens roubados do celular              |
| **M6**  | Inadequate Privacy          | Coleta excessiva de dados                          | Médio   | Violação de privacidade                 |
| **M7**  | Client-side Injection       | Injeção de código no app                           | Alto    | Roubo de token, acesso de dados         |
| **M8**  | Security Misconfiguration   | Configuração incorreta de segurança                | Crítico | Segredos expostos no código             |
| **M9**  | Extraneous Data Exposure    | Armazenamento sem criptografia                     | Crítico | Tokens/senhas extraídas                 |
| **M10** | Extraneous Functionality    | Funcionalidade debug exposta                       | Médio   | Bypass de segurança                     |

---

## Como OWASP Mobile Se Relaciona com Autenticação Segura

### A Fórmula de Autenticação Segura (do arquivo anterior)

```
PROTOCOLO SEGURO (OAuth 2.0 + PKCE)  ← M1, M4, M8
        ↓
TOKEN ASSINADO (JWT)                  ← M7
        ↓
CURTA VALIDADE (15 minutos)            ← M4
        ↓
ARMAZENAMENTO PROTEGIDO (Keystore)     ← M5, M9
        ↓
TRANSMISSÃO POR HTTPS                  ← M1, M3
        ↓
IDENTIFICADOR ÚNICO POR REQUISIÇÃO     ← M3
        ↓
= SEGURANÇA
```

### Mapeamento: Cada Categoria OWASP Protege Uma Camada

| Camada        | Protegida Por             | Categoria OWASP |
| ------------- | ------------------------- | --------------- |
| Protocolo     | OAuth 2.0 + PKCE          | M1, M4, M8      |
| Token         | JWT assinado, prazo curto | M4, M7          |
| Armazenamento | Keystore/Keychain         | M5, M9          |
| Transmissão   | HTTPS, Idempotência       | M1, M3          |
| Gerenciamento | Revogar tokens, logs      | M4, M8          |

---

## Exemplo: Quebrando Cada Camada de Segurança

Imagine um invasor tentando atacar o app FinanceTrack:

### Ataque 1: Roubar Token na Transmissão

```
Invasor na rede WiFi pública
        ↓
Captura requisição HTTP (sem HTTPS) = M1, M3
        ↓
Extrai token JWT da requisição
        ↓
Usa token para fazer transferências
        ↓
RESULTADO: Prejuízo financeiro
```

**Proteção:** HTTPS obrigatório (M1, M3)

### Ataque 2: Roubar Token do Celular

```
Celular perde as mãos ou roubado
        ↓
Invasor conecta via USB
        ↓
Lê SharedPreferences onde token foi guardado = M5, M9
        ↓
Extrai token em texto puro
        ↓
Usa token para acessar conta
        ↓
RESULTADO: Prejuízo financeiro
```

**Proteção:** Android Keystore/iOS Keychain (M5, M9)

### Ataque 3: Roubar Segredo do App

```
Invasor baixa APK do Google Play
        ↓
Descompila com apktool
        ↓
Encontra client_secret no código = M8
        ↓
Cria novo app falso com o segredo
        ↓
Faz login falsificado
        ↓
RESULTADO: Prejuízo financeiro
```

**Proteção:** Usar PKCE (sem client_secret) (M4, M8)

### Ataque 4: Requisição Duplicada

```
Usuário tenta fazer transferência
        ↓
Conexão 4G cai
        ↓
App tenta novamente (sem idempotência)
        ↓
Servidor processa mesma requisição 2 vezes = M3
        ↓
Débito dobrado
        ↓
RESULTADO: Prejuízo do usuário
```

**Proteção:** Idempotency-Key em requisições (M3)

---

## Por Que Isso Importa: O Ciclo de Segurança

```
1. IDENTIDADE (Autenticação)
   ↓ Você prova quem é (OAuth 2.0 + PKCE)
   ↓ M1, M4, M8 protegem isso

2. ACESSO (Token)
   ↓ Você recebe um "cartão de acesso"
   ↓ M4 protege (validade curta)
   ↓ M7 protege (token não é injetável)

3. TRANSMISSÃO (HTTPS)
   ↓ Você envia token com segurança
   ↓ M1, M3 protegem

4. ARMAZENAMENTO (Keystore)
   ↓ Você guarda token criptografado no celular
   ↓ M5, M9 protegem

5. OPERAÇÕES (Idempotência)
   ↓ Suas ações não são duplicadas
   ↓ M3 protege

6. REVOGAR (Logout)
   ↓ Token é invalidado no servidor
   ↓ M4 protege

Se qualquer uma dessas camadas falhar, o todo cai.
```

---

## Resumo Final

### O Que Você Precisa Saber Sobre OWASP Mobile Top 10

1. **M1, M4, M8** = Autenticação correta (OAuth 2.0 + PKCE, não client_secret no app)
2. **M3** = Comunicação segura (HTTPS, retry, idempotência)
3. **M5, M9** = Armazenamento seguro (Keystore, não SharedPreferences)
4. **M2, M6, M7, M10** = Outros problemas também importantes

### A Lista Não É Independente

As categorias se relacionam. Violação de M1 afeta M4. Problema em M8 causa M5.

### Próximos Passos

No próximo arquivo, você verá como o FinanceTrack violou **todas essas proteções** e por isso sofreu incidentes reais.
