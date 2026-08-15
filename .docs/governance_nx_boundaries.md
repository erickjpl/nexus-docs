# Especificación: Gobernanza de Fronteras en Nx (`governance_nx_boundaries.md`)

Este documento define las reglas estáticas y la configuración de **ESLint (`@nx/enforce-module-boundaries`)** que
blindan la arquitectura del monorepo. Su objetivo es impedir automáticamente en tiempo de desarrollo e integración
continua (CI) que se rompa la regla de dependencia de la Onion Architecture, que se contamine el frontend con
librerías de servidor o que dos Bounded Contexts se acoplen directamente.

---

## 1. Sistema Dimensional de Etiquetas (Tags)

Cada proyecto (`project.json`) en el monorepo debe declarar obligatoriamente dos dimensiones de etiquetas en su
array `tags`:

1. **Dimensión de Capa / Tipo (`type:*`):** Define qué rol juega la librería dentro de la arquitectura cebolla.
2. **Dimensión de Dominio / Alcance (`scope:*`):** Define a qué Bounded Context o aplicación pertenece.

```json
// Ejemplo: libs/users/domain/project.json
{
  "name": "users-domain",
  "tags": ["type:domain", "scope:users"]
}
```

---

## 2. Matriz Definitiva de Dependencias Permitidas

| Tag Origen (Source) | Tags Permitidos a Importar (Only Depend On) | Prohibiciones Críticas (Falla el Linter) |
| :--- | :--- | :--- |
| `type:shared-domain` | *(Ninguno — 0 dependencias)* | Todo framework, ORM o librería externa. |
| `type:shared-application` | `type:shared-domain` | Todo código de infraestructura o UI. |
| `type:shared-infra-server`| `type:shared-domain`, `type:shared-application` | `type:ui`, `type:shared-infra-client` |
| `type:shared-infra-client`| `type:shared-domain`, `type:shared-application` | `type:shared-infra-server` |
| `type:shared-testing` | `type:shared-domain`, `type:shared-application` | Código de producción. |
| `type:domain` | `type:shared-domain` | `type:application`, `type:infra-*`, `type:ui`, otros `scope:*`. |
| `type:application` | `type:domain`, `type:shared-domain`, `type:shared-application` | `type:infra-*`, `type:ui`, `scope:*` |
| `type:infra-server` | `type:domain`, `type:application`, `type:shared-domain`, `type:shared-application`, `type:shared-infra-server` | `type:ui`, `type:infra-client`, apps |
| `type:infra-client` | `type:domain`, `type:application`, `type:shared-domain`, `type:shared-infra-client` | `type:infra-server` |
| `type:ui` | `type:domain`, `type:application`, `type:infra-client`, `type:shared-domain`, `type:shared-infra-client` | `type:infra-server` |
| `type:testing` | `type:domain`, `type:application`, `type:shared-domain`, `type:shared-application`, `type:shared-testing` | Código de producción. |
| `type:app-backend` | `type:infra-server`, `type:shared-infra-server`, `type:shared-domain` | `type:ui`, `type:infra-client`, frontend |
| `type:app-frontend` | `type:ui`, `type:infra-client`, `type:shared-infra-client`, `type:domain` | `type:infra-server`, `type:app-backend` |
| `type:app-mobile` | `type:ui`, `type:infra-client`, `type:shared-infra-client`, `type:domain` | `type:infra-server`, paquetes web |
| `type:app-desktop` | `type:ui`, `type:infra-client`, `type:shared-infra-client`, `type:domain` | `type:infra-server` |

---

## 3. Configuración de ESLint (`.eslintrc.json` / `eslint.config.js`)

Esta configuración se ubica en la raíz del monorepo y valida el grafo de dependencias en cada comando de guardado
o `nx lint`:

```json
{
  "root": true,
  "plugins": ["@nx"],
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "enforceBuildableLibCheck": true,
        "allow": [],
        "depConstraints": [
          {
            "sourceTag": "type:shared-domain",
            "onlyDependOnLibsWithTags": []
          },
          {
            "sourceTag": "type:shared-application",
            "onlyDependOnLibsWithTags": ["type:shared-domain"]
          },
          {
            "sourceTag": "type:shared-infra-server",
            "onlyDependOnLibsWithTags": ["type:shared-domain", "type:shared-application"]
          },
          {
            "sourceTag": "type:shared-infra-client",
            "onlyDependOnLibsWithTags": ["type:shared-domain", "type:shared-application"]
          },
          {
            "sourceTag": "type:shared-testing",
            "onlyDependOnLibsWithTags": ["type:shared-domain", "type:shared-application"]
          },
          {
            "sourceTag": "type:domain",
            "onlyDependOnLibsWithTags": ["type:shared-domain"]
          },
          {
            "sourceTag": "type:application",
            "onlyDependOnLibsWithTags": ["type:domain", "type:shared-domain", "type:shared-application"]
          },
          {
            "sourceTag": "type:infra-server",
            "onlyDependOnLibsWithTags": [
              "type:domain",
              "type:application",
              "type:shared-domain",
              "type:shared-application",
              "type:shared-infra-server"
            ]
          },
          {
            "sourceTag": "type:infra-client",
            "onlyDependOnLibsWithTags": [
              "type:domain",
              "type:application",
              "type:shared-domain",
              "type:shared-infra-client"
            ]
          },
          {
            "sourceTag": "type:ui",
            "onlyDependOnLibsWithTags": [
              "type:domain",
              "type:application",
              "type:infra-client",
              "type:shared-domain",
              "type:shared-infra-client"
            ]
          },
          {
            "sourceTag": "type:testing",
            "onlyDependOnLibsWithTags": [
              "type:domain",
              "type:application",
              "type:shared-domain",
              "type:shared-application",
              "type:shared-testing"
            ]
          },
          {
            "sourceTag": "type:app-backend",
            "onlyDependOnLibsWithTags": [
              "type:infra-server",
              "type:shared-infra-server",
              "type:shared-domain"
            ]
          },
          {
            "sourceTag": "type:app-frontend",
            "onlyDependOnLibsWithTags": [
              "type:ui",
              "type:infra-client",
              "type:shared-infra-client",
              "type:domain",
              "type:shared-domain"
            ]
          },
          {
            "sourceTag": "type:app-mobile",
            "onlyDependOnLibsWithTags": [
              "type:ui",
              "type:infra-client",
              "type:shared-infra-client",
              "type:domain",
              "type:shared-domain"
            ]
          },
          {
            "sourceTag": "type:app-desktop",
            "onlyDependOnLibsWithTags": [
              "type:ui",
              "type:infra-client",
              "type:shared-infra-client",
              "type:domain",
              "type:shared-domain"
            ]
          }
        ]
      }
    ]
  }
}
```

También puedes utilizar el nuevo formato de Flat Config de ESLint (`eslint.config.js`):

```typescript
// eslint.config.js
import { FlatCompat } from '@eslint/eslintrc';
import nxPlugin from '@nx/eslint-plugin';

const compat = new FlatCompat();

export default [
  ...compat.config({
    extends: ['plugin:@nx/enforce-module-boundaries'],
    rules: {
      '@nx/enforce-module-boundaries': ['error', {
        enforceBuildableLibDependency: true,
        depConstraints: [
          // ... mismas reglas que en el formato JSON
        ],
      }],
    },
  }),
];
```

---

## 4. Reglas de Aislamiento Inter-Contextos (`scope:*`)

Para garantizar que un Bounded Context no importe directamente código de otro Bounded Context:

```json
{
  "sourceTag": "scope:users",
  "onlyDependOnLibsWithTags": ["scope:users", "scope:shared"]
},
{
  "sourceTag": "scope:billing",
  "onlyDependOnLibsWithTags": ["scope:billing", "scope:shared"]
},
{
  "sourceTag": "scope:auth",
  "onlyDependOnLibsWithTags": ["scope:auth", "scope:shared"]
}
```

> **Nota:** Para evitar tener que registrar manualmente cada nuevo contexto, puedes utilizar variables en los tags de origen, definiendo una regla genérica basada en patrones como `*` (dependiendo de la versión de Nx) o asegurándote de que el generador añada automáticamente estas reglas al crear un nuevo contexto.

> **Regla de Oro:** Si `billing` necesita datos de `users`, debe escuchar el evento de dominio `user.registered` o
> consumir un endpoint público a través de un adaptador de infraestructura. **Jamás** puede hacer
> `import { User } from '@monorepo/users/domain'`.

---

## 5. Comandos de Validación en CI

Para verificar que el monorepo cumple al 100% las fronteras arquitectónicas en cada Pull Request:

```bash
# 1. Validar reglas de fronteras y linting en todo el monorepo
npx nx run-many -t lint --all

# 2. Visualizar el grafo de dependencias interactivamente
npx nx graph

# 3. Validar únicamente proyectos afectados por el cambio actual
npx nx affected -t lint test
```
