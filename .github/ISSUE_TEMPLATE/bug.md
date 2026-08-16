---
name: Bug Report
about: Reportar un fallo o comportamiento inesperado en el sistema
title: "fix(<scope>): <resumen conciso del error>"
labels: ["type: bug", "status: todo"]
---

## 1. Descripción del Comportamiento Inesperado

<!--
Explica qué está fallando de forma clara.
-->

## 2. Pasos para Reproducir

1. Enviar petición HTTP `...` a `...`
2. Con el payload `...`
3. Observar el error `...`

## 3. Comportamiento Esperado vs Comportamiento Actual

- **Esperado:** 
- **Actual:** 

## 4. Severidad y Ámbito Afectado

- **Severidad:** `severity: blocker` | `severity: major` | `severity: minor`
- **Capa Afectada:** `layer: domain` | `layer: application` | `layer: infra-server` | `layer: infra-client` | `layer: ui`

## 5. Checklist de Corrección

- [ ] Rama creada siguiendo nomenclatura: `bugfix/<id_issue>-<nombre-kebab>` desde `develop`
- [ ] Test unitario o de integración reproduciendo el bug (Red-Green-Refactor)
- [ ] Corrección implementada sin efectos secundarios
- [ ] Pre-push gates en verde (`npx nx run-many -t lint typecheck test`)
