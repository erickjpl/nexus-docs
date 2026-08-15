# Architecture Review — nexus-docs

> **Fecha:** 2026-08-15
> **Estado:** ✅ ALL CLEAR — Documentación verificada y corregida

---

## Resumen Ejecutivo

Se auditaron los **20 archivos** del directorio `.docs/` que sirven como source of truth
para un monorepo Nx con Onion Architecture (NestJS + React/React Native/Electron).

La auditoría inicial detectó **4 errores críticos** y **4 errores menores**.
Todos fueron corregidos y la verificación final confirmó que **cada corrección
está correctamente aplicada**.

---

## 1. Hallazgos Críticos (Todos Resueltos ✅)

### 1.1 UI Bypass — Hooks instanciaban Aggregates con `User.create()`

| Propiedad | Detalle |
| :--- | :--- |
| **Severidad** | 🔴 Crítica |
| **Archivo** | `libs_bounded_context_ui.md` |
| **Problema** | `useRegisterUser` llamaba `User.create()` → acumulaba domain events que nunca se despachaban |
| **Corrección** | Reemplazado por `new User(id, name, email)` (DTO pass-through) |
| **Verificado** | ✅ Línea 94: `const user = new User(id, name, email);` |

### 1.2 Testing imports en código de producción — `CriteriaMother`

| Propiedad | Detalle |
| :--- | :--- |
| **Severidad** | 🔴 Crítica |
| **Archivos** | `apps_web.md`, `apps_mobile.md`, `libs_bounded_context_ui.md` |
| **Problema** | Importaban `CriteriaMother` desde `@monorepo/shared/testing` en producción |
| **Corrección** | Reemplazado por `Criteria.empty()` desde `@monorepo/shared/domain` |
| **Verificado** | ✅ Zero ocurrencias de `CriteriaMother` o `@monorepo/shared/testing` en los 3 archivos |

### 1.3 Transactional Outbox sin formalizar

| Propiedad | Detalle |
| :--- | :--- |
| **Severidad** | 🔴 Crítica |
| **Archivo** | `architectural_documentation.md` |
| **Problema** | Mencionaba "save + publish" sin definir atomicidad, UnitOfWork ni outbox table |
| **Corrección** | Sección completa con Persistencia Atómica, tabla `outbox`, `UnitOfWork`, `FailoverPublisher`, at-least-once delivery |
| **Verificado** | ✅ Líneas 107–119 documentan el patrón completo |

### 1.4 `Uuid.random()` con fallback inseguro `Math.random()`

| Propiedad | Detalle |
| :--- | :--- |
| **Severidad** | 🔴 Crítica |
| **Archivo** | `libs_shared_domain.md` |
| **Problema** | Generaba UUIDs con `Math.random()` si `crypto.randomUUID` no existía |
| **Corrección** | Lanza `Error` si `crypto.randomUUID` no está disponible |
| **Verificado** | ✅ Líneas 209–214: throw sin fallback |

---

## 2. Hallazgos Menores (Todos Resueltos ✅)

### 2.1 `DomainExceptionFilter` con string matching

| Propiedad | Detalle |
| :--- | :--- |
| **Archivo** | `apps_api.md` |
| **Problema** | Usaba `.includes('NotFound')` para mapear HTTP status codes |
| **Corrección** | Refactorizado a `instanceof DomainNotFoundError`, `DomainConflictError`, `InvalidArgumentError` |
| **Verificado** | ✅ Líneas 73–109: cadena de `instanceof` checks |

### 2.2 `ValueObject` asignaba antes de validar

| Propiedad | Detalle |
| :--- | :--- |
| **Archivo** | `libs_shared_domain.md` |
| **Problema** | `this.value = Object.freeze(value)` antes de `ensureValueIsDefined()` |
| **Corrección** | Validación primero, asignación después |
| **Verificado** | ✅ Líneas 162–163: `ensureValueIsDefined` → `Object.freeze` |

### 2.3 Bus interfaces con `any`

| Propiedad | Detalle |
| :--- | :--- |
| **Archivo** | `libs_shared_domain.md` |
| **Problema** | `CommandBus.dispatch(command: any)` y `QueryBus.ask(query: any)` |
| **Corrección** | Generics con `Command`/`Query` marker interfaces |
| **Verificado** | ✅ Líneas 448–463: `dispatch<C extends Command>` / `ask<R, Q extends Query>` |

### 2.4 Tag incorrecto `type:shared-app`

| Propiedad | Detalle |
| :--- | :--- |
| **Archivos** | `governance_nx_boundaries.md`, `architectural_documentation.md` |
| **Problema** | Tag Nx `type:shared-app` no existía — debía ser `type:shared-application` |
| **Corrección** | Reemplazado en ambos archivos |
| **Verificado** | ✅ Zero ocurrencias de `type:shared-app` |

---

## 3. Hallazgos de la Verificación Final

### 3.1 Numeración duplicada de secciones (Resuelto ✅)

Detectado y corregido durante la verificación final:

| Archivo | Problema | Corrección |
| :--- | :--- | :--- |
| `libs_shared_domain.md` | Dos secciones `### 3.3` (exceptions + event) | Renumerado: 3.3 exceptions → 3.4 event → 3.5 criteria → 3.6 bus → 3.7 result |
| `libs_bounded_context_domain.md` | Dos secciones `### 3.4` (ports + services) | Renumerado: services → `### 3.5` |

---

## 4. Correcciones Complementarias Aplicadas

| Corrección | Archivos |
| :--- | :--- |
| LaTeX `$\rightarrow$` → Unicode `→` | `README.md`, `architectural_documentation.md` |
| Jerarquía tipada `DomainException` + subclases | `libs_shared_domain.md`, `libs_bounded_context_domain.md` |
| `Criteria.empty()` + `Filters.empty()` factories | `libs_shared_domain.md` |
| Exports barrel actualizados (exceptions) | `libs_shared_domain.md`, `libs_bounded_context_domain.md` |
| Error handling `err: unknown` + `instanceof` | `libs_bounded_context_ui.md` |
| UI rule explícita (no instanciar aggregates) | `architectural_documentation.md` |

---

## 5. Archivos Modificados

| Archivo | Cambios |
| :--- | :--- |
| `.docs/README.md` | LaTeX → Unicode |
| `.docs/apps_api.md` | `DomainExceptionFilter` refactorizado a `instanceof` |
| `.docs/apps_web.md` | `CriteriaMother` → `Criteria.empty()` |
| `.docs/apps_mobile.md` | `CriteriaMother` → `Criteria.empty()` |
| `.docs/architectural_documentation.md` | Outbox, UI rules, boundaries table, LaTeX |
| `.docs/governance_nx_boundaries.md` | `type:shared-app` → `type:shared-application` |
| `.docs/libs_shared_domain.md` | VO order, Uuid security, bus typing, exceptions, factories, section numbering |
| `.docs/libs_bounded_context_domain.md` | Typed exceptions, exports, section numbering |
| `.docs/libs_bounded_context_ui.md` | Hook refactor, error handling, `Criteria.empty()` |

---

## 6. Archivos Sin Hallazgos

Los siguientes archivos fueron auditados y no presentaron problemas:

- `.docs/apps_desktop.md`
- `.docs/libs_bounded_context_application.md`
- `.docs/libs_bounded_context_infrastructure_client.md`
- `.docs/libs_bounded_context_infrastructure_server.md`
- `.docs/libs_shared_application.md`
- `.docs/libs_shared_infrastructure_client.md`
- `.docs/libs_shared_infrastructure_server.md`
- `.docs/libs_shared_testing.md`
- `.docs/libs_shared_ui.md`
- `.docs/testing_strategy.md`
- `.docs/tools_generators.md`

---

## Veredicto Final

> **La documentación está lista para usarse como source of truth de implementación.**
>
> No quedan errores críticos, menores ni inconsistencias pendientes.
> La suite de 20 documentos es coherente, las reglas de arquitectura son claras,
> y los ejemplos de código reflejan fielmente los principios definidos.
