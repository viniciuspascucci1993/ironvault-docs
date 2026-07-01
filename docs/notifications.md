# 🔔 ironvault-notifications

Serviço de notificações do ecossistema IronVault. Responsável por enviar emails via Resend.

## Stack
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- Resend (envio de emails)

## Estrutura de Pastas

```
src/main/java/com/ironvault/notifications/
├── adapter/
│   ├── in/
│   │   └── web/
│   │       └── NotificationController.java
│   └── out/
│       ├── entity/
│       └── persistence/
├── application/
│   ├── NotificationEventService.java
│   └── TemplateNotificationService.java
└── domain/
    ├── model/
    ├── port/
    │   ├── in/
    │   └── out/
    └── enums/
        ├── NotificationEventType.java
        └── EmailTemplate.java
```

## Variáveis de Ambiente

```env
DB_URL=jdbc:postgresql://<host>/ironvault-notifications?sslmode=require
DB_USERNAME=
DB_PASSWORD=
RESEND_API_KEY=
RESEND_FROM_EMAIL=noreply@ironvaultpayments.com.br
NOTIFICATIONS_API_KEY=
PORT=8082
```

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/notifications/events | API Key | Receber evento |

## Tipos de Eventos

| Evento | Descrição | Template |
|--------|-----------|----------|
| USER_REGISTERED | Novo usuário cadastrado | welcome.html |
| EMAIL_CONFIRMATION | Confirmação de email | email-confirmation.html |
| PASSWORD_RESET | Reset de senha | password-reset.html |
| PIX_GENERATED | PIX gerado | pix-generated.html |
| PAYMENT_APPROVED | Pagamento aprovado | payment-approved.html |
| PAYMENT_FAILED | Pagamento falhou | payment-failed.html |

## Banco de Dados

```sql
-- Tabelas principais
notification_events
notification_logs
```

## Observações

> A autenticação é feita via `X-API-KEY` no header.
> A chave deve ser a mesma configurada em `NOTIFICATIONS_API_KEY`.