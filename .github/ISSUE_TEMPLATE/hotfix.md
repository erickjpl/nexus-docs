---
name: Hotfix Crítico
about: Corrección urgente bloqueante en entorno de producción
title: "fix(<scope>): <corrección urgente>"
labels: ["type: hotfix", "priority: critical", "severity: blocker", "status: in-progress"]
---

## 1. Impacto en Producción

<!--
Describe la afectación a usuarios, transacciones o datos.
-->

## 2. Causa Raíz Identificada

<!--
Explicación técnica de la causa raíz.
-->

## 3. Plan de Mitigación / Solución

<!--
Solución mínima requerida para estabilizar producción.
-->

## 4. Checklist del Proceso Hotfix

- [ ] Rama creada desde `main`: `hotfix/<id_issue>-<nombre-kebab>`
- [ ] Test unitario que valida la corrección
- [ ] PR 1 abierto hacia `main` con `Closes #<id_issue>`, aprobado y fusionado con squash
- [ ] Tag inmutable emitido en `main`
- [ ] PR 2 abierto hacia `develop` (o retro-merge de `main` a `develop`)
- [ ] Rama `hotfix/` eliminada tras completar ambos merges
