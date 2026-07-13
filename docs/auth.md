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
│   │   │   ├── request/
│   │   │   └── response/
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
│   └── UpdateUserRoleService.java
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
PORT=8081
```

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
| GET | /api/auth/login-logs | ADMIN | Listar logs de login |

## Banco de Dados

```sql
-- Tabelas principais
users
refresh_token
email_confirmation_tokens
password_reset_tokens
```