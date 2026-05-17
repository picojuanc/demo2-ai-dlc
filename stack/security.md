# Security

> **Servicio**: `demo2-ai-dlc`
> **Estado**: completado durante bootstrap. El Security Agent
> (consultivo, §7) revisa specs y PRs contra este archivo.

## Autenticación

- **Mecanismo**: JWT firmado, transportado en **cookie HTTP**:
  - `httpOnly`, `Secure`, `SameSite=Lax`, `Path=/`.
  - Nombre de cookie: `dad_session` (`d`emo `a`i `d`lc).
- **Emisor**: este mismo servicio emite y verifica.
- **Algoritmo de firma**: **HS256** con secret compartido.
  - Secret: env `JWT_SIGNING_SECRET`, mínimo 32 bytes random,
    inyectado por Sealed Secrets (ver § Secretos).
  - **Riesgo conocido**: HS256 con secret simétrico significa que
    cualquier proceso con acceso al secret puede emitir tokens
    válidos. Por eso este secret **nunca se comparte con otro
    servicio**. Si en el futuro otro servicio necesita verificar,
    migramos a RS256 + JWKS (ADR obligatorio).
- **Claims obligatorios**:
  - `sub` (user id, opaco — no email)
  - `iss` (`demo2-ai-dlc`)
  - `aud` (`demo2-ai-dlc`)
  - `iat`, `exp`
  - `roles` (lista de strings, ver § Autorización)
- **TTL del access token**: 60 minutos (ajustable vía
  `JWT_TTL_SECONDS`).
- **Refresh**: por ahora **re-login** cuando expira (sin refresh
  token). Cuando aparezca la primera feature que lo requiera,
  agregamos refresh token rotativo en cookie separada y se actualiza
  este archivo.
- **Validación**: en `middleware.ts` (Edge runtime) para rutas
  protegidas, usando `jose` (`jwtVerify` con la misma secret). Falla
  → redirect a `/login` (UI) o `401` (API).
- **Logout**: endpoint `POST /api/auth/logout` borra la cookie
  (`Set-Cookie: dad_session=; Max-Age=0`).

### Reglas no negociables sobre el JWT

- **Nunca** aceptar `alg: none`. La verificación se hace con
  `jose.jwtVerify(token, secret, { algorithms: ["HS256"] })`
  explícito.
- **Nunca** poner el token en `localStorage` ni en headers visibles
  para JS — siempre cookie httpOnly.
- **Nunca** loguear el token completo. Si hay que debuggear, loguear
  sólo `claims.jti` (si está) o un hash truncado.

## Autorización

- **Modelo**: **RBAC simple** sobre claim `roles` del JWT.
- **Roles iniciales**: por definir cuando exista la primera feature;
  formato `roles: ["admin", "user", ...]`.
- **Dónde se aplica**:
  - **Coarse**: en `middleware.ts` por route group (ej. `(admin)/*`
    requiere `roles.includes("admin")`).
  - **Fine**: dentro del use case, comparando `claims.roles` contra
    la operación. Use cases sensibles reciben las claims como
    parámetro tipado, no leen la cookie directamente.
- **Granularidad**: nivel de endpoint y nivel de use case. No hay
  permisos por recurso (no hay ReBAC) — si una feature futura lo
  necesita, abrir ADR y reevaluar pasar a Cerbos / OPA.

## Secretos

- **Storage en prod**: **Sealed Secrets** en OpenShift (default del
  methodology §14). Cada secret se manifiesta como
  `SealedSecret` en el repo de GitOps, se descifra dentro del
  cluster y se monta como env var del pod.
- **Storage en dev**: archivo `.env.local` en la raíz (gitignored,
  incluido en `.gitignore` desde día cero). Plantilla pública:
  `.env.example` con keys y descripciones, valores dummy.
- **Rotación**:
  - `JWT_SIGNING_SECRET`: cada 90 días o ante incidente. Procedimiento:
    publicar nuevo secret bajo `JWT_SIGNING_SECRET_NEXT`, aceptar
    ambos durante 1× el TTL del JWT (60 min), luego promover y
    eliminar el viejo.
  - Cualquier otro secret de integración: según política del proveedor.
- **Acceso desde código**: leído **una sola vez** al boot en
  `src/infrastructure/config/env.ts` (parseado y validado con Zod).
  El resto del código consume el objeto `env` tipado, nunca
  `process.env.X` directo.
- **Reglas no negociables**:
  - **NUNCA** secrets hardcoded en código ni en tests.
  - **NUNCA** secrets en logs (ni siquiera primeros/últimos chars).
  - **NUNCA** commitear `.env.local`.
  - `.env.example` no contiene valores reales, sólo placeholders.

## PII y residencia de datos

- **PII manejada hoy**: ninguna persistida (el servicio es stateless).
  - Lo único que toca el servicio en runtime es el JWT (que contiene
    `sub` opaco, no email).
- **Política**: no se introduce PII sin actualizar este archivo y
  crear un ADR. Si una feature futura requiere manejar email, nombre,
  documento, ubicación o cualquier categoría regulada, **se escala al
  Security Agent antes de implementar**.
- **Región**: heredada del cluster de deploy (a decidir).
- **Retention** / **anonymization**: N/A mientras no haya persistencia.

## Compliance

- **Marcos aplicables hoy**: ninguno específico declarado.
- **Baseline obligatorio igualmente** (aplica aunque no haya marco
  formal):
  - No PII en logs ni en error responses.
  - TLS 1.2+ obligatorio en todo tráfico (in transit).
  - Secrets fuera de código.
  - Stack traces no expuestos a clientes externos.
- Cuando aparezca el primer cliente regulado, se actualiza este
  archivo con el marco que aplica y se hace gap analysis.

## Encriptación

- **At rest**: N/A (sin persistencia). Si un día se introduce, default
  es encriptación de disco completo a nivel del provider + columna
  específica para campos sensibles.
- **In transit**:
  - TLS 1.2+ obligatorio (TLS 1.3 preferido).
  - HSTS habilitado en producción (header
    `Strict-Transport-Security: max-age=31536000; includeSubDomains`).
  - Cookies con `Secure` (sólo HTTPS).
- **Algoritmos aprobados para JWT**: HS256 (hoy), RS256 / ES256 (si
  migramos a IdP externo). Prohibido HS384/HS512 sin ADR (no aportan
  sobre HS256 con secret de 32 bytes).

## Headers de seguridad (API HTTP)

Configurados centralmente en `middleware.ts` y aplicados a toda
respuesta:

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY` (no se permite embebido)
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Content-Security-Policy`: política restrictiva, `default-src
  'self'`, sin `unsafe-inline` salvo nonce para hidratación de Next.
- `Permissions-Policy`: deshabilita features no usadas
  (camera/microphone/geolocation).
- **CORS**: por default **bloqueado**. Si se necesita exponer a otro
  origen, definir allowlist explícita por endpoint (no global `*`).

## Rate limiting

- **Estado**: no implementado todavía (sin persistencia ni cache).
- **Default cuando se agregue**: límite por IP en endpoints
  no autenticados (`/api/auth/login`), implementado vía middleware
  con backing store en Redis. Por ahora, asumimos que el WAF /
  ingress del cluster maneja DDoS de capa 7.
- Cuando aparezca un endpoint sensible (login, password reset),
  rate limiting se vuelve **bloqueante** y se introduce Redis.

## Validación de entrada

- **Patrón**: **Zod** para todo input externo:
  - Bodies de Route Handlers: `schema.safeParse(await req.json())`.
  - Query params y headers: `schema.safeParse(...)`.
  - Env vars: validadas al boot en `src/infrastructure/config/env.ts`.
- **Falla de validación** → `400 Bad Request` con cuerpo:
  `{ error: "validation", issues: [...] }`. Sin filtrar paths
  internos ni stack traces.
- **Sanitización HTML**: cuando aparezca contenido renderizable
  proveniente del usuario, usar `DOMPurify` server-side. Hoy no
  aplica.

## Información expuesta a clientes

- **Stack traces**: nunca al cliente. En prod, errores no manejados
  → response genérica `500 { error: "internal" }` + log estructurado
  completo en backend.
- **Mensajes de error**: genéricos para fallas de auth (no revelar
  si el user existe vs. password incorrecto — siempre
  "credenciales inválidas").
- **404 vs 403**: para recursos privados, retornar **404** ante
  recurso inexistente Y ante recurso al que el user no tiene
  acceso (no revelar existencia).

## Anti-patrones críticos de seguridad

Los siguientes son **PROHIBIDOS** absolutos. Cualquier PR que los
viole se rechaza automáticamente:

- **NUNCA** loguear el JWT completo, ni passwords, ni cualquier PII.
- **NUNCA** secrets hardcoded en código, fixtures ni tests.
- **NUNCA** exponer stack traces o paths internos en responses HTTP.
- **NUNCA** aceptar `alg: none` o algoritmos no whitelistados en
  verificación de JWT.
- **NUNCA** guardar el token de sesión en `localStorage` o
  `sessionStorage`.
- **NUNCA** desactivar `httpOnly` / `Secure` / `SameSite` en la
  cookie de sesión.
- **NUNCA** introducir CORS `*` global.
- **NUNCA** lanzar a producción con `JWT_TTL_SECONDS` > 24h sin
  refresh token rotativo.

## Auditoría

- Logs estructurados a `stdout` (recogidos por la plataforma).
- Eventos de seguridad obligatorios a loguear con `level: warn` o
  `error`:
  - Login exitoso / fallido (sin revelar password ni token).
  - Logout.
  - Acceso denegado por authz (401/403).
  - Cualquier excepción de verificación de JWT.
- Cuando exista persistencia de eventos auditables, abrir ADR
  específico.
