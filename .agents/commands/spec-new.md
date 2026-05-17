---
name: spec-new
description: Iniciar una feature spec con entrevista guiada
---

# `/spec-new <feature-slug>` — Iniciar una feature spec

Sigue el protocolo §7 (AGENTS.md). Para la feature `<feature-slug>`:

1. **CONTEXT** — verificar:
   - Repo actual y rama de trabajo.
   - **`stack/` completo**: si algún archivo de `stack/` aún tiene
     `TODO`, **parar** y proponer `Bootstrap` (ver AGENTS.md § Bootstrap)
     antes de iniciar la spec.
   - ¿La feature pertenece a una Initiative? Si sí, pedir URL o slug
     (recordar: Initiative es opcional, §6 del methodology).
   - ¿Hay un PR de requerimiento del cliente, work item de origen,
     conversación previa relevante?

2. **WORKTREE** — preparar el espacio de trabajo aislado (§6 del
   methodology *Configuración del repo* + *Worktree, ramas y flujo de
   promoción*):
   - **Leer** `repo-config.yaml` y obtener las ramas declaradas en
     `environments[].branch`. Si el archivo no existe, **parar** y
     proponer crearlo antes de seguir (no asumir `pruebas/qa/main`
     por reflejo).
   - **Preguntar** la rama base ofreciendo **sólo** las ramas
     declaradas (default = la primera de `promotion_path`).
   - **Proponer** crear:
     `git worktree add -b feat/<feature-slug> ../<repo>--<feature-slug> origin/<base>`
     y pedir OK antes de ejecutar (acción reversible pero observable).
   - Tras crear, **verificar** que el `cwd` quedó en el worktree nuevo
     antes de continuar.

3. **CLARIFY** — entrevista guiada, **una pregunta a la vez**:
   - ¿Cuál es el problema que resuelve esta feature? ¿Quién es el
     usuario primario?
   - ¿Cuáles son los criterios de éxito **observables**? (forzar NFRs
     medibles — "rápido" no vale; "p99 < 500ms" sí)
   - ¿Restricciones legales / compliance / residencia de datos? (cruzar
     con `stack/security.md`)
   - ¿Toca otros servicios? ¿De qué equipos? Si sí, **escalar al
     Architect Agent** (§7 del methodology).
   - ¿Depende de algo que aún no existe (SP, endpoint, librería,
     componente de diseño)? — futuras `D-N` (§6).
   - ¿Cómo se prueba cada R*.* (unit / integration / e2e / contract /
     load / accessibility)? Cruzar con `stack/testing.md`.
   - Si no hay respuesta clara, marcar `OPEN_QUESTION` en la spec —
     **NO inventar** (§3.12 del methodology).

4. **PROPOSE** la estructura inicial; pedir OK antes de escribir.

5. **EXECUTE** — crear `specs/<feature-slug>/`:
   - `requirements.md` (EARS R1.1, R1.2... + Dependencies si aplica +
     Tests strategy por R*.*)
   - `design.md` (esqueleto con secciones obligatorias, a llenar tras
     aprobación de requirements). Aplicar `stack/architecture.md`.
   - `tasks.md` (vacío hasta que design esté firmado)
   - `status.md` (state: not-started, todas tasks pending — §6
     Lifecycle del methodology)

6. **CLOSE** — reportar qué se creó, qué `OPEN_QUESTION` quedan
   abiertas (bloquean aprobación), y siguiente paso sugerido.

NO escribir código de producción. Esperar aprobación de la spec.
