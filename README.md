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
- **Marketplace/Split payment** — cada lojista recebe direto na própria conta Mercado Pago, com comissão do IronVault retida automaticamente via `application_fee`

## 🏗️ Arquitetura

```
ironvault-backoffice (Next.js 16)
         ↓
ironvault-bff (Fastify/Node.js)
    ↓      ↓      ↓        ↓
ironvault  ironvault  ironvault  ironvault
-auth    -payments  -notif.   -merchants
(Java 21)  (Java 21)  (Java 21)  (Java 21)
    ↓         ↓                    ↓
  Neon      Neon      Resend      Neon
PostgreSQL PostgreSQL (emails)  PostgreSQL
    ↓         ↓
  Redis    Mercado Pago
(rate limit)  (PIX / OAuth)
```

`ironvault-auth` e `ironvault-merchants` trocam informação de forma assíncrona (via chamada interna autenticada por `INTERNAL_API_KEY`) para sincronizar o vínculo `userId ↔ merchantId`, permitindo que o JWT carregue a claim `merchantId` sem acoplar o login a uma chamada em tempo real entre os serviços.

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

| Serviço       | URL                                            |
| ------------- | ---------------------------------------------- |
| Auth          | https://auth.ironvaultpayments.com.br          |
| Payments      | https://api.ironvaultpayments.com.br           |
| Notifications | https://notifications.ironvaultpayments.com.br |
| BFF           | https://bff.ironvaultpayments.com.br           |
| Backoffice    | https://backoffice.ironvaultpayments.com.br    |
| Landing       | https://landing.ironvaultpayments.com.br       |
| Merchants     | https://merchants.ironvaultpayments.com.br     |

## 🚀 Como Rodar Localmente

### Pré-requisitos

- Java 21
- Node.js 22+
- Docker (para PostgreSQL e Redis local)
- Maven

### 1. Clonar os repositórios

````bash
git clone https://github.com/viniciuspascucci1993/ironvault-auth.git
git clone https://github.com/viniciuspascucci1993/ironvault-payments.git
git clone https://github.com/viniciuspascucci1993/ironvault-notifications.git
git clone https://github.com/viniciuspascucci1993/ironvault-merchants.git
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

# Merchants (porta 8083)
cd ironvault-merchants
./mvnw spring-boot:run
````

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

| Serviço       | Plataforma        | Observação         |
| ------------- | ----------------- | ------------------ |
| PostgreSQL    | Neon (serverless) | 3 bancos separados |
| Redis         | Railway           | Rate limiting      |
| Serviços Java | Railway           | Deploy via Docker  |
| BFF           | Railway           | Node.js            |
| Backoffice    | Railway           | Next.js            |

### Ordem de deploy

Ao alterar algo que envolva a integração `merchantId` (auth ↔ merchants ↔ payments), seguir esta ordem para evitar erros transitórios:

1. `ironvault-notifications`
2. `ironvault-auth`
3. `ironvault-merchants`
4. `ironvault-payments`
5. `ironvault-bff`
6. `ironvault-backoffice`

### Bancos de dados (Neon)

| Banco                   | Serviço                                               |
| ----------------------- | ----------------------------------------------------- |
| ironvault-auth          | Users, tokens, refresh tokens, api keys               |
| ironvault-payments      | Payments, transactions, webhooks                      |
| ironvault-notifications | Notification events, logs                             |
| ironvault-merchants     | Merchant profiles, gateway credentials (Mercado Pago) |

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
   (accessToken inclui merchantId se o usuário tiver um MerchantProfile vinculado)
```

### Cadastro de lojista e conexão Mercado Pago (split payment)

```
1. Lojista se cadastra na landing → BFF cria o User (PENDING) e já cria o
   MerchantProfile no ironvault-merchants na mesma requisição (gera o merchantId)
2. ironvault-merchants sincroniza o merchantId de volta ao ironvault-auth
3. Admin aprova o lojista no backoffice
4. Lojista redefine a senha (link recebido por email no cadastro) e loga
   → JWT já vem com a claim merchantId
5. Lojista conecta a própria conta Mercado Pago via OAuth
   (authorize-url → autorização → callback salva o access_token do lojista)
```

### Pagamento PIX (com split)
```
1. POST /api/payments (JWT do lojista logado)
2. Payments extrai o merchantId da claim do token e vincula ao Payment/Transaction
3. Payments salva com status PROCESSING (outbox pattern)
4. Scheduler assíncrono (a cada 500ms) dispara o processamento:
   - Busca o access_token do lojista no ironvault-merchants
   - Calcula a comissão do IronVault (`application_fee`, percentual configurável)
   - Cria o PIX no Mercado Pago autenticado como o lojista
5. Dinheiro cai direto na conta do lojista; comissão cai automaticamente
   na conta do IronVault
6. Payments envia evento para Notifications
7. Notifications envia email com QR Code
8. Webhook MercadoPago → atualiza status (APPROVED/FAILED)
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

## 🗺️ Roadmap

### Em andamento

- [ ] Botão "Conectar Mercado Pago" no backoffice (frontend)
- [ ] Endpoint de status de conexão MP (`GET /status/{merchantId}`) e redirect amigável no callback OAuth

### Próximos passos

- [ ] Pagar.me (cartão de crédito) — aguarda abertura de MEI
- [ ] E-commerce de calçados (primeiro cliente real)
- [ ] Migração de `ddl-auto: update` para Flyway (controle de schema versionado)
- [ ] CNPJ/regularização formal do IronVault antes de operar split com volume real

### Futuro

- [ ] App mobile
- [ ] API pública para integração de parceiros
- [ ] Dashboard analytics avançado por MERCHANT
- [ ] Webhook configurável por MERCHANT
- [ ] PCI-DSS compliance
- [ ] Landing page institucional

### Concluído

- [x] Auth completo (JWT, refresh, rate limit, email confirm, forgot/reset/change password)
- [x] Payments PIX via MercadoPago
- [x] Webhook MercadoPago implementado
- [x] Notifications (templates, Resend)
- [x] BFF (todas as rotas, middleware JWT, CORS, dashboard)
- [x] Backoffice MVP (login, dashboard, transactions, users, profile, route protection, role-based access, token refresh)
- [x] Deploy de todos os serviços no Railway
- [x] Domínios customizados configurados
- [x] Cadastro de novos usuários (MERCHANT) pelo backoffice com senha temporária
- [x] Gráficos no dashboard (Recharts)
  - Transações por período
  - Distribuição por status
  - Receita acumulada
- [x] Link "Esqueceu a senha?" na tela de login
- [x] Log de login (usuário, data/hora, IP)
- [x] Teste real do webhook MercadoPago em produção — status atualizado automaticamente após pagamento
- [x] Página de configurações no backoffice
- [x] API Key por MERCHANT para integração
- [x] Fluxo de aprovação de MERCHANTs no backoffice
- [x] Email de notificação ao aprovar/rejeitar MERCHANT
- [x] Fluxo simplificado de cadastro de MERCHANT (email confirmado automaticamente)
- [x] Landing page com formulário de cadastro de parceiros
- [x] Validação de conta ativa no login (MERCHANT pendente bloqueado)
- [x] Toast de feedback no login
- [x] Melhorias no dashboard por role (MERCHANT não vê Total de Usuários)
- [x] Seed automático do ADMIN via variáveis de ambiente
- [x] Serviço ironvault-merchants criado
- [x] Perfil do MERCHANT visível no backoffice para aprovação
- [x] Fluxo OAuth de conexão da conta Mercado Pago do lojista (authorize-url, callback, persistência do access_token/refresh_token)
- [x] `merchantId` propagado em Payment e Transaction, com ownership check (dono ou ADMIN) em todos os endpoints de consulta
- [x] Sincronização assíncrona `merchantId` entre ironvault-merchants e ironvault-auth, com claim `merchantId` no JWT
- [x] Split de pagamento implementado: ironvault-payments usa o access_token
      do lojista + application_fee (comissão configurável) na criação do PIX
