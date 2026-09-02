# 🛍️ ironvault-store

Serviço de e-commerce do ecossistema IronVault. Responsável pelo catálogo de produtos dos lojistas e pelo checkout de convidado, orquestrando a criação do pagamento no `ironvault-payments`.

Diferente dos demais serviços do IronVault, o `ironvault-store` é um **monólito modular**: um único processo/deploy, mas organizado internamente em domínios independentes (`product`, `customer`, `address`, `order`, `checkout`), cada um seguindo o mesmo padrão hexagonal (`domain` → `application` → `adapter`) usado nos outros serviços. Essa decisão evita o custo operacional de manter múltiplos microsserviços num estágio do projeto onde isso ainda não é necessário, sem abrir mão da separação de responsabilidades — cada domínio já nasce como uma unidade isolada, pronta para ser extraída como serviço próprio caso o volume um dia justifique.

## Stack
- Java 21 + Spring Boot 3.3.5
- PostgreSQL (Neon)
- Spring Security + JWT (jjwt 0.12.3) — reaproveita o mesmo `JWT_SECRET` dos demais serviços
- RestTemplate — para chamadas internas ao `ironvault-payments`

## Estrutura de Pastas

```
src/main/java/com/ironvault/store/
├── config/
│   ├── security/
│   │   ├── SecurityConfig.java
│   │   ├── JwtAuthFilter.java
│   │   └── JwtTokenValidator.java
│   ├── RestTemplateConfig.java
│   └── OpenApiConfig.java
├── product/
│   ├── adapter/
│   │   ├── in/{web, dto, mapper}
│   │   └── out/{entity, persistence}
│   ├── application/
│   └── domain/{model, port/in, port/out}
├── customer/
│   ├── adapter/out/{entity, persistence}
│   ├── application/
│   └── domain/{model, port/in, port/out}
├── address/
│   ├── adapter/out/{entity, persistence}
│   ├── application/
│   └── domain/{model, port/in, port/out}
├── order/
│   ├── adapter/out/{entity, persistence}
│   ├── application/
│   └── domain/{model, port/in, port/out, enums}
└── checkout/
    ├── adapter/
    │   ├── in/{web, dto}
    │   └── out/ (PaymentsClient)
    └── application/ (CheckoutService)
```

`customer` e `address` não possuem Controller próprio — são usados internamente pelo `checkout`, já que o cliente final nunca os acessa diretamente (ver seção de fluxos). `checkout` não possui uma interface `UseCase` própria: como só existe uma implementação e ele já representa o topo da orquestração, criar uma porta ali não agregaria isolamento real.

## Variáveis de Ambiente

```env
DB_URL=jdbc:postgresql://<host>/ironvault-store?sslmode=require
DB_USERNAME=
DB_PASSWORD=
JWT_SECRET=
PAYMENTS_URL=https://api.ironvaultpayments.com.br
INTERNAL_API_KEY=
PORT=8084
```

`JWT_SECRET` e `INTERNAL_API_KEY` são os mesmos valores compartilhados entre `auth`, `payments` e `merchants`.

## Endpoints

| Método | Rota | Autenticação | Descrição |
|--------|------|-------------|-----------|
| POST | /api/products | JWT (lojista) | Cria um produto com suas variantes (tamanho, preço e estoque independentes) |
| GET | /api/products | JWT (lojista) | Lista os produtos do lojista logado |
| GET | /api/products/{id} | JWT (lojista)* | Busca um produto por id |
| POST | /api/checkout | Público | Fluxo completo de checkout do cliente final (ver fluxo abaixo) |

\* `GET /api/products/{id}` ainda exige JWT hoje, mas é o endpoint que o `storefront` público vai precisar consumir sem autenticação — liberar essa rota é uma pendência conhecida (ver Roadmap no README principal).

## Banco de Dados

```sql
products               -- catálogo (nome, descrição, imagem, ativo), vinculado a merchant_id
product_variants        -- tamanho, preço e estoque por variante, vinculado a product_id
customers                -- criado automaticamente no checkout, único por (merchant_id, email)
addresses                -- endereço de entrega, vinculado a customer_id
orders                   -- pedido: status, total, payment_id (referência ao ironvault-payments)
order_items               -- itens do pedido, com preço e nome do produto copiados no momento da compra
```

Nenhuma tabela usa relacionamentos JPA (`@OneToMany`/`@ManyToOne`) — todo vínculo entre tabelas é feito via `UUID` simples e resolvido explicitamente nos repositórios, seguindo o mesmo padrão adotado nos demais serviços do IronVault (evita N+1 queries, lazy loading exceptions e acoplamento do domínio a detalhes de persistência).

## Fluxos principais

### Cadastro de produto (lojista)

```
1. Lojista logado (JWT) chama POST /api/products
2. merchantId é extraído da claim do token (mesmo padrão do ironvault-payments)
3. CreateProductService cria o Product e, na mesma transação, as ProductVariant
   informadas (uma por tamanho, cada uma com seu próprio preço e estoque)
```

### Checkout (cliente final, sem login)

```
1. Cliente final monta o carrinho no storefront (ainda não implementado)
2. POST /api/checkout, sem JWT, com: dados do cliente, endereço e itens
   (productVariantId + quantity — nunca o preço, que é sempre buscado no servidor)
3. FindOrCreateCustomerUseCase busca o Customer pelo par (merchantId, email);
   cria um novo, sem senha, caso não exista
4. CreateAddressUseCase salva o endereço vinculado a esse Customer
5. CreateOrderUseCase valida o estoque de cada ProductVariant, busca o preço
   real e o nome do produto, calcula o total e cria o Order + OrderItem
6. PaymentsClient chama POST /api/internal/payments no ironvault-payments
   (autenticado via INTERNAL_API_KEY, com Idempotency-Key derivada do orderId
   para evitar cobrança duplicada em caso de retry)
7. UpdateOrderPaymentUseCase vincula o paymentId retornado ao Order
8. O cliente recebe: orderId, totalAmount, paymentId, status, QR code e
   código Pix copia-e-cola — prontos para exibição numa tela de pagamento
```

O checkout é hoje inteiramente "de convidado": não existe cadastro de senha nem login para o cliente final. Se ele comprar novamente com o mesmo e-mail, o `Customer` existente é reaproveisado automaticamente (endereço incluso), sem duplicar o cadastro. Uma eventual opção de criar login *depois* da primeira compra está mapeada como melhoria futura, mas não bloqueia o fluxo atual.

## Decisões de arquitetura

- **Monólito modular, não microsserviços separados**: `product`, `customer`, `address`, `order` e `checkout` vivem no mesmo deploy, mas cada um é uma fatia hexagonal isolada — caso um domínio precise escalar independentemente no futuro, a extração para um serviço próprio é apenas mover a pasta, sem reescrever a lógica de negócio.
- **Preço nunca confia no cliente**: todo valor cobrado no checkout é buscado do `ProductVariant` persistido no momento da criação do pedido, nunca do payload recebido.
- **Split payment reaproveitado, não duplicado**: o `checkout` não conhece nada de Mercado Pago, `application_fee` ou tokens de vendedor — ele só chama o endpoint interno do `ironvault-payments`, que já resolve todo o split (ver README do `ironvault-payments`).