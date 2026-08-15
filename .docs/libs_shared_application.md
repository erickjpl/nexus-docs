# Especificación: `libs/shared/application`

Este módulo constituye la **Capa de Aplicación Compartida (Shared Application)** del monorepo. Proporciona las
abstracciones base, contratos y tipos genéricos necesarios para implementar casos de uso, orquestación, CQRS
(Command Query Responsibility Segregation) y manejo de respuestas en cualquier Bounded Context.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Aislamiento Total de Frameworks:** Prohibido importar `@nestjs/common`, `@nestjs/cqrs`, librerías HTTP,
  ORMs o paquetes de React. Todas las clases son TypeScript nativo puro.
* **[ESTRICTO] Dependencias Permitidas:** Solo depende de `@monorepo/shared/domain` (`type:shared-domain`).
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:shared-application", "scope:shared"]`.
* **[ESTRICTO] Responsabilidad Única:** Esta capa solo define abstracciones y contratos de orquestación. La ejecución
  en memoria, el registro dinámico o el binding con brokers de mensajería pertenecen a `infrastructure`.

---

## 2. Estructura de Directorios

```text
libs/shared/application/
├── src/
│   ├── command/
│   │   ├── command.ts                    # [ESTRICTO] Contrato base para Comandos (mutaciones de estado)
│   │   ├── command-handler.ts            # [ESTRICTO] Interfaz para manejadores de comandos
│   │   └── command-class.type.ts         # [ESTRICTO] Tipo para el constructor/firma del comando
│   │
│   ├── query/
│   │   ├── query.ts                      # [ESTRICTO] Contrato base para Consultas (lecturas sin efecto secundario)
│   │   ├── query-handler.ts              # [ESTRICTO] Interfaz para manejadores de consultas
│   │   ├── query-class.type.ts           # [ESTRICTO] Tipo para el constructor/firma de la consulta
│   │   └── response.ts                   # [ESTRICTO] Interfaz marcadora para respuestas de queries (DTOs)
│   │
│   ├── use-case/
│   │   └── use-case.interface.ts         # [OPCIONAL] Interfaz genérica para orquestadores directos (sin bus)
│   │
│   └── index.ts                          # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `command/`

#### `command.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Representa una intención inequívoca de modificar el estado del sistema (ej. `RegisterUserCommand`,
  `UpdateUserEmailCommand`). Es un DTO inmutable que transporta primitivos validados desde el exterior hacia el núcleo.
* **Comportamiento exigido:**
  * Debe ser una clase abstracta o interfaz inmutable.
  * No contiene lógica de negocio ni comportamiento; solo datos primitivos de entrada.

```typescript
export abstract class Command {
  // Marcador base para tipado de comandos en el CommandBus
}
```

---

#### `command-class.type.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Permite que el `CommandBus` mapee en tiempo de compilación y ejecución la clase exacta del
  comando con su respectivo handler sin acoplarse a strings mágicos.

```typescript
import { Command } from './command';

export type CommandClass<T extends Command> = {
  new (...args: any[]): T;
};
```

---

#### `command-handler.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es el contrato que debe cumplir cualquier clase encargada de procesar un comando específico.
* **Comportamiento exigido:**
  * `subscribedTo(): CommandClass<C>`: Retorna la clase del comando que este handler es capaz de procesar.
  * `handle(command: C): Promise<void>`: Ejecuta el caso de uso orquestando agregados, repositorios y eventos.
  * **No retorna datos de negocio**, únicamente resuelve o rechaza la promesa.

```typescript
import { Command } from './command';
import { CommandClass } from './command-class.type';

export interface CommandHandler<C extends Command> {
  subscribedTo(): CommandClass<C>;
  handle(command: C): Promise<void>;
}
```

---

### 3.2 Bloque: `query/`

#### `query.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Representa una solicitud de lectura de información. A diferencia de un comando, una query
  **jamás** produce efectos secundarios ni muta el estado del sistema.

```typescript
export abstract class Query {
  // Marcador base para tipado de consultas en el QueryBus
}
```

---

#### `query-class.type.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Tipado del constructor de la consulta para el registro en el `QueryBus`.

```typescript
import { Query } from './query';

export type QueryClass<Q extends Query> = {
  new (...args: any[]): Q;
};
```

---

#### `response.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Marca las estructuras de retorno de una consulta. Evita que los casos de uso retornen Entidades
  o Agregados de Dominio mutables directamente al exterior, forzando el retorno de DTOs o Read Models primitivos.

```typescript
export interface Response {
  // Interfaz marcadora para todos los DTOs de respuesta de consultas
}
```

---

#### `query-handler.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Contrato para los procesadores de consultas.
* **Comportamiento exigido:**
  * `subscribedTo(): QueryClass<Q>`: Especifica la Query asociada.
  * `handle(query: Q): Promise<R>`: Ejecuta la búsqueda y transforma los datos en una `Response`.

```typescript
import { Query } from './query';
import { QueryClass } from './query-class.type';
import { Response } from './response';

export interface QueryHandler<Q extends Query, R extends Response> {
  subscribedTo(): QueryClass<Q>;
  handle(query: Q): Promise<R>;
}
```

---

### 3.3 Bloque: `use-case/`

#### `use-case.interface.ts`
* **Nivel:** **[OPCIONAL]**
* **Por qué existe:** Para escenarios donde un Bounded Context o una aplicación de frontend decide invocar un servicio
  de aplicación directamente sin pasar por un CommandBus/QueryBus.

```typescript
export interface UseCase<Input, Output> {
  execute(request: Input): Promise<Output>;
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/application/src/index.ts
export * from './command/command';
export * from './command/command-handler';
export * from './command/command-class.type';

export * from './query/query';
export * from './query/query-handler';
export * from './query/query-class.type';
export * from './query/response';

export * from './use-case/use-case.interface';
```
