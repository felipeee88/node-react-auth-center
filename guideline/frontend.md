# Guideline — Frontend (React + Vite)

Este documento define as convenções de arquitetura, código e UX para o frontend do **node-react-auth-center**. Serve como referência ao implementar as telas de **login**, **registro**, **perfil** e **gestão de usuários (admin)**.

## Stack

- **React 18** + **TypeScript** (strict)
- **Vite** — build e dev server
- **Zustand** — estado global (auth + usuário corrente), com middleware `persist`
- **React Router** — roteamento com rotas protegidas
- **React Hook Form** + **Zod** — formulários tipados e validação
- **Axios** — cliente HTTP com interceptors (JWT + refresh automático)
- **Vitest** + **React Testing Library** — testes
- **ESLint** + **Prettier** para padronização

## Estrutura de pastas

```
front/
└── src/
    ├── api/              (cliente axios, chamadas por recurso)
    ├── components/       (reutilizáveis — Button, Input, Modal, FormField)
    ├── features/
    │   ├── auth/         (login, register, forgot-password, reset-password)
    │   ├── profile/      (perfil do usuário logado)
    │   └── users/        (gestão de usuários — admin)
    ├── stores/           (Zustand — authStore, userStore)
    ├── hooks/            (useAuth, useCurrentUser, useDebounce)
    ├── routes/           (React Router + rotas protegidas)
    ├── schemas/          (schemas Zod — loginSchema, registerSchema, etc.)
    └── types/            (tipos TypeScript compartilhados)
```

Regras:

- **api/** — um arquivo por recurso (`auth.api.ts`, `users.api.ts`); funções retornam o payload já desembrulhado do envelope `{ data, success }`.
- **components/** — sem estado de servidor; apenas apresentação reutilizável.
- **features/** — cada feature contém seus componentes, rotas e hooks específicos.
- **stores/** — um arquivo por store; seletores finos exportados junto para evitar re-renders.
- **schemas/** — schemas Zod **compartilhados** (usados por mais de uma feature). Schemas específicos de uma tela vivem dentro da própria feature.

## Autenticação no cliente

### authStore (Zustand)

```ts
interface AuthState {
  accessToken: string | null
  refreshToken: string | null
  user: User | null
  setTokens: (tokens: Tokens) => void
  setUser: (user: User) => void
  clear: () => void
}
```

- Criado com o middleware `persist` para sobreviver a reloads (chave única: `auth-center:auth`).
- **Seletores finos** — `useAuthStore(s => s.user)` em vez de `useAuthStore(s => s)` para evitar re-render em toda mutação.
- Tokens vivem em `localStorage`. Reconhecer o trade-off: localStorage é vulnerável a XSS; produção deve mover refresh para cookie `httpOnly` (requer ajuste no backend).

### Interceptor Axios

Em `lib/axios.ts`:

- **Request interceptor** — injeta `Authorization: Bearer {accessToken}` a partir do store.
- **Response interceptor** em 401:
  1. Se não há `refreshToken`, limpa store e redireciona para `/login`.
  2. Caso contrário, chama `POST /auth/refresh` com o refresh atual.
  3. Atualiza tokens no store e **repete a requisição original**.
  4. Se o refresh falhar, limpa store e redireciona.
- **Reentrância** — se um refresh já está em voo, novas 401s **aguardam a mesma Promise** (evita múltiplos refreshes concorrentes que invalidariam uns aos outros, dado o modelo rotativo do backend).

Esboço:

```ts
let refreshPromise: Promise<Tokens> | null = null

axios.interceptors.response.use(undefined, async (error) => {
  if (error.response?.status !== 401) throw error
  refreshPromise ??= refreshTokens()
  try {
    const tokens = await refreshPromise
    useAuthStore.getState().setTokens(tokens)
    error.config.headers.Authorization = `Bearer ${tokens.accessToken}`
    return axios(error.config)
  } finally {
    refreshPromise = null
  }
})
```

### Rotas protegidas

- Componente `<RequireAuth />` envolve rotas autenticadas; lê `accessToken` do store e redireciona para `/login` preservando a rota de origem (`state.from`).
- Componente `<RequireRole role="admin" />` para rotas de admin — **apenas UX**; autoridade real é do backend.

## Formulários (React Hook Form + Zod)

- Um **schema Zod por formulário** em `schemas/` ou na feature:

  ```ts
  export const loginSchema = z.object({
    email: z.string().email('Email inválido'),
    password: z.string().min(8, 'Mínimo 8 caracteres'),
  })
  export type LoginFormData = z.infer<typeof loginSchema>
  ```

- `useForm<LoginFormData>({ resolver: zodResolver(loginSchema) })`.
- Tipos derivados do schema são a fonte única da verdade — **não duplicar** com interfaces manuais.
- Submissão desabilita o botão enquanto a requisição está em voo.
- Erros 400 (`VALIDATION_ERROR`) do backend são mapeados de volta aos controles via `setError` usando `error.details`.
- Erros 401/403/409 viram mensagens globais (snackbar/toast) — não controle específico.

## RBAC no frontend

- **Apenas UX** — esconder menus, desabilitar botões, hide de rotas.
- Toda autoridade fica no backend (`RolesGuard`). O frontend **não deve confiar** no próprio estado para autorizar ações críticas.
- Helper `useHasRole('admin')` lê `user.roles` do store.

## Comunicação com a API

- Funções em `api/` retornam **payload já desembrulhado**:

  ```ts
  export async function login(dto: LoginDto): Promise<LoginResponse> {
    const { data } = await http.post<Envelope<LoginResponse>>('/auth/login', dto)
    return data.data
  }
  ```

- Tipos de request/response em `types/` ou ao lado do serviço.
- `VITE_API_URL` em `front/.env` define a base URL.

## Padrões de código

- **TypeScript strict** — nada de `any`; `unknown` quando necessário com narrowing explícito.
- **Componentes funcionais** apenas.
- Nomes de arquivo em `kebab-case` (`login-page.tsx`, `auth.store.ts`, `users.api.ts`).
- Um componente/export por arquivo; exports nomeados.
- Hooks prefixados com `use`.
- Props tipadas via `interface` ou `type`; `children` com `React.ReactNode`.
- `useCallback`/`useMemo` só quando há ganho real (filho memoizado, lista grande).

## Estado

- **Auth** (tokens, user) → Zustand com `persist`.
- **Estado de UI efêmero** (modal aberto, menu) → `useState` local.
- **Listagens do backend** → se o projeto crescer, adotar React Query. Neste escopo inicial (CRUD simples de auth/users), chamadas diretas via `api/` com `useState` são aceitáveis.

## Testes

- **Vitest + RTL** com foco em:
  - Validação de formulários (schemas Zod aplicados corretamente, mensagens de erro exibidas).
  - Fluxos de auth — login sucesso, login falha, logout limpa o store.
  - Interceptor de refresh — mock do axios, simulação de 401 e verificação de retry único.
  - Componentes de rota protegida.
- Criar um `authStore` limpo em cada teste (evitar vazamento de estado entre testes).

## UX e acessibilidade

- Mensagens de erro claras, próximas ao campo; mensagens globais via `Snackbar`/`Toast`.
- Labels sempre associados a inputs (`<label htmlFor="...">` ou input dentro do label).
- Foco visível e navegação por teclado em todos os formulários.
- Redirect pós-login respeita a rota original (`state.from`).
- `/forgot-password` e `/reset-password` existem mesmo que o envio de e-mail seja mock em dev.

## Build e ambientes

- `npm run dev` — Vite em `http://localhost:5173`.
- `npm run build` — produção em `dist/`.
- `front/.env` — apenas variáveis **públicas** (o frontend é servido ao cliente). Nunca colocar `JWT_SECRET` ou segredos do backend aqui.
- `npm run lint` e `npm run test` passam como pré-requisito para merge.
