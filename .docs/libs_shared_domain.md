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
│   │   └── enum.vo.ts                    # [ESTRICTO] VO base para enumeraciones tipadas
│   │
│   ├── exceptions/
│   │   ├── domain.exception.ts           # [ESTRICTO] Clase base para todas las excepciones de dominio
│   │   ├── invalid-argument.error.ts     # [ESTRICTO] Error arrojado ante violación de invariantes de VOs
│   │   ├── domain-not-found.error.ts     # [ESTRICTO] Error cuando un agregado o recurso de negocio no existe
│   │   └── domain-conflict.error.ts      # [ESTRICTO] Error ante conflictos de reglas (duplicados, estado incompatible)
│   │
│   ├── event/
│   │   ├── domain-event.ts               # [ESTRICTO] Clase base abstracta de eventos de dominio
│   │   ├── domain-event-subscriber.ts    # [ESTRICTO] Interfaz de contrato para suscriptores
│   │   └── domain-event-class.ts         # [ESTRICTO] Tipo para la referencia estática de eventos
│   │
│   ├── criteria/
│   │   ├── criteria.ts                   # [ESTRICTO] Raíz de la especificación de consulta (con factory Criteria.empty())
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
│   │   ├── command-bus.ts                # [ESTRICTO] Interfaz del bus de comandos tipado
│   │   ├── query-bus.ts                  # [ESTRICTO] Interfaz del bus de consultas tipado
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
  * La propiedad `readonly value: T` debe ser de solo lectura e inmutable.
  * Valida invariantes antes de asignar y congelar el estado.
  * `equals(other: ValueObject<T>): boolean`: Compara por valor profundo o primitivo.
  * `toString(): string`: Representación en texto para logs y serialización.

```typescript
import { InvalidArgumentError } from '../exceptions/invalid-argument.error';

export abstract class ValueObject<T extends Object | string | number | boolean> {
  readonly value: T;

  constructor(value: T) {
    this.ensureValueIsDefined(value);
    this.value = Object.freeze(value);
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

#### `string.vo.ts` & `number.vo.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// string.vo.ts
import { InvalidArgumentError } from '../exceptions/invalid-argument.error';

export abstract class StringValueObject {
  constructor(readonly value: string) {
    this.ensureIsNotEmpty(value);
  }

  private ensureIsNotEmpty(value: string): void {
    if (value.trim().length === 0) {
      throw new InvalidArgumentError(`${this.constructor.name} cannot be empty`);
    }
  }

  toString(): string {
    return this.value;
  }

  equals(other: StringValueObject): boolean {
    return this.value === other.value;
  }
}
```

```typescript
// number.vo.ts
import { InvalidArgumentError } from '../exceptions/invalid-argument.error';

export abstract class NumberValueObject {
  constructor(readonly value: number) {
    this.ensureIsValidNumber(value);
  }

  private ensureIsValidNumber(value: number): void {
    if (!Number.isFinite(value)) {
      throw new InvalidArgumentError(`${this.constructor.name} must be a finite number`);
    }
  }

  equals(other: NumberValueObject): boolean {
    return this.value === other.value;
  }
}
```

---

#### `uuid.vo.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Estandariza los identificadores únicos en todo el sistema.
* **Comportamiento exigido:**
  * Valida formato UUID v4 estricto mediante regex.
  * `static random(): Uuid`: Fábrica estática para generar nuevos identificadores criptográficamente seguros
    utilizando el estándar Web Crypto API (`crypto.randomUUID()`).

```typescript
import { ValueObject } from './value-object';
import { InvalidArgumentError } from '../exceptions/invalid-argument.error';

export class Uuid extends ValueObject<string> {
  private static readonly UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

  constructor(value: string) {
    super(value);
    this.ensureIsValidUuid(value);
  }

  static random(): Uuid {
    if (typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function') {
      return new (this as any)(crypto.randomUUID());
    }
    throw new Error('Cryptographically secure crypto.randomUUID() is required in the runtime environment');
  }

  private ensureIsValidUuid(id: string): void {
    if (!Uuid.UUID_REGEX.test(id)) {
      throw new InvalidArgumentError(`<${this.constructor.name}> does not allow the value <${id}>`);
    }
  }
}
```

> **Nota de Portabilidad:** En entornos como React Native con Hermes, puede ser necesario incluir un polyfill para `crypto.randomUUID()` (ej. `react-native-get-random-values`).

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
import { InvalidArgumentError } from '../exceptions/invalid-argument.error';

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

### 3.3 Bloque: `exceptions/` (Jerarquía Tipada de Errores de Dominio)

* **Por qué existe:** Define una jerarquía de errores de dominio de TypeScript puro que permite a los filtros de
  infraestructura (como `DomainExceptionFilter`) capturar y traducir errores con `instanceof`, eliminando el
  frágil antipatrón de comparar nombres de error con strings.

```typescript
// domain.exception.ts
export abstract class DomainException extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

// invalid-argument.error.ts
export class InvalidArgumentError extends DomainException {}

// domain-not-found.error.ts
export class DomainNotFoundError extends DomainException {}

// domain-conflict.error.ts
export class DomainConflictError extends DomainException {}

// forbidden.exception.ts
export class ForbiddenException extends DomainException {}
```

---

### 3.4 Bloque: `event/`

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

### 3.5 Bloque: `criteria/` (Specification Pattern)

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
// filter-field.ts
export class FilterField {
  constructor(readonly value: string) {}
}

// filter-value.ts
export class FilterValue {
  constructor(readonly value: string | number | boolean) {}
}
```

```typescript
// filters.ts
export class Filters {
  constructor(readonly filters: Array<Filter>) {}

  static fromValues(values: Array<Map<string, string>>): Filters {
    return new Filters(values.map((filterValues) => Filter.fromValues(filterValues)));
  }

  static empty(): Filters {
    return new Filters([]);
  }
}

// criteria.ts
export class Criteria {
  constructor(
    readonly filters: Filters,
    readonly order: Order,
    readonly limit?: number,
    readonly offset?: number
  ) {}

  static empty(): Criteria {
    return new Criteria(Filters.empty(), Order.none());
  }

  hasFilters(): boolean {
    return this.filters.filters.length > 0;
  }

  hasOrder(): boolean {
    return !this.order.orderType.isNone();
  }
}
```

```typescript
// order-type.ts
export enum OrderTypes {
  ASC = 'asc',
  DESC = 'desc',
  NONE = 'none',
}

export class OrderType {
  constructor(readonly value: OrderTypes) {}

  static asc(): OrderType { return new OrderType(OrderTypes.ASC); }
  static desc(): OrderType { return new OrderType(OrderTypes.DESC); }
  static none(): OrderType { return new OrderType(OrderTypes.NONE); }

  isNone(): boolean { return this.value === OrderTypes.NONE; }
}

// order-by.ts
export class OrderBy {
  constructor(readonly value: string) {}
}

// order.ts
import { OrderBy } from './order-by';
import { OrderType, OrderTypes } from './order-type';

export class Order {
  constructor(
    readonly orderBy: OrderBy,
    readonly orderType: OrderType,
  ) {}

  static none(): Order {
    return new Order(new OrderBy(''), OrderType.none());
  }

  static asc(orderBy: string): Order {
    return new Order(new OrderBy(orderBy), OrderType.asc());
  }

  static desc(orderBy: string): Order {
    return new Order(new OrderBy(orderBy), OrderType.desc());
  }

  hasOrder(): boolean {
    return !this.orderType.isNone();
  }
}
```

---

### 3.6 Bloque: `bus/`

* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Define los contratos abstractos de mensajería sincrónica y asincrónica para desacoplar los
  emisores de los receptores con tipado estricto.

```typescript
// command-bus.ts
// Nota: La interfaz base Command conceptualmente reside en @monorepo/shared/application

export interface CommandBus {
  dispatch<C>(command: C): Promise<void>;
}

// query-bus.ts
// Nota: La interfaz base Query conceptualmente reside en @monorepo/shared/application

export interface QueryBus {
  ask<R, Q>(query: Q): Promise<R>;
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

### 3.7 Bloque: `result/` (Either / Result Pattern)

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

### 3.8 Guía de Decisión: Result vs DomainException

* **`DomainException` (por defecto):** Para violaciones de invariantes en Value Objects y Agregados, entidad no encontrada, o acceso denegado (forbidden). Son capturados automáticamente por el `DomainExceptionFilter` en NestJS.
* **`Result<T, Failure>` (opcional):** Para cadenas complejas de validación en servicios de frontend o cuando es necesario acumular múltiples errores. **NO** se utiliza en handlers de comandos/consultas del backend (esos usan excepciones).

---

### 3.9 Bloque: `types/`

* **Nivel:** **[OPCIONAL]**

```typescript
// nullable.type.ts
export type Nullable<T> = T | null;
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

export * from './exceptions/domain.exception';
export * from './exceptions/invalid-argument.error';
export * from './exceptions/domain-not-found.error';
export * from './exceptions/domain-conflict.error';
export * from './exceptions/forbidden.exception';

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

export * from './types/nullable.type';
```
