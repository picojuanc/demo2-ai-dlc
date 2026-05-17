# Patterns — naming, commits, organización

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap. El Service Agent aplica
> estas reglas literalmente al generar archivos en `/spec-implement` y
> al revisar PRs.

## Naming

### Archivos

- **Todos los archivos** usan **camelCase** (incluidos los `.tsx`).
  - `userProfile.tsx`, `authorizeRequest.ts`, `jwtVerifier.ts`.
- **Excepciones obligatorias** (impuestas por Next 15 / convenciones
  externas, no se renombran):
  - Archivos especiales de Next: `page.tsx`, `layout.tsx`,
    `loading.tsx`, `error.tsx`, `not-found.tsx`, `route.ts`,
    `middleware.ts`.
  - Componentes generados por shadcn en `src/components/ui/` (siguen
    kebab-case por convención de la herramienta).
  - Archivos de config en la raíz: `next.config.ts`, `tailwind.config.ts`,
    etc.

### Símbolos exportados

| Elemento | Patrón | Ejemplo |
|---|---|---|
| Componente React | PascalCase | `export function UserProfile()` |
| Hook | camelCase con prefijo `use` | `export function useSessionUser()` |
| Use case (función factory) | camelCase con `create` o verbo de dominio | `createAuthorizeRequest`, `authorizeRequest` |
| Puerto / interface | PascalCase, sufijo del rol | `JwtVerifier`, `HttpClient` |
| Entidad / Value Object | PascalCase | `UserId`, `SessionToken` |
| Tipo de error tipado | PascalCase, sufijo `Error` | `AuthError`, `ValidationError` |
| Schema Zod | camelCase, sufijo `Schema` | `loginRequestSchema` |
| DTO / shape REST | PascalCase, sufijo `Dto` o `Request` / `Response` | `LoginRequest`, `LoginResponse` |
| Constante | UPPER_SNAKE_CASE | `MAX_TOKEN_AGE_SECONDS` |
| Test file | mismo nombre que el archivo bajo prueba + `.test.ts` | `authorizeRequest.test.ts` |

### Carpetas

- **Carpetas** también en **camelCase**, salvo grupos de ruta Next con
  paréntesis (`(marketing)`, `(app)`) y dinámicas (`[id]`,
  `[...slug]`), donde Next impone el formato.

## Manejo de errores

**Política**: errores de negocio son **valores**, no excepciones. Los
use cases retornan `Result<T, E>` tipado. Sólo se lanza una excepción
ante condiciones verdaderamente inesperadas (bugs, dependencia
externa caída sin handler).

### Tipo `Result`

Definido una sola vez en `src/shared/result.ts`:

```ts
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok  = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

### Errores tipados por bounded context

Cada contexto define sus errores como **union discriminada** en
`src/domain/<ctx>/errors.ts`:

```ts
export type AuthError =
  | { kind: "invalidToken"; reason: "expired" | "malformed" | "badSignature" }
  | { kind: "missingCookie" }
  | { kind: "issuerMismatch"; expected: string; got: string };
```

### Mapeo a HTTP en Route Handlers

El Route Handler hace `switch` exhaustivo sobre `error.kind` y mapea
a status HTTP. No hay middleware mágico — la traducción es explícita
y testeable.

```ts
// app/api/session/route.ts
const result = await authorizeRequest({ cookie });
if (!result.ok) {
  switch (result.error.kind) {
    case "missingCookie":  return Response.json({ error: "unauthenticated" }, { status: 401 });
    case "invalidToken":   return Response.json({ error: "unauthenticated" }, { status: 401 });
    case "issuerMismatch": return Response.json({ error: "forbidden" },       { status: 403 });
  }
}
return Response.json(result.value);
```

### Cuándo sí lanzar

- Falla de invariante de programación (`assert(false, "unreachable")`).
- Falla irrecuperable de configuración al boot (env var requerida no
  presente — falla rápido en `composition.ts`).

## Formato de commits

```
<type>(demo2-ai-dlc): T<n> - <descripción corta> [R<x>.<y>] AB#<workitem-id>
```

Donde:

- `<type>`: `feat | fix | refactor | test | docs | chore | perf | sign`
- `T<n>`: ID de la task en `specs/<feature>/tasks.md` (si aplica)
- `R<x>.<y>`: requirement EARS cubierto (puede ser lista
  `[R1.2, R1.3]`)
- `AB#<id>`: work item de Azure DevOps vinculado

Ejemplos:

```
feat(demo2-ai-dlc): T1 - login con JWT en cookie httpOnly [R1.3] AB#42
fix(demo2-ai-dlc): T3 - rechazar token con iss inválido [R1.5] AB#48
refactor(demo2-ai-dlc): T7 - extraer jwtVerifier a infra [R1.3] AB#42
chore(demo2-ai-dlc): bump next a 15.0.3
docs(demo2-ai-dlc): ADR 0001 sobre Clean Architecture
```

`chore` y `docs` pueden omitir `T<n>`, `[R*.*]` y `AB#` si la mejora
no está atada a una task de spec.

`sign` se usa exclusivamente para **firmar gates** (ver `AGENTS.md` §
Gates aplicables a este servicio → Mecanismo de firma). El subject
nombra el gate y la feature; el body documenta qué se firma, qué
commits se revisaron y cualquier contexto relevante. No lleva `T<n>`
ni `[R*.*]` salvo que el gate aplique a un requirement específico.

Ejemplos `sign`:

```
sign(demo2-ai-dlc): G2 events-calendar approved
sign(demo2-ai-dlc): G3 events-calendar design+tasks approved
sign(demo2-ai-dlc): G6 events-calendar rolled out
```

## Organización de tests

Cada test cita su requirement de origen con un comentario
**obligatorio** justo encima del `it` / `test`:

```ts
// Derived from R1.3
it("retorna 401 cuando la cookie no contiene un JWT válido", async () => { ... });
```

- **Framework**: Vitest 2.x (rápido, ESM nativo, compatible con
  ecosistema Next).
- **Estructura**:
  - `tests/unit/<ruta-espejo>/<archivo>.test.ts` para `domain/` y
    `application/`.
  - `tests/integration/<recurso>.test.ts` para Route Handlers, con
    `next-test-api-route-handler` o `vitest` levantando el handler.
  - `tests/contract/openapi.test.ts` valida que la spec OpenAPI
    coincide con los handlers.
  - `tests/e2e/` para Playwright.
- **Naming de tests**: `it("<comportamiento esperado> cuando <condición>", ...)` en español o inglés consistente dentro del archivo.
- **Fixtures**: en `tests/fixtures/` (JSON o factories TypeScript).

Reglas detalladas y cobertura mínima: ver `stack/testing.md`.

## Imports y módulos

- **Alias absoluto** `@/*` apunta a `src/*` (configurado en
  `tsconfig.json`). Úsalo para imports cross-capa.
- **Imports relativos** sólo dentro del mismo archivo padre directo
  (`./helpers`, `./types`).
- **Prohibidos**:
  - Imports circulares.
  - Que `domain/` importe nada fuera de `domain/` (ver
    `stack/architecture.md` § Reglas de dependencia).
  - `import *` (siempre nombres explícitos).

## Formato y linting

- **Formatter**: Prettier 3.x con config en `.prettierrc`. Sin punto y
  coma final: **NO** — usar punto y coma (default). 2 espacios de
  indentación, `printWidth: 100`, `trailingComma: "all"`.
- **Linter**: ESLint 9 (flat config) con:
  - `next/core-web-vitals`
  - `@typescript-eslint/recommended-type-checked`
  - `eslint-plugin-import` con `no-cycle` activado
  - regla custom (o `eslint-plugin-boundaries`) que prohíbe cruces de
    capa (`domain` → ningún import fuera, etc.).
- **Pre-commit**: Husky 9 + lint-staged corren en archivos staged:
  - `pnpm exec prettier --write`
  - `pnpm exec eslint --fix`
  - `pnpm exec tsc --noEmit` (sólo si hay archivos `.ts/.tsx` staged)

## TypeScript

- `strict: true` (incluye `strictNullChecks`, `noImplicitAny`,
  `strictFunctionTypes`).
- `noUncheckedIndexedAccess: true`.
- `noImplicitOverride: true`.
- Prohibido `any` salvo en código de boundary (parsing de JSON externo
  inmediatamente seguido de validación Zod).

## Logging

- **Librería**: `pino` configurado en
  `src/infrastructure/observability/logger.ts`.
- **Formato**: JSON estructurado (default de pino en producción).
- **Nivel por defecto**: `info` en prod, `debug` en dev (vía env
  `LOG_LEVEL`).
- **Campos obligatorios en cada log entry**:
  - `requestId` (UUID v4 generado en middleware, propagado por
    AsyncLocalStorage).
  - `service: "demo2-ai-dlc"` (default del child logger).
  - `traceId` y `spanId` cuando OTel esté activo.
  - `userIdHash` (SHA-256 truncado a 16 chars) si hay sesión —
    **nunca el user id raw ni el email**.
- **Prohibido loguear**:
  - JWT completos (sólo `claims.sub` hasheado).
  - Bodies de request / response sin sanitizar.
  - Cualquier PII en plain text (ver `stack/security.md`).
- **Contexto correlacionado**: usar `logger.child({ requestId })` al
  inicio del handler; no pasar `logger` por parámetros.
