# 🔐 ironvault-auth

Serviço de autenticação e gestão de usuários do ecossistema IronVault.

## Stack
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- Redis (rate limiting)
- JWT (jjwt 0.12.3)

## Estrutura de Pastas

```
src/main/java/com/ironvault/auth/
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   ├── AuthController.java
│   │   │   ├── UsersController.java
│   │   │   ├── ApiKeyController.java
│   │   │   ├── InternalController.java
│   │   │   ├── LoginLogsController.java
│   │   │   ├── request/
│   │   │   └── response/
│   │   ├── common/
│   │   │   └── GlobalExceptionHandler.java
│   │   └── security/
│   │       ├── SecurityConfig.java
│   │       └── JwtAuthFilter.java
│   └── out/
│       ├── client/
│       │   └── NotificationClient.java
│       ├── entity/
│       └── persistence/
├── application/
│   ├── LoginService.java
│   ├── RegisterUserService.java
│   ├── ConfirmationEmailService.java
│   ├── ForgotPasswordService.java
│   ├── ResetPasswordService.java
│   ├── ChangePasswordService.java
│   ├── RefreshTokenService.java
│   ├── RateLimitingService.java
│   ├── GetAllUsersService.java
│   ├── GetUserByIdService.java
│   ├── UpdateUserStatusService.java
│   ├── UpdateUserRoleService.java
│   ├── UpdateUserMerchantIdService.java
│   ├── ApproveUserService.java
│   ├── RejectUserService.java
│   ├── GenerateApiKeyService.java
│   ├── GetApiKeyService.java
│   ├── ValidateApiKeyService.java
│   └── RevokeApiKeyService.java
└── domain/
    ├── model/
    ├── port/
    │   ├── in/
    │   └── out/
    └── enums/
```

## Variáveis de Ambiente

```env
DB_URL=jdbc:postgresql://<host>/ironvault-auth?sslmode=require
DB_USERNAME=
DB_PASSWORD=
JWT_SECRET=
JWT_EXPIRATION_MS=86400000
JWT_REFRESH_EXPIRATION_MS=604800000
NOTIFICATIONS_URL=https://notifications.ironvaultpayments.com.br
NOTIFICATIONS_API_KEY=
REDIS_URL=redis://default:<password>@<host>:<port>
AUTH_CONFIRMATION_URL=https://auth.ironvaultpayments.com.br
INTERNAL_API_KEY=
ADMIN_EMAIL=
ADMIN_PASSWORD=
PORT=8081
```

`INTERNAL_API_KEY` é compartilhada entre `ironvault-auth`, `ironvault-payments` e `ironvault-merchants` — usada para autenticar chamadas serviço-a-serviço (não é o JWT do usuário).

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/auth/register | Público | Cadastro de usuário |
| POST | /api/auth/login | Público | Login |
| POST | /api/auth/refresh | Público | Renovar token |
| GET | /api/auth/confirm | Público | Confirmar email |
| POST | /api/auth/forgot-password | Público | Solicitar reset de senha |
| POST | /api/auth/reset-password | Público | Resetar senha |
| GET | /api/auth/reset-password | Público | Formulário de reset |
| POST | /api/auth/reset-password-form | Público | Processar reset via form |
| POST | /api/auth/change-password | JWT | Alterar senha |
| GET | /api/users | ADMIN | Listar usuários |
| GET | /api/users/{id} | ADMIN | Buscar usuário |
| PATCH | /api/users/{id}/status | ADMIN | Atualizar status |
| PATCH | /api/users/{id}/role | ADMIN | Atualizar role |
| PATCH | /api/users/{id}/approve | ADMIN | Aprovar cadastro de MERCHANT |
| PATCH | /api/users/{id}/reject | ADMIN | Rejeitar cadastro de MERCHANT |
| GET | /api/auth/login-logs | ADMIN | Listar logs de login |
| POST | /api/keys/generate | JWT | Gerar API Key pessoal |
| GET | /api/keys | JWT | Consultar API Key pessoal |
| DELETE | /api/keys/revoke | JWT | Revogar API Key pessoal |
| GET | /api/keys/validate | X-Internal-Key | Validar API Key (uso interno) |
| PATCH | /api/internal/users/{userId}/merchant | X-Internal-Key | Sincroniza o `merchantId` de um usuário (chamado pelo `ironvault-merchants`) |

## Banco de Dados

```sql
-- Tabelas principais
users              -- inclui coluna merchant_id (uuid, nullable)
refresh_token
email_confirmation_tokens
password_reset_tokens
api_keys
login_log
```

## Claim `merchantId` no JWT

Quando um usuário com `Role.MERCHANT` tem um `MerchantProfile` criado no `ironvault-merchants`, o campo `merchant_id` da tabela `users` é preenchido via o endpoint interno acima. A partir desse momento, todo token gerado (login ou refresh) para esse usuário passa a incluir a claim `merchantId`. Usuários sem merchant vinculado (ex: ADMIN) não recebem essa claim.

Esse fluxo é assíncrono em relação ao cadastro: o usuário precisa gerar um **novo token** (novo login ou refresh) depois que o `MerchantProfile` for criado para que a claim apareça — tokens emitidos antes disso não são retroativamente atualizados.
