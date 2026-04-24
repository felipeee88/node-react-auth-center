# Auth Center — Gestão de Usuários e Autenticação

## Objetivo
Resolver o problema recorrente de **gestão de usuários e controle de acesso** em aplicações SaaS: cadastro, login, recuperação de senha, perfil do usuário logado e administração de contas por um papel privilegiado (admin). Entrega um núcleo de autenticação reutilizável, com fluxo de **access token + refresh token rotativo**, que pode ser plugado como backbone de identidade em outros produtos.

## Stack
- **Backend:** Node.js 20+, NestJS 10, TypeScript strict, Passport (JWT), class-validator, bcrypt
- **Frontend:** React 18, TypeScript, Vite, Zustand, React Router, React Hook Form, Zod, Axios
- **Banco:** MongoDB + Mongoose
- **Cloud:** Docker / docker-compose (dev); pronto para deploy em qualquer runtime Node (Render, Fly.io, ECS, Azure App Service)
- **Testes:** Jest + Supertest (backend e2e com mongodb-memory-server); Vitest + React Testing Library (frontend)

## Arquitetura
- **Modular Architecture (NestJS):** cada bounded context é um módulo auto-contido (`AuthModule`, `UsersModule`, `RolesModule`), com controller + service + schemas + DTOs próprios.
- **SOLID:** controllers finos, services carregam regra de negócio, repositórios/models isolados, dependências injetadas via DI do Nest.
- **Camadas claras:** `Controller → Service → Model (Mongoose) → MongoDB`, com DTOs validados por `class-validator`.
- **Cross-cutting concerns centralizados:** `LoggerMiddleware` (request-id), `TransformInterceptor` (envelope de resposta), `ValidationPipe` global, `HttpExceptionFilter` global.
- **Segurança:** JWT access de curta duração + refresh rotativo armazenado como hash; `JwtAuthGuard` global e `RolesGuard` para RBAC.

## Funcionalidades
- **Cadastro** de usuário (`POST /auth/register`) com hash bcrypt.
- **Login** com emissão do par access + refresh (`POST /auth/login`).
- **Refresh rotativo** (`POST /auth/refresh`) — invalida o token anterior.
- **Logout** com revogação do refresh atual.
- **Esqueci minha senha / reset** via token de recuperação.
- **Edição** do próprio perfil (`PATCH /me`) e troca de senha (`PATCH /me/password`).
- **Exclusão / desativação** de usuários (admin).
- **Listagem** paginada de usuários com **filtros** (`page`, `size`, `sort`, `role`, `q` por nome/email, `active`).
- **Autenticação** ponta-a-ponta: rotas protegidas no front (`<RequireAuth />`) + guards no back, com refresh transparente via interceptor Axios.
- **RBAC** por papéis (`user`, `admin`) com decorators `@Roles()` e `@CurrentUser()`.

## Como executar
```bash
# 1. Sobe MongoDB
docker-compose up -d

# 2. Backend
cd api
npm install
cp .env.example .env    # defina JWT_SECRET, REFRESH_SECRET, MONGO_URI
npm run start:dev       # http://localhost:3000  (Swagger em /docs)

# 3. Frontend (outro terminal)
cd front
npm install
npm run dev             # http://localhost:5173
```

## Prints
> Adicionar imagens das telas: **Login**, **Cadastro**, **Perfil**, **Listagem de usuários (admin)**.
>
> Sugestão de estrutura: salvar em `docs/screenshots/` e referenciar aqui.

## Aprendizados demonstrados
O projeto prova, na prática:
- **NestJS Modules** — organização por bounded context, `imports`/`exports` explícitos, composição no `AppModule`.
- **JWT** — estratégia `passport-jwt` com segredo e TTL configuráveis por env.
- **Refresh Token rotativo** — persistido como hash no Mongo, revogado e reemitido a cada refresh; logout invalida cadeia.
- **Guards** — `JwtAuthGuard` global + `@Public()` para exceções; `RolesGuard` para RBAC declarativo.
- **Middlewares / Interceptors / Pipes / Filters** — `LoggerMiddleware` com request-id, `TransformInterceptor` de envelope, `ValidationPipe` (whitelist + forbidNonWhitelisted) e `HttpExceptionFilter` padronizando erros.
- **Zustand** — estado global de auth com middleware `persist`; seletores finos para evitar re-render.
- **React Hook Form + Zod** — um schema por tela, validação tipada e imediata no client, duplicando intencionalmente as regras do `class-validator` no back (autoridade final).
- **Axios com refresh transparente** — interceptor reentrante que une 401s concorrentes em uma única Promise de refresh.
- **Testes em camadas** — unit para services/guards, e2e via Supertest contra Mongo efêmero, componentes via RTL.
