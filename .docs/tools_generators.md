# Especificación: Generadores Nx de Arquitectura (`tools_generators.md`)

Este documento especifica el diseño y construcción de los **Nx Custom Generators** ubicados en
`tools/plugins/architecture/`. Su objetivo es automatizar la creación de nuevos **Bounded Contexts** y
**Vertical Slices** con un solo comando, garantizando que el 100% de la estructura de directorios, plantillas de
código, Value Objects, tags de ESLint y aliases de `tsconfig.base.json` se configuren de forma automática.

---

## 1. Generadores Disponibles

1. **`bounded-context`:** Genera el árbol completo de 6 librerías para un nuevo Bounded Context (`domain`,
   `application`, `infrastructure/server`, `infrastructure/client`, `ui`, `testing`) con sus `project.json`, tags y
   contratos iniciales.
2. **`vertical-slice`:** Genera un nuevo caso de uso atómico (Command o Query) dentro de
   `libs/{context}/application/src/slices/`, creando el comando/query, handler, servicio orquestador, test unitario con
   Object Mother y DTOs.

---

## 2. Estructura de Directorios en `tools/`

```text
tools/
├── plugins/
│   └── architecture/
│       ├── src/
│       │   └── generators/
│       │       ├── bounded-context/
│       │       │   ├── schema.json               # [ESTRICTO] Esquema JSON de opciones CLI
│       │       │   ├── schema.d.ts               # [ESTRICTO] Tipado TypeScript de opciones
│       │       │   ├── generator.ts              # [ESTRICTO] Lógica de creación con @nx/devkit
│       │       │   └── files/                    # [ESTRICTO] Plantillas de código EJS
│       │       │
│       │       └── vertical-slice/
│       │           ├── schema.json               # [ESTRICTO] Esquema JSON para Slices
│       │           ├── schema.d.ts               # [ESTRICTO] Tipado de opciones de Slice
│       │           ├── generator.ts              # [ESTRICTO] Lógica de inyección del Slice
│       │           └── files/                    # [ESTRICTO] Plantillas EJS de Command/Query
│       │
│       ├── generators.json                       # [ESTRICTO] Manifiesto de registro del plugin Nx
│       ├── package.json
│       ├── project.json
│       └── tsconfig.json
```

### Configuración en `nx.json`
Para registrar el plugin y hacerlo disponible en todo el workspace, es necesario añadirlo a los plugins de `nx.json`:

```json
{
  "plugins": [
    { "plugin": "./tools/plugins/architecture" }
  ]
}
```

> **Variables EJS Disponibles:** Todos los generadores utilizan las funciones del SDK de Nx para transformar los nombres. Al usar `<%= variable %>` en plantillas, tienes acceso a variantes como `__fileName__` (kebab-case), `className` (PascalCase), `propertyName` (camelCase) y `constantName` (UPPER_SNAKE_CASE).

---

## 3. Especificación del Generador: `bounded-context`

### 3.1 Esquema de Opciones (`schema.json`)

```json
{
  "$schema": "http://json-schema.org/schema",
  "id": "bounded-context",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Nombre del Bounded Context en kebab-case (ej. users, billing, orders)",
      "$default": {
        "$source": "argv",
        "index": 0
      },
      "x-prompt": "¿Cuál es el nombre del Bounded Context?"
    }
  },
  "required": ["name"]
}
```

### 3.2 Tipado del Esquema (`schema.d.ts`)

```typescript
// tools/plugins/architecture/src/generators/bounded-context/schema.d.ts
export interface BoundedContextGeneratorSchema {
  name: string;
  directory?: string;
  tags?: string;
}
```

---

### 3.2 Lógica del Generador (`generator.ts`)

```typescript
// tools/plugins/architecture/src/generators/bounded-context/generator.ts
import {
  Tree,
  formatFiles,
  generateFiles,
  joinPathFragments,
  names,
  updateJson
} from '@nx/devkit';
import { BoundedContextGeneratorSchema } from './schema';

export async function boundedContextGenerator(
  tree: Tree,
  options: BoundedContextGeneratorSchema
) {
  const contextNames = names(options.name);
  const contextRoot = joinPathFragments('libs', contextNames.fileName);

  const layers = [
    { name: 'domain', tag: 'type:domain' },
    { name: 'application', tag: 'type:application' },
    { name: 'infrastructure/server', tag: 'type:infra-server' },
    { name: 'infrastructure/client', tag: 'type:infra-client' },
    { name: 'ui', tag: 'type:ui' },
    { name: 'testing', tag: 'type:testing' }
  ];

  for (const layer of layers) {
    const layerPath = joinPathFragments(contextRoot, layer.name);
    
    generateFiles(
      tree,
      joinPathFragments(__dirname, 'files', layer.name),
      layerPath,
      {
        ...contextNames,
        tmpl: '',
        layerTag: layer.tag,
        scopeTag: `scope:${contextNames.fileName}`
      }
    );

    tree.write(
      joinPathFragments(layerPath, 'project.json'),
      JSON.stringify(
        {
          name: `${contextNames.fileName}-${layer.name.replace('/', '-')}`,
          sourceRoot: `${layerPath}/src`,
          projectType: 'library',
          tags: [layer.tag, `scope:${contextNames.fileName}`],
          targets: {
            lint: { executor: '@nx/eslint:lint' },
            test: { executor: '@nx/jest:jest' }
          }
        },
        null,
        2
      )
    );
  }

  updateJson(tree, 'tsconfig.base.json', (json) => {
    json.compilerOptions = json.compilerOptions || {};
    json.compilerOptions.paths = json.compilerOptions.paths || {};

    const baseAlias = `@monorepo/${contextNames.fileName}`;
    json.compilerOptions.paths[`${baseAlias}/domain`] = [`libs/${contextNames.fileName}/domain/src/index.ts`];
    json.compilerOptions.paths[`${baseAlias}/application`] = [`libs/${contextNames.fileName}/application/src/index.ts`];
    const infraServerPath = `libs/${contextNames.fileName}/infrastructure/server/src/index.ts`;
    json.compilerOptions.paths[`${baseAlias}/infrastructure/server`] = [infraServerPath];
    const infraClientPath = `libs/${contextNames.fileName}/infrastructure/client/src/index.ts`;
    json.compilerOptions.paths[`${baseAlias}/infrastructure/client`] = [infraClientPath];
    json.compilerOptions.paths[`${baseAlias}/ui`] = [`libs/${contextNames.fileName}/ui/src/index.ts`];
    json.compilerOptions.paths[`${baseAlias}/testing`] = [`libs/${contextNames.fileName}/testing/src/index.ts`];

    return json;
  });

  await formatFiles(tree);
}

export default boundedContextGenerator;
```

> **Nota sobre ESLint:** El generador `bounded-context` también debe encargarse de actualizar el archivo `eslint.config.js` (o `.eslintrc.json`) para añadir automáticamente las nuevas restricciones de dependencia (`depConstraints`) con el nuevo tag `scope:{context}`, de forma que se aísle del resto de contextos existentes.

---

## 4. Especificación del Generador: `vertical-slice`

### 4.1 Esquema de Opciones (`schema.json`)

```json
{
  "$schema": "http://json-schema.org/schema",
  "id": "vertical-slice",
  "type": "object",
  "properties": {
    "context": {
      "type": "string",
      "description": "Nombre del Bounded Context existente",
      "x-prompt": "¿En qué Bounded Context se creará el Slice?"
    },
    "name": {
      "type": "string",
      "description": "Nombre del caso de uso en kebab-case (ej. register-user, update-profile)",
      "x-prompt": "¿Cuál es el nombre del caso de uso?"
    },
    "type": {
      "type": "string",
      "enum": ["command", "query"],
      "description": "Tipo de operación CQRS",
      "x-prompt": "¿Es un Command (mutación) o una Query (consulta)?"
    }
  },
  "required": ["context", "name", "type"]
}
```

### 4.2 Plantilla EJS de Comando (`__fileName__.command.ts.template`)

```ejs
// tools/plugins/architecture/src/generators/vertical-slice/files/command-slice/__fileName__.command.ts.template
import { Command } from '@monorepo/shared/application';

export class <%= className %>Command extends Command {
  constructor(
    readonly id: string,
    // TODO: Add command properties
  ) {
    super();
  }
}
```

---

### 4.2 Lógica del Generador (`generator.ts`)

```typescript
// tools/plugins/architecture/src/generators/vertical-slice/generator.ts
import {
  Tree,
  formatFiles,
  generateFiles,
  joinPathFragments,
  names
} from '@nx/devkit';
import { VerticalSliceGeneratorSchema } from './schema';

export async function verticalSliceGenerator(
  tree: Tree,
  options: VerticalSliceGeneratorSchema
) {
  const contextNames = names(options.context);
  const sliceNames = names(options.name);

  const sliceRoot = joinPathFragments(
    'libs',
    contextNames.fileName,
    'application',
    'src',
    'slices',
    sliceNames.fileName
  );

  const templateFolder = options.type === 'command' ? 'command-slice' : 'query-slice';

  generateFiles(
    tree,
    joinPathFragments(__dirname, 'files', templateFolder),
    sliceRoot,
    {
      ...sliceNames,
      contextName: contextNames.fileName,
      contextClassName: contextNames.className,
      tmpl: ''
    }
  );

  await formatFiles(tree);
}

export default verticalSliceGenerator;
```

---

## 5. Manifiesto del Plugin (`generators.json`)

```json
{
  "$schema": "http://json-schema.org/schema",
  "name": "architecture",
  "version": "1.0.0",
  "generators": {
    "bounded-context": {
      "factory": "./src/generators/bounded-context/generator",
      "schema": "./src/generators/bounded-context/schema.json",
      "description": "Crea un Bounded Context completo con las 6 capas DDD y configuración de Nx"
    },
    "vertical-slice": {
      "factory": "./src/generators/vertical-slice/generator",
      "schema": "./src/generators/vertical-slice/schema.json",
      "description": "Crea un Vertical Slice atómico (Command/Query, Handler, Service y Spec)"
    }
  }
}
```

---

## 6. Comandos de Uso en la Terminal

```bash
# 1. Crear un nuevo Bounded Context completo llamado 'users'
npx nx g @monorepo/architecture:bounded-context users

# 2. Crear un caso de uso de tipo Command en el contexto 'users'
npx nx g @monorepo/architecture:vertical-slice --context=users --name=register-user --type=command

# 3. Crear un caso de uso de tipo Query en el contexto 'users'
npx nx g @monorepo/architecture:vertical-slice --context=users --name=get-user-by-id --type=query
```
