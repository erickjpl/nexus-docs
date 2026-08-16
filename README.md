# Nexus Architecture & Monorepo Blueprint

Nexus es la arquitectura canónica de referencia empresarial para monorepos multi-aplicación construidos sobre **Nx**, **Clean Architecture**, **Domain-Driven Design (DDD)** táctico, **CQRS** (Command Query Responsibility Segregation) y **TypeScript estricto** en todas sus capas.

---

## 🏛️ Principios y Fundamentos Arquitectónicos

- **Onion / Clean Architecture:** Separación estricta de 4 capas concéntricas:
  1. **Domain (`libs/{context}/domain`):** Entidades, Agregados, Value Objects, Domain Events y Puertos. Cero dependencias externas.
  2. **Application (`libs/{context}/application`):** Casos de uso estructurados como Vertical Slices (Commands, Handlers, Queries, Responses).
  3. **Infrastructure Server (`libs/{context}/infrastructure/server`):** Módulos NestJS, Controladores HTTP de acción única, Esquemas TypeORM/Mappers y Bus Adapters.
  4. **Infrastructure Client & UI (`libs/{context}/infrastructure/client` & `libs/{context}/ui`):** Repositorios HTTP con `FetchHttpClient`, Gestión de Estado con TanStack Query, Formularios Zod y UI Wrappers.
- **Shared Kernel transversal:** Abstracciones reutilizables en `libs/shared/*` (`domain`, `application`, `infrastructure/server`, `infrastructure/client`, `testing`).
- **Gobernanza Automatizada de Fronteras en Nx:** Restricciones estrictas por tags (`type:*`, `scope:*`) validadas vía `@nx/enforce-module-boundaries`.
- **Regla de Oro de Seguridad:** **"1 Acción = 1 Permiso Dedicado (Enum)"** tanto en backend (`@RequirePermission` + `ActionPermissionGuard`) como en frontend (`hasPermission(PermissionEnum.ACTION)`).
- **Documentación Viva de APIs:** Colecciones Bruno (`rest-client/api/`) con documentación Markdown integrada para cada endpoint.

Para consultar las instrucciones obligatorias y el protocolo de ejecución para IA y desarrolladores, ver [`AGENTS.md`](AGENTS.md).  
Para explorar el catálogo exhaustivo de especificaciones técnicas, ver [`.docs/README.md`](.docs/README.md).

---

## 📁 Estructura del Monorepo

```text
nexus-monorepo/
├── apps/
│   ├── api/                   # Backend NestJS 11 (API Gateway & Composition Root)
│   ├── web/                   # Frontend React 19 + Vite
│   ├── mobile/                # Mobile React Native / Expo
│   └── desktop/               # Desktop Electron
├── libs/
│   ├── {bounded_context}/     # Bounded Contexts (domain, application, infra/server, infra/client, ui, testing)
│   └── shared/                # Shared Kernel universal
├── rest-client/               # 🚀 Colecciones vivas de Bruno (.bru)
├── .docs/                     # 📚 Documentación técnica exhaustiva por capa
├── .github/                   # ⚙️ Workflows CI y plantillas de Issues/PRs
└── AGENTS.md                  # 🤖 Protocolo y directrices para Agentes de IA
```

---

## 🚀 Comandos Operativos y de Verificación

```bash
# Ejecutar suite de pruebas en todo el monorepo
npx nx run-many -t test

# Chequeo estricto de tipos TypeScript
npx nx run-many -t typecheck

# Linter y validación de fronteras arquitectónicas Nx
npx nx run-many -t lint

# Formato de código con ESLint (autoridad única de estilo)
npx eslint . --fix

# Ejecutar pre-push gates completos (Husky)
npx nx run-many -t lint typecheck test
```
