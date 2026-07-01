# 🔀 ironvault-bff

Backend for Frontend do ecossistema IronVault. Camada intermediária entre o backoffice e os serviços Java.

## Stack
- Node.js 22
- Fastify v5
- TypeScript
- Axios
- fastify-plugin

## Estrutura de Pastas

```
src/
├── plugins/
│   └── authenticate.ts       ← middleware JWT
├── routes/
│   ├── auth/
│   │   └── authRoutes.ts
│   ├── payments/
│   │   └── paymentsRoutes.ts
│   ├── users/
│   │   └── usersRoutes.ts
│   └── dashboard/
│       └── dashboardRoutes.ts
├── services/
│   ├── authService.ts
│   ├── paymentsService.ts
│   ├── usersService.ts
│   └── dashboardService.ts
├── types/
│   └── fastify.d.ts
└── server.ts
```

## Variáveis de Ambiente

```env
PORT=3000
JWT_SECRET=
AUTH_SERVICE_URL=https://auth.ironvaultpayments.com.br
PAYMENTS_SERVICE_URL=https://api.ironvaultpayments.com.br
NOTIFICATIONS_SERVICE_URL=https://notifications.ironvaultpayments.com.br
```

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| GET | /health | Público | Health check |
| POST | /api/auth/register | Público | Cadastro |
| POST | /api/auth/login | Público | Login |
| POST | /api/auth/refresh | Público | Renovar token |
| POST | /api/auth/forgot-password | Público | Recuperar senha |
| POST | /api/auth/reset-password | Público | Resetar senha |
| POST | /api/auth/change-password | JWT | Alterar senha |
| GET | /api/payments | JWT | Listar pagamentos |
| GET | /api/payments/:id | JWT | Buscar pagamento |
| PATCH | /api/payments/:id/status | JWT | Atualizar status |
| GET | /api/users | JWT | Listar usuários |
| GET | /api/users/:id | JWT | Buscar usuário |
| PATCH | /api/users/:id/status | JWT | Atualizar status |
| PATCH | /api/users/:id/role | JWT | Atualizar role |
| GET | /api/dashboard/summary | JWT | Resumo dashboard |

## Como criar o projeto

```bash
# Inicializar projeto
npm init -y

# Instalar dependências
npm install fastify @fastify/jwt @fastify/cors @fastify/rate-limit axios dotenv fastify-plugin

# Instalar dependências de desenvolvimento
npm install -D typescript @types/node ts-node nodemon

# Inicializar TypeScript
npx tsc --init
```

## Scripts

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  }
}
```