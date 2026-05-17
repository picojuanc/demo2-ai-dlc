# Testing

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap. El Service Agent genera
> tests según estas reglas y `/spec-verify` audita trazabilidad.

## Niveles obligatorios

| Nivel | Cuándo aplica | Herramienta | Cobertura mínima |
|---|---|---|---|
| Unit | Toda función en `domain/` y `application/` | **Vitest 2.x** | ≥ 90 % en esas capas |
| Integration | Cada Route Handler en `app/api/**/route.ts` y middleware | **Vitest** + handler real con puertos mockeados (`msw` para HTTP externo) | 100 % de Route Handlers |
| Contract | Validación bidireccional Zod ↔ OpenAPI | **Vitest** + `zod-to-openapi` | 100 % de endpoints publicados en `docs/openapi.yaml` |
| E2E | Flows críticos de UI y APIs públicas | **Playwright** | flows de la spec marcados como críticos |
| Load | Cuando hay NFR de p99 / throughput en `requirements.md` | **k6** | NFRs declarados |
| Accessibility | Cada vista pública (rutas `(marketing)`, login, formularios) | `@axe-core/playwright` integrado en E2E | WCAG AA |

## Cobertura mínima global

- **Total**: **80 %** (líneas + branches).
- **Por capa**:
  - `src/domain/` y `src/application/`: **≥ 90 %**.
  - `src/infrastructure/`: ≥ 70 % (foco en adapters críticos, no
    en wrappers triviales).
  - `src/app/`: sin umbral rígido; el foco es integration / E2E que
    cubre handlers y páginas.
- **Excepciones permitidas** (no cuentan al denominador, declaradas
  en `vitest.config.ts` `coverage.exclude`):
  - Archivos generados (`src/components/ui/*` de shadcn).
  - Tipos puros (`*.types.ts`, `*.d.ts`).
  - `next.config.ts`, `tailwind.config.ts`, `instrumentation.ts`.
  - `src/composition.ts` (cubierto indirectamente por integration).
  - Storybook stories si existen.

## Trazabilidad spec ↔ test

Cada test cita su requirement de origen. El comentario es
**obligatorio** justo encima del `it`:

```ts
// Derived from R1.3
it("retorna 401 cuando el JWT está firmado con secret incorrecto", async () => {
  // ...
});
```

`/spec-verify` reporta como **gap**:

- Tests sin `// Derived from R<x>.<y>`.
- Tests cuyo `R<x>.<y>` ya no existe en `requirements.md` (drift).
- `R<x>.<y>` que no tiene ningún test asociado.

Para tests que cubren múltiples requirements, usar lista:
`// Derived from R1.3, R1.5`.

## Estrategia por requirement

En `requirements.md`, cada `R*.*` declara su(s) nivel(es) de test al
escribirlo:

```markdown
### R1.3 — Cookie de sesión HTTP-only
WHEN un usuario hace login exitoso THEN el sistema SHALL setear una
cookie `dad_session` con flags `httpOnly`, `Secure`, `SameSite=Lax`.
Tests: integration, contract
```

`/spec-verify` valida que los niveles declarados existen
efectivamente en `tests/`.

## Mocking

- **Política general**: mockear sólo **puertos** (interfaces de
  `application/<ctx>/ports/`) y procesos externos (HTTP, reloj).
  Nunca mockear nada de `domain/`.
- **HTTP externo**: **MSW** (`msw/node`) interceptando `fetch` en
  unit/integration. Para E2E, MSW corriendo en el server de Next
  (vía `instrumentation.ts` con guard de `NODE_ENV !== "production"`).
- **Reloj**: usar `vi.useFakeTimers()` para tests con TTL de JWT.
- **JWT real en tests**: firmar tokens de test con `jose` y el mismo
  secret que carga `env.ts` desde `.env.test`. **Nunca** reutilizar
  el secret de dev/prod.
- **Cuándo retirar un mock** (de un servicio externo): el archivo del
  mock debe declarar al inicio:

  ```ts
  // Ready to unmock: cuando identity-api esté desplegado en staging
  //                  y el contrato C-3 esté firmado por su equipo.
  ```

  Esto se sigue automáticamente por `/spec-verify` (§6 del methodology).

## Datos de prueba

- **Fixtures**: en `tests/fixtures/` (JSON con datos canónicos y/o
  factories TypeScript con `vitest`-compatible).
- **Factories**: una factory por entidad del dominio en
  `tests/factories/<entidad>.ts`, devuelve objetos válidos con
  overrides opcionales.
- **Seeding**: N/A (sin BD).
- **Cleanup**: `beforeEach` resetea mocks (`vi.resetAllMocks()`) y
  el handler de MSW (`server.resetHandlers()`).

## Tests E2E (Playwright)

- **Status**: **bloqueante en CI desde la primera feature**.
- **Flows iniciales** (se concretan al cerrar la primera spec):
  - Login con credenciales válidas → cookie seteada → acceso a ruta
    protegida.
  - Login fallido → mensaje genérico, sin cookie.
  - Logout → cookie removida → redirect a `/login`.
- **Ubicación**: `tests/e2e/<flow>.spec.ts`.
- **Setup**:
  - `playwright.config.ts` levanta `pnpm build && pnpm start` en
    `webServer`.
  - MSW activo para servicios externos (vía `instrumentation.ts`).
  - JWT firmado en `globalSetup` para tests autenticados.
- **Accessibility**: cada test E2E que renderiza UI corre
  `injectAxe()` + `checkA11y()`. Violaciones críticas fallan el
  test.

## Contract tests (OpenAPI)

- **Fuente de verdad**: schemas Zod de los Route Handlers.
- **Generación**: `zod-to-openapi` produce `docs/openapi.yaml` en CI
  (no committeado a mano).
- **Validación bidireccional**: tests en `tests/contract/`:
  1. **Forward**: para cada endpoint, hace request real, captura la
     response, valida contra el schema OpenAPI generado.
  2. **Backward**: el schema actual no rompe el contrato previo
     publicado en `.org/contracts/` (si aplica) — diff semántico,
     no textual.
- **Drift**: si el schema generado difiere del committed en
  `.org/contracts/`, falla el build hasta que se publique
  formalmente la nueva versión (con bump semver del contrato).

## Tests de seguridad (mini)

Aunque no son un "nivel" formal, son obligatorios para cada Route
Handler protegido:

- Caso negativo de auth: sin cookie → 401.
- Caso negativo de auth: cookie con JWT expirado → 401.
- Caso negativo de auth: cookie con firma inválida → 401.
- Caso negativo de authz (si aplica): rol insuficiente → 403/404
  según política (`security.md` § Información expuesta).

Cada uno debe tener `// Derived from R<x>.<y>` que mapee al
requirement de seguridad correspondiente.

## CI

- **Pipeline**: `azure-pipelines.yml` (se crea al primer feature).
- **Jobs y bloqueo de merge**:
  - `lint`: ESLint + `tsc --noEmit` — **bloqueante**.
  - `unit`: `pnpm test:unit` (Vitest, coverage report) — **bloqueante**.
  - `integration`: `pnpm test:integration` — **bloqueante**.
  - `contract`: `pnpm test:contract` — **bloqueante**.
  - `e2e`: `pnpm test:e2e` (Playwright) — **bloqueante** desde la
    primera feature.
  - `a11y`: corre dentro de `e2e` — bloqueante (sólo violaciones
    `serious`/`critical`).
  - `load`: sólo si la spec lo declara como `D-N`/NFR; corre nightly
    o on-demand; **no bloquea** PR salvo regresión > umbral
    declarado en la spec.
- **Coverage report**: subido como artifact + comentado en el PR. PR
  con coverage por debajo del umbral global o por capa bloquea
  merge.
- **Tests bloqueantes resumido**: `lint`, `unit`, `integration`,
  `contract`, `e2e` (+ `a11y` dentro de e2e).
- **No bloqueantes**: `load`, security scans (SAST/DAST) cuando se
  integren.

## Scripts npm/pnpm de referencia

```jsonc
{
  "scripts": {
    "test":             "pnpm test:unit && pnpm test:integration && pnpm test:contract",
    "test:unit":        "vitest run --coverage tests/unit",
    "test:integration": "vitest run tests/integration",
    "test:contract":    "vitest run tests/contract",
    "test:e2e":         "playwright test",
    "test:watch":       "vitest"
  }
}
```
