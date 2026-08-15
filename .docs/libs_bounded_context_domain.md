# Especificación: `libs/{bounded_context}/domain`

Este módulo constituye el **Dominio Puro del Bounded Context** (el centro de la Onion Architecture). Aquí residen
exclusivamente las reglas de negocio, el modelo conceptual, el Lenguaje Ubicuo (*Ubiquitous Language*), los eventos
de dominio y los contratos (puertos) de persistencia y servicios.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Dependencias Prohibidas:** No se permite ninguna dependencia hacia frameworks (NestJS, React, Express),
  ORMs (TypeORM, Prisma, Mongoose), clientes HTTP o librerías de infraestructura.
* **[ESTRICTO] Dependencias Permitidas:** Únicamente puede importar abstracciones desde `@monorepo/shared/domain`
  (etiqueta Nx `type:shared-domain`).
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:domain", "scope:{context}"]`.
* **[ESTRICTO] Invariantes Encapsuladas:** Es imposible crear un Agregado, Entidad o Value Object en un estado inválido.
  Todas las validaciones de negocio se ejecutan durante la instanciación.

---

## 2. Estructura de Directorios

```text
libs/{bounded_context}/domain/
├── src/
│   ├── model/
│   │   ├── {aggregate}.aggregate.ts              # [ESTRICTO] Raíz del agregado principal
│   │   ├── {aggregate}-id.vo.ts                  # [ESTRICTO] Identificador único del agregado (extiende Uuid)
│   │   ├── {value_object}.vo.ts                  # [ESTRICTO] Value Objects específicos del modelo
│   │   └── {entity}.entity.ts                    # [OPCIONAL] Entidades secundarias pertenecientes al agregado
│   │
│   ├── events/
│   │   └── {aggregate}-{action}.domain-event.ts  # [ESTRICTO] Eventos de negocio disparados por el agregado
│   │
│   ├── exceptions/
│   │   └── {business-error}.exception.ts         # [ESTRICTO] Excepciones/Failures específicos del dominio
│   │
│   ├── factories/
│   │   └── {aggregate}.factory.ts                # [OPCIONAL] Construcción compleja con múltiples pasos
│   │
│   ├── services/
│   │   └── {name}.domain-service.ts              # [OPCIONAL] Lógica de negocio que cruza múltiples agregados
│   │
│   ├── ports/
│   │   ├── repositories/
│   │   │   └── {aggregate}.repository.ts         # [ESTRICTO] Contrato puro de persistencia del agregado
│   │   └── services/
│   │       └── {external-contract}.port.ts       # [OPCIONAL] Contrato para servicios externos requeridos
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `model/`

#### `{aggregate}.aggregate.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es el guardián de la coherencia transaccional del Bounded Context. Protege las invariantes de
  negocio y emite eventos de dominio cuando su estado muta.
* **Comportamiento exigido:**
  * Debe extender `AggregateRoot` de `@monorepo/shared/domain`.
  * Constructor con Value Objects tipados.
  * **Método estático `create(...)` (Fábrica de Negocio):** Se utiliza al crear un agregado nuevo; ejecuta validaciones
    y registra el evento de creación con `this.record(new UserRegisteredDomainEvent(...))`.
  * **Método estático `fromPrimitives(plainData)` (Rehidratación):** Se utiliza para reconstruir el agregado desde
    la base de datos **sin** disparar eventos de dominio.
  * **Método `toPrimitives()`:** Serializa el agregado y sus Value Objects a primitivos puros de JavaScript.

```typescript
// libs/users/domain/src/model/user.aggregate.ts
import { AggregateRoot } from '@monorepo/shared/domain';
import { UserId } from './user-id.vo';
import { UserName } from './user-name.vo';
import { UserEmail } from './user-email.vo';
import { UserRegisteredDomainEvent } from '../events/user-registered.domain-event';

export class User extends AggregateRoot {
  constructor(
    readonly id: UserId,
    private _name: UserName,
    private _email: UserEmail
  ) {
    super();
  }

  static create(id: UserId, name: UserName, email: UserEmail): User {
    const user = new User(id, name, email);

    user.record(
      new UserRegisteredDomainEvent({
        aggregateId: id.value,
        name: name.value,
        email: email.value
      })
    );

    return user;
  }

  static fromPrimitives(plain: { id: string; name: string; email: string }): User {
    return new User(
      new UserId(plain.id),
      new UserName(plain.name),
      new UserEmail(plain.email)
    );
  }

  toPrimitives(): { id: string; name: string; email: string } {
    return {
      id: this.id.value,
      name: this._name.value,
      email: this._email.value
    };
  }

  rename(newName: UserName): void {
    this._name = newName;
  }

  get name(): UserName {
    return this._name;
  }

  get email(): UserEmail {
    return this._email;
  }
}
```

---

#### `{aggregate}-id.vo.ts` & Value Objects
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Evita tipos primitivos ambiguos y encapsula las reglas de formato, tamaño o rango.

```typescript
// libs/users/domain/src/model/user-id.vo.ts
import { Uuid } from '@monorepo/shared/domain';

export class UserId extends Uuid {}

// libs/users/domain/src/model/user-name.vo.ts
import { StringValueObject, InvalidArgumentError } from '@monorepo/shared/domain';

export class UserName extends StringValueObject {
  constructor(value: string) {
    super(value);
    this.ensureLengthIsBetween(value, 3, 50);
  }

  private ensureLengthIsBetween(value: string, min: number, max: number): void {
    if (value.length < min || value.length > max) {
      throw new InvalidArgumentError(
        `User name <${value}> must be between ${min} and ${max} characters`
      );
    }
  }
}

// libs/users/domain/src/model/user-email.vo.ts
import { StringValueObject, InvalidArgumentError } from '@monorepo/shared/domain';

export class UserEmail extends StringValueObject {
  private static readonly EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  constructor(value: string) {
    super(value);
    this.ensureIsValidEmail(value);
  }

  private ensureIsValidEmail(email: string): void {
    if (!UserEmail.EMAIL_REGEX.test(email)) {
      throw new InvalidArgumentError(`User email <${email}> is invalid`);
    }
  }
}
```

---

### 3.2 Bloque: `events/`

#### `{aggregate}-{action}.domain-event.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/domain/src/events/user-registered.domain-event.ts
import { DomainEvent } from '@monorepo/shared/domain';

type UserRegisteredDomainEventAttributes = {
  readonly name: string;
  readonly email: string;
};

export class UserRegisteredDomainEvent extends DomainEvent {
  static readonly EVENT_NAME = 'user.registered';

  readonly name: string;
  readonly email: string;

  constructor(params: {
    aggregateId: string;
    name: string;
    email: string;
    eventId?: string;
    occurredOn?: Date;
  }) {
    super({
      eventName: UserRegisteredDomainEvent.EVENT_NAME,
      aggregateId: params.aggregateId,
      eventId: params.eventId,
      occurredOn: params.occurredOn
    });

    this.name = params.name;
    this.email = params.email;
  }

  toPrimitives(): UserRegisteredDomainEventAttributes {
    return {
      name: this.name,
      email: this.email
    };
  }

  static fromPrimitives(params: {
    aggregateId: string;
    attributes: UserRegisteredDomainEventAttributes;
    eventId: string;
    occurredOn: Date;
  }): DomainEvent {
    return new UserRegisteredDomainEvent({
      aggregateId: params.aggregateId,
      name: params.attributes.name,
      email: params.attributes.email,
      eventId: params.eventId,
      occurredOn: params.occurredOn
    });
  }
}
```

---

### 3.3 Bloque: `ports/repositories/`

#### `{aggregate}.repository.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es el contrato de persistencia puro que exige el dominio.

```typescript
// libs/users/domain/src/ports/repositories/user.repository.ts
import { Criteria } from '@monorepo/shared/domain';
import { User } from '../../model/user.aggregate';
import { UserId } from '../../model/user-id.vo';

export interface UserRepository {
  save(user: User): Promise<void>;
  search(id: UserId): Promise<User | null>;
  searchAll(): Promise<Array<User>>;
  matching(criteria: Criteria): Promise<Array<User>>;
}
```

---

### 3.4 Bloque: `services/` (Domain Services)

#### `{name}.domain-service.ts`
* **Nivel:** **[OPCIONAL]**

```typescript
// libs/users/domain/src/services/user-email-uniqueness-checker.service.ts
import { UserRepository } from '../ports/repositories/user.repository';
import { UserEmail } from '../model/user-email.vo';
import {
  Criteria,
  Filter,
  FilterField,
  FilterOperator,
  FilterValue,
  Operator,
  Filters,
  Order
} from '@monorepo/shared/domain';

export class UserEmailUniquenessChecker {
  constructor(private readonly repository: UserRepository) {}

  async isUnique(email: UserEmail): Promise<boolean> {
    const criteria = new Criteria(
      new Filters([
        new Filter(
          new FilterField('email'),
          new FilterOperator(Operator.EQUAL),
          new FilterValue(email.value)
        )
      ]),
      Order.none(),
      1
    );

    const existingUsers = await this.repository.matching(criteria);
    return existingUsers.length === 0;
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/domain/src/index.ts
export * from './model/user.aggregate';
export * from './model/user-id.vo';
export * from './model/user-name.vo';
export * from './model/user-email.vo';

export * from './events/user-registered.domain-event';

export * from './ports/repositories/user.repository';
export * from './services/user-email-uniqueness-checker.service';
```
