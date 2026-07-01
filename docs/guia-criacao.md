# 📖 Guia de Criação do Ecossistema IronVault

Este guia documenta como o ecossistema IronVault foi criado do zero, com decisões de arquitetura e comandos utilizados.

## 📋 Índice

- [Pré-requisitos](#pré-requisitos)
- [Ordem de Criação](#ordem-de-criação)
- [ironvault-auth](#ironvault-auth)
- [ironvault-payments](#ironvault-payments)
- [ironvault-notifications](#ironvault-notifications)
- [ironvault-bff](#ironvault-bff)
- [ironvault-backoffice](#ironvault-backoffice)
- [Infraestrutura](#infraestrutura)

## Pré-requisitos

> ⚠️ **Observação:** As ferramentas e serviços abaixo foram utilizados neste projeto mas podem variar conforme suas necessidades. Sinta-se livre para substituir por alternativas equivalentes.

### Ferramentas de desenvolvimento
- Java 21 (ou versão LTS mais recente)
- Node.js 22+ (ou versão LTS mais recente)
- Maven (ou Gradle)
- Docker (para ambiente local)
- Git

### Serviços de infraestrutura
- **Deploy:** Railway *(alternativas: Render, Fly.io, AWS, GCP)*
- **Banco de dados:** Neon PostgreSQL *(alternativas: PlanetScale, Supabase, RDS)*
- **Cache:** Redis *(pode ser hospedado no Railway ou Redis Cloud)*

### Serviços externos
- **Email:** Resend *(alternativas: SendGrid, Amazon SES, Mailgun)*
- **Pagamentos PIX:** MercadoPago *(alternativas: Pagar.me, PicPay, Gerencianet)*
- **Pagamentos Cartão:** Pagar.me *(alternativas: Stripe, Adyen)*

### Domínio
- Registro de domínio com suporte a CNAME *(registro.br, GoDaddy, Cloudflare, etc)*


## Ordem de Criação

A ordem de criação foi pensada para que cada serviço dependa do anterior:

```
1. ironvault-notifications  ← base de emails
2. ironvault-auth           ← depende do notifications
3. ironvault-payments       ← depende do notifications
4. ironvault-bff            ← depende de auth e payments
5. ironvault-backoffice     ← depende do bff
```

## ironvault-auth

### Criação do projeto
Criado via **Spring Initializr** (start.spring.io) com:
- Java 21
- Spring Boot 3.3.5
- Dependências: Spring Web, Spring Data JPA, Spring Security, PostgreSQL, Validation, Lombok, MapStruct

### Arquitetura
Arquitetura **Hexagonal** (Ports and Adapters):
- `adapter/in` — controllers, filtros de segurança
- `adapter/out` — repositórios JPA, clientes HTTP
- `application` — services (casos de uso)
- `domain` — modelos, ports, enums

### Decisões técnicas
- **JWT com jjwt 0.12.3** — token enriquecido com `jti`, `userId`, `role`, `ip`, `userAgent`
- **Refresh Token** — tabela no banco, 7 dias, revogado no login
- **Rate Limiting** — Bucket4j + Redis, 5 tentativas, bloqueio 15min
- **Email confirmado** — obrigatório para fazer login
- **Comunicação com Notifications** — via HTTP com API Key

## ironvault-payments

### Criação do projeto
Criado via **Spring Initializr** com:
- Java 21
- Spring Boot 3.3.5
- Dependências: Spring Web, Spring Data JPA, Spring Security, PostgreSQL, Validation, Lombok, MapStruct, WebFlux

### Decisões técnicas
- **Idempotência** — `Idempotency-Key` no header para evitar duplicatas
- **Outbox Pattern** — eventos salvos no banco antes de enviar
- **Webhook com retry** — 3 tentativas com backoff exponencial
- **Paginação** — `PageResult<T>` customizado

## ironvault-notifications

### Criação do projeto
Criado via **Spring Initializr** com:
- Java 21
- Spring Boot 3.3.5
- Dependências: Spring Web, Spring Data JPA, PostgreSQL, Validation, Lombok

### Decisões técnicas
- **Autenticação via API Key** — header `X-API-KEY`
- **Templates HTML** — carregados do classpath
- **Resend** — provider de email

## ironvault-bff

### Criação do projeto

```bash
npm init -y
npm install fastify @fastify/jwt @fastify/cors @fastify/rate-limit axios dotenv fastify-plugin
npm install -D typescript @types/node ts-node nodemon
npx tsc --init
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Decisões técnicas
- **Fastify v5** — mais performático que Express
- **fastify-plugin** — necessário para compartilhar decorators entre plugins
- **JWT Middleware** — valida token antes de cada rota protegida
- **Dashboard aggregation** — agrega dados de payments + auth em uma chamada

## ironvault-backoffice

### Criação do projeto

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
npm install axios lucide-react jwt-decode
```

### Decisões técnicas
- **HttpOnly cookies** — tokens nunca expostos ao JavaScript (proteção XSS)
- **Server Components** — páginas de dados buscam no servidor e passam token no header
- **proxy.ts** — Next.js 16 renomeou middleware para proxy
- **Role-based access** — MERCHANT só acessa dashboard, transações e perfil

## Infraestrutura

### Railway
- Deploy automático a partir da branch principal
- Cada serviço tem suas próprias variáveis de ambiente
- Redis compartilhado entre serviços

### Neon (PostgreSQL Serverless)
- 3 bancos separados por serviço
- Scale to zero — economiza quando não há requisições
- Região: sa-east-1 (São Paulo)

### Domínios
Todos os subdomínios configurados via CNAME no registro do domínio:

```
auth.ironvaultpayments.com.br
api.ironvaultpayments.com.br
notifications.ironvaultpayments.com.br
bff.ironvaultpayments.com.br
backoffice.ironvaultpayments.com.br
```