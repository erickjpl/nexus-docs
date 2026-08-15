# Especificación: `libs/shared/testing`

Este módulo constituye las **Utilidades Compartidas de Testing (Shared Testing Kernel)**. Proporciona generadores de
datos aleatorios deterministas (**Object Mother Pattern**), wrappers de librerías de falsificación de datos (Faker) y
Mocks semánticos de buses con aserciones declarativas de negocio.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Testing:** Este paquete está diseñado **únicamente** para ser importado por archivos
  de prueba (`*.spec.ts`, `*.test.ts`, `*.steps.ts`) y carpetas `testing/` de los Bounded Contexts. Nunca debe
  importarse en código de producción.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:shared-testing", "scope:shared"]`.
* **[ESTRICTO] Tests Desacoplados de Fixtures Rígidos:** Prohibido el uso de archivos JSON estáticos con datos fijos.
  Cada test debe generar sus propios datos válidos y aleatorios a través de los *Object Mothers*.

---

## 2. Estructura de Directorios

```text
libs/shared/testing/
├── src/
│   ├── mother/
│   │   ├── mother-creator.ts                     # [ESTRICTO] Wrapper centralizado de Faker
│   │   ├── uuid.mother.ts                        # [ESTRICTO] Generador de identificadores UUID válidos
│   │   ├── string.mother.ts                      # [ESTRICTO] Generador de cadenas, palabras y textos
│   │   ├── number.mother.ts                      # [ESTRICTO] Generador de números enteros y rangos
│   │   ├── boolean.mother.ts                     # [OPCIONAL] Generador de booleanos aleatorios
│   │   └── date.mother.ts                        # [OPCIONAL] Generador de fechas
│   │
│   ├── criteria/
│   │   └── criteria.mother.ts                    # [ESTRICTO] Generador de especificaciones Criteria para tests
│   │
│   ├── mocks/
│   │   ├── event-bus.mock.ts                     # [ESTRICTO] Mock de EventBus con aserciones semánticas
│   │   ├── command-bus.mock.ts                   # [ESTRICTO] Mock de CommandBus para verificar despachos
│   │   └── query-bus.mock.ts                     # [ESTRICTO] Mock de QueryBus con respuestas programables
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `mother/` (Object Mother Pattern)

#### `mother-creator.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Encapsula la librería generadora de datos falsos (ej. `@faker-js/faker`).

> [!NOTE]
> Todos los ejemplos asumen el uso de `@faker-js/faker` versión 8 o superior.

```typescript
import { faker } from '@faker-js/faker';

export class MotherCreator {
  static random(): typeof faker {
    return faker;
  }
}
```

---

#### `uuid.mother.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
import { MotherCreator } from './mother-creator';

export class UuidMother {
  static random(): string {
    return MotherCreator.random().string.uuid();
  }
}
```

---

#### `string.mother.ts` & `number.mother.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// string.mother.ts
import { MotherCreator } from './mother-creator';

export class StringMother {
  static word(): string {
    return MotherCreator.random().lorem.word();
  }

  static words(count: number = 3): string {
    return MotherCreator.random().lorem.words(count);
  }

  static paragraph(): string {
    return MotherCreator.random().lorem.paragraph();
  }
}

// number.mother.ts
import { MotherCreator } from './mother-creator';

export class NumberMother {
  static random(min: number = 1, max: number = 1000): number {
    return MotherCreator.random().number.int({ min, max });
  }

  static positive(): number {
    return this.random(1, 999999);
  }
}
```

---

#### `boolean.mother.ts` & `date.mother.ts`
* **Nivel:** **[OPCIONAL]**

```typescript
// boolean.mother.ts
import { faker } from '@faker-js/faker';

export class BooleanMother {
  static random(): boolean {
    return faker.datatype.boolean();
  }

  static true(): boolean { return true; }
  static false(): boolean { return false; }
}
```

```typescript
// date.mother.ts
import { faker } from '@faker-js/faker';

export class DateMother {
  static random(): Date {
    return faker.date.recent();
  }

  static past(): Date { return faker.date.past(); }
  static future(): Date { return faker.date.future(); }

  static randomIso(): string {
    return DateMother.random().toISOString();
  }
}
```

---

### 3.2 Bloque: `criteria/`

#### `criteria.mother.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
import {
  Criteria,
  Filters,
  Order,
  Filter,
  FilterField,
  FilterOperator,
  FilterValue,
  Operator
} from '@monorepo/shared/domain';

export class CriteriaMother {
  static empty(): Criteria {
    return new Criteria(new Filters([]), Order.none());
  }

  static withOneFilter(field: string, operator: Operator, value: string): Criteria {
    const filter = new Filter(new FilterField(field), new FilterOperator(operator), new FilterValue(value));
    return new Criteria(new Filters([filter]), Order.none());
  }
}
```

---

### 3.3 Bloque: `mocks/` (Mocks Semánticos con Aserciones)

#### `event-bus.mock.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
import { DomainEvent, DomainEventSubscriber, EventBus } from '@monorepo/shared/domain';

export class EventBusMock implements EventBus {
  private publishSpy = jest.fn();
  private publishedEvents: Array<DomainEvent> = [];

  async publish(events: Array<DomainEvent>): Promise<void> {
    this.publishSpy(events);
    this.publishedEvents.push(...events);
  }

  addSubscribers(subscribers: Array<DomainEventSubscriber<DomainEvent>>): void {
    // No-op para tests unitarios
  }

  assertLastPublishedEventIs(expectedEventClass: { EVENT_NAME: string }): void {
    expect(this.publishedEvents.length).toBeGreaterThan(0);
    const lastEvent = this.publishedEvents[this.publishedEvents.length - 1];
    expect(lastEvent.eventName).toEqual(expectedEventClass.EVENT_NAME);
  }

  assertPublishedEvents(expectedEvents: Array<DomainEvent>): void {
    expect(this.publishedEvents.length).toEqual(expectedEvents.length);
    expect(this.publishedEvents).toEqual(expectedEvents);
  }

  assertNotPublishedAnyEvent(): void {
    expect(this.publishedEvents.length).toEqual(0);
  }
}
```

---

#### `command-bus.mock.ts` & `query-bus.mock.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// command-bus.mock.ts
import { Command } from '@monorepo/shared/application';
import { CommandBus } from '@monorepo/shared/domain';

export class CommandBusMock implements CommandBus {
  private dispatchSpy = jest.fn();

  async dispatch(command: Command): Promise<void> {
    this.dispatchSpy(command);
  }

  assertDispatchedCommand(expectedCommand: Command): void {
    expect(this.dispatchSpy).toHaveBeenCalledWith(expectedCommand);
  }
}

// query-bus.mock.ts
import { Query, Response } from '@monorepo/shared/application';
import { QueryBus } from '@monorepo/shared/domain';

export class QueryBusMock implements QueryBus {
  private response?: Response;

  setResponse<R extends Response>(response: R): void {
    this.response = response;
  }

  async ask<R extends Response>(query: Query): Promise<R> {
    if (!this.response) {
      throw new Error('QueryBusMock response has not been set. Call setResponse() before executing the test.');
    }
    return this.response as R;
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/testing/src/index.ts
export * from './mother/mother-creator';
export * from './mother/uuid.mother';
export * from './mother/string.mother';
export * from './mother/number.mother';
export * from './mother/boolean.mother';
export * from './mother/date.mother';

export * from './criteria/criteria.mother';

export * from './mocks/event-bus.mock';
export * from './mocks/command-bus.mock';
export * from './mocks/query-bus.mock';
```
