<!-- nx configuration start-->
<!-- Leave the start & end comments to automatically receive updates. -->

# General Guidelines for working with Nx

- For navigating/exploring the workspace, code generation, running tasks, or package linking, invoke the unified `nx` skill located in `.agents/skills/nx/SKILL.md`.
- When running tasks (for example build, lint, test, e2e, etc.), always prefer running the task through `nx` (i.e. `npx nx run`, `npx nx run-many`, `npx nx affected`) instead of using underlying tooling directly.
- Prefix nx commands with the workspace's package manager (`npx nx` or `npm exec nx`) to avoid using globally installed CLI.
- You have access to the Nx MCP server and its tools, use them to help the user.
- For Nx plugin best practices, check `node_modules/@nx/<plugin>/PLUGIN.md`. Not all plugins have this file - proceed without it if unavailable.
- NEVER guess CLI flags - always check `nx_docs` or `--help` first when unsure.

## Scaffolding & Generators

- For scaffolding tasks (creating apps, libs, project structure, setup), ALWAYS invoke the `nx` skill FIRST before exploring or calling MCP tools.
- Use `--no-interactive` and run with `--dry-run` first to preview changes.

## When to use nx_docs

- USE for: advanced config options, unfamiliar flags, migration guides, plugin configuration, edge cases.
- DON'T USE for: basic generator syntax (`nx g @nx/react:app`), standard commands, things you already know.
- The `nx` skill handles generator discovery internally - don't call nx_docs just to look up generator syntax.

<!-- nx configuration end-->

---

# Nexus Monorepo — Instrucciones Obligatorias para Agentes de IA

Este archivo es el **Punto de Entrada y Gobernanza Principal** para cualquier Agente de Inteligencia Artificial.
Todas las reglas arquitectónicas, patrones de diseño y especificaciones técnicas detalladas residen en la carpeta [`.docs/`](.docs/README.md). **Es obligatorio consultar `.docs/` antes de escribir o modificar código.**

---

## 1. Protocolo de Ejecución Obligatorio

Antes de crear, refactorizar o modificar cualquier archivo en este repositorio, el agente **DEBE** seguir estos pasos sin excepción:

1. **Regla "Issue-First" y Tablero de GitLab Obligatorios:** Prohibido crear ramas o escribir código sin un Issue previo en GitLab (ver [`.docs/governance_gitflow.md`](.docs/governance_gitflow.md)). Las tarjetas del Issue Board **DEBEN** transicionar formalmente en cada etapa mediante Scoped Labels: `status::backlog` ➔ `status::to-do` ➔ `status::in-progress` (al crear rama) ➔ `status::in-review` (al abrir MR y ejecutar suite de tests/pre-push gates) ➔ `status::ready-for-release` (al fusionar MR a `develop`) ➔ `Closed / Done` (merge a `main` con Tag de Release).
2. **Identificar la Capa y el Bounded Context:** Determinar si la tarea corresponde a `domain`, `application`, `infrastructure/server`, `infrastructure/client`, `ui` o `testing`.
3. **Consultar la Documentación Específica en `.docs/`:** Abrir y leer la especificación técnica asociada a la tarea en la [Matriz de Enrutamiento](#2-matriz-de-enrutamiento-tarea--especificación-en-docs) antes de implementar.
4. **Cumplir las Restricciones Arquitectónicas Fundamentales:**
   - **Cero decoradores de NestJS (`@Injectable`)** en `domain/` y `application/`.
   - **Cero dependencias de Node.js / TypeORM / Servidor** en `ui/` o `infrastructure/client/`.
   - **Invariantes en Value Objects** e instanciación de agregados exclusivamente mediante `create()` o `fromPrimitives()`.
   - **Acceso exclusivo al dominio mediante Aggregate Roots.** Las entidades externas jamás manipulan el estado interno de un agregado directamente.
   - **Tags de Nx obligatorios** en `project.json` (ver [`.docs/governance_nx_boundaries.md`](.docs/governance_nx_boundaries.md)).
   - **Regla "1 Acción = 1 Permiso Dedicado (Enum)":** Cada endpoint en backend valida su propio `case` de enum tipado (`PermissionEnum.ACTION`) con `@UseGuards(ActionPermissionGuard)` y en frontend cada componente interactivo se gobierna con el mismo enum.
   - **Colección Bruno Obligatoria (`rest-client/api/`):** Todo endpoint creado o modificado debe tener su archivo `.bru` con bloque `docs` en Markdown.
   - **Pre-Push Gates Obligatorios:** Linter limpio, chequeo de tipos (`npx nx run-many -t typecheck`) y 100% de tests en verde (`npx nx run-many -t test`).
   - **Gobernanza de CHANGELOG:** Todo cambio que se integre a release debe actualizar `CHANGELOG.md` siguiendo el estándar Keep a Changelog.

---

## 2. Matriz de Enrutamiento (Tarea → Especificación en `.docs/`)

| Tarea a Implementar / Modificar                                                          | Documento de Especificación Obligatorio                                                                      |
| :--------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| **Metodología GitFlow, Commits, Scoped Labels de GitLab y Releases**                     | [`.docs/governance_gitflow.md`](.docs/governance_gitflow.md)                                                 |
| **Fundamentos de Arquitectura Onion, EDA, CQRS y Ciclo de Eventos**                      | [`.docs/architectural_documentation.md`](.docs/architectural_documentation.md)                               |
| **Clases base del Shared Kernel (`AggregateRoot`, `ValueObject`, `Criteria`, `Result`)** | [`.docs/libs_shared_domain.md`](.docs/libs_shared_domain.md)                                                 |
| **Abstracciones CQRS compartidas (`Command`, `Query`, `CommandBus`, `QueryBus`)**        | [`.docs/libs_shared_application.md`](.docs/libs_shared_application.md)                                       |
| **Buses de servidor, RabbitMQ, Criteria Converters y NestJS global**                     | [`.docs/libs_shared_infrastructure_server.md`](.docs/libs_shared_infrastructure_server.md)                   |
| **Clientes HTTP (`FetchHttpClient`), Storage (`LocalStorageAdapter`) y UI Wrappers**     | [`.docs/libs_shared_infrastructure_client.md`](.docs/libs_shared_infrastructure_client.md)                   |
| **Utilidades de test globales (`MotherCreator`, Faker, Mocks base)**                     | [`.docs/libs_shared_testing.md`](.docs/libs_shared_testing.md)                                               |
| **Agregados, Value Objects, Domain Events y Puertos de Repositorio de Contexto**         | [`.docs/libs_bounded_context_domain.md`](.docs/libs_bounded_context_domain.md)                               |
| **Casos de uso (Vertical Slices: Commands, Handlers, Queries, DTOs)**                    | [`.docs/libs_bounded_context_application.md`](.docs/libs_bounded_context_application.md)                     |
| **Esquemas TypeORM (`EntitySchema`), Mappers, Controladores NestJS y Modules**           | [`.docs/libs_bounded_context_infrastructure_server.md`](.docs/libs_bounded_context_infrastructure_server.md) |
| **Repositorios HTTP de cliente para consumir APIs desde Frontend**                       | [`.docs/libs_bounded_context_infrastructure_client.md`](.docs/libs_bounded_context_infrastructure_client.md) |
| **Custom Hooks, Vistas (`*.view.tsx`), Componentes y Esquemas Zod en UI**                | [`.docs/libs_bounded_context_ui.md`](.docs/libs_bounded_context_ui.md)                                       |
| **Tests unitarios con Object Mothers y Mocks del Bounded Context**                       | [`.docs/libs_bounded_context_testing.md`](.docs/libs_bounded_context_testing.md)                             |
| **Bootstrap Backend NestJS, `main.ts`, `DomainExceptionFilter` y `AppModule`**           | [`.docs/apps_api.md`](.docs/apps_api.md)                                                                     |
| **Bootstrap Frontend Web React, `main.tsx`, `AppRouter` y `RepositoryProvider`**         | [`.docs/apps_web.md`](.docs/apps_web.md)                                                                     |
| **Bootstrap Mobile React Native, navegación y almacenamiento nativo**                    | [`.docs/apps_mobile.md`](.docs/apps_mobile.md)                                                               |
| **Bootstrap Desktop Electron, proceso principal e IPC handlers**                         | [`.docs/apps_desktop.md`](.docs/apps_desktop.md)                                                             |
| **Fronteras de Módulos Nx y Reglas ESLint (`@nx/enforce-module-boundaries`)**            | [`.docs/governance_nx_boundaries.md`](.docs/governance_nx_boundaries.md)                                     |
| **Generadores Nx automáticos para nuevos Bounded Contexts o Vertical Slices**            | [`.docs/tools_generators.md`](.docs/tools_generators.md)                                                     |

---

## 3. Convenciones y Comandos Operativos de Nx

- **Generación de Código:** Usar siempre `npx nx generate` con `--no-interactive` y `--dry-run` previo.
- **Validación Continua:** Ejecutar `npx nx test <proyecto>` y `npx nx run-many -t test,typecheck,lint` tras cada modificación.
- **Sincronización:** Ejecutar `npx nx sync` si cambian referencias de proyectos TypeScript.
- **Formato:** ESLint es la única autoridad de formato (`npx eslint . --fix`): 2 espacios de indentación, 120 caracteres max, punto y coma, comillas simples, llaves de clase Allman y un argumento/parámetro por línea para 2+.
