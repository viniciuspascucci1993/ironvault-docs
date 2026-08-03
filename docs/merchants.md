# 🏪 ironvault-merchants

Serviço responsável pelo cadastro de lojistas (merchants) e pela conexão das contas Mercado Pago deles, habilitando o modelo de **split payment** do IronVault (venda cai direto na conta do lojista, com comissão retida automaticamente para o IronVault via `application_fee`).

## Stack
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- RestTemplate (chamadas HTTP para Mercado Pago e ironvault-auth)

## Estrutura de Pastas

```
src/main/java/com/ironvault/merchants/
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   ├── MerchantProfileController.java
│   │   │   ├── MercadoPagoOAuthController.java
│   │   │   ├── request/
│   │   │   └── response/
│   │   └── exceptions/
│   │       └── GlobalExceptionHandler.java
│   └── out/
│       ├── auth/
│       │   └── AuthClient.java
│       ├── mercadopago/
│       │   └── MercadoPagoOAuthAdapter.java
│       ├── entity/
│       │   ├── MerchantProfileEntity.java
│       │   └── MerchantGatewayCredentialsEntity.java
│       └── persistence/
├── application/
│   ├── CreateMerchantProfileService.java
│   ├── GetMerchantProfileService.java
│   └── ConnectMerchantMercadoPagoAccountService.java
├── config/
│   └── RestTemplateConfig.java
└── domain/
    ├── model/
    │   ├── MerchantProfile.java
    │   └── MerchantGatewayCredentials.java
    └── port/
        ├── in/
        └── out/
```

## Variáveis de Ambiente

```env
DB_URL=jdbc:postgresql://<host>/ironvault-merchants?sslmode=require
DB_USERNAME=
DB_PASSWORD=
AUTH_URL=https://auth.ironvaultpayments.com.br
INTERNAL_API_KEY=
MP_CLIENT_ID=
MP_CLIENT_SECRET=
MP_REDIRECT_URI=https://merchants.ironvaultpayments.com.br/api/merchants/mercadopago/callback
PORT=8083
```

`AUTH_URL` + `INTERNAL_API_KEY` são usados pelo `AuthClient` para sincronizar o `merchantId` de volta para o `ironvault-auth` (ver seção abaixo). `INTERNAL_API_KEY` é a mesma chave compartilhada entre `auth`, `payments` e `merchants`.

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/merchants/profile | X-User-Id (via BFF) | Cria o perfil do lojista (nome do negócio, CPF/CNPJ, segmento, telefone, site) |
| GET | /api/merchants/profile | X-User-Id (via BFF) | Consulta o perfil do lojista logado |
| GET | /api/merchants/mercadopago/authorize-url/{merchantId} | — | Gera a URL de autorização OAuth do Mercado Pago para aquele merchant |
| GET | /api/merchants/mercadopago/callback | — | Callback do OAuth; troca o `code` pelo `access_token` e salva as credenciais |

## Banco de Dados

```sql
merchant_profiles              -- dados de negócio do lojista (id = merchantId)
merchant_gateway_credentials   -- access_token/refresh_token da conta MP do lojista, vinculado a merchant_id
```

## Fluxos principais

### 1. Cadastro do perfil do lojista

```
1. Lojista já aprovado (ironvault-auth) loga no backoffice
2. Backoffice → ironvault-bff → POST /api/merchants/profile (X-User-Id)
3. CreateMerchantProfileService cria o MerchantProfile (gera o merchantId)
4. AuthClient.updateUserMerchantId() sincroniza o merchantId de volta
   para o ironvault-auth (PATCH /api/internal/users/{userId}/merchant)
   → fire-and-forget: se falhar, apenas loga o erro, não bloqueia o cadastro
```

> A partir desse momento, o próximo login/refresh do lojista no `ironvault-auth` já emite um JWT com a claim `merchantId`.

### 2. Conexão da conta Mercado Pago do lojista (OAuth)

```
1. Backoffice chama GET /api/merchants/mercadopago/authorize-url/{merchantId}
2. Lojista é redirecionado para o Mercado Pago e autoriza o acesso
3. Mercado Pago redireciona para /api/merchants/mercadopago/callback?code=...&state={merchantId}
4. ConnectMerchantMercadoPagoAccountService troca o code pelo access_token
   (MercadoPagoOAuthAdapter) e salva em MerchantGatewayCredentials
```

O parâmetro `state` carrega o `merchantId` para vincular corretamente o token retornado à conta certa, já que o Mercado Pago não informa isso por si só.

### 3. Uso do token na hora da venda

O `ironvault-payments` consulta o `access_token` salvo aqui (via o `merchantId` do pagamento) para criar o PIX diretamente na conta do lojista, incluindo `application_fee` como comissão do IronVault. *(Integração ainda pendente — ver Roadmap no README principal.)*
