# Respostas da Atividade: Classificação OWASP das Vulnerabilidades do FinanceTrack

## Introdução

Este arquivo contém as **respostas completas e detalhadas** para a classificação OWASP Mobile Top 10 2024 de cada uma das 5 vulnerabilidades do FinanceTrack.

Para cada vulnerabilidade, você encontrará:

1. **A vulnerabilidade** — O que é
2. **Classificação OWASP** — Qual categoria (M1-M10)
3. **Explicação fundamentada** — Por que essa categoria
4. **Relação com autenticação segura** — Como se conecta com OAuth 2.0
5. **Cenário de ataque** — Como um invasor exploraria
6. **Solução recomendada** — Como corrigir

---

## VULNERABILIDADE 1: Chave Secreta Exposta no Repositório

### Resumo

A `client_secret` da API estava escrita diretamente no código-fonte do app, em um repositório público no GitHub. Foi descoberta por um bot automatizado e agora circula em fóruns de hackers há 3 semanas.

```java
// ❌ Código Problemático
public class OAuthConfig {
    public static final String CLIENT_SECRET = "[CREDENCIAL_REMOVIDA]";
}
```

---

### Classificação OWASP

## **M8 — Security Misconfiguration (Má Configuração de Segurança)**

---

### Por Que M8 Se Aplica?

#### Definição de M8

M8 cobre **qualquer erro de configuração ou decisão arquitetural inadequada** que exponha segurança. Inclui:

- Colocar segredos no código-fonte ❌
- Debug ativado em produção ❌
- Permissões desnecessárias ❌
- Dependências desatualizadas ❌
- Certificados inválidos ❌

#### No FinanceTrack

Colocar `client_secret` no app é o exemplo clássico de **configuração errada**:

1. **Decisão errada**: "Vou guardar o segredo no app"
   - Um segredo deve estar apenas no servidor
2. **Visibilidade errada**: Repositório público
   - Qualquer pessoa consegue ver

3. **Implementação errada**: Em texto puro, no código-fonte
   - Descompilação trivial

#### O Raciocínio

Não é M1 (credenciais mal usadas) porque o problema não está em **usar** a credencial, mas em **onde** ela foi configurada.

---

### Relação com Autenticação Segura

#### OAuth 2.0 + PKCE Como Solução

O padrão OAuth 2.0 foi criado **especificamente** porque apps móveis não conseguem guardar segredos com segurança:

```
❌ ERRADO (Configuração atual do FinanceTrack):
    App contém: client_secret = "sk_live_..."
    ↓
    Qualquer pessoa descompila e encontra

✅ CORRETO (OAuth 2.0 + PKCE):
    App contém: client_id = "financetrack_app" (público)
    ↓
    App gera code_verifier único a cada autenticação
    ↓
    Server valida usando PKCE (SHA-256)
    ↓
    Nenhum segredo no app!
```

#### Por Que PKCE Protege

```
Invasor descompila o app:
    ↓
Encontra: client_id (público, não tem problema)
    ↓
Não encontra: client_secret (não existe!)
    ↓
Não encontra: code_verifier (gerado aleatoriamente a cada vez)
    ↓
Resultado: Nada útil para atacar
```

---

### Cenário de Ataque

```
PASSO 1: Bot de varredura encontra
========================================
client_secret = "[CREDENCIAL_REMOVIDA]"
no repositório público GitHub

PASSO 2: Segredo fica em banco de dados público
========================================
Hacker consulta: "Quais segredos foram vazados?"
Encontra: [CREDENCIAL_REMOVIDA]

PASSO 3: Hacker cria um app falso
========================================
Cria: MyFinanceTracker (app clone)
Embute: client_secret = "[CREDENCIAL_REMOVIDA]"

PASSO 4: Usuário engana-se e instala app falso
========================================
App falso redireciona login para servidor legítimo
Usuário faz login normalmente
App falso captura o authorization_code

PASSO 5: App falso faz login como usuário real
========================================
POST /oauth/token
{
  "client_id": "financetrack_app",
  "client_secret": "[CREDENCIAL_REMOVIDA]",  ← Segredo roubado!
  "code": authorization_code_do_usuario_real
}

PASSO 6: App falso recebe access_token
========================================
Token é válido por 30 dias (vulnerabilidade 5!)

PASSO 7: Hacker consegue acessar conta do usuário
========================================
GET /api/saldo
Authorization: Bearer [access_token do usuário]
Resposta: { "saldo": 50000 }

PASSO 8: Hacker faz transferências
========================================
POST /api/transferencia
{
  "destino": "conta_do_hacker",
  "valor": 50000
}

RESULTADO: Prejuízo de R$ 50.000
```

---

### Solução Recomendada

#### 1. Remover Segredo do Código (Imediato)

```
❌ ANTES:
public static final String CLIENT_SECRET = "sk_live_...";

✅ DEPOIS:
// Nenhum segredo no app
```

#### 2. Implementar OAuth 2.0 + PKCE

```java
// ✅ Implementação correta
public class OAuthManager {

    // Gera code_verifier único a cada autenticação
    String codeVerifier = generateRandomString(128);

    // Calcula code_challenge
    String codeChallenge = Base64.getUrlSafeEncoder()
        .encodeToString(sha256(codeVerifier));

    // Envia apenas o code_challenge (nunca o verifier)
    String authUrl = SERVER_URL + "/oauth/authorize?" +
        "client_id=financetrack_app&" +
        "code_challenge=" + codeChallenge + "&" +
        "response_type=code";

    // Passo 5: Envia code + verifier original
    // (verifier nunca foi transmitido antes)
    Map<String, String> body = new HashMap<>();
    body.put("code", authorizationCode);
    body.put("code_verifier", codeVerifier);

    // Servidor verifica: SHA-256(verifier) === challenge?
}
```

#### 3. Revocar Segredo Comprometido

```
Imediato:
  1. Revogar: [CREDENCIAL_REMOVIDA]
  2. Notificar: Todos os usuários
  3. Gerar: Novo client_secret no servidor

30 dias:
  1. Invalidar: Todos os access_tokens antigos
  2. Forçar: Novo login de todos os usuários
```

#### 4. Implementar Secrets Management

```
❌ ANTES:
public static final String CLIENT_SECRET = "hardcoded";

✅ DEPOIS:
// Usar servidor de secrets (AWS Secrets Manager, HashiCorp Vault)
String secret = SecretsManager.getSecret("financetrack/client_secret");
// Ou: Usar environment variables, nunca código-fonte
```

---

### Tabela Comparativa: Antes vs Depois

| Aspecto              | ❌ Antes (M8)      | ✅ Depois (Seguro)          |
| -------------------- | ------------------ | --------------------------- |
| Segredo no app?      | Sim, em texto puro | Não                         |
| Repositório público? | Sim, exposto       | Não, ou mascarado           |
| Se descompilar app?  | Encontra segredo   | Não encontra nada           |
| Se roubar token?     | Usa por 30 dias    | Usa por 15 minutos          |
| Solução              | PKCE               | OAuth 2.0 + PKCE + Keystore |

---

---

## VULNERABILIDADE 2: Token de Acesso Sem Proteção

### Resumo

O app salva o JWT de autenticação em **SharedPreferences** (Android) ou **NSUserDefaults** (iOS) — ambos armazenam dados em **texto puro** sem criptografia. Um testador conseguiu extrair o token em 2 minutos usando ADB.

```java
// ❌ Código Problemático
SharedPreferences prefs = context.getSharedPreferences("financetrack", MODE_PRIVATE);
String token = response.getJWT();
prefs.edit().putString("token", token).apply();
// Armazenado sem criptografia ❌
```

---

### Classificação OWASP

## **M9 — Insecure Data Storage (Armazenamento Inseguro de Dados)**

---

### Por Que M9 Se Aplica?

#### Definição de M9

M9 cobre dados sensíveis guardados de forma insegura no celular:

- Senhas em arquivo texto ❌
- Tokens sem criptografia ❌
- Informação pessoal em cache ❌
- Dados em backup não criptografado ❌

#### No FinanceTrack

O JWT (token de autenticação) é **muito sensível**:

```
O Token Contém:
  - Identidade do usuário
  - Permissões
  - Prova de autenticação

Se alguém conseguir o token:
  - Consegue acessar a conta
  - Consegue fazer transferências
  - Consegue ver dados privados
```

Guardá-lo em **SharedPreferences** (sem criptografia) é catastrófico:

```
SharedPreferences:
  └── financetrack.xml (arquivo em texto puro)
      ├── token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
      ├── email = "usuario@email.com"
      └── saldo = "1250.00"

Qualquer app no celular pode ler!
Backup via ADB expõe tudo!
```

---

### Relação com Autenticação Segura

#### A Camada 4 da Fórmula de Segurança

Lembre-se da fórmula de autenticação segura:

```
CAMADA 1: Protocolo seguro (OAuth 2.0 + PKCE)     ← Vulnerabilidade 1
CAMADA 2: Token assinado (JWT)
CAMADA 3: Validade curta (15 minutos)              ← Vulnerabilidade 5
CAMADA 4: ARMAZENAMENTO SEGURO (Keystore)          ← Vulnerabilidade 2 ❌
CAMADA 5: Transmissão por HTTPS                    ← Vulnerabilidade 3-4
```

Se a camada 4 falhar, **todo o resto não importa**:

```
Mesmo com OAuth 2.0 perfeito:
  Se o token fica guardado em texto puro,
  qualquer um consegue roubar.
```

---

### Cenário de Ataque

```
PASSO 1: Usuário usa FinanceTrack normalmente
========================================
Faz login → Recebe token JWT
App salva em SharedPreferences (sem criptografia)

PASSO 2: Celular é roubado ou emprestado
========================================
Hacker tem posse física do celular

PASSO 3: Hacker conecta celular via USB
========================================
Hacker usa ADB (Android Debug Bridge):

$ adb shell
$ run-as com.financetrack
$ cat files/shared_prefs/financetrack.xml
```

RESULTADO:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
  <string name="token">eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoiam9hb0BmaW5hbmNl
LmNvbSIsImV4cCI6MTcyNTAwMDAwMH0.SflKxwRJSMeKKF2QT4fw
pMeJf36POk6yJV_adQssw5c</string>
  <string name="email">joao@finance.com</string>
  <string name="saldo">5000.00</string>
</map>
```

# PASSO 4: Hacker extrai o token

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
email = "joao@finance.com"

# PASSO 5: Hacker abre seu próprio app e muda o token

SharedPreferences do seu app também tem: token = "..."
Hacker muda para: token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
(do usuário roubado)

# PASSO 6: Hacker faz requisições como o usuário real

GET /api/extrato
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Resposta: Extrato completo do usuário

POST /api/transferencia
{
"destino": "conta_do_hacker",
"valor": 5000
}
Resultado: ✓ Transferência realizada

TEMPO TOTAL: ~2 minutos (como relatado no pentest)

RESULTADO: Prejuízo = R$ 5.000

````

---

### Solução Recomendada

#### 1. Usar Android Keystore (Android)

```java
// ✅ Implementação correta para Android
import android.security.keystore.KeyGenParameterSpec;
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import android.security.keystore.KeyProperties;

public class SecureTokenStorage {

    private static final String KEYSTORE_ALIAS = "financetrack_token_key";
    private static final String CIPHER_TRANSFORMATION =
        "AES/GCM/NoPadding";

    // Gera chave no Android Keystore (protegida pelo hardware)
    public static void createKey() throws Exception {
        KeyGenerator keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore");

        keyGenerator.init(new KeyGenParameterSpec.Builder(
            KEYSTORE_ALIAS,
            KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .build());

        keyGenerator.generateKey();
    }

    // Armazena token de forma criptografada
    public static void saveToken(String token) throws Exception {
        SecretKey key = (SecretKey) KeyStore
            .getInstance("AndroidKeyStore")
            .getKey(KEYSTORE_ALIAS, null);

        Cipher cipher = Cipher.getInstance(CIPHER_TRANSFORMATION);
        cipher.init(Cipher.ENCRYPT_MODE, key);

        byte[] encryptedToken = cipher.doFinal(token.getBytes());

        SharedPreferences prefs = context.getSharedPreferences(
            "financetrack_secure", MODE_PRIVATE);
        prefs.edit()
            .putString("token", Base64.encodeToString(encryptedToken, 0))
            .apply();
    }

    // Recupera token descriptografado
    public static String getToken() throws Exception {
        SecretKey key = (SecretKey) KeyStore
            .getInstance("AndroidKeyStore")
            .getKey(KEYSTORE_ALIAS, null);

        SharedPreferences prefs = context.getSharedPreferences(
            "financetrack_secure", MODE_PRIVATE);
        String encryptedToken = prefs.getString("token", null);

        Cipher cipher = Cipher.getInstance(CIPHER_TRANSFORMATION);
        cipher.init(Cipher.DECRYPT_MODE, key);

        byte[] decryptedToken = cipher.doFinal(
            Base64.decode(encryptedToken, 0));

        return new String(decryptedToken);
    }
}

// Uso:
SecureTokenStorage.createKey();
SecureTokenStorage.saveToken("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...");
String token = SecureTokenStorage.getToken();
````

#### 2. Usar iOS Keychain (iOS)

```swift
// ✅ Implementação correta para iOS
import Foundation

class SecureTokenStorage {

    static func saveToken(_ token: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "financetrack_token",
            kSecValueData as String: token.data(using: .utf8)!,
            kSecAttrAccessible as String:
                kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        // Remove token antigo se existir
        SecItemDelete(query as CFDictionary)

        // Adiciona novo token ao Keychain
        SecItemAdd(query as CFDictionary, nil)
    }

    static func getToken() -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "financetrack_token",
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(
            query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data,
              let token = String(data: data, encoding: .utf8) else {
            return nil
        }

        return token
    }

    static func deleteToken() {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "financetrack_token"
        ]

        SecItemDelete(query as CFDictionary)
    }
}

// Uso:
SecureTokenStorage.saveToken("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")
let token = SecureTokenStorage.getToken()
SecureTokenStorage.deleteToken() // ao fazer logout
```

#### 3. Por Que Keystore/Keychain Funciona

```
SharedPreferences (❌ Inseguro):
  └── Arquivo XML em texto puro
      └── Qualquer processo consegue ler
      └── Backup via ADB expõe

Android Keystore (✅ Seguro):
  └── Chave criptográfica no hardware
      ├── Material criptográfico protegido pelo TPM
      ├── Não pode ser extraído em texto puro
      └── Mesmo com acesso físico, não consegue token
```

---

### Tabela Comparativa: Antes vs Depois

| Aspecto             | ❌ Antes (M9)                  | ✅ Depois (Seguro)               |
| ------------------- | ------------------------------ | -------------------------------- |
| Armazenamento       | SharedPreferences (texto puro) | Android Keystore (criptografado) |
| Proteção            | Nenhuma                        | Hardware TPM                     |
| Se conectar USB?    | Extrai token em 2 minutos      | Não consegue extrair             |
| Se celular roubado? | Token em texto puro            | Token criptografado, inacessível |
| Validade do token?  | 30 dias de risco               | 15 minutos (+ outra solução)     |

---

---

## VULNERABILIDADE 3: Sem Tratamento de Falha de Rede

### Resumo

Quando o servidor retorna erro temporário (503 — Service Unavailable), o app exibe uma tela de erro genérica sem tentar novamente automaticamente. O usuário precisa reiniciar manualmente o app.

```
Erro 503: Servidor temporariamente indisponível
    ↓
App exibe: "Erro ao processar. Tente novamente mais tarde."
    ↓
Usuário fecha tela de erro
    ↓
Sem retry automático
    ↓
Usuário desiste ou tenta manualmente (causa vuln. 4)
```

---

### Classificação OWASP

## **M3 — Insecure Communication (Comunicação Insegura)**

---

### Por Que M3 Se Aplica?

#### Definição de M3

M3 cobre problemas com **transmissão e tratamento de dados** entre app e servidor:

- Comunicação sem HTTPS ❌
- SSL/TLS mal configurado ❌
- Certificados inválidos aceitos ❌
- **Sem tratamento de erro/falha** ❌
- **Requisições duplicadas (sem idempotência)** ❌

#### No FinanceTrack

Pode parecer que não tem nada a ver com "comunicação insegura", mas tem:

```
M3 não é SÓ sobre criptografia.

É também sobre:
  - Confiabilidade da comunicação
  - Tratamento de falhas
  - Garantia de que operação foi processada
```

Em finanças, **confiabilidade é segurança**:

```
Usuário não sabe se transferência foi processada:
  ↓
Usuário tenta de novo manualmente
  ↓
Requisição é enviada duas vezes (sem idempotência)
  ↓
Transação é duplicada (vulnerabilidade 4)
  ↓
Prejuízo financeiro
```

---

### Relação com Autenticação Segura

#### Confiabilidade é Parte da Segurança

```
OAuth 2.0 garante: "Você é quem diz ser"

Mas não garante:
  - "Sua operação foi processada"
  - "Sua operação foi processada UMA VEZ"
  - "Você sabe o resultado"

Sem retry + idempotência, operações financeiras são inseguras.
```

#### Conexão com Vulnerabilidades Anteriores

```
Mesmo que tudo antes funcione:
  ✅ OAuth 2.0 correto (não M8)
  ✅ Token seguro no celular (não M9)
  ✅ Token válido por 15 minutos (não M5)

Se não tiver retry:
  ❌ Operações incertas
  ❌ Sem confirmação de sucesso
  ❌ Abertura para vulnerabilidade 4
```

---

### Cenário de Ataque

```
PASSO 1: Usuário em rede instável (metrô, avião)
========================================
Conexão 4G intermitente
Latência alta, timeouts frequentes

PASSO 2: Usuário tenta fazer transferência
========================================
POST /api/transferencia
{
  "valor": 150,
  "destino": "conta_mae"
}

PASSO 3: Conexão cai
========================================
Requisição enviada, mas servidor está indisponível (503)

PASSO 4: App exibe erro genérico
========================================
"Erro ao processar requisição. Tente novamente mais tarde."

PASSO 5: App não tenta novamente automaticamente
========================================
Sem retry automático!

PASSO 6: Usuário tem dúvida
========================================
"A transferência saiu? Ou não?"
"Se eu reiniciar o app, posso perder a transferência?"
"Se não reiniciar, vou enviar de novo e duplicar?"

PASSO 7: Usuário toma decisão errada
========================================
Opção 1: Não faz nada (operação não é processada, reclamação)
Opção 2: Clica em "Tentar Novamente" manualmente (manual retry)
Opção 3: Reinicia app (segundo tentativa, novo retry)

Se opção 2 ou 3:
  ↓
  VULNERABILIDADE 4 (transação duplicada!)

RESULTADO: Ou prejuízo (não foi processada)
          Ou duplicação (foi processada 2 vezes)
```

---

### Solução Recomendada

#### 1. Implementar Retry Automático com Backoff Exponencial

```java
// ✅ Implementação correta
public class RetryManager {

    private static final int MAX_RETRIES = 3;
    private static final int INITIAL_DELAY = 1000; // 1 segundo

    public static Response makeRequestWithRetry(Request request)
            throws Exception {

        int retryCount = 0;
        long delay = INITIAL_DELAY;
        Exception lastException = null;

        while (retryCount <= MAX_RETRIES) {
            try {
                Response response = client.execute(request);

                // Sucesso
                if (response.code() == 200) {
                    return response;
                }

                // Erro temporário (5xx) = retry
                if (response.code() >= 500) {
                    throw new TemporaryException("Erro temporário: " +
                        response.code());
                }

                // Erro permanente (4xx) = não tenta
                if (response.code() >= 400) {
                    return response;
                }

            } catch (IOException | TemporaryException e) {
                lastException = e;
                retryCount++;

                if (retryCount <= MAX_RETRIES) {
                    // Espera antes de tentar de novo (backoff exponencial)
                    Thread.sleep(delay);
                    delay *= 2; // Próxima tentativa espera 2x mais

                    Log.d("Retry", "Tentativa " + retryCount +
                        " em " + delay + "ms");
                } else {
                    break;
                }
            }
        }

        // Falhou após todas as tentativas
        throw new Exception("Falhou após " + MAX_RETRIES +
            " tentativas", lastException);
    }
}

// Uso:
Request request = new Request.Builder()
    .url("https://api.financetrack.com/transferencia")
    .post(body)
    .build();

try {
    Response response = RetryManager.makeRequestWithRetry(request);
    handleSuccess(response);
} catch (Exception e) {
    handleFailure(e); // Mostra erro final
}
```

#### 2. Definir Timeouts Apropriados

```java
// ✅ Configuração de timeout
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)   // Espera 10s para conectar
    .writeTimeout(10, TimeUnit.SECONDS)     // Espera 10s para escrever
    .readTimeout(30, TimeUnit.SECONDS)      // Espera 30s para resposta
    .retryOnConnectionFailure(true)         // Retry automático
    .build();
```

#### 3. Padrão: Retry com Jitter

Para evitar "thundering herd" (todas as requisições retentam ao mesmo tempo):

```java
// ✅ Retry com jitter (aleatoriedade)
long delay = INITIAL_DELAY + random.nextInt((int) INITIAL_DELAY);
Thread.sleep(delay);
```

#### 4. Diferençar Erro Temporário de Permanente

```java
// ✅ Lógica inteligente de retry
public static boolean shouldRetry(Response response) {

    int code = response.code();

    // ✅ Retentáveis (temporários)
    if (code == 408 ||   // Request Timeout
        code == 429 ||   // Too Many Requests
        code >= 500) {   // Server errors (5xx)
        return true;
    }

    // ❌ Não retentáveis (permanentes)
    if (code == 401 ||   // Unauthorized
        code == 403 ||   // Forbidden
        code == 404 ||   // Not Found
        code < 500) {    // Client errors (4xx)
        return false;
    }

    return false;
}
```

---

### Padrão Recomendado: Circuit Breaker

Para evitar sobrecarregar servidor:

```
Estado FECHADO: Requisições vão normalmente
    ↓ 5 falhas consecutivas
Estado ABERTO: Todas as requisições falham imediatamente
    ↓ Espera 30 segundos
Estado MEIO-ABERTO: Tenta 1 requisição
    ↓ Se suceder: volta a FECHADO
    ↓ Se falhar: volta a ABERTO
```

---

### Tabela Comparativa: Antes vs Depois

| Aspecto          | ❌ Antes (M3)                    | ✅ Depois (Seguro)          |
| ---------------- | -------------------------------- | --------------------------- |
| Erro 503?        | "Erro. Tente depois"             | Retry automático 3x         |
| Tempo de espera? | Nenhum                           | 1s, 2s, 4s (exponencial)    |
| Resultado?       | Incerto, usuário tenta novamente | Confirmado ou erro claro    |
| Duplicação?      | Possível (vuln 4)                | Impossível (+ Idempotência) |

---

---

## VULNERABILIDADE 4: Transação Duplicada (Sem Idempotência)

### Resumo

Em rede instável, a mesma transação financeira foi registrada **3 vezes** porque o app reenviou sem identificador único. Resultado: usuário tinha R$ 150 debitados, mas perdeu R$ 450 em vez de R$ 150 (prejuízo de R$ 300).

```
Requisição 1: POST /api/transferencia { "valor": 150 }
    ↓ Enviada
    ↓ Conexão cai antes de receber resposta

Requisição 2: POST /api/transferencia { "valor": 150 }
    ↓ App tenta de novo (retry)
    ↓ Conexão cai de novo

Requisição 3: POST /api/transferencia { "valor": 150 }
    ↓ App tenta de novo (retry)
    ↓ Desta vez funciona!

Resultado: Servidor processou todas as 3 = 3 × R$ 150 = R$ 450
Dano: -R$ 300 ao usuário
```

---

### Classificação OWASP

## **M3 — Insecure Communication (Comunicação Insegura)**

_Mesma categoria da Vulnerabilidade 3, mas por motivo diferente_

---

### Por Que M3 Se Aplica?

#### Definição de M3 (Parte 2: Idempotência)

M3 não é apenas sobre HTTPS. É sobre **integridade e confiabilidade** da comunicação.

Especificamente: **operações devem ser idempotentes** — repetir não deve mudar o resultado.

#### O Problema: Operações Não-Idempotentes

```
❌ NÃO IDEMPOTENTE:
POST /api/transferencia { "valor": 150 }
Enviada 1x: Débito de R$ 150
Enviada 2x: Débito de R$ 300 (ERRADO!)
Enviada 3x: Débito de R$ 450 (MUITO ERRADO!)

✅ IDEMPOTENTE:
POST /api/transferencia
  { "valor": 150, "idempotency_key": "uuid_1234" }
Enviada 1x: Débito de R$ 150, guarda UUID
Enviada 2x: Servidor vê UUID duplicado, ignora
Enviada 3x: Servidor vê UUID duplicado, ignora
Resultado: Débito único de R$ 150 ✓
```

#### No FinanceTrack

O app não implementa **Idempotency-Key**:

```java
// ❌ Errado
POST /api/transferencia
{
  "destino": "conta_mae",
  "valor": 150
}
// Sem identificador único!

// ✅ Correto
POST /api/transferencia
{
  "destino": "conta_mae",
  "valor": 150,
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

### Relação com Autenticação Segura

#### Idempotência é Parte de Segurança em Finanças

```
Mesmo com:
  ✅ OAuth 2.0 correto
  ✅ Token seguro
  ✅ HTTPS correto
  ✅ Retry automático

Se não for idempotente:
  ❌ Operações duplicam
  ❌ Prejuízo financeiro
  ❌ Violação da integridade dos dados
```

#### Relação com Vulnerabilidade 3

```
Vulnerabilidade 3 (sem retry):
  ❌ Operações não são confirmadas

Solução: Adicionar retry (Vulnerabilidade 3 resolvida)

Mas: Retry sem idempotência causa Vulnerabilidade 4!

Solução completa: Retry + Idempotência
```

---

### Cenário de Ataque (Ou Melhor: Falha de Segurança)

```
PASSO 1: Usuário em rede 4G instável
========================================
Latência: 3000ms, packet loss: 20%

PASSO 2: Usuário faz transferência
========================================
POST /api/transferencia HTTP/1.1
Authorization: Bearer [token]
Content-Type: application/json

{
  "destino": "conta_mae",
  "valor": 150
}

PASSO 3: Servidor recebe e começa a processar
========================================
UPDATE usuarios SET saldo = saldo - 150 WHERE id = joao

PASSO 4: Rede cai DEPOIS do servidor receber
========================================
Servidor completou a operação
MAS a resposta não chegou ao app

App recebe timeout (conexão caiu)

PASSO 5: App implementa retry (Vuln 3 foi "resolvida")
========================================
Espera 1 segundo
Tenta de novo

POST /api/transferencia HTTP/1.1
{
  "destino": "conta_mae",
  "valor": 150
}

PASSO 6: Sem identificador único
========================================
Servidor não consegue diferenciar:
  "Essa é a mesma requisição que caiu?"
  ou
  "É uma nova transferência?"

PASSO 7: Servidor processa como NOVA transferência
========================================
UPDATE usuarios SET saldo = saldo - 150 WHERE id = joao

Saldo: Inicial R$ 300
       Após 1ª transf: R$ 150
       Após 2ª transf: R$ 0

PASSO 8: App tenta novamente (2ª retry)
========================================
POST /api/transferencia HTTP/1.1
{
  "destino": "conta_mae",
  "valor": 150
}

PASSO 9: Terceira vez é a vencida
========================================
Servidor processa como MAIS UMA transferência

Saldo: R$ 0
       Após 3ª transf: R$ -150 (NEGATIVO!)

RESULTADO:
  Transferência intencional: R$ 150
  Débitos reais: R$ 450
  PREJUÍZO: R$ 300
```

---

### Solução Recomendada

#### 1. Implementar Idempotency-Key

```java
// ✅ Gerar UUID único a cada operação
import java.util.UUID;

public class TransferenceManager {

    public static String makeTransfer(String destination,
                                     double amount) throws Exception {

        // Gera um UUID único para essa operação
        String idempotencyKey = UUID.randomUUID().toString();

        // Armazena localmente para poder reenviar se necessário
        saveIdempotencyKey(idempotencyKey);

        Request request = new Request.Builder()
            .url("https://api.financetrack.com/transferencia")
            .post(RequestBody.create(
                "{\"destino\": \"" + destination + "\", " +
                "\"valor\": " + amount + ", " +
                "\"idempotency_key\": \"" + idempotencyKey + "\"}",
                MediaType.parse("application/json")
            ))
            .addHeader("Authorization", "Bearer " + getToken())
            .addHeader("Idempotency-Key", idempotencyKey) // ← Importante!
            .build();

        Response response = client.newCall(request).execute();
        return response.body().string();
    }
}
```

#### 2. Lado do Servidor: Guardar Operações Processadas

```python
# ✅ Implementação no backend (exemplo Python/Flask)
from flask import Flask, request
import json
from datetime import datetime, timedelta

app = Flask(__name__)

# Banco de dados (em produção, seria SQL)
processed_idempotency_keys = {}

@app.route('/api/transferencia', methods=['POST'])
def transferencia():

    data = request.get_json()
    idempotency_key = request.headers.get('Idempotency-Key')

    # Verifica se já processamos esse Idempotency-Key
    if idempotency_key in processed_idempotency_keys:
        # Retorna a resposta antiga (sem processar de novo)
        cached_response = processed_idempotency_keys[idempotency_key]
        print(f"Idempotency-Key {idempotency_key} já foi processada!")
        return cached_response['response'], cached_response['status']

    # Processa transferência normalmente
    try:
        valor = data['valor']
        destino = data['destino']
        usuario_id = get_current_user_id()  # Do token

        # Executa transferência (simplificado)
        database.transfer(usuario_id, destino, valor)

        response = {
            'status': 'sucesso',
            'transacao_id': 'txn_123456'
        }
        status = 200

        # Guarda para próximas requisições duplicadas
        processed_idempotency_keys[idempotency_key] = {
            'response': response,
            'status': status,
            'timestamp': datetime.now()
        }

        return response, status

    except Exception as e:
        return {'erro': str(e)}, 500


# Limpeza: remover chaves antigas (ex: após 24 horas)
def cleanup_old_keys():
    now = datetime.now()
    expired_keys = [
        key for key, data in processed_idempotency_keys.items()
        if (now - data['timestamp']) > timedelta(hours=24)
    ]
    for key in expired_keys:
        del processed_idempotency_keys[key]
```

#### 3. Armazenar Localmente para Reenvio

```java
// ✅ Guardar operações pendentes
public class PendingOperations {

    public static void savePendingTransfer(String idempotencyKey,
                                          String destination,
                                          double amount) {
        // Armazena em SQLite local
        ContentValues values = new ContentValues();
        values.put("idempotency_key", idempotencyKey);
        values.put("destination", destination);
        values.put("amount", amount);
        values.put("status", "pending");
        values.put("timestamp", System.currentTimeMillis());

        db.insert("pending_transfers", null, values);
    }

    public static void markAsCompleted(String idempotencyKey) {
        // Marca como concluída
        ContentValues values = new ContentValues();
        values.put("status", "completed");

        db.update("pending_transfers", values,
                 "idempotency_key = ?",
                 new String[]{idempotencyKey});
    }

    // Reenviar operações pendentes quando app abrir
    public static void resendPendingOperations() {
        Cursor cursor = db.query("pending_transfers",
            null, "status = ?", new String[]{"pending"},
            null, null, null);

        while (cursor.moveToNext()) {
            String key = cursor.getString(
                cursor.getColumnIndex("idempotency_key"));
            String dest = cursor.getString(
                cursor.getColumnIndex("destination"));
            double amt = cursor.getDouble(
                cursor.getColumnIndex("amount"));

            // Reenviar
            makeTransfer(dest, amt, key);
        }
    }
}
```

---

### Padrão Recomendado: Rastreabilidade Completa

```
UUID único (Idempotency-Key)
    ↓ Garante operação única
    ↓
Armazenar no servidor por 24-48 horas
    ↓ Permite reenvio sem duplicação
    ↓
Armazenar localmente no app
    ↓ Permite reenvio offline e reconfirmação
```

---

### Tabela Comparativa: Antes vs Depois

| Aspecto                    | ❌ Antes (M3)    | ✅ Depois (Seguro)     |
| -------------------------- | ---------------- | ---------------------- |
| Identificador único?       | Não              | Sim (UUID)             |
| Reenvio de falha?          | Duplica          | Ignorado (idempotente) |
| Rastreabilidade?           | Nenhuma          | Completa               |
| Resultado de 3 tentativas? | R$ 450 debitados | R$ 150 debitados ✓     |

---

---

## VULNERABILIDADE 5: Token de Longa Duração (30 Dias)

### Resumo

O access token JWT é válido por **30 dias**. A RFC 9700 (padrão OAuth 2.0) recomenda 5 a 15 minutos. Se o token for roubado, o invasor tem 30 dias de acesso irrestrito à conta.

```
Access Token:
  ├─ Válido por: 30 dias (❌ ERRADO)
  ├─ Se roubado: 30 dias de acesso
  └─ Deveria ser: 5-15 minutos (✅ CORRETO)
```

---

### Classificação OWASP

## **M4 — Insufficient Authentication (Autenticação Insuficiente)**

---

### Por Que M4 Se Aplica?

#### Definição de M4

M4 cobre fraqueza no processo de autenticação e gestão de sessão:

- Senha fraca aceita ❌
- Token válido por tempo muito longo ❌
- Sem autenticação multifator (2FA) ❌
- Sem revogar sessão ao deslogar ❌
- OAuth sem PKCE ❌

#### No FinanceTrack

Um access token é essencialmente uma **sessão de usuário**.

```
Sessão normal na web:
  "Você fez login, aqui está seu cookie"
  Cookie válido por 1-2 horas

Access token no app:
  "Você fez login, aqui está seu token"
  Token válido por... 30 DIAS?!

Isso é anormalmente longo.
```

#### O Princípio: Curta Validade = Menos Dano

```
Se token é roubado:

Token de 15 minutos:
  Invasor consegue usar por: ≤ 15 minutos
  Dano: Limitado

Token de 30 dias:
  Invasor consegue usar por: ≤ 30 dias
  Dano: Massivo (transferências, saques, etc.)
```

---

### Relação com Autenticação Segura

#### A Estratégia: Access Token Curto + Refresh Token Revogarável

```
ANTES (❌ Inseguro):
Login → access_token válido 30 dias → USA POR 30 DIAS

DEPOIS (✅ Seguro):
Login → access_token (15 min) + refresh_token (30 dias)
  ↓
Após 15 min:
  access_token expirou
  ↓
App usa refresh_token para pedir novo access_token
  ↓
Servidor: OK, aqui está novo access_token (15 min)
  ↓
Se refresh_token for roubado:
  Usuário faz logout
  ↓
  Servidor revoga refresh_token
  ↓
  Hacker não consegue mais renovar
```

#### Por Que Refresh Token é Melhor

```
Access Token (curto): Usado em cada requisição
  - Expõe menos (validade curta)
  - Viaja pela rede em cada requisição

Refresh Token (longo): Usado raramente
  - Guardado com segurança (Keystore)
  - Viaja pela rede apenas ao renovar
  - Pode ser revogado instantaneamente
```

---

### Cenário de Ataque

```
PASSO 1: Token roubado (Vulnerabilidade 2)
========================================
Hacker consegue token via ADB em celular

Token = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

PASSO 2: Hacker vê que é válido por 30 dias
========================================
Decodifica token, vê: "exp": 1756000000 (daqui a 30 dias)

PASSO 3: Hacker faz requisições como o usuário
========================================
DIA 1:
  GET /api/saldo
  Authorization: Bearer eyJ...
  Resposta: Saldo R$ 50.000

DIA 2:
  POST /api/transferencia
  { "destino": "hacker_account", "valor": 10000 }
  Resultado: ✓ Transferência processada

DIA 3-5:
  Hacker continua fazendo transferências
  Múltiplas transferências de R$ 10.000
  Total: R$ 50.000 transferidos

DIA 30:
  Token expira
  Hacker não consegue mais acessar

PASSO 4: Usuário descobre depois de semanas
========================================
"Por que minha conta está vazia?!"
"Quando isso aconteceu?"
"Não tenho mais dinheiro!"

RESULTADO: Prejuízo total de R$ 50.000+
          (e cansaço administrativo/judicial)
```

**Compare com:**

```
Token de 15 minutos:

DIA 1, HORA 1:30:
  Hacker rouba token

DIA 1, HORA 1:45:
  Token expira
  Hacker tenta fazer transferência

Resposta: 401 Unauthorized (token expirado)

Hacker não consegue fazer nada!

RESULTADO: Prejuízo total = R$ 0
```

---

### Solução Recomendada

#### 1. Configurar Access Token Curto (15 minutos)

```java
// ✅ No servidor: gerar token com expiração curta
public class TokenGenerator {

    public static Map<String, Object> generateTokens(String userId) {

        long now = System.currentTimeMillis();

        // Access Token: 15 minutos
        long accessTokenExpiry = now + (15 * 60 * 1000);

        String accessToken = JWT.create()
            .withSubject(userId)
            .withIssuedAt(new Date(now))
            .withExpiresAt(new Date(accessTokenExpiry))
            .withClaim("type", "access")
            .sign(Algorithm.HMAC256(SECRET_KEY));

        // Refresh Token: 7 dias
        long refreshTokenExpiry = now + (7 * 24 * 60 * 60 * 1000);

        String refreshToken = JWT.create()
            .withSubject(userId)
            .withIssuedAt(new Date(now))
            .withExpiresAt(new Date(refreshTokenExpiry))
            .withClaim("type", "refresh")
            .sign(Algorithm.HMAC256(SECRET_KEY));

        Map<String, Object> tokens = new HashMap<>();
        tokens.put("access_token", accessToken);
        tokens.put("token_type", "Bearer");
        tokens.put("expires_in", 900); // 15 minutos em segundos
        tokens.put("refresh_token", refreshToken);

        return tokens;
    }
}
```

#### 2. Implementar Endpoint de Renovação

```java
// ✅ Endpoint para renovar access token
@PostMapping("/oauth/token")
public ResponseEntity<?> refreshToken(
    @RequestBody TokenRefreshRequest request) {

    try {
        // Valida o refresh_token
        String refreshToken = request.getRefreshToken();
        DecodedJWT decoded = JWT.require(
            Algorithm.HMAC256(SECRET_KEY)
        ).build().verify(refreshToken);

        String userId = decoded.getSubject();
        String tokenType = decoded.getClaim("type").asString();

        // Verifica se é realmente um refresh_token
        if (!"refresh".equals(tokenType)) {
            return ResponseEntity.status(401)
                .body("Inválido token");
        }

        // Gera novo access_token
        String newAccessToken = generateAccessToken(userId);

        return ResponseEntity.ok(Map.of(
            "access_token", newAccessToken,
            "token_type", "Bearer",
            "expires_in", 900
        ));

    } catch (JWTVerificationException e) {
        return ResponseEntity.status(401)
            .body("Refresh token inválido ou expirado");
    }
}

private String generateAccessToken(String userId) {
    long now = System.currentTimeMillis();
    long expiry = now + (15 * 60 * 1000); // 15 minutos

    return JWT.create()
        .withSubject(userId)
        .withIssuedAt(new Date(now))
        .withExpiresAt(new Date(expiry))
        .withClaim("type", "access")
        .sign(Algorithm.HMAC256(SECRET_KEY));
}
```

#### 3. App: Renovar Automaticamente Antes de Expirar

```java
// ✅ No app: renovar token antes de expirar
public class TokenRefreshManager {

    private static final long REFRESH_THRESHOLD = 5 * 60 * 1000; // 5 min antes

    public static void ensureTokenValid() throws Exception {

        String accessToken = SecureTokenStorage.getAccessToken();
        String refreshToken = SecureTokenStorage.getRefreshToken();

        if (accessToken == null) {
            throw new Exception("Sem access token");
        }

        // Decodifica para ver expiração
        DecodedJWT decoded = JWT.decode(accessToken);
        long expiresAt = decoded.getExpiresAt().getTime();
        long now = System.currentTimeMillis();
        long timeUntilExpiry = expiresAt - now;

        // Se vai expirar em menos de 5 minutos, renova AGORA
        if (timeUntilExpiry < REFRESH_THRESHOLD) {
            refreshAccessToken(refreshToken);
        }
    }

    private static void refreshAccessToken(String refreshToken)
            throws Exception {

        Request request = new Request.Builder()
            .url("https://api.financetrack.com/oauth/token")
            .post(RequestBody.create(
                "{\"grant_type\": \"refresh_token\", " +
                "\"refresh_token\": \"" + refreshToken + "\"}",
                MediaType.parse("application/json")
            ))
            .build();

        Response response = client.newCall(request).execute();

        if (response.code() == 200) {
            String body = response.body().string();
            JSONObject json = new JSONObject(body);

            String newAccessToken = json.getString("access_token");

            // Guarda novo token
            SecureTokenStorage.saveAccessToken(newAccessToken);
        } else {
            throw new Exception("Falha ao renovar token");
        }
    }
}

// Usar antes de cada requisição importante:
@Override
protected Response doSomethingImportant() throws Exception {
    TokenRefreshManager.ensureTokenValid();

    return makeRequest("/api/transferencia", ...);
}
```

#### 4. Implementar Revogação de Refresh Token

```java
// ✅ Revogação instantânea
@PostMapping("/oauth/logout")
public ResponseEntity<?> logout(
    @RequestBody LogoutRequest request,
    @RequestHeader("Authorization") String authHeader) {

    String refreshToken = request.getRefreshToken();

    // Adiciona refresh_token à lista negra (blacklist)
    tokenBlacklist.add(refreshToken);

    // Ou: marca como revogado no banco
    tokenRepository.revokeRefreshToken(refreshToken);

    return ResponseEntity.ok(Map.of("message", "Logout bem-sucedido"));
}

// Ao renovar, verifica blacklist:
if (tokenBlacklist.contains(refreshToken)) {
    throw new Exception("Token foi revogado");
}
```

---

### Comparação: Antes vs Depois

```
❌ ANTES (Inseguro):
Access Token: Válido por 30 dias
Se roubado: 30 dias de acesso
Revogação: Impossível (token já foi assinado)

✅ DEPOIS (Seguro):
Access Token: Válido por 15 minutos
Se roubado: ≤ 15 minutos de acesso
Refresh Token: Válido por 7 dias, revogarável
Se refresh_token roubado: Revoga instantaneamente
```

---

### Tabela Comparativa: Antes vs Depois

| Aspecto           | ❌ Antes (M4)                     | ✅ Depois (Seguro)  |
| ----------------- | --------------------------------- | ------------------- |
| Validade do token | 30 dias                           | 15 minutos          |
| Se roubado        | 30 dias de acesso                 | ≤ 15 minutos        |
| Refresh token     | Não existe                        | 7 dias, revogarável |
| Revogação         | Impossível                        | Instantânea         |
| Conformidade RFC  | Não (RFC 9700 recomenda 5-15 min) | Sim ✓               |

---

---

## RESUMO FINAL: Tabela Completa das 5 Vulnerabilidades

### Classificação OWASP

| #   | Vulnerabilidade              | Categoria OWASP                      | Severidade | Incidente?   |
| --- | ---------------------------- | ------------------------------------ | ---------- | ------------ |
| 1   | Chave secreta no repositório | **M8** — Security Misconfiguration   | CRÍTICA    | ✅ Sim       |
| 2   | Token sem proteção           | **M9** — Insecure Data Storage       | CRÍTICA    | ✅ Sim       |
| 3   | Sem tratamento de rede       | **M3** — Insecure Communication      | MODERADA   | ❌ Não       |
| 4   | Transação duplicada          | **M3** — Insecure Communication      | ALTA       | ✅ Sim       |
| 5   | Token por 30 dias            | **M4** — Insufficient Authentication | ALTA       | ❌ Não (yet) |

---

### Por Que Cada Categoria?

| Vuln | Categoria | Raciocínio                               |
| ---- | --------- | ---------------------------------------- |
| 1    | M8        | Configuração errada: segredo no código   |
| 2    | M9        | Dados sensíveis guardados inseguramente  |
| 3    | M3        | Comunicação não confiável (sem retry)    |
| 4    | M3        | Operações duplicadas (sem idempotência)  |
| 5    | M4        | Token com validade inadequadamente longa |

---

### Cadeia de Causa e Efeito

```
M8 (Vuln 1): Segredo no código
    ↓
Hacker consegue client_secret
    ↓
Faz login falsificado

M9 (Vuln 2): Token sem proteção
    ↓
Hacker rouba token via USB
    ↓
Acessa conta do usuário

M4 (Vuln 5): Token válido 30 dias
    ↓
Hacker tem 30 dias com acesso
    ↓
Faz transferências múltiplas

M3 (Vuln 3-4): Sem retry ou idempotência
    ↓
Usuário clica "tentar de novo" manualmente
    ↓
Transação é duplicada
    ↓
Débito múltiplo
```

---

### Fórmula de Segurança Completa (e Como Cada Vuln Quebra)

```
✅ OAuth 2.0 + PKCE     ← Protegido por M8 (Vuln 1 quebra aqui)
   ├─ Client_secret no servidor, não app
   └─ code_verifier aleatório a cada vez

✅ Armazenamento Seguro  ← Protegido por M9 (Vuln 2 quebra aqui)
   ├─ Android Keystore / iOS Keychain
   └─ Token criptografado

✅ Access Token Curto    ← Protegido por M4 (Vuln 5 quebra aqui)
   ├─ Válido por 15 minutos
   └─ Dano limitado se roubado

✅ Comunicação Confiável ← Protegido por M3 (Vuln 3-4 quebram aqui)
   ├─ Retry automático
   ├─ Idempotency-Key
   └─ Garantia de processamento único

= SEGURANÇA COMPLETA
```

---

### Checklist de Segurança para o FinanceTrack

```
[ ] 1. Remover client_secret do código (M8)
[ ] 2. Implementar OAuth 2.0 + PKCE (M8)
[ ] 3. Guardar token em Android Keystore (M9)
[ ] 4. Guardar token em iOS Keychain (M9)
[ ] 5. Implementar retry automático (M3)
[ ] 6. Implementar Idempotency-Key (M3)
[ ] 7. Alterar access_token para 15 minutos (M4)
[ ] 8. Implementar refresh_token (M4)
[ ] 9. Implementar revogação de refresh_token (M4)
[ ] 10. Auditar logs de acesso (M8, M4)
```

---

## Conclusão

O FinanceTrack sofreu de vulnerabilidades em **praticamente todas as camadas** de segurança:

- **Autenticação** (M8, M4): Segredos expostos, tokens inválidos
- **Armazenamento** (M9): Dados sensíveis em texto puro
- **Comunicação** (M3): Sem retry ou idempotência

A boa notícia: **todas têm solução**, e as soluções são bem conhecidas e padronizadas.

Implementando OAuth 2.0 + PKCE corretamente, com armazenamento seguro e comunicação confiável, o FinanceTrack seria um app seguro.

---

## Próximas Etapas (Para o Desenvolvimento Real)

1. **Auditoria**: Revisar todo o código para vulnerabilidades similares
2. **Testes**: Pentest novo após implementar correções
3. **Notificação**: Alertar usuários sobre a exposição
4. **Revogação**: Invalidar todos os tokens antigos
5. **Monitoramento**: Detectar atividades suspeitas
6. **Documentação**: Criar guia de segurança para equipe

---

**Fim da análise OWASP Mobile Top 10 2024 do FinanceTrack**
