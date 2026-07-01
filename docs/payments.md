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
│   │   │   ├── PaymentWebhookController.java
│   │   │   ├── dto/
│   │   │   └── mapper/
│   │   └── security/
│   └── out/
│       ├── entity/
│       ├── gateway/
│       └── persistence/
├── application/
│   ├── CreatePaymentService.java
│   ├── GetAllPaymentsService.java
│   ├── GetPaymentByIdService.java
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
MERCADOPAGO_ACCESS_TOKEN=
NOTIFICATIONS_URL=https://notifications.ironvaultpayments.com.br
NOTIFICATIONS_API_KEY=
PORT=8080
```

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/payments | JWT | Criar pagamento PIX |
| GET | /api/payments | JWT | Listar pagamentos |
| GET | /api/payments/{id} | JWT | Buscar pagamento |
| PATCH | /api/payments/{id}/status | ADMIN | Atualizar status |
| POST | /api/payments/webhooks/mock | HMAC | Webhook mock |
| POST | /api/payments/webhooks/mercadopago | Público | Webhook MercadoPago |

## Banco de Dados

```sql
-- Tabelas principais
payments
transactions
idempotency
outbox_events
webhook_delivery_attempts
```