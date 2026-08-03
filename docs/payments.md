# 💳 ironvault-payments

Serviço de pagamentos do ecossistema IronVault. Responsável por processar pagamentos PIX via MercadoPago.

## Stack
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- MercadoPago SDK
- WebFlux (chamadas reativas)

## Estrutura de Pastas

```
src/main/java/com/ironvault/payments/
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   ├── PaymentController.java
│   │   │   ├── TransactionController.java
│   │   │   ├── PaymentWebhookController.java
│   │   │   ├── dto/
│   │   │   └── mapper/
│   │   ├── common/
│   │   │   └── GlobalExceptionHandler.java
│   │   └── security/
│   │       ├── JwtAuthFilter.java
│   │       └── JwtTokenValidator.java
│   └── out/
│       ├── entity/
│       ├── gateway/
│       └── persistence/
│           └── specification/
│               ├── PaymentSpecification.java
│               └── TransactionSpecification.java
├── application/
│   ├── CreatePaymentService.java
│   ├── GetAllPaymentsService.java
│   ├── GetPaymentByIdService.java
│   ├── GetAllTransactionsService.java
│   ├── GetTransactionsByPaymentService.java
│   ├── UpdatePaymentStatusService.java
│   ├── WebhookProcessingService.java
│   └── WebhookRetryService.java
└── domain/
    ├── model/
    ├── port/
    │   ├── in/
    │   └── out/
    ├── enums/
    └── query/
        ├── PageQuery.java
        ├── PageResult.java
        └── PaymentFilter.java
```

## Variáveis de Ambiente

```env
DB_URL=jdbc:postgresql://<host>/ironvault-payments?sslmode=require
DB_USERNAME=
DB_PASSWORD=
MERCADO_PAGO_ACCESS_TOKEN=
JWT_SECRET=
WEBHOOK_SECRET=
NOTIFICATIONS_URL=https://notifications.ironvaultpayments.com.br
NOTIFICATIONS_API_KEY=
AUTH_URL=https://auth.ironvaultpayments.com.br
INTERNAL_API_KEY=
PORT=8080
```

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/payments | JWT | Criar pagamento PIX (merchantId extraído do token) |
| GET | /api/payments | JWT | Listar pagamentos (filtrado por merchantId; ADMIN vê todos) |
| GET | /api/payments/{id} | JWT | Buscar pagamento (apenas dono ou ADMIN) |
| PATCH | /api/payments/{id}/status | ADMIN | Atualizar status |
| GET | /api/payments/{id}/transactions | JWT | Listar transações de um pagamento (apenas dono ou ADMIN) |
| GET | /api/transactions | JWT | Histórico de transações (filtrado por merchantId; ADMIN vê todos) |
| POST | /api/payments/webhooks/mock | HMAC | Webhook mock |
| POST | /api/payments/webhooks/mercadopago | Público | Webhook MercadoPago |

## Banco de Dados

```sql
-- Tabelas principais
payments           -- inclui coluna merchant_id (uuid, not null)
transactions        -- inclui coluna merchant_id (uuid, not null)
idempotency
outbox_events
webhook_delivery_attempts
```

## Ownership por `merchantId`

Todo `Payment` e `Transaction` pertence a um `merchantId`, extraído da claim `merchantId` do JWT no momento da criação (não é enviado no corpo da requisição, por segurança). Isso garante:

- Um lojista só consegue **criar** pagamentos em seu próprio nome.
- Um lojista só consegue **consultar** (`GET /{id}`, listagens) os próprios registros — tentativa de acessar registro de outro merchant retorna `403 Forbidden` (`PaymentAccessDeniedException`).
- Um usuário `ADMIN` não sofre esse filtro e enxerga todos os registros.

A filtragem dinâmica usa `JpaSpecificationExecutor` (`PaymentSpecification` / `TransactionSpecification`), permitindo combinar `merchantId` com os demais filtros (status, moeda, valor, tipo) sem duplicar métodos de repositório.

> ⚠️ Pré-requisito: o token JWT precisa carregar a claim `merchantId`, o que só ocorre depois que o `ironvault-merchants` sincroniza o vínculo com o `ironvault-auth` (ver README do `ironvault-auth`). Sem essa claim, todo endpoint que exige dono retorna `403` para não-ADMIN.
