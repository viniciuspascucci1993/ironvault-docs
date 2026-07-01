# 🖥️ ironvault-backoffice

Painel administrativo do ecossistema IronVault. Consome o BFF para exibir dados e gerenciar o sistema.

## Stack
- Next.js 16 (App Router)
- TypeScript
- Tailwind CSS
- Lucide React
- Axios

## Estrutura de Pastas

```
src/
├── app/
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── transactions/
│   │   │   └── page.tsx
│   │   ├── users/
│   │   │   └── page.tsx
│   │   └── profile/
│   │       └── page.tsx
│   └── api/
│       └── auth/
│           ├── login/
│           │   └── route.ts
│           ├── logout/
│           │   └── route.ts
│           ├── refresh/
│           │   └── route.ts
│           └── change-password/
│               └── route.ts
├── components/
│   ├── layout/
│   │   ├── Sidebar.tsx
│   │   └── Header.tsx
│   └── TokenRefresher.tsx
├── lib/
│   ├── api.ts           ← Axios client-side
│   └── serverApi.ts     ← Axios server-side
├── types/
│   └── index.ts
└── proxy.ts             ← proteção de rotas
```

## Variáveis de Ambiente

```env
BFF_URL=https://bff.ironvaultpayments.com.br
NEXT_PUBLIC_BFF_URL=https://bff.ironvaultpayments.com.br
```

## Como criar o projeto

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

## Instalar dependências

```bash
npm install axios lucide-react jwt-decode
```

## Criar estrutura de pastas

```bash
mkdir -p "src/app/(auth)/login" "src/app/(dashboard)/dashboard" "src/app/(dashboard)/transactions" "src/app/(dashboard)/users" "src/app/(dashboard)/profile" src/components/layout src/lib src/types
mkdir -p "src/app/api/auth/login" "src/app/api/auth/logout" "src/app/api/auth/refresh" "src/app/api/auth/change-password"
```

## Criar arquivos

```bash
touch "src/app/(auth)/login/page.tsx" "src/app/(dashboard)/layout.tsx" "src/app/(dashboard)/dashboard/page.tsx" "src/app/(dashboard)/transactions/page.tsx" "src/app/(dashboard)/users/page.tsx" "src/app/(dashboard)/profile/page.tsx" src/components/layout/Sidebar.tsx src/components/layout/Header.tsx src/components/TokenRefresher.tsx src/lib/api.ts src/lib/serverApi.ts src/types/index.ts src/proxy.ts
touch "src/app/api/auth/login/route.ts" "src/app/api/auth/logout/route.ts" "src/app/api/auth/refresh/route.ts" "src/app/api/auth/change-password/route.ts"
```

## Telas

| Rota | Acesso | Descrição |
|------|--------|-----------|
| /login | Público | Login com email e senha |
| /dashboard | ADMIN, MERCHANT | Resumo geral |
| /transactions | ADMIN, MERCHANT | Listagem de transações |
| /users | ADMIN | Gestão de usuários |
| /profile | ADMIN, MERCHANT | Alterar senha |

## Decisões de Arquitetura

- **HttpOnly cookies** — tokens nunca expostos ao JavaScript
- **Server Components** — páginas de dados buscam direto no servidor
- **Client Components** — apenas páginas com interação (login, perfil)
- **proxy.ts** — proteção de rotas e controle por role
- **Role-based access** — MERCHANT só vê Dashboard, Transações e Perfil