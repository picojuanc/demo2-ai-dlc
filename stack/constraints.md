# Constraints — qué está prohibido en este repo

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap. Lista corta y específica
> al proyecto. Anti-patrones genéricos de seguridad viven en
> `stack/security.md`. El Service Agent rechaza PRs que violen estas
> reglas; no las relaja sin ADR.

## Datos y estado

- **NO** introducir persistencia (BD, KV store, filesystem) sin
  ADR + actualización de `stack/tech-stack.md`. El servicio es
  stateless por diseño.
- **NO** usar `globalThis` / variables a nivel de módulo para
  mantener estado mutable entre requests. Next 15 con RSC puede
  servir el mismo módulo entre renders concurrentes.
- **NO** confiar en `process.memoryUsage` ni en cache en memoria para
  datos que afecten la lógica de negocio. Sólo `unstable_cache` o el
  cache de `fetch` de Next, ambos con `revalidate` explícito.

## Lógica de negocio

- **NO** decidir autorización en el cliente. El middleware o el use
  case server-side es la única fuente de verdad.
- **NO** confiar en valores derivados del JWT sin verificación. Si un
  Server Component o Route Handler usa `claims`, debe venir del
  `jwtVerifier`, no de decodificar el token sin firma.
- **NO** mezclar lógica de negocio en componentes React (RSC o
  Client). Los componentes consumen un use case ya cableado vía
  `composition.ts`.

## Capas y dependencias

- **NO** importar desde `infrastructure/` en `app/` directamente —
  siempre vía `composition.ts`.
- **NO** importar desde `application/` o `infrastructure/` dentro de
  `domain/`. `domain/` es puro.
- **NO** crear dependencias circulares entre features ni entre
  módulos de la misma capa. ESLint `no-cycle` está activo.
- **NO** poner archivos `.tsx` (JSX) en `domain/`, `application/` o
  `infrastructure/`. JSX vive sólo en `app/`, `components/` y `lib/`.

## Manejo de errores

- **NO** lanzar excepciones para flujo de negocio. Los use cases
  retornan `Result<T, E>` (ver `stack/patterns.md` § Error handling).
- **NO** hacer `catch (e) { /* ignore */ }`. Cada catch debe loguear o
  re-lanzar, nunca silenciar.
- **NO** usar `any` para errores. Usa una union discriminada en
  `domain/<ctx>/errors.ts`.

## Tests

- **NO** mockear el módulo `application/` desde tests de Route
  Handlers — testear el handler end-to-end con el use case real y
  mockear sólo los puertos de `infrastructure/`.
- **NO** escribir tests dependientes del orden de ejecución. Cada
  `it` debe ser independiente; usa `beforeEach` para setup.
- **NO** usar `fetch` real contra servicios externos en tests
  (unit/integration). Mocks con `msw` o stubs de puerto.
- **NO** dejar tests sin el comentario `// Derived from R<x>.<y>`
  encima — `/spec-verify` los reporta como gap.
- **NO** commitear tests skipeados (`it.skip`) sin un comentario
  `TODO: <razón> + <work item>`.

## Performance / Next.js

- **NO** usar `"use client"` en componentes que sólo necesitan
  renderizar datos del server. Por default todo es RSC; el cliente
  es opt-in.
- **NO** hacer `await fetch(...)` sin pasar opciones de cache
  explícitas (`cache`, `next.revalidate`). El comportamiento default
  varía entre Server Components, Route Handlers y Server Actions.
- **NO** importar librerías pesadas (íconos completos, charting
  libs) sin `dynamic(() => import(...), { ssr: false })` cuando sólo
  son necesarias en el cliente.
- **NO** poner side effects costosos en `layout.tsx` raíz — se
  ejecuta en todas las navegaciones server.

## Información expuesta a clientes

- **NO** exponer stack traces, paths de archivos ni mensajes
  internos en responses 4xx/5xx. Mensaje genérico al cliente,
  detalle completo al log.
- **NO** diferenciar 404 "no existe" de 403 "no tienes permiso" en
  recursos privados — siempre 404.
- **NO** incluir headers `X-Powered-By` ni similares (deshabilitar
  `poweredByHeader: false` en `next.config.ts`).

## Deploy

- **Pendiente** hasta cerrar el deploy target (ver
  `stack/tech-stack.md` § Deploy target). Cuando se cierre, agregar
  aquí las restricciones específicas (canary mínimo, ventana de
  deploy, healthchecks obligatorios, etc.).

## Convenciones de código

- **NO** usar `enum` de TypeScript. Usa unions de string literales
  o `as const` objects (mejor tree-shaking, no genera código JS).
- **NO** usar `default export` salvo donde Next lo requiera
  (`page.tsx`, `layout.tsx`, `route.ts`, etc.). Named exports en
  todo lo demás para refactors confiables.
- **NO** usar `React.FC<>`. Tipa props inline:
  `function Foo({x}: {x: string}) { ... }`.
- **NO** silenciar errores de ESLint con `// eslint-disable-next-line`
  sin un comentario adicional explicando la razón.
- **NO** dejar `console.log` / `console.error` en código mergeado.
  Usar el logger de `infrastructure/observability/logger.ts`.
