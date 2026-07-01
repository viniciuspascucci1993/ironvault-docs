# ⚡ IronVault Payments

Gateway de pagamentos brasileiro focado em PIX, construído com arquitetura de microsserviços.

## 📋 Índice

- [Sobre o Projeto](#sobre-o-projeto)
- [Arquitetura](#arquitetura)
- [Stack Tecnológica](#stack-tecnológica)
- [Serviços](#serviços)
- [Como Rodar Localmente](#como-rodar-localmente)
- [Deploy](#deploy)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Endpoints](#endpoints)
- [Fluxos Principais](#fluxos-principais)
- [Roadmap](#roadmap)

## 💡 Sobre o Projeto

O IronVault Payments é um ecossistema completo de pagamentos desenvolvido do zero com foco em:

- **Segurança** — JWT, HttpOnly cookies, rate limiting, confirmação de email
- **Escalabilidade** — arquitetura de microsserviços independentes
- **Confiabilidade** — retry automático, idempotência, outbox pattern
- **Developer Experience** — API bem documentada, BFF para o frontend

## 🏗️ Arquitetura

ironvault-backoffice (Next.js 16)
↓
ironvault-bff (Fastify/Node.js)
↓         ↓         ↓
ironvault  ironvault  ironvault
-auth    -payments  -notifications
(Java 21)  (Java 21)   (Java 21)
↓         ↓
Neon      Neon       Resend
PostgreSQL PostgreSQL  (emails)
↓
Redis
(rate limit)

## 🛠️ Stack Tecnológica

### Backend
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- Redis (rate limiting)
- MapStruct + Lombok
- JWT (jjwt 0.12.3)
- MercadoPago (PIX)
- Resend (emails)

### BFF
- Node.js + Fastify v5
- TypeScript
- Axios
- fastify-plugin

### Frontend
- Next.js 16 (App Router)
- TypeScript
- Tailwind CSS
- Lucide React

### Infraestrutura
- Railway (deploy)
- Neon (PostgreSQL serverless)
- Redis (Railway)

## 🌐 Serviços em Produção

| Serviço | URL |
|---------|-----|
| Auth | https://auth.ironvaultpayments.com.br |
| Payments | api.ironvaultpayments.com.br |
| Notifications | https://notifications.ironvaultpayments.com.br |
| BFF | https://bff.ironvaultpayments.com.br |
| Backoffice | https://backoffice.ironvaultpayments.com.br |

## 🚀 Como Rodar Localmente

### Pré-requisitos
- Java 21
- Node.js 22+
- Docker (para PostgreSQL e Redis local)
- Maven

### 1. Clonar os repositórios

```bash
git clone https://github.com/viniciuspascucci1993/ironvault-auth.git
git clone https://github.com/viniciuspascucci1993/ironvault-payments.git
git clone https://github.com/viniciuspascucci1993/ironvault-notifications.git
git clone https://github.com/viniciuspascucci1993/ironvault-bff.git
git clone https://github.com/viniciuspascucci1993/ironvault-backoffice.git

### 2. Subir infraestrutura local (Docker)

```bash
# No diretório do ironvault-auth
docker-compose up -d

### 3. Rodar os serviços Java

```bash
# Auth (porta 8081)
cd ironvault-auth
./mvnw spring-boot:run

# Payments (porta 8080)
cd ironvault-payments
./mvnw spring-boot:run

# Notifications (porta 8082)
cd ironvault-notifications
./mvnw spring-boot:run

### 4. Rodar o BFF

```bash
cd ironvault-bff
npm install
npm run dev
```


### 5. Rodar o Backoffice

```bash
cd ironvault-backoffice
npm install
npm run dev
```

Acessa `http://localhost:3000/login` 🚀

## 🚢 Deploy

Todos os serviços são deployados no **Railway** com deploy automático a partir da branch principal (`main` ou `master`).

### Infraestrutura

| Serviço | Plataforma | Observação |
|---------|-----------|------------|
| PostgreSQL | Neon (serverless) | 3 bancos separados |
| Redis | Railway | Rate limiting |
| Serviços Java | Railway | Deploy via Docker |
| BFF | Railway | Node.js |
| Backoffice | Railway | Next.js |

### Ordem de deploy

1. `ironvault-notifications`
2. `ironvault-auth`
3. `ironvault-payments`
4. `ironvault-bff`
5. `ironvault-backoffice`

### Bancos de dados (Neon)

| Banco | Serviço |
|-------|---------|
| ironvault-auth | Users, tokens, refresh tokens |
| ironvault-payments | Payments, transactions, webhooks |
| ironvault-notifications | Notification events, logs |

## 🔄 Fluxos Principais

### Registro de usuário
```
1. POST /api/auth/register
2. Auth salva usuário no banco
3. Auth envia evento para Notifications
4. Notifications envia email de confirmação
5. Usuário clica no link → GET /api/auth/confirm?token=
6. Auth confirma o email
```

### Login
```
1. POST /api/auth/login
2. Auth valida email e senha
3. Auth verifica rate limiting (Redis)
4. Auth verifica email confirmado
5. Auth retorna accessToken + refreshToken
```

### Pagamento PIX
```
1. POST /api/payments
2. Payments gera PIX via MercadoPago
3. Payments salva no banco com status PROCESSING
4. Payments envia evento para Notifications
5. Notifications envia email com QR Code
6. Webhook MercadoPago → atualiza status (APPROVED/FAILED)
```


### Recuperação de senha
```
1. POST /api/auth/forgot-password
2. Auth gera token de reset
3. Auth envia evento para Notifications
4. Notifications envia email com link
5. Usuário acessa link → POST /api/auth/reset-password
6. Auth atualiza senha
```

> ⚠️ **Observação:** O PIX é gerado em produção via MercadoPago.
> Porém o **webhook** do MercadoPago ainda não está configurado —
> o status é atualizado manualmente via `PATCH /api/payments/{id}/status`
> ou via endpoint mock `POST /api/payments/webhooks/mock`.
> A configuração do webhook real está no roadmap.

## 🗺️ Roadmap

### Em andamento
- [ ] Webhook MercadoPago (atualização automática de status)

### Próximos passos
- [ ] Pagar.me (cartão de crédito) — aguarda abertura de MEI
- [ ] E-commerce de calçados (primeiro cliente real)
- [ ] Logs de auditoria no backoffice
- [ ] Página de configurações no backoffice
- [ ] README de cada serviço

### Futuro
- [ ] App mobile
- [ ] API pública para integração de parceiros
- [ ] Dashboard analytics avançado
- [ ] PCI-DSS compliance
- [ ] Gráficos no dashboard (Recharts)
  - Transações por período
  - Distribuição por status
  - Receita acumulada
  - Novos usuários

### Concluído
- [x] Auth completo (JWT, refresh, rate limit, email confirm, forgot/reset/change password)
- [x] Payments PIX via MercadoPago
- [x] Notifications (templates, Resend)
- [x] BFF (todas as rotas, middleware JWT, CORS, dashboard)
- [x] Backoffice MVP (login, dashboard, transactions, users, profile, route protection, role-based access, token refresh)
- [x] Deploy de todos os serviços no Railway
- [x] Domínios customizados configurados