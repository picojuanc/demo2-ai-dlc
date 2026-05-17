---
name: spec-amend
description: Cambio de spec post-aprobación (cliente, regulación, negocio)
---

# `/spec-amend <feature-slug> --reason "<motivo>"` — Cambio de spec post-aprobación

Para feature `<feature-slug>` con motivo `<motivo>`:

1. Leer `requirements.md`, `tasks.md`, `status.md` actuales.
2. Identificar qué `R*.*` y tasks están potencialmente afectadas (proponer, NO decidir en solitario).
3. Confirmar con el usuario el alcance final del cambio.
4. Editar `requirements.md`:
   - `R*.*` que dejan de aplicar se marcan ~~tachadas~~ (no se borran).
   - `R*.*` que cambian se reescriben in-place.
   - `R*.*` nuevas se añaden con la siguiente numeración disponible.
5. Editar `tasks.md`: tasks que dejan de aplicar → `cancelled`; tasks
   que cambian → modificadas; tasks nuevas → al final, ordenadas por
   dependencia.
6. Anotar el evento en `amendments.md` (crear si no existe):

   ```
   ## AMD-NNN — <título corto> (<fecha>)
   - Motivo: <descripción + fuente: cliente / legal / negocio>
   - Autor: <quién lo dictó> vía <quién lo registró>
   - R*.* afectadas: <lista>
   - Tasks afectadas: <lista>
   - PR de spec: !<id>
   - PR de implementación: !<id>
   ```

7. Los commits posteriores citan `AMD-NNN` además de `R*.*`.

Un Amendment **NO** es un bug Tipo B. Tipo B son cosas que estaban mal
desde el inicio; un Amendment es un evento nuevo posterior a la
aprobación. Mantener la distinción mejora la métrica de calidad de
spec authoring.
