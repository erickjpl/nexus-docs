# Especificación: Metodología GitFlow y Gobernanza en GitHub (`governance_gitflow.md`)

Este documento establece la metodología oficial de trabajo con **GitFlow**, el sistema de **Labels de GitHub**,
las convenciones de commits semánticos, el ciclo de vida de **Release Candidates (RC)**, versionamiento **SemVer** y
la generación de **CHANGELOG**. Su cumplimiento es estricto y obligatorio para todos los desarrolladores (desde
Juniors hasta Seniors) y **Agentes de Inteligencia Artificial**.

---

## 1. La Regla de Oro: Filosofía "Issue-First"

En este monorepo **está estrictamente prohibido escribir código o crear ramas sin un Issue previo**.

```text
[1. Crear Issue en GitHub] ──> [2. Asignar Labels] ──> [3. Crear Rama Asociada]
                                                                        │
[6. Auto-Eliminar Rama] <── [5. Aprobar PR + Squash] <── [4. Desarrollar + Commits Atómicos]
```

### ¿Por qué existe esta regla?

1. **Trazabilidad Total:** Cada línea de código responde a una necesidad de negocio o corrección documentada.
2. **Visibilidad en el Tablero (GitHub Projects):** Permite a todo el equipo conocer en tiempo real el estado de una tarea
   simplemente observando su columna de estado.
3. **Automatización de Cierre:** Todo Pull Request (PR) debe incluir la directiva `Closes #<id_issue>`, cerrando
   automáticamente la tarjeta y moviéndola a completada al momento de fusionar.

---

### 1.1 Ciclo de Vida de las Tarjetas en GitHub Projects (Las 6 Columnas)

El tablero de GitHub Projects gobierna el flujo de trabajo a través de 6 columnas formales y secuenciales:

```text
[📋 Listado de tareas] ──> [🔨 En desarrollo] ──> [🧪 Probando (testing)] ──> [🚀 Desarrollado] ──> [📦 Probado] ──> [✅ Hecho]
```

| # | Columna | Momento de la Transición | Acciones Obligatorias del Desarrollador / Agente | Comando CLI |
| :- | :--- | :--- | :--- | :--- |
| **1** | **`📋 Listado de tareas`** | **Backlog**: Estado inicial al crear el Issue. | Tarea documentada esperando ser priorizada. | `gh issue create ...` |
| **2** | **`🔨 En desarrollo`** | **Inicio de implementación**: Al comenzar a escribir código. | 1. Mover tarjeta a `🔨 En desarrollo`.<br>2. Crear rama `feature/<id>-<slug>` o `bugfix/<id>-<slug>` desde `develop`.<br>3. Desarrollar con TDD y commits atómicos. | `gh project item-edit ...` |
| **3** | **`🧪 Probando (testing)`** | **Fase de Validación**: Al terminar el código y verificar calidad. | 1. Mover tarjeta a `🧪 Probando (testing)`.<br>2. Ejecutar pre-push gates: `npx nx test <app>`, `npx nx run-many -t test,typecheck,lint` y pruebas de endpoints con Bruno (`rest-client/`). | `gh project item-edit ...` |
| **4** | **`🚀 Desarrollado`** | **Merge a develop**: PR aprobado y fusionado. | 1. Abrir PR hacia `develop` con `Closes #<id>`.<br>2. Squash & Merge en GitHub.<br>3. Mover tarjeta a `🚀 Desarrollado` y auto-eliminar rama de feature. | `gh pr create` + `gh pr merge` |
| **5** | **`📦 Probado`** | **Acumulado en develop**: Listo para empaquetado. | Tarea integrada en `develop`, verificada y en espera del corte de release. | `gh project item-edit ...` |
| **6** | **`✅ Hecho`** | **En Producción**: Release publicado con Tag. | 1. Crear rama `release/vX.Y.Z` desde `develop`.<br>2. Mergear a `main` (con Tag inmutable) y retro-merge a `develop`.<br>3. Mover tarjeta a `✅ Hecho`. | `gh project item-edit ...` |

---

## 2. Ramas del Repositorio y su Ciclo de Vida

GitFlow organiza el trabajo dividiendo las ramas en **Eternas (Protegidas)** y **Temporales (de Soporte)**:

```text
  main (v1.0.0) ───────────────────────────────────────────────> (v1.1.0 Tag)
      │                                                               ▲
      │                                       ┌── release/v1.1.0 ─────┘
      ▼                                       │ (Fijación de RC)      │
  develop ──────┬─────────────────┬───────────┴───────────────────────┴──>
                │                 ▲
                └── feature/12-register-user ──┘ (Se auto-elimina al mergear)
```

### 2.1 Ramas Principales (Eternas y Protegidas)

* **`main` (Producción):**
  * **Propósito:** Contiene exclusivamente código desplegado en producción. Es 100% estable y auditado.
  * **Regla estricta:** Prohibido el `push` directo. Solo recibe merges mediante Pull Request desde ramas `release/*`
    o `hotfix/*`.
  * **Tags:** Cada merge a `main` genera obligatoriamente un Tag inmutable de versión (`vX.Y.Z`).

* **`develop` (Integración Continua / Desarrollo):**
  * **Propósito:** Es la rama central de integración donde convergen las nuevas funcionalidades y correcciones.
  * **Regla estricta:** Prohibido el `push` directo. Todo cambio ingresa mediante Pull Request desde ramas `feature/*`
    o `bugfix/*`.

---

### 2.2 Ramas de Soporte (Temporales)

Toda rama temporal **debe auto-eliminarse** en GitHub al ser aprobada y mergeada (`Delete branch` activado).

| Prefijo | Nomenclatura | Origen | Destino | Propósito |
| :--- | :--- | :--- | :--- | :--- |
| `feature/` | `feature/<id>-<kebab>` | `develop` | `develop` | Nueva funcionalidad o caso de uso. |
| `bugfix/` | `bugfix/<id>-<kebab>` | `develop` | `develop` | Corrección de errores en desarrollo o QA. |
| `refactor/`| `refactor/<id>-<kebab>` | `develop` | `develop` | Reestructuración de código sin alterar lógica. |
| `release/` | `release/v<semver>` | `develop` | `main` y `develop` | Preparación de versión y estabilización de RC. |
| `hotfix/` | `hotfix/<id>-<kebab>` | `main` | `main` y `develop` | Corrección crítica urgente de producción. Proceso de 2 pasos: 1. PR a `main`, aprobar y mergear. 2. PR a `develop` (o retro-merge de `main` a `develop`). Eliminar la rama de hotfix solo después de ambos merges. |

---

## 3. Catálogo de Prefijos y Conventional Commits

Los mensajes de commit deben ser **atómicos** (un solo cambio lógico) y seguir la especificación
**Conventional Commits**.

### 3.1 Estructura Obligatoria del Commit

```text
<tipo>(<alcance>): <descripción imperativa en presente>

[Cuerpo opcional: explica el QUÉ y el PORQUÉ, no el CÓMO]

Closes #<id_issue>
```

---

### 3.2 Tipos de Commit Estándar

| Tipo | Significado y Cuándo Usarlo | Ejemplo Real |
| :--- | :--- | :--- |
| `feat` | Nueva funcionalidad o nuevo caso de uso. | `feat(users): implement user registration slice` |
| `fix` | Corrección de un error o bug en el código. | `fix(users): handle duplicated email error` |
| `refactor` | Cambio interno sin alterar comportamiento. | `refactor(shared): extract criteria converter` |
| `test` | Agregar o corregir tests, mocks o mothers. | `test(users): add user mother and repo specs` |
| `docs` | Cambios exclusivos en documentación. | `docs(gitflow): document github labels` |
| `perf` | Mejora de rendimiento o consumo de recursos. | `perf(api): optimize criteria search query` |
| `chore` | Mantenimiento de herramientas o librerías. | `chore(deps): update nestjs packages` |
| `ci` | Modificaciones en pipelines de CI/CD. | `ci(github): configure nx run-many workflow` |
| `style` | Formateo de código o espacios (sin lógica). | `style(users): fix indentation and wrapping` |

---

### 3.3 Ejemplos de Commits Profesionales Atómicos

> **Nota:** La directiva `Closes #<id>` va en la descripción del Pull Request y en el squash commit al mergear, no en cada commit atómico.

```text
feat(users): implement register user command and handler

- Add RegisterUserCommand DTO with primitive validations
- Create UserRegistrar application service orchestrating domain
- Connect CommandBus with RegisterUserCommandHandler
```

```text
fix(users): validate email format before aggregate creation

- Add regex invariant inside UserEmail value object
- Throw InvalidArgumentError on malformed emails
```

---

### 3.4 Controles Obligatorios Pre-Push (Pre-Push Verification Gates)

Ningún desarrollador ni agente de IA puede realizar `git push` sin que pasen de forma exitosa los siguientes tres controles locales (configurados vía Husky en `.husky/pre-push`):

1. **Linter Estricto:** `npx nx run-many -t lint` debe pasar sin errores, **sin variables no utilizadas** y **sin importaciones huérfanas**.
2. **Chequeo de Tipos Estricto:** `npx nx run-many -t typecheck` debe pasar con cero errores de tipado.
3. **Suite de Pruebas al 100%:** `npx nx run-many -t test` debe ejecutar y aprobar la totalidad de pruebas del módulo o proyecto.

---

### 3.5 Documentación Viva de APIs con Colección Bruno (`rest-client/api/`)

Toda creación o modificación de un endpoint HTTP exige obligatoriamente:
1. **Archivo `.bru` Dedicado:** Crear o actualizar el archivo correspondiente en el directorio de la colección (`rest-client/api/...`).
2. **Sincronización con DTO (class-validator):** Si cambian los headers, query params o el body, el `.bru` debe actualizarse inmediatamente.
3. **Bloque `docs` en Markdown Obligatorio:** Todo `.bru` debe incluir documentación Markdown que detalle:
   - Propósito del endpoint.
   - Permiso atómico requerido (`PermissionEnum.RESOURCE_ACTION`).
   - Descripción de parámetros de entrada y respuestas esperadas.

---

## 4. Taxonomía Exhaustiva de Labels de GitHub

GitHub categoriza los Issues a través de etiquetas estructuradas con formato `categoria: valor`:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CATEGORÍAS DE LABELS                               │
├───────────────┬───────────────┬───────────────┬──────────────┬──────────────┤
│    type: *    │    layer: *   │    status: *  │  priority: * │  severity: * │
└───────────────┴───────────────┴───────────────┴──────────────┴──────────────┘
```

---

### 4.1 Categoría: `type: *` (Naturaleza del Trabajo)

Define la intención técnica o funcional de la tarea:

> **Nota:** Existe un mapeo entre etiquetas de GitHub y Conventional Commits: la etiqueta `type: hotfix` genera commits `fix(...)`, mientras que los commits `ci(...)` o `style(...)` se agrupan en issues `type: chore`.

- `type: feature` (Color Verde): Nueva funcionalidad, caso de uso o endpoint.
- `type: bug` (Color Rojo): Comportamiento anómalo o error que rompe funcionalidad existente.
- `type: hotfix` (Color Naranja Fuerte): Bug crítico en producción que requiere atención inmediata.
- `type: refactor` (Color Violeta): Limpieza arquitectónica o desacoplamiento sin alterar comportamiento.
- `type: test` (Color Amarillo): Aumento de cobertura, nuevos Object Mothers o tests de integración.
- `type: docs` (Color Azul Claro): Documentación técnica, diagramas o guías de arquitectura.
- `type: chore` (Color Gris): Tareas rutinarias de mantenimiento, actualización de dependencias o scripts.
- `type: perf` (Color Verde Azulado): Optimizaciones de rendimiento, queries lentas o memoria.

---

### 4.2 Categoría: `layer: *` (Capa Arquitectónica Afectada)

Identifica en qué capa de la Onion Architecture se concentra el cambio:

- `layer: domain`: Agregados, Value Objects, Domain Events, Domain Services o Puertos.
- `layer: application`: Vertical Slices, Commands, Queries, Handlers o DTOs.
- `layer: infra-server`: TypeORM EntitySchema, Mappers, Controladores NestJS o Consumers RabbitMQ.
- `layer: infra-client`: Repositorios HTTP para frontend, adaptadores Storage o APIs cliente.
- `layer: ui`: Componentes visuales React/React Native, Custom Hooks o Vistas.
- `layer: testing`: Object Mothers, Mocks semánticos o fixtures de prueba.
- `layer: cross-cutting`: Cambios que cruzan múltiples capas o módulos compartidos.

---

### 4.3 Categoría: `status: *` (Estado en el Tablero de GitHub Projects)

Gobierna el flujo Kanban y el ciclo de vida de la tarea:

- `status: backlog`: Tarea identificada y redactada, pendiente de priorización.
- `status: todo`: Tarea priorizada y lista para ser tomada por un desarrollador.
- `status: in-progress`: Rama creada y desarrollo activo en curso.
- `status: in-review`: Pull Request abierto, en proceso de revisión por pares y CI.
- `status: blocked`: Tarea detenida por dependencias externas o impedimentos técnicos.
- `status: ready-for-release`: Mergeada en `develop`, esperando empaquetado en release.

---

### 4.4 Categoría: `priority: *` (Urgencia de Planificación)

Define el orden en que el equipo debe atender las tareas:

- `priority: critical`: Bloqueante para el negocio o release inmediato.
- `priority: high`: Alta importancia, planificada para la iteración actual.
- `priority: medium`: Importancia estándar, se ejecuta en orden cronológico normal.
- `priority: low`: Deseable pero postergable si hay prioridades mayores.

---

### 4.5 Categoría: `severity: *` (Gravedad del Bug)

Aplica exclusivamente a tareas de tipo `type: bug` o `type: hotfix`:

- `severity: blocker`: Caída total del sistema o pérdida crítica de datos.
- `severity: major`: Funcionalidad principal rota sin alternativa de mitigación.
- `severity: minor`: Funcionalidad secundaria afectada o bug cosmético con workaround disponible.

---

### 4.6 Categoría: `scope: *` (Bounded Context o Aplicación)

Indica el módulo específico afectado:

- `scope: users`: Bounded Context de Usuarios.
- `scope: shared`: Shared Kernel (`libs/shared/*`).
- `scope: api`: Aplicación Backend NestJS (`apps/api`).
- `scope: web`: Aplicación Frontend React (`apps/web`).
- `scope: mobile`: Aplicación Mobile React Native/Expo (`apps/mobile`).
- `scope: desktop`: Aplicación Desktop Electron (`apps/desktop`).

---

## 5. Ciclo de Vida de Release Candidates (RC), SemVer y Changelog

```text
develop ──> [Crear release/v1.2.0] ──> Tag v1.2.0-rc.1 (QA) ──> Tag v1.2.0-rc.2
                                                                      │
main <───────────────────────── [Merge Final + Tag v1.2.0] <──────────┘
  │                                     │
  └──> Retro-merge hacia develop ───────┘
```

---

### 5.1 Versionamiento Semántico (SemVer 2.0.0)

El número de versión sigue el formato estricto `v<MAJOR>.<MINOR>.<PATCH>`:

1. **`MAJOR` (Incompatible):** Cambios que rompen compatibilidad (_Breaking Changes_), rediseño de contratos públicos.
2. **`MINOR` (Funcionalidad):** Nuevas features y casos de uso 100% retrocompatibles.
3. **`PATCH` (Corrección):** Corrección de bugs y parches de seguridad retrocompatibles.

---

### 5.2 El Flujo de Release Candidate (RC)

Cuando se planifica una entrega:

1. **Creación de la Rama de Release:** Se crea desde `develop`:
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b release/v1.2.0
   ```
2. **Emisión de Release Candidate:** Se publica el primer candidato para QA y pruebas de integración:
   ```bash
   git tag -a v1.2.0-rc.1 -m "Release Candidate 1 for v1.2.0"
   git push origin v1.2.0-rc.1
   ```
3. **Estabilización de la RC:**
   - Solo se permiten commits de tipo `fix`, `docs` o `style` en la rama `release/v1.2.0`.
   - Si se corrigen bugs, se emite `v1.2.0-rc.2`, `v1.2.0-rc.3`, etc.
4. **Criterios de Aprobación para Paso a Producción:**
   - [ ] Cero bugs abiertos de severidad `blocker` o `major`.
   - [ ] Suite completa de tests (`nx run-many -t test lint`) pasando al 100%.
   - [ ] La última RC ha permanecido al menos 24 horas estable en entorno de Staging.
   - [ ] `CHANGELOG.md` actualizado con todas las notas de la versión.
5. **Cierre de Release:**
   - Se mergea `release/v1.2.0` hacia `main` mediante PR.
   - Se crea el Tag definitivo en `main`: `git tag -a v1.2.0 -m "Release v1.2.0" && git push origin v1.2.0`.
   - Se retro-mergea `release/v1.2.0` hacia `develop` para que las correcciones queden integradas.
   - Se elimina la rama `release/v1.2.0`.

---

### 5.3 Estándar del Archivo `CHANGELOG.md`

El archivo `CHANGELOG.md` se ubica en la raíz del monorepo y sigue el formato _Keep a Changelog_:

```markdown
# Changelog

Todos los cambios notables en este proyecto serán documentados en este archivo.
El formato está basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/)
y este proyecto adhiere a [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [v1.2.0] - 2026-08-15

### ✨ Añadido (Added)

- `users`: Implementación del caso de uso de registro de usuario (`UserRegistrar`) y su comando (#42).
- `users`: Endpoint HTTP `PUT /api/users/:id` en NestJS con validación por DTO (#42).
- `shared`: Convertidor de criterios para TypeORM (`TypeOrmCriteriaConverter`) (#35).

### 🐛 Corregido (Fixed)

- `users`: Validación de formato de email en `UserEmail` arrojando `InvalidArgumentError` (#58).

### 🔧 Modificado (Changed)

- `shared`: Migración de repositorios base a TypeScript 5.x (#20).

### ⚠️ Breaking Changes

- `api`: Reestructuración del formato de respuesta de errores de validación al estándar ApiResponse envelope.
```

---

## 6. Guía Rápida de Comandos Git (Cheat Sheet)

### Caso 1: Iniciar una nueva Feature

```bash
# 1. Actualizar develop
git checkout develop
git pull origin develop

# 2. Crear rama asociada a la Issue #42
git checkout -b feature/42-register-user

# 3. Desarrollar y realizar commits atómicos
git add .
git commit -m "feat(users): implement user aggregate and invariants"

# 4. Publicar rama en GitHub
git push -u origin feature/42-register-user
```

---

### Caso 2: Corregir un Hotfix Urgente en Producción

```bash
# 1. Partir desde main
git checkout main
git pull origin main

# 2. Crear rama de hotfix asociada a la Issue #99
git checkout -b hotfix/99-fix-auth-token-crash

# 3. Corregir y commitear
git add .
git commit -m "fix(auth): handle expired token exception on guard"

# 4. Crear PR hacia main y hacia develop
git push -u origin hotfix/99-fix-auth-token-crash
```
