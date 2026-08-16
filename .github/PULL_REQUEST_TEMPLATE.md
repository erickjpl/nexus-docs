## 📌 Resumen de Cambios

<!--
Describe brevemente qué cambios introduce este Pull Request y qué problema resuelve.
-->

Closes #<!-- id del issue -->

---

## 🏗️ Capas Modificadas

- [ ] `domain` (Agregados, Value Objects, Domain Events, Puertos)
- [ ] `application` (Commands, Queries, Handlers, DTOs de respuesta)
- [ ] `infrastructure/server` (NestJS, Controladores, TypeORM, Repositorios)
- [ ] `infrastructure/client` (Clientes HTTP, Storage, Adaptadores)
- [ ] `ui` (Componentes, Vistas, Hooks, Schemas Zod)
- [ ] `testing` (Object Mothers, Mocks)

---

## 🧪 Verificación de Calidad

- [ ] Linter ejecutado y limpio (`npx nx run-many -t lint`)
- [ ] Chequeo estricto de tipos (`npx nx run-many -t typecheck`)
- [ ] 100% de tests pasando (`npx nx run-many -t test`)
- [ ] Colección Bruno actualizada en `rest-client/` (si aplica a endpoints de API)
- [ ] Tarjeta de GitHub Projects movida a `🧪 Probando (testing)` / `🚀 Desarrollado`
