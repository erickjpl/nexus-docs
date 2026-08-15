# Especificación: `libs/shared/domain`

Este módulo constituye el **Shared Domain Kernel** del monorepo. Contiene las abstracciones fundamentales, contratos y
bloques de construcción universales del Dominio. Es la base sobre la cual se construyen todos los Bounded Contexts.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Dependencias Cero:** Este paquete **NO** puede tener dependencias en su `package.json` hacia ningún
  framework (NestJS, React, Express), ORM (TypeORM, Prisma, Mongoose) ni librerías de infraestructura (Axios, RabbitMQ).
  Solo se permite TypeScript puro y utilidades de bajo nivel universalmente aprobadas (ej. `crypto.randomUUID()`).
* **[ESTRICTO] Independencia de Plataforma:** Debe poder ejecutarse sin alteraciones en Node.js, V8 (Navegadores),
  JavaScriptCore/Hermes (React Native) y Electron.
* **[ESTRICTO] Tag de Nx:** Debe estar etiquetado en `project.json` como `tags: ["type:shared-domain", "scope:shared"]`.
* **[ESTRICTO] Inmutabilidad y Encapsulación:** Todos los Value Objects deben ser inmutables (`readonly`).
  No se permiten setters públicos.

---

## 2. Estructura de Directorios

```text
libs/shared/domain/
├── src/
│   ├── aggregate/
│   │   ├── aggregate-root.ts             # [ESTRICTO] Clase base para todas las raíces de agregado
│   │   └── entity.ts                     # [ESTRICTO] Clase base para entidades con identidad propia
│   │
│   ├── value-object/
│   │   ├── value-object.ts               # [ESTRICTO] Clase base abstracta universal para VOs
│   │   ├── string.vo.ts                  # [ESTRICTO] VO base para tipos string
│   │   ├── number.vo.ts                  # [ESTRICTO] VO base para tipos numéricos
│   │   ├── uuid.vo.ts                    # [ESTRICTO] VO especializado para identificadores únicos
│   │   ├── enum.vo.ts                    # [ESTRICTO] VO base para enumeraciones tipadas
│   │   └── invalid-argument.error.ts     # [ESTRICTO] Error arrojado ante violación de invariantes de VOs
│   │
│   ├── event/
│   │   ├── domain-event.ts               # [ESTRICTO] Clase base abstracta de eventos de dominio
│   │   ├── domain-event-subscriber.ts    # [ESTRICTO] Interfaz de contrato para suscriptores
│   │   └── domain-event-class.ts         # [ESTRICTO] Tipo para la referencia estática de eventos
│   │
│   ├── criteria/
│   │   ├── criteria.ts                   # [ESTRICTO] Raíz de la especificación de consulta
│   │   ├── filter.ts                     # [ESTRICTO] Contenedor de un filtro individual
│   │   ├── filters.ts                    # [ESTRICTO] Colección inmutable de filtros
│   │   ├── filter-field.ts               # [ESTRICTO] Campo a filtrar (Value Object)
│   │   ├── filter-operator.ts            # [ESTRICTO] Operador de comparación (Enum VO)
│   │   ├── filter-value.ts               # [ESTRICTO] Valor del filtro (Value Object)
│   │   ├── order.ts                      # [ESTRICTO] Criterio de ordenamiento
│   │   ├── order-by.ts                   # [ESTRICTO] Campo de orden (Value Object)
│   │   └── order-type.ts                 # [ESTRICTO] Dirección del orden: ASC, DESC, NONE (Enum VO)
│   │
│   ├── bus/
│   │   ├── command-bus.ts                # [ESTRICTO] Interfaz del bus de comandos
│   │   ├── query-bus.ts                  # [ESTRICTO] Interfaz del bus de consultas
│   │   └── event-bus.ts                  # [ESTRICTO] Interfaz del bus de eventos de dominio
│   │
│   ├── result/
│   │   ├── result.ts                     # [ESTRICTO] Implementación del patrón Result<T, E>
│   │   └── failure.ts                    # [ESTRICTO] Clase base para errores de negocio controlados
│   │
│   ├── types/
│   │   └── nullable.type.ts              # [OPCIONAL] Tipo auxiliar para Nullable<T> = T | null
│   │
│   └── index.ts                          # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `aggregate/`

#### `aggregate-root.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es la unidad atómica de consistencia transaccional del negocio. Encapsula entidades internas
  y gestiona la acumulación de eventos de dominio generados durante la ejecución de sus métodos de negocio.
* **Comportamiento exigido:**
  * Debe mantener una colección privada `domainEvents: Array<DomainEvent>`.
  * `record(event: DomainEvent): void`: Método protegido para registrar que algo ocurrió.
  * `pullDomainEvents(): Array<DomainEvent>`: Extrae todos los eventos acumulados y vacía la colección interna de
    forma atómica para evitar despachos duplicados.
  * `abstract toPrimitives(): Record<string, any>`: Obliga a cada agregado a saber cómo representarse en tipos
    primitivos puros.

```typescript
import { DomainEvent } from '../event/domain-event';

export abstract class AggregateRoot {
  private domainEvents: Array<DomainEvent> = [];

  pullDomainEvents(): Array<DomainEvent> {
    const recordedEvents = this.domainEvents.slice();
    this.domainEvents = [];
    return recordedEvents;
  }

  protected record(event: DomainEvent): void {
    this.domainEvents.push(event);
  }

  abstract toPrimitives(): Record<string, any>;
}
```

---

#### `entity.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Modela objetos con identidad conceptual que viven dentro de un agregado pero que no son la raíz.
* **Comportamiento exigido:**
  * Debe definir una propiedad `readonly id: ValueObject<any>`.
  * `equals(other: Entity<T>): boolean`: La igualdad se determina exclusivamente por la coincidencia estricta de
    sus `id.equals(other.id)`.

```typescript
import { ValueObject } from '../value-object/value-object';

export abstract class Entity<T extends ValueObject<any>> {
  constructor(readonly id: T) {}

  equals(other: Entity<T>): boolean {
    if (other === null || other === undefined) return false;
    if (!(other instanceof Entity)) return false;
    return this.id.equals(other.id);
  }

  abstract toPrimitives(): Record<string, any>;
}
```

---

### 3.2 Bloque: `value-object/`

#### `value-object.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Elimina el antipatrón de *Primitive Obsession*. Garantiza que ningún dato entre al sistema en un
  estado inválido y modela atributos por su valor, no por identidad.
* **Comportamiento exigido:**
  * La propiedad `readonly value: T` debe ser de solo lectura.
  * Valida invariantes en el momento de la construcción.
  * `equals(other: ValueObject<T>): boolean`: Compara por valor profundo o primitivo.
  * `toString(): string`: Representación en texto para logs y serialización.

```typescript
import { InvalidArgumentError } from './invalid-argument.error';

export abstract class ValueObject<T extends Object | string | number | boolean> {
  readonly value: T;

  constructor(value: T) {
    this.value = Object.freeze(value);
    this.ensureValueIsDefined(value);
  }

  private ensureValueIsDefined(value: T): void {
    if (value === null || value === undefined) {
      throw new InvalidArgumentError('Value must be defined and not null');
    }
  }

  equals(other: ValueObject<T>): boolean {
    return (
      other !== null &&
      other !== undefined &&
      other.constructor === this.constructor &&
      other.value === this.value
    );
  }

  toString(): string {
    return String(this.value);
  }
}
```

---

#### `uuid.vo.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Estandariza los identificadores únicos en todo el sistema.
* **Comportamiento exigido:**
  * Valida formato UUID v4 estricto mediante regex.
  * `static random(): Uuid`: Fábrica estática para generar nuevos identificadores criptográficamente seguros.

```typescript
import { ValueObject } from './value-object';
import { InvalidArgumentError } from './invalid-argument.error';

export class Uuid extends ValueObject<string> {
  private static readonly UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

  constructor(value: string) {
    super(value);
    this.ensureIsValidUuid(value);
  }

  static random(): Uuid {
    const randomUuid = typeof crypto !== 'undefined' && crypto.randomUUID
      ? crypto.randomUUID()
      : 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
          const r = (Math.random() * 16) | 0;
          const v = c === 'x' ? r : (r & 0x3) | 0x8;
          return v.toString(16);
        });
    return new (this as any)(randomUuid);
  }

  private ensureIsValidUuid(id: string): void {
    if (!Uuid.UUID_REGEX.test(id)) {
      throw new InvalidArgumentError(`<${this.constructor.name}> does not allow the value <${id}>`);
    }
  }
}
```

---

#### `enum.vo.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Encapsula listas de valores finitos permitidos garantizando que solo estados válidos del negocio
  puedan instanciarse.
* **Comportamiento exigido:**
  * Recibe en su constructor el array de valores permitidos `validValues: T[]`.
  * Lanza `InvalidArgumentError` si el valor provisto no pertenece a la enumeración.

```typescript
import { ValueObject } from './value-object';
import { InvalidArgumentError } from './invalid-argument.error';

export abstract class EnumValueObject<T extends string | number> extends ValueObject<T> {
  constructor(value: T, readonly validValues: T[]) {
    super(value);
    this.checkValueIsValid(value);
  }

  private checkValueIsValid(value: T): void {
    if (!this.validValues.includes(value)) {
      this.throwErrorForInvalidValue(value);
    }
  }

  protected abstract throwErrorForInvalidValue(value: T): void;
}
```

---

### 3.3 Bloque: `event/`

#### `domain-event.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Modela un hecho inmutable que ocurrió en el pasado dentro del dominio. Permite la comunicación
  asíncrona y la sincronización eventual entre Bounded Contexts.
* **Comportamiento exigido:**
  * Propiedades inmutables: `aggregateId`, `eventId`, `occurredOn`, `eventName`.
  * `abstract toPrimitives(): Record<string, any>`: Devuelve los atributos serializados del evento.
  * Propiedad estática `static EVENT_NAME: string`: Nombre canónico del evento en notación de contexto
    (ej. `user.registered`, `order.paid`).
  * Método estático `fromPrimitives(...)`: Reconstruye el evento a partir de su carga JSON deserializada.

```typescript
import { Uuid } from '../value-object/uuid.vo';

export abstract class DomainEvent {
  static EVENT_NAME: string;

  readonly aggregateId: string;
  readonly eventId: string;
  readonly occurredOn: Date;
  readonly eventName: string;

  constructor(params: { eventName: string; aggregateId: string; eventId?: string; occurredOn?: Date }) {
    this.eventName = params.eventName;
    this.aggregateId = params.aggregateId;
    this.eventId = params.eventId || Uuid.random().value;
    this.occurredOn = params.occurredOn || new Date();
  }

  abstract toPrimitives(): Record<string, any>;
}
```

---

#### `domain-event-subscriber.ts` & `domain-event-class.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Define el contrato formal para cualquier receptor de eventos de dominio.

```typescript
import { DomainEvent } from './domain-event';

export type DomainEventClass = {
  EVENT_NAME: string;
  fromPrimitives(params: {
    aggregateId: string;
    eventId: string;
    occurredOn: Date;
    attributes: any;
  }): DomainEvent;
};

export interface DomainEventSubscriber<T extends DomainEvent> {
  subscribedTo(): Array<DomainEventClass>;
  on(domainEvent: T): Promise<void>;
}
```

---

### 3.4 Bloque: `criteria/` (Specification Pattern)

* **Por qué existe:** Permite que los casos de uso soliciten colecciones dinámicas filtradas, ordenadas y paginadas
  sin acoplarse a SQL, sintaxis de TypeORM, MongoDB Query Filters ni ElasticSearch DSL.
* **Componentes obligatorios [ESTRICTO]:**

```typescript
// filter-operator.ts
export enum Operator {
  EQUAL = '=',
  NOT_EQUAL = '!=',
  GT = '>',
  LT = '<',
  CONTAINS = 'CONTAINS',
  NOT_CONTAINS = 'NOT_CONTAINS'
}

export class FilterOperator extends EnumValueObject<Operator> {
  constructor(value: Operator) {
    super(value, Object.values(Operator));
  }

  protected throwErrorForInvalidValue(value: Operator): void {
    throw new InvalidArgumentError(`The filter operator <${value}> is invalid`);
  }
}
```

```typescript
// filter.ts
export class Filter {
  constructor(
    readonly field: FilterField,
    readonly operator: FilterOperator,
    readonly value: FilterValue
  ) {}

  static fromValues(values: Map<string, string>): Filter {
    const field = values.get('field');
    const operator = values.get('operator');
    const value = values.get('value');

    if (!field || !operator || !value) {
      throw new InvalidArgumentError('Filter is not valid. Must contain field, operator, and value');
    }

    return new Filter(
      new FilterField(field),
      new FilterOperator(operator as Operator),
      new FilterValue(value)
    );
  }
}
```

```typescript
// criteria.ts
export class Criteria {
  constructor(
    readonly filters: Filters,
    readonly order: Order,
    readonly limit?: number,
    readonly offset?: number
  ) {}

  hasFilters(): boolean {
    return this.filters.filters.length > 0;
  }

  hasOrder(): boolean {
    return !this.order.orderType.isNone();
  }
}
```

---

### 3.5 Bloque: `bus/`

* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Define los contratos abstractos de mensajería sincrónica y asincrónica para desacoplar los
  emisores de los receptores.

```typescript
// command-bus.ts
export interface CommandBus {
  dispatch(command: any): Promise<void>;
}

// query-bus.ts
export interface QueryBus {
  ask<R>(query: any): Promise<R>;
}

// event-bus.ts
import { DomainEvent } from '../event/domain-event';
import { DomainEventSubscriber } from '../event/domain-event-subscriber';

export interface EventBus {
  publish(events: Array<DomainEvent>): Promise<void>;
  addSubscribers(subscribers: Array<DomainEventSubscriber<DomainEvent>>): void;
}
```

---

### 3.6 Bloque: `result/` (Either / Result Pattern)

* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Modela el resultado de operaciones de dominio de forma determinista y tipada. Obliga a que los
  errores de negocio esperados se traten como valores de retorno en lugar de excepciones no controladas.

```typescript
// failure.ts
export abstract class Failure {
  constructor(
    readonly message: string,
    readonly code: string,
    readonly context?: Record<string, any>
  ) {}
}

// result.ts
export class Result<T, E extends Failure> {
  private constructor(
    private readonly _isSuccess: boolean,
    private readonly _value?: T,
    private readonly _error?: E
  ) {}

  static ok<T, E extends Failure>(value: T): Result<T, E> {
    return new Result<T, E>(true, value, undefined);
  }

  static fail<T, E extends Failure>(error: E): Result<T, E> {
    return new Result<T, E>(false, undefined, error);
  }

  get isSuccess(): boolean {
    return this._isSuccess;
  }

  get isFailure(): boolean {
    return !this._isSuccess;
  }

  getValue(): T {
    if (!this._isSuccess) {
      throw new Error('Cannot get value from a failed Result. Check isSuccess first.');
    }
    return this._value as T;
  }

  getError(): E {
    if (this._isSuccess) {
      throw new Error('Cannot get error from a successful Result. Check isFailure first.');
    }
    return this._error as E;
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

* **[ESTRICTO]:** Todo símbolo accesible para los Bounded Contexts debe ser re-exportado explícitamente desde
  `src/index.ts`.

```typescript
// libs/shared/domain/src/index.ts
export * from './aggregate/aggregate-root';
export * from './aggregate/entity';

export * from './value-object/value-object';
export * from './value-object/string.vo';
export * from './value-object/number.vo';
export * from './value-object/uuid.vo';
export * from './value-object/enum.vo';
export * from './value-object/invalid-argument.error';

export * from './event/domain-event';
export * from './event/domain-event-subscriber';
export * from './event/domain-event-class';

export * from './criteria/criteria';
export * from './criteria/filter';
export * from './criteria/filters';
export * from './criteria/filter-field';
export * from './criteria/filter-operator';
export * from './criteria/filter-value';
export * from './criteria/order';
export * from './criteria/order-by';
export * from './criteria/order-type';

export * from './bus/command-bus';
export * from './bus/query-bus';
export * from './bus/event-bus';

export * from './result/result';
export * from './result/failure';
```
