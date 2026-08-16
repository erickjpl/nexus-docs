---
name: Hotfix Report
about: Incidente crítico en producción que requiere corrección y release inmediato
title: "hotfix(<scope>): <título crítico>"
---

<!--
GUÍA DE LABELS PARA ESTA ISSUE:
- type::hotfix
- layer::[domain | application | infra-server | infra-client | ui]
- status::in-progress
- priority::critical
- severity::blocker
- scope::[users | shared | api | web | mobile | desktop]
-->

## 1. Resumen del Incidente en Producción

<!--
Explicación del fallo crítico que está afectando actualmente a los usuarios o la operación del sistema.
-->

## 2. Impacto en el Negocio / Usuarios

- **Usuarios / Cuentas afectadas:** [ej. Todos los usuarios que intentan registrarse]
- **Servicios caídos o degradados:** [ej. Endpoint de autenticación]

## 3. Plan de Mitigación y Corrección Rápida

<!--
¿Cuál es la solución puntual para restaurar el servicio sin generar efectos colaterales?
-->

## 4. Checklist de Emergencia

- [ ] Rama creada a partir de `main`: `hotfix/<id_issue>-<nombre-kebab>`
- [ ] Corrección validada localmente con suite de tests
- [ ] MR abierto hacia `main` (para release inmediato con nuevo Patch Tag)
- [ ] MR abierto hacia `develop` (para sincronizar la corrección con el equipo)
- [ ] Tag de versión parche generado (ej. `v1.2.1`)
