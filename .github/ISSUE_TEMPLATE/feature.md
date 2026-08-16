---
name: Feature Request
about: Proponer una nueva funcionalidad o caso de uso en el sistema
title: "feat(<scope>): <título descriptivo>"
labels: ["type: feature", "status: todo"]
---

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
Escenario: Registro exitoso de entidad con datos válidos
  Dado que el cliente envía datos válidos
  Cuando se procesa el comando correspondiente
  Entonces la entidad se persiste en la base de datos
  Y se emite el evento de dominio respectivo
```

- [ ] Criterio 1
- [ ] Criterio 2

## 4. Consideraciones Técnicas y Capas Afectadas

- [ ] **Dominio:** Nuevos Agregados, Value Objects, Domain Events o Puertos.
- [ ] **Aplicación:** Vertical Slice (Command/Query), Handler y DTOs.
- [ ] **Infraestructura Servidor:** Esquema TypeORM/Mongo, Mapper, Controlador NestJS.
- [ ] **Infraestructura Cliente:** Repositorio HTTP de API para frontend.
- [ ] **UI:** Custom Hook, Vistas presentacionales y Esquemas Zod.

## 5. Checklist de Entrega

- [ ] Rama creada siguiendo nomenclatura: `feature/<id_issue>-<nombre-kebab>` desde `develop`
- [ ] Tarjeta del tablero movida a `🔨 En desarrollo`
- [ ] Commits atómicos con Conventional Commits (`feat(...)`)
- [ ] Suite de pruebas unitarias cubriendo casos válidos e inválidos
- [ ] Pre-push gates en verde (`npx nx run-many -t lint typecheck test`)
