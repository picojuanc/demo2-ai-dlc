---
name: ado-link
description: Vincular un PR a uno o más work items de Azure DevOps (sólo si tracker=azure-devops)
---

# `/ado-link <pr-id> <wi-id>` — Vincular PR a work items de Azure DevOps

**Aplicabilidad**: este comando aplica **sólo si**
`repo-config.yaml > tracker: azure-devops` (§6 *Configuración del
repo*). Si el tracker es otro o `none`, **parar** y proponer el
equivalente (`/gh-link`, `/jira-link`, etc.) o reportar que no aplica.

Para PR `<pr-id>` con work items `<wi-ids>` (separados por coma):

0. **Leer `repo-config.yaml`** y verificar `tracker: azure-devops`. Si
   no lo es, parar con el mensaje arriba.
1. Verificar que el PR existe y que el repo pertenece al project
   declarado en `tracker_config.org` / `tracker_config.project`.
2. Verificar que los work items existen; si están en otro project,
   usar relación **Related** (no Parent), §6 del methodology.
3. Linkear vía `az repos pr update --id <pr-id> --work-items <wi-ids>`.
4. Si el PR description no menciona `AB#<id>`, proponer agregarlo
   (mejora la auto-trazabilidad de ADO).
5. Reportar resultado y URLs.

Requiere MCP `azure-devops` configurado (ver AGENTS.md § Servidores MCP)
o `az` CLI autenticado.
