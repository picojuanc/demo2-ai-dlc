# stack/ — definición del stack y convenciones

> Esta carpeta contiene la **configuración estable** del repo: stack
> técnico, arquitectura, patrones, seguridad, restricciones, testing.
> Es el primer set de archivos que el Service Agent lee al arrancar.
>
> **Estado inicial**: todos los archivos tienen `TODO` en cada sección.
> El Service Agent (`AGENTS.md` § Bootstrap) se **negará a generar
> código** mientras estos archivos no estén completos, porque sin ellos
> los `design.md` de cada feature salen genéricos y los tests no pueden
> trazar a convenciones de naming.

## Cómo llenar esto

Dos modos:

### Modo A — conversacional (recomendado)

Abre Claude Code en este repo y di algo como:

> *"Bootstrap del stack: arranquemos por `tech-stack.md`. Quiero usar
> .NET 9 con EF Core y Postgres."*

El Service Agent te va a hacer preguntas concretas, una a la vez, y a
escribir los archivos a medida que decides.

### Modo B — manual

Edita cada archivo de esta carpeta directamente. Cuando termines, di al
Service Agent: *"el stack está completo, sustituye los placeholders en
AGENTS.md"*.

## Orden recomendado

1. `tech-stack.md` — qué lenguaje / framework / BD / runtime usas.
2. `architecture.md` — qué patrón arquitectónico (Clean Arch, Hexagonal,
   MVC, etc.).
3. `patterns.md` — naming, formato de commits, organización de tests.
4. `security.md` — auth, secretos, PII, compliance.
5. `constraints.md` — qué está prohibido (anti-patrones de **este**
   proyecto).
6. `testing.md` — niveles de test, cobertura mínima, herramientas.

Cada archivo bloquea al siguiente: no tiene sentido decidir `patterns.md`
antes de saber qué framework usas.
