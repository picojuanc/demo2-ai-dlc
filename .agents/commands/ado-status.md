---
name: ado-status
description: Estado de un pipeline de Azure DevOps (sólo si tracker=azure-devops o runtime usa ADO Pipelines)
---

# `/ado-status <pipeline-id>` — Estado de un pipeline de Azure DevOps

**Aplicabilidad**: aplica **sólo si**
`repo-config.yaml > tracker: azure-devops` **o** el CI/CD del repo
corre en ADO Pipelines. Si no aplica, proponer el equivalente
(`/gh-status`, etc.) o reportar consulta manual.

Para pipeline `<pipeline-id>`:

0. **Leer `repo-config.yaml`** y verificar que ADO Pipelines aplica.
1. `az pipelines runs list --pipeline-id <pipeline-id> --top 5`.
2. Reportar: última run (status + duración), stage donde falló, link
   al log y al PR asociado.
3. Si el pipeline está vinculado a una feature por convención de
   commit `AB#<id>`, cruzar con `status.md` y decir si el deploy del
   último commit `done` ya está reflejado.

Requiere MCP `azure-devops` configurado o `az` CLI autenticado.
