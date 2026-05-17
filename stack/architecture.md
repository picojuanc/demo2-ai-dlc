# Architecture

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap.

## Patrón arquitectónico

- **Patrón principal**: **Clean Architecture / Hexagonal**, adaptado a
  Next.js 15 App Router.
- **Razón**:
  - Mantiene la lógica de negocio independiente del framework (si en
    el futuro hace falta migrar el frontend o exponer otra interfaz,
    `domain/` y `application/` no se tocan).
  - Hace explícitos los puertos hacia integraciones externas, lo que
    facilita testeo (mocks puntuales) y revisión de contratos
    cross-service (`.org/contracts/`).
  - El costo en ceremonia es manejable porque el servicio es
    stateless: `infrastructure/` se mantiene delgada (clientes HTTP,
    logger, validación), sin capa de persistencia.

## Capas y reglas de dependencia

Sólo se permite que las capas internas sean importadas por las
externas. La dirección de dependencia es estricta:

```
presentation (app/)  ──►  application/  ──►  domain/
        │                       │
        └──►  infrastructure/  ─┘
                  │
                  ▼
              (procesos externos: HTTP, OTel, env, etc.)
```

Reglas concretas (el Service Agent rechaza PRs que las violen):

- `domain/` **no importa** de ninguna otra capa ni de librerías de
  framework (sólo TypeScript puro + Zod para schemas si aplica).
- `application/` puede importar de `domain/`. Define **puertos**
  (interfaces) que `infrastructure/` implementa.
- `infrastructure/` puede importar de `application/` y `domain/`.
  Aquí viven adapters: clientes HTTP, logger, lectura de env vars,
  validadores OpenAPI, etc.
- `app/` (Next.js: páginas, layouts, route handlers, server actions)
  puede importar de `application/` y `domain/`. **No importa**
  directamente de `infrastructure/` salvo a través de un *composition
  root* (`src/composition.ts`) que cablea puertos a implementaciones.
- `shared/` contiene utilidades puras transversales (formatters,
  errores tipados). Puede ser importado por cualquier capa, pero no
  importa de nadie.

## Estructura de carpetas

```
src/
├── app/                          # Presentation (Next 15 App Router)
│   ├── (marketing)/              # route group público
│   ├── (app)/                    # route group autenticado
│   ├── api/
│   │   └── <recurso>/route.ts    # Route Handlers REST
│   ├── layout.tsx
│   ├── middleware.ts             # auth en edge (validación JWT)
│   └── globals.css
│
├── domain/                       # núcleo: entidades, value objects, errores
│   ├── <bounded-context>/
│   │   ├── entities.ts
│   │   ├── value-objects.ts
│   │   └── errors.ts
│   └── shared/                   # tipos transversales del dominio
│
├── application/                  # use cases + puertos
│   ├── <bounded-context>/
│   │   ├── use-cases/
│   │   │   └── <verbo-objeto>.ts # ej. authorize-request.ts
│   │   └── ports/                # interfaces que infra implementa
│   └── shared/                   # tipos comunes (Result<T>, etc.)
│
├── infrastructure/               # adapters concretos
│   ├── http/                     # clientes a APIs externas
│   ├── auth/                     # verificación JWT, lectura de cookie
│   ├── observability/            # pino, OTel setup
│   ├── config/                   # env vars (zod-validated)
│   └── openapi/                  # generación / serving del schema
│
├── components/
│   └── ui/                       # shadcn/ui (generados, no editar a mano)
│
├── lib/                          # utilidades de UI (cn(), hooks, etc.)
│
├── shared/                       # cross-capa: errores base, helpers puros
│
└── composition.ts                # composition root: wiring puerto→impl

tests/
├── unit/                         # domain + application (sin I/O)
├── integration/                  # route handlers con infra real (mocks externos)
├── contract/                     # validación de schemas OpenAPI
└── e2e/                          # Playwright sobre la app desplegada
```

## Composition root

`src/composition.ts` es el **único** lugar donde se ensamblan
implementaciones concretas. Expone una factory que los Route Handlers
y Server Actions invocan para obtener un use case ya cableado:

```ts
// src/composition.ts (esqueleto)
import { createAuthorizeRequest } from "@/application/auth/use-cases/authorize-request";
import { jwtVerifier } from "@/infrastructure/auth/jwt-verifier";

export const authorizeRequest = createAuthorizeRequest({ jwtVerifier });
```

Nada en `app/` debería instanciar adapters directamente.

## Componentes principales

| Componente | Responsabilidad | Carpeta |
|---|---|---|
| Route Handlers | Recibir HTTP, validar entrada (zod), llamar use case, serializar respuesta | `src/app/api/**/route.ts` |
| Middleware | Verificar cookie JWT en rutas protegidas | `src/middleware.ts` |
| Use cases | Lógica de aplicación (orquesta dominio + puertos) | `src/application/<ctx>/use-cases/` |
| Entidades / VOs | Reglas de negocio puras, invariantes | `src/domain/<ctx>/` |
| HTTP clients | Llamadas a APIs externas (cuando aparezcan) | `src/infrastructure/http/` |
| Composition root | Cableado de implementaciones a puertos | `src/composition.ts` |
| OpenAPI spec | Contrato REST, fuente de verdad de la API | `docs/openapi.yaml` |

## Diagrama de capas

```
┌────────────────────────────────────────────────────────────┐
│  Presentation:  app/  (React Server Components + Route     │
│                 Handlers + middleware)                     │
└──────────┬─────────────────────────────────────────────────┘
           │ usa
           ▼
┌────────────────────────────────────────────────────────────┐
│  Application:   use cases  ─────────── ports (interfaces)  │
└──────────┬─────────────────────────────────────┬───────────┘
           │                                     ▲
           │ usa                       implementa │
           ▼                                     │
┌──────────────────────────────────┐  ┌──────────┴───────────┐
│  Domain:    entidades, VOs,      │  │  Infrastructure:     │
│             reglas puras         │  │  HTTP, JWT, OTel,    │
│                                  │  │  env, OpenAPI        │
└──────────────────────────────────┘  └──────────────────────┘
```

## Integraciones externas

Hoy: **ninguna conocida**.

Cuando aparezca la primera, se documenta aquí y se crea un `D-N`
(Dependencies) en `specs/<feature>/requirements.md` (§6 del methodology).
Para cada integración:

- Definir un **puerto** en `src/application/<ctx>/ports/`.
- Implementar el adapter en `src/infrastructure/http/`.
- Si es cross-team, publicar el contrato en `.org/contracts/` y
  escalar al Architect Agent antes de implementar.

## Decisiones arquitectónicas (ADRs)

Las decisiones importantes se registran en
`docs/decisions/NNNN-<titulo-kebab>.md` (formato MADR liviano: contexto,
decisión, consecuencias). Si la decisión afecta a otros repos, copiar
también a `.org/decisions/`.

ADRs iniciales (todavía sin escribir):

- `0001-clean-architecture.md` — por qué Clean Arch sobre vertical
  slice en un servicio Next stateless.
- `0002-jwt-en-cookie-httponly.md` — por qué cookie y no `Authorization`
  header.

Se crean el día que se necesiten justificar las decisiones a un nuevo
miembro del equipo o a un reviewer externo — no como deber retroactivo.
