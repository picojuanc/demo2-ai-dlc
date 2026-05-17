---
name: bug-triage
description: Clasificar un bug en la taxonomía A/B/C/D/E (§8) con entrevista
---

# `/bug-triage <descripción>` — Clasificar bug en taxonomía A/B/C/D/E

Sigue el protocolo §7 (AGENTS.md). Para el bug descrito en `<descripción>`:

1. **CONTEXT** — verificar: en qué feature/spec aparece; repro estable
   o intermitente; ¿ya hay `BUG-NNN` abierto con síntomas similares?

2. **CLARIFY** — entrevistar al reportero: qué se esperaba vs qué
   pasó; si la spec cubre el caso explícitamente (cita `R*.*`); si es
   cambio externo o defecto técnico; si la dependencia es 3rd party.

3. **PROPOSE** clasificación:
   - **A** — spec cubre el caso y el código está mal. Regression test
     + fix. **No tocar spec.**
   - **B** — spec NO cubre el caso (gap). Nuevo `R*.*` en
     `requirements.md` antes del fix.
   - **C** — spec lo cubre pero es ambigua. Refinar `requirements.md`
     + posible fix.
   - **D** — incidente en prod con SLA roto. Hotfix directo + spec
     retroactiva en post-mortem.
   - **E** — causa raíz es paquete / SaaS 3rd party. Reportar al
     vendor. Estrategia: `WORKAROUND` / `PIN` / `WAIT`. La `R*.*`
     afectada queda `blocked_by: ext:<id>` en `status.md`.
   - **Amendment** — no es bug; es cambio externo. Redirigir a
     `/spec-amend` (no contaminar la métrica de Tipo B).
   - Pedir confirmación al reportero.

4. **EXECUTE** — registrar en `bugs.md` (formato §8 Tracking):
   `BUG-NNN`, tipo, requirement afectado, fecha, reportero. Para Tipo
   B/C: abrir PR de spec antes del fix. Para Tipo E: marcar
   `blocked_by: ext:<id>` en `status.md`.

5. **CLOSE** — siguiente paso sugerido (PR de spec, fix directo,
   workaround, escalación a vendor, etc.).
