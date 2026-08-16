# Hub de Arquitectura y Guía de Navegación (.docs/)

Este archivo es el **Punto Central de Enrutamiento y Gobernanza** para ingenieros humanos y **Agentes de IA**.
Define el mapa de documentación y las directrices obligatorias antes de crear o refactorizar código en el monorepo.

---

## 1. Protocolo Obligatorio para Agentes de IA y Desarrolladores

Si vas a escribir código o realizar cambios en este repositorio, debés seguir este protocolo:

1. **Regla "Issue-First" Obligatoria:** Prohibido crear ramas o escribir código sin un Issue previo en GitLab
   (ver [`governance_gitflow.md`](./governance_gitflow.md)). Todo MR y su squash commit debe referenciar `Closes #<id_issue>` en la descripción (no en cada commit atómico).
2. **Identificá la Capa y el Bounded Context:** Determiná si tu tarea corresponde a `domain`, `application`,
   `infrastructure/server`, `infrastructure/client`, `ui` o `testing`.
3. **Consultá el Archivo Específico:** Abrí y leé **únicamente** la especificación técnica asociada a tu tarea en la
   [Matriz de Enrutamiento](#2-matriz-de-enrutamiento-tarea--documento).
4. **Cumplí las Restricciones Arquitectónicas:**
   - **Cero decoradores de NestJS (`@Injectable`)** en `domain/` y `application/`.
   - **Cero dependencias de Node.js / TypeORM** en `ui/` o `infrastructure/client/`.
   - **Invariantes en Value Objects** e instanciación de agregados mediante `create()` o `fromPrimitives()`.
   - **Tags de Nx obligatorios** en `project.json` (ver [`governance_nx_boundaries.md`](./governance_nx_boundaries.md)).
   - **Nomenclatura GitFlow:** Ramas `feature/<id>-<kebab>`, `bugfix/<id>-<kebab>`, `hotfix/<id>-<kebab>`.
   - **Regla "1 Acción = 1 Permiso Dedicado (Enum)":** Cada acción de backend valida su propio `case` de enum en su DTO (class-validator/zod) + @UseGuards(ActionPermissionGuard) y en frontend cada componente interactivo se gobierna con el mismo enum para eliminar falsos positivos.
   - **Colección Bruno Obligatoria (`rest-client/api/`):** Todo endpoint creado o modificado debe tener su archivo `.bru` con bloque `docs` en Markdown.
   - **Pre-Push Gates Obligatorios:** Linter limpio (0 variables/imports sin usar), chequeo de tipos (`tsc -b`) y 100% de tests en verde antes de hacer push. Se ejecutan automáticamente vía Husky: `npx nx run-many -t lint typecheck test`.

---

## 2. Matriz de Enrutamiento (Tarea → Documento)

* **Metodología GitFlow, Scoped Labels, Convención de Commits y Releases:**
  → [`governance_gitflow.md`](./governance_gitflow.md)
* **Arquitectura global, Onion, CQRS, EDA o ciclo de eventos:**
  → [`architectural_documentation.md`](./architectural_documentation.md)
* **Clases base del núcleo (`AggregateRoot`, `ValueObject`, `Criteria`, `Result`):**
  → [`libs_shared_domain.md`](./libs_shared_domain.md)
* **Abstracciones universales de CQRS (`Command`, `Query`, `CommandBus`):**
  → [`libs_shared_application.md`](./libs_shared_application.md)
* **Buses de servidor, RabbitMQ, Criteria Converters y NestJS global:**
  → [`libs_shared_infrastructure_server.md`](./libs_shared_infrastructure_server.md)
* **Clientes HTTP (`FetchHttpClient`), storage o buses de UI:**
  → [`libs_shared_infrastructure_client.md`](./libs_shared_infrastructure_client.md)
* **Utilidades de test globales (`MotherCreator`, Faker, Mocks base):**
  → [`libs_shared_testing.md`](./libs_shared_testing.md)
* **Agregados, Value Objects, Domain Events o Puertos de Repositorio:**
  → [`libs_bounded_context_domain.md`](./libs_bounded_context_domain.md)
* **Casos de uso (Vertical Slices: Commands, Handlers, Queries, DTOs):**
  → [`libs_bounded_context_application.md`](./libs_bounded_context_application.md)
* **Esquemas TypeORM (`EntitySchema`), Mappers, Controllers o Providers:**
  → [`libs_bounded_context_infrastructure_server.md`](./libs_bounded_context_infrastructure_server.md)
* **Repositorios HTTP de cliente para consumir APIs desde frontend:**
  → [`libs_bounded_context_infrastructure_client.md`](./libs_bounded_context_infrastructure_client.md)
* **Custom Hooks, Vistas presentacionales (`*.view.tsx`) o Contenedores UI:**
  → [`libs_bounded_context_ui.md`](./libs_bounded_context_ui.md)
* **Tests unitarios con Object Mothers y Mocks del contexto:**
  → [`libs_bounded_context_testing.md`](./libs_bounded_context_testing.md)
* **Bootstrap Backend NestJS, filtros de excepción o `app.module.ts`:**
  → [`apps_api.md`](./apps_api.md)
* **Bootstrap Web React, rutas React Router o `DependencyProvider`:**
  → [`apps_web.md`](./apps_web.md)
* **Bootstrap Mobile (Expo/RN), navegación o storage nativo:**
  → [`apps_mobile.md`](./apps_mobile.md)
* **Main Process Electron, script `preload.ts` o ventana desktop:**
  → [`apps_desktop.md`](./apps_desktop.md)
* **Configuración de tags Nx y reglas `@nx/enforce-module-boundaries`:**
  → [`governance_nx_boundaries.md`](./governance_nx_boundaries.md)
* **Generadores Nx automáticos para nuevos contextos o slices:**
  → [`tools_generators.md`](./tools_generators.md)

---

## 3. Índice Estructurado de la Documentación

```text
.docs/
├── README.md                                  # [ESTE ARCHIVO] Hub de enrutamiento para humanos y agentes
├── architectural_documentation.md             # Fundamentos, principios SOLID, Onion, EDA, CQRS y taxonomía
├── governance_gitflow.md                      # GitFlow, Scoped Labels, Commits, SemVer, RC y Releases
│
├── libs_shared_domain.md                      # Núcleo compartido: AggregateRoot, VO, Criteria, DomainEvent, Result
├── libs_shared_application.md                 # Abstracciones compartidas: Command, Query, Handler, Response
├── libs_shared_infrastructure_server.md       # Servidor compartido: InMemoryBus, RabbitMQ, CriteriaConverters, NestJS
├── libs_shared_infrastructure_client.md       # Cliente compartido: HttpClient, Storage, ClientEventBus
├── libs_shared_testing.md                     # Testing compartido: MotherCreator, UuidMother, Mocks base
│
├── libs_bounded_context_domain.md             # Dominio de contexto: Agregado (User), VOs, Eventos, Puertos
├── libs_bounded_context_application.md        # Aplicación de contexto: Slices (register-user), Handlers, DTOs
├── libs_bounded_context_infrastructure_server.md # Servidor de contexto: EntitySchema, Mapper, Controllers, NestModule
├── libs_bounded_context_infrastructure_client.md # Cliente de contexto: HttpUserApiRepository, API DTOs
├── libs_bounded_context_ui.md                 # UI de contexto: useRegisterUser Hook, View y Container
├── libs_bounded_context_testing.md            # Testing de contexto: UserMother, UserRepositoryMock, Specs
│
├── apps_api.md                                # Backend NestJS: bootstrap main.ts, DomainExceptionFilter, AppModule
├── apps_web.md                                # Frontend Web: main.tsx, AppRouter, DependencyProvider
├── apps_mobile.md                             # Mobile Expo/RN: AppNavigator, AsyncStorageAdapter
├── apps_desktop.md                            # Desktop Electron: main.ts, preload.ts seguro, Renderer
│
├── governance_nx_boundaries.md                # Reglas ESLint de Enforce Module Boundaries y Tags
├── tools_generators.md                        # Generadores Nx: bounded-context y vertical-slice
│
├── rest-client/                               # Colección Bruno de la API
│   └── api/                                   # Peticiones de API
└── .husky/                                    # Git hooks
```

---

## 4. Convenciones Globales de Código

* **Identación:** Estrictamente **2 espacios** en todos los archivos TypeScript, JSON, Markdown y YAML.
* **Longitud Máxima de Línea:** **120 caracteres** por línea.
* **Final de Archivo:** Exactamente **una sola línea vacía** al final de cada fichero.
* **Nomenclatura de Archivos:**
  * Clases/Archivos: `kebab-case.tipo.ts` (ej. `user-registrar.service.ts`, `user.aggregate.ts`).
  * Tests: `*.spec.ts` para unitarios/integración, `*.test.ts` para E2E.
  * Mocks/Mothers: `*.mother.ts`, `*.mock.ts`.
