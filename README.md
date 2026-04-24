# node-react-auth-center

Sistema de **autenticação e gestão de usuários** com cadastro, login, perfil e permissões baseadas em papéis. API em **NestJS** seguindo **Modular Architecture + SOLID** com **JWT + Refresh Token**, consumida por um frontend **React + TypeScript** com **Zustand**, **React Hook Form** e **Zod**.

## Stack

### Backend

- **Node.js 20+** · **NestJS 10** · **TypeScript** strict
- **MongoDB** com **Mongoose** ODM
- **JWT** (access token de curta duração) + **Refresh Token** rotativo
- **Passport** (`passport-jwt`) para estratégias de autenticação
- **class-validator** + **class-transformer** para validação de DTOs
- **bcrypt** para hash de senhas
- **Jest** + **Supertest** para testes

### Frontend

- **React 18** + **TypeScript** (strict)
- **Vite** — build e dev server
- **Zustand** — estado global (auth, usuário corrente)
- **React Router** — roteamento com rotas protegidas
- **React Hook Form** + **Zod** — formulários tipados e validação
- **Axios** — cliente HTTP com interceptors (JWT + refresh automático)
- **Vitest** + **React Testing Library** — testes

## Pré-requisitos

- **Node.js 20+** e **npm** (ou pnpm)
- **Docker Desktop** — recomendado para MongoDB em dev
- **Nest CLI** (opcional, útil para scaffold): `npm i -g @nestjs/cli`

## Estrutura do repositório

```
node-react-auth-center/
├── api/                              (app NestJS)
│   └── src/
│       ├── modules/
│       │   ├── auth/                 (login, register, refresh, logout)
│       │   │   ├── guards/           (JwtAuthGuard, RolesGuard)
│       │   │   ├── strategies/       (jwt.strategy.ts, refresh.strategy.ts)
│       │   │   ├── decorators/       (@Roles, @CurrentUser, @Public)
│       │   │   ├── dto/
│       │   │   ├── auth.controller.ts
│       │   │   ├── auth.service.ts
│       │   │   └── auth.module.ts
│       │   ├── users/                (CRUD de usuários, perfil)
│       │   │   ├── schemas/          (user.schema.ts)
│       │   │   ├── dto/
│       │   │   ├── users.controller.ts
│       │   │   ├── users.service.ts
│       │   │   └── users.module.ts
│       │   └── roles/                (papéis e permissões)
│       ├── common/
│       │   ├── middlewares/          (logger, request-id)
│       │   ├── filters/              (HttpExceptionFilter global)
│       │   ├── interceptors/         (TransformInterceptor — envelope)
│       │   └── pipes/                (ValidationPipe customizado)
│       ├── config/                   (env validation, mongoose, jwt)
│       ├── app.module.ts
│       └── main.ts
└── front/                            (app React + Vite)
    └── src/
        ├── api/                      (axios client, chamadas por recurso)
        ├── components/               (componentes reutilizáveis)
        ├── features/
        │   ├── auth/                 (login, register, forgot-password)
        │   ├── profile/              (perfil do usuário logado)
        │   └── users/                (gestão de usuários — admin)
        ├── stores/                   (Zustand — authStore, userStore)
        ├── hooks/                    (useAuth, useCurrentUser)
        ├── routes/                   (React Router + rotas protegidas)
        ├── schemas/                  (schemas Zod — login, register, etc.)
        └── types/                    (tipos TypeScript compartilhados)
```

## Executando o backend

A partir da raiz do repositório:

```bash
# 1. Subir MongoDB
docker-compose up -d authcenter.database

# 2. Instalar dependências
cd api
npm install

# 3. Configurar variáveis de ambiente
cp .env.example .env
# edite .env com os segredos (JWT_SECRET, REFRESH_SECRET, MONGO_URI)

# 4. Rodar em dev
npm run start:dev
```

Endpoints ao subir:

- Health: http://localhost:3000/health
- Swagger UI: http://localhost:3000/docs

### Variáveis de ambiente (`api/.env`)

| Variável | Exemplo | Descrição |
|---|---|---|
| `PORT` | `3000` | Porta HTTP |
| `MONGO_URI` | `mongodb://authcenter:authcenter@localhost:27017/authcenter?authSource=admin` | Connection string |
| `JWT_SECRET` | `<hex 64>` | Segredo do access token |
| `JWT_EXPIRES_IN` | `15m` | TTL do access token |
| `REFRESH_SECRET` | `<hex 64>` | Segredo do refresh token |
| `REFRESH_EXPIRES_IN` | `7d` | TTL do refresh token |
| `BCRYPT_ROUNDS` | `12` | Rounds de hash de senha |

## Executando o frontend

```bash
cd front
npm install
npm run dev
```

App disponível em http://localhost:5173. A URL da API fica em `front/.env` (ex.: `VITE_API_URL=http://localhost:3000`).

## Autenticação

Fluxo **access + refresh token** com rotação:

1. `POST /auth/register` cria o usuário (senha com bcrypt).
2. `POST /auth/login` retorna `{ accessToken, refreshToken, user }`.
3. Requisições protegidas usam `Authorization: Bearer {accessToken}`.
4. Quando o access expira (401), o front chama `POST /auth/refresh` com o refresh token atual; recebe um novo par. O refresh anterior é invalidado (rotação).
5. `POST /auth/logout` revoga o refresh token atual.

No frontend:

- **Zustand** guarda `accessToken`, `refreshToken` e `user` (persistidos em `localStorage` via middleware `persist`).
- Um **interceptor Axios** injeta `Authorization` e, ao receber 401, tenta um refresh transparente antes de repetir a requisição original. Se o refresh falhar, limpa o estado e redireciona para `/login`.

### Registrar

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Felipe",
    "email": "felipe@authcenter.test",
    "password": "Senha@123"
  }'
```

### Login

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"felipe@authcenter.test","password":"Senha@123"}'
```

Resposta:

```json
{
  "data": {
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "eyJhbGciOi...",
    "user": {
      "id": "6620...",
      "name": "Felipe",
      "email": "felipe@authcenter.test",
      "roles": ["user"]
    }
  },
  "success": true
}
```

### Refresh

```bash
curl -X POST http://localhost:3000/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"eyJhbGciOi..."}'
```

### Logout

```bash
curl -X POST http://localhost:3000/auth/logout \
  -H "Authorization: Bearer $ACCESS"
```

## Permissões

Modelo baseado em **papéis**:

- `user` — usuário comum; pode gerenciar o próprio perfil.
- `admin` — pode listar, criar, atualizar e desativar outros usuários.

Aplicação:

- **`JwtAuthGuard`** — guarda global; rotas públicas usam o decorator `@Public()`.
- **`RolesGuard`** + **`@Roles('admin')`** — restringe endpoints a papéis específicos.
- **`@CurrentUser()`** — decorator de parâmetro que injeta o usuário autenticado no handler.

Exemplo:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get()
list(@CurrentUser() user: UserDoc) {
  return this.usersService.findAll();
}
```

## API — endpoints principais

### Auth

| Método + Rota | Público? | Descrição |
|---|---|---|
| `POST /auth/register` | sim | Cria novo usuário |
| `POST /auth/login` | sim | Retorna par access + refresh |
| `POST /auth/refresh` | sim | Rotaciona o par de tokens |
| `POST /auth/logout` | auth | Revoga o refresh token atual |
| `POST /auth/forgot-password` | sim | Gera token de recuperação (placeholder de e-mail em dev) |
| `POST /auth/reset-password` | sim | Consome o token e redefine a senha |

### Perfil (usuário logado)

| Método + Rota | Status |
|---|---|
| `GET /me` | 200 / 401 |
| `PATCH /me` | 200 / 400 / 401 |
| `PATCH /me/password` | 204 / 400 / 401 |

### Usuários (admin)

| Método + Rota | Status |
|---|---|
| `GET /users` | 200 / 401 / 403 |
| `GET /users/{id}` | 200 / 401 / 403 / 404 |
| `POST /users` | 201 / 400 / 401 / 403 |
| `PATCH /users/{id}` | 200 / 400 / 401 / 403 / 404 |
| `PATCH /users/{id}/roles` | 200 / 400 / 401 / 403 / 404 |
| `DELETE /users/{id}` | 204 / 401 / 403 / 404 |

### Listagem com filtros

```bash
curl "http://localhost:3000/users?page=1&size=20&sort=createdAt:desc&role=admin&q=felipe" \
  -H "Authorization: Bearer $ACCESS"
```

Query params: `page`, `size`, `sort`, `role`, `q` (busca por nome/email, case-insensitive), `active`.

## Tratamento de erros

Um `HttpExceptionFilter` global padroniza a resposta de erros:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email must be an email",
    "details": [ { "field": "email", "message": "must be an email" } ]
  },
  "timestamp": "2026-04-24T12:34:56Z",
  "path": "/auth/register"
}
```

| Exceção | HTTP | `code` |
|---|---|---|
| `BadRequestException` (validação) | 400 | `VALIDATION_ERROR` |
| `UnauthorizedException` | 401 | `UNAUTHORIZED` |
| `ForbiddenException` | 403 | `FORBIDDEN` |
| `NotFoundException` | 404 | `NOT_FOUND` |
| `ConflictException` | 409 | `CONFLICT` |
| Demais | 500 | `INTERNAL_ERROR` |

Stack traces nunca vão na resposta; são logados com request-id correlacionável.

## Arquitetura

### Modular (NestJS)

```
AppModule
├── ConfigModule          (validação de env com Joi)
├── MongooseModule        (conexão MongoDB)
├── AuthModule            (estratégias JWT, guards, login/refresh)
├── UsersModule           (CRUD, perfil, permissões)
├── RolesModule           (definição de papéis e permissões)
└── CommonModule          (filtros, interceptors, pipes globais)
```

Cada módulo é **auto-contido**: controller + service + schemas + DTOs + guards específicos. Dependências entre módulos são explícitas via `imports` e `exports`.

### Camadas por módulo

```
Controller ──▶ Service ──▶ Repository/Model (Mongoose) ──▶ MongoDB
     │
     └──▶ DTO validation (class-validator)
```

- **Controllers** são finos — delegam para o service.
- **Services** carregam a regra de negócio e orquestram chamadas ao modelo.
- **Schemas Mongoose** ficam em `modules/<m>/schemas/` e não vazam para controllers.

### Middlewares e interceptors

- **`LoggerMiddleware`** — loga método, URL, status e tempo de resposta por request, com `request-id` gerado.
- **`TransformInterceptor`** — envelopa toda resposta bem-sucedida em `{ data, success }`.
- **`ValidationPipe`** global — transforma e valida DTOs (whitelist + forbidNonWhitelisted).
- **`HttpExceptionFilter`** global — padroniza erros como descrito acima.

### Fluxo de login

```
POST /auth/login
    │
    ▼
AuthController.login(LoginDto)
    │
    ▼
AuthService.validateUser(email, password)
    │      └──▶ UsersService.findByEmail ──▶ bcrypt.compare
    ▼
AuthService.issueTokens(user)
    │      ├──▶ JwtService.sign(access, JWT_SECRET, 15m)
    │      └──▶ JwtService.sign(refresh, REFRESH_SECRET, 7d)
    │           └──▶ grava refresh hash em RefreshTokenModel
    ▼
{ accessToken, refreshToken, user }
```

### Refresh token rotativo

- Cada refresh token é armazenado no Mongo como **hash** (junto com `userId`, `expiresAt`, `revokedAt`).
- Em `POST /auth/refresh`:
  1. Verifica assinatura e TTL.
  2. Localiza o hash correspondente; se revogado ou ausente → 401.
  3. Revoga o atual e emite um novo par.
- `logout` revoga o refresh atual; logout em todas as sessões revoga todos os do usuário.

## Frontend — pontos-chave

- **Zustand** com middleware `persist` para sobreviver a reloads; seletores finos evitam re-render desnecessário.
- **React Hook Form + Zod** em todos os formulários; um schema Zod por tela (`loginSchema`, `registerSchema`, etc.) em `schemas/`.
- **Rotas protegidas** via componente `<RequireAuth />` que lê o estado do `authStore` e redireciona para `/login`.
- **RBAC no front** é apenas UX (esconder menus/botões); a autoridade real está no backend (`RolesGuard`).
- **Interceptor de refresh** é reentrante: se já existe um refresh em voo, novas 401s aguardam a mesma Promise.

## Testes

Backend:

```bash
cd api
npm run test          # unit
npm run test:e2e      # end-to-end com Supertest
```

- **Unit** — services, guards e pipes; Mongoose mockado.
- **E2E** — `@nestjs/testing` + Supertest contra um MongoDB efêmero (mongodb-memory-server).

Frontend:

```bash
cd front
npm run test
```

## Decisões / notas

- **NestJS modular** — cada bounded context é um módulo; `AppModule` apenas compõe. Facilita extrair módulos para microsserviços futuramente.
- **Access curto + refresh rotativo** — access de 15min minimiza janela de token vazado; rotação invalida cadeia inteira se um refresh vaza.
- **Refresh armazenado como hash** — mesmo que o Mongo vaze, os refresh tokens não são utilizáveis diretamente.
- **RBAC simples por papéis** — suficiente para o escopo (user/admin). Pode evoluir para permissões granulares (`users:read`, `users:write`) sem quebrar o `RolesGuard`.
- **Validação com class-validator** — centralizada via `ValidationPipe` global; DTOs decoram as regras. `whitelist: true` descarta propriedades não declaradas.
- **Zod no front × class-validator no back** — intencionalmente duplicados: Zod entrega validação imediata no client; o back é a autoridade final. A duplicação é pequena e vale a UX.
- **Senhas com bcrypt (12 rounds)** — custo balanceado para API síncrona; ajustável via env.
- **Request-id por requisição** — gerado pelo middleware e propagado nos logs; todo erro 500 registra o id e a resposta o inclui para correlação suporte ↔ log.
