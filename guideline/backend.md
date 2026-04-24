# Guideline — Backend (API NestJS)

Este documento define as convenções de arquitetura, código e testes para a API do **node-react-auth-center**. Serve como referência ao implementar os módulos `auth`, `users` e `roles`.

## Objetivo do sistema

Sistema de autenticação e gestão de usuários com **cadastro, login, perfil e permissões** baseadas em papéis. Usa **JWT de curta duração** + **Refresh Token rotativo** e expõe endpoints REST consumidos por um frontend React.

## Stack

- **Node.js 20+** · **NestJS 10** · **TypeScript** strict
- **MongoDB** com **Mongoose** ODM
- **JWT** (access curto) + **Refresh Token** rotativo, armazenado como **hash**
- **Passport** (`passport-jwt`) para estratégias
- **class-validator** + **class-transformer** para validação de DTOs
- **bcrypt** para hash de senhas
- **Jest** + **Supertest** + **mongodb-memory-server** para testes

## Metodologias

- **Modular Architecture (NestJS)** — cada bounded context é um módulo auto-contido com controller + service + schemas + DTOs + guards/decorators específicos. `AppModule` apenas compõe.
- **SOLID** — SRP nos services (uma regra por método), DIP via injeção do Nest, OCP nos guards (novos papéis não exigem alterar `RolesGuard`).
- **JWT Auth com rotação de refresh** — access token curto minimiza janela de vazamento; refresh rotativo invalida a cadeia inteira se um token vaza.

## Estrutura do projeto

```
api/
└── src/
    ├── modules/
    │   ├── auth/
    │   │   ├── guards/        (JwtAuthGuard, RolesGuard)
    │   │   ├── strategies/    (jwt.strategy.ts, refresh.strategy.ts)
    │   │   ├── decorators/    (@Roles, @CurrentUser, @Public)
    │   │   ├── dto/           (LoginDto, RegisterDto, RefreshDto)
    │   │   ├── schemas/       (refresh-token.schema.ts)
    │   │   ├── auth.controller.ts
    │   │   ├── auth.service.ts
    │   │   └── auth.module.ts
    │   ├── users/
    │   │   ├── schemas/       (user.schema.ts)
    │   │   ├── dto/
    │   │   ├── users.controller.ts
    │   │   ├── users.service.ts
    │   │   └── users.module.ts
    │   └── roles/             (enum Role, definição de permissões)
    ├── common/
    │   ├── middlewares/       (logger, request-id)
    │   ├── filters/           (HttpExceptionFilter global)
    │   ├── interceptors/      (TransformInterceptor — envelope de resposta)
    │   └── pipes/             (ValidationPipe customizado)
    ├── config/                (env validation com Joi, mongoose, jwt)
    ├── app.module.ts
    └── main.ts
```

Regras:

- **Módulos são autocontidos** — dependências entre módulos são explícitas via `imports` e `exports`. Nunca importe um service de outro módulo sem que o módulo de origem o tenha exportado.
- **Controllers são finos** — validação via DTOs + pipes, orquestração delegada ao service.
- **Schemas Mongoose não vazam** — ficam em `modules/<m>/schemas/` e são acessados apenas pelo service daquele módulo.
- **common/** não tem lógica de negócio — só cross-cutting.

## Autenticação

Fluxo **access + refresh** com rotação:

1. `POST /auth/register` cria o usuário (senha com bcrypt, 12 rounds por padrão).
2. `POST /auth/login` emite `{ accessToken, refreshToken, user }`.
3. Requisições protegidas usam `Authorization: Bearer {accessToken}`.
4. `POST /auth/refresh` recebe o refresh atual, **valida**, **revoga** e emite um par novo.
5. `POST /auth/logout` revoga o refresh token atual (logout "em todas as sessões" revoga todos os do usuário).

### Refresh token rotativo

- Cada refresh é armazenado no Mongo **como hash** (`bcrypt` ou SHA-256 com pepper) junto com `userId`, `expiresAt` e `revokedAt`.
- No `refresh`:
  1. Verifica assinatura e TTL via `JwtService`.
  2. Localiza o hash correspondente; se não existir, estiver revogado ou expirado → `401`.
  3. Revoga o atual (`revokedAt = now`) e emite um novo par.
- **Nunca** retornar ou aceitar o refresh em cookie simples sem `httpOnly` + `secure` + `sameSite`. Neste projeto ele viaja no corpo por simplicidade, mas a produção deve migrar para cookie `httpOnly`.

### Variáveis de ambiente exigidas

Validadas via Joi em `config/env.validation.ts`. Faltando → a app não sobe.

- `PORT`, `MONGO_URI`
- `JWT_SECRET`, `JWT_EXPIRES_IN` (padrão `15m`)
- `REFRESH_SECRET`, `REFRESH_EXPIRES_IN` (padrão `7d`)
- `BCRYPT_ROUNDS` (padrão `12`)

## Permissões (RBAC)

Modelo baseado em papéis:

- `user` — pode gerenciar o próprio perfil.
- `admin` — pode listar, criar, atualizar e desativar outros usuários.

Aplicação:

- **`JwtAuthGuard`** — registrado como guard global; rotas públicas usam o decorator `@Public()`.
- **`RolesGuard`** + **`@Roles('admin')`** — restringe por papel.
- **`@CurrentUser()`** — injeta o usuário autenticado no handler (lido do `ClaimsPrincipal`/request).

Exemplo:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get()
list(@CurrentUser() user: UserDoc) {
  return this.usersService.findAll();
}
```

Evolução futura: migrar para **permissões granulares** (`users:read`, `users:write`) sem quebrar o `RolesGuard` — basta trocar o decorator por `@Permissions(...)` e ajustar o guard.

## Middlewares, interceptors e filtros

Registrados globalmente em `main.ts` / `AppModule`:

- **`LoggerMiddleware`** — loga método, URL, status e tempo de resposta; gera `request-id` (`x-request-id`) e o propaga nos logs.
- **`TransformInterceptor`** — envelopa respostas bem-sucedidas em `{ data, success }`.
- **`ValidationPipe` global** — transforma e valida DTOs com `whitelist: true` e `forbidNonWhitelisted: true`.
- **`HttpExceptionFilter`** — padroniza erros (veja abaixo) e correlaciona com `request-id`.

## Tratamento de erros

Formato único:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email must be an email",
    "details": [{ "field": "email", "message": "must be an email" }]
  },
  "timestamp": "2026-04-24T12:34:56Z",
  "path": "/auth/register",
  "requestId": "..."
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

Stack traces **não** vão na resposta; são logados com o `requestId` correspondente.

## Convenções de API

- Rotas no plural quando coleção (`/users`, `/users/:id`), singular quando recurso do caller (`/me`).
- Paginação/filtro: `page`, `size`, `sort` (formato `campo:asc|desc`), filtros específicos do recurso.
- Envelope `{ data, success }` em toda resposta de sucesso.
- Status codes respeitados (201 Created, 204 No Content, 409 Conflict para e-mail duplicado, etc.).
- `userId` do autor **nunca** vem no payload — sempre lido do token.

## Persistência (Mongoose)

- Schemas com `@Schema({ timestamps: true })` — `createdAt` e `updatedAt` automáticos.
- Senhas: campo `password` com `select: false` no schema, transform no `toJSON` removendo `password` e `__v`.
- Índices explícitos: `email` único em `users`, composto `{ userId: 1, revokedAt: 1 }` em `refresh_tokens`.
- TTL index opcional em `refresh_tokens.expiresAt` para limpeza automática.

## Testes

Dois sabores, ambos em Jest:

- **Unit** (`npm run test`) — services, guards e pipes. Mongoose e dependências mockados.
- **E2E** (`npm run test:e2e`) — `@nestjs/testing` + Supertest contra MongoDB efêmero via `mongodb-memory-server`. Cobre fluxos completos (register → login → refresh → logout).

Padrões:

- Arrange/Act/Assert; nomes `Method_Scenario_Expectation`.
- Zero uso de `@Inject` real em unit tests — sempre via `Test.createTestingModule` com providers mockados.

## Padrões de código

- **TypeScript strict** em toda a base.
- **async/await** em todo I/O; sem `.then()` encadeado exceto em cleanup.
- **DTOs imutáveis** — campos `readonly`, validados via decorators de `class-validator`.
- **Services retornam DTOs**, não documentos Mongoose crus (evita vazar `password`, `__v`, etc.).
- **Erros de domínio** lançados como `HttpException` apropriado (`ConflictException` para e-mail duplicado, `ForbiddenException` para ações não autorizadas, etc.).
- **Sem lógica em controllers** — só delegação.
- **Nada de `any`** — `unknown` quando necessário, com narrowing explícito.

## Segurança — checklist

- Senhas sempre hasheadas com bcrypt (rounds via env).
- Refresh tokens armazenados **apenas como hash**.
- `ValidationPipe` com `whitelist` descarta payload excedente.
- `helmet` habilitado em `main.ts`.
- CORS restrito à URL do frontend em produção.
- Rate limiting (`@nestjs/throttler`) em `/auth/login` e `/auth/refresh`.
- Logs **nunca** imprimem senhas, tokens ou headers de `Authorization`.
