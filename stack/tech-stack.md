# Tech stack

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap. Cambios posteriores requieren
> ADR en `stack/architecture.md` y, si afectan compliance, escalación al
> Security Agent.

## Lenguaje y runtime

- **Lenguaje**: TypeScript (modo `strict: true`)
- **Versión runtime**: Node.js 22 LTS
- **Runtime de deploy**: `node:22-alpine` (container OCI estándar)

## Framework principal

- **Framework**: Next.js 15 con **App Router**
- **Modo**: full-stack (UI en React Server Components + API routes en
  `app/api/*/route.ts`)
- **Razón**: un solo deploy para frontend y backend; SSR/RSC reducen
  payload al cliente; Route Handlers ofrecen API REST sin necesidad de
  un backend separado para este tamaño de servicio.

## UI / styling

- **CSS**: Tailwind CSS (utility-first, configurado vía
  `tailwind.config.ts`)
- **Componentes**: shadcn/ui (componentes copiables sobre Radix UI; se
  instalan vía `pnpm dlx shadcn@latest add <component>` y viven en
  `src/components/ui/`)
- **Iconos**: lucide-react (default de shadcn)
- **Fuentes**: `next/font` (auto-optimización, sin requests externos)

## Persistencia

- **Base de datos**: **ninguna** — el servicio es stateless.
- **Patrón de ownership** (§D7): N/A (no posee datos).
- Si en el futuro se introduce persistencia, requiere ADR y actualizar
  este archivo + `architecture.md`.

## Mensajería / eventos

- **Broker**: ninguno por ahora.
- Si se introduce, candidato por defecto: **Azure Service Bus** (por
  alineación con OpenShift sobre Azure citado en `AGENTS.md`).

## Cache

- **Tecnología**: ninguna externa.
- **Cache de Next**: se usa el cache nativo de Next 15 (`fetch` con
  `revalidate`, `unstable_cache` para datos derivados). No depende de
  Redis ni de almacenamiento externo.

## Autenticación / sesión

- **Mecanismo**: JWT firmado, transportado en **cookie `httpOnly`**
  (también `Secure`, `SameSite=Lax`).
- **Storage de sesión**: ninguno — el token contiene la identidad y se
  valida por firma en cada request.
- **Validación**: en `middleware.ts` (Edge runtime) para rutas
  protegidas, y dentro de Route Handlers para chequeos finos.
- **Refresh**: TODO al definir la primera feature con auth (decidir si
  rotación silenciosa o re-login).
- Detalles de issuer / algoritmo / rotación de claves: ver
  `stack/security.md`.

## API style

- **Estilo**: REST sobre HTTP, JSON.
- **Documentación**: **OpenAPI 3.1**, generada/mantenida en
  `docs/openapi.yaml`. Endpoint `/api/openapi.json` lo expone en tiempo
  de ejecución para herramientas tipo Swagger UI.
- **Ubicación de contratos cross-service**: `.org/contracts/` si el
  endpoint lo consume otro repo; `docs/openapi.yaml` para contratos
  internos al servicio.

## Observabilidad

- **Logs**: `pino` con salida JSON estructurada a `stdout` (recolección
  por la plataforma de deploy).
- **Métricas + tracing**: **OpenTelemetry** vía
  `@vercel/otel` + `@opentelemetry/api`. Exporter OTLP HTTP al
  collector interno (URL vía env `OTEL_EXPORTER_OTLP_ENDPOINT`).
- **Campos obligatorios en logs**: ver `stack/patterns.md` § Logging.
- **APM**: lo que tenga la org detrás del collector (Jaeger / Tempo /
  Loki / Prometheus). No se asume vendor específico.

## Build y empaquetado

- **Package manager**: **pnpm** (lockfile `pnpm-lock.yaml` committeado).
- **Build tool**: Next CLI (`pnpm build`, `pnpm start`).
- **Imagen base**: `node:22-alpine` con multi-stage build (deps →
  build → runner) y `output: 'standalone'` en `next.config.ts`.
- **CI**: Azure DevOps Pipelines (`azure-pipelines.yml` — se crea con
  la primera feature, ver §13 del methodology).
- **CD**: a definir junto con el deploy target (ver siguiente sección).

## Deploy target

- **Estado**: **no decidido todavía**.
- Candidatos:
  - OpenShift namespace (alineado con el default de `AGENTS.md`).
  - Vercel (nativo para Next.js, menos control).
  - Container genérico en otra nube (AKS / Cloud Run / ECS).
- **Bloqueante para**: gates G5 / G6, secrets management (Sealed
  Secrets vs. Key Vault vs. Vercel env vars), pipeline CD.
- Hasta que se decida, mantener el código agnóstico (sin APIs
  específicas de Vercel ni de OpenShift).

## Versiones congeladas

Pinear en `package.json` (sin `^` ni `~` para estas):

- `next@15.x` (versión exacta al cierre del bootstrap; subir vía PR
  dedicado)
- `react@19.x` (incluida con Next 15)
- `typescript@5.x`

El resto de dependencias siguen `^` salvo que el Security Agent
indique pinear por CVE.
