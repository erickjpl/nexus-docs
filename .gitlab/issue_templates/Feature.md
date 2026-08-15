---
name: Feature Request
about: Proponer una nueva funcionalidad o caso de uso en el sistema
title: "feat(<scope>): <título descriptivo>"
---

<!--
GUÍA DE LABELS PARA ESTA ISSUE:
- type::feature
- layer::[domain | application | infra-server | infra-client | ui]
- status::to-do
- priority::[critical | high | medium | low]
- scope::[users | shared | api | web | mobile | desktop]
-->

## 1. Descripción de la Funcionalidad

<!--
Explica de forma clara y concisa qué debe hacer esta nueva funcionalidad desde la perspectiva del usuario o del sistema.
-->

## 2. Contexto y Motivación de Negocio

<!--
¿Qué problema resuelve? ¿Por qué es necesaria para el negocio o la arquitectura?
-->

## 3. Criterios de Aceptación (Formato Gherkin / BDD)

```gherkin
Escenario: Registro exitoso de usuario con datos válidos
  Dado que el usuario envía un ID, nombre y email válidos
  Cuando se procesa el comando de registro
  Entonces el usuario se persiste en la base de datos
  Y se emite el evento de dominio 'user.registered'
```

- [ ] Criterio 1
- [ ] Criterio 2

## 4. Consideraciones Técnicas y Capas Afectadas

- [ ] **Dominio:** Nuevos Agregados, Value Objects, Domain Events o Puertos.
- [ ] **Aplicación:** Vertical Slice (Command/Query), Handler y DTOs.
- [ ] **Infraestructura Servidor:** Esquema TypeORM/Mongo, Mapper, Controlador NestJS.
- [ ] **Infraestructura Cliente:** Repositorio HTTP de API para frontend.
- [ ] **UI:** Custom Hook, Contenedor y Vista presentacional.

## 5. Checklist de Entrega

- [ ] Rama creada siguiendo nomenclatura: `feature/<id_issue>-<nombre-kebab>`
- [ ] Commits atómicos con Conventional Commits (`feat(...)`) y `Closes #<id_issue>`
- [ ] Suite de pruebas unitarias cubriendo casos válidos e inválidos
- [ ] Linter y Boundaries de Nx validados (`npx nx affected -t lint test`)
